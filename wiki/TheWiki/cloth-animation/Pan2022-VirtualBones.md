---
type: concept
title: Pan 2022 — Bone-Driven Motion Networks
created: 2026-08-31
updated: 2026-09-02
tags: [cloth-animation, virtual-bones, gru, siggraph-2022]
summary: Pan 等 2022 SIGGRAPH 方法总览：SSDR 从物理模拟提取虚拟骨骼，GRU 学身体动作→骨骼 motion，GNN 补高频，RBF 集成模拟参数；裙摆 18546 顶点降为 80 骨骼，首篇 bone-driven loose garment，但 GRU pose-driven、无自碰撞、数据需求大。
---

# Pan 2022 — Bone-Driven Motion Networks

> Pan et al., SIGGRAPH 2022。原文全文存 [[RawSource/articles/Pan-2022-VirtualBones|RawSource/articles/Pan-2022-VirtualBones]]，配套仓库 `VirtualBones/`。

## 一句话

从物理模拟用 SSDR 提取 [[VirtualBones|虚拟骨骼]]，GRU 学"身体动作→骨骼 motion" + GNN 补高频细节 + RBF 集成模拟参数，首篇用 bone-driven 学宽松服装变形，但低频 GRU 是 pose-driven 的确定性回归、数据需求大、无 self-collision。

## 问题定义

输入身体动作序列 + 模拟参数 → 输出 loose garment 网格序列。交互速率。针对裙摆等宽松服装。

## 方法（三段）

| 层 | 机制 | 作用 |
|---|---|---|
| 表示 | SSDR 提 [[VirtualBones\|虚拟骨骼]] + 蒙皮权重 | 裙摆降为 80 骨骼（DOF ↓~77×）|
| 低频 | GRU(body→bone) + LBS | 学身体动作到骨骼的时序映射，重建大形 |
| 高频 | GRU(全局) ‖ EdgeConv-GNN(局部) | 补细节皱纹 |
| 参数 | RBF 集成 K=8 pivot network | 支持未见模拟参数 |

## 数据与结果

Houdini Vellum；3 件 dress + MMD 舞蹈；10 组参数 ×40000 帧。比 TailorNet/DNG/Santesteban2019：RMSE ↓~20%、Hausdorff/STED ↓~10%。

## 局限

- LBS 线性 artifact（建议 DQS）
- 无 self-collision
- 低频 GRU pose-driven、数据需求大、泛化差（[[Jin2024-RopeChainQNN\|Jin 2024]] 实证）

## 与方案的关系

- 表示层（SSDR 虚拟骨骼 + LBS + 高低频 + 碰撞 loss）可复用
- 生成层（GRU body→bone）若要 keyframe 驱动需换 in-betweening 网络
- 虚拟骨骼 motion 序列与 [[MotionInbetweeningTwoStage\|motion in-betweening]] 输入同构

## 关联

- 原文：[[RawSource/articles/Pan-2022-VirtualBones]]
- 对比：[[Compare-Pan2022-vs-Jin2024]]
- 概念：[[VirtualBones]]
- 演进：[[Jin2024-RopeChainQNN]]

## 后续与研究谱系（2026-09 补）

- **原班作者未出骨骼驱动宽松布料的直接续作**：一作 Pan 转向通用角色 rigging（TOG 2026「ACT」，把 SSDR/虚拟骨骼思路泛化到任意拓扑角色），Tianjia Shao 继续布料（2025 CGF 持久褶皱）。
- [[Jin2024-RopeChainQNN]] **作者与本文不重叠**（Yongxu Jin / Fedkiw，Stanford 组为主）——是"同研究线、不同班底"的推进，不是原班续作。
- 2026 Meta Reality Labs 的 [[HyperBones2026]] / [[PhySkin2026]] 承接本文骨骼驱动表示：骨骼改 FPS 采样、动态改教师蒸馏或主动放弃、身份改调制。详见 [[Compare-BoneDrivenCloth-Lineage]]。
