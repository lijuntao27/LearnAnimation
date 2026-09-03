---
type: entity
title: Decompressor（解压网络 D）+ Compressor（压缩网络 C）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, learned-motion-matching, neural-network, autoencoder]
summary: Learned Motion Matching 三网络之一，用 autoencoder 结构替代 Animation Database Y 的逐帧查表；输入 (x, z) 输出 pose y，是 DMM 单独可用形态。
---

# Decompressor（D）+ Compressor（C）

> [[LearnedMotionMatching]] 三阶段的「解压」阶段替代网络。对应论文 §4.1、Algorithm 1。

## 用途与定位

在 [[MotionMatching]] 中，**Decompression** 阶段是从索引 `i` 查 Animation Database `Y` 取出姿态 `y_i`。Decompressor 的目标是**消除对 Y 的内存依赖**：给定 `(x_i, z_i)`，直接产出 `y_i`。

单独训练好后即可独立使用（即 **DMM**，Decompressed Motion Matching）：存 `Z=[z_0,...]` 代替 `Y` 大幅省内存，不改变 MM 行为。见 [[Compare-MM-vs-LMM]]。

## Compressor C —— 隐变量发现

`x` 通常不足以完全推断 `y`（虽含足位/足速等关键信息）。引入 autoencoder 思路：
- **Compressor C**：`y_i → z_i`（低维隐变量，`z ∈ R^32`），同时输入 local 与 character space 数据（可把有用特征直接拷到隐空间，如 character space 足速利于预测接触信息）
- **Decompressor D**：`(x_i, z_i) → ỹ_i`，重构原姿态

C 只在训练时用，运行时不需要 → 可用更大网络与更贵激活函数。

## 网络架构

| 网络 | 层数 | 隐藏单元 | 激活 |
|---|---|---|---|
| C（Compressor） | 5 | 512 | ELU |
| D（Decompressor） | 3 | 512 | ReLU |

## 损失函数（关键设计，Algorithm 1）

朴素 MSE 会产生 jittery 低质动画。改用「视觉感知误差」：

| 损失 | 含义 |
|---|---|
| `L_loc` | local space 姿态误差 |
| `L_chr` | 经正向运动学（FK）映射到 character space 的误差 |
| `L_lvel` / `L_cvel` | local / character space 速度损失（相邻帧差分；避免用太噪的 `ẏr, ẏt` 加速度） |
| `L_lreg = ‖Z‖₂²` | L2 正则 |
| `L_sreg = ‖Z‖₁` | L1 稀疏 |
| `L_vreg = ‖(Z₀-Z₁)/δt‖₁` | 隐变量时间平滑 |

权重：姿态类损失大致等权，正则小权。`⊖` 算子算姿态差：基本相减，唯 `y_r` 旋转差先转旋转矩阵再相减。

## 训练约束

- 必须在三网络中**最先训练**（才能算出 `Z` 喂给 [[Stepper]] / [[Projector]]）
- 训练后 S、P 可并行；D 单独即 DMM

## 数值精度（Terrain 场景）

关节数值误差均值 **1.4cm**，标准差 1.1cm —— 视觉上平滑可接受。

## 关键接口（伪代码）

```
# 训练
Q  = ForwardKinematics(Y)            # character space
Z  = C([Y, Q]^T; θC)                 # 编码隐变量
Ỹ  = D([X, Z]^T; θD)                # 重构姿态
Q̃ = ForwardKinematics(Ỹ)            # 重算 FK
# 各 loss 见上表 → RAdam 更新 θC, θD

# 运行时（DMM / LMM 复用）
ỹ = D([x_i, z_i]; θD)
```

## 注意事项

- 隐变量维度 `z ∈ R^32`；存 Z 代替 Y 是 DMM 的核心省内存手段
- 速度损失用相邻帧差分，而非直接拟合 `ẏ`（原文：加速度太噪）
- 与 [[Stepper]]、[[Projector]] 配合时，z 由 Projector/Stepper 在运行时直接产出
