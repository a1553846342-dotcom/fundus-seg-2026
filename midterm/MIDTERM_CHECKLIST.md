# Midterm Presentation Checklist

**Last updated:** 2026-06-04  
**Status key:** ✅ Ready | ⚠️ Partial | ❌ Missing (needs script run)

---

## Figures

### U-Net Figures (`midterm/figures/unet/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `unet_architecture.png` | ✅ Generated | YES | Block diagram: encoder / bottleneck / decoder / skip connections |
| `class_imbalance.png` | ✅ Generated | YES | Log-scale pixel counts + percentage bars (109:1 ratio) |
| `confusion_matrix.png` | ✅ Generated | YES | Row-normalised; reflects current IoU (EX detected, SE=0) |
| `unet_metrics.png` | ✅ Generated | YES | Per-class IoU bar chart, valid vs test, with mean dashed lines |
| `training_strategy.png` | ✅ Generated | YES | 4-stage loss evolution table |
| `dataset_class_distribution.png` | ✅ Copied | YES | Original bar chart from `analyze_dataset.py` |
| `sample_1_visualization.png` | ✅ Copied | YES | Original image + GT mask + overlay |
| `sample_2_visualization.png` | ✅ Copied | YES | Original image + GT mask + overlay |
| `sample_3_visualization.png` | ✅ Copied | YES | Original image + GT mask + overlay |
| `coverage_analysis.png` | ✅ Copied | YES | Per-image lesion coverage distribution |
| `audit_valid_inspection.png` | ✅ Copied | YES | Validation split random inspection |
| `split_consistency.png` | ✅ Copied | YES | Train/valid/test class distribution comparison |
| `unet_predictions/` | ❌ MISSING | NO | Run: `python src/model/predict.py` |
| `loss_curve.png` | ❌ MISSING | NO | Training log not saved; not available |

### SMP Figures (`midterm/figures/smp/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `smp_metrics.png` | ✅ Generated | YES | Per-class Dice + IoU bar charts (valid & test side-by-side) |
| `unet_vs_smp_comparison.png` | ✅ Generated | YES | Custom U-Net vs SMP all metrics side-by-side |
| `smp_predictions/` | ❌ MISSING | NO | Run: `python src/baselines/smp/predict.py` |
| `smp_loss_curve.png` | ❌ MISSING | NO | Training log not saved; not available |

---

## Tables (`midterm/tables/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `unet_metrics.csv` | ✅ Complete | YES | Real results from `evaluation0604.txt` |
| `unet_vs_smp.csv` | ✅ Complete | YES | Both models filled with actual numbers |
| `dataset_stats.csv` | ✅ Complete | YES | Train/valid/test class pixel distribution |

---

## Reports (`midterm/reports/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `unet_summary.md` | ✅ Complete | YES | Architecture, training strategy, real results, full analysis |
| `smp_reproduction.md` | ✅ Complete | YES | Architecture, training config, real results, analysis |

---

## Paper Notes (`midterm/paper_notes/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `model_comparison.md` | ✅ Complete | YES | Both models vs published DDR benchmarks; gap analysis |

---

## Presentation Materials (`midterm/ppt_materials/`)

| File | Status | Ready for PPT? | Notes |
|---|---|---|---|
| `unet_presentation.md` | ✅ Complete | YES | 7 slides with real results, bullet points, speaker notes |
| `smp_presentation.md` | ✅ Complete | YES | 7 slides with real results, bullet points, speaker notes |

---

## Actual Results Summary

### Custom U-Net (FocalTversky + ForegroundPatch + CLAHE)

| Split | Pixel Acc | mIoU (fg) | IoU MA | IoU HE | IoU EX | IoU SE |
|---|---|---|---|---|---|---|
| valid | 0.9867 | 0.0569 | 0.0193 | 0.0490 | 0.1593 | 0.0000 |
| test | 0.9816 | 0.0965 | 0.0122 | 0.0800 | 0.2940 | 0.0000 |

### SMP U-Net (ResNet34 / ImageNet)

| Split | mDice (fg) | mIoU (fg) | Dice MA | Dice HE | Dice EX | Dice SE |
|---|---|---|---|---|---|---|
| valid | 0.1902 | 0.1073 | 0.0726 | 0.1811 | 0.2990 | 0.2083 |
| test | 0.1995 | 0.1212 | 0.0502 | 0.2274 | 0.4629 | 0.0573 |

---

## Missing Items — Commands to Run

### Generate U-Net Prediction Visualizations

```bash
ssh -p 32246 root@region-9.autodl.pro
cd /root/fundus-lesion-segmentation
git pull
python src/model/predict.py
# Output: outputs/predictions/*.png
# Copy to: midterm/figures/unet/
```

### Generate SMP Prediction Visualizations

```bash
python src/baselines/smp/predict.py
# Output: outputs/smp/predictions/*.png
# Copy to: midterm/figures/smp/
```

---

## Overall Status

| Category | Ready | Missing |
|---|---|---|
| Architecture diagrams | ✅ 1 | — |
| Dataset analysis figures | ✅ 7 | — |
| Metric figures (U-Net) | ✅ 2 (confusion matrix + metrics) | loss curve |
| Metric figures (SMP) | ✅ 2 (metrics + comparison) | loss curve |
| Prediction visualizations | ❌ 0 | 2 (run predict.py for each) |
| Metric tables | ✅ 3 (all complete) | — |
| Written reports | ✅ 2 | — |
| Paper notes | ✅ 1 | — |
| PPT slide scripts | ✅ 2 | — |

**Only blocker for midterm:** prediction visualization PNGs (optional but recommended).  
All metrics, tables, reports, and slide scripts are complete and use real results.
