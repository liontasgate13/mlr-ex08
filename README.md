# mlr-ex08 — Leaf Classifier in the browser

Live leaf classification from the phone's rear camera, running entirely in the
browser. No server, no install.

**Page: https://liontasgate13.github.io/mlr-ex08/**

Point your phone camera at a leaf and the four class probabilities update on every
frame. Works on iPhone (Safari) and on desktop; if the camera is unavailable there is
a **Upload photo** fallback.

## What it does

`leaf_model.onnx` is applied exactly as described in `model_info.json`:

| step | detail |
| --- | --- |
| crop | central square of the frame (`side = min(w, h)`, centred) |
| resize | that square to `input_size` = 96 |
| scale | RGB bytes to 0–1 (`/255`) |
| standardise | per channel with `mean = [0.485, 0.456, 0.406]`, `std = [0.229, 0.224, 0.225]` |
| tensor | `1 × 3 × 96 × 96` float32, planar CHW, batch 1 |
| head | the graph's output `logits` is pre-softmax, so softmax is applied in JS |

Classes, in output-index order: `apple`, `cherry`, `chestnut`, `maple`.

Preprocessing happens once, on a hidden 96×96 canvas, and the same code path serves
the camera, the upload fallback and the self-check.

## Self-check

On load the page runs `check_image.jpg` through that same path and prints a table
against `check_expected.json`. Reference and page agree to ~3e-6:

| class | expected | page |
| --- | --- | --- |
| apple | 0.829588 | 0.829591 |
| cherry | 0.167952 | 0.167949 |
| chestnut | 0.002456 | 0.002456 |
| maple | 0.0000037 | 0.0000040 |

`check_image.jpg` is already exactly 96×96, so it validates scaling, standardisation,
CHW order and softmax, but not the crop. The crop was verified separately by feeding a
non-square image (a leaf padded with magenta bars) and comparing the browser's input
tensor with a reference implementation: it matched the true centre square
(mean |Δ| ≈ 0.007, i.e. resize-kernel noise only), while a top-left crop or an
aspect-squashed resize were ~4.4 out.

## Notes

- ONNX Runtime Web 1.31.1 from jsDelivr, WASM backend, `numThreads = 1`. Threads would
  need `SharedArrayBuffer`, which needs COOP/COEP headers GitHub Pages does not send,
  so the single-threaded path is deliberate.
- iOS Safari needs a user gesture for the camera, hence the **Start camera** button;
  the page also tries to start by itself and falls back to the button if refused.
  The camera requires HTTPS (or `localhost`).
- `<video playsinline muted>` keeps the stream inside the page on iOS rather than in a
  fullscreen player. The video element is square with `object-fit: cover`, so the
  picture you see *is* the central square the model receives.
- Uploaded JPEGs are decoded with `imageOrientation: 'from-image'`, so phone photos are
  un-rotated by their EXIF orientation before classification.
- `qr.png` is generated locally with the `qrcode` library and decoded back with
  `zxing-cpp` to confirm it encodes the live URL. No third-party QR service is used.
- The model reports what it was trained on: four classes. It is not a general leaf
  identifier.

## Files

| file | purpose |
| --- | --- |
| `index.html` | the whole app (plain JavaScript, no build step) |
| `leaf_model.onnx` | the classifier (PyTorch export, 4-class head) |
| `model_info.json` | classes, `input_size`, `mean`, `std` |
| `check_image.jpg` / `check_expected.json` | the self-check fixture |
| `qr.png` | QR of the page URL |

## Run locally

```sh
python -m http.server 8000   # then open http://localhost:8000/
```

`file://` will not work: fetching the model and camera access both need http(s).
