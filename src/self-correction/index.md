# Self-Correction

Hermes does not assume the model or the tools are infallible. Every layer of the stack has error detection, classification, and recovery logic. This chapter describes how errors are caught, classified, retried, and ultimately resolved.

## 4.1 Error Sources

Errors can originate from three sources:

1. **LLM API errors** — Rate limits, context-length overflows, authentication failures, malformed requests.
2. **Tool execution errors** — Command failures, missing files, network timeouts, permission denied.
3. **System errors** — Disk full, memory pressure, process crashes, thread deadlocks.

Each source has its own classification and recovery path.

## 4.2 LLM API Error Classification

`agent/error_classifier.py::classify_api_error()` inspects the exception and HTTP response to produce a `FailoverReason` enum:

| Enum | Triggers | Recovery |
|------|----------|----------|
| `RATE_LIMIT` | HTTP 429, `Retry-After` header | Exponential backoff + credential pool rotation |
| `CONTEXT_LENGTH` | "context length", "too long" | Compress history or fall back to larger-context model |
| `AUTH_EXPIRED` | "expired", "invalid token" | Refresh OAuth or prompt re-auth |
| `QUOTA_EXHAUSTED` | "quota", "billing", "insufficient" | Rotate to fallback model |
| `MALFORMED_REQUEST` | "invalid json", "bad request" | Sanitize messages and retry |
| `SERVER_ERROR` | HTTP 500, 503, 502 | Jittered backoff (up to 5 retries) |
| `MODEL_UNAVAILABLE` | "model not found", "deprecated" | Switch to fallback model |
| `UNKNOWN` | Everything else | Log and abort turn |

### The `_pool_may_recover_from_rate_limit` Gate

Before burning retry budget on rate-limit retries, Hermes checks `agent/run_agent.py::_pool_may_recover_from_rate_limit()`. This gate determines whether credential pool rotation is worthwhile:

- If the pool has only one entry, rotation is pointless — the same quota will 429 again.
- If the provider is Google CloudCode / Gemini CLI, quotas are account-wide across all pool entries, so rotation won't help.
- In both cases, Hermes skips straight to the fallback model instead of waiting.

### Retry Strategy

Retries use `agent/retry_utils.py::jittered_backoff()`:

```python
def jittered_backoff(attempt: int, base: float = 1.0, max_delay: float = 60.0) -> float:
    return min(base * (2 ** attempt) + random.uniform(0, 1), max_delay)
```

The maximum number of retries is provider-dependent (typically 3–5), with a hard ceiling to prevent runaway loops.

## 4.3 Tool Execution Error Classification

Tool errors are classified at three levels:

### Level 1: Raw Exit Code (terminal tools)

`terminal` returns `{"output": "...", "exit_code": N}`. Nonzero exit codes are failures.

### Level 2: JSON Error Keys

Most tools return JSON. If the top-level object contains `"error"` or `"failed"`, it's a failure. The `classify_tool_failure()` function in `agent/tool_guardrails.py` does this consistently.

### Level 3: Result Pattern Matching

For tools that don't return structured JSON, the first 500 characters are inspected for `"Error"`, `"failed"`, `"error"`, `"not found"`, etc. This is a heuristic safety net.

### Special Case: File Mutation Success

`file_mutation_result_landed()` in `agent/tool_result_classification.py` returns True if a `write_file` or `patch` call successfully modified a file. Even if stderr contains warnings, the call is **not** a failure for guardrail purposes. This prevents the tool loop guardrail from punishing successful edits.

## 4.4 Self-Corception During a Turn

When a tool fails, the error is injected back into the conversation as a `role="tool"` message. The model sees the error on its next iteration and can choose a different approach. This is the primary self-correction mechanism: **the model observes failure and adapts**.

### Typical Correction Flow

```
Turn 1: Model calls terminal("git push origin main")
        → Result: {"output": "error: failed to push...", "exit_code": 1}

Turn 2: Model sees error, calls terminal("git pull origin main")
        → Result: success

Turn 3: Model calls terminal("git push origin main")
        → Result: success
```

The guardrails ensure this doesn't loop indefinitely:
- After 2 identical failures, a warning is injected.
- After 5 identical failures (if hard_stop_enabled), the call is blocked.
- After 8 same-tool failures, the turn is halted.

## 4.5 Compression Failure Recovery

`agent/context_compressor.py` and `agent/conversation_compression.py` can fail if the LLM refuses to summarize or returns malformed output. The recovery path:

1. **Retry** the compression call once with a shorter prompt.
2. **Fallback** to the older compressor (local token-counting deletion).
3. **Nuclear option** — discard the oldest N messages, keeping the system prompt and the most recent exchange.

This ensures the conversation never dies from context overflow.

## 4.6 Fallback Chain

When the primary model fails permanently (quota exhausted, model deprecated, auth revoked), Hermes falls back to the configured `fallback_model`:

```yaml
model:
  default: "anthropic/claude-sonnet-4"
  fallback_model: "openai/gpt-4o"
  provider: "openrouter"
```

The fallback preserves the conversation history but swaps the model. This happens transparently to the user, though the assistant may notice a change in "personality."

## 4.7 Background Review (Post-Turn)

After a turn completes, `agent/background_review.py` optionally spawns a lightweight review agent that:

1. Reads the completed turn's trajectory.
2. Checks for missed skills, incorrect tool usage, or suboptimal patterns.
3. If a correction is found, it can queue a `/steer` message for the next turn.

This is a **post-hoc** correction mechanism — it doesn't fix the current turn, but improves the next one.

## 4.8 Summary Table: Error Handling

| Error Type | Detector | Immediate Action | Escalation |
|------------|----------|------------------|------------|
| Rate limit 429 | `error_classifier.py` | Backoff + pool rotate | Fallback model |
| Context overflow | Token estimator | Compress history | Discard oldest |
| Tool failure | Exit code / JSON key | Inject error result | Guardrail warn/block |
| Same tool loop | Guardrail controller | Warn → Block → Halt | Turn aborted |
| Auth expired | `error_classifier.py` | Refresh token | Prompt user |
| Model unavailable | `error_classifier.py` | Switch to fallback | Notify user |
| Compression failure | Compressor try/except | Retry → older compressor | Nuclear delete |
| Background review miss | Review agent | Queue steer for next turn | None |
