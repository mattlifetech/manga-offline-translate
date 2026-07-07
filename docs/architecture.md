# Architecture

A Tauri desktop app whose "backend" is a self-contained Axum HTTP server embedded in
the Rust process, and whose "frontend" is a Next.js app compiled to static assets and
served by that same server. The Tauri webview just loads `http://localhost:<port>/`.

## Process shape (`koharu/src/app.rs`)
On startup: resolves app data dirs (`APP_ROOT`/`LIB_ROOT`/`MODEL_ROOT`/`CACHE_DIR`
under the OS data-local dir), ensures/preloads native ML runtime dylibs
(`koharu-runtime`), builds `AppResources` (ML facade, renderer, shared `State`), starts
an Axum server (`koharu-rpc::server`) on a `TcpListener`, and launches the Tauri
window pointed at it. A `--download` CLI flag can fetch dylibs and exit without
starting the GUI (used in CI/first-run provisioning).

## Backend ↔ frontend protocol (`koharu-rpc`, `koharu-types`)
Two endpoints on the one Axum router:
- `/ws` — the primary channel. Msgpack-encoded request/response over WebSocket
  (`rpc.rs`): the frontend sends `{id, method, params}`, gets back either a
  `Response {id, result|error}` or an out-of-band `Notification {method, params}`
  (used for pipeline progress streaming). `koharu-types::Method` is the enum of every
  callable RPC method; `koharu-types::commands` defines each method's payload/result
  types — this pairing is the actual API contract between Rust and the UI.
- `/mcp` — an MCP (Model Context Protocol) server exposing the same underlying
  operations to AI agents, not just the bundled UI (`koharu-rpc/src/mcp/`). This is
  what `docs/skills/manga-offline-translate-batch` drives.
- Everything else falls back to serving the embedded Next.js static build (SPA-style
  fallback to `index.html`).

## Document state (`koharu-pipeline::state_tx`)
The in-memory unit of work is a `Document` (per opened image): text blocks, detected
regions, OCR text, translations, inpaint/brush layers, render state. All mutation goes
through `state_tx::read_doc`/`update_doc` — RPC handlers and pipeline steps never touch
shared state directly, keeping concurrent access (UI edits vs. a running batch
pipeline) consistent.

## The pipeline (`koharu-pipeline::pipeline`, `ops/`)
`ops::process` is the "run all images" entry point; `ops::vision` exposes the
individual per-image steps (`detect`, `ocr`, `inpaint`, `render`) that the UI can also
call one at a time for interactive editing. A pipeline run is cancellable
(`AtomicBool` flag on `PipelineHandle`) and streams `PipelineProgress` events over a
broadcast channel, which `rpc.rs` forwards to the frontend as WebSocket notifications.
Only one pipeline can run at a time (`process` bails if `state.pipeline` is already
`Some`).

## ML (`koharu-ml`)
All inference is local via `candle`, wrapped per-model in submodules
(`comic_text_detector`, `manga_ocr`/`mit48px_ocr`, `lama`, `font_detector`) and unified
by `facade.rs` into the `Ml` struct the pipeline calls into. `lama` (inpainting) has
separate CPU/CUDA/Metal FFT backends for performance. LLM translation additionally
supports cloud API providers (Claude/Gemini/OpenAI) as an alternative to local
quantized models — see `llm-translation.md`.

## Rendering (`koharu-renderer`)
Puts translated text back onto the page: font selection/fallback (`FontBook`), layout
including CJK vertical writing mode and Latin text box expansion heuristics, and
shape/shader-based text effects (stroke, etc.), rasterized via `tiny-skia`.

## Testing
`e2e/` (Playwright) drives the actual Tauri app end-to-end, one spec per pipeline
stage — this is the primary regression net for the detect→ocr→inpaint→translate→render
flow and its UI. `koharu-renderer/tests/` and `benches/` cover rendering correctness
and performance in isolation.
