# Architecture & The Conversation Loop

The Hermes agent is built around a single, long-running **conversation loop** that drives every interaction — whether through the CLI, a messaging gateway, a cron job, or a delegated subagent. This chapter describes the loop's structure, the message flow within it, how the system prompt is constructed, and the key invariants that keep the system stable.

## 1.1 The Core Loop

The canonical entry point is `AIAgent.run_conversation()` in `run_agent.py`. Despite the file's ~6,000 lines, the loop itself has been extracted into `agent/conversation_loop.py` (~5,300 lines) and is invoked as a thin forwarder from `run_agent`.

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

### Key Parameters

| Parameter | Default | Meaning |
|-----------|---------|---------|
| `max_turns` | 90 | Hard iteration ceiling per user message |
| `tool_use_enforcement` | true | Require model to emit tools when schema is present |
| `context_length` | model-dependent | Token budget before compression fires |
| `compression.threshold` | 0.50 | Fraction of context filled before summarization |
| `compression.target_ratio` | 0.20 | How much to shrink context when compressing |
| `compression.protect_first_n` | 3 | Head messages never summarized (in addition to system prompt) |
| `compression.protect_last_n` | 20 | Recent messages always preserved (~10 full turns) |
| `compression.codex_gpt55_autoraise` | true | Raise threshold to 85% for Codex GPT-5.5 |
| `compression.codex_app_server_auto` | native | Codex app-server thread-compaction mode |
| `prompt_caching.cache_ttl` | "5m" | Anthropic cached-prefix TTL ("5m" or "1h") |

## 1.2 System Prompt Construction

The system prompt is assembled by a new `agent/system_prompt.py` module (`build_system_prompt_parts`), which superseded the older `agent/prompt_builder.py` as the primary entry point. `prompt_builder.py` still provides component helpers, but the top-level assembly now lives in `system_prompt.py`. The prompt is **byte-stable for the lifetime of a conversation** — only context compression triggers a rebuild (`invalidate_system_prompt()`). This stability is sacred — it enables prompt caching on providers like Anthropic and reduces cost.

### Three-Tier Architecture

The system prompt is organized into three tiers, joined with `\n\n` and ordered for cache friendliness:

1. **`stable`** — Identity (SOUL.md or `DEFAULT_AGENT_IDENTITY`), Hermes-agent help guidance, task-completion guidance, parallel-tool-call guidance, tool-aware behavioral guidance (memory/session_search/skills/kanban), steer-channel note, computer-use guidance, Nous subscription block, tool-use enforcement with per-model operational guidance (Google models, OpenAI/xAI models), skills prompt, Alibaba model-name workaround, environment hints, coding-context system blocks, environment probe line, active-profile hint, platform hints.

2. **`context`** — Caller-supplied `system_message` plus context files (AGENTS.md, .cursorrules, etc.) discovered under `TERMINAL_CWD`. Context files now use a **dynamic cap** (`_dynamic_context_file_max_chars`) that scales to the model's context window rather than a flat default.

3. **`volatile`** — Memory snapshot (MEMORY.md), USER.md profile, external memory provider block, timestamp line (date-only for byte stability), session ID, model, provider.

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

A "turn" is one user message plus however many loop iterations it takes to get a final text response. The lifecycle is managed by `agent/turn_context.py` (~565 lines) and `agent/turn_finalizer.py` (~507 lines):

1. **Pre-turn** — `build_turn_context()` gathers ambient state: source platform (cli, telegram, discord, ...), session ID, user ID, thread/topic ID, loaded skills, and the current model/provider. It also warms the environment probe off-thread, creates the DB session row (deferred to after system prompt build), and handles memory prefetch and persist overrides.
2. **Execution** — The loop runs as described above.
3. **Finalization** — `turn_finalizer.py` handles:
   - Token usage tracking and cost estimation (`agent/usage_pricing.py`)
   - Trajectory / scratchpad persistence (`agent/trajectory.py`)
   - Background memory review nudge (`agent/background_review.py`)
   - Session state sync to SQLite (`hermes_state.py`)
   - Compression check — if context is near the threshold, `agent/context_compressor.py` or `agent/conversation_compression.py` fires.

### Iteration Budget

The loop enforces iteration limits via `agent/iteration_budget.py` (62 lines), a thread-safe per-agent consume/refund counter. The parent agent defaults to `max_iterations = 90`; subagents default to `delegation.max_iterations = 50`. The `execute_code` tool gets iterations refunded because its internal Python loop is conceptually part of a single tool call.

A `_budget_grace_call` escape hatch allows one extra iteration when the budget is exhausted, preventing hard cutoff on the edge case where a final API call is in-flight.

### Coding Context

When Hermes detects it is operating in a code workspace (git repo, project directory), `agent/coding_context.py` (883 lines) injects an operating brief and git snapshot into the system prompt's stable tier. This gives the model awareness of the project structure, branch state, and working directory without requiring explicit tool calls.

## 1.5 Parallel Tool Execution

Hermes supports parallel tool dispatch when the model emits multiple tool calls in one response. The parallelism is governed by `agent/tool_dispatch_helpers.py::_should_parallelize_tool_batch()`, which inspects the tool names for data dependencies.

Rules:
- **Always parallel** if tools operate on disjoint resources (e.g., `web_search` + `read_file` on unrelated paths).
- **Serial** if tools mutate the same file or directory (detected by `_paths_overlap()`).
- **MCP parallel safety** — `_is_mcp_tool_parallel_safe()` checks the MCP server's config to determine if parallel calls are supported. MCP servers must explicitly opt in (`supports_parallel_tool_calls: true`).
- **Destructive commands** (e.g., `rm -rf`, `git reset --hard`) require user approval before execution unless `--yolo` mode is active.

The maximum worker pool size is `_MAX_TOOL_WORKERS = 8`, now defined in `agent/tool_executor.py` (relocated from `tool_dispatch_helpers.py`). Concurrent execution uses a `DaemonThreadPoolExecutor` (from `tools/daemon_pool.py`) instead of stdlib `ThreadPoolExecutor` to avoid non-daemon threads hanging at process exit.

## 1.6 Context Compression

When the conversation approaches the model's context limit, Hermes compresses history rather than failing. Two compressor implementations exist:

- **`agent/context_compressor.py`** — An older, local-token-counting compressor that removes oldest messages iteratively.
- **`agent/conversation_compression.py`** — A newer LLM-driven compressor that summarizes blocks of conversation into dense summaries.

Compression is triggered when the ratio of used tokens to context length exceeds `compression.threshold` (default 50%). The target is to shrink to `compression.target_ratio` (default 20%) of the limit. Compression runs **between turns**, never mid-loop, to preserve prompt caching.

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

A lease-based compression lock prevents two compressors from racing on the same session (e.g. parent agent + background-review fork both deciding to compress). The lock uses a refresher thread (`_CompressionLockLeaseRefresher`) and expires after 5 minutes.

## 1.7 Provider Abstraction Layer

Hermes supports 20+ LLM providers. The provider abstraction lives in `agent/` with per-provider adapters:

- `agent/anthropic_adapter.py`
- `agent/bedrock_adapter.py`
- `agent/gemini_native_adapter.py`
- `agent/vertex_adapter.py` (Google Vertex AI for Gemini, OAuth2)
- `agent/codex_responses_adapter.py`
- `agent/azure_identity_adapter.py`
- ...and many more

A provider is selected by `model.provider` in config. The loop itself is provider-agnostic; adapters handle schema translation (OpenAI tools ↔ Anthropic tools ↔ Gemini function calling), token counting, and streaming response parsing. Custom providers can also specify extra HTTP headers via config (`model.extra_headers`) for proxy gateways and enterprise endpoints.

## 1.8 Streaming vs Non-Streaming

The loop supports both streaming (token-by-token delivery to the user) and non-blocking batch mode. Streaming is the default for CLI and gateway; non-streaming is used for cron jobs, delegation, and some gateway platforms where partial output would be confusing.

Streaming is handled by `agent/stream_diag.py` and provider-specific streaming parsers. Tool calls arriving in a stream are buffered until the full JSON argument blob is received, then dispatched.

### Stale-Stream Circuit Breaker

A cross-turn safety mechanism in `agent/chat_completion_helpers.py` detects consecutive stale streams (where the stream completes without producing usable output). A `_consecutive_stale_streams` counter on the agent is incremented on each stale stream and checked by `_check_stale_giveup()`. If too many consecutive stale streams occur, the loop aborts rather than burning retries. The counter resets on model switch and fallback activation.

## 1.9 Interrupt Handling

Users can interrupt a running turn with Ctrl-C (CLI) or a platform-specific cancel signal. The interrupt propagates via `tools.interrupt.set_interrupt()`, which sets a flag checked between loop iterations. If an interrupt fires while waiting for the LLM, the string `"Operation interrupted: waiting for model response (..."` is emitted and the loop exits cleanly.

## 1.10 Summary Diagram

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
