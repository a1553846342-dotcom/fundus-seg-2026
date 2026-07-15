# Model Comparison: Paper Results vs Reproduced Results

**Date:** 2026-06-04

---

## 1. Custom U-Net

### Paper Reference

**Title:** U-Net: Convolutional Networks for Biomedical Image Segmentation  
**Authors:** Olaf Ronneberger, Philipp Fischer, Thomas Brox  
**Venue:** MICCAI 2015  
**arXiv:** 1505.04597  

**Abstract Summary:**  
U-Net is a fully convolutional network designed for biomedical image segmentation. It features a contracting path (encoder) to capture context and a symmetric expanding path (decoder) that enables precise localisation. The architecture uses skip connections to pass high-resolution feature maps from the encoder to the decoder. The network is trained end-to-end from very few images using data augmentation. On the ISBI cell tracking challenge, U-Net achieved state-of-the-art results.

### Paper Reported Results

The original U-Net paper reports results on biomedical cell segmentation datasets, not retinal fundus datasets. For **retinal lesion segmentation** specifically, the relevant benchmarks come from later works using U-Net as a backbone.

**DDR Dataset Benchmarks (from DDR dataset paper):**  
*Li et al., "Diagnostic Assessment of Deep Learning Algorithms for Diabetic Retinopathy Screening," Information Sciences 2019.*

| Model | mIoU (fg) | MA IoU | HE IoU | EX IoU | SE IoU |
|---|---|---|---|---|---|
| FCN-8s | ~0.14 | — | — | — | — |
| SegNet | ~0.16 | — | — | — | — |
| DeepLab v3 | ~0.22 | — | — | — | — |
| U-Net (reported) | ~0.18–0.25 | — | — | — | — |

> Note: Exact numbers depend on the specific split and preprocessing used in each paper. These are approximate reference values.

### Reproduced Results

| Split | Pixel Accuracy | Mean IoU (fg) | IoU MA | IoU HE | IoU EX | IoU SE |
|---|---|---|---|---|---|---|
| valid | 0.9867 | 0.0569 | 0.0193 | 0.0490 | 0.1593 | 0.0000 |
| test  | 0.9816 | 0.0965 | 0.0122 | 0.0800 | 0.2940 | 0.0000 |

**Performance Gap:** Valid mIoU 0.0569 vs paper ~0.18–0.25 (gap ≈ 0.12–0.19).

**Reasons for Gap:**
1. **No encoder pretraining** — training from scratch on only 338 images
2. **SE never detected** (IoU=0.0000) — only 1,207 training polygons, too rare for the model to learn
3. **MA barely detected** — 0.037% of pixels, ~2–5 px radius at 512×512
4. Published benchmarks use larger datasets and often pretrained backbones

---

## 2. SMP U-Net with ResNet34

### Paper Reference

**Title:** Segmentation Models PyTorch (SMP)  
**Authors:** Pavel Yakubovskiy  
**Year:** 2019–2021  
**Repository:** https://github.com/qubvel/segmentation_models.pytorch

**ResNet34 Encoder Reference:**  
*He et al., "Deep Residual Learning for Image Recognition," CVPR 2016.*

**Abstract Summary (ResNet):**  
Deep residual networks introduce shortcut (skip) connections that bypass one or more layers, allowing gradients to flow directly to earlier layers. This enables training of very deep networks (up to 152 layers). ResNet won ILSVRC 2015 on image classification, detection, and localisation.

### Expected Advantages vs Custom U-Net

1. **ImageNet pretraining:** ResNet34 encoder has seen 1.2M images across 1000 categories. Features like edges, textures, and local patterns are already learned, giving a strong initialisation.
2. **Residual connections in encoder:** Avoids vanishing gradient in the encoder.
3. **Less training data required:** Pretrained features transfer well to small medical datasets.
4. **Proven decoder:** SMP's U-Net decoder is a well-tested reference implementation.

### Reproduced Results

| Split | Mean Dice (fg) | Mean IoU (fg) | Dice MA | Dice HE | Dice EX | Dice SE |
|---|---|---|---|---|---|---|
| valid | 0.1902 | 0.1073 | 0.0726 | 0.1811 | 0.2990 | 0.2083 |
| test | 0.1995 | 0.1212 | 0.0502 | 0.2274 | 0.4629 | 0.0573 |

---

## 3. Full Comparison Table

| Model | Source | mIoU (fg) valid | mIoU (fg) test | Notes |
|---|---|---|---|---|
| U-Net (paper, DDR) | Li et al. 2019 | ~0.18–0.25 | — | DDR benchmark reference |
| Custom U-Net (ours) | This repo | 0.0569 | 0.0965 | FocalTversky + ForegroundPatch; SE=0 |
| **SMP ResNet34 (ours)** | This repo | **0.1073** | **0.1212** | Pretrained encoder; best overall |

---

## 4. Possible Reasons for Performance Gap

### Why Custom U-Net Underperforms vs SMP

| Reason | Evidence | Status |
|---|---|---|
| No encoder pretraining | mIoU 0.0569 vs SMP 0.1073 | Fundamental: fixed by using SMP |
| SE never detected | SE IoU=0.0000 on both splits | 1,207 training polygons — too few |
| MA barely detected | MA IoU=0.0193 valid | Too small at 512×512 resolution |
| Small dataset (338 images) | High inter-class variance | Partially mitigated by fg-biased sampling |

### Why SMP May Outperform Custom U-Net

1. **Pretrained encoder** — better initialisation, converges faster
2. **Residual blocks** — more stable gradient flow in encoder
3. **Class-weighted loss** — explicit upweighting of rare classes (MA weight 4× in CE)
4. **AdamW with weight decay** — regularisation prevents overfitting on small dataset

### Why Both May Still Struggle

1. **MA is extremely rare** — 0.037% of pixels, tiny dot-like lesions (~2–5 px radius at 512×512)
2. **Resolution loss** — original images are megapixels; 512×512 loses fine detail
3. **SE is scarce** — only 1,207 polygons in training vs 21,566 for EX

---

## 5. Next Steps

1. Run U-Net training on AutoDL with FocalTversky + ForegroundPatch
2. Run SMP training on AutoDL
3. Compare per-class results — if SMP >> U-Net, the architecture is the bottleneck
4. If SMP also fails on MA/SE → data difficulty and resolution are the bottleneck
5. Consider multi-scale or attention-based architectures for very small lesions
