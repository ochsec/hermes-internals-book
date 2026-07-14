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
| `auth` | HTTP 401/403 (transient) | Refresh/rotate credentials |
| `auth_permanent` | Auth failed after refresh | Abort turn |
| `billing` | HTTP 402, credit exhaustion, Anthropic "out of extra usage" 400 | Rotate to fallback model |
| `rate_limit` | HTTP 429, `Retry-After` header (account quota) | Exponential backoff + credential pool rotation |
| `upstream_rate_limit` | Aggregator 429 (model rate-limited, user key healthy) | Fallback to different model (no credential rotation) |
| `overloaded` | 503/529, "service is temporarily overloaded" | Adaptive backoff (provider-specific) |
| `server_error` | HTTP 500, 503, 502 | Jittered backoff (up to 5 retries) |
| `timeout` | Connection/read timeout, HTTP 408 | Rebuild client + retry |
| `ssl_cert_verification` | TLS handshake failure | Fail fast with fix hints (no retry) |
| `context_overflow` | "context length", "too long" | Compress history or fall back to larger-context model |
| `model_not_found` | 404, "model not found", OpenRouter "no tool use" 404 | Switch to fallback model |
| `content_policy_blocked` | Provider safety filter, MiniMax "new_sensitive" | Don't retry unchanged; inform user |
| `format_error` | "invalid json", 400 bad request | Sanitize messages and retry |
| `unknown` | Everything else | Retry with backoff |

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

### Z.AI/GLM-5.2 Overload Adaptive Backoff

Z.AI Coding Plan models (e.g. GLM-5.2) return a specific overload error (HTTP 1305, "The service may be temporarily overloaded...") that requires a different retry strategy. Standard short-retry loops tend to hammer the same overloaded window. `agent/retry_utils.py` detects this pattern via `is_zai_coding_overload_error()` and applies adaptive backoff:

- **Short attempts** — A bounded number of quick retries (`_ZAI_CODING_OVERLOAD_SHORT_ATTEMPTS`) with minimal delay, to catch transient blips.
- **Long attempts** — After short attempts are exhausted, switch to longer jittered backoff intervals to ride out sustained overload.
- The ceiling is computed by `zai_coding_overload_retry_ceiling()`, which sizes the total retry loop so every attempt fits within the provider's overload window.

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

## 4.4 Self-Correction During a Turn

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

### Compression Fallback Streak

`SessionDB` now persists a `compression_fallback_streak` counter per session (`get_compression_fallback_streak()` / `set_compression_fallback_streak()`). This tracks consecutive compression fallbacks — each time the LLM-driven compressor fails and Hermes falls back to the deterministic local deletion compressor, the streak increments. The streak is used by the compaction breaker (below) to detect persistently broken compression and avoid wasting cycles on a model that consistently fails to summarize.

### Fallback Compaction Breaker

A persistent `compaction_breaker` now counts fallback compactions as ineffective. If the LLM compressor repeatedly falls back to the deterministic compressor, the breaker trips and prevents further compression attempts within a cooldown window. The breaker state is persisted on the session row (`compression_failure_cooldown_until`, `compression_failure_error`), so a process restart doesn't reset the cooldown.

### Output-Cap Error Exemption

Parseable output-cap errors from vLLM and LM Studio (e.g., `"This model's maximum output tokens"`) are now exempted from the compression-disabled guard. Previously, an output-cap error would trigger compression, but if compression was disabled, the turn would fail. Now these errors are classified separately — the safe output budget is recalculated from the current input on each output-cap retry, and the retry cap is derived from the provider's `available_out` plus the request estimate rather than a fixed ceiling.

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

The review agent is **isolated from the canonical session**: it runs in a forked `AIAgent` that does not participate in session finalization, so its work does not pollute the session's token counts or trigger spurious compression. The auxiliary model used for the review is selectable via `auxiliary.review` config, and the review fork's memory tool is gated on `memory_enabled` to prevent unauthorized writes. The fork respects pinned skills and is scoped to read skills before writing them.

## 4.8 Summary Table: Error Handling

| Error Type | Detector | Immediate Action | Escalation |
|------------|----------|------------------|------------|
| Rate limit 429 | `error_classifier.py` | Backoff + pool rotate | Fallback model |
| Upstream rate limit | `error_classifier.py` | Fallback to different model | Notify user |
| Overloaded (503/529) | `error_classifier.py` | Adaptive backoff (Z.AI: short→long) | Fallback model |
| TLS cert failure | `error_classifier.py` | Fail fast with fix hints | Notify user |
| Timeout (408) | `error_classifier.py` | Rebuild client + retry | Fallback model |
| Context overflow | Token estimator | Compress history | Discard oldest |
| Tool failure | Exit code / JSON key | Inject error result | Guardrail warn/block |
| Same tool loop | Guardrail controller | Warn → Block → Halt | Turn aborted |
| Auth expired | `error_classifier.py` | Refresh token | Prompt user |
| Model not found | `error_classifier.py` | Switch to fallback | Notify user |
| Content policy blocked | `error_classifier.py` | Don't retry unchanged | Inform user |
| Compression failure | Compressor try/except | Retry → older compressor | Nuclear delete |
| Background review miss | Review agent | Queue steer for next turn | None |
