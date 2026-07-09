# LLM Influence on Behavior

Hermes is provider-agnostic, but different models, providers, and modalities meaningfully alter how the agent behaves. This chapter explains the axes of variation and how Hermes adapts to them.

## 6.1 The Provider Adapter Layer

Every supported LLM provider has a dedicated adapter in `agent/`:

| Provider | Adapter File | Key Quirk |
|----------|--------------|-----------|
| Anthropic | `anthropic_adapter.py` | Native tool format, prompt caching via `cache_control`, ordered replay cache markers |
| OpenAI | Built into loop | Standard function calling, no streaming tool calls until recently |
| Gemini | `gemini_native_adapter.py`, `gemini_cloudcode_adapter.py` | Different tool schema, quota sharing across pool entries, consecutive same-role content merging |
| Vertex AI | `vertex_adapter.py` | Google Vertex AI for Gemini (OAuth2), IAM-based auth |
| Bedrock | `bedrock_adapter.py` | IAM role auth, no streaming |
| Codex | `codex_responses_adapter.py` | Responses API, SSE error frames, commentary-phase preamble routed to reasoning channel |
| Azure | `azure_identity_adapter.py` | Entra ID auth, no API key |
| Local/Ollama | Handled generically | Must set `num_ctx` manually; vision detection via `/api/show`; live vLLM context limits honored |
| Copilot | `copilot_acp_client.py` | ACP (Agent Context Protocol) transport |
| xAI Grok | Native | OAuth device-code flow; grok-4.5 GA with reasoning-effort allowlist |
| Nous Portal | Native | OAuth + quota entitlements |
| Z.AI (GLM) | Native | Overload adaptive backoff for GLM-5.2 (HTTP 1305) |
| Tencent | Native | Hy3 GA model in catalog |
| vLLM | Native | Token-based output-cap error parsing, live context limits from endpoint |

Adapters translate between Hermes's internal representation (OpenAI-style messages + tool schemas) and each provider's native format. They also handle streaming parsers, token counting, and error extraction.

## 6.2 Tool Schema Influence

The tool schemas sent to the LLM are identical regardless of provider — Hermes normalizes them. However, **models vary wildly in how well they adhere to schemas**:

- **Claude** — Excellent schema adherence; rarely invents parameter names.
- **GPT-4o** — Good adherence; occasionally omits optional parameters.
- **Gemini** — Mixed; sometimes produces nested JSON blobs instead of flat params.
- **Local models** (Llama, Mistral) — Poor adherence; frequent hallucinated parameters or malformed JSON.

`agent/message_sanitization.py::_repair_tool_call_arguments()` fixes common model mistakes:
- Unknown parameter names are dropped.
- Malformed JSON is re-parsed with relaxed rules.
- Missing required parameters trigger a synthetic error result.

## 6.3 Multimodal Variation

Hermes supports images, video, and audio via the `vision`, `video`, and `tts` toolsets. How these are presented to the model depends on the provider:

- **Claude** — Images as `image` content blocks in the message array.
- **GPT-4o** — Images as base64 data URIs in `image_url` fields.
- **Gemini** — Images uploaded via the Files API, referenced by URI.
- **Local models** — Depends on server support (Ollama, LM Studio, etc.).

`agent/image_routing.py` selects the appropriate provider based on `model.supports_vision` and `auxiliary.vision.provider` config. In auto mode, Hermes prefers the main model's native vision capability over falling back to an auxiliary vision model, so images are analyzed by the model already in context when possible.

### Ollama Vision Detection

For local Ollama models, vision capability is detected by querying the `/api/show` endpoint, which exposes the model's modality configuration. This allows Hermes to route images to local vision-capable models (e.g. LLaVA) without requiring the user to manually configure `supports_vision`.

### Image Error Handling

If an image exceeds the provider's dimension limit, `_image_error_max_dimension()` extracts the max allowed size from the error message and resizes the image before retry. This is provider-specific — Anthropic allows 8K×8K, while Gemini has smaller limits.

## 6.4 Reasoning / Thinking Tokens

Some models emit reasoning tokens (Chain-of-Thought, "think" blocks):

- **Claude 3.7 Sonnet** — Emits `<thinking>` tags.
- **DeepSeek-R1** — Emits `<reasoning>` blocks.
- **o1 / o3** — Hidden reasoning, not exposed in API.

`agent/think_scrubber.py` strips these from the final text before presenting to the user, but preserves them in the session log for debugging.

The `/reasoning` slash command lets users toggle how much reasoning is shown:
- `none` — Strip all reasoning.
- `minimal` — Show only conclusions.
- `medium` — Show key reasoning steps.
- `high` / `xhigh` — Show full reasoning (can be very verbose).

## 6.5 Context Length Sensitivity

Different models have different context lengths, and this dramatically affects behavior:

| Model Family | Context | Behavior at Limit |
|--------------|---------|-------------------|
| Claude 3.5/4 | 200K | Rarely hits limit; long conversations feel coherent |
| GPT-4o | 128K | Hits limit on large codebases; compression fires |
| Gemini 1.5 | 1M | Rarely limits, but tool use degrades near the end |
| Local 7B | 4K–8K | Constant compression; tool schemas alone fill 30%+ |
| Ollama defaults | 2K–4K | Completely unusable for tool use without `num_ctx` boost |

`agent/model_metadata.py` estimates token counts per message. When the estimate crosses `compression.threshold`, compression fires. Small-context models spend a disproportionate fraction of their budget on the system prompt + tool schemas, leaving little room for conversation.

### vLLM Live Context Limits

For local vLLM endpoints, Hermes queries the server's model metadata to honor live context limits rather than relying on hardcoded estimates. The vLLM token-based output-cap error format (`"This model's maximum output tokens"`) is parsed to detect when the model is forced to truncate, triggering a retry or compression rather than death-looping on repeated over-cap 400 errors.

## 6.6 Temperature and Sampling

Hermes defaults to temperature=0.7 for most providers. This is configurable per-provider:

```yaml
model:
  temperature: 0.7
  top_p: 1.0
  frequency_penalty: 0.0
  presence_penalty: 0.0
```

Lower temperatures make tool calling more deterministic (good for structured outputs). Higher temperatures increase creativity but can cause tool schema violations.

## 6.7 Max Completion Tokens

Some providers (Gemini, Codex) enforce a `max_completion_tokens` limit. `utils.py::model_forces_max_completion_tokens()` detects these providers and caps the output. If a model is forced to truncate, the turn may end mid-sentence or mid-tool-call, triggering a retry or error.

## 6.8 Provider-Specific Streaming

Streaming behavior varies by provider:

- **Anthropic** — Native SSE with `message_delta` events; tool calls arrive in `content_block_start` / `content_block_delta`.
- **OpenAI** — Standard SSE; tool calls arrive in a single `function_call` chunk at the end.
- **Gemini** — Streaming is non-standard; chunks may arrive out of order.
- **Local** — Varies by server implementation; some don't support streaming at all.

`agent/stream_diag.py` normalizes all streaming formats into a single internal event stream.

## 6.9 Model Personality Drift

Swapping models mid-conversation (fallback) can cause "personality drift" — the assistant suddenly changes tone, formatting, or tool-calling style. Hermes mitigates this by:

- Preserving the system prompt exactly across model swaps.
- Using standardized tool schemas.
- Warning the user in logs when a fallback occurs.

## 6.10 Summary: Model Behavior Matrix

| Model | Tool Adherence | Context | Streaming | Multimodal | Reasoning |
|-------|---------------|---------|-----------|------------|-----------|
| Claude Sonnet 4 | Excellent | 200K | Native SSE | Yes | Hidden |
| GPT-4o / GPT-5.5 | Very Good | 128K / 272K | Standard | Yes | Codex commentary channel |
| Gemini Pro 2.5 | Good | 1M+ | Non-standard | Yes | None visible |
| Grok-4.5 | Good | 256K | Standard | Yes (Imagine) | Reasoning-effort allowlist |
| DeepSeek-R1 | Good | 128K | Standard | No | Explicit blocks |
| GLM-5.2 (Z.AI) | Good | 128K | Standard | No | Overload adaptive backoff |
| Llama 3 70B | Moderate | 128K | Standard | No | None |
| Local 7B (Ollama) | Poor | 4K–8K | Varies | Via `/api/show` detection | None |
| vLLM endpoints | Varies | Live metadata | Varies | Varies | None |

A model's position in this matrix determines which guardrails fire most often, how frequently compression runs, and whether multimodal or reasoning features are available.
