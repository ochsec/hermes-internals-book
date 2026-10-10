# Guardrails & Safety

Hermes employs a layered safety model: **prompt-instructed norms** (the model is told what not to do), **runtime guardrails** (code that detects and intervenes), and **approval gating** (human-in-the-loop for destructive operations). This chapter describes all three layers.

## 2.1 Prompt-Level Guardrails

The system prompt is assembled by `agent/prompt_builder.py` from themed blocks. There is no single "guardrails block"; safety guidance is woven into execution-discipline blocks that ship in every prompt:

- **Tool persistence** — the model is told to keep working until the task is actually complete and never to stop with a summary of planned work. This is the prompt-side counterweight to the runtime loop guardrails in §2.2.
- **Act, don't ask** — questions with an obvious default interpretation are answered with a tool call, not a clarification request; only ambiguity that changes which tool to call is worth asking about.
- **Scope confirmation** — "Safety: if the next step has side effects (file writes, commands, API calls), confirm scope before executing."
- **Skill Safety Rule** — the `SKILL_SAFETY_HEADING` constant (referenced by tests and compaction summaries) teaches the `[SKILL_PRUNED]` reload contract: a placeholder that lost its content to context compression must be reloaded with `skill_view`, never invented.

Per-tool guidance lines are gated on the session's active toolset, so the prompt never advertises instructions for tools the model cannot call.

These instructions are **advisory only** — they rely on the model's ability to follow instructions. Stronger models are more reliable at self-policing; weaker models may ignore them, which is why the runtime layers below exist.

## 2.2 Runtime Guardrails: ToolCallGuardrailController

For cases where prompt instructions fail, `agent/tool_guardrails.py` provides a **pure, deterministic, side-effect-free** controller that tracks every tool call within a turn and enforces hard limits.

### Architecture

The `ToolCallGuardrailController` is instantiated per turn (state reset by `reset_for_turn()` at turn start) and maintains several counters and windows:

1. **`_exact_failure_counts`** — Maps a `ToolCallSignature` (tool name + SHA-256 of the canonical sorted-JSON args) → failure count. Detects identical repeated calls.
2. **`_same_tool_failure_counts`** — Maps `tool_name` → failure count. Detects repeated use of the same tool regardless of arguments.
3. **`_no_progress`** — Maps signature → `(result_hash, repeat_count)`. Detects idempotent tools returning the same result repeatedly.
4. **`_progress_since_failure`** — Signature → True once a mutating tool (`write_file`, `patch`, `terminal`, …) or a landed file mutation succeeded since that signature's last failure. A retry after a real edit counts as a *new experiment*, not a replay, and restarts the exact-args streak.
5. **Stall-guard state** — the identical-call streak (`_identical_streak_*`) and a bounded call history (`_call_history`) used by the cycle detector described below.
6. **Loop-cap counters** — `_turn_web_search_count` / `_turn_subagent_count` for the per-turn ceilings below.

A further classification matters before any counting: **failure-tolerant tools** (`terminal`, `execute_code`, `process_manage`, `browser_navigate`, `web_extract`) treat failure as normal work output — a run of *distinct* red commands is diagnosis, not a loop, so `same_tool_failure` never halts them; only an exact-args replay with no intervening change, or an identical-result streak, can.

### Decision Flow

```
before_call(tool_name, args)
    │
    ├── loop cap reached? (web_search / delegate_task)   → BLOCK (always, even without hard stops)
    │
    ├── hard_stop_enabled?
    │   ├── mutating progress since last failure?  → treat count as 0
    │   ├── exact_count ≥ exact_failure_block_after  → BLOCK
    │   └── idempotent + same_result ≥ no_progress_block_after  → BLOCK
    │
    └── otherwise → ALLOW

after_call(tool_name, args, result, failed=?)
    │
    ├── failed == True
    │   ├── progress since last failure?  → reset the exact-args streak
    │   ├── increment exact_failure_counts
    │   ├── increment same_tool_failure_counts
    │   ├── hard_stop + not failure-tolerant + same_count ≥ same_tool_failure_halt_after  → HALT
    │   ├── warn + exact_count ≥ exact_failure_warn_after  → WARN
    │   └── warn + same_count ≥ same_tool_failure_warn_after  → WARN (tool-specific recovery hint)
    │
    └── failed == False
        ├── clear failure counters (success breaks the chain)
        ├── mutating success marks progress for every still-failing signature
        ├── idempotent tool?  → compare result_hash
        │   └── same result ≥ no_progress_warn_after  → WARN
        └── otherwise → ALLOW
```

### Classification of "Failure"

The guardrail relies on `classify_tool_failure()` in `tool_guardrails.py`, which deliberately mirrors `agent.display._detect_tool_failure` (so the guardrail never disagrees with the CLI's `[error]` tag) and inspects the raw tool result string for:

- `"error"` or `"failed"` JSON keys in the first 500 characters, or `result.startswith("Error")`
- Terminal-specific nonzero exit codes (parsed from the result JSON)
- Memory-full errors (the `memory` tool reporting `success: false` with `"exceed the limit"`)

**Exclusions**: If `file_mutation_result_landed()` returns True (a `write_file`/`patch` result proves the write landed), the result is **not** considered a failure even if it contains error text. Likewise, a harness **guardrail refusal** of a redundant call (repeated identical read/search) carries `"error"` for the model's benefit — `is_guardrail_refusal()` detects it so the cheap refusal cannot feed the streak that fires the next, harder one.

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
| `non_interactive_hard_stop_enabled` | true | On unattended platforms (gateway/cron), enable hard stops even though `hard_stop_enabled` is false — nobody can `/stop` a model that ignores warnings |
| `loop_caps.max_web_searches` | 50 | Per-turn `web_search` ceiling (0 disables) |
| `loop_caps.max_subagents` | 50 | Per-turn subagent-spawn ceiling (0 disables) |

Interactive surfaces (`cli`, `tui`, `desktop`, `acp`, `subagent`, `api_server`) keep the warn-only default: `hard_stop_enabled` is **false** and Hermes warns but never blocks. Users can enable blocking by setting `hard_stop_enabled: true`. Unattended gateway/cron platforms hard-stop by default via `non_interactive_hard_stop_enabled`.

### Stall Guards: Identical Streaks and Cycles

`observe_call()` adds a second, tool-agnostic detector that watches consecutive identical `(tool, args, result)` triples:

- From the 3rd consecutive identical call (`STALL_GUARD_IDENTICAL_CALL_THRESHOLD`), an observational `[hermes note: …]` is appended to the result. Poller-style tools (`process_manage`, and any `*_get_result` / `*_poll` names) are exempt — an unchanged poll *is* progress.
- With hard stops enabled, the streak halts at `no_progress_block_after`. This closes a real gap: the no-progress *block* in `before_call` only covers idempotent tools, so a model replaying the same **successful** `terminal` call would otherwise run to the iteration budget.
- **Batch-cycle detection** catches repeating multi-call cycles (A,B,A,B,… with identical arguments *and* identical results) up to a period of 4 — a cycle that resets the consecutive streak on every alternation. Cycle laps reuse the same thresholds (notice at 3, halt at `no_progress_block_after`).
- From the 2nd byte-identical repeat, a result of 512+ characters is replaced by a **reference stub** ("this result is byte-identical to the earlier `tool_name` result this turn"), so deduplication evicts duplicate payload from context without lying about what executed. Stubs can link to the persisted-output path if the original result spilled to disk.

The per-call result hash also normalizes `execute_code` output: that tool's own per-call metadata (`kernel.execution_count`, `duration_seconds`) changes on every invocation even when the code did nothing new, and hashing it as-is once let a model re-run an empty probe 147 times unflagged.

### Per-Turn Loop Caps

`tool_loop_guardrails.loop_caps` sets hard ceilings on runaway-prone tools: `max_web_searches: 50` and `max_subagents: 50` per turn (0 disables a cap). Unlike the failure detector, caps apply **regardless of `hard_stop_enabled`** — the check runs in `before_call`, so the (cap+1)-th call is refused with a synthetic block result before it executes. A `delegate_task` batch spawning N subagents counts N against its cap; control actions (`list`/`steer`/`stop`) count nothing.

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

Before any tool that could damage the system runs, Hermes checks whether it needs user approval. The approval machinery lives in the `tools/approval*.py` family — `approval.py` (the gate itself), `approval_detection.py` (pure command classification), `approval_smart.py` (the auxiliary-LLM verdict), and `approval_context.py` (config and session-identity contextvars).

### Approval Modes

Set via `approvals.mode` in `config.yaml` (shipped default: `smart`):

- **`smart`** — An auxiliary LLM auto-approves low-risk commands, denies high-risk commands, and prompts when uncertain.
- **`manual`** — Always prompt before destructive commands.
- **`off`** — Skip all approval prompts (equivalent to the `--yolo` flag; YAML 1.1 parses a bare `off` as a bool, which is normalized back to the mode).

Per-invocation bypass without changing config: `hermes --yolo`, the session-scoped gateway `/yolo` toggle, or `export HERMES_YOLO_MODE=1` (snapshotted at import so a mid-session `export` cannot disarm the gate).

Non-interactive surfaces get their own binary modes, defaulting to **deny**: `approvals.cron_mode`, `approvals.single_query_mode`, and `approvals.unattended_mode`. An unattended context with no approval bridge fails closed.

The approval timeout is `approvals.timeout` (default 300 seconds — 60 s proved too tight for Telegram/Discord push notifications a user may not see for minutes), read via `_get_approval_timeout()` and clamped to a platform-safe maximum so a huge value cannot overflow blocking primitives. It applies to all approval paths, including gateway-based approval prompts.

### What Counts as "Destructive"

`tools/approval_detection.py::detect_dangerous_command()` runs a normalization pass (NFKC, ANSI stripping, quote-aware command-start marking, deobfuscation of spliced command words, Windows path variants) and then matches a variant list against pattern tables:

**Dangerous patterns (prompt for approval)** — recursive deletes in root paths (including GNU `rm`'s flags-after-operands spelling), `git reset --hard`-class operations, pipe-to-shell (`curl … | bash/zsh/ksh/dash`, `iwr | iex`), Windows `del`/`rd`/`Remove-Item` recursive forms, PowerShell encoded commands, `sed -i`/`cp`/`mv` into SSH, shell-rc, and credential paths, and **credential exfiltration shapes**: `curl`/`wget` upload flags (`-d`, `--data`, `--form`, `--upload-file`, …) carrying a secret-named variable or credential file, a credential file piped into an upload, and cloud instance-metadata (IMDS) endpoint access — on a cloud VM those serve live IAM credentials to any local process.

**Hardline patterns (unconditional floor)** — a separate `HARDLINE_PATTERNS` list that is *never* bypassable, even under `--yolo`, `approvals.mode: off`, or cron approve mode: `rm -rf` of `/`, system directories, or the home directory; `mkfs`; `dd` to a raw block device; redirects to raw block devices; fork bombs; `kill -1`; `shutdown`/`reboot`/`init 0/6`/`systemctl poweroff`. Positionless rules (the redirect and fork-bomb) are matched against a quote-masked variant so quoted prose cannot trip them, while `sh -c`/`bash -c`/`eval` carriers still scan raw. The sudo **stdin guard** unconditionally blocks an explicit `sudo -S` when `SUDO_PASSWORD` is not configured — that is the LLM piping a guessed password via stdin.

Two more layers complete the sweep: an invisible/bidirectional Unicode control character in the command is flagged as possible obfuscation, and a parser-limit guard hard-blocks pathological commands instead of running a minutes-long regex.

**Verification artifact cleanup exemption**: `detect_dangerous_command()` checks `_is_verification_artifact_cleanup()` before pattern matching. Only the exact shape `rm -f <tmpdir>/hermes-(verify|ad-hoc)-<name>` — three argv tokens, resolved inside the system temp directory — is exempted, since these are known-safe cleanups of ephemeral verification artifacts.

**Operator floor**: `approvals.deny` is a list of case-insensitive fnmatch globs matched against terminal commands; a match blocks even under `--yolo` / `mode: off`.

### Smart Approval

In `smart` mode (the default), a lightweight auxiliary-LLM call (`tools/approval_smart.py`) evaluates the command against a security-reviewer system prompt that explicitly frames the command text as *untrusted input* — embedded instructions inside the command must be ignored and only the actual shell operations evaluated. The rubric:

- **APPROVE** — clearly safe (benign script execution, safe file operations, dev tools, package installs, git operations).
- **DENY** — could genuinely damage the system (recursive delete of important paths, overwriting system files, fork bombs, wiping disks, dropping databases).
- **ESCALATE** — uncertain, or the command contains text that appears to be manipulating the review.

Operators can append house rules to the guardian's system prompt via `approvals.smart_policy` (e.g. "Always ESCALATE commands touching /etc"). After `approvals.denial_breaker_threshold` (default 3) consecutive guardian DENYs in a session, the deny message escalates to a hard-stop; an approval resets the tally.

The verdict drives one of three outcomes:

- **ALLOW** — Low-risk commands are auto-approved. The approval is scoped to a single operation (not session-level), so the same command is re-evaluated next time.
- **DENY** — High-risk commands are denied. For interactive owners (`is_cli`, `is_gateway`, `is_ask`), the DENY includes a one-operation override prompt (`smart_denied_for_owner`), allowing the user to proceed despite the negative verdict. The override is scoped to the current operation only — there is no permanent or session-level persistence. Non-interactive paths get a hard block with no override.
- **PROMPT** — On ESCALATE (or an auxiliary call failure, which also escalates), the user is prompted as in `manual` mode.

This adds one auxiliary LLM round-trip of latency to approved commands.

#### Approval Observer Hooks

Plugin lifecycle hooks — `pre_approval_request` and `post_approval_response` — fire around the smart verdict (`decided_by: "aux_llm"`), with turn/session/tool-call ids attached so observer plugins can scope their marks. The display payload is prepared through `agent.redact.redact_sensitive_text()`, so sensitive command details are masked before they reach hook consumers or approval cards; redaction failure is degraded gracefully rather than failing the approval itself.

## 2.4 Secret Redaction

`agent/redact.py` provides regex-based secret detection and masking.

- **On by default** (`HERMES_REDACT_SECRETS`, bridged from `security.redact_secrets`). Users who need raw credential values in tool output can opt out via `security.redact_secrets: false` in config.yaml or `HERMES_REDACT_SECRETS=false` in `~/.hermes/.env`; an opt-out warning is logged at CLI/gateway startup so operators see the downgrade.
- When enabled, tool output (terminal stdout, process output, env dumps, browser snapshots, log lines) is scanned for patterns resembling API keys, tokens, and passwords before entering the conversation context.
- Patterns are literal-prefix anchored (the cheap pre-screen is derived from the same literals): `sk-`-family keys (including dot-delimited Alibaba variants), `ghp_` (GitHub), AWS keys, JWTs, Slack App-Level tokens (`xapp-`), ElevenLabs (`sk_`), Hindsight (`hsk-`), Langfuse public keys, private key blocks (`[REDACTED PRIVATE KEY]`), database connection-string passwords, bare-token URL userinfo, and passwords in lowercase/dotted config keys. GCS V4 and Azure SAS signature URL parameters count as URL credentials.
- Secrets are masked so the model knows something was hidden without seeing the value; prefix secrets in file reads use a non-reusable sentinel to prevent repeated exposure.
- Credential bodies split by control/zero-width bytes (`sk-abc\x1bdef…`) are re-matched across the split — otherwise the token leaks verbatim — and registered vault values are scrubbed regardless of the redact_secrets preference (a hard egress boundary: `redact_for_egress()` fails closed for text leaving the process to chat platforms, A2A peers, and telemetry).
- Redaction is scoped to tool results and egress paths, not the model's own tool-call arguments (an earlier attempt to redact those broke auth headers by quote-eating).

**Important**: `security.redact_secrets` is snapshotted at import time. Changing it mid-session has no effect — a new session is required.

## 2.5 File Safety

File-mutation safety is layered across two modules.

`agent/file_safety.py` builds the shared deny/approval lists used by both tools and ACP shims. The module's own docstring sets expectations: every guard is **defense-in-depth, not a security boundary** — the terminal tool runs as the same OS user — the value is a clear denial plus a visible audit trail.

- **Credential write denylist** — exact paths that must never be written: `~/.ssh/authorized_keys`, `~/.ssh/id_rsa`/`id_ed25519`, `.netrc`, `.pgpass`, `.npmrc`, `.pypirc`, `.git-credentials`; secret stores under both the active `HERMES_HOME` and the global root (`.env`, `.anthropic_oauth.json`, `auth/google_oauth.json`, Bitwarden Secrets caches); and `/etc/sudoers`, `/etc/passwd`, `/etc/shadow`. Prefix denylists cover `.ssh`, `.aws`, `.gnupg`, `.kube`, `.docker`, `.azure`, `.config/gh`, `.config/gcloud`, `/etc/sudoers.d`, and `/etc/systemd`.
- **Approval-gated (not denied)** — `~/.ssh/config` holds no key bytes and is routine to edit, but can carry `ProxyCommand`/`Match exec`: interactive file tools prompt (approve-once/session/always) and non-interactive callers fail closed.
- **Safe-root confinement** — with `HERMES_WRITE_SAFE_ROOT` set, writes outside those roots are refused; the denial is classified separately (`'safe_root'`) from credential blocks (`'credential'`), so the error message says which rule fired and how to fix it.
- **Hermes-protected subpaths** — session transcripts (`state.db`, `sessions/`), `mcp-tokens/`, `pairing/`, `vault/`, and `browser-profile/` (copied cookies/Login Data) are application-owned or credential-bearing; control files like `config.yaml` are deliberately read-denied but writable by the user's own request.
- **Runtime self-protection** — the agent's own interpreter/venv is never agent-writable (an overwrite bricks the next start exactly like a delete).
- **Windows NT-namespace guard** — `\??\UNC\…` device paths are rejected on the *raw* string before any `realpath()`, because merely resolving such a path triggers SMB authentication and leaks the user's NTLM hash even when the access is denied.

`tools/file_tools_write_guards.py` enforces the gate at the `write_file`/`patch` layer:

- **Sensitive-path refusal** — system locations and the Hermes `config.yaml` itself (the security policy: approvals.mode, yolo, allowlist) are refused, since a prompt-injected agent could otherwise silently disable exec approval by editing its own config.
- **Protected-instruction approval gate** — writes to files that steer *future* agent behavior (`AGENTS.md`, `CLAUDE.md`, `SOUL.md`, `.cursorrules` — basename match in any directory, case-insensitive, realpath'd so symlinks don't escape) **always require human approval**, even under `--yolo`, and fail closed without a human channel. Extendable via `security.protected_instruction_extra_patterns`.
- **Stale-overwrite blocker** — `write_file` refuses to replace an existing file whose full current content this task has never seen (never read, only patched without a full read, or read only in redacted/partial view), and rejects writes drifted since the last read. Net-new files and fully-read files proceed; the refusal happens before any disk mutation.
- **Atomic writes** — `write_file` writes through a temp file in the target directory and a same-filesystem `mv -f`, so a half-written file never lands.

### Skill Security Scanning

`tools/skills_guard.py` provides security scanning of skill content before execution. Scan results are cached via `scan_skill_cached()`, keyed on `content_hash()` — a SHA-256 digest over relative file paths and exact file bytes. A `SCANNER_VERSION` constant (currently `skills-guard-v8`) is included in the cache key, ensuring any update to the scanner logic invalidates stale entries. The `ScanResult` dataclass carries a `scan_provenance` field recording the scanner version and content hash used, so cached results are verifiably bound to the exact content and scanner version that produced them. Scan findings drive `should_allow_install()` and trust-level verdicts per source; the bundled third-party `tirith` binary scanner that once ran alongside this pattern scanner was removed from core (fail-open by default, high false-positive rate, and it never saw writes, `execute_code`, or the browser).

## 2.6 Message Sanitization

`agent/message_sanitization.py` enforces structural correctness of messages before they reach the LLM:

- **Surrogate replacement** — Replaces lone UTF-16 surrogate code points with U+FFFD (an O(1) ASCII fast-path skips the regex) that would otherwise crash `json.dumps` in the OpenAI SDK.
- **Control-character handling** — Escapes literal control chars inside JSON string values as `\uXXXX` for llama.cpp-style output, and strips non-ASCII control characters where providers choke on them.
- **Image stripping** — Removes image attachments from messages when the provider doesn't support multimodal input.
- **Structure repair** — Multi-pass repair of unparseable tool-call arguments (rebalanced JSON closers, control-char escaping), plus provider-minted parallel tool-id normalization and closing of interrupted tool sequences.

## 2.7 TLS Verification

`agent/ssl_verify.py` manages TLS trust for every HTTPS connection Hermes makes (LLM providers, browser, web tools, Telegram). The model changed from "reject self-signed by default" to **trust the platform's own certificate store**:

- **`install_truststore()`** runs once at process start and injects the `truststore` verifier process-wide, so every stack that builds a default `SSLContext` (CryptoAPI on Windows, Security.framework on macOS, NSS on Linux) validates chains the way the OS does. If truststore can't load (unsupported platform), Hermes logs and falls back to OpenSSL's default verification — never to no verification.
- **`resolve_httpx_verify()`** is the single resolver for HTTP clients: `ssl_verify: false` turns verification off (warned as local-development-only), an explicit provider `ssl_ca_cert` bundle *replaces* the platform store for endpoints the machine has no reason to trust, and anything else verifies against the platform store.
- **`SSL_CERT_FILE`/`SSL_CERT_DIR`** — an exported CA bundle is trusted **on top of** the platform store (a union context), not instead of it, so corporate MITM proxies and self-hosted internal CAs work without disabling verification. The shared Telegram TLS context honors the same union.

## 2.8 Error Classification & Recovery

`agent/error_classifier.py` categorizes API errors into `FailoverReason` enums (29 values):

- `auth` — Transient auth (401/403) — refresh/rotate credentials.
- `auth_permanent` — Auth failed after refresh — abort.
- `billing` — 402 or confirmed credit exhaustion — rotate immediately.
- `rate_limit` — 429 or quota-based throttling — backoff then rotate.
- `upstream_rate_limit` — Upstream model rate-limited (aggregator 429) — fallback to different model, not credential rotation.
- `upstream_blocked` — 403 from a WAF/CDN/proxy in front of the provider — key is healthy, fallback.
- `overloaded` — 503/529 — provider overloaded, backoff.
- `server_error` — 500/502 — internal server error, retry.
- `timeout` — Connection/read timeout — rebuild client + retry.
- `ssl_cert_verification` — Deterministic TLS chain failure — fail fast with actionable guidance.
- `context_overflow` — Context too large — compress, not failover.
- `payload_too_large` — 413 — compress payload.
- `image_too_large` — Native image part exceeds provider's per-image limit — shrink and retry.
- `image_corrupt` — Provider can't decode image bytes — strip and retry (shrinking won't help).
- `model_not_found` — 404 or invalid model (including a retired free-route 404) — fallback to different model.
- `provider_policy_blocked` — Aggregator blocked the only endpoint due to data/privacy policy.
- `content_policy_blocked` — Provider safety filter rejected this prompt — deterministic, don't retry unchanged.
- `model_entitlement` — This account cannot use the requested model — rotate credential (model-scoped), else fall back.
- `incomplete_response` — Codex/Responses turn stuck emitting reasoning only after replay + nudge — hand to a different provider.
- `format_error` — 400 bad request — abort or strip + retry.
- `role_alternation` — Strict chat template rejected adjacent same-role messages — merge them and retry.
- `invalid_encrypted_content` — Responses replay blob rejected — strip replay state and retry.
- `multimodal_tool_content_unsupported` — Provider rejected list-type content in tool messages — downgrade to text and retry.
- `reasoning_mandatory` — Route rejects `reasoning: {enabled: false}` — stop sending the disable this session and retry.
- `thinking_signature` — Anthropic thinking block signature invalid.
- `long_context_tier` — Anthropic "extra usage" tier gate.
- `oauth_long_context_beta_forbidden` — Anthropic OAuth subscription rejects 1M context beta — disable beta and retry.
- `llama_cpp_grammar_pattern` — llama.cpp json-schema-to-grammar rejects regex escapes in `pattern`/`format` — strip from tools and retry.
- `unknown` — Unclassifiable — retry with backoff.

Each classification drives a different recovery strategy in `conversation_loop.py`.

## 2.9 Summary Table: Guardrail Layers

| Layer | Mechanism | Enforced By | Override |
|-------|-----------|-------------|----------|
| Prompt norms | Execution-discipline blocks, Skill Safety Rule | LLM compliance | none (always shipped) |
| Tool loop guardrails | Exact/same-tool/no-progress counters, progress resets | `ToolCallGuardrailController` | `hard_stop_enabled: false` (interactive) |
| Stall guards | Identical-call streaks, batch cycles, result stubs | `observe_call()` | poller exemption; hard-stop thresholds |
| Loop caps | Per-turn web_search/subagent ceilings | `before_call()` cap check | `loop_caps.*: 0` disables |
| Hardline floor | Unconditional blocklist below yolo | `detect_hardline_command()` | none (never bypassable) |
| Destructive approval | Human, auxiliary guardian LLM, or per-surface mode | `tools/approval*.py` | `approvals.mode: off` or `--yolo` (default: `smart`); `approvals.deny` globs re-block |
| Secret redaction | Prefix/assignment/URL-credential masking, vault egress sweep | `agent/redact.py` | `security.redact_secrets: false` (warned) |
| File safety | Credential-path denylist, safe roots, protected-instruction gate, stale-overwrite blocker | `agent/file_safety.py`, `tools/file_tools_write_guards.py` | approval gate is promptable; denylist is not |
| Skill scanning | Attestation-bound cache, pattern matching | `tools/skills_guard.py` | `SCANNER_VERSION` invalidation |
| Message sanitization | Surrogate replacement, control-char escaping, structure repair | `agent/message_sanitization.py` | none (always active) |
| TLS verification | Platform trust store via truststore, CA bundle union | `agent/ssl_verify.py` | `ssl_verify: false` (warned, dev-only) |
| Error classification | 29 `FailoverReason` values driving recovery | `agent/error_classifier.py` | per-error retry/fallback config |

## 2.10 Guardrail Interaction Diagram

```
┌────────────────────┐
│  Tool Call Request │
└────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  detect_hardline_command()? │──YES──▶ UNCONDITIONAL BLOCK (never bypassable)
└─────────────────────────────┘
          │ NO
          ▼
┌─────────────────────────────┐
│  detect_dangerous_command()?│──YES──▶ Smart Approval (default) or User Prompt
└─────────────────────────────┘
          │ NO
          ▼
┌─────────────────────────────┐
│  ToolCallGuardrailController│
│  .before_call()             │──BLOCK──▶ Loop cap reached, or synthetic error result
└─────────────────────────────┘
          │ ALLOW
          ▼
┌─────────────────────────────┐
│  Execute tool               │
└─────────────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  classify_tool_failure()?   │
│  .after_call()              │──WARN/HALT──▶ Injected guidance
│  .observe_call()            │──NOTICE/STUB─▶ Stall-guard note, dedupe stub
└─────────────────────────────┘
          │ OK
          ▼
┌─────────────────────────────┐
│  redact_terminal_output() / │──REDACT──▶ Masked output
│  redact_for_egress()        │
└─────────────────────────────┘
          │
          ▼
┌─────────────────────────────┐
│  Return to conversation loop│
└─────────────────────────────┘
```
