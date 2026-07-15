# U-Net Section — Presentation Materials

**Target duration:** 5–8 minutes  
**Audience:** Project midterm presentation (ML/CV course)  
**Evaluation source:** `outputs/unet/evaluation/evaluation0604.txt`

---

## Slide 1: Section Title

**Title:** U-Net for Retinal Lesion Segmentation

**Bullet points:**
- Task: Semantic segmentation of 4 lesion types (MA, HE, EX, SE) in fundus images
- Architecture: Classic U-Net trained from scratch (~31M parameters)
- Dataset: 338 training images, YOLO polygon annotations → semantic masks
- Core challenge: Extreme class imbalance — background occupies 99.09% of pixels

**Figure recommendation:** `../figures/unet/sample_1_visualization.png`  
Three-panel: original fundus image | ground-truth mask (colour-coded) | overlay. Good visual hook.

**Speaker notes:**  
"My section covers the custom U-Net we built from scratch and the challenges we went through to get it working. The core difficulty is not the architecture — it's that the lesions we want to detect occupy less than 1% of the image."

---

## Slide 2: Dataset & Class Imbalance

**Title:** The Core Challenge — Extreme Class Imbalance

**Bullet points:**
- 5 classes: background (99.09%) + MA, HE, EX, SE (total 0.91%)
- Background-to-foreground ratio: **109:1**
- MA (microaneurysm): only **0.037%** of all pixels — rarest and smallest
- A trivial "predict all background" classifier achieves **99.5% pixel accuracy**
- → Pixel accuracy is meaningless here; we use mean IoU over foreground classes

**Figure recommendation:** `../figures/unet/class_imbalance.png`  
Log-scale pixel counts + percentage bar chart.

**Speaker notes:**  
"This is why standard accuracy is useless for this problem. The confusion matrix initially showed 100% of lesion pixels classified as background, yet accuracy was 99.6%. The real test is mean IoU over the four lesion classes."

---

## Slide 3: U-Net Architecture

**Title:** U-Net Architecture

**Bullet points:**
- **Encoder:** 5 levels, 64 → 128 → 256 → 512 → 1024 channels, MaxPool downsampling
- **Decoder:** 4 upsampling levels, ConvTranspose2d + skip connection concatenation
- **DoubleConv block:** Conv3×3 → BN → ReLU → Conv3×3 → BN → ReLU (at every level)
- Skip connections pass spatial detail from encoder to decoder
- Output: 5-channel logit map → argmax → per-pixel class label
- Parameters: ~31 million

**Figure recommendation:** `../figures/unet/unet_architecture.png`  
Clean block diagram with encoder, bottleneck, decoder, and skip connections labelled.

**Speaker notes:**  
"The architecture is the standard U-Net from Ronneberger et al., 2015. Each encoder level halves the spatial size and doubles the channels. The decoder reverses this with transposed convolutions. Skip connections are what allow the model to be both context-aware (bottleneck) and spatially precise (skip)."

---

## Slide 4: Training Strategy

**Title:** Training Strategy — Solving Class Imbalance

**Bullet points:**
- **Problem:** Full-image CrossEntropy → model learns "predict background" to minimise loss
- **Fix 1 — Foreground-biased patch sampling:**  
  80% of training crops are centred on a lesion polygon (guaranteed foreground)
- **Fix 2 — Focal Tversky Loss** (α=0.3, β=0.7, γ=0.75):  
  Penalises missed lesions (FN) 2.3× more than false alarms (FP)
- **Preprocessing:** CLAHE local contrast enhancement + gamma correction (γ=0.8)
- Optimizer: Adam (lr=3×10⁻⁴), ReduceLROnPlateau scheduler, early stopping patience=20

**Figure recommendation:** `../figures/unet/training_strategy.png`  
4-stage evolution table showing what changed and why.

**Speaker notes:**  
"The key insight is that re-weighting the loss was not enough. We had to change how training patches are sampled. With ForegroundPatchDataset, 80% of every batch contains at least one lesion — so the gradient signal is actually driven by lesion detection rather than background memorisation."

---

## Slide 5: Experimental Results

**Title:** Experimental Results

**Bullet points:**

| Class | Valid IoU | Test IoU |
|---|---|---|
| Background | 0.9877 | 0.9824 |
| MA | 0.0193 | 0.0122 |
| HE | 0.0490 | 0.0800 |
| EX | 0.1593 | **0.2940** |
| SE | 0.0000 | 0.0000 |
| **Mean (foreground)** | **0.0569** | **0.0965** |
| Pixel Accuracy | 0.9867 | 0.9816 |

**Key takeaways:**
- Class collapse is resolved for MA, HE, EX — the model detects lesions
- **EX is the strongest class** (Test IoU 0.2940): largest, brightest, most frequent lesion
- **SE remains undetected** (IoU=0.0000): only 1,207 training polygons, too few to learn

**Figure recommendation:** `../figures/unet/unet_metrics.png`  
Per-class IoU bar chart, valid vs test side-by-side, with mean IoU dashed lines.

**Speaker notes:**  
"Three out of four lesion classes are now detected. EX — exudates — achieves 0.29 IoU on the test set, which is a meaningful result. SE — soft exudate — has zero IoU on both splits because there are only 1,207 training examples of this class, and even with foreground-biased sampling it's rarely seen."

---

## Slide 6: Problem Analysis

**Title:** Analysis — What's Still Hard

**Bullet points:**

**SE: Fully undetected (IoU = 0.0000)**
- 1,207 training polygons vs 21,566 for EX — 18× fewer
- The foreground sampler still rarely picks SE patches

**MA: Marginal detection (IoU ≤ 0.019)**
- Rarest class: 0.037% of all pixels
- Microaneurysms are ~2–5 px radius at 512×512 — near the resolution limit
- Original megapixel images are heavily downsampled

**EX improves on test vs valid (0.1593 → 0.2940)**
- Test set has proportionally larger exudate regions (0.377% vs 0.071%)
- Not generalisation improvement — dataset split variance

**Comparison with SMP baseline:**
| Model | Valid mIoU | Test mIoU |
|---|---|---|
| Custom U-Net (ours) | 0.0569 | 0.0965 |
| SMP ResNet34 | 0.1073 | 0.1212 |

**Figure recommendation:** `../figures/smp/unet_vs_smp_comparison.png`  
Side-by-side bar chart across all metrics.

**Speaker notes:**  
"The SMP model with a pretrained ResNet34 encoder outperforms our custom U-Net by about 2×. This tells us two things: the data pipeline is correct, and pretraining matters a lot when you only have 338 images. The custom U-Net is learning from scratch, which is a much harder task."

---

## Slide 7: Conclusions & Next Steps

**Title:** Summary & Next Steps

**Bullet points:**

**Achieved:**
- Broke class collapse — MA, HE, EX now detected
- EX IoU = 0.2940 on test (meaningful result)
- Identified exact failure modes per class

**Remaining problems:**
- SE: needs per-class minimum sampling (not just random fg bias)
- MA: needs higher input resolution or tiled inference
- Gap to pretrained SMP: ~2× on mIoU — pretraining is a major advantage

**Next steps:**
- Per-class stratified sampling to guarantee SE exposure
- Explore higher resolution (768×768) or patch-based inference
- Attention mechanisms for tiny lesion detection

**Figure recommendation:** `../figures/unet/coverage_analysis.png`  
Per-image foreground coverage — shows how sparse lesions are across the dataset.

**Speaker notes:**  
"The main lesson is that solving class imbalance in segmentation requires changing both the loss function and the data sampling strategy. We have a working model now, but SE and MA remain hard problems. The path forward is either more targeted sampling or higher resolution."

---

## Timing Guide

| Slide | Suggested Time |
|---|---|
| 1 — Title | 30 sec |
| 2 — Class Imbalance | 1 min |
| 3 — Architecture | 1 min |
| 4 — Training Strategy | 1.5 min |
| 5 — Results | 1.5 min |
| 6 — Analysis | 1 min |
| 7 — Conclusions | 30 sec |
| **Total** | **~7 min** |
