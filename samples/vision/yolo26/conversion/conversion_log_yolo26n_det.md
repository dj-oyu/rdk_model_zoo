# YOLO26n Detection Model Conversion Log

**Date:** 2026-02-02 20:14 - 20:17 (JST)
**Status:** Success

---

## Toolchain Versions

| Component | Version |
|-----------|---------|
| hb_mapper | 1.24.3 |
| hbdk | 3.49.15 |
| hbdk runtime | 3.15.55.0 |
| horizon_nn | 1.1.0 |

---

## Model Information

### Input
- **Name:** images
- **Shape:** 1x3x640x640
- **Type (runtime):** nv12
- **Type (train):** rgb
- **Layout:** NCHW
- **Normalization:** data_scale (0.003921568627451)

### Output (6 heads)
| Output | Shape | Description |
|--------|-------|-------------|
| output0 | [1, 80, 80, 4] | P3 bbox |
| 580 | [1, 80, 80, 80] | P3 cls |
| 588 | [1, 40, 40, 4] | P4 bbox |
| 602 | [1, 40, 40, 80] | P4 cls |
| 610 | [1, 20, 20, 4] | P5 bbox |
| 624 | [1, 20, 20, 80] | P5 cls |

---

## Conversion Parameters

| Parameter | Value |
|-----------|-------|
| BPU march | bayes-e |
| Quantization | int8 |
| Optimization | O3 |
| Compile mode | latency |
| Core num | 1 |
| Jobs | 16 |
| Input source | pyramid |

---

## Calibration

- **Method:** max-percentile (percentile=0.99995)
- **Samples:** 20 images (COCO128)
- **Data type:** float32

---

## Performance Metrics

| Metric | Value |
|--------|-------|
| **FPS** | 107.9 |
| **Latency** | 9267.8 us (9.3 ms) |
| **DDR** | 25,402,720 bytes (~24.2 MB) |
| Compile time | 110.966 s |

---

## Quantization Accuracy (Cosine Similarity)

| Output | Cosine Similarity | L1 Distance |
|--------|-------------------|-------------|
| output0 (P3 bbox) | 0.990516 | 0.229035 |
| 580 (P3 cls) | 0.999485 | 0.474841 |
| 588 (P4 bbox) | 0.993816 | 0.229287 |
| 602 (P4 cls) | 0.991239 | 1.462434 |
| 610 (P5 bbox) | 0.995228 | 0.224891 |
| 624 (P5 cls) | 0.990630 | 1.514647 |

**Average Cosine Similarity:** ~0.993

---

## Output Files

- `yolo26n_det_bpu_bayese_640x640_nv12.bin` (3.6 MB)

---

## Notes

- All layers run on BPU (no CPU fallback)
- Softmax optimized with int8 input/output
- Model uses attention mechanism (PSA blocks)
