---
type: entity
title: 本地代码仓库索引（LearnAnimation 外部工程）
created: 2026-09-03
updated: 2026-09-07
tags: [index, code, local-repo, tooling]
summary: H:\Animation 下 4 个论文配套代码仓库 + 2 个个人工程（SkirtQNN、NeuralNetworksAndDeepLearning）的 upstream/本地路径/关联论文清单；均各自独立 git 管理，不纳入 LearnAnimation 版本控制。
---

# 本地代码仓库索引

> 记录 `H:\Animation` 下**不纳入 LearnAnimation git 仓库**的代码工程（父仓库只管理 `wiki/` 与根 `README`，见根 README「本地代码工程」节）。
> 这些目录在本地保留各自的 `.git`，修改由各自上游管理；LearnAnimation 通过根 `.gitignore` 排除。
> **在新机器恢复工作区**：clone 本仓库得到 `wiki/`，再按下表 upstream 逐个 `git clone` 到 `H:\Animation/` 下对应目录即可。

## 论文配套代码仓库（各有上游 git）

| 本地目录 | 上游地址 | 本地 HEAD（2026-09-03） | 关联论文 / wiki | 说明 |
|---|---|---|---|---|
| `motion_inbetweening/` | <https://github.com/victorqin/motion_inbetweening> | `fa9b6dc` (master) | [[RawSource/articles/Motion-In-betweening-2022\|Motion In-betweening 2022 通读]] · [[MotionInbetweeningTwoStage]] · [[Summary-MotionInbetweeningTwoStage]] | Motion In-betweening via Two-stage Transformers 官方代码（Qin 2022） |
| `VirtualBones/` | <https://github.com/non-void/VirtualBones> | `afb378e` (main) | [[RawSource/articles/Pan-2022-VirtualBones\|Pan 2022 通读]] · [[Pan2022-VirtualBones]] | Pan 2022 官方代码：SSDR 虚拟骨骼 + GRU/GNN 骨骼驱动布料；大权重走 git-lfs |
| `physkin/` | <https://github.com/facebookresearch/physkin> | `9877a8b` (main) | [[RawSource/articles/PhySkin-2026\|PhySkin 2026 通读]] · [[PhySkin2026]] | facebookresearch 发布的骨驱动布料神经框架；含嵌套 submodule `third_party/diffusion-net`（未 init） |
| `Dual-Quaternion-Skinning-Sample-Codes/` | <https://github.com/brainexcerpts/Dual-Quaternion-Skinning-Sample-Codes> | `d76994c` (master) | [[VirtualBones]]（骨驱动蒙皮语境）| LBS / Dual Quaternion 蒙皮教学示例（Kavan 思路）；本地含未提交演示 `lbs_vs_dqs.html` |

## 个人工程（独立 git，暂无 upstream）

| 本地目录 | 仓库 | 本地 HEAD（2026-09-03） | 关联 wiki | 说明 |
|---|---|---|---|---|
| `SkirtQNN/` | 独立 git（`git init`，无 remote） | `01aa3be` (main) | [[Jin2024-RopeChainQNN]] · [[QNN]] | 裙摆 QNN 运行时工程：从动画工程采集 身体→裙摆骨骼 姿势对 → 训练 QNN 预测裙骨 6D 变换 → 导出 ONNX/ORT → 写入引擎裙骨节点（onnxruntime 推理）。`venv*`、`extract/raw*` 捕获数据、`*.npz` 数据集不入库（见 `SkirtQNN/.gitignore`）。若后续需要可单独推送到 GitHub |
| `NeuralNetworksAndDeepLearning/` | 独立 git（`git init`，无 remote） | `1ea5d1c` (main) | 书 RawSource：[[RawSource/articles/NeuralNetworksAndDeepLearning-Nielsen-2015/ch01-handwritten-digits\|Nielsen《NN & DL》]]（6 章 md）· [[NeuralNetworkBasics]] · [[Backpropagation]] | 逐章精读 Nielsen《Neural Networks and Deep Learning》的 py3 移植配套代码：仿官方仓库 `src/` 平铺（`network.py`/`mnist_loader.py`…），仅机械迁移 py2→py3 保持与书对应；`data/mnist.pkl.gz` 不入库（gitignore）。无 upstream，本地独立 git |

> 更新时间 2026-09-07。HEAD 为记录当日的本地提交；上游仓库如有更新以 `git fetch` 为准。
