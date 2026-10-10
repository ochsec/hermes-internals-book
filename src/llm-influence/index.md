# LLM Influence on Behavior

Hermes is provider-agnostic, but different models, providers, and modalities meaningfully alter how the agent behaves. This chapter explains the axes of variation and how Hermes adapts to them.

## 6.1 The Provider Adapter Layer

Every supported provider is a `ProviderProfile` registered through `providers/base.py` and `providers.register_provider`. Profiles ship either bundled in `plugins/model-providers/<name>/` or as external model-provider plugins; a profile owns its auth type, base URL, capability metadata, error classification (`classify_api_error`), and wire quirks. Native-wire providers additionally have adapters in `agent/` that translate between Hermes's internal representation (OpenAI-style messages + tool schemas) and the provider's own format.

| Provider | Where | Key Quirk |
|----------|-------|-----------|
| Anthropic | `agent/anthropic_adapter.py` | Native tool format; `cache_control` prompt-caching markers (`anthropic_message_convert.py`); ordered thinking-replay markers so signed thinking blocks replay in order (`anthropic_thinking_replay.py`) |
| OpenAI chat-completions | `agent/transports/chat_completions.py` | Standard function calling; shared transport for aggregators and custom `base_url`s |
| Gemini | `agent/gemini_native_adapter.py` | Different tool schema; consecutive same-role contents merge (HTTP 400 otherwise); free-tier quota-error detection |
| Vertex AI | `agent/vertex_adapter.py` | Google Vertex AI for Gemini (OAuth2), IAM-based auth |
| Bedrock | `agent/bedrock_adapter.py` | Converse API via boto3; IAM auth; streaming via `converse_stream`; Converse `cachePoint` prompt caching with refusal-retry; 1M-window sizing for the GPT-6 family and prefixed inference profiles |
| Codex | `agent/codex_responses_adapter.py` | Responses API; SSE error frames; encrypted reasoning blobs sealed to the minting endpoint; commentary / `final_answer` message phases |
| Azure / Foundry | `agent/azure_identity_adapter.py` | Entra ID `DefaultAzureCredential` chain — keyless, no API key, tokens never persisted |
| Local/Ollama | `agent/transports/chat_completions.py`, `agent/model_metadata.py` | Local server's live `num_ctx` honored (only local endpoints); context and modality detection via `/api/show`; live vLLM context limits honored |
| Copilot | `agent/copilot_acp_client.py` | ACP (Agent Context Protocol) transport |
| xAI Grok | `plugins/model-providers/xai` | Responses API transport (`api_mode="codex_responses"`); OAuth device-code flow; Grok 4.5/4.6 reasoning-effort allowlists |
| Nous Portal | `plugins/model-providers/nous` | OAuth device-code flow + quota entitlements (`nous_rate_guard.py`, `portal_tags.py`) |
| Z.AI (GLM) | `plugins/model-providers/zai` | Overload adaptive backoff for the GLM-5.2 coding plan (429 code 1305), implemented in `agent/retry_utils.py` |
| Tencent | Catalog profiles (`hermes_cli/models_catalog_static.py`) | Hy4 preview plus Hy3 via TokenHub (chat completions) and TokenPlan (Anthropic Messages) |
| DeepInfra | `plugins/model-providers/deepinfra` | Provider-prefix aliases for model parsing |
| Solstice | `plugins/model-providers/solstice` | OAuth PKCE through a Nous-portal broker; per-user-quota Gemini subscription; hidden from pickers until the user signs in by name |
| Upstage Solar | `plugins/model-providers/upstage` | Top-level `reasoning_effort`; an unset reasoning config defaults reasoning **on** at `medium` (Solar's server default is off) |
| vLLM / relays | Custom profiles | Token-based output-cap error parsing, live context limits from the endpoint |

Profiles can also ship per-model capability metadata and, for OAuth-shaped plugins, own `hermes auth add/status/logout/refresh` end to end; pre-release profiles stay off discovery until the user signs in.

## 6.2 Tool Schema Influence

The tool schemas sent to the LLM are identical regardless of provider — Hermes normalizes them. However, **models vary wildly in how well they adhere to schemas**:

- **Claude** — Excellent schema adherence; rarely invents parameter names.
- **GPT-4o** — Good adherence; occasionally omits optional parameters.
- **Gemini** — Mixed; sometimes produces nested JSON blobs instead of flat params.
- **Local models** (Llama, Mistral) — Poor adherence; frequent hallucinated parameters or malformed JSON.

`agent/message_sanitization.py::_repair_tool_call_arguments()` fixes common model mistakes in tool-call JSON:
- Unescaped control characters are re-serialized to wire-valid JSON.
- Malformed JSON is re-parsed with relaxed rules: trailing commas stripped, unclosed structures closed in stack order, excess closers trimmed (bounded), Python `None` treated as `{}`.
- If nothing repairs it, the arguments become `{}` so the request still succeeds instead of dying; every repair logs at WARNING.

Parameter-name and required-argument validation happen downstream in the tool layer, where an invalid call becomes a synthetic error result the model can retry from.

## 6.3 Multimodal Variation

Hermes supports images, video, and audio via the `vision`, `video`, and `tts` toolsets. How these are presented to the model depends on the provider:

- **Claude** — Images as `image` content blocks in the message array.
- **GPT-4o** — Images as base64 data URIs in `image_url` fields.
- **Gemini** — Images uploaded via the Files API, referenced by URI.
- **Local models** — Depends on server support (Ollama, LM Studio, etc.).

`agent/image_routing.py` decides per image via `agent.image_input_mode` (`auto` | `native` | `text`). In `auto`, a vision-capable main model gets images natively — so images are analyzed by the model already in context — but an explicit `auxiliary.vision` backend forces `text` mode even for vision-capable main models; `native` is the absolute override. The `supports_vision` check resolves in order: `model.supports_vision` → `providers.<p>.models.<model>` → `models.<model>`, with string booleans coerced (a quoted `supports_vision: "false"` is not `true`).

### Ollama Vision Detection

For local Ollama models, vision capability is detected by querying the `/api/show` endpoint, which exposes the model's modality configuration. This allows Hermes to route images to local vision-capable models (e.g. LLaVA) without requiring the user to manually configure `supports_vision`.

### Image Error Handling

If an image exceeds the provider's dimension limit, `agent/turn_recovery.py::_image_error_max_dimension()` extracts the max allowed size from the error message and resizes the image before retry. This is provider-specific — Anthropic allows 8K×8K, while Gemini has smaller limits.

## 6.4 Reasoning / Thinking Tokens

Some models emit reasoning tokens (Chain-of-Thought, "think" blocks):

- **Claude** — Signed `thinking` blocks, replayed in order via `agent/anthropic_thinking_replay.py`.
- **DeepSeek-R1** — Emits `reasoning` blocks.
- **Codex / GPT-5.x** — Encrypted reasoning items (`reasoning.encrypted_content`) sealed to the minting endpoint; replay is bound to the endpoint and wire model.
- **o1 / o3** — Hidden reasoning, not exposed in the API.

`agent/think_scrubber.py` provides `StreamingThinkScrubber`, a stateful scrubber that strips think blocks from streamed deltas without eating an opening `&#x3C;think>` that arrives alone. Its `THINK_TAG_NAMES` list is the one shared tag vocabulary — the CLI stream filter, the gateway stream filter, and the final-response stripper all bind to it — and includes CJK tag names (思考, 反思, 推理, 推敲) for models like MiniMax-M3 that emit Chinese reasoning tags.

### Effort Vocabulary and Clamping

`agent/reasoning_effort.py` is the single source of truth for reasoning effort. The internal ladder is `none, minimal, low, medium, high, xhigh, max, ultra` (`EFFORT_LADDER`); `ultra` is Hermes-internal (the Codex product tier) — no wire accepts it. `clamp_effort` keeps a supported level verbatim, otherwise clamps to the nearest **weaker** level; unset stays unset. Each provider declares its own wire vocabulary: Codex splits by model generation (GPT-5.6/legacy/Astra, with Astra and GPT-6.1 Sol rejecting `none`/`minimal`), xAI splits Grok 4.6+ (adds `xhigh`) from older Grok, Kimi K3 has positional quirks (`medium` rounds to `high`, `xhigh` rounds to `max`), and per-model reasoning_effort overrides ride the same resolver. `hermes_constants.parse_reasoning_effort()` maps config strings — `none`/`false`/`disabled` mean "reasoning off", not a wire level.

The `/reasoning` slash command handles both halves: `/reasoning <level> [--global]` sets the effort (session scope by default), while `/reasoning show|hide|full|clamp` toggles the display — `full` shows all reasoning lines, the default clamps to the first 10.

### Answer-in-Reasoning Promotion

A `reasoning` field is not provenance — OpenRouter, DeepSeek, Bedrock and several adapters all put private chain-of-thought there. `agent/reasoning_promotion.py` gates whether a reasoning-only clean stop may be promoted to the visible answer, decided from the agent's **live route** on every call (fallback activation, primary restore, and `/model` switches never carry a stale decision). Trusted: a per-model `answer_in_reasoning` opt-in or a provider-level `capabilities: {answer_in_reasoning: ...}` block on the live custom-provider route, or the local Nemotron-3.5-Lightning parser route. OpenRouter and non-chat-completions transports are never trusted.

## 6.5 Context Length Sensitivity

Different models have different context lengths, and this dramatically affects behavior:

| Model Family | Context | Behavior at Limit |
|--------------|---------|-------------------|
| Claude | 200K (older) / 1M (Opus & Sonnet 4.6+, 5) | Rarely hits limit; long conversations feel coherent |
| GPT-5.x | 400K–1.05M (Codex OAuth: 272K, 900K on eligible bases) | Hits limit on large codebases; compression fires |
| Gemini | 1M | Rarely limits, but tool use degrades near the end |
| Kimi K3 | 1M (endpoint-scoped; K2.5 256K) | Rarely limits; the resolver guards against a stale 32K underreport |
| GLM-5.2 / 5.3 | 1M / 1.31M | Rarely limits on the coding plan |
| Local 7B | 4K–8K | Constant compression; tool schemas alone fill 30%+ |
| Ollama defaults | 2K–4K | Completely unusable for tool use without `num_ctx` boost |

`agent/model_metadata.py` estimates token counts per message (`estimate_messages_tokens_rough()`). When the estimate crosses `compression.threshold`, compression fires. Small-context models spend a disproportionate fraction of their budget on the system prompt + tool schemas, leaving little room for conversation. `get_model_context_length()` resolves the window in a fixed order: 0 config override / MoA aggregator / model_overrides / custom_providers / endpoint-scoped entries, then the persistent cache, then provider-aware probes, ending in a 256K fallback.

### Config Context-Length Short-Circuit

When `model.context_length` is set explicitly in `config.yaml`, `_resolve_active_context_length()` (`model_tools.py`) short-circuits the OpenRouter `/models` probe at `get_model_context_length()` step 0. This means non-OpenRouter providers no longer pay the ~2–3 second OpenRouter fetch at every CLI startup just to discover the context window. The config value takes precedence when it is a positive integer; an on-disk metadata cache (avoids a ~200 ms probe) sits between the config override and the live resolver. For local Ollama servers, the live `num_ctx` wins over the GGUF's advertised window — only a local endpoint can honor it, so a stale override never admits a hosted model.

### vLLM Live Context Limits

For local vLLM endpoints, Hermes queries the server's model metadata to honor live context limits rather than relying on hardcoded estimates. The vLLM token-based output-cap error format (`"This model's maximum output tokens"`) is parsed to detect when the model is forced to truncate, triggering a retry or compression rather than death-looping on repeated over-cap 400 errors.

## 6.6 Temperature and Sampling

The main chat loop does not hardcode a temperature: requests leave sampling at the provider's server default unless the user sets values in config or a custom provider's `api_kwargs` pass them through verbatim. Sampling knobs the Hermes core does set itself live elsewhere — TTS voices, MCP sampling, and vision tools each carry their own temperature, tuned per surface.

Lower temperatures make tool calling more deterministic (good for structured outputs). Higher temperatures increase creativity but can cause tool schema violations.

## 6.7 Max Completion Tokens

`utils.py::model_forces_max_completion_tokens()` detects OpenAI families (`gpt-4o`, `gpt-4.1`, `gpt-5`, `o1`, `o3`, `o4`) that reject `max_tokens` with an HTTP 400 `unsupported_parameter`, and the request is capped as `max_completion_tokens` instead. Gemini output caps get a separate treatment: `agent/transports/chat_completions.py` raises the cap when thinking tokens (billed against `max_tokens`) would otherwise exhaust it. When a model is forced to truncate, `agent/turn_truncation.py::boosted_output_cap()` bumps the ephemeral output cap and retries instead of dying on repeated over-cap errors.

## 6.8 Provider-Specific Streaming

Streaming behavior varies by provider:

- **Anthropic** — Native SSE with `message_delta` events; tool calls arrive in `content_block_start` / `content_block_delta`; stream spans keep their input and cache token counts.
- **OpenAI / chat-completions** — Standard SSE; streamed tool-call deltas accumulate incrementally.
- **Bedrock** — `converse_stream()` event stream normalized by `normalize_converse_stream_events()`.
- **Gemini** — Streaming is non-standard; chunks may arrive out of order.
- **Local** — Varies by server implementation; some don't support streaming at all.

`agent/stream_diag.py` doesn't normalize anything — it is the diagnostics layer: per-attempt counters (bytes, chunks, first-chunk latency, HTTP status, serving provider from `cf-ray` / `x-openrouter-provider` headers), exception chains, and retry logging when a stream dies mid-response. Normalization into Hermes's internal event stream happens in each adapter/transport.

## 6.9 Model Personality Drift

Swapping models mid-conversation (fallback) can cause "personality drift" — the assistant suddenly changes tone, formatting, or tool-calling style. Hermes mitigates this by:

- Preserving the system prompt exactly across model swaps.
- Using standardized tool schemas.
- Warning the user in logs when a fallback occurs.

## 6.10 Summary: Model Behavior Matrix

| Model | Tool Adherence | Context | Streaming | Multimodal | Reasoning |
|-------|---------------|---------|-----------|------------|-----------|
| Claude (4.6+ / 5) | Excellent | 1M (200K older) | Native SSE | Yes | Signed thinking, clamped effort ladder |
| GPT-5.5 / 5.6 | Very Good | 1.05M (272K–900K Codex OAuth) | Standard | Yes | Encrypted reasoning items; generation-specific effort sets |
| Gemini 2.5/3.5 | Good | 1M+ | Non-standard | Yes | None visible |
| Grok-4.5 / 4.6 | Good | 500K | Responses API | Yes | Effort allowlist: low/med/high (4.6 adds `xhigh`, both reject `none`) |
| Kimi K3 | Good | 1M | Standard | No | Effort ladder low/high/max with positional quirks |
| DeepSeek-R1 | Good | 128K | Standard | No | Explicit blocks |
| GLM-5.2 / 5.3 (Z.AI) | Good | 1M / 1.31M | Standard | No | Overload adaptive backoff (429/1305) |
| Upstage Solar | Good | Per catalog | Standard | No | Top-level `reasoning_effort`, defaults on at `medium` |
| Llama 3 70B | Moderate | 128K | Standard | No | None |
| Local 7B (Ollama) | Poor | 4K–8K (live `num_ctx` on local servers) | Varies | Via `/api/show` detection | None |
| vLLM endpoints | Varies | Live metadata | Varies | Varies | None |

A model's position in this matrix determines which guardrails fire most often, how frequently compression runs, and whether multimodal or reasoning features are available.
