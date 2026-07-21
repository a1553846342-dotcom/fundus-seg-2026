# 新 Session 工作指南 · 第 0 周 + 第 1 周

> **给谁看：** 一个全新的 Claude session（冷启动、无上下文）。
> **你在为谁工作：** 用户是**项目负责人**，同时承担技术分工中的 **B 角色**。你既要帮他完成 B 的活，也要帮他**监督 A 的交付**。
> **使命：** 完成**第 0 周（开工准备）**与**第 1 周（对齐口径 + 复现基线）**。
> **一句话目标：** 把数据划分改回官方标准、把评测统一到 mAUPR、并把 HRDecoder 复现到已发表水平——为后续创新打好可信地基。
> **配套文档：** `暑期一个月冲刺-研究路线与目标.md`、`暑期冲刺-每周TodoList.md`

---

## 0. 开始前请先读这些（30 分钟上手）

按顺序读，读完就有足够上下文开工：

1. `CLAUDE.md` —— 项目铁律、目录结构、AutoDL/GitHub 工作流（**最重要**）
2. `src/dataset.py` —— 统一数据管线（禁止另建 loader）
3. `src/model/train.py`、`src/model/evaluate.py`、`src/model/loss.py` —— 现有训练/评测/损失
4. `configs/hrdecoder.yaml`、`configs/m2mrf.yaml` —— 复现基线配置
5. `src/baselines/hrdecoder/`、`src/models/m2mrf/` —— 两个 SOTA 复现代码
6. `midterm/reports/`、`暑期一个月冲刺-研究路线与目标.md` —— 现状与目标数字

---

## 1. 角色与分工（**重要**）

| 角色 | 谁 | 负责内容 |
|---|---|---|
| **B（执行）** | **用户（我）** | 统一评测脚本、M2MRF 复现、重算旧基线、产出【表 1】 |
| **A（执行）** | 另一位同学 | 官方数据划分、HRDecoder 复现、实验框架 |
| **负责人（监督）** | **用户（我）** | 盯全局进度、独立验收 A 的交付、守红线、管台账 |

**你（新 session）的定位：** 用户的技术搭档。
- 对标 **【B·我做】** 的任务 → 你直接动手实现。
- 对标 **【A·我监督】** 的任务 → 你不代替 A 干活，但要帮用户**准备验收清单、独立复算 A 的数字、发现问题及时预警**。

> 关键杠杆：**评测脚本掌握在 B 手里**。这意味着用户可以用自己的脚本独立复评 A 的 checkpoint，不必轻信 A 报的数字。这是本项目最重要的质量闸门。

---

## 2. 项目背景速览

**任务：** 糖尿病视网膜病变（DR）眼底图像**多病灶语义分割**。
**数据集：** DDR（分割子集 757 张），4 类病灶 + 背景，共 5 类。

| 语义标签 | 0 | 1 | 2 | 3 | 4 |
|---|---|---|---|---|---|
| 含义 | background | MA(微动脉瘤) | HE(出血) | EX(硬性渗出) | SE(软性渗出) |

- 类别极度不均衡，MA 最难（约占训练像素 0.037%，背景:MA ≈ 2712:1）。
- 图像原始分辨率 1500×1100 ~ 3200×2400，尺寸不一。
- **本领域主指标是 mAUPR**（逐病灶 PR 曲线下面积），不是 IoU。

**当前进度：**
- 自建 U-Net（从零）：mIoU 0.057/0.096，SE=0（太弱，仅作最低基线）
- SMP U-Net(ResNet34)：mIoU 0.107/0.121
- MMSeg 复现线：测试 mAUPR **44.23%** / mIoU 28.51%（**尚未复现到已发表水平**）

**SOTA 坐标（DDR 测试集，官方划分）：**

| 方法 | mAUPR | mDice | mIoU |
|---|---|---|---|
| M2MRF (PR 2022) | ~51.1 | ~49.2 | ~38.1 |
| MLNet (2024) | 51.81 | 49.85 | 37.19 |
| HRDecoder (MICCAI 2024) | ~52–53 | — | ~37–38 |

**差距 = 我们复现 44% vs 已发表 51–53%，第 1 周就是要补上这 ~8 个点。**

---

## 3. 环境与基础设施

| 项目 | 值 |
|---|---|
| **新 GitHub 仓库** | **https://github.com/sonderhyr-cyber/fundus-seg-2026** |
| 本地项目根目录 | `/Users/sonder/Desktop/fundus-seg-2026` |
| 旧仓库（仅供查阅历史） | `/Users/sonder/Desktop/fundus-lesion-segmentation` |
| AutoDL SSH | `ssh -p 32246 root@region-9.autodl.pro` |
| AutoDL 项目根 | `/root/fundus-lesion-segmentation` |
| GPU | A100-PCIE-40GB × 1 |
| 数据集根 | `dataset/Segmentation/{images,labels}/{train,valid,test}` |
| 输出目录 | `outputs/`；日志统一 `outputs/<模块名>/logs/stdout.log` |

**标准工作流（每次改代码）：**
```bash
# 本地：从项目根目录出发
cd /Users/sonder/Desktop/fundus-seg-2026
git add <改动文件> && git commit -m "描述" && git push

# AutoDL：先 SSH 登录
ssh -p 32246 root@region-9.autodl.pro
cd /root/fundus-lesion-segmentation && git pull && python <脚本>
```

> AutoDL 端如尚未指向新仓库，需更新 remote 到 `fundus-seg-2026` 后再 pull。

---

## 4. 铁律（违反 = 结果不可用）

1. **只做 DDR 分割这一条线。** 不碰 YOLO 分类/分级（那是另一篇论文）。
2. **数据用官方划分 383/149/225。** 用别的划分 = 无法和任何论文对比 = 不可发表。
3. **主指标 mAUPR**（附 mDice、mIoU），口径必须和 M2MRF/HRDecoder 论文一致。
4. **先复现、再创新。** 第 1 周不引入任何新模型/新模块。
5. **不新建 `train.py` / `dataset.py` / `evaluate.py`。** 统一管线，只在现有文件内扩展。
6. **一切相对路径 + `pathlib`**，保持 AutoDL 兼容；不硬编码 `/Users/...`。
7. **重实验先本地冒烟测试（CPU/MPS），真训练放 AutoDL。**
8. **每一步先验证再前进**，改完代码走标准 git 工作流同步到 AutoDL。

---

## 5. 第 0 周任务：开工准备（约半天~1 天）

### T0.1 环境自检（AutoDL）【B·我做】
- [ ] SSH 登录 AutoDL，确认 GPU 可见：`nvidia-smi`
- [ ] 确认依赖（PyTorch 2.0.1+cu118 / MMCV 1.7.2 / MMSegmentation 0.16.0）：
  ```bash
  python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
  python -c "import mmcv, mmseg; print(mmcv.__version__, mmseg.__version__)"
  ```
- [ ] 确认 AutoDL 端 git remote 已指向 `fundus-seg-2026`
- [ ] 若缺依赖，只在 `requirements.txt` 增补，不改已有版本
- **验收：** `nvidia-smi` 正常、`torch.cuda` 为 True、mmcv/mmseg 可导入

### T0.2 建实验台账【B·我做，负责人维护】
- [ ] 建实验台账（仓库内 `docs/experiments.csv` 或在线表格），列：
  `实验ID | 日期 | 负责人 | 模型 | 配置 | 划分 | 关键超参 | mAUPR | mDice | mIoU | EX/HE/SE/MA各类AUPR | checkpoint | 备注`
- [ ] **作为负责人**：明确要求 A 每次训练也必须登记
- **验收：** 表格存在、列齐全、A/B 双方都已知晓登记义务

### T0.3 分支与目录规范【B·我做】
- [ ] 建工作分支：`git checkout -b sprint-week1`
- [ ] 约定日志落 `outputs/<模块名>/logs/stdout.log`，checkpoint 落 `outputs/checkpoints/`
- **验收：** 分支创建成功，规范写入 README

### T0.4 砍线与对齐【负责人·我】
- [ ] 向 A 明确本月冻结 YOLO 分类线与自建 U-Net 调参，只推进 MMSeg 分割线
- [ ] 与 A 确认第 1 周分工与交付时间点
- **验收：** A 已确认分工与红线

**第 0 周红线：** 环境不通、无法在 AutoDL 启动一次训练 → 先解决环境，不进入第 1 周。

---

## 6. 第 1 周任务：对齐口径 + 复现基线

### T1.1 恢复官方数据划分 383/149/225 【A 做 · 我监督】
现状 338/149/208，**训练集少约 45 张**（很可能过滤了"无病灶"图）。

A 需交付：
- 官方划分清单文件（如 `splits/{train,val,test}.txt`），来源以 DDR 原始发布 / M2MRF / HRDecoder 仓库为准
- `dataset.py` 读取该清单，划分固化不再变
- 三划分的图像数与逐类像素占比打印

**我（负责人）的验收动作：**
- [ ] 独立跑一遍计数，确认 `383/149/225`，不接受口头汇报
- [ ] 抽查若干文件名是否与官方清单一致
- [ ] 确认 mask 插值仍是 `cv2.INTER_NEAREST`（改了会污染标签）
- [ ] 确认排查结论：原来的 338 是怎么来的、是否已还原

### T1.2【最高优先级】升级统一评测脚本（主指标 mAUPR）【B·我做】
现有 `evaluate.py` 只用 argmax 混淆矩阵算 IoU。**AUPR 必须用 softmax 概率图算，不能用 argmax。**

在现有 evaluate 内扩展（不新建文件）：
- [ ] 前向取 `softmax(logits)`，对每个前景类 c∈{1,2,3,4}：
  - 收集全体像素该类**预测概率** `p_c` 与**二值 GT** `(mask==c)`
  - `AUPR_c = average_precision_score(gt_c.flatten(), p_c.flatten())`
- [ ] 逐类输出 AUPR / Dice / IoU，再求前景均值 mAUPR / mDice / mIoU
- [ ] 背景类（0）不计入均值，口径对齐 M2MRF/HRDecoder
- [ ] 结果打印 + 落盘 `outputs/<模型>/eval.txt`
- [ ] 内存注意：逐图流式累计，不要一次性堆全部像素概率
- [ ] 提供一个"评测任意 checkpoint"的入口，方便我复评 A 的模型

```python
from sklearn.metrics import average_precision_score
# 每图: probs [C,H,W] = softmax(logits); mask [H,W]
# 对每个前景类累积 (score, label)，最后统一算 AP
```

- **验收：** 能对任意 checkpoint 输出逐类+均值指标；与 A 各跑一次同一 checkpoint，数字一致（±0.1%）

### T1.3【本周锚点】复现 HRDecoder 到已发表 ±1% 【A 做 · 我监督】
A 需交付：
- `python train.py --model hrdecoder` 训练完成
- 日志 `outputs/hrdecoder/logs/stdout.log` + checkpoint `outputs/checkpoints/hrdecoder_best.pth`
- 官方 test(225) 上的 mAUPR

**我（负责人）的验收动作：**
- [ ] **用我自己的 T1.2 脚本独立复评 A 的 checkpoint**，不采信 A 单方数字
- [ ] 确认落在已发表 ±1%（~51–53% mAUPR）
- [ ] 若差距大，和 A 一起排查：划分 / 输入分辨率 / iter 数 / 损失 / 学习率调度是否与论文一致
- [ ] 确认日志与权重已按规范落盘、已登记台账

### T1.4 复现 M2MRF 作为第二基线【B·我做】
- [ ] `python train.py --model m2mrf`，用统一脚本评测，登记逐类 AUPR
- [ ] 与论文对齐（DDR mAUPR ~51%）
- **验收：** M2MRF 有一组可信的 mAUPR / 逐类结果

### T1.5 用新脚本重算旧基线【B·我做】
- [ ] 用 T1.2 脚本对 自建 U-Net、SMP U-Net 已有 checkpoint 重新评测（补 mAUPR）
- [ ] 旧 checkpoint 若是在旧划分上训练的，需在官方划分上重训，或至少在官方 test 上评测并注明
- **验收：** 四个基线在统一官方划分 + 统一口径下都有数

### T1.6 产出【表 1】统一口径 baseline 对比表【B·我做】
- [ ] 行 = 方法（UNet / SMP / M2MRF / HRDecoder），列 = mAUPR / mDice / mIoU + 逐类 AUPR
- [ ] 存 `docs/table1_baselines.md`（或 csv），登记进台账
- **验收：** 表 1 完成，可直接进论文实验章节初稿

---

## 7. 负责人监督清单（用户专属）

### 7.1 每日快检（5 分钟）
- [ ] A 今天有无提交？commit 是否遵守铁律（没新建 pipeline 文件、没硬编码路径）？
- [ ] 昨天跑的实验有没有登记台账？
- [ ] 有没有人偷偷改了数据划分或 mask 插值？

### 7.2 关键闸门（必须亲自把关，不可代劳）
| 闸门 | 验收方式 | 不通过怎么办 |
|---|---|---|
| 官方划分 383/149/225 | 我独立跑计数核对 | 卡住，不许往下走 |
| 评测口径一致 | 同 checkpoint 双人复算 ±0.1% | 先修脚本 |
| **HRDecoder 复现 ±1%** | **我用自己的脚本独立复评** | **不进第 2 周** |

### 7.3 周末对账会议清单
- [ ] 表 1 是否齐全（四基线 × 全指标）？
- [ ] HRDecoder 是否达标？没达标的根因是什么、下周怎么补？
- [ ] 台账是否完整可复盘？
- [ ] 代码是否已 commit + push，AutoDL 是否同步？
- [ ] 下周（创新点实现）的分工与前置条件是否就绪？

### 7.4 风险预警（发现即上报/干预）
- 🚨 A 报的数字我复算不出来 → 立刻核对划分与评测口径
- 🚨 HRDecoder 复现卡住超过 3 天 → 考虑降级：先用 M2MRF 当主基线
- 🚨 有人开始"顺手"做新模块/新架构 → 违反"先复现再创新"，立即叫停
- 🚨 台账断更超过 2 天 → 一周后无法复盘，立即补齐

---

## 8. 第 1 周交付物清单（Definition of Done）

**B（我）交付：**
- [ ] `evaluate.py` 升级：逐类+均值 **mAUPR/mDice/mIoU**，双人复算一致
- [ ] M2MRF 第二基线结果
- [ ] 四基线统一口径重算完成
- [ ] 【表 1】统一口径对比表

**A 交付（我验收）：**
- [ ] 官方划分 383/149/225 固化 + 可复现核对
- [ ] **HRDecoder 复现 mAUPR 在已发表 ±1% 内**（本周最重要的锚点）

**共同：**
- [ ] 所有实验登记台账，日志/权重规范落盘，代码已 commit + push

---

## 9. 常见坑与提醒

- **AUPR 用概率不用 argmax**：升级评测最容易做错的点。
- **划分不一致会让所有对比作废**：T1.1 必须先做且做对。
- **HRDecoder 没复现到位就别进第 2 周**：没有可信 baseline，后续"提升"不可信。
- **mask 插值 `INTER_NEAREST` 不可改**。
- **别在本地跑完整训练**：本地只冒烟测试，真训练在 A100。
- **每次实验必登记台账**。
- **不轻信口头数字**：一切以我用自己脚本复算的结果为准。

---

## 10. 参考文献

- M2MRF (PR 2022) — https://arxiv.org/pdf/2111.00193
- HRDecoder (MICCAI 2024) — https://arxiv.org/abs/2411.03976 · 代码 https://github.com/CVIU-CSU/HRDecoder
- MLNet (2024) — https://doi.org/10.3390/a17040164
- DDR 数据集基准 — https://arxiv.org/pdf/2008.09772

---

**本周结束时向用户汇报：四基线统一口径对比表 + HRDecoder 复现是否达标（±1%）。达标才进入第 2 周。**
