---
type: entity
title: Stepper（步进网络 S）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, learned-motion-matching, neural-network, autoregressive]
summary: Learned Motion Matching 三网络之一，自回归地预测 (x,z) 的帧间增量，替代 Matching/隐变量库的逐帧查表，承担系统全部动力学。
---

# Stepper（S）

> [[LearnedMotionMatching]] 三阶段的「步进」阶段替代网络。对应论文 §4.2、Algorithm 2。

## 用途与定位

[[MotionMatching]] 的 **Stepping** 阶段：每帧 `i := i+1` 后从 `X`、`Z` 查下一行。要消除对 X、Z 的逐帧查表依赖，引入 Stepper：输入 `(x_i, z_i)`，输出 delta，加到当前得 `(x_{i+1}, z_{i+1})`。

> 一旦配上 [[Decompressor]] 单独使用，Stepper 能产出连续 `(x,z)` 流；但最近邻搜索仍需 X、Z 在内存 → 必须再有 [[Projector]] 才能彻底脱离数据库。

## 网络架构

| 层数 | 隐藏单元 | 激活 |
|---|---|---|
| 4 | 512 | ReLU |

## 训练（Algorithm 2，自回归）

给一段长 `s = 2N` 帧（N≈10，即约 20 帧）的窗口 `X`、`Z`，反复用上一步预测值喂下一步：

```
X̃₀, Z̃₀ ← X₀, Z₀                          # 初始用真值
for i ← 1 to s:
    δx̃, δz̃ = S([X̃ᵢ₋₁, Z̃ᵢ₋₁]^T; θS)      # 预测增量
    X̃ᵢ = X̃ᵢ₋₁ + δx̃
    Z̃ᵢ = Z̃ᵢ₋₁ + δz̃
```

| 损失 | 含义 |
|---|---|
| `L_xval` / `L_zval` | x / z 的值逼近（预测序列 vs 真值） |
| `L_xvel` / `L_zvel` | x / z 的速度损失（差分） |

权重：各损失大致等权。

## 运行时行为

每帧执行一次，输出 delta 加到当前 `(x, z)`；配合 [[Decompressor]] 解出姿态。每 N 帧由 [[Projector]] 重新投影一次（类似 MM 的搜索周期）。

## 设计要点

- **全部动力学集中在 S**：[[Decompressor]] 与 [[Projector]] 都按 per-pose 工作，不涉时序 → 系统更稳定、易懂、易扩展（如变时间步只需缩放 S 输出并在多种步长上训练；端到端模型改这点要大改重训）
- 自回归训练天然有 teacher-forcing 缺陷的考量，因此窗口设为 `2N` 与搜索周期对齐
- 训练可与 [[Projector]] **并行**（D 必须先训完出 Z）

## 注意事项

- 初始状态可由 [[Decompressor]] 的 Compressor C 生成，但完整 LMM 下由 Projector 投影给出
- Stepper 不直接产出姿态，需 Decompressor 解码
- S 的失真会沿自回归链累积，速度损失 `L_*vel` 是抑制漂移的关键
