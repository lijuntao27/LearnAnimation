---
type: raw-article
title: Unified Particle Physics for Real-Time Applications (Macklin et al., SIGGRAPH 2014)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, unified-solver, rigid-bodies, fluids, cloth, gpu, flex, siggraph-2014]
summary: Macklin、Müller、Chentanez、Kim（NVIDIA）SIGGRAPH 2014 论文：以"粒子+约束"为唯一基石统一表示气体/液体/软体/刚体/布料——刚体=粒子集+shape matching 约束，接触与碰撞统一成粒子间约束、天然两向耦合；贡献含约束平均的并行 Jacobi 求解、近似激波传播加速刚体堆叠收敛、PBD 静/动摩擦、双向流体-固体耦合与气体模拟；NVIDIA Flex SDK 的直接来源。
---

# Unified Particle Physics for Real-Time Applications (SIGGRAPH 2014)

> 原文 PDF：`Macklin-2014-UnifiedParticlePhysics.pdf`（本目录，ACM TOG 正式版）；提取文本：`Macklin-2014-UnifiedParticlePhysics.txt`。
> 谱系定位：PBD 扩展期集大成 · NVIDIA Flex 的学术出处 · 阅读顺序见 [[PBDReadingGuide]] 选修篇目。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Miles Macklin, Matthias Müller, Nuttapong Chentanez, Tae-Yong Kim |
| 机构 | NVIDIA |
| 发表 | ACM Transactions on Graphics (SIGGRAPH 2014) 33(4), Article 153 |

## 核心机制一句话

**一切皆粒子+约束**：流体直接是粒子（密度约束）、布料是粒子网格（距离/弯曲约束）、刚体是粒子集合加一个 shape matching 约束（最小二乘刚体变换把粒子拉回形状）、毛发是粒子串——不同物态的相互作用（布-水、刚-水）自动变成"粒子与粒子之间的接触约束"，**两向耦合免费获得**，一套并行求解器全处理。

## 关键设计（论文自列五大贡献）

- **约束平均的并行执行**（§4）：PBD 的 Gauss-Seidel 顺序执行不利 GPU；改 Jacobi 式并行求解 + 多约束修正的平均/松弛策略
- **近似激波传播**（§5.2）：刚体堆叠收敛慢（底部压力要层层迭代传上去）；按堆叠深度对约束修正加权，让底部先解——大幅加速收敛
- **PBD 静/动摩擦**（§6.1）
- **双向流体-固体耦合**（§7.1）：流体密度约束与固体粒子在同一求解循环里互推
- **气体模拟**（§7.2）
- **粒子 phase 标识**：整型分组成员，方便控制属性与碰撞过滤（如布料自碰撞关闭）
- **设计哲学**：不用统一连续介质本构（对照 SPH 统一法），而是**每种材质定制位置级约束**——牺牲物理统一性换工程简单性

## 主要结果

- 淋浴兔子场景：布料、刚体、流体经约束耦合无缝交互
- GPU 实时跑多物态混合场景；NVIDIA **Flex SDK** 直接基于此论文实现（后来的 FleX → Newton 的技术脉络起点）

## 局限

- 精度让位于实时：作者明言"compromises in realism to meet real-time"
- 刚体表示比传统刚体引擎粗（shape matching 是近似）；堆叠仍需激波传播技巧辅助
- 约束平均（Jacobi）收敛比 Gauss-Seidel 慢——后来的 [[Macklin-2019-SmallSteps|Small Steps 2019]] 表明子步化 + 单迭代才是这对矛盾的真正解法

## 谱系与关联

- **上游**：[[Muller-2006-PositionBasedDynamics|PBD 2006]]（求解器）；Müller 2005 Shape Matching（刚体模块）；[[Macklin-2013-PositionBasedFluids|PBF 2013]]（流体模块）；Stam Nucleus 2009（统一求解器的先行者，速度层）
- **下游**：NVIDIA Flex；[[Macklin-2016-XPBD|XPBD 2016]]（同组下一步：给这套框架装上物理正确的柔度）；[[Macklin-2019-SmallSteps|Small Steps 2019]]（再下一步：子步化）
- **知识库关联**：[[SkirtQNN-Design]] 评估过的工业级统一物理方案（Flex 类）的学术底座
