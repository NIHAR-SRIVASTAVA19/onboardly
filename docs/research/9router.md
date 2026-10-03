# Research: what does 9router support?

Issue #2 (parent map #1). Researched 2026-10-03.

## Identification

"9router" matches [decolua/9router](https://github.com/decolua/9router) (site 9router.com): an MIT-licensed, self-hosted Node/Next.js LLM gateway that exposes one OpenAI-compatible endpoint and routes to 40+ providers with fallback. Several forks exist (tungbach12, nightwalker89, Alexi5000); confidence that decolua/9router is the upstream is high but not certain. **Confirm this is the router the user runs** (check the dashboard/version).

## Fact sheet

| Question | Finding | Source |
|---|---|---|
| OpenAI-compatible? | Yes. `POST /v1/chat/completions`, `GET /v1/models`, plus embeddings, audio (STT/TTS), image endpoints. Also accepts Claude/Responses-API formats. | [README](https://github.com/decolua/9router), [open-sse/AGENTS.md](https://github.com/decolua/9router/blob/master/open-sse/AGENTS.md) |
| Base URL | `http://localhost:20128/v1` (default port 20128; `PORT` env on the router) | README |
| Auth | Dashboard generates keys; send `Authorization: Bearer <key>`. Enforced on `/v1/*` only if router env `REQUIRE_API_KEY` is set, so a local default may accept any value. | README |
| Model naming | `provider/model`, e.g. `cc/claude-opus-4-7`, `glm/glm-5.1`, `kr/claude-sonnet-4.5`; user-defined "combos" referenced by bare name (e.g. `premium-coding`) with fallback. List real names via `GET /v1/models`. | README; `services/model.js parseModel` per AGENTS.md |
| Tool/function calling | Translated per provider. OpenAI `tools`/`tool_choice` are converted to Claude (`openai-to-claude.js`) and Gemini `functionDeclarations` (`openai-to-gemini.js`). README does not promise it; supported in source. Works at the router level, but real behavior depends on the backing model/provider. | [openai-to-claude.js](https://github.com/decolua/9router/blob/master/open-sse/translator/request/openai-to-claude.js), [openai-to-gemini.js](https://github.com/decolua/9router/blob/master/open-sse/translator/request/openai-to-gemini.js) |
| Structured output (JSON schema) | Not natively enforced. For Claude targets, `response_format: json_schema` is converted into a system-prompt instruction ("respond with valid JSON that strictly follows this schema"), i.e. prompt-level only, no constrained decoding. No `response_format` handling found in the Gemini translator (may be dropped or passed through). Pass-through for OpenAI-compatible providers is likely but unverified. | openai-to-claude.js lines ~117-125; grep of openai-to-gemini.js |
| Vision / image input | Image content parts (`image_url`) are recognised (`modality.js` classifies as "vision") and converted for Claude (`openai-to-claude.js`). AGENTS.md warns the OpenAI bridge is lossy for **non-base64 images**, so send base64 data URLs, not remote URLs. | [modality.js](https://github.com/decolua/9router/blob/master/open-sse/translator/concerns/modality.js), AGENTS.md line 36 |
| Streaming | SSE, translated back to client format | AGENTS.md |
| Unsupported-param stripping | Router silently drops some params per provider/model (e.g. `temperature` for all Claude models) | [paramSupport.js](https://github.com/decolua/9router/blob/master/open-sse/translator/concerns/paramSupport.js) |

Not found in any primary doc: an explicit compatibility matrix for tools, JSON schema, or vision. Everything above for those three is inferred from translator source, not documented guarantees.

## What the onboarding agent can rely on

- **Rely on:** plain chat completions, streaming, `/v1/models` discovery, bearer-key header.
- **Rely on with validation:** tool calling (translated, model-dependent). Use a model/combo verified with a smoke test.
- **Do not rely on:** server-enforced JSON-schema output. Treat it as best-effort: request JSON, then validate with Pydantic and retry/repair on failure. Alternative: HR-request parsing via a single forced tool call whose arguments are the schema (tools are translated more faithfully than `response_format`), still validated client-side.
- **Vision (document analysis):** feasible only with a vision-capable model behind the route; send base64 data URLs. Smoke-test with a sample document; for PDFs, render pages to images (or extract text) before sending, since no PDF/file input path is documented.

## Recommended LLM interface

Library: official `openai` Python SDK (`AsyncOpenAI`) directly, or `langchain-openai` `ChatOpenAI(base_url=..., api_key=...)` inside LangGraph. Both just need a base URL; no 9router-specific client exists or is needed.

Env vars:

```
LLM_BASE_URL=http://localhost:20128/v1
LLM_API_KEY=<dashboard key; any non-empty string if REQUIRE_API_KEY is off>
LLM_MODEL=<provider/model or combo name from GET /v1/models>
LLM_VISION_MODEL=<vision-capable model/combo; defaults to LLM_MODEL>
```

Shape: a small `LLMClient` seam (one module) exposing `complete(messages)`, `extract(messages, PydanticModel)` (JSON/forced-tool call + Pydantic validation + bounded retry) and `describe_image(image_bytes, prompt)`. Keep all 9router quirks behind it so the model can be swapped or a different gateway used. Add a startup/CI smoke test hitting `/v1/models`, a tool call, a JSON extraction and an image prompt against the chosen models.

## Open items for the user

1. Which provider/model or combo is the target (decides real tool/vision/JSON behavior)?
2. Is `REQUIRE_API_KEY` on in the local install?
