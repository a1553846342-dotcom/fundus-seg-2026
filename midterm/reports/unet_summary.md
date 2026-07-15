# U-Net Experiment Summary

**Project:** Retinal Lesion Segmentation — DDR / IDRiD Dataset  
**Date:** 2026-06-04  
**Evaluation source:** `outputs/unet/evaluation/evaluation0604.txt`

---

## 1. Model Architecture

| Item | Value |
|---|---|
| Architecture | Classic U-Net (encoder–decoder with skip connections) |
| Input size | 3 × 512 × 512 (RGB) |
| Output size | 5 × 512 × 512 (per-pixel class logits) |
| Parameters | ~31 million |
| Framework | PyTorch |

### Encoder

Each level uses a `DoubleConv` block: two consecutive 3×3 Conv → BatchNorm → ReLU sequences.  
Downsampling is performed by 2×2 MaxPool before each level.

| Level | Output Channels | Spatial Size |
|---|---|---|
| inc (DoubleConv) | 64 | 512 × 512 |
| down1 | 128 | 256 × 256 |
| down2 | 256 | 128 × 128 |
| down3 | 512 | 64 × 64 |
| down4 (bottleneck) | 1024 | 32 × 32 |

### Decoder

Upsampling uses ConvTranspose2d (stride=2), followed by concatenation with the skip connection, then a `DoubleConv` block.

| Level | Output Channels | Spatial Size |
|---|---|---|
| up1 | 512 | 64 × 64 |
| up2 | 256 | 128 × 128 |
| up3 | 128 | 256 × 256 |
| up4 | 64 | 512 × 512 |
| outc (1×1 Conv) | 5 | 512 × 512 |

See: `../figures/unet/unet_architecture.png`

---

## 2. Training Strategy

### Configuration

| Hyperparameter | Value |
|---|---|
| Optimizer | Adam |
| Learning Rate | 3 × 10⁻⁴ |
| Batch Size | 4 |
| Max Epochs | 100 |
| LR Scheduler | ReduceLROnPlateau (factor=0.5, patience=5) |
| Early Stopping | patience=20 epochs |
| Loss Function | FocalTverskyLoss (α=0.3, β=0.7, γ=0.75) |
| Patches per Epoch | 1500 |
| Foreground Bias | 80% patches centred on lesion polygons |

### Loss Function: Focal Tversky Loss

$$\text{TI}_c = \frac{TP + \epsilon}{TP + \alpha \cdot FP + \beta \cdot FN + \epsilon}, \quad \text{FTL} = \frac{1}{C-1}\sum_{c=1}^{C-1}(1-\text{TI}_c)^{\gamma}$$

- α=0.3, β=0.7 → penalises missed lesions (FN) more than false alarms (FP)  
- γ=0.75 → moderate focal weight on hard examples  
- Background excluded from loss computation

### Data Preprocessing

1. **CLAHE** per-channel in LAB space (clip_limit=2.0, tile_size=8×8)
2. **Gamma correction** γ=0.8 — brightens dark regions
3. **Resize** to 512×512 (images: `cv2.INTER_LINEAR`, masks: `cv2.INTER_NEAREST`)
4. **Normalise** to [0, 1] float

### Foreground-Biased Patch Sampling (`ForegroundPatchDataset`)

Standard full-image training fails because lesions occupy <1% of pixels. Solution:
- Crop 512×512 patches from native-resolution images
- 80% of patches centred on a randomly sampled lesion polygon (weighted by area)
- 20% random background crops
- Epoch index re-sampled every epoch (1500 patches/epoch)
- All 338 training images cached in RAM (~1–2 GB)

### Training Strategy Evolution

| Stage | Loss | Key Change | Outcome |
|---|---|---|---|
| Stage 1 | CrossEntropyLoss | Baseline | Class collapse: IoU=0 |
| Stage 2 | DiceCE (50/50) | + Dice term | Minor improvement |
| Stage 3 | DiceCE + class weights | + Per-class weighting | Still collapsed |
| Stage 4 (current) | FocalTversky | ForegroundPatch + CLAHE | **Non-zero detection** |

---

## 3. Dataset Statistics

| Class | ID | Train Pixels | Train % | BG ratio |
|---|---|---|---|---|
| Background | 0 | 87,796,031 | 99.09% | — |
| MA | 1 | 32,379 | 0.037% | ×2,714 |
| HE | 2 | 452,458 | 0.511% | ×194 |
| EX | 3 | 238,607 | 0.269% | ×368 |
| SE | 4 | 85,197 | 0.096% | ×1,031 |

**Overall background-to-foreground ratio: 109:1**

See: `../figures/unet/class_imbalance.png`

---

## 4. Experimental Results

### Validation Split

| Class | IoU |
|---|---|
| Background | 0.9877 |
| MA | 0.0193 |
| HE | 0.0490 |
| EX | 0.1593 |
| SE | 0.0000 |
| **Mean (foreground)** | **0.0569** |
| Pixel Accuracy | 0.9867 |

### Test Split

| Class | IoU |
|---|---|
| Background | 0.9824 |
| MA | 0.0122 |
| HE | 0.0800 |
| EX | 0.2940 |
| SE | 0.0000 |
| **Mean (foreground)** | **0.0965** |
| Pixel Accuracy | 0.9816 |

See: `../figures/unet/unet_metrics.png`  
See: `../tables/unet_metrics.csv`

---

## 5. Confusion Matrix

See: `../figures/unet/confusion_matrix.png`

The model now predicts non-background classes for a fraction of pixels. EX achieves the best recall due to its larger lesion size and higher training frequency. SE recall remains near zero — the model has not learned to detect soft exudates.

---

## 6. Problem Analysis

### 6.1 Class Collapse (Resolved for Most Classes)

The previous checkpoint predicted 100% background (IoU=0 everywhere). The ForegroundPatch + FocalTversky pipeline successfully broke the collapse:

| Class | Old IoU | New IoU (valid) | New IoU (test) |
|---|---|---|---|
| MA | 0.0000 | **0.0193** | 0.0122 |
| HE | 0.0000 | **0.0490** | 0.0800 |
| EX | 0.0000 | **0.1593** | **0.2940** |
| SE | 0.0000 | 0.0000 | 0.0000 |

### 6.2 SE Remains Fully Collapsed (IoU = 0.0000)

SE (soft exudate) IoU = 0.0000 on both valid and test. Two reasons:

1. **Scarcity:** Only 1,207 SE polygons in training (vs 21,566 for EX). SE occupies 0.096% of training pixels — the model rarely encounters enough SE to learn from.
2. **Imbalance within foreground:** Even with foreground-biased sampling, SE patches are ~10× rarer than EX patches, and SE lesions often overlap in visual appearance with bright exudate regions.

### 6.3 EX Outperforms Other Classes

EX (exudate) achieves the best IoU: valid 0.1593, test 0.2940. Reasons:
- Most frequent foreground class (21,566 polygons, 0.269% of pixels)
- Larger, brighter lesions with strong contrast against the fundus background
- More patches per epoch contain EX → better gradient signal

### 6.4 Test vs Validation Gap

EX improves significantly from valid (0.1593) to test (0.2940) — the test set contains larger exudate regions (0.377% of test pixels vs 0.071% on valid). This is dataset split variance, not generalisation improvement.  
MA degrades from valid (0.0193) to test (0.0122) — the test set has fewer MA pixels proportionally.

### 6.5 MA Remains Very Low

MA IoU: valid 0.0193, test 0.0122. MA lesions are:
- Rarest class: 0.037% of training pixels
- Smallest in size: microaneurysms are ~2–5 px radius at 512×512
- After downsampling from original megapixel images, most MA are near the resolution limit

### 6.6 Comparison with SMP Baseline

| Model | Valid mIoU (fg) | Test mIoU (fg) |
|---|---|---|
| Custom U-Net (old, collapsed) | 0.0000 | 0.0000 |
| **Custom U-Net (current)** | **0.0569** | **0.0965** |
| SMP ResNet34 | 0.1073 | 0.1212 |

The SMP model with a pretrained encoder outperforms the custom U-Net across all metrics. The gap (SMP 0.1073 vs U-Net 0.0569 on valid) reflects the advantage of ImageNet pretraining on a small dataset of 338 images.

See: `../figures/smp/unet_vs_smp_comparison.png`

### 6.7 Class Imbalance — Residual Issue

Despite foreground-biased sampling, the class imbalance problem is not fully solved:
- SE is effectively invisible to the model
- MA detection is marginal
- Background Dice (0.9877) dominates the training signal

Further improvements to consider:
- Increase fg_ratio further for rare classes (SE, MA)
- Per-class foreground sampling (guarantee a minimum number of SE/MA patches per epoch)
- Higher-resolution input (e.g. 768×768 or tiled inference)
- Attention mechanisms to focus on small lesion regions

---

## 7. Visual Results

Training data quality samples (not model predictions):
- `../figures/unet/sample_1_visualization.png`
- `../figures/unet/sample_2_visualization.png`
- `../figures/unet/sample_3_visualization.png`
- `../figures/unet/audit_valid_inspection.png`

> Prediction visualizations (`outputs/predictions/*.png`) can be generated by running `python src/model/predict.py` with the current checkpoint.
