# 1-bit Camera

A single-file browser app that converts live webcam input into a 1-bit dithered image in real time, rendered inside a faithful recreation of the Mac System 1 (1984) window chrome. No build step, no dependencies, no server — open the HTML file and go.

---

## Demo

Open `slowframe-1bit.html` in a modern browser and click **Allow Camera**.

Camera access is required. All processing is on-device. Nothing is transmitted.

---

## Features

- Live webcam → 1-bit dithered output using MediaPipe body segmentation
- Six ordered dither modes: Bayer 8×8, Clustered Dot, Void-and-Cluster, Horizontal Line, Diagonal 45°, Blue Noise
- Two color palettes: 1-BIT MAC (ink on cream) and PHOSPHOR (dim P31 green on black)
- Three macro-pixel sizes: 1×, 2×, 4×
- Dual EMA buffers for temporal stabilization — one on the segmentation mask, one on luminance
- Luminance quantization to suppress inter-frame flicker from camera sensor noise
- Debug panel with seven live parameter sliders
- Mac System 1 window chrome with authentic title bar stripe texture and close box
- Fully responsive — fills viewport, works on mobile

---

## How It Works

### Pipeline Overview

```
Webcam → proc canvas → MediaPipe segmentation → mask EMA → luma EMA → quantize → dither → display canvas
```

### 1. Video Capture

The browser's `getUserMedia` API provides webcam frames. Each frame is drawn mirrored onto a **processing canvas** capped at 256px on the short axis, preserving aspect ratio. This resolution cap keeps segmentation inference fast regardless of display size.

### 2. Body Segmentation

MediaPipe Tasks Image Segmenter runs on each frame using the `selfie_multiclass_256x256` model in `VIDEO` mode with GPU delegate. The model outputs six per-pixel confidence masks:

```
0 = background   1 = hair   2 = body-skin
3 = face-skin    4 = clothes   5 = others / accessories
```

A single person mask is built by taking the per-pixel maximum confidence value across channels 1–5. Using only one channel (e.g. channel 1, hair) produces a silhouette with hollow interior — all non-background classes must be combined.

### 3. Mask Stabilization

The person mask is stabilized using an exponential moving average:

```
mask[i] = mask[i] * (1 - α) + raw[i] * α
```

Lower α values smooth the silhouette edge over time, reducing boundary flicker at the cost of lag on fast movement. α is tunable via the **EMA α** debug slider.

### 4. Luminance Extraction and EMA

Perceptual luminance is computed at **display grid resolution** — not at proc canvas resolution — using the standard weighted formula:

```
lum = 0.299R + 0.587G + 0.114B
```

A second EMA buffer runs on these luminance values frame-to-frame. This is the primary anti-flicker mechanism: small variations in camera sensor output are averaged out before reaching the dither comparator. The luma EMA is tunable via the **LUMA EMA α** debug slider.

### 5. Brightness, Contrast, and Quantization

Brightness and contrast are applied around the midpoint:

```
lum = clamp((lum - 0.5) * contrast + 0.5 + brightness)
```

Luminance is then quantized to N discrete steps:

```
lum = round(lum / step) * step    where step = 1 / quantLevels
```

Quantization is the second anti-flicker mechanism. Once snapped to a discrete level, sub-threshold sensor noise cannot flip the value. Lower levels produce chunkier tonal jumps but maximum stability.

### 6. Ordered Dithering

Each person pixel is compared against the active 8×8 dither matrix at that grid coordinate:

```
bit = lum < matrix[x & 7][y & 7] * ditherScale ? INK : CREAM
```

Background pixels are written as the `BG` color — cream in 1-BIT MAC mode, pure black in PHOSPHOR mode.

### 7. Display

The 1-bit grid is written to an `ImageData` buffer and pushed to the display canvas via `putImageData`. The canvas is CSS-scaled to fill the screen with `image-rendering: pixelated` to preserve hard pixel edges at all macro-pixel sizes.

---

## Dither Modes

| Name | Description |
|---|---|
| Bayer 8 | Classic 8×8 dispersed ordered matrix — the historical standard for ordered halftoning |
| Cluster | Dots nucleate at cell centers and grow outward — traditional print / offset halftone feel |
| Void+Clu | Ulichney 1993 void-and-cluster algorithm — perceptually optimal ordered dither, no visible structure |
| H-Line | Threshold varies by row only, giving horizontal bands — CRT scanline / engraving aesthetic |
| Diag 45 | Diagonal 45° line screen — classic magazine and photocopy halftone |
| Blue NN | Perceptually uniform dispersed noise — filmic grain, most organic of the six |

---

## Color Modes

| Name | Description |
|---|---|
| 1-BIT MAC | Ink on cream. References the e-paper / aged print aesthetic of period Mac documentation and the warm cream tone of the physical Mac 128K case. Background fills with cream. |
| Phosphor | Dim P31 green on black. References the P31 phosphor coating of the Mac 128K's 9-inch CRT. Lit pixels glow `#33cc33`. Unlit background pixels are pure black — physically correct, as undriven phosphor emits no light. Full UI chrome recolors to match. |

---

## Requirements

Any modern browser with webcam access and support for:
- `getUserMedia`
- Canvas 2D
- ES modules (`type="module"`)
- WebAssembly (for MediaPipe Tasks Vision)

GPU delegate requires WebGL. Falls back to CPU automatically — inference continues, slower.

---

## License

MIT

---

## Notes for Modification

### Combining all mask channels is load-bearing — do not revert to a single channel

The MediaPipe `selfie_multiclass_256x256` model outputs six confidence masks. Channels 1–5 cover hair, body-skin, face-skin, clothes, and accessories respectively. Using only one channel produces a visually correct edge for that class (e.g. a clean hair outline) but a hollow interior — face and torso confidence lives in different channels. The per-pixel maximum across channels 1–5 is what produces a full-body mask. If you switch to a single channel for performance reasons, expect a hollow silhouette with a visible ring.

### The luma EMA must run at display resolution, not proc resolution

An earlier version of this pipeline ran the luminance EMA on the proc canvas (≤256px). At that resolution, downsampling has already averaged out most camera noise before the EMA runs — the slider has no perceptible effect. The EMA buffer is deliberately sized at `gridW × gridH` (full display resolution) so it operates on the actual pixel variation that reaches the dither comparator. If you move luma EMA back to proc resolution to save memory, the LUMA EMA α slider will appear to do nothing.

### Ordered dithering only — error-diffusion algorithms are not suitable for live video

Floyd-Steinberg, Atkinson, and similar error-diffusion algorithms produce superior results on static images but are inherently sequential: each pixel's output depends on accumulated error from all previous pixels in the scan. On live video, any luminance variation in one pixel propagates through the rest of the frame, creating whole-frame instability that upstream EMA and quantization cannot prevent. Ordered dithering is stateless per pixel — the same input always produces the same output at the same coordinate — which is why it is stable across frames. Do not add error-diffusion modes to the live pipeline.

### The two stabilization mechanisms target different noise sources

The luma EMA targets **temporal noise** — the same pixel varying across frames due to sensor noise. Quantization targets **threshold proximity** — a pixel whose luminance sits close to a Bayer threshold value and flips state due to sub-quantization-step variation. Both mechanisms are necessary. Removing EMA without increasing quantization levels, or vice versa, will increase flicker in different ways. Tune them together.

### ditherScale modifies matrix thresholds, not luminance

The **DITHER SCALE** slider multiplies the Bayer threshold value, not the luminance input. This shifts the ink density of the entire output — above 1.0 lowers thresholds (more ink), below 1.0 raises them (less ink). It is not equivalent to a brightness adjustment. Brightness shifts which luminance values land near thresholds; ditherScale shifts which thresholds land near luminance values. The perceptual result is similar but the two controls interact, and setting both simultaneously can push the output toward near-solid ink or near-solid cream.
