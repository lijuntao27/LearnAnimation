---
type: concept
title: Learned Motion Matching（学习方法）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, learned-motion-matching, neural-network]
summary: Holden 等 2020 论文方法，把 Motion Matching 拆为投影/步进/解压三阶段各用一个专门小神经网络替代，消除动画库内存依赖，兼得 MM 质量可控与 NN 可扩展性。
---

# Learned Motion Matching（LMM）

> 论文核心方法。源文：[[RawSource/articles/Learned-Motion-Matching-2020|论文原文]]。

## 核心思想

把 [[MotionMatching]] 算法**拆成三个可分离步骤**，每步用一个专门小神经网络替代，从而既保留 MM 的质量/可控/快迭代，又获得神经网络的可扩展性（内存不随数据线性增长）。

## 三阶段 → 三网络

| MM 阶段 | 作用 | 替代网络 | 消除的库 |
|---|---|---|---|
| Projection（投影） | 最近邻搜索 `x̂ → x_{k*}` | [[Projector]] | X |
| Stepping（步进） | 索引前进 `i → i+1` | [[Stepper]] | X、Z 查表 |
| Decompression（解压） | 查 `y_i` | [[Decompressor]]（+Compressor C） | Y |

引入隐变量 `z ∈ R^32`：x 不足以完全推断 y，用 autoencoder 思路由 Compressor 编码 `y → z`，Decompressor 用 `(x, z)` 重构 `y`。

## 运行时流程（论文 Fig.3）

```
用户输入 x̂ ──[Projector P]──> (x, z)
                              │
              每 N帧重投影     │ 每帧
                              ▼
                         [Stepper S] ──> (x_{i+1}, z_{i+1})
                              │
                              ▼
                       [Decompressor D] ──> 姿态 y
                              │
                        inertialization blend 去不连续
```

## 训练顺序（§4.4）

1. **Decompressor 必须先训**（才能算出 Z 喂给后两者）
2. Stepper、Projector 可**并行**训练（总时间取较长者）

配置：PyTorch + RAdam，batch 32，lr 0.001 每 1000 迭代 ×0.99 decay；Stepper 窗口 `s=2N`（N≈10）；最多 500,000 迭代；**单线程 Intel Xeon 3.5GHz 12 核 CPU**（小网络 CPU 训练反比 GPU 高效）；几小时出结果、过夜最优。

## 三种可用形态

| 形态 | 用到的网络 | 适用 |
|---|---|---|
| **MM** | 无（原始数据库） | 基线，迭代期使用 |
| **DMM** | D（存 Z 代替 Y） | 数据量不大、求速度时的折中 |
| **LMM** | D + S + P | 完全脱离数据库，最大省内存 |

数值对比见 [[Compare-MM-vs-LMM]]。

## 为什么用多网络而非端到端（§7.1）

1. 可独立训练/调试/toggle，逐个验证行为
2. Decompressor 单独用即是良好折中
3. 姿态（D、P 按 per-pose）与动力学（S）解耦 → 更稳定、易懂、易扩展（变时间步只缩放 S 输出并多种步长训练；端到端模型改这点要大改重训）
4. Projector 双重作用：分布映射（处理未训练的无效请求）+ 定期 reset 防递归漂移

## 泛化策略（§7.2，主动不泛化）

[[Projector]] 模拟最近邻 → **不插值生成训练集外新动画**。代价：需更彻底数据覆盖（靠可验证的数据增强 + 特征权重控制弥补）。收益：系统状态始终贴近训练态，避免外推坏行为；小网络防过拟合，对输入小扰动仍鲁棒。

## 数据增强案例：Bear（§5）

用 **carpet unrolling**（Miller 2015）把 ~3581 帧关键帧扩展到 ~700,000 帧。这种大体量增强库对 basic MM 极不友好（内存爆炸），但 LMM 轻松容纳——这正是 LMM 的核心价值。

## 局限与未来（§8）

- **[[Projector]] 精度是整体质量瓶颈**，最大最耗内存，加深层提精度但运行时间超预算（用户研究中的"额外平滑感"或源于此）
- 未来：用 **memory layers**（Lample 2019）提精度不牺牲速度；探索端到端训练

## 用户研究结论（§6.4）

34 人在 AAA 制作中实测第三人称 locomotion，两阶段测试（A/B 对比 + 10 个随机系统分类）。受试者难分辨 MM 与 LMM，连动画总监都分不清；32/34 认为可互相替代。LMM 被认为"transitions 更平滑"，MM 更"dynamic/snappier"。
