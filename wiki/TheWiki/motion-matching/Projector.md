---
type: entity
title: Projector（投影网络 P）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, learned-motion-matching, neural-network, nearest-neighbor]
summary: Learned Motion Matching 三网络之一，用抗噪训练模拟最近邻搜索，从用户查询 x̂ 直接映射到 (x, z)，消除对 Matching Database X 的依赖；兼有分布映射与防递归漂移作用。
---

# Projector（P）

> [[LearnedMotionMatching]] 三阶段的「投影」阶段替代网络。对应论文 §4.3、Algorithm 3。

## 用途与定位

[[MotionMatching]] 的 **Projection** 阶段：最近邻搜索 `k* = argmin ‖x̂ - x_k‖²` 在 X 上做。要消除对 X 的内存依赖，Projector 学会模拟该搜索：输入用户查询 `x̂`，直接输出最近邻对应的 `(x_{k*}, z_{k*})`。

> Projector 是三网络中**最大、最耗内存、对最终质量影响最大**的一个（论文 §8 Limitations）。每 N（≈10）帧才跑一次，均摊后对帧时间影响小。

## 网络架构

| 层数 | 隐藏单元 | 激活 |
|---|---|---|
| 6 | 512 | ReLU |

## 训练（Algorithm 3，抗噪是关键）

```
nσ ~ U(0, 1)                              # 随机噪声幅度
n  ~ N(0, 1)                               # 高斯噪声向量
x̂ = x + nσ · n                             # 造查询点
k* = Nearest(x̂, X)                         # 真做最近邻
x̃, z̃ = P(x̂; θP)                           # 训 P 输出最近邻对应值
```

采样不同 `nσ` 使 P 对各种量级扰动鲁棒。

| 损失 | 含义 |
|---|---|
| `L_xval = ‖x_{k*} - x̃‖₁` | x 值逼近 |
| `L_zval = ‖z_{k*} - z̃‖₁` | z 值逼近 |
| `L_dist = | ‖x̂-x_{k*}‖₂² - ‖x̂-x̃‖₂² |` | 距离保持——保证 P 不只死记最近邻、还维持距离几何 |

权重：各损失大致等权。

## 运行时行为

每 N 帧把用户查询 `x̂` 喂给 P → 得 `(x, z)` → 交给 [[Stepper]] 逐帧前进、[[Decompressor]] 解码姿态。

## Projector 的双重作用（论文 §7.1）

1. **分布映射**：把用户控制映射到训练数据分布。用户给出未训练过的无效请求时，P 会投影到最近的合法状态
2. **定期 reset**：每 N 帧重投影，防止自回归递归「die out」、卡小循环、漂离数据流形（传统 RNN 通病，[[LearnedMotionMatching#泛化策略]] 由此受益）

## 主动不泛化（§7.2）

因 P 模拟最近邻，**不插值生成训练集外的新动画**。代价：需要更彻底的数据覆盖（靠数据增强 + 特征权重控制弥补）。收益：系统状态始终贴近训练态，避免外推出坏行为；小网络规模也防过拟合，对输入小扰动仍鲁棒。

## 注意事项

- 加深 P 的层数能提精度但运行时间超预算 → 论文未来工作建议用 **memory layers**（Lample 2019）破此权衡
- 当前训练为两阶段（先 D 再 S/P 并行），论文未来想改为端到端
- P 的额外平滑感（用户研究反馈）可能源于其精度上限
