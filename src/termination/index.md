# Termination Conditions

Hermes must decide when to stop — within a single turn, within a session, and when to abandon a failing path entirely. This chapter describes every condition that causes the agent loop to terminate.

## 5.1 Normal Turn Termination

A turn ends successfully when the LLM returns a plain text response instead of tool calls. This is the happy-path termination condition.

```python
# Inside conversation_loop.py — one phase machine per loop iteration
_v = _run_phase(
    run_tool_round if s.assistant_message.tool_calls else finish_text_response, agent, s
)
```

The no-tool-call branch (`agent/turn_final_response.py` → `finish_text_response`) is not a rubber stamp: the candidate answer passes through empty-response recovery, length-continuation, dropped-tool-call re-prompting, and the stop gates (§5.9) before the final message is appended and flushed. Only then does the loop return it.

After the loop exits, `agent/turn_finalizer.py` (`finalize_turn`) performs post-turn bookkeeping: it stamps the liveness activity clock ("turn end logged"), records the `turn_exit_reason` in `agent.log`, and persists the transcript before the next user message arrives.

## 5.2 Iteration Budget Exhaustion

Each agent instance carries an `IterationBudget` (`agent/iteration_budget.py`). The parent's cap is `agent.max_turns` in config — **null (unlimited) by default**, because a finite cap caused silent mid-task truncation; under the hood `AIAgent` defaults to `sys.maxsize`. Subagents get `delegation.max_iterations`, default **250** (raised from 50 by a config migration because 50 truncated substantial delegated work).

The budget is consumed once per loop iteration; the loop condition is

```python
while (s.api_call_count < agent.max_iterations
       and agent.iteration_budget.remaining > 0) or agent._budget_grace_call:
```

If the budget is exhausted before text is returned, `handle_max_iterations` (`agent/chat_completion_helpers.py`) prints `⚠️ Reached maximum iterations (N). Requesting summary...` and injects a fixed user-role nudge (`MAX_ITERATIONS_SUMMARY_REQUEST` in `agent/context_compressor.py`: "…Please provide a final response summarizing what you've found and accomplished so far, without calling any more tools."). The summary call gets exactly one retry on an empty response; if it succeeds, the summary becomes the final text. If the summary itself fails, the fallback copy is:

```text
"I ran out of steps for this turn ({limit} tool calls) before finishing, and couldn't
produce a summary. Send `continue` to keep going, or raise `max_iterations` in your config."
```

If the summary call is cancelled, the unanswered nudge is dropped and the turn ends as **interrupted** so the pending message is requeued. A turn that exits on budget exhaustion gets the typed exit reason `max_iterations_reached(N/M)` from the finalizer.

The `execute_code` tool is special: when it is the only tool called in an iteration, the iteration is **refunded** via `budget.refund()` because the Python execution loop is conceptually part of a single tool call, not a separate turn. A surviving final response may also earn exactly one `_budget_grace_call` — the loop runs one extra iteration to deliver the pending fallback, then clears the grace flag.

### Budget Warnings

There is no fixed "low budget" threshold. The warning is **ratio-based and opt-in**: `agent.budget_warning_ratio` (default null = off) fires a one-shot checkpoint notice — `[SYSTEM NOTICE — iteration budget checkpoint] You have used {used} of {maximum} iterations…` — appended to the newest tool result once `used ≥ ratio × max_total`. Kanban workers get an implicit ratio of 0.9 plus a completion-advice suffix. Cancellation suppresses the warning: cancelled results must be persisted, but must not urge more work.

### The Wall-Clock Run Budget

Separate from the iteration count, `agent.run_budget_seconds` (also `hermes chat --run-budget N`) bounds a turn in **wall-clock time**. At 80 % elapsed, a one-shot wrap-up notice is injected into the newest tool result ("Run time budget nearly exhausted. Stop new discovery/verification work now…"), and implicit provider stale-stream timeouts are capped to the remaining budget (`cap_to_run_budget`) so a watched stream cannot outlive the turn.

## 5.3 Guardrail Halts

The tool-loop guardrails (`agent/tool_guardrails.py`) are configured via `tool_loop_guardrails` in config. Hard stops are **opt-in on interactive platforms** (`hard_stop_enabled`, default false) and **default-on for unattended ones** (`non_interactive_hard_stop_enabled`, default true — gateway, cron, and other non-attended platforms auto-enable halts).

- **Exact failure halt** — Same tool + same args failed 5+ times (`exact_failure_block_after`; warning from the 2nd).
- **Same tool failure halt** — Same tool failed 8+ times this turn (`same_tool_failure_halt_after`; warning from the 3rd).
- **Idempotent no-progress halt** — A read-only tool returned the same result 5+ times (`no_progress_block_after`; warning from the 2nd).

Two tool sets shape these counts: *failure-tolerant* tools (`terminal`, `execute_code`, `web_extract`, …) treat a "failure" as normal work output, so only an exact-args replay or identical-result streak can halt them; a success on a *progress-reset* tool (`write_file`, `patch`, `terminal`, …) marks progress for every failing signature, so the next retry is a new experiment, not a replay.

Two stall guards catch subtler loops:

- **Identical-call streak** — the 3rd consecutive identical (tool, args, result) call fires a notice; with hard stops enabled it halts at `no_progress_block_after` laps. Repeated tools like `process_manage` are exempt.
- **Cycle detection** — repeating multi-call batches (A,B,A,B,… with identical args *and* results) defeat the streak counter, so the longest repeating cycle period (up to 4) is detected and reuses the same thresholds.

Separately, `tool_loop_guardrails.loop_caps` are hard per-turn ceilings that **fire regardless of `hard_stop_enabled`**: `max_web_searches` (default 50) and `max_subagents` (default 50) count total calls, not repeats; `0` disables a cap.

When a halt triggers, `ToolGuardrailDecision.action == "halt"` is recorded as the turn's halt decision. The tool round sees it, sets `turn_exit_reason = "guardrail_halt"`, appends the controlled halt response to the conversation (and streams it so SSE/TUI clients see the explanation rather than a crash), and breaks the loop. The finalizer attaches the decision as `result["guardrail"]` metadata.

## 5.4 API Error Classification and Failover

Every provider failure is classified by `agent/error_classifier.py` into a `FailoverReason`, and the verdict — not the raw HTTP status — decides whether the turn retries, rotates credentials, compresses, or aborts. The dominant reasons:

- **`auth` / `auth_permanent`** — transient 401/403 refreshes or rotates; auth that fails after refresh aborts the turn.
- **`billing`** — confirmed credit/quota exhaustion rotates the credential immediately (never retried in place).
- **`rate_limit` / `upstream_rate_limit` / `overloaded`** — backoff (honoring `Retry-After`), then fallback; the key is healthy.
- **`model_not_found`** — the model no longer exists on the provider; the fallback chain switches models.
- **`ssl_cert_verification`** — deterministic TLS chain failure; fails fast with guidance instead of retrying.
- **`content_policy_blocked`** — the provider's safety filter rejected this prompt; a per-request deterministic refusal, never retried unchanged.
- **`context_overflow` / `payload_too_large`** — compression, not failover (§5.9's overflow paths).

There is no single "fatal error" enum: a turn ends immediately when a verdict is marked non-retryable (`auth_permanent`, `ssl_cert_verification`, `content_policy_blocked`) or when the fallback chain itself is exhausted (`_fallback_chain_exhausted`), which ends the attempt cycle with the classified failure surfaced.

One case ends a watched turn **at once**: on the Nous free tier with an attended session, a rate-limit cooldown longer than a person would sit through (over the 60 s live-retry cap) triggers `free_tier_cooldown_ends_turn` — the attempt cycle ends immediately with fallback activation or the reset time and the sign-in/model doors, because a 600 s retry previously met a second 429 while its 1069 s window was still running. Unattended runs (cron, messaging, `hermes chat -q`) always wait out the `Retry-After`.

## 5.5 User Interrupt

The user can interrupt a running turn:

- **CLI**: Ctrl-C sets the interrupt flag.
- **Gateway**: Platform-specific cancel (e.g., Telegram stop button) — recorded as one of the typed interrupt reasons (`stop`, `reset`, `timeout`, `sse_disconnect`, `evicted`, `gateway_shutdown`, `gateway_restart`).
- **ACP**: IDE stop command.

The interrupt propagates via `tools.interrupt.set_interrupt()` (thread-targeted, through `agent/interrupt_control.py`). Between loop iterations and inside the interruptible API call, the flag is checked. If set during an API wait, the turn emits:

```text
"Operation interrupted: waiting for model response ({t}s elapsed)."
```

…a stable prefix ACP/TUI clients match on to treat the text as cancellation metadata, not assistant prose. Running tool processes (terminal commands, browser sessions) are also signaled to terminate, and an iteration-summary call interrupted mid-flight ends the turn as interrupted so the pending message is requeued.

## 5.6 Session-Level Termination

A session ends when:

- **User types `/quit` or `/exit`** — The CLI exits (`--delete` also removes session history); gateway sessions persist for the next message.
- **Gateway inactivity timeout** — `agent.gateway_timeout` (default 1800 s) fires only when the agent is completely idle — not while calling tools or receiving API responses.
- **`/new` or `/reset`** — A new session is created; the old one is archived in SQLite. Both interrupt a busy turn first (`interrupt_then_dispatch`).
- **`hermes pause` (ESTOP)** — A sentinel file at `$HERMES_HOME/ESTOP` (`agent/estop.py`) makes the cron scheduler, kanban dispatcher, and new gateway turns skip work until `hermes resume`. In-flight work is never killed; a corrupt or empty sentinel still counts as engaged.
- **Disk pressure** — `hermes sessions prune` hides old sessions via `prune_sessions`.

Sessions are stored in SQLite through the `hermes_state_*` module family (`hermes_state.py` and its split-out schema/messages/sessions/compression layers), which manages session metadata, message history (role, content, timestamps, tool calls and results), and compression checkpoints.

## 5.7 Cron Job Termination

Cron jobs (`cronjob` tool, `hermes cron`) have their own termination conditions:

- **Pre-run script timeout** — `_DEFAULT_SCRIPT_TIMEOUT = 3600` (1 hour) in `cron/scheduler.py`, configurable via `cron.script_timeout_seconds`.
- `.tick.lock` file — Prevents duplicate execution across processes.
- **Empty stdout = silent** — If a `no_agent=True` script produces no output, nothing is delivered.
- **Non-zero exit / timeout** — Sends an error alert.
- **Interrupted jobs** — Pre-kill output is suppressed; only post-completion output is delivered to avoid partial/garbled messages.
- **Host sleep excluded** — Job inactivity is measured against wall-clock activity stamps, not raw elapsed time, so a machine that slept does not see its jobs misjudged as stalled.
- **Interpreter shutdown** — Delivery and dispatch are skipped when the Python interpreter is shutting down to avoid errors during teardown.
- **Gateway shutdown drain** — In-flight cron jobs are drained before the gateway kills workers, making the shutdown visible to cron work (a drain observer can even name which job is holding the gateway open). API-server shutdown also drains in-flight runs.
- **Live one-shot preservation** — When the running-set check fails (e.g., a stale process lock), live one-shots are kept rather than dropped. This prevents a momentary race from silently discarding a scheduled job.
- **SessionDB timeout from config** — The `SessionDB` init timeout is resolved from `config.yaml`, and the init is bounded so a hang can't wedge cron forever; a SessionDB whose constructor finishes after the timeout is closed up by a done-callback.
- **Security** — The scheduler backstop fails closed when the validator errors, and `base_url` overrides that could exfiltrate provider credentials are blocked.
- **TERMINAL_CWD isolation** — Each job gets an isolated working directory, and the `TERMINAL_CWD` lock is released even when the job body raises an exception.

## 5.8 Subagent Termination

Delegated subagents terminate when:

- They return text (task complete).
- They hit their iteration budget (`delegation.max_iterations`, default 250).
- They exceed the inactivity cap (`delegation.child_timeout_seconds`, default 0 = off) — the cap measures time with **no progress**, so a child waiting on a long completion keeps its window alive; only a frozen one is caught.
- The parent is interrupted (cascading cancellation via `_attach_child`).
- They throw an unhandled exception (error summary returned to parent).

The child's `exit_reason` is reported honestly to the parent: `"completed"` vs `"max_iterations"` — a provider rejection or provider-side truncation is never relabeled as budget exhaustion (`truncated == (exit_reason == "max_iterations")`).

### Kanban Worker Termination

Kanban workers have additional termination paths:

- **Terminal-tool stop gate** — A worker that narrates the next step and stops with no tool call would exit clean (`rc=0`) and be booked as a protocol violation. Instead, the kanban stop gate (`agent/kanban_stop.py`, `agent/turn_stop_gates.py`) returns a bounded synthetic nudge — capped at 2 attempts — telling the worker to call a terminal board tool (`kanban_complete`, `kanban_block`, `kanban_schedule`, `kanban_request_review`, `kanban_request_changes`). The synthetic candidate answer is flagged `_kanban_stop_synthetic` so it never survives into the durable transcript.
- **Turn recovery with an attempt budget** — A failed worker turn can be re-entered in place (`agent/kanban_turn_recovery.py`): up to `HERMES_KANBAN_TURN_RECOVERY` attempts (default 3, clamped to [0, 10]). Quota walls and billing blocks (`rate_limit`, `billing`, `upstream_rate_limit`) are **never** recovered in place — they need the dispatcher's cooldown/breaker accounting.
- **`max_iterations_reached` as a terminal settlement** — The finalizer writes `max_iterations_reached(N/M)` right after recording `outcome="timed_out"` for an out-of-budget worker, and that exit prefix closes the run and releases its claim instead of attempting recovery.
- **Cron fire reservation** — During gateway drains, cron fire work is reserved so a shutdown doesn't silently drop a scheduled kanban task.

## 5.9 Degenerate Response Recovery

Several mechanisms can *continue* a turn the model tried to end, or abort it in a controlled way:

- **Empty / thinking-only ladder** (`agent/turn_empty_response.py`) — partial-stream recovery → reuse prior turn content (housekeeping tools only) → one post-tool nudge → thinking-only prefill continuation (×2) → budgeted empty-response retries with a deterministic-empty short-circuit → fallback provider → terminal `(empty)` sentinel. Ladder order is load-bearing.
- **Length-stop continuation** (`agent/turn_truncation.py`) — `finish_reason == "length"` responses are continued from the truncated tail: fragments are joined, commentary cut at a tool-call boundary is preserved, and a length stop carrying only side-channel reasoning is continued rather than dropped. A response truncated by the provider/router is never misreported as an output-length limit; a true window-filled stop ends the turn with `/compress` or `/new` guidance.
- **Repetition guard** — A stop-path answer flagged as runaway repetition (`is_runaway_repetition`, gated on shape and length) is discarded instead of delivered.
- **Stop gates** (`agent/turn_stop_gates.py`) — Three gates may turn a text answer into an interim row plus a synthetic nudge and continue the turn: verify-on-stop (the model tries to finish right after editing code without fresh evidence), the `pre_verify` plugin hook, and the kanban terminal-tool guard (§5.8). Each keeps the candidate answer as the budget-exhaustion fallback (`pending_verification_response`).
- **Overflow recovery** (`agent/turn_overflow.py`) — 413/context-length errors compress and restart, defer softly, or end the turn with a typed result.
- **Outer-loop error cap** — Exceptions that escape the inner retry/fallback machinery end the turn after 8 occurrences (`_MAX_OUTER_LOOP_ERRORS`), so one deterministic local bug cannot loop forever. A session whose transcript cannot be persisted halts immediately (`session_persistence_failed`); a compression window that cannot shrink ends the turn with `/new` guidance after one idle budget, not the full ceiling.

## 5.10 Watchdogs and Turn Liveness

Several watchdogs exist because a turn can wedge **without any error**:

- **Turn liveness watchdog** (`agent/turn_liveness.py`) — Samples the agent's activity clock (`_touch_activity`, generation-stamped under a shared lock) and hard-aborts a turn that stalled for `agent.turn_liveness.timeout_s` (default 600 s). The abort decision is bound to the observed `(generation, timestamp)` and revalidated at commit, so a turn that resumed in between is never cancelled. The finalizer stamps the clock again at loop end, so a wedge in the post-loop tail is measured — and aborted — from when the loop actually finished.
- **Gateway turn-inactivity watchdog** — `_watch_gateway_turn_inactivity` in `gateway/run.py` runs as a plain thread (immune to a starved asyncio loop) and abandons a timed-out turn by task id, using the agent's activity snapshot as the liveness clock and falling back to elapsed worker time before the clock is usable.
- **Auxiliary stream watchdog** (`agent/auxiliary_stream_watchdog.py`) — A consumer blocked inside the SDK's chunk iterator only notices a silent stream when the full request timeout expires, so a `Timer` shuts down just that attempt's socket to wake the blocked read; it is re-armed by substantive chunks, never touches a pooled connection, and the finalizer dumps thread stacks if the loop thread appears blocked.
- **Metrics** — Per-process exit markers and turn watchdog firings feed `hermes.process.exit` so silent kills are observable.

## 5.11 Concurrency Tripwires

Overlapping turns on one session can interleave transcript writes, so `note_turn_start` (`agent/agent_runtime_helpers.py`) is a **tripwire, not a guard**: it detects and names the overlap, it does not prevent it.

- **Per-agent leg** — Warns when a turn starts while the previous turn of the same agent object has not finished its turn-end persist.
- **Cross-agent leg** — The gateway caches agents per routing key while the transcript is keyed by `session_id`, so two agent objects can run concurrent turns on one session unseen by per-agent state. A session-scoped in-flight registry catches the case where two routing keys map to one `session_id`.
- **Fork exemption** — Persist-disabled review forks (background review, side questions) share the parent's `session_id` but never write, so they neither register nor pop a session slot — symmetrically in `note_turn_start` and `note_turn_persisted`.

The tripwire under-reports rather than double-reports: on a real overlap the first persist clears the second slot. The warning names both turn ids so the dispatch route that bypassed the busy guard is findable in logs.

## 5.12 Summary Diagram

```
┌─────────────────┐
│   Loop Start    │
└─────────────────┘
         │
         ▼
┌─────────────────────────┐
│  LLM returns text?      │──YES──▶ Stop gates / recovery ladder
└─────────────────────────┘         │ accept ──▶ Turn ends normally
         │ NO                       └ continue ─▶ Next Iteration
         ▼
┌─────────────────────────┐
│  Tool calls returned?   │──NO──▶ Error / Interrupt path
└─────────────────────────┘
         │ YES
         ▼
┌─────────────────────────┐
│  Iteration budget > 0?  │──NO──▶ Summary nudge, grace call,
└─────────────────────────┘        max_iterations_reached(N/M)
         │ YES
         ▼
┌─────────────────────────┐
│  Guardrail halt flag?   │──YES──▶ Controlled halt (guardrail_halt)
└─────────────────────────┘
         │ NO
         ▼
┌─────────────────────────┐
│  Dispatch tools          │
│  (check interrupts,      │
│   loop caps, watchdogs)  │
└─────────────────────────┘
         │
         ▼
┌─────────────────────────┐
│  Inject results          │
└─────────────────────────┘
         │
         └──────────────────────┐
                              ▼
                    ┌─────────────────┐
                    │   Next Iteration │
                    └─────────────────┘
```
