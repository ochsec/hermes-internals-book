# Guardrails & Safety

Hermes employs a layered safety model: **prompt-instructed norms** (the model is told what not to do), **runtime guardrails** (code that detects and intervenes), and **approval gating** (human-in-the-loop for destructive operations). This chapter describes all three layers.

## 2.1 Prompt-Level Guardrails

The primary guardrail is injected into the system prompt itself. The exact text varies by version, but the intent is constant: instruct the model to avoid infinite tool loops, avoid repeating failed calls, and respect user boundaries.

The prompt builder (`agent/prompt_builder.py`) constructs a guardrails block that covers:

- **Tool loop avoidance** — "Do not call the same tool with the same arguments repeatedly. If a tool fails, diagnose before retrying."
- **Destructive command caution** — "Always confirm before `rm -rf`, `git reset --hard`, or similar irreversible operations."
- **Secret hygiene** — "Scan output for API keys, tokens, and passwords before returning results to the user."
- **File safety** — "Never overwrite user files without checking content first."
- **Approval mode** — "Respect the user's approval settings (`manual`, `smart`, or `off`)."

These instructions are **advisory only** — they rely on the model's ability to follow instructions. Stronger models (Claude, GPT-4, Gemini) are more reliable at self-policing; weaker models may ignore them.

## 2.2 Runtime Guardrails: ToolCallGuardrailController

For cases where prompt instructions fail, `agent/tool_guardrails.py` provides a **pure, deterministic, side-effect-free** controller that tracks every tool call within a turn and enforces hard limits.

### Architecture

The `ToolCallGuardrailController` is instantiated per turn and maintains three counters:

1. **`_exact_failure_counts`** — Maps `(tool_name + canonical_args_hash)` → failure count. Detects identical repeated calls.
2. **`_same_tool_failure_counts`** — Maps `tool_name` → failure count. Detects repeated use of the same tool regardless of arguments.
3. **`_no_progress`** — Maps `(tool_name + args_hash)` → `(result_hash, repeat_count)`. Detects idempotent tools returning the same result repeatedly.

### Decision Flow

```
before_call(tool_name, args)
    │
    ├── hard_stop_enabled?
    │   ├── exact_count ≥ exact_failure_block_after  → BLOCK
    │   └── idempotent + same_result ≥ no_progress_block_after  → BLOCK
    │
    └── otherwise → ALLOW

after_call(tool_name, args, result, failed=?)
    │
    ├── failed == True
    │   ├── increment exact_failure_counts
    │   ├── increment same_tool_failure_counts
    │   ├── hard_stop + same_count ≥ same_tool_failure_halt_after  → HALT
    │   ├── warn + exact_count ≥ exact_failure_warn_after  → WARN
    │   └── warn + same_count ≥ same_tool_failure_warn_after  → WARN
    │
    └── failed == False
        ├── clear failure counters (success breaks the chain)
        ├── idempotent tool?  → compare result_hash
        │   └── same result ≥ no_progress_warn_after  → WARN
        └── otherwise → ALLOW
```

### Classification of "Failure"

The guardrail relies on `classify_tool_failure()` in `tool_guardrails.py`, which inspects the raw tool result string for:

- `"error"` or `"failed"` JSON keys
- `result.startswith("Error")`
- Terminal-specific nonzero exit codes
- Memory-full errors (`"exceed the limit"`)

**Exclusion**: If `file_mutation_result_landed()` returns True (a file was successfully mutated), the result is **not** considered a failure even if it contains error text. This prevents guardrails from penalizing successful file edits that also log warnings.

### Configurable Thresholds

All thresholds live under `tool_loop_guardrails` in `config.yaml`:

| Key | Default | Meaning |
|-----|---------|---------|
| `warnings_enabled` | true | Emit loop warnings |
| `hard_stop_enabled` | false | Enable circuit-breaker blocking |
| `warn_after.exact_failure` | 2 | Warn on identical repeated failures |
| `warn_after.same_tool_failure` | 3 | Warn on same-tool failures |
| `warn_after.idempotent_no_progress` | 2 | Warn on identical idempotent results |
| `hard_stop_after.exact_failure` | 5 | Block identical repeated failures |
| `hard_stop_after.same_tool_failure` | 8 | Halt same-tool failures |
| `hard_stop_after.idempotent_no_progress` | 5 | Block identical idempotent repeats |

By default, `hard_stop_enabled` is **false** — Hermes warns but never blocks. Users can enable blocking by setting `hard_stop_enabled: true`.

### Synthetic Results

When a call is `block`ed or `halt`ed, the guardrail synthesizes a fake tool result:

```json
{
  "error": "Blocked {tool_name}: the same tool call failed {N} times with identical arguments. Stop retrying it unchanged; change strategy or explain the blocker.",
  "guardrail": { "action": "block", "code": "repeated_exact_failure_block", ... }
}
```

This synthetic result is injected into the conversation as a `role="tool"` message, so the model sees the block and can adapt.

## 2.3 Destructive Command Approval

Before any tool that could damage the system runs, Hermes checks whether it needs user approval.

### Approval Modes

Set via `approvals.mode` in `config.yaml`:

- **`manual`** — Always prompt before destructive commands (default).
- **`smart`** — An auxiliary LLM auto-approves low-risk commands, prompts on high-risk.
- **`off`** — Skip all approval prompts (equivalent to `--yolo` flag).

Per-invocation bypass without changing config: `hermes --yolo` or `export HERMES_YOLO_MODE=1`.

### What Counts as "Destructive"

`agent/tool_dispatch_helpers.py::_is_destructive_command()` inspects terminal commands for:

- `rm -rf`, `rm -r`, `rm -f` (recursive force deletion)
- `git reset --hard`
- `dd` (disk writing)
- `mkfs`, `fdisk`, `parted`
- `chmod 000`, `chown -R`
- Any command containing `>` redirect overwriting critical files (e.g., `> /etc/passwd`)

File-mutating tools (`write_file`, `patch`) on existing files also trigger approval unless the file was created in the current turn.

### Smart Approval

In `smart` mode, a lightweight LLM call evaluates the command string against a rubric:

- Does the command target a file in the current project directory?
- Is the command reversible (git-tracked, backup exists)?
- Does the command pattern match known-safe operations?

If the auxiliary model returns "safe", the command executes without interruption. If "uncertain", the user is prompted. This adds ~200–500 ms latency to approved commands.

## 2.4 Secret Redaction

`agent/redact.py` provides regex-based secret detection and masking.

- **Off by default** (`security.redact_secrets: false`).
- When enabled, all tool output (terminal stdout, `read_file`, web content, subagent summaries) is scanned for patterns resembling API keys, tokens, and passwords before entering the conversation context.
- Patterns cover: `sk-` (OpenAI), `ghp_` (GitHub), AWS keys, JWT tokens, base64-encoded credentials, generic hex strings, Slack App-Level tokens (`xapp-`), Fireworks AI API keys, bare-token credentials in URL userinfo, database connection strings, and passwords in lowercase/dotted config keys.
- Secrets are replaced with `[REDACTED-<hash>]` so the model knows something was hidden without seeing the value.
- Prefix secrets in file reads use a non-reusable sentinel to prevent repeated exposure.
- Redaction is applied to background process output and foreground environment dumps.
- Tool-call arguments in conversation history are **not** redacted (this was found to break auth headers by quote-eating); redaction is scoped to tool results, not the model's own call parameters.

**Important**: `security.redact_secrets` is snapshotted at import time. Changing it mid-session has no effect — a new session is required.

## 2.5 File Safety

`agent/file_safety.py` provides additional runtime checks for file mutations:

- **Atomic writes** — `write_file` and `patch` use atomic rename (`utils.atomic_json_write`) to prevent half-written files.
- **Backup on overwrite** — Existing files are backed up to `~/.hermes/backups/` before mutation.
- **Symlink traversal guard** — Prevents writing through symlinks that escape the intended directory.
- **Critical path denylist** — Refuses to write to `/etc/`, `/usr/`, `/bin/`, and other system directories unless running as root (and even then, warns).

## 2.6 Message Sanitization

`agent/message_sanitization.py` enforces structural correctness of messages before they reach the LLM:

- **Surrogate removal** — Strips orphaned UTF-16 surrogate pairs that break JSON serialization.
- **Non-ASCII cleaning** — Removes control characters that confuse providers.
- **Image stripping** — Removes image attachments from messages when the provider doesn't support multimodal input.
- **Structure repair** — Fixes nested JSON strings and escaped characters in tool call arguments.

## 2.7 SSL Guard

`agent/ssl_guard.py` validates TLS certificates for HTTPS connections made by tools (browser, web search, API calls). It can be configured to:

- Reject self-signed certificates (default).
- Allow specific pinned certificates for internal services.
- Log certificate fingerprints for auditing.

## 2.8 Error Classification & Recovery

`agent/error_classifier.py` categorizes API errors into `FailoverReason` enums:

- `auth` — Transient auth (401/403) — refresh/rotate credentials.
- `auth_permanent` — Auth failed after refresh — abort.
- `billing` — 402 or confirmed credit exhaustion — rotate immediately.
- `rate_limit` — 429 or quota-based throttling — backoff then rotate.
- `upstream_rate_limit` — Upstream model rate-limited (aggregator 429) — fallback to different model, not credential rotation.
- `overloaded` — 503/529 — provider overloaded, backoff.
- `server_error` — 500/502 — internal server error, retry.
- `timeout` — Connection/read timeout — rebuild client + retry.
- `ssl_cert_verification` — TLS certificate verification failure — fail fast with actionable guidance.
- `context_overflow` — Context too large — compress, not failover.
- `payload_too_large` — 413 — compress payload.
- `image_too_large` — Native image part exceeds provider's per-image limit — shrink and retry.
- `model_not_found` — 404 or invalid model — fallback to different model.
- `provider_policy_blocked` — Aggregator blocked the only endpoint due to data/privacy policy.
- `content_policy_blocked` — Provider safety filter rejected this prompt — deterministic, don't retry unchanged.
- `format_error` — 400 bad request — abort or strip + retry.
- `invalid_encrypted_content` — Responses replay blob rejected — strip replay state and retry.
- `multimodal_tool_content_unsupported` — Provider rejected list-type content in tool messages — downgrade to text and retry.
- `thinking_signature` — Anthropic thinking block signature invalid.
- `long_context_tier` — Anthropic "extra usage" tier gate.
- `oauth_long_context_beta_forbidden` — Anthropic OAuth subscription rejects 1M context beta — disable beta and retry.
- `llama_cpp_grammar_pattern` — llama.cpp json-schema-to-grammar rejects regex escapes in `pattern`/`format` — strip from tools and retry.
- `unknown` — Unclassifiable — retry with backoff.

Each classification drives a different recovery strategy in `conversation_loop.py`.

## 2.9 Summary Table: Guardrail Layers

| Layer | Mechanism | Enforced By | Override |
|-------|-----------|-------------|----------|
| Prompt norms | System prompt instructions | LLM compliance | `--yolo` (weakens instructions) |
| Tool loop guardrails | Exact/same-tool/no-progress counters | `ToolCallGuardrailController` | `hard_stop_enabled: false` |
| Destructive approval | Human or auxiliary LLM | `tool_dispatch_helpers.py` | `approvals.mode: off` or `--yolo` |
| Secret redaction | Regex masking | `agent/redact.py` | `security.redact_secrets: false` |
| File safety | Atomic writes, backups, path guards, credential-path blocking | `agent/file_safety.py` | none (always active) |
| Message sanitization | Surrogate/control-char removal | `agent/message_sanitization.py` | none (always active) |
| SSL validation | Certificate pinning, fail-fast on verification failure | `agent/ssl_guard.py` | `ssl.verify: false` |
| Error classification | 23 `FailoverReason` values driving recovery | `agent/error_classifier.py` | per-error retry/fallback config |

## 2.10 Guardrail Interaction Diagram

```
┌────────────────────┐
│  Tool Call Request │
└────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  _is_destructive_command()? │──YES──▶ User Approval (or smart bypass)
└─────────────────────────────┘
          │ NO
          ▼
┌─────────────────────────────┐
│  ToolCallGuardrailController│
│  .before_call()             │──BLOCK──▶ Synthetic error result
└─────────────────────────────┘
          │ ALLOW
          ▼
┌─────────────────────────────┐
│  Execute tool               │
└─────────────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  classify_tool_failure()?     │
│  ToolCallGuardrailController│
│  .after_call()              │──WARN/HALT──▶ Injected guidance
└─────────────────────────────┘
          │ OK
          ▼
┌─────────────────────────────┐
│  redact_sensitive_text()?   │──REDACT──▶ Masked output
└─────────────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  Return to conversation loop│
└─────────────────────────────┘
```
