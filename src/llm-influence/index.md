# LLM Influence on Behavior

Hermes is provider-agnostic, but different models, providers, and modalities meaningfully alter how the agent behaves. This chapter explains the axes of variation and how Hermes adapts to them.

## 6.1 The Provider Adapter Layer

Every supported LLM provider has a dedicated adapter in `agent/`:

| Provider | Adapter File | Key Quirk |
|----------|--------------|-----------|
| Anthropic | `anthropic_adapter.py` | Native tool format, prompt caching via `cache_control` |
| OpenAI | Built into loop | Standard function calling, no streaming tool calls until recently |
| Gemini | `gemini_native_adapter.py`, `gemini_cloudcode_adapter.py` | Different tool schema, quota sharing across pool entries |
| Bedrock | `bedrock_adapter.py` | IAM role auth, no streaming |
| Codex | `codex_responses_adapter.py` | Responses API, SSE error frames |
| Azure | `azure_identity_adapter.py` | Entra ID auth, no API key |
| Local/Ollama | Handled generically | Must set `num_ctx` manually; small context crashes tool use |
| Copilot | `copilot_acp_client.py` | ACP (Agent Context Protocol) transport |
| Nous Portal | Native | OAuth + quota entitlements |

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

`agent/image_routing.py` selects the appropriate provider based on `model.supports_vision` and `auxiliary.vision.provider` config.

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
| Claude Sonnet | Excellent | 200K | Native | Yes | Hidden |
| GPT-4o | Very Good | 128K | Standard | Yes | None visible |
| Gemini Pro | Good | 1M | Non-standard | Yes | None visible |
| DeepSeek-R1 | Good | 128K | Standard | No | Explicit blocks |
| Llama 3 70B | Moderate | 128K | Standard | No | None |
| Local 7B | Poor | 4K–8K | Varies | Varies | None |

A model's position in this matrix determines which guardrails fire most often, how frequently compression runs, and whether multimodal or reasoning features are available.
