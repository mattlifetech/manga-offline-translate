# RPC & MCP server

`koharu-rpc/src/rpc.rs`, `server.rs`, `mcp/`, `shared.rs`.

## HTTP server (`server.rs`)
One Axum `Router`, built in `build_router`, serving three things:
- `/ws` — the WebSocket RPC endpoint (below), state = `WsState { resources }`.
- `/mcp` — `StreamableHttpService` wrapping `KoharuMcp` (an MCP server instance per
  session, via `LocalSessionManager`) — see MCP section below.
- fallback — serves the embedded Next.js static build via `SharedAssetResolver` (a
  closure resolving a path to bytes+mime-type), falling back to `index.html` for SPA
  routing, 404 if even that resolver returns nothing.

## WebSocket RPC (`rpc.rs`)
Msgpack-encoded (`rmpv::Value`) request/response protocol:
- Incoming: `RawIncoming { id, method, params }`.
- Outgoing: `OutgoingMessage` is either `Response { id, result|error }` (tag `"res"`)
  or `Notification { method, params }` (tag `"ntf"`, used for things like pipeline
  progress — see `pipeline-orchestration.md`).
- `method` is dispatched against `koharu_types::Method` / the functions re-exported
  from `koharu_pipeline::operations` — this is the single place that maps a wire
  method name to a Rust handler function.

## MCP server (`mcp/`)
Exposes the same underlying document/pipeline operations as an MCP server, so external
AI agents (not just the bundled Next.js UI) can drive the app — e.g. batch-translate a
folder of images programmatically. `mcp/helpers.rs` adapts between MCP's tool-call
shape and the same `AppResources`/`operations` functions the WebSocket RPC uses, so
adding a new capability generally means adding it once at the `ops::` level and
exposing it from both `rpc.rs` and `mcp/mod.rs`.

See `docs/skills/manga-offline-translate-batch/` for a Claude Code skill built against
this MCP endpoint.

## Shared resources (`shared.rs`)
`SharedResources`/`get_resources` — the `Arc`-wrapped app state (ML facade, renderer,
document state, pipeline handle) both the WS and MCP endpoints are given `State`
access to, so both protocols operate on the same live app state.
