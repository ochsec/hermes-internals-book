# Architecture & The Conversation Loop

The Hermes agent is built around a single, long-running **conversation loop** that drives every interaction — whether through the CLI, a messaging gateway, a cron job, or a delegated subagent. This chapter describes the loop's structure, the message flow within it, how the system prompt is constructed, and the key invariants that keep the system stable.

## 1.1 The Core Loop

The canonical entry point is `AIAgent.run_conversation()`. The `AIAgent` class in `run_agent.py` (~1,600 lines) is now a composition of mixins (`TurnFacadeMixin`, `CompressionFacadeMixin`, `SessionPersistenceMixin`, `InterruptControlMixin`, ...), with the run/chat entry point defined in `agent/turn_facade.py`. That facade handles **turn admission** before the loop runs: a durable cross-process session turn lease with a refresher thread and liveness watchdog (`agent/turn_facade_lease.py`), relay/accounting/portal scopes, and balanced start/finish marks. The loop itself lives in `agent/conversation_loop.py` (~1,800 lines) as a module-level `run_conversation(agent, ...)` wrapping the actual turn in `_run_conversation_turn()`.

At its heart, the loop is a **single-pass turn processor** with tool-result reinjection:

```text
┌─────────────────────────────────────────────────────────────┐
│  USER MESSAGE arrives (CLI input, gateway payload, etc.)  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  BUILD SYSTEM PROMPT                                        │
│  ── identity, environment hints, active skills, tool        │
│     schemas, memory context, guardrails                     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  LOOP while iterations < max_turns (default 90)               │
│                                                             │
│  a. CALL LLM with messages + tool schemas                   │
│                                                             │
│  b. IF response contains tool_calls                         │
│        → dispatch each tool via handle_function_call()       │
│        → append tool results as assistant/tool messages        │
│        → CONTINUE (next iteration)                           │
│                                                             │
│  c. IF text response                                         │
│        → finalize turn (compress, charge, memory review)     │
│        → RETURN text to caller                               │
│                                                             │
│  d. IF error (API failure, rate limit, context overflow)     │
│        → classify error → retry or fallback                  │
│        → if fatal, abort loop                                │
└─────────────────────────────────────────────────────────────┘
```

The loop is **not** a planning-then-executing two-pass system. The LLM sees the full conversation history (including prior tool results) on every iteration and decides, in a single call, whether to invoke more tools or emit final text. This is architecturally important: it preserves **prompt caching** (the system prompt and early turns are reused as a cached prefix) and avoids the latency of a separate planning phase.

### The Phased Loop

The loop body is decomposed into **named phase functions**, each in its own `agent/turn_*.py` module, driven by a `_run_phase()` dispatcher over a shared `_LoopState` dataclass. Every phase returns a verdict — `"continue"`, `"break"`, or `"return"` (with the turn result) — so the loop reads as a flat pipeline of decision points rather than one monolith:

| Phase | Module | Role |
|-------|--------|------|
| `begin_iteration`, `prepare_iteration`, `announce_api_call` | `agent/turn_iteration_prep.py` | Per-iteration setup, budget checkpoint warning, status output |
| `assemble_api_request` | `agent/turn_request_assembly.py` | Build messages and request kwargs for the API call |
| `run_preflight_gate` | `agent/turn_preflight_gate.py` | Pre-flight compression and abort checks before the call |
| `_run_api_retry_loop` | `agent/conversation_loop.py` | Retry, fallback, and error classification around the call |
| `normalize_model_response` | `agent/conversation_loop.py` | Response normalization, stale-stream checks |
| `run_tool_round` | `agent/turn_tool_round.py` | Validate → persist-before-execute → dispatch tools → post-tool compression |
| `finish_text_response` | `agent/turn_final_response.py` | Final-text path |
| `handle_outer_loop_error` | `agent/turn_loop_errors.py` | Outer-loop error classification → verdict |

Two invariants stand out. Inside `run_tool_round`, the tool-call assistant message is persisted **before any tool side effect** — a crash or a destructive-tool restart of Hermes must leave a resume point. And post-loop finalization is its own module: `agent/turn_finalizer.py::finalize_turn()` runs everything after the loop exits (see §1.4).

When `api_mode == "codex_app_server"`, the whole turn is handed to the Codex app-server subprocess (`_run_codex_app_server_turn` → `agent/codex_runtime.py`). If that runtime fails, `agent/turn_recovery.py::activate_codex_app_server_fallback` rewrites provider/model/api_mode and retries the same user turn on the generic loop, keeping Codex's projected rows in the turn's accounting.

### Key Parameters

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `agent.max_turns` | None (unlimited) | Turn cap. Caps caused silent mid-task truncation, so the default was removed; positive ints cap via `resolve_turn_limit()`, and `"none"`/`"unlimited"`/`"inf"`/`0`/`-1` also mean unlimited |
| `agent.budget_warning_ratio` | None | One-time model-visible checkpoint warning before a finite turn cap is exhausted (ratio strictly between 0 and 1) |
| `agent.run_budget_seconds` | None | Wall-clock budget per run: one-time wrap-up notice at 80% elapsed; provider stale timeouts capped to the remaining budget |
| `tool_use_enforcement` | "auto" | "auto" enforces tool use for the `codex_responses` api_mode only; `true` forces it for all models |
| `delegation.max_iterations` | 250 | Per-subagent iteration cap (own budget, independent of the parent's) |
| `context_length` | model-dependent | Token budget before compression fires |
| `compression.threshold` | 0.50 | Fraction of context filled before summarization |
| `compression.target_ratio` | 0.20 | How much to shrink context when compressing |
| `compression.protect_first_n` | 3 | Head messages never summarized (in addition to system prompt) |
| `compression.protect_last_n` | 20 | Recent messages always preserved (~10 full turns) |
| `compression.codex_gpt55_autoraise` | true | Raise threshold to 85% for Codex GPT-5.5 |
| `compression.codex_app_server_auto` | native | Codex app-server thread-compaction mode |
| `prompt_caching.cache_ttl` | "5m" | Anthropic cached-prefix TTL ("5m", "1h", or "auto") |

## 1.2 System Prompt Construction

The system prompt is assembled by `agent/system_prompt.py` (`build_system_prompt_parts`), which superseded the older `agent/prompt_builder.py` as the primary entry point. `prompt_builder.py` still provides the component constants and helpers, but the top-level assembly lives in `system_prompt.py`. The prompt is **byte-stable for the lifetime of a conversation** — only context compression triggers a rebuild (`invalidate_system_prompt()`, which also reloads memory from disk and clears the frozen plugin-section snapshot, stashing the previous bytes as a fail-open fallback if a plugin's re-render raises). This stability is sacred — it enables prompt caching on providers like Anthropic and reduces cost.

### Three-Tier Architecture

The system prompt is organized into three tiers, joined with `\n\n` and ordered for cache friendliness:

1. **`stable`** — Identity (SOUL.md or `DEFAULT_AGENT_IDENTITY`), Hermes-agent help guidance (with a `skill_view`-pointer variant chosen after the skills index is rendered), task-completion / parallel-tool-call / tool-aware behavioral guidance, steer-channel note, platform hints, pinned auto-loaded skills, the Alibaba model-name workaround, and the coding **operating brief** (posture only — see below). Pinned skills are per-agent constants resolved once, so they belong in the stable prefix.

2. **`context`** — Caller-supplied `system_message`, context files (AGENTS.md, .cursorrules, etc.) discovered under the resolved context cwd, and the coding **workspace snapshot**. Context files use a **dynamic cap** (`_dynamic_context_file_max_chars`) that scales to the model's context window rather than a flat default. Worktree-dependent blocks are deliberately placed *after* project context so a shared context file can remain in the longest common prefix across worktrees.

3. **`volatile`** — The **skills index leads this band**: skills are runtime-mutable, so on a longest-prefix backend an unchanged index stays inside the reused prefix while a changed one re-prefills from here. Followed by memory parts (MEMORY.md snapshot, USER.md profile, external memory provider block), frozen plugin prompt sections (confined to one coarse anchor so a resumed process can reconstruct the stable prefix without re-running plugins), the active-profile line, the timestamp line (date-only for byte stability), and the renderer-owned runtime environment hints anchor.

### Tool Schemas Are Not in the Prompt Text

The book's earlier edition listed "tool schemas" as a system prompt component. In the current architecture, tool schemas are sent as the `tools` parameter in the API call, not embedded in the system prompt text. This distinction matters for caching: the `tools` array is part of the API request but not part of the text prefix.

The assembled prompt is sent on **every** LLM call within the conversation. Changing tools, skills, or config mid-conversation would invalidate the cache, so Hermes requires a `/reset` (new session) for such changes to take effect.

### Prompt Caching TTL

When using Anthropic (native or via OpenRouter), the system prompt prefix is eligible for provider-side caching. The `prompt_caching.cache_ttl` config option controls the cache lifetime: `"5m"` (default) or `"1h"` for long sessions with pauses between turns. This does not change the prompt content — it only tells the provider how long to keep the cached prefix.

## 1.3 Message Flow & Role Alternation

Hermes enforces **strict message role alternation**: no two assistant messages or two user messages may appear consecutively in the message array sent to the LLM. This is required by many provider APIs and is actively guarded in `agent/message_sanitization.py`.

When a tool call is made, the flow is:

```text
ASSISTANT message with tool_calls[]
   │
   ├──► TOOL message with role="tool" (result of tool execution)
   │
   └──► (repeat for each parallel tool)
   │
ASSISTANT message (next LLM call)
```

If multiple tools are called in parallel (see §1.5), each gets its own `role="tool"` message in the history.

## 1.4 The Turn Lifecycle

A "turn" is one user message plus however many loop iterations it takes to get a final text response. The lifecycle is managed by `agent/turn_context.py` (~1,330 lines) and `agent/turn_finalizer.py` (~800 lines):

1. **Pre-turn** — `build_turn_context()` gathers ambient state: source platform (cli, telegram, discord, ...), session ID, user ID, thread/topic ID, loaded skills, and the current model/provider. It also warms the environment probe off-thread, handles memory prefetch and persist overrides, and runs **turn-start compaction** (`agent/turn_context_compaction.py::run_turn_start_compaction`). Ordering matters: the DB session row is created only *after* the system prompt is built (else it persists `system_prompt=NULL` and costs a cache miss) and *before* preflight compression.
2. **Execution** — The loop runs as described above.
3. **Finalization** — `turn_finalizer.py::finalize_turn()` handles:
   - Token usage tracking and cost estimation (`agent/usage_pricing.py`)
   - Trajectory / scratchpad persistence (`agent/trajectory.py`)
   - Background memory review nudge (`agent/background_review.py`)
   - Session state sync to SQLite (`hermes_state.py`)
   - Compression check — if context is near the threshold, `agent/context_compressor.py` or `agent/conversation_compression.py` fires.

### Iteration Budget

The loop enforces iteration limits via `agent/iteration_budget.py`, a thread-safe per-agent consume/refund counter. The parent's cap resolves from `agent.max_turns` — **unlimited by default** (the constructor default is `sys.maxsize`, and config `null` means unlimited, because finite caps caused silent mid-task truncation); each subagent's cap is `delegation.max_iterations` (default 250), so total iterations across parent + subagents can exceed the parent's cap. The `execute_code` tool gets iterations refunded because its internal Python loop is conceptually part of a single tool call.

A `_budget_grace_call` escape hatch allows one extra iteration when the budget is exhausted, preventing hard cutoff on the edge case where a final API call is in-flight. Optional guards soften finite caps rather than restoring them: `agent.budget_warning_ratio` injects a one-time model-visible checkpoint warning before the cap is exhausted, and `agent.run_budget_seconds` wraps the run with a wall-clock budget (wrap-up notice at 80% elapsed; `_maybe_inject_run_budget_wrapup` in the loop).

### Coding Context

When Hermes detects it is operating in a code workspace (git repo, project directory), `agent/coding_context.py` (~570 lines) produces the coding brief. The **operating brief** (project posture, branch state awareness) lands in the stable tier, while the **workspace snapshot** (structure, cwd/backend) is placed in the context tier, after project context files — the environment block contains the current cwd and would otherwise invalidate the shared prefix across worktrees. This gives the model awareness of the project without requiring explicit tool calls.

## 1.5 Parallel & Segmented Tool Execution

Hermes supports parallel tool dispatch when the model emits multiple tool calls in one response. The parallelism is governed by `agent/tool_dispatch_helpers.py::_should_parallelize_tool_batch()`, which inspects the tool names for data dependencies. A mixed batch (some tools parallel-safe, some not) is no longer run all-or-nothing: `agent/tool_executor.py::execute_tool_calls_segmented()` splits it into ordered **segments** using the `(kind, calls)` plan from `_plan_tool_batch_segments()` — parallel segments run on the worker pool, sequential segments one call at a time — preserving per-call result order. Turn-end work (budget enforcement, `/steer` injection) runs exactly once for the whole batch; each segment re-checks the interrupt flag, so an interrupt drains the remaining segments with one result per call.

Batched calls issued through the `tool_search` dispatch tool get the same treatment: `agent/tool_call_batches.py::expand_local_tool_batches()` splits a registry batch into individual calls **before persistence and execution**, so every local entry travels the ordinary agent path (inline tools, guardrails, per-server concurrency policy) instead of a registry-only dispatcher that would lose them. Connector-only batches keep their existing dispatcher and wire shape, and native providers (Anthropic, Bedrock) replay the replacement content blocks in place to preserve signed thinking/text blocks and their order.

Rules:
- **Always parallel** if tools operate on disjoint resources (e.g., `web_search` + `read_file` on unrelated paths).
- **Serial** if tools mutate the same file or directory (detected by `_paths_overlap()`).
- **MCP parallel safety** — `_is_mcp_tool_parallel_safe()` checks the MCP server's config to determine if parallel calls are supported. MCP servers must explicitly opt in (`supports_parallel_tool_calls: true`).
- **Destructive commands** (e.g., `rm -rf`, `git reset --hard`) require user approval before execution unless `--yolo` mode is active.

The maximum worker pool size is `_MAX_TOOL_WORKERS = 8`, now defined in `agent/tool_executor.py` (relocated from `tool_dispatch_helpers.py`). Concurrent execution uses a `DaemonThreadPoolExecutor` (from `tools/daemon_pool.py`) instead of stdlib `ThreadPoolExecutor` to avoid non-daemon threads hanging at process exit.

## 1.6 Context Compression & the Context Engine

Context management near the token limit is abstracted behind **pluggable context engines** (`agent/context_engine.py`). The `context.engine` config selects the implementation — `"compressor"` (default: the built-in lossy summarizer) or a plugin name installed under `plugins/context_engine/<name>/` or `~/.hermes/plugins/`. An engine's lifecycle is `on_session_start()` → per-API-response `update_from_response()` → per-turn `should_compress()` / `compress()` → `on_session_end()` at **real session boundaries only** (CLI exit, `/reset`, gateway expiry), never per-turn. Engines also decide how compaction is surfaced: the host-visible status goes through `automatic_compaction_status_message()`, which an engine can silence (`emit_automatic_compaction_status = False` — "quiet compaction") or customize (`get_automatic_compaction_status_message(...)`).

The default engine wraps two cooperating implementations:

- **`agent/context_compressor.py`** — The token-counting compressor: thresholds, protected zones, prune passes, and the summary generation.
- **`agent/conversation_compression.py`** — The LLM-driven summarizer that condenses blocks of conversation into dense summaries, plus the compression lock/fence machinery.

Compaction can now fire at **several points, not only between turns**:

- **Turn start** — `run_turn_start_compaction()` in `agent/turn_context_compaction.py`, invoked from `build_turn_context()`.
- **After tool rounds** — `agent/turn_preflight.py::compress_after_tool_results()`. Pressure is measured from the API-reported `prompt_tokens` (a tight lower bound; rough estimates inflate thinking models), with a deterministic **tool-result-only prune** as fallback when full compression is blocked — committed only when the engine returns a new list, never by rebuilding `conversation_history` in place.
- **Between turns** — the original finalization-time check.

### Micro-Compaction

`agent/micro_compaction.py` implements a rolling per-exchange summarization that folds old user/assistant exchanges into a single summary marker between turns. It is **off by default**: every pass rewrites the prompt prefix and breaks the provider prompt cache.

### Native Server-Side Compaction

`agent/native_compaction.py` arms OpenAI Responses **server-side** compaction on supported routes (`context_management=[{"type": "compaction", "compact_threshold": N}]`): the server summarizes older context into an opaque `compaction` item once input crosses N tokens. Deliberately narrow — gpt-5.6 on api.openai.com or the ChatGPT Codex backend, plus gpt-6-astra on official Codex OAuth — with the local compressor kept armed as fallback (the native threshold is clamped below the local trigger so the server gets the first shot). Compaction items ride the `codex_reasoning_items` sidecar.

### Small-Context Threshold Floor

Models with context windows below 512K tokens (`_SMALL_CTX_WINDOW_LIMIT = 512_000`) are floored at a 0.75 (75%) trigger threshold — raise-only. This prevents compaction from firing when half the window is still free, which caused "compaction thrash" on smaller models. Models at or above 512K keep the configured threshold.

### Protected Messages

Two protection zones prevent compression from eating critical context:

- **`protect_first_n`** (default 3) — Head messages (in addition to the system prompt) are never summarized. This preserves the initial request and first response for session framing.
- **`protect_last_n`** (default 20) — The most recent ~10 full turns are always preserved intact.

### Codex App-Server Compaction

When running against the Codex app-server (the Codex CLI runtime), Hermes cannot shrink the real thread context — the app server owns it. The `compression.codex_app_server_auto` config controls the mode:

- `native` (default) — Let Codex decide when to compact its own thread.
- `hermes` — Let Hermes' threshold trigger Codex thread/compact/start.
- `off` — Hermes will not auto-trigger; Codex may still compact natively.

Additionally, `compression.codex_gpt55_autoraise` (default true) raises the Hermes compaction trigger to 85% for the ChatGPT Codex OAuth route, since the backend caps the effective window at 272K.

### Reasoning Trace Exclusion

Reasoning fields (`reasoning`, `reasoning_content`, `reasoning_details`, `codex_reasoning_items`) are excluded from the compression summary input and from token-count estimates. This prevents large reasoning blobs from inflating the apparent context size and triggering premature compression. Inline `<think>`/`<reasoning>` blocks are stripped from assistant content before serialization to the summarizer, and the summarizer's own output is stripped of think blocks before storage.

### No Summary Output Cap

The compression summary call no longer sets `max_tokens` on the API request. The output cap was truncating summaries on Anthropic and NIM paths — thinking models burn the cap on internal reasoning, leaving the actual summary empty. The summary budget is now enforced via prompt-level guidance only, with a ceiling of 10,000 tokens (`_SUMMARY_TOKENS_CEILING`).

### Compression Lock

A lease-based compression lock prevents two compressors from racing on the same session (e.g. parent agent + background-review fork both deciding to compress). The lock is now a **cross-process SQLite lease** (`hermes_state_compression.py::try_acquire_compression_lock()`): a 300-second TTL, holder-qualified release (`DELETE ... WHERE holder = ?`), and transparent reclaim of expired locks plus structured holders whose recorded PID is provably dead — so a crashed process cannot wedge the session, and PID reuse can never steal a live lease. On top of the lease, `agent/conversation_compression.py` adds a cancellation fence (`retain_compression_lock_until_worker_done()` / `release_cancelled_compression_lock()`) so a cancelled compressor cannot release the lease while a worker it spawned is still writing.

## 1.7 Provider Abstraction Layer

Hermes supports 20+ LLM providers. The provider abstraction lives in `agent/` with per-provider adapters:

- `agent/anthropic_adapter.py`
- `agent/bedrock_adapter.py`
- `agent/gemini_native_adapter.py`
- `agent/vertex_adapter.py` (Google Vertex AI for Gemini, OAuth2)
- `agent/codex_responses_adapter.py`
- `agent/azure_identity_adapter.py`
- ...and many more

A provider is selected by `model.provider` in config. The loop itself is provider-agnostic; per-wire-format **transports** (`agent/transports/` — chat_completions, anthropic, bedrock, codex, codex_app_server, ...) and adapters handle schema translation (OpenAI tools ↔ Anthropic tools ↔ Gemini function calling), token counting, and streaming response parsing. Custom providers can also specify extra HTTP headers via config (`model.extra_headers`) for proxy gateways and enterprise endpoints.

## 1.8 Streaming vs Non-Streaming

The loop supports both streaming (token-by-token delivery to the user) and non-blocking batch mode. Streaming is the default for CLI and gateway; non-streaming is used for cron jobs, delegation, and some gateway platforms where partial output would be confusing.

Streaming is handled by `agent/stream_diag.py` and provider-specific streaming parsers. Tool calls arriving in a stream are buffered until the full JSON argument blob is received, then dispatched.

### Stale-Stream Circuit Breaker

A cross-turn safety mechanism in `agent/chat_completion_helpers.py` detects consecutive stale streams (where the stream completes without producing usable output). A `_consecutive_stale_streams` counter on the agent is incremented on each stale stream and checked by `_check_stale_giveup()`. If too many consecutive stale streams occur, the loop aborts rather than burning retries. The counter resets on model switch and fallback activation.

### Reasoning-Only Answer Promotion

Some routes return a "clean stop" whose entire payload is private chain-of-thought in a reasoning field. Whether that reasoning-only response may be **promoted to the visible answer** is decided by `agent/reasoning_promotion.py::answer_in_reasoning_capability()`. Because a field name is not provenance (OpenRouter, DeepSeek, Bedrock and several adapters all put CoT in `reasoning`), promotion is limited to **trusted routes**: a per-model or provider-level `answer_in_reasoning` capability opt-in on the agent's *live* route, or the local Nemotron parser route. The trust check is re-derived on every call, so fallback activation, primary restore, and model switches never carry a decision to another route — OpenRouter and non-chat-completions transports are never trusted.

## 1.9 Interrupt Handling

Users can interrupt a running turn with Ctrl-C (CLI) or a platform-specific cancel signal. The interrupt propagates via `tools.interrupt.set_interrupt()`, which sets a flag checked between loop iterations. If an interrupt fires while waiting for the LLM, the string `"Operation interrupted: waiting for model response (...` is emitted and the loop exits cleanly.

When an interrupt fires, the agent also calls `_active_request_abort("interrupt_abort")` if an active HTTP request is in-flight. This cross-thread abort shuts down the active sockets promptly rather than waiting for the API to return. Cron turns, which perform their API request on the conversation thread (to avoid a nested interrupt-worker deadlock), also register their client so the cross-thread interrupt can reach them.

### Active-Turn Redirect

When the user steers mid-turn (an interrupt with a new instruction) and the provider supports it (`_supports_active_turn_redirect`), the loop repairs the transcript with `_apply_active_turn_redirect()` in `agent/conversation_loop.py` rather than dropping the partial response. It appends an **assistant placeholder** (the stripped visible text so far, or a hidden placeholder if empty) followed by the **user correction**, keeping strict role alternation. Invariants: raw chain-of-thought never enters replayable content (inlined CoT reads as a prefill jailbreak); the interruption scaffold is carried only in the user correction's `api_content`, which the pre-call sanitizer substitutes back on replay — so the transcript the user sees shows their own words while the provider replays the scaffolded form. A visible looped-repetition shape is dropped entirely instead of being replayed.

### Persistence Lock

Session persistence (`_commit_session_messages` and `_flush_messages_to_session_db`) now holds a per-agent `_session_persist_lock`. Close and turn-start persistence can run on separate CLI threads; without the lock, both could observe the same unmarked message dict and write duplicate durable rows. The lock is acquired as a critical section covering the scaffolding removal, session message assignment, JSONL save, and SQLite flush.

### Multimodal Content Override

The persistence override logic distinguishes text-only overrides from list overrides. A plain-text override replaces the live message content before the API call only when the content is not multimodal (not a list). A list override, however, is the original clean multimodal payload (e.g., before a queued `/model` note) and must replace the API-local list once the turn is final. This preserves image/audio blocks in the live messages list during the API call while still persisting the clean version.

### Fallback Notice

When a provider/model fallback succeeds, `_emit_pending_fallback_notice()` surfaces a one-shot status notice. The switch is a durable state change operators must see, unlike transient retry chatter that `_clear_status_buffer` drops. The notice is emitted exactly once via `_emit_status` and then cleared. On terminal failure, the buffered switch line is flushed instead (via `_flush_status_buffer`), so the user always sees the switch once.

### Kanban Stop Scaffolding

A `_kanban_stop_synthetic` flag was added to `_EPHEMERAL_SCAFFOLDING_FLAGS`. This prevents kanban worker exit narration (a worker that exits without `kanban_complete`/`block`) from surviving into the real transcript and breaking prompt-prefix cache reuse on later turns.

## 1.10 Mixture-of-Agents Turns

The `/moa` slash command marks a single user turn as MoA-enabled. The runtime lives in `agent/moa_loop.py` (~1,550 lines), and its design constraint is that **the normal agent loop still owns tool calling and turn termination** — the module only gathers context from reference-model advisors via the auxiliary client before each model iteration and injects a labelled guidance block into the aggregator prompt. Supporting modules: `agent/moa_alternation.py` (merges advisor feedback while keeping strict role alternation — strict providers reject MoA-shaped message sequences), and `agent/moa_trace.py` (trace files for the fan-out).

Advisor outputs pass a dedicated privacy filter (`moa.privacy_filter: '' | display | full`) layered on top of the central redactor (`agent.redact`): advisor echoes can surface user PII — emails, formatted phone numbers — into UI blocks, trace files, and (in `full` mode) the guidance block injected into the aggregator prompt. Secret/credential shapes remain the central redactor's job; the MoA filter only covers the PII classes the redactor deliberately leaves alone for log/tool output.

## 1.11 Auxiliary Model Calls

Every LLM call that is *not* the main conversation loop routes through the auxiliary runtime — `agent/auxiliary_client.py`, the largest module in `agent/` at ~8,400 lines, with topical siblings (`auxiliary_stream_watchdog.py`, `auxiliary_health.py`, `aux_accounting.py`, `auxiliary_codex_response.py`, ...). Auxiliary slots serve memory review, conversation titles, MoA advisors, compaction summaries, goal judging, and more.

Key mechanisms:

- **Slot routing with inheritance** — auxiliary tasks can inherit a base slot at read time; an inherited base follows its parent until the user pins one, and write-only empties never shadow it.
- **Stream watchdogs** — `agent/auxiliary_stream_watchdog.py` cuts only its own unfinished stream at the no-progress window (`providers.<id>.stale_timeout_seconds`), shutting the attempt's socket down rather than closing the stream.
- **Stall fallback** — a stall-fallback pin **replaces** the main route, never merges over it; a failing summary model is benched with a deadline rather than blocking compaction.
- **Route-aware normalization** — Codex/Responses-shaped auxiliary responses are normalized in `agent/auxiliary_codex_response.py`: leak rescue keeps completed aux text that resembles a leaked tool call, treats unfinished non-server-side items as partials, and preserves completion status and phase.
- **Outcome accounting** — `agent/aux_accounting.py` and `agent/auxiliary_call_outcome.py` record each auxiliary model call with its real outcome; the main-credential inheritance borrows on the same terms as the client build and only across identical origins.

## 1.12 The Relay Observability Runtime

Hermes owns profile-scoped **NeMo Relay runtimes** for structured tracing and metrics (`agent/relay_runtime.py`, `agent/relay_llm.py`, `agent/relay_tools.py`, `agent/relay_cwd.py`). Relay models observation in three scopes — `hermes.session`, `hermes.turn`, and `hermes.logical_llm_call` — pushed around a context-scoped pipeline; each physical provider attempt is adapted through per-api_mode codec adapters in `relay_llm.py`.

Two design points matter for the loop's stability:

- **Stream handoff concurrency** — Relay invokes its collector for every post-intercept chunk and finalizes as soon as the provider stream ends, concurrently with Hermes' consumer thread. The finalizer therefore builds the recorded response from collector-observed state only, never from the consumer loop's closures (`agent/chat_completion_helpers_relay.py`), and each streamed chunk is handed over before the provider sends the next so concurrent provider streams keep draining.
- **Bounded scope ops** — native scope operations are gated by a 10-second bound: a wedged pipeline costs one lost span, never a blocked agent. Session scopes rotate on compaction or `max_turns` exhaustion, and task-run metrics (`start_task_run` / `finish_task_run` in `hermes_cli.observability`) bracket each turn.

## 1.13 Summary Diagram

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  User Message   │────▶│  System Prompt   │────▶│   LLM Call      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
                                                           │
                              ┌────────────────────────────┼────────────────────────────┐
                              │                            │                            │
                              ▼                            ▼                            ▼
                        ┌──────────┐              ┌──────────────┐               ┌──────────┐
                        │  Text    │              │  Tool Calls  │               │  Error   │
                        │  Done    │              │  Dispatch    │               │  Retry   │
                        └──────────┘              └──────────────┘               └──────────┘
                                                          │
                                                          ▼
                                              ┌────────────────────┐
                                              │  Results injected  │
                                              │  into next LLM call│
                                              └────────────────────┘
```

The loop continues until either text is returned, the iteration budget is exhausted, or an unrecoverable error occurs.
