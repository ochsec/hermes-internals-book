# Agent Spawning & Delegation

Hermes can create additional agent instances to work in parallel. There are three distinct mechanisms for this: **subagent delegation** (`delegate_task`, implemented in `tools/delegate_tool*.py` and `tools/async_delegation.py`), **process spawning** (running a new `hermes` process), and the **kanban dispatcher** (gateway-driven worker spawning). This chapter explains all three, their isolation boundaries, and when each is used.

## 3.1 Three Mechanisms Compared

| Dimension | `delegate_task` | Spawning `hermes` process | Kanban worker |
|-----------|----------------|----------------------------|---------------|
| **Isolation** | Separate conversation, shared process | Fully independent OS process | Independent OS process, board-pinned |
| **Duration** | Minutes to an hour | Hours or days | Task-scoped; dispatcher-observed |
| **Tool access** | Parent's toolsets minus child-blocked tools | Full tool access | Focused `kanban_*` + task tools |
| **Interactivity** | No — `clarify` is blocked | Yes — supports PTY mode | No — headless `chat -q` |
| **Parent blocking** | No — dispatch returns a handle; results arrive as new turns (inline join only in one-shot CLI and cron sessions) | Parent continues immediately | Dispatcher never blocks |
| **Use case** | Reasoning-heavy parallel subtasks | Long autonomous missions | Durable, resumable work queues |

## 3.2 Delegation (`delegate_task`)

Implemented in the `delegation` toolset (exposed as `delegate_task`), this is the lighter-weight of the mechanisms.

### Invocation

```python
# Single task
delegate_task(goal="Research GRPO papers", context="Focus on 2024+")

# Batch (parallel)
delegate_task(tasks=[
    {"goal": "Backend API design", "context": "FastAPI"},
    {"goal": "Frontend scaffolding", "context": "React"},
])

# Structured result contract (per task)
delegate_task(goal="Extract invoice fields", output_schema={...}, images=["./scan.png"])
```

### Background-First Dispatch

Top-level `delegate_task` calls always dispatch to the background rail: the call returns a dispatch handle with live transcript paths (`~/.hermes/cache/delegation/live/<id>/task-<n>.log`) and each result re-enters the parent **as a new message between turns**. The legacy `background` parameter is deprecated and ignored. The one exception: sessions without a later-result consumer (one-shot CLI runs, cron jobs) join the children synchronously and get results in the tool call itself. By default the whole batch returns as one consolidated message; `delegation.independent_completions: true` makes each task (or result-delivery `group`) return on its own as it finishes.

### Live Orchestration (`action`)

While children run, the same tool controls them: `action="list"` returns ids/goals/status/transcripts, `action="steer"` queues a course-correction into one child (`subagent_id` + `message`), and `action="stop"` ends one child early (a partial result still returns). Control actions run synchronously and bypass the pause gate, depth limit and async dispatch. New spawns can also be paused globally via the TUI (`p` in `/agents`) or the `delegation.pause` RPC.

### Lifecycle

1. **Parent sends goal + context** — The parent Hermes instance calls `delegate_task`. The goal becomes the child's first user turn, not part of its system prompt.
2. **Child agent instantiated** — A new `AIAgent` is created with:
   - Isolated conversation history (an ephemeral, focused system prompt built from goal, context, workspace path and the project context files — AGENTS.md, CLAUDE.md, etc. — but never SOUL.md or the user's profile)
   - The parent's toolsets minus child-blocked tools (`DELEGATE_BLOCKED_TOOLS`: `delegate_task`, `clarify`, `memory`, `send_message`, `cronjob_manage`), stripped via whole-toolset removal plus one-tool deny toolsets so blocked names inside composite bundles still subtract. MCP toolsets are kept by default (`delegation.inherit_mcp_toolsets`, default true).
   - Its own `IterationBudget` capped at `delegation.max_iterations` (default 250; caller-supplied values are ignored, config is authoritative)
   - A dedicated child `SessionDB` handle so a background child's transcript survives the parent's lifecycle closing its own connection
   - Non-interactive approval resolution: `delegation.subagent_auto_approve` (default false → auto-deny) because the parent TUI owns stdin
3. **Child runs independently** — The child executes the loop until it returns text, hits its iteration budget, or crosses the inactivity cap (`delegation.child_timeout_seconds`, default off). A heartbeat monitor marks a child stale after 15 idle cycles (450 s) or 40 cycles stuck on one tool (1200 s); an in-flight model wait refreshes the activity clock, so slow models are not killed, and after a stale verdict the parent keeps polling for 2 s to collect a real result the child already wrote.
4. **Summary returned** — The child returns a condensed summary, trimmed to `min(dynamic headroom budget, delegation.max_summary_chars=24000)`; over-cap text spills to `~/.hermes/cache/delegation/` as head+tail with a `read_file` offset footer.
5. **Parent resumes** — The summary re-enters as a new turn (or the tool result, in inline-join sessions).

Asynchronously dispatched delegations (via the daemon pool in `tools/async_delegation.py`) extend the lifecycle with a **`finalizing`** transitional state and a **`stalling`** state. After the child completes but before the result is persisted to SQLite and published to the completion queue, the record's status flips to `finalizing` so process shutdown cannot kill a worker in that gap; `active_count()` counts all live states (`running`, `stalling`, `finalizing`) so capacity is not reclaimed prematurely. A single stale-monitor thread samples each dispatch's progress token every 30 s; a frozen child is interrupted and given a 120 s grace window to unwind via the normal finalize path before being force-finalized with a terminal `stalled` event (structured `stall_*` metadata attached).

### ACP Transport: Config-Pinned Only

The model still cannot choose a subagent's transport: `delegate_task` runs in-process children, and no model-facing parameter selects ACP. A trusted operator can pin an ACP transport through the delegation config (`override_acp_command` / `override_acp_args`, resolving the `delegation.command` key) so every child runs over e.g. `copilot-acp`; a pinned ACP command forces provider `copilot-acp` with `chat_completions` mode, and a pinned provider without an ACP command clears the parent's ACP transport entirely.

### Isolation Guarantees

- **Conversation isolation** — The child cannot see the parent's history, and vice versa.
- **Terminal isolation** — The child gets its own terminal session (its own `task_id`, working directory and shell state, plus a file-ops cache).
- **Memory isolation** — `skip_memory=True` prevents the child from reading or writing persistent memory; the `memory` tool is also blocked. The child's prompt embeds project context files instead.
- **Process isolation** — The child runs in the same Python process but in a separate thread. If the parent is interrupted, the child is cancelled and reports an `interrupted` completion with partial output.
- **Optional worktree isolation** — `delegation.worktree_isolation: true` gives each child its own git worktree off the parent's HEAD (`<repo>/.worktrees/subagent-<id>`, branch `hermes-subagent/<id>`); git-only and local-terminal-backend-only, silently downgraded otherwise. After the child finishes, the worktree is pruned only on proof (zero commits and a clean tree); any failed probe keeps it with an `inspection_failed` note so "no work" is never assumed. Git subprocesses run with neutralized repository-named filters and non-interactive env (GHSA-7x36-8jrh-v4pw class hardening).

### Roles: Depth-Derived Nesting

The `role` argument is legacy wire-compat and ignored. Capability is derived from depth: a child may itself call `delegate_task` iff the `delegation.orchestrator_enabled` kill switch (default on) is set and its depth stays below `delegation.max_spawn_depth` (default 1 = flat; floor 1, no ceiling). Raising `max_spawn_depth` enables hierarchical decomposition — each level multiplies API cost. Eligible orchestrators get the `delegation` toolset re-added unconditionally (role-granted, not inherited).

### Configuration

```yaml
delegation:
  model: ""                    # empty = inherit parent model
  provider: ""                 # empty = inherit parent provider + credentials
  base_url: ""
  api_key: ""
  api_mode: ""                 # chat_completions | codex_responses | anthropic_messages; empty = auto-detect
  request_overrides: {}        # per-child API kwargs (extra_body deep-merged 1 level)
  fallback_providers: null     # child fallback chain; null = inherit, [] = disable
  compression_threshold_tokens: 0   # optional absolute cap on a child's compaction trigger
  max_iterations: 250
  max_concurrent_children: 10  # parallel batch limit AND background concurrency cap
  max_summary_chars: 24000
  child_timeout_seconds: 0     # inactivity cap; 0 = off
  reasoning_effort: ""         # empty = inherit parent
  independent_completions: false
  max_spawn_depth: 1           # 1 = flat, 2 = orchestrator→leaf, 3+ = deeper
  orchestrator_enabled: true
  oneshot_max_children: 2      # spawn budget for hermes chat -q / --oneshot runs
  subagent_auto_approve: false
  surface_child_process_notifications: false
  worktree_isolation: false
```

### Child Agent Provider Overrides

`_build_child_agent` accepts `override_provider`, `override_base_url`, `override_api_key`, `override_api_mode`, `override_request_overrides` and `override_acp_command`/`override_acp_args` parameters that control how the child's provider configuration diverges from the parent's:

- **`override_request_overrides`** — When set, the child receives a fresh `request_overrides` dict (from `_resolve_delegation_credentials`, which merges over the parent's) rather than inheriting the parent's as-is. This ensures provider-specific request parameters (extra headers, body fields, etc.) are correct for the delegated provider, not leaked from the parent's.
- **max_tokens** — There is no separate override parameter any more; the child inherits the parent's integer `max_tokens` directly.

The child inherits a set of OpenRouter routing filters from the parent, including the two provider-gating attributes `provider_require_parameters` (default `False`) and `provider_data_collection` (default `""`). When a provider override is in effect, all of `_ROUTING_FILTER_DEFAULTS` (`providers_allowed`, `providers_ignored`, `providers_order`, `provider_sort`, `provider_require_parameters`, `provider_data_collection`) are reset to their defaults, since the parent's filters (e.g. `only=["Anthropic"]`) would silently force the child back onto the parent's provider. `api_mode` is re-derived rather than inherited whenever the child's provider differs from the parent's (each provider has its own wire protocol).

### 3.2.1 Durable Delegation Persistence

Async delegations dispatched through the daemon pool are persisted to an `async_delegations` table in `state.db` (the same SQLite database used by kanban and session state). This durability layer ensures that completions survive process crashes and that competing gateway processes do not double-deliver results.

#### Schema

The `async_delegations` table stores the full lifecycle of each delegation:

| Column | Purpose |
|--------|---------|
| `delegation_id` | Primary key (UUID) |
| `origin_session` | Session key that originated the delegation |
| `origin_ui_session_id` | UI session identifier |
| `origin_session_id` | Originating session's ID (added so restored events can prove ownership) |
| `parent_session_id` | Parent's session ID for context |
| `state` | `running` → `stalling` (stale, interrupted) → `finalizing` → `completed` / `failed` / `stalled` / `unknown` |
| `dispatched_at` | Epoch timestamp when the delegation was dispatched |
| `completed_at` | Epoch timestamp when the child agent finished |
| `updated_at` | Last modification time (used for retention pruning) |
| `event_json` | Serialized completion event (goal, summary, error, status, diagnostics) |
| `result_json` | Serialized result dict |
| `delivery_state` | `pending`, `delivered`, or `dropped` |
| `delivery_attempts` | Count of delivery attempts (incremented on each claim) |
| `delivered_at` | Timestamp of successful delivery |
| `owner_pid` | PID of the process that dispatched the delegation |
| `owner_started_at` | Process start time (for crash detection) |
| `task_json` | Serialized task payload (goal, context, model, role, routing origin, etc.) |
| `delivery_claim` | Current delivery claim token |
| `delivery_claimed_at` | Timestamp when the current claim was made |

The durable shape is owned by the canonical `SCHEMA_SQL` in `hermes_state_common.py`: opening the database runs `reconcile_state_schema(conn)`, which replays the idempotent DDL and reconciles columns declaratively. The module previously carried its own `CREATE TABLE` + `ALTER TABLE ADD COLUMN` list, which drifted from the canonical schema (same-name columns with different nullability depending on which authority touched the database first). A module-level `_DB_LOCK` still serializes writes within the process, but journal mode is owned by SessionDB — connections open with `wal=False` and apply durability barriers rather than forcing `PRAGMA journal_mode=WAL` from every short-lived connection. The database file gets owner-only (0600) hardening via `_secure_state_db_files`, and a replay must not resurrect a removed named profile's directory.

#### Dispatch Persistence

When a delegation is dispatched (`_persist_dispatch`), a row is inserted (upsert) with `state='running'`, `delivery_state='pending'`, the owning process's PID and start time, and a serialized task payload. The payload also captures a **routing origin** (`scope_id`, `user_id`, `user_name`, snapshotted on the parent thread because the daemon worker has no contextvars) so a restart-replayed completion can rebuild a full `SessionSource`, plus the owner's `cwd` so recovery can add a git-state hint. If the thread pool rejects the submission, the durable row is deleted so it does not linger as an orphan.

#### Completion Persistence

When the child agent finishes, `_persist_completion` updates the row with the completion event and result JSON, sets `completed_at`, and resets `delivery_state` to `pending` so the completion can be delivered to the origin session. This happens during the `finalizing` window — the in-memory record stays in `finalizing` status until after this SQLite commit, preventing a race where process shutdown kills the worker between the status flip and the commit.

#### Abandoned Delegation Recovery

`recover_abandoned_delegations()` runs at process startup and whenever completions are restored. It scans for records in `running` or `finalizing` state whose owning process has disappeared — determined by checking whether `owner_pid` still exists and whether its current start time matches `owner_started_at`. Records whose owner is gone are marked `state='unknown'` with a synthetic completion event (`status: "unknown"`, `error: "Delegation owner exited before recording a terminal result; outcome unknown."`) and `delivery_state='pending'`, so the parent session is at least notified that the delegation's outcome is indeterminate rather than silently lost. Two refinements:

- **Multi-child units** — children a batch had already recorded (`record_unit_child`) are replayed with their real results; the error text reports how many of the unit's children were recovered ("N/M child results recorded").
- **Recovery diagnostics** — verbatim transcript tails of the dead owner's task transcripts and a git snapshot hint of the owner's cwd (from `tools/async_delegation_recovery_hints.py`) ride in the event, so the parent can continue or re-dispatch from the event alone instead of opening files.

#### Delivery Claiming

When multiple gateway processes may compete to deliver the same completion, a claim-based protocol prevents duplicate delivery:

- `claim_completion_delivery(delegation_id, claim_id)` — Atomically claims a pending completion. A claim succeeds only if `delivery_state='pending'` and no active claim exists (or the previous claim is older than 5 minutes — `_CLAIM_LEASE_S` — treating it as stale). Returns `True` if the claim was acquired.
- `complete_completion_delivery(delegation_id, claim_id)` — Marks the completion as `delivered` and clears the claim. Only the consumer holding the matching `claim_id` can finalize delivery.
- `release_completion_delivery(delegation_id, claim_id)` — Releases a failed delivery claim so another consumer may retry. Clears the claim if it matches and the delivery is still pending.
- `defer_completion_delivery(delegation_id, claim_id)` — Backs off without consuming the row (the consumer is busy).
- `drop_completion_delivery(delegation_id, claim_id)` — Terminally marks a completion `dropped`; after `_MAX_DELIVERY_ATTEMPTS = 8` failed attempts an unroutable row converges here automatically so it cannot retry forever.
- `mark_completion_delivered(delegation_id)` — Simpler unconditional acknowledge for paths that do not need claim-based competition.

The event consumer path wraps these: `claim_event_delivery(evt, consumer)` generates a claim ID (`{consumer}:{pid}:{uuid}`) and calls `claim_completion_delivery`, returning the claim ID on success or `None` if the claim could not be acquired (or `""` for non-durable events that need no token). `complete_event_delivery(evt, claim_id)` and `release_event_delivery(evt, claim_id)` delegate to the corresponding SQLite operations. Legacy events created before durable dispatch are handled gracefully — if no row exists for the `delegation_id`, the claim is treated as successful so the event proceeds without blocking.

#### Undelivered Completion Restoration

`restore_undelivered_completions(target_queue)` is called at process startup to re-enqueue durable pending completions as fresh turns. It first calls `recover_abandoned_delegations()` to classify orphaned records, then selects all records where `state != 'running'` and `delivery_state='pending'` with a non-null `event_json`, ordered by `completed_at` then `delegation_id`, and puts each event onto the target queue. Two guards bound the replay:

- **Ownership stamping** — Every restored event is stamped `restored=True` (in memory only). It came from a *previous* process, so drain paths without an ownership filter must leave it queued for a consumer that can positively prove ownership; otherwise a brand-new session would adopt a dead session's delegation results seconds after boot.
- **Replay age** — Pending completions older than `_MAX_COMPLETION_REPLAY_AGE_S` (48 hours) are terminally dropped instead of re-running as a full-context turn nobody is waiting on.

A periodic **orphan sweep** (`sweep_orphaned_completions`, at most once per `ORPHAN_SWEEP_INTERVAL_S = 30` per home) covers the gap a desktop reload leaves: a completion whose owner died while *this* process was already running would otherwise wait for the next restart. Rows just touched (under `_ORPHAN_STALE_S = 60` idle) are skipped. A consumer that discards its copy with the row still pending hands it back via `return_completion_offer` — the delivery claim stays the only thing that settles the row.

#### Retention Bounds

Two constants bound the growth of the durable table:

- `_DURABLE_RETENTION_SECONDS = 7 * 24 * 60 * 60` (7 days) — Records older than this with `delivery_state='delivered'` are deleted.
- `_MAX_DURABLE_PENDING = 1000` — Maximum number of terminal records with `delivery_state='pending'`. Excess pending records are pruned oldest-first.

`_prune_durable_records()` runs after each dispatch. It first deletes delivered records past the retention window, then trims terminal records (those not in `running` or `finalizing`) to `_MAX_RETAINED_COMPLETED` (50), preferring delivered records for deletion (they sort first in the `ORDER BY CASE delivery_state WHEN 'delivered' THEN 0 ELSE 1 END` clause). Finally, if pending terminal records exceed `_MAX_DURABLE_PENDING`, the oldest are removed.


## 3.3 Asynchronous Process Spawning

For work that must outlive the current turn, Hermes can spawn a completely new OS process running `hermes chat -q` or an interactive `hermes` session inside tmux.

### One-Shot Mode

Fire-and-forget for tasks that run unattended:

```python
terminal(
    command="hermes chat -q 'Set up CI/CD for ~/myapp'",
    timeout=300
)
```

A one-shot run (`hermes chat -q`, `--oneshot`, `hermes -z`) carries a smaller self-improvement footprint (`agent/oneshot_footprint.py`): `skill_manage` is hidden, the skills-prompt "record it / patch it" coaching is dropped, and delegation is capped at `delegation.oneshot_max_children` (default 2) — each child would otherwise re-pay a cold system prompt and re-explore the repo.

### Background Mode

Runs silently with notification on completion:

```python
terminal(
    command="hermes chat -q 'Monitor trading bot'",
    background=True,
    notify_on_complete=True
)
```

### Interactive PTY Mode (tmux)

For tasks requiring interactive prompting or long-running TUI sessions:

```python
# Start
terminal(command="tmux new-session -d -s agent1 -x 120 -y 40 'hermes'")

# Send work
terminal(command="tmux send-keys -t agent1 'Build auth service' Enter")

# Read output
terminal(command="tmux capture-pane -t agent1 -p")

# Clean up
terminal(command="tmux send-keys -t agent1 '/exit' Enter")
terminal(command="tmux kill-session -t agent1")
```

### Multi-Agent Coordination

Spawning is used for parallel backend/frontend development, where two agents work on the same codebase but in different tmux sessions, with a human or a coordinator agent relaying context between them.

## 3.4 Worktree Mode (`-w`)

When spawning agents that edit code, use the `-w` / `--worktree` flag:

```bash
hermes -w
```

This creates an isolated git worktree under `<repo>/.worktrees/hermes-<id>` (the directory is auto-gitignored) on a dedicated branch `hermes/<name>`, preventing conflicts when two agents modify the same repository while sharing the same `.git` object database. Details that matter in practice:

- **Base ref** — by default the worktree branches from the freshly *fetched* remote tip so the session starts current with the project; `worktree_sync: false` reverts to local HEAD. `.worktreeinclude`-listed untracked paths (env files, local config) are copied into the new tree.
- **Locking** — the worktree is locked (`git worktree lock --reason hermes pid=...`) so other processes and the pruner do not reap a live session.
- **Reclamation** — a pruner classifies candidates and deletes only on proof: stale worktrees (24 h), worktrees whose PR is merged on GitHub, pushed open-PR lanes (their disk is reclaimed but the branch survives), and orphaned `hermes/*` branches. Anything with unpushed commits or a dirty tree is preserved.

## 3.5 Why Spawn?

Hermes spawns agents for three architectural reasons:

1. **Capacity** — Each child gets its own fresh iteration budget and its own context window; work that would flood the parent's context (or a single long loop) is offloaded wholesale. Turn caps are opt-in (`agent.max_turns`, default unlimited — finite caps caused silent mid-task truncation), so capacity scaling comes from context isolation and delegation budgets, not a default ceiling.
2. **Parallelism** — Some tasks are naturally parallel (research multiple topics, audit multiple files). Delegation runs subagents concurrently, bounded by `delegation.max_concurrent_children`.
3. **Durability** — The parent conversation may end (user disconnects, gateway restarts). A spawned process or a kanban worker survives because it is independent; durable delegation results survive via the SQLite ledger even when the parent does not.

## 3.6 Kanban: The Durable Work Queue

For durable multi-agent coordination, Hermes provides `kanban` — a SQLite-backed work queue with dispatcher-driven worker spawning (opt in per platform via `hermes tools enable kanban --platform X`, or by enabling the `kanban` toolset for a profile).

### Architecture

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│   Board     │◄────►│  Dispatcher  │◄────►│   Workers    │
│  (SQLite)   │      │  (gateway)   │      │  (spawned)   │
└─────────────┘      └──────────────┘      └──────────────┘
```

1. Tasks are created on the board with `kanban_create`.
2. The dispatcher (running inside the gateway, enumerating boards every tick) reclaims stale claims, promotes ready tasks, and can auto-decompose triage tasks into workgraphs before fanning out.
3. Workers are spawned as fire-and-forget `hermes -p <profile> chat -q` subprocesses (`_default_spawn` in `hermes_cli/kanban_db_dispatch.py`) with the task environment pinned in their process env: `HERMES_KANBAN_TASK`, `HERMES_KANBAN_DB`, `HERMES_KANBAN_BOARD`, `HERMES_KANBAN_WORKSPACES_ROOT`, `HERMES_KANBAN_RUN_ID`/`CLAIM_LOCK`, `TERMINAL_CWD` (anchored on the task workspace), and `HERMES_SESSION_SOURCE=kanban`. Secrets are scrubbed to the assigned profile's scope, session-routing vars from the dispatching turn are stripped, and the marker `HERMES_DELEGATED_CHILD_CONTEXT` is explicitly removed — the spawn is the grant boundary.
4. Workers see a focused `kanban_*` toolset to update task status. Lifecycle tools (`kanban_complete`, `kanban_block`, `kanban_schedule`, `kanban_request_review`, `kanban_request_changes`) require proof of run ownership (the worker's `HERMES_KANBAN_RUN_ID` must resolve, or the mutation is refused), and a worker may only mutate its own `HERMES_KANBAN_TASK` — a prompt-injected `task_id` for a sibling task is rejected. Board-routing tools (`kanban_list`, `kanban_unblock`) are hidden from task workers.
5. A turn-end **stop guard** (`agent/kanban_stop.py`) nudges workers that narrate a next step without calling a terminal board tool — some models would otherwise exit "cleanly" (rc=0) and trip the dispatcher's protocol-violation breaker. The nudge fires only for the dispatcher-owned worker (in-process `delegate_task` children and cron turns inherit the env vars but own no board task) and after at most 2 attempts.
6. A **circuit breaker** auto-blocks cards that keep failing: `kanban.failure_limit` (default 2) consecutive crashes trip it, terminal provider errors (revoked credential, missing model — a retry cannot heal them) trip immediately and stickily, and ≥3 identical crash fingerprints in one tick (systemic failure) trip at once.
7. Worker sessions are titled from their card so session browsers can follow them; the module-form worker argv carries the install root on `PYTHONPATH`, and spawn failures on the module path are retried. Workers on systemd-managed gateways use restart-safe argv so a service restart does not kill a worker mid-handoff.
8. A first-class **review lifecycle** hands finished cards over: `kanban_request_review` moves work into a review column (optionally naming a reviewer profile, with a judge gate that rejects premature handoffs), and `kanban_request_changes` returns it to `ready` for the builder.

Workers are fully isolated: they get `HERMES_KANBAN_DB` / `HERMES_KANBAN_BOARD` / `HERMES_KANBAN_WORKSPACES_ROOT` pinned in their environment, and the board is the hard boundary for context sharing — board identity follows the bound profile override, not the launch environment, and an in-process `delegate_task` child running inside a worker is fenced: its subprocess env has the kanban task keys scrubbed down to the delegated lineage's board root (`agent/delegation_context.py`), so a delegated grandchild can never act on (or write to) the parent worker's card.

## 3.7 Spawning Diagram

```
┌─────────────────┐
│  Parent Agent   │
│  (main loop)    │
└─────────────────┘
         │
    ┌────┴────┬─────────────┬──────────────┐
    │         │             │              │
    ▼         ▼             ▼              ▼
┌────────┐ ┌────────┐ ┌────────────┐ ┌──────────────┐
│delegate│ │ tmux   │ │  cronjob   │ │    kanban    │
│_task   │ │ spawn  │ │  scheduler │ │  dispatcher  │
└────────┘ └────────┘ └────────────┘ └──────────────┘
    │         │             │              │
    ▼         ▼             ▼              ▼
┌────────┐ ┌────────┐ ┌────────────┐ ┌──────────────┐
│ Leaf/  │ │ Full   │ │  Script or │ │   Board      │
│ Orch.  │ │ Hermes │ │  LLM agent │ │   worker     │
│ child  │ │ (async)│ │  (timed)   │ │ (chat -q)    │
└────────┘ └────────┘ └────────────┘ └──────────────┘
```
