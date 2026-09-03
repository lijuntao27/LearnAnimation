---
type: concept
title: QNN（准静态神经网络）
created: 2026-08-31
updated: 2026-08-31
tags: [cloth-animation, qnn, deep-learning]
summary: 不含时序信息、逐帧独立处理的神经网络，学"状态本身"而非"状态转移"，数据需求小、泛化好；代价是忽略弹道动态，必须物理补。Jin 2024 用 5000 帧训练。
---

# QNN（准静态神经网络）

> [[Jin2024-RopeChainQNN|Jin 2024]] 提出的形状推断网络，相对 RNN/GRU 的省数据替代。

## 定义

QNN = quasistatic neural network = "neural networks that lack temporal information"（Jin 2024 §1）。不含时序信息、逐帧独立处理，假设每帧形状只依赖当前状态（准静态）。

## 与 RNN/GRU 对比

| | RNN/GRU | QNN |
|---|---|---|
| 学什么 | 状态间时序转移 | 状态本身 |
| hidden state | 有 | 无 |
| 数据需求 | 大 | 小（5000 帧级）|
| 泛化 | 差 | 好 |
| 弹道动态 | 可学 | 不能（需物理补）|

## 为什么省数据

学"状态转移"要覆盖各种动态情形（数据量大）；学"状态本身"只需覆盖位姿空间（数据量小）。Jin 2024：每帧 non-sequential，减高容量网络需求、减数据负担、避免过拟合。

另一重机制在输出侧：QNN 只输出 ~100 个 PCA 系数，重建靠写死的线性公式 `网格 = 基线 + Σ wᵢ·Bᵢ`。PCA 子空间把有效自由度从 55638 顶点压到 100，既是降维也是正则——数学见 [[PCA]]。

## 代价与补救

准静态忽略惯性/飞甩。Jin 用 [[RopeChain|rope chain]] 物理补弹道动态，QNN 只学形状。

## 工程红利（无状态）

逐帧独立 → 动画打断无残留、可降频/LOD、可 batch 多角色、确定性可验收。

## 关联

- [[Jin2024-RopeChainQNN]]（提出者）
- 对比：[[RecurrentNetworks]]（RNN 基础，被 QNN 替代的对象）
- 数学：[[PCA]]（PCA 子空间、SVD、为什么 100 系数能重建网格）
- 应用：[[cloth-animation/_index]]
