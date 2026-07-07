# koharu (Manga Offline Translate) — module map

ML-powered offline manga/comic translator, written in Rust, packaged as a Tauri
desktop app with a Next.js frontend. Fork of `mayocream/koharu`, maintained by MLT.
Pipeline: detect speech bubbles → OCR → inpaint (remove original text) → LLM translate
→ render translated text back onto the page. All ML inference runs locally
(`candle`); no image data leaves the machine unless the user opts into a cloud LLM
provider for translation.

## Cargo workspace crates

| Crate | What it does |
|---|---|
| `koharu/` | Tauri app shell: CLI args, app bootstrap (`src/app.rs`), embeds the RPC/asset HTTP server, ships the compiled Next.js UI as static assets |
| `koharu-types/` | Shared types: `Document`/`TextBlock`/`TextStyle` data model, RPC `commands.rs` (request/response payloads per method), `events.rs` (pipeline progress events), `method.rs` (RPC method enum) |
| `koharu-pipeline/` | Orchestration: `pipeline.rs` (the detect→ocr→inpaint→translate→render sequence, cancellable, progress-broadcasting), `state_tx.rs` (transactional read/update of in-memory `Document` state), `ops/` (one module per operation group: `vision` = detect/ocr/inpaint/render, `llm` = translation + API key management, `edit` = text block/mask/brush editing, `core`, `process` = the "run all" entry point, `utils`) |
| `koharu-ml/` | ML model wrappers on `candle`: `comic_text_detector` (bubble detection), `manga_ocr`/`mit48px_ocr` (OCR), `lama` (inpainting, with CPU/CUDA/Metal FFT backends), `font_detector`, `llm` (local quantized models + cloud API providers). `facade.rs` ties these into one `Ml` object used by the pipeline |
| `koharu-renderer/` | Renders translated text back onto the image: font matching (`font.rs`, `FontBook`), text layout incl. CJK vertical writing mode (`layout.rs`, `text/`), shape/shader effects (`shape.rs`), `facade.rs` is the entry point used by `ops/vision.rs`'s `render` |
| `koharu-rpc/` | Axum HTTP server: WebSocket RPC endpoint (msgpack-encoded request/response + notifications, `rpc.rs`), MCP server endpoint for AI agents (`mcp/`), static asset serving for the embedded UI (`server.rs`) |
| `koharu-http/` | HTTP client helpers: model downloads from Hugging Face Hub (`hf_hub.rs`), byte-range requests, download progress reporting |
| `koharu-runtime/` | Runtime dylib management — downloading/loading the native ML runtime libraries the app depends on (`dylib.rs`, `zip.rs`) |

## Frontend (`ui/`, Next.js)

| Path | What it does |
|---|---|
| `ui/app/(app)/page.tsx`, `layout.tsx` | Main editor shell |
| `ui/app/(app)/settings/page.tsx`, `about/page.tsx` | Settings, about |
| `ui/components/Canvas.tsx`, `canvas/*` | The image canvas: zoom/pan, tool rail, workspace, text block sprite layer/annotations, status bar |
| `ui/components/Panels.tsx`, `panels/*` | Side panels: layers, text blocks, render controls |
| `ui/components/MenuBar.tsx`, `Navigator.tsx` | Top menu, file navigator |
| `ui/components/CbzExportDialog.tsx` | "Run all to CBZ" export dialog |
| `ui/lib/api.ts`, `rpc-types.ts`, `rpcSchemas.ts`, `ws.ts` | RPC client — talks to the Rust backend's WebSocket endpoint |
| `ui/lib/query/*` | React Query hooks/mutations wrapping RPC calls |
| `ui/lib/stores/*` | Zustand-style UI state stores (editor UI, LLM UI, operation progress, preferences, errors) |
| `ui/hooks/*` | Canvas zoom, mask drawing, brush drawing, text block drafting/context menu, RPC connection |
| `ui/public/locales/<lang>/translation.json` | UI translations (en-US, es-ES, ja-JP, ru-RU, zh-CN, zh-TW) |

## Other
- `e2e/` — Playwright end-to-end tests, one spec per pipeline stage (import, zoom,
  detect/OCR, edit, LLM load/translate/unload, mask/inpaint, brush, style/rerender,
  process current/all, cancel, export).
- `scripts/` — Python scripts for model conversion/export (ONNX) and dataset tooling;
  `dev.ts`/`release.ts` for the Rust+Next.js dev/release workflow.
- `docs/skills/manga-offline-translate-batch/` — a Claude Code skill + script for
  batch-driving this app via its MCP server.

## Feature docs
See `docs/features/`:
- `detection-ocr.md`
- `inpainting.md`
- `llm-translation.md`
- `text-rendering.md`
- `pipeline-orchestration.md`
- `rpc-mcp-server.md`
- `editor-ui.md`
