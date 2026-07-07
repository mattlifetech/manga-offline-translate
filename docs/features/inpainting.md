# Inpainting

`koharu-pipeline/src/ops/vision.rs` (`inpaint`), `koharu-pipeline/src/ops/edit.rs`
(mask/brush editing), `koharu-ml/src/lama/`.

## Model
LaMa (`koharu-ml/src/lama/model.rs`) removes the original (source-language) text from
a detected region so translated text can be rendered cleanly on top. Has separate FFT
backend implementations for CPU, CUDA, and Metal (`lama/fft/`) — inpainting is the most
GPU-sensitive step in the pipeline; per the top-level README, an oversized/problematic
region can be skipped rather than aborting the whole batch if it would exceed GPU
allocation limits.

## Manual correction
Automatic inpainting isn't always perfect over complex art. `ops/edit.rs` exposes:
- `UpdateInpaintMaskPayload` — edit the inpaint mask for a region directly.
- `MaskMorphPayload` — morphological adjustment (grow/shrink) of a mask
  (`imageproc::distance_transform`).
- `UpdateBrushLayerPayload` — free-hand brush layer, for manual touch-up beyond the
  mask-based region editing.
- `InpaintPartialPayload` — re-run inpainting for a specific region/mask rather than
  the whole page.

## Output
Per the top-level README: batch processing writes `inpainted/` pages (post text
removal) next to the source images, so results are inspectable/archivable outside the
app's own cache.
