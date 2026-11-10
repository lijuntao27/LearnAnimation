---
type: raw-article
title: AnchorDEF - Learning Anchor Transformations for 3D Garment Animation (2023)
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, anchor, skinning, lbs, qem, cvpr-2023]
summary: Zhao 等（腾讯 AI Lab）CVPR 2023 论文：AnchorDEF 用网格表面自适应锚点的刚体变换混合 + canonical 空间逐顶点位移表示服装变形，位置/法向/方向一致性约束保锚点变换物理含义，QEM 网格简化监督把锚点推向褶皱与边界，宽松裙装与紧身 T 恤全面超过 TailorNet/VirtualBones。
---

# AnchorDEF: Learning Anchor Transformations for 3D Garment Animation (2023)

> 原文 PDF：`AnchorDEF-2023.pdf`（本目录）；提取文本：`AnchorDEF-2023.txt`。
> 中文译本：[[AnchorDEF-2023-中文翻译]]。
> arXiv: 2304.00761 · CVPR 2023 · 演示页：https://semanticdh.github.io/AnchorDEF
> 谱系定位：[[Pan2022-VirtualBones]] → **本文（锚点线）** → [[HyperBones2026]]（FPS 骨骼来源的先声）· 全景见 [[BoneDrivenCloth-CitationMap]]

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Fang Zhao, Zekun Li, Shaoli Huang*, Junwu Weng, Tianfei Zhou, Guo-Sen Xie, Jue Wang, Ying Shan |
| 机构 | 腾讯 AI Lab；ETH Zurich；南京理工大学 |
| 数据 | VTO 数据集（dress + T-shirt，Santesteban 2019）；训练序列长 30，4 clip 测试 |

## 核心机制一句话

服装变形 = **锚点刚体变换的 LBS 加权混合**（公式 2/5）+ **canonical 空间逐顶点非线性位移**（公式 5 的 D，蒙皮前加在模板上）；锚点变换由 GRU(body→anchor) 预测，宽松/紧身两种配置策略（公式 3/4，裙装可按到身体关节距离分类混用）。

## 关键设计

- **锚点初始化与稀疏权重**：k-means 聚类中心；每顶点只受 N'=8 最近锚点影响
- **宽松策略**：预测相对身体根关节的变换（利用全局朝向先验）；**紧身策略**：预测在最近身体顶点变换基础上的增量
- **L_consis**（公式 10）：变换后锚点须匹配 K=128 近邻变形顶点的均值位置+法向 → 锚点变换=邻域平均变换，防过拟合、保泛化
- **L_dir**（公式 11）：变换后锚点推向目标法向正侧 → 减少服装-身体穿透（锚点稀疏所以便宜）
- **自适应锚点更新**：QEM 网格简化顶点作监督（Chamfer，公式 12），锚点位置参数化为 K 近邻 softmax 注意力（公式 13/14）→ 锚点被推向褶皱与边界；在整个数据集上优化（模板平滑区摆姿后可能出新褶皱）
- 锚点数甜点 160（20/80/320 都差）；锚点过多测试期预测不稳定

## 主要结果

- Dress RMSE 16.05 / Hausdorff 74.20 / STED 0.0493（TailorNet 22.95/76.80/0.0757；VirtualBones 19.91/83.39/0.0722）
- T-shirt RMSE 6.25 / STED 0.0262（全面最优）
- 对 VirtualBones 的批评：SSDR 在拉普拉斯平滑后的低频序列上反解、骨骼位置/权重训练中固定 → 学到的骨骼未必最优；AnchorDEF 三者联合学习

## 局限

- 仍是 GRU 时序预测动态（继承 pose-driven 回归的泛化问题）
- 极端姿态仍会服装-身体穿透（无 self-collision）
- 缺细粒度动态（作者自述）

## 关联

- 被对比者与思想来源：[[Pan2022-VirtualBones]]（[26]）、TailorNet（[27]）、SSDR（[18]）
- 下游承接：[[HyperBones2026]] FPS 采样即此"mesh 面锚点"思路
- 项目对照：[[SkirtQNN-Design]]（L_consis/L_dir 可迁移为准静态网络的输出正则；宽松/紧身双策略对应裙腰紧/裙摆松）
