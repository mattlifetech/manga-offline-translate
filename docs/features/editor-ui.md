# Editor UI

`ui/` — Next.js app, compiled to static assets and served by the Rust backend
(`koharu-rpc/src/server.rs`). Runs inside the Tauri webview.

## RPC client
`ui/lib/backend.ts` wraps the raw WebSocket transport (`ws.ts`); `ui/lib/api.ts`
builds typed calls on top, validating every response against a Zod schema
(`rpcSchemas.ts`) before handing it to the rest of the app — a malformed backend
response fails fast at the parse boundary (`parseWithSchema`) rather than propagating
`unknown` shapes into components. `rpc-types.ts` holds the corresponding TypeScript
types shared with the Rust side's `koharu-types::commands` contract.

## Data fetching
`ui/lib/query/` wraps the RPC client in React Query (`hooks.ts`, `mutations.ts`,
`keys.ts`) — standard query/mutation/cache pattern for document state, LLM model
lists, pipeline progress, etc.

## UI state (Zustand stores, `ui/lib/stores/`)
- `editorUiStore` — canvas/editor state: zoom scale, current document index, layer
  visibility toggles (segmentation mask, inpainted image, brush layer, rendered image,
  text block overlay), tool mode, selected block, render effect/stroke/color/align
  defaults.
- `llmUiStore` — LLM panel state (selected model/provider, target language, etc.).
- `operationStore` — tracks in-flight pipeline operations for progress UI.
- `preferencesStore` — persisted user preferences (referenced by `editorUiStore` for
  defaults).
- `uiErrorStore` — surfaces backend/RPC errors to the UI.

## Canvas (`ui/components/Canvas.tsx`, `canvas/*`)
The core editing surface: `Workspace.tsx` composes the image + overlay layers,
`ToolRail.tsx`/`CanvasToolbar.tsx` are the tool switcher, `TextBlockSpriteLayer.tsx` +
`TextBlockAnnotations.tsx` render/interact with detected text blocks,
`canvasViewport.ts`/`zoomGestures.ts` handle pan/zoom math, `StatusBar.tsx` shows
current state. Backed by hooks in `ui/hooks/`: `useCanvasZoom`, `useMaskDrawing`,
`useRenderBrushDrawing`, `useBlockDrafting`, `useBlockContextMenu`, `useTextBlocks`,
`usePointerToDocument` (screen ↔ document coordinate mapping), `useRpcConnection`.

## Panels (`ui/components/panels/`)
`LayersPanel` (visibility toggles), `TextBlocksPanel` (list/edit detected blocks),
`RenderControlsPanel` (font/effect/style controls feeding `text-rendering.md`'s
render step).

## Export
`CbzExportDialog.tsx` + `ui/lib/cbz-export.ts` — "Run all to CBZ" packaging of
processed pages into a single archive.

## i18n
`ui/lib/i18n.ts` + `ui/public/locales/<lang>/translation.json` — en-US, es-ES, ja-JP,
ru-RU, zh-CN, zh-TW.
