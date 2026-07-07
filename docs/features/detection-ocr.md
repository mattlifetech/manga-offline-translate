# Detection & OCR

`koharu-pipeline/src/ops/vision.rs` (`detect`, `ocr`), `koharu-ml/src/comic_text_detector/`,
`koharu-ml/src/manga_ocr/`, `koharu-ml/src/mit48px_ocr/`, `koharu-ml/src/font_detector/`.

## Detection
`comic_text_detector` locates speech-bubble/text regions on a page (YOLOv5-style
detector + a U-Net/DBNet segmentation stage — see `yolo_v5.rs`, `unet.rs`, `dbnet.rs`,
`postprocess.rs`). Produces the initial set of `TextBlock` regions on the `Document`.

## OCR
Two OCR backends exist: `manga_ocr` (transformer-based, `bert.rs` + `model.rs` +
`tokenizer.rs`) and `mit48px_ocr`. Both read the pixels within a detected text region
and populate that block's recognized source text.

## Font detection
`font_detector` (`models.rs`) predicts likely font characteristics from the source
text region — feeds into rendering font choice (`text-rendering.md`) rather than OCR
itself.

## Flow
Both `detect` and `ocr` follow the same pattern in `ops/vision.rs`: read the current
`Document` snapshot via `state_tx::read_doc`, call the matching method on the shared
`Ml` facade, write the mutated snapshot back via `state_tx::update_doc`. Each can be
invoked standalone (interactive, one image) or as part of the full pipeline
(`pipeline-orchestration.md`).
