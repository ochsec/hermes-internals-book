# Termination Conditions

Hermes must decide when to stop — within a single turn, within a session, and when to abandon a failing path entirely. This chapter describes every condition that causes the agent loop to terminate.

## 5.1 Normal Turn Termination

A turn ends successfully when the LLM returns a plain text response instead of tool calls. This is the happy-path termination condition.

```python
# Inside conversation_loop.py
if not response.tool_calls:
    return response.content  # Turn terminates here
```

After text is returned, `agent/turn_finalizer.py` performs post-turn bookkeeping before the next user message arrives.

## 5.2 Iteration Budget Exhaustion

Each agent instance carries an `IterationBudget` (`agent/iteration_budget.py`). The parent agent defaults to `max_iterations = 90`; subagents default to `delegation.max_iterations = 50`.

The budget is consumed once per loop iteration. If the budget is exhausted before text is returned, the loop aborts with a synthesized message:

```text
"I ran out of iterations ({used}/{max}). Let me summarize what I've accomplished so far..."
```

The `execute_code` tool is special: its iterations are **refunded** via `budget.refund()` because the Python execution loop is conceptually part of a single tool call, not a separate turn.

### Budget Calculation

```
total_available = min(config.max_turns, model_context_limit_estimate)
used = iteration_counter
remaining = total_available - used
```

If remaining ≤ 3, a low-budget warning is injected into the system prompt so the model knows to wrap up quickly.

## 5.3 Guardrail Halts

If `tool_loop_guardrails.hard_stop_enabled` is true, the guardrail can forcibly halt a turn:

- **Exact failure halt** — Same tool + same args failed 5+ times.
- **Same tool failure halt** — Same tool failed 8+ times this turn.
- **Idempotent no-progress halt** — A read-only tool returned the same result 5+ times.

When a halt triggers, `ToolGuardrailDecision.action == "halt"` is set. The loop sees this, appends the synthetic error result to the conversation, and returns a final summary text to the user.

## 5.4 API Fatal Errors

Certain LLM API errors are considered unrecoverable:

- **Auth revoked** — The user's API key or OAuth token is permanently invalid.
- **Model deprecated** — The model no longer exists on the provider.
- **Billing suspended** — The account has no usable payment method.
- **TLS/Certificate failure** — Cannot establish a secure connection to the provider.

These raise a `FailoverReason` that bypasses retry logic and aborts the turn immediately.

## 5.5 User Interrupt

The user can interrupt a running turn:

- **CLI**: Ctrl-C sets the interrupt flag.
- **Gateway**: Platform-specific cancel (e.g., Telegram stop button).
- **ACP**: IDE stop command.

The interrupt propagates via `tools.interrupt.set_interrupt()`. Between loop iterations, `conversation_loop.py` checks the flag. If set, it emits:

```text
"Operation interrupted: waiting for model response (elapsed: {t}s)"
```

…and exits cleanly. Running tool processes (terminal commands, browser sessions) are also signaled to terminate.

## 5.6 Session-Level Termination

A session ends when:

- **User types `/quit` or `/exit`** — The CLI exits; gateway sessions persist for the next message.
- **Gateway inactivity timeout** — If no message arrives within a configurable idle period, the session is marked dormant.
- **`/new` or `/reset`** — A new session is created; the old one is archived in SQLite.
- **Disk pressure** — If `~/.hermes/sessions/` grows too large, old sessions are pruned via `hermes sessions prune`.

Sessions are stored in `hermes_state.py` (SQLite), which manages:
- Session metadata (id, title, created_at, last_message_at)
- Message history (role, content, timestamp, tool_calls, tool_results)
- Compression checkpoints (summaries of pruned messages)

## 5.7 Cron Job Termination

Cron jobs (`cronjob` tool, `hermes cron`) have their own termination conditions:

- **Pre-run script timeout** — `_DEFAULT_SCRIPT_TIMEOUT = 3600` (1 hour), configurable via `cron.script_timeout_seconds`. Previously 120 seconds.
- `.tick.lock` file — Prevents duplicate execution across processes.
- **Empty stdout = silent** — If a `no_agent=True` script produces no output, nothing is delivered.
- **Non-zero exit / timeout** — Sends an error alert.
- **Interrupted jobs** — Pre-kill output is suppressed; only post-completion output is delivered to avoid partial/garbled messages.
- **Interpreter shutdown** — Delivery and dispatch are skipped when the Python interpreter is shutting down to avoid errors during teardown.
- **Gateway shutdown drain** — In-flight cron jobs are drained before the gateway kills workers, making the shutdown visible to cron work.
- **Security** — The scheduler backstop fails closed when the validator errors, and `base_url` overrides that could exfiltrate provider credentials are blocked.
- **TERMINAL_CWD isolation** — Each job gets an isolated working directory, and the `TERMINAL_CWD` lock is released even when the job body raises an exception.

## 5.8 Subagent Termination

Delegated subagents terminate when:

- They return text (task complete).
- They hit their iteration budget (`delegation.max_iterations`).
- The parent is interrupted (cascading cancellation).
- They throw an unhandled exception (error summary returned to parent).

## 5.9 Summary Diagram

```
┌─────────────────┐
│   Loop Start    │
└─────────────────┘
         │
         ▼
┌─────────────────────────┐
│  LLM returns text?      │──YES──▶ Turn ends normally
└─────────────────────────┘
         │ NO
         ▼
┌─────────────────────────┐
│  Tool calls returned?   │──NO──▶ Error / Interrupt path
└─────────────────────────┘
         │ YES
         ▼
┌─────────────────────────┐
│  Iteration budget > 0?  │──NO──▶ Budget exhausted, abort
└─────────────────────────┘
         │ YES
         ▼
┌─────────────────────────┐
│  Guardrail halt flag?   │──YES──▶ Halt with synthetic error
└─────────────────────────┘
         │ NO
         ▼
┌─────────────────────────┐
│  Dispatch tools          │
│  (check interrupts)     │
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
