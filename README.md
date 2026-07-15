# fundus-seg-2026

糖尿病视网膜病变（DR）眼底病灶分割 —— 2026 暑期冲刺仓库。
数据集：**DDR**（4 类病灶 EX / HE / MA / SE），主指标 **mAUPR**。

## 目标
一个月内做出可发表成果：以 HRDecoder（MICCAI 2024）为骨干，针对微小病灶
（MA/SE）提出改进模块与尺寸感知损失，在 DDR + IDRiD 上做完整对比与消融。
详见 `暑期一个月冲刺-研究路线与目标.md` 与 `暑期冲刺-每周TodoList.md`。

## 目录结构
```
src/
  dataset.py            统一数据管线（勿新建 loader）
  model/                自建 U-Net baseline + train/evaluate/predict/loss
  models/m2mrf/         M2MRF 复现
  baselines/hrdecoder/  HRDecoder 复现
  baselines/smp/        SMP U-Net baseline
configs/                hrdecoder.yaml / m2mrf.yaml
train.py                统一入口: python train.py --model <name>
requirements.txt
midterm/                中期材料（图表、报告、对比表）
```

## 数据集（不入 git）
`dataset/Segmentation/{images,labels}/{train,valid,test}`
**用官方划分 383/149/225**，单独上传到 AutoDL。

## 环境
```bash
pip install -r requirements.txt   # PyTorch 按 CUDA 版本单独装
```

## 训练
```bash
python train.py --model hrdecoder
python train.py --model m2mrf
python train.py --model unet
```

## 开发约定
- 统一数据管线，勿新建 train.py / dataset.py / evaluate.py
- 一切用相对路径、pathlib，保持 AutoDL 兼容
- 主指标 mAUPR，数据用官方划分，先复现再创新
