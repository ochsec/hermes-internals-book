# Agent Spawning & Delegation

Hermes can create additional agent instances to work in parallel. There are two distinct mechanisms for this: **synchronous delegation** (`delegate_task`) and **asynchronous process spawning** (running a new `hermes` process). This chapter explains both, their isolation boundaries, and when each is used.

## 3.1 Two Mechanisms Compared

| Dimension | `delegate_task` | Spawning `hermes` process |
|-----------|----------------|----------------------------|
| **Isolation** | Separate conversation, shared process | Fully independent OS process |
| **Duration** | Minutes (bounded by parent loop) | Hours or days |
| **Tool access** | Subset of parent's tools | Full tool access |
| **Interactivity** | No — batch only | Yes — supports PTY mode |
| **Parent blocking** | Parent waits for summary | Parent continues immediately |
| **Use case** | Quick parallel subtasks | Long autonomous missions |

## 3.2 Synchronous Delegation (`delegate_task`)

Implemented in the `delegation` toolset (exposed as `delegate_task`), this is the lighter-weight of the two mechanisms.

### Invocation

```python
# Single task
delegate_task(goal="Research GRPO papers", context="Focus on 2024+")

# Batch (parallel)
delegate_task(tasks=[
    {"goal": "Backend API design", "context": "FastAPI"},
    {"goal": "Frontend scaffolding", "context": "React"},
])
```

### Lifecycle

1. **Parent sends goal + context** — The parent Hermes instance calls `delegate_task`.
2. **Child agent instantiated** — A new `AIAgent` is created with:
   - Isolated conversation history (empty except for the system prompt and the goal)
   - The parent's full toolset — subagents **always inherit the parent's toolsets** automatically. The model-facing `toolsets` argument was removed; the child gets the parent's tool definitions without the model needing to specify them. MCP toolsets are inherited by default (`delegation.inherit_mcp_toolsets`, default true).
   - Its own `IterationBudget` capped at `delegation.max_iterations` (default 50)
3. **Child runs independently** — The child executes the loop until it returns text or hits its iteration budget.
4. **Summary returned** — The child returns a condensed summary (not the full conversation). The parent receives this summary as a tool result.
5. **Parent resumes** — The parent sees the summary in its next loop iteration and continues.

When delegations are dispatched asynchronously (via the daemon pool in `tools/async_delegation.py`), the lifecycle extends with a **`finalizing`** transitional state. After the child agent completes but before the result is persisted to SQLite and published to the completion queue, the record's status flips to `finalizing` rather than directly to `completed` or `failed`. This prevents process shutdown from killing a daemon worker in the narrow gap after the status flips but before the SQLite commit lands. `active_count()` counts both `running` and `finalizing` records, so the pool does not reclaim capacity prematurely. Only after the completion event is persisted and enqueued does the record transition to its terminal state (`completed`, `failed`, `unknown`) and become eligible for in-memory pruning.

### Security: No Model-Controlled ACP Transport

The delegate ACP (Agent Context Protocol) transport was removed from model control. The model cannot choose to spawn subagents via ACP — delegation is always handled through the internal `delegate_task` tool path, preventing a compromised model from exfiltrating context through an ACP channel.

### Isolation Guarantees

- **Conversation isolation** — The child cannot see the parent's history, and vice versa.
- **Terminal isolation** — The child gets its own terminal session (separate working directory and shell state).
- **Memory isolation** — By default, `skip_memory=True` prevents the child from reading or writing persistent memory. The child still gets the user's profile injected in its system prompt.
- **Process isolation** — The child runs in the same Python process but in a separate thread. If the parent is interrupted, the child is cancelled.

### Roles: Leaf vs Orchestrator

- **Leaf** (default) — A focused worker that cannot re-delegate. Suitable for single concrete tasks.
- **Orchestrator** — Can call `delegate_task` itself to spawn further workers, bounded by `delegation.max_spawn_depth`. Useful for hierarchical task decomposition. (Note: orchestrator spawning is currently disabled by default — `max_spawn_depth=1` — and can be enabled via config.)

### Configuration

```yaml
delegation:
  model: "gpt-4o-mini"        # Model for subagents (can differ from parent)
  provider: "openai"
  base_url: ""
  api_key: ""
  max_iterations: 50
  max_concurrent_children: 3    # Parallel batch limit
  reasoning_effort: "medium"
```

### Child Agent Provider Overrides

`_build_child_agent` accepts `override_request_overrides` and `override_max_tokens` parameters that control how the child agent's provider configuration diverges from the parent's:

- **`override_request_overrides`** — When a provider is overridden, the child receives a fresh `request_overrides` dict (from `_resolve_delegation_credentials`) rather than inheriting the parent's. When no override is in effect, the child inherits the parent's `request_overrides` as-is. This ensures that provider-specific request parameters (extra headers, body fields, etc.) are correct for the delegated provider, not leaked from the parent's.
- **`override_max_tokens`** — Sets the child's `max_tokens` independently of the parent. Falls back to the parent's `max_tokens` when not specified.

The child also inherits two provider-gating attributes from the parent: `provider_require_parameters` (default `False`) and `provider_data_collection` (default `""`). When a provider override is in effect, both are reset to their defaults (`False` / `""`) since the overridden provider's gating policy should apply independently.

### 3.2.1 Durable Delegation Persistence

Async delegations dispatched through the daemon pool are persisted to an `async_delegations` table in `state.db` (the same SQLite database used by kanban and session state). This durability layer ensures that completions survive process crashes and that competing gateway processes do not double-deliver results.

#### Schema

The `async_delegations` table stores the full lifecycle of each delegation:

| Column | Purpose |
|--------|---------|
| `delegation_id` | Primary key (UUID) |
| `origin_session` | Session key that originated the delegation |
| `origin_ui_session_id` | UI session identifier |
| `parent_session_id` | Parent's session ID for context |
| `state` | `running` → `finalizing` → `completed` / `failed` / `unknown` |
| `dispatched_at` | Epoch timestamp when the delegation was dispatched |
| `completed_at` | Epoch timestamp when the child agent finished |
| `updated_at` | Last modification time (used for retention pruning) |
| `event_json` | Serialized completion event (goal, summary, error, status) |
| `result_json` | Serialized result dict |
| `delivery_state` | `pending` or `delivered` |
| `delivery_attempts` | Count of delivery attempts (incremented on each claim) |
| `delivered_at` | Timestamp of successful delivery |
| `owner_pid` | PID of the process that dispatched the delegation |
| `owner_started_at` | Process start time (for crash detection) |
| `task_json` | Serialized task payload (goal, context, model, role, etc.) |
| `delivery_claim` | Current delivery claim token |
| `delivery_claimed_at` | Timestamp when the current claim was made |

The database connection uses WAL journal mode (`PRAGMA journal_mode=WAL`) for concurrent read/write compatibility. A module-level `_DB_LOCK` serializes writes within the process. The schema is created lazily via `CREATE TABLE IF NOT EXISTS`, with `ALTER TABLE ADD COLUMN` migrations for columns added after initial deployment.

#### Dispatch Persistence

When a delegation is dispatched (`_persist_dispatch`), a row is inserted with `state='running'`, `delivery_state='pending'`, the owning process's PID and start time, and a serialized task payload. If the thread pool rejects the submission, the durable row is deleted (`_delete_durable_delegation`) so it does not linger as an orphan.

#### Completion Persistence

When the child agent finishes, `_persist_completion` updates the row with the completion event and result JSON, sets `completed_at`, and resets `delivery_state` to `pending` so the completion can be delivered to the origin session. This happens during the `finalizing` window — the in-memory record stays in `finalizing` status until after this SQLite commit, preventing a race where process shutdown kills the worker between the status flip and the commit.

#### Abandoned Delegation Recovery

`recover_abandoned_delegations()` runs at process startup and whenever completions are restored. It scans for records in `running` or `finalizing` state whose owning process has disappeared — determined by checking whether `owner_pid` still exists and whether its current start time matches `owner_started_at`. Records whose owner is gone are marked `state='unknown'` with a synthetic completion event (`status: "unknown"`, `error: "Delegation owner exited before recording a terminal result; outcome unknown."`) and `delivery_state='pending'`, so the parent session is at least notified that the delegation's outcome is indeterminate rather than silently lost.

#### Delivery Claiming

When multiple gateway processes may compete to deliver the same completion, a claim-based protocol prevents duplicate delivery:

- `claim_completion_delivery(delegation_id, claim_id)` — Atomically claims a pending completion. A claim succeeds only if `delivery_state='pending'` and no active claim exists (or the previous claim is older than 5 minutes, treating it as stale). Returns `True` if the claim was acquired.
- `complete_completion_delivery(delegation_id, claim_id)` — Marks the completion as `delivered` and clears the claim. Only the consumer holding the matching `claim_id` can finalize delivery.
- `release_completion_delivery(delegation_id, claim_id)` — Releases a failed delivery claim so another consumer may retry. Clears the claim if it matches and the delivery is still pending.
- `mark_completion_delivered(delegation_id)` — Simpler unconditional acknowledge for paths that do not need claim-based competition.

The event consumer path wraps these: `claim_event_delivery(evt, consumer)` generates a claim ID (`{consumer}:{pid}:{uuid}`) and calls `claim_completion_delivery`, returning the claim ID on success or `None` if the claim could not be acquired (or `""` for non-durable events that need no token). `complete_event_delivery(evt, claim_id)` and `release_event_delivery(evt, claim_id)` delegate to the corresponding SQLite operations. Legacy events created before durable dispatch are handled gracefully — if no row exists for the `delegation_id`, the claim is treated as successful so the event proceeds without blocking.

#### Undelivered Completion Restoration

`restore_undelivered_completions(target_queue)` is called at process startup to re-enqueue durable pending completions as fresh turns. It first calls `recover_abandoned_delegations()` to classify orphaned records, then selects all records where `state != 'running'` and `delivery_state='pending'` with a non-null `event_json`, ordered by `completed_at` then `delegation_id`, and puts each event onto the target queue. This ensures that completions produced by a previous process incarnation — but never delivered because the process crashed — are replayed after restart.

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

When spawning agents that edit code, use the `-w` flag:

```bash
hermes -w
```

This creates an isolated git worktree, preventing conflicts when two agents modify the same repository. Each spawned agent gets its own branch and working directory while sharing the same `.git` object database.

## 3.5 Why Spawn?

Hermes spawns agents for three architectural reasons:

1. **Capacity** — A single agent is bounded by `max_turns` (default 90). Complex multi-step tasks burn through this budget quickly. Spawning allows horizontal scaling of reasoning steps.
2. **Parallelism** — Some tasks are naturally parallel (research multiple topics, audit multiple files). Delegation runs subagents concurrently.
3. **Durability** — The parent conversation may end (user disconnects, gateway restarts). A spawned process survives because it's independent.

## 3.6 Kanban: The Durable Work Queue

For durable multi-agent coordination, Hermes provides `kanban` — a SQLite-backed work queue with dispatcher-driven worker spawning.

### Architecture

```
┌─────────────┐      ┌──────────────┐      ┌──────────────┐
│   Board     │◄────►│  Dispatcher  │◄────►│   Workers    │
│  (SQLite)   │      │  (gateway)   │      │  (spawned)   │
└─────────────┘      └──────────────┘      └──────────────┘
```

1. Tasks are created on the board with `kanban_create`.
2. The dispatcher (running inside the gateway) reclaims stale claims and promotes ready tasks.
3. Workers are spawned with `HERMES_KANBAN_TASK` set in their environment.
4. Workers see a focused `kanban_*` toolset to update task status.
5. After ~5 consecutive spawn failures on a task, the dispatcher auto-blocks it (circuit breaker).

Workers are fully isolated: they get `HERMES_KANBAN_BOARD` pinned in their environment, and the board ID is the hard boundary for context sharing.

## 3.7 Spawning Diagram

```
┌─────────────────┐
│  Parent Agent   │
│  (main loop)    │
└─────────────────┘
         │
    ┌────┴────┬─────────────┐
    │         │             │
    ▼         ▼             ▼
┌────────┐ ┌────────┐ ┌────────────┐
│delegate│ │ tmux   │ │  cronjob   │
│_task   │ │ spawn  │ │  scheduler │
└────────┘ └────────┘ └────────────┘
    │         │             │
    ▼         ▼             ▼
┌────────┐ ┌────────┐ ┌────────────┐
│ Leaf   │ │ Full   │ │  Script or │
│ Agent  │ │ Hermes │ │  LLM agent │
│ (sync) │ │ (async)│ │  (timed)   │
└────────┘ └────────┘ └────────────┘
```
