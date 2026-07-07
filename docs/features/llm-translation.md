# LLM translation

`koharu-pipeline/src/ops/llm.rs`, `koharu-ml/src/llm/`.

## Two model sources
- **Local, quantized models** run entirely offline via `candle`:
  `quantized_llama`, `quantized_qwen2`, `quantized_lfm2`, `quantized_hunyuan_dense`
  (`koharu-ml/src/llm/quantized_*.rs`), driven through the shared `Llm` type
  (`model.rs`) with a common `GenerateOptions`/tokenizer (`tokenizer.rs`).
- **Cloud API providers** — Claude, Gemini, OpenAI (`koharu-ml/src/llm/provider/`),
  behind the `AnyProvider` trait (`translate(source, target_language, model)`). All
  providers share one system prompt template
  (`"You are a professional manga/comic translator..."`, `provider::system_prompt`)
  and a common quota/rate-limit error classifier
  (`ensure_provider_success` — detects 429s and provider-specific "quota exceeded"
  strings, surfaces them as `provider_quota_exceeded:<provider>` so the UI can show a
  distinct message from a generic failure).
- `ModelId` (`koharu-ml::llm::ModelId`) is the identifier used across both sources —
  API model IDs are namespaced `provider:model` (e.g. checked via `.contains(':')` in
  `ops/llm.rs` when deciding whether an API key is needed).

## API key storage
`ops/llm.rs` stores cloud provider API keys in the OS keyring (`keyring` crate,
service name `"koharu"`), keyed per provider (`llm_provider_api_key_<provider>`).
There's a one-time migration path from a legacy key-naming scheme
(`llm-provider-api-key:<provider>`) — `get_saved_api_key` checks the new name first,
falls back to the legacy name, and migrates it forward if found.

## RPC surface
`LlmListPayload` (list available models/providers), `LlmLoadPayload` (load a local
model into memory), `LlmGeneratePayload` (translate), `ApiKeyGetPayload`/
`ApiKeySetPayload` (manage stored keys) — see `koharu-types::commands`.

## Supported languages
`koharu-ml::llm::SUPPORTED_LANGUAGES` (macro-generated `(code, name)` pairs) is the
source of truth for target-language options shown in the UI.
