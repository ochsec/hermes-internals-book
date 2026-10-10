# Self-Correction

Hermes does not assume the model or the tools are infallible. Every layer of the stack has error detection, classification, and recovery logic. This chapter describes how errors are caught, classified, retried, and ultimately resolved.

## 4.1 Error Sources

Errors can originate from three sources:

1. **LLM API errors** — Rate limits, context-length overflows, authentication failures, malformed requests.
2. **Tool execution errors** — Command failures, missing files, network timeouts, permission denied.
3. **System errors** — Disk full, memory pressure, process crashes, thread deadlocks.

Each source has its own classification and recovery path.

## 4.2 LLM API Error Classification

`agent/error_classifier.py::classify_api_error()` inspects the exception, HTTP status,
structured error code, and error body to produce a `ClassifiedError` — a `FailoverReason`
enum plus recovery hints (`retryable`, `should_compress`, `should_rotate_credential`,
`should_fallback`) that the retry loop checks instead of re-classifying:

| Enum | Triggers | Recovery |
|------|----------|----------|
| `auth` | HTTP 401/403 (transient) | Refresh/rotate credentials |
| `auth_permanent` | Auth failed after refresh | Abort turn |
| `billing` | HTTP 402, credit exhaustion, Anthropic "out of extra usage" 400 | Rotate to fallback model |
| `rate_limit` | HTTP 429, `Retry-After` header (account quota) | Exponential backoff + credential pool rotation |
| `upstream_rate_limit` | Aggregator 429 (model rate-limited, user key healthy) | Fallback to different model (no credential rotation) |
| `overloaded` | 503/529, "service is temporarily overloaded" | Adaptive backoff (provider-specific) |
| `server_error` | HTTP 500, 502, empty provider response | Jittered backoff |
| `timeout` | Connection/read timeout, HTTP 408, transport error types | Rebuild client + retry |
| `ssl_cert_verification` | TLS handshake failure | Fail fast with fix hints (no retry) |
| `context_overflow` | "context length", "too long", Z.AI GLM code 1210 ("tokens in request more than max tokens allowed") | Compress history or fall back to larger-context model |
| `payload_too_large` | HTTP 413 | Compress payload |
| `image_too_large` | Image part exceeds provider per-image limit | Shrink image and retry |
| `image_corrupt` | Provider cannot decode image bytes | Strip image and retry |
| `model_not_found` | 404, "model not found", OpenRouter "no tool use" 404, Nous retired free-route 404 ("is no longer free") | Switch to fallback model (non-retryable) |
| `upstream_blocked` | 403 from a WAF/CDN/proxy in front of the provider | Fallback model — key is healthy |
| `provider_policy_blocked` | Aggregator account data/privacy policy excluded the only endpoint ("no endpoints found matching your data policy") | Fallback model |
| `model_entitlement` | Account cannot use the requested model (e.g. Codex entitlement 400) | Rotate credential (model-scoped), else fallback |
| `incomplete_response` | Codex/Responses turn stuck emitting reasoning only after replay + nudge | Hand to a different provider |
| `role_alternation` | Strict chat template rejected adjacent same-role messages | Merge same-role messages and retry |
| `invalid_encrypted_content` | Responses replay blob rejected | Strip replay state and retry |
| `multimodal_tool_content_unsupported` | Provider rejected list-type content in tool messages (e.g. Xiaomi MiMo) | Downgrade to text and retry |
| `reasoning_mandatory` | Route rejects `reasoning: {enabled: false}` | Step effort up to the floor; don't send the disable again |
| `content_policy_blocked` | Provider safety filter, MiniMax "new_sensitive" | Don't retry unchanged; inform user |
| `format_error` | 400 bad request, status-less stream render errors ("error rendering", jinja) | Abort to fallback chain; callers with a sanitize step strip and retry |
| `thinking_signature` | Anthropic thinking block signature invalid | Retry (provider-specific) |
| `long_context_tier` | Anthropic "extra usage" tier gate 429 | Cap tokens and compress |
| `llama_cpp_grammar_pattern` | llama.cpp grammar rejects regex `pattern`/`format` | Strip from tools and retry |
| `oauth_long_context_beta_forbidden` | Anthropic OAuth rejects the 1M beta header | Disable beta and retry |
| `unknown` | Everything else | Retry with backoff |

A few notable additions to the pattern tables. Z.AI/Zhipu GLM reports context overflow as
"tokens in request more than max tokens allowed" (error code 1210), which previously
classified as `unknown`/retryable — the agent would retry an oversized request forever
instead of triggering compression. The OpenRouter "no endpoints found that support tool
use" 404 and the Nous gateway's "This model is no longer free" 404 (a retired free route)
both map to `model_not_found`, so the retry loop stops hammering a slug that can never
succeed. Two other verdicts deserve mention: `_V_MALFORMED_TOOL_ARGS` maps proxy-rejected
model-emitted tool-call JSON (Ollama, OpenRouter) to `format_error` with
`should_fallback=False` — no other provider can fix that output, so the loop's argument
repair handles it instead of replaying the broken turn on every fallback candidate — and
`_nous_welcome_tier()` classifies the anonymous free tier's fairshare 429s: a tier gate
(`model_not_free`, `feature_not_free`) falls back, a capacity refusal (`at_capacity`,
`admission_closed`) honors `retry_after` and never rotates the tier's only credential.

### The Classification Pipeline

`classify_api_error()` runs the verdict through a staged pipeline (`_STAGES`), first match
wins:

1. **Plugin hook** — a `kind: model-provider` plugin's `classify_error` hook can add or
   correct verdicts before anything built-in runs.
2. **Provider profile hook** — the provider's own `ProviderProfile.classify_api_error`
   (e.g. a vendor 403 `quota_exhausted` that is billing, not auth). A hook that asks for
   fallback gets the built-in default `retryable` for its reason, so it cascades like a
   built-in verdict.
3. **Provider special cases** — Nous welcome-tier refusals, Codex account entitlements,
   reasoning-rejection shapes.
4. **HTTP status** — with message-aware refinement (e.g. a 404 body is checked for
   billing/policy/model-not-found wording; overflow 5xx patterns run before the generic
   5xx verdict).
5. **MoA aggregator shapes, structured error code, message patterns, SSL → disconnect
   heuristic → transport error types**, then `unknown` (retryable with backoff).

Only reasons in `RETRYABLE_CLIENT_REASONS` (rate limit family, overload, overflow,
payload_too_large, long_context_tier, thinking_signature) keep retrying with backoff
before the fallback chain runs; everything else is non-retryable and switches routes
immediately.

### The `_pool_may_recover_from_rate_limit` Gate

Before burning retry budget on rate-limit retries, Hermes checks
`run_agent.py::_pool_may_recover_from_rate_limit()` (repo root; the check is invoked from
`agent/turn_recovery.py`). The gate is now purely structural — rotation only helps when
the pool has somewhere to go:

- The pool must exist, have a credential currently available, and hold **more than one
  entry**. A single-credential pool would retry the same quota, so rotation is skipped.
- An upstream-aggregator 429 (`upstream_rate_limit`) bypasses the gate entirely: the user
  key is healthy, so Hermes always falls back to a different model.

In both no-recovery cases, Hermes switches to the fallback model instead of waiting.

### Retry Strategy

Retries use `agent/retry_utils.py::jittered_backoff()`. The attempt is 1-based, the delay
is `min(base_delay * 2^(attempt-1), max_delay)` plus uniform jitter in
`[0, jitter_ratio * delay]`, and the jitter seed mixes `time.time_ns()` with a
lock-protected counter so concurrent gateway sessions decorrelate even on coarse clocks:

```python
def jittered_backoff(attempt: int, *, base_delay: float = 5.0, max_delay: float = 120.0,
                     jitter_ratio: float = 0.5) -> float:
    ...
    exponent = max(0, attempt - 1)
    delay = max_delay if (exponent >= 63 or base_delay <= 0) else min(base_delay * (2 ** exponent), max_delay)
    seed = (time.time_ns() ^ (tick * 0x9E3779B9)) & 0xFFFFFFFF
    return delay + random.Random(seed).uniform(0, jitter_ratio * delay)
```

The retry budget is `agent.api_max_retries` in config (default **3**), with a hard ceiling
to prevent runaway loops; rate-limit waits parsed from provider error bodies
(`Retry-After`, "resets in 4hr 5min", `resets_in_seconds`) feed both the retry wait and the
credential pool's cooldown so they agree. A wait longer than the interactive
`LIVE_RETRY_WAIT_CAP_S` (60s) is announced to the user when it starts.

### Z.AI/GLM-5.2 Overload Adaptive Backoff

Z.AI Coding Plan models (e.g. GLM-5.2) return a specific overload error (HTTP **429 with
code 1305**, "The service may be temporarily overloaded...") that requires a different
retry strategy. Standard short-retry loops tend to hammer the same overloaded window.
`agent/retry_utils.py` detects this shape via `is_zai_coding_overload_error()` (scoped to
the `api.z.ai/api/coding/paas/v4` base URL and the `glm-5.2` model, so ordinary quota 429s
still fail fast) and `adaptive_rate_limit_backoff()` applies the schedule:

- **Short attempts** — the first `_ZAI_CODING_OVERLOAD_SHORT_ATTEMPTS` (3) retries wait the
  normal default, to catch transient blips.
- **Long attempts** — after short attempts are exhausted, the wait widens through the
  `_ZAI_CODING_OVERLOAD_LONG_BACKOFF` ladder (30 → 60 → 90 → 120s) with light jitter, to
  ride out sustained overload.
- The ceiling is computed by `zai_coding_overload_retry_ceiling()` (`short_attempts +
  len(long ladder) + 1` = 7), which sizes the total retry loop so every attempt fits
  within the provider's overload window; turn recovery raises `max_retries` to it when the
  overload shape is detected.

## 4.3 Tool Execution Error Classification

Tool errors are classified at three levels:

### Level 1: Raw Exit Code (terminal tools)

`terminal` returns `{"output": "...", "exit_code": N}`. Nonzero exit codes are failures.

### Level 2: JSON Error Keys

Most tools return JSON. If the top-level object contains `"error"` or `"failed"`, it's a failure. The `classify_tool_failure()` function in `agent/tool_guardrails.py` does this consistently.

### Level 3: Result Pattern Matching

For tools that don't return structured JSON, `classify_tool_failure()` in
`agent/tool_guardrails.py` inspects the first 500 characters (lowercased) for a quoted
`"error"` or `"failed"` JSON key, or a result string starting with `Error`. This mirrors
`agent.display._detect_tool_failure` so the guardrail never disagrees with the CLI's
`[error]` tag. A harness **refusal** of a redundant call (a repeated identical
read/search, stamped `guardrail_refusal: true`) also carries `"error"` for the model's
benefit — but nothing failed, so refusals are exempted before the substring test; counting
them would let a cheap refusal feed the streak that fires a harder block later.

### Special Case: File Mutation Success

`file_mutation_result_landed()` in `agent/tool_result_classification.py` returns True when
a file mutation result **proves the write landed**: a `write_file` result containing
`bytes_written`, or a `patch` result with `success: true`, and no `error` key. Such a
result is **not** a failure for guardrail purposes, even if it carries warnings. This
prevents the tool loop guardrail from punishing successful edits.

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

The guardrails ensure this doesn't loop indefinitely (`agent/tool_guardrails.py`, defaults
from `ToolCallGuardrailConfig`):
- After 2 identical-argument failures, a warning is injected.
- After 5 identical failures (if hard_stop_enabled — opt-in on interactive platforms,
  default-on for unattended gateway/cron sessions), the call is blocked.
- Same-tool failures with *different* arguments warn at 3 and halt the turn at 8.
- An identical-call *streak* (same tool, same args, same result) and repeating multi-call
  **cycles** (A,B,A,B,... with identical args and results) are also detected — a cycle
  resets the consecutive streak but not the per-turn call history.
- Per-turn `loop_caps` hard-ceiling `web_search` calls and subagent spawns regardless of
  `hard_stop_enabled`.

On halt, the canonical decision is recorded as the turn's `halt_decision`, and byte-identical
duplicate results from the 2nd repeat are replaced with a reference stub to save context.

## 4.5 Compression Failure Recovery

`agent/context_compressor.py` and `agent/conversation_compression.py` can fail if the LLM
refuses to summarize, stalls mid-stream, or the turn outruns the compaction budget. Every
summary route runs under a progress-aware watchdog: `compression.context_timeout_seconds`
(default **120s**) of silence kills the attempt, and
`compression.context_total_ceiling_seconds` (default **600s**) bounds the whole
compaction. The recovery ladder:

1. **Stall retry on pinned routes** — a stalled or failed primary summary route is retried
   on the compression fallback route from the fallback chain, bounded the same way as the
   primary (one silent idle window ends it). Once a stall-class backoff has already burned
   a window in this session, the retry ladder also includes the deterministic route.
2. **Deterministic fallback summary** — a local, no-LLM deletion summary commits instead,
   so the turn keeps going even when no summary model answers. The committed boundary
   proves compressibility and lifts structural backoff.
3. **Budgeted attempts** — threshold-triggered compaction runs at most
   `compression.max_attempts` (default 3, cap 10) preflight passes per turn; a pass that
   moves neither rows nor tokens fails the turn closed rather than looping.

### Fallback Streak and Summary-Model Benching

`SessionDB` persists a `compression_fallback_streak` counter per session
(`get_compression_fallback_streak()` / `set_compression_fallback_streak()` in
`hermes_state_compression.py`). Each time the LLM-driven compressor fails and the
deterministic fallback summary commits, the streak increments; a healthy summary resets it
to zero. The streak persists across compaction rotations (the parent row's streak carries
into the fresh child row).

The streak does **not** block compaction — a static fallback still reclaims the window,
and blocking on it would let the prompt run into the provider's hard limit. Instead the
streak **benches** the failing summary model: after repeated fallbacks the compressor
skips calling the summary model for a while (a pre-LLM summary skip, handled by
`PreLlmSkipMixin` in `agent/context_compressor_prellm.py`), then periodically re-probes it
(the bench deadline is persisted in the model-config blob and carried across rotating
compaction, so a restart doesn't re-try a model that just failed). A deliberate skip is
streak-neutral: it neither extends nor resets the streak, because it proves nothing about
the model's health.

### Anti-Thrash Breaker (Ineffective Compactions)

A separate breaker, `_ineffective_compression_count`, counts compactions that were
**rejected before commit** (would have grown the transcript) or otherwise failed to shrink
the request. Only provider-confirmed ineffective compactions trip it — at **2 strikes**
automatic compaction is blocked, then after a 300-second anti-thrash recovery window one
probe attempt is allowed (the count drops to a single strike, so another ineffective pass
re-trips immediately). A *structural* no-op (nothing eligible to compact) does not strike
the breaker — it only defers retries via a 300s backoff, since striking would permanently
disarm auto-compaction on short sessions. Strike counts are persisted on the session row
(`get_compression_ineffective_count`), and summary-failure cooldowns on
`compression_failure_cooldown_until` / `compression_failure_error`, so a process restart
doesn't reset either. Consecutive timeout-class failures persist as
`backoff:<kind>:strategy=<tail_mode>` records, and the timeout cooldown is never shorter
than the idle window that just failed.

### Output-Cap Error Exemption

Parseable output-cap errors from vLLM and LM Studio (e.g., `"This model's maximum output tokens"`) are exempted from the compression-disabled guard: with `compression.enabled: false`, an overflow from a too-large `max_tokens` still recovers by clamping the cap instead of failing the turn. The clamp (`agent/turn_overflow.py::_clamp_output_cap`) recomputes the safe budget from the *current* request on each retry — `min(provider-reported available_out, context_length − estimated request tokens) − 64` — rather than a fixed ceiling, and additionally compresses history once so the total fits. Some gateways wrap the upstream output-cap 400 as an HTTP 429, which would otherwise classify as `rate_limit` and trigger eager fallback; turn recovery parses the cap figure out of the body and exempts it from the fallback switch, because only the `max_tokens` clamp fixes a request-shape problem.

## 4.6 Fallback Chain

When the primary route fails permanently (quota exhausted, model deprecated, auth revoked),
Hermes walks the configured fallback **chain** in order. The chain is stored as top-level
`fallback_providers` in the config — a list of `{provider, model, base_url?}` entries — and
is managed by the `hermes fallback add/list/remove/clear` command (which reuses the
`hermes model` picker). The legacy single `fallback_model` key is still read and merged
(appended, deduplicated by provider/model/base_url) for backwards compatibility, but
`hermes fallback add` rewrites the config to `fallback_providers` only.

```yaml
fallback_providers:
  - provider: "openrouter"
    model: "openai/gpt-4o"
  - provider: "google"
    model: "gemini-3-pro"
```

The chain preserves the conversation history and swaps the route. It is consumed eagerly,
not just after permanent failure: rate-limit and billing verdicts switch immediately (the
primary won't recover in the retry window — unless the output-cap exemption applies or the
credential pool gate says rotation may recover), transport failures get 2 retries first,
and a 401/403 surviving credential refresh escalates to the chain exactly once. A pinned
owner (a delegated child or cron job that explicitly pinned its provider/model) never
borrows the inherited chain — predictability beats liveness for an explicit pin; an
explicit `fallback_providers: []` disables fallback. This happens transparently to the
user, though the assistant may notice a change in "personality."

## 4.7 Background Review (Post-Turn)

After a turn completes, `agent/background_review.py` may spawn a daemon thread that
replays the completed turn's conversation snapshot in a **forked `AIAgent`** and asks one
question: *should any skill or memory be saved or updated?* Unlike the user-facing
`/review` command (a full-privilege review subagent routed via `auxiliary.review`), this
post-turn fork writes its findings straight to the memory and skill stores; the main
conversation and the prompt cache are never touched.

The fork **inherits the parent's live runtime** — provider, model, credentials, and the
cached system prompt — so it hits the same warm prefix cache instead of re-reading the
whole conversation, and it runs under a dispatch-side tool whitelist. Its aggregate
input-token budget is capped (75% of the fork's resolved context window, with a hard 600k
ceiling), and when `auxiliary.background_review.{provider,model}` routes it to a *different*
model than the main one, it replays a compact digest instead of the full replay (the cache
would be cold anyway). The fork's memory tool is gated on the parent's `memory_enabled`
(and user-profile flag) to prevent unauthorized writes, and it may only edit
curator-managed skills — bundled, hub, pinned, and user-owned skills are off-limits; only
a foreground session can change a pinned one. The fork's token usage is attributed to the
**parent** session, and a new live turn preempts (cancels) an in-flight review the same
way it would cancel any pending work.

This is a **post-hoc** self-correction mechanism — it doesn't fix the current turn, but it
improves the next one by promoting the right skills and memories from what just happened.

## 4.8 Summary Table: Error Handling

| Error Type | Detector | Immediate Action | Escalation |
|------------|----------|------------------|------------|
| Rate limit 429 | `error_classifier.py` | Backoff + pool rotate | Fallback model |
| Upstream rate limit | `error_classifier.py` | Fallback to different model | Notify user |
| Overloaded (503/529) | `error_classifier.py` | Adaptive backoff (Z.AI: short→long) | Fallback model |
| TLS cert failure | `error_classifier.py` | Fail fast with fix hints | Notify user |
| Timeout (408) | `error_classifier.py` | Rebuild client + retry | Fallback model |
| Context overflow | `error_classifier.py` | Compress history (watchdog-bounded) | Output-cap clamp / fail closed |
| Tool failure | Exit code / JSON key | Inject error result | Guardrail warn/block |
| Same tool loop | Guardrail controller | Warn → Block → Halt | Turn aborted |
| Auth expired | `error_classifier.py` | Refresh token | Fallback chain (once) |
| Model not found | `error_classifier.py` | Switch to fallback | Notify user |
| Content policy blocked | `error_classifier.py` | Don't retry unchanged | Inform user |
| Compression failure | Summary watchdog / try/except | Stall retry on fallback-chain routes | Deterministic fallback summary |
| Ineffective compaction | Anti-thrash breaker | Block auto-compaction (2 strikes) | 300s recovery, then one probe |
| Background review miss | Memory/skill review fork | Write skill/memory stores | None |
