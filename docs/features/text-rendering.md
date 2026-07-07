# Text rendering

`koharu-pipeline/src/ops/vision.rs` (`render`, `list_font_families`), `koharu-renderer/`.

## What it does
Draws translated text back onto the (inpainted) image, replacing the removed source
text. Entry point is `Renderer::render` (`facade.rs`), called from `ops/vision.rs`
with the text block index, a `TextShaderEffect`, optional stroke style, and optional
font family override.

## Layout
`layout.rs` + `text/` handle the hard part: matching text to the original bubble's
shape/size.
- CJK text uses vertical writing mode (`WritingMode`, `text/script.rs`
  `writing_mode_for_block`) where appropriate.
- Latin-script translations get their own box-fitting heuristics
  (`text/latin.rs`): `expand_latin_layout_box_relaxed`/`_strict` grow the box to fit
  overflowing translated text, `latin_width_overflow_factor`/`latin_layout_underfilled`
  detect when a translation doesn't fit or looks too sparse, and
  `pick_better_latin_candidate` picks between layout attempts.
- `script.rs` also picks font families per text (`font_families_for_text`) and
  normalizes translated text for layout purposes.

## Fonts
`FontBook` (`font.rs`) manages loaded fonts plus a set of symbol fallback fonts
(`load_symbol_fallbacks`) for glyphs the primary font can't render.
`list_font_families` (in `ops/vision.rs`) exposes the available font list to the UI's
render-controls panel.

## Effects
`TextShaderEffect`/`TextStrokeStyle` (from `koharu-types`) control stroke/shading —
applied at render time via `shape.rs`/`renderer.rs` (backed by `tiny-skia`).

## Rerendering
Because render is a separate step from detect/ocr/inpaint/translate, the UI can change
font/effect/style and re-run just `render` without repeating the expensive ML steps
("Apply style to all slides" in the README's toolbar reference).
