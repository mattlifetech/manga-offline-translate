# Pipeline orchestration

`koharu-pipeline/src/pipeline.rs`, `state_tx.rs`, `ops/process.rs`, `ops/core.rs`.

## Document state (`state_tx`)
Every image open in the app has a `Document` (text blocks, detected regions, OCR/
translation text, mask/brush/render state) held in shared app state. All reads/writes
go through `state_tx::read_doc(&state, index)` / `update_doc(&state, index, snapshot)`
— a read-modify-write pattern that keeps interactive UI edits and an in-progress batch
pipeline from corrupting each other's view of a document.

## Full batch run (`ops::process`)
`process(state, ProcessRequest)` is "Run all" from the README's toolbar:
1. If an LLM API model ID is set but no key was passed explicitly, look up the saved
   key from the OS keyring (see `llm-translation.md`).
2. Refuse to start if a pipeline is already running (`state.pipeline` is `Some`) — only
   one batch run at a time.
3. Register a `PipelineHandle` with a cancel flag (`AtomicBool`), spawn
   `pipeline::run_pipeline` on a background Tokio task, return immediately (the caller
   gets progress via the broadcast channel, not by waiting on this call).

## Progress streaming
`pipeline.rs` holds a process-wide `broadcast::Sender<PipelineProgress>`
(`PIPELINE_TX`, lazily initialized). `run_pipeline` calls `emit()` at each
step/image (`PipelineStep`, `PipelineStatus`); `koharu-rpc` subscribes and forwards
these as WebSocket notifications to the connected UI, and the MCP server can likely
surface the same stream to agent callers. `subscribe()` is how any consumer taps into
this stream.

## Cancellation
`PipelineHandle.cancel: Arc<AtomicBool>` is checked between steps/images inside
`run_pipeline` — setting it (from a "Cancel" RPC call) stops the batch after the
current unit of work rather than killing it mid-operation.

## Single-image vs. batch
The same underlying step functions (`ops::vision::detect/ocr/inpaint/render`) are used
both for one-off interactive calls (user reruns just OCR on one image) and inside the
batch loop — `ops::process` is a driver around them, not a separate implementation.
