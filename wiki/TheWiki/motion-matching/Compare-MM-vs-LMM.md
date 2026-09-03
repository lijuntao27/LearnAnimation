---
type: comparison
title: MM vs DMM vs LMM vs PFNN vs MANN
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, learned-motion-matching, pfnn, mann, benchmark]
summary: 以论文 Table 3 数值对比五种方法的内存、运行时与训练时间，Locomotion 5.3MB vs 52.1MB、Bear 约 140× 压缩；LMM 比 MM 慢但远快于 PFNN/MANN。
---

# MM vs DMM vs LMM vs PFNN vs MANN

> 数据来源：论文 §6.2 Table 3（单线程 Intel Xeon 3.5GHz 12 核 CPU，自研优化 NN 推理库）。

## 方法一览

| 方法 | 用到的网络/库 | 说明 |
|---|---|---|
| **MM** | 原始 X + Y | 基线 [[MotionMatching]]，无训练 |
| **DMM** | X + Z + [[Decompressor]] | 仅解压网络，存 Z 代替 Y |
| **LMM** | [[Decompressor]] + [[Stepper]] + [[Projector]] | [[LearnedMotionMatching]] 完整，脱离数据库 |
| **PFNN** | Phase-Functioned NN（Holden 2017） | 端到端生成模型，GPU 训练 |
| **MANN** | Mode-Adaptive NN（Zhang 2018） | 四足端到端，GPU 训练 |

## 内存对比（MB，Locomotion 场景）

| 方法 | X | Y | Z | D | S | P | **总** |
|---|---|---|---|---|---|---|---|
| MM | 9.4 | 42.7 | – | – | – | – | **52.1** |
| DMM | 9.4 | – | 11.1 | 0.9 | – | – | 21.4 |
| LMM | – | – | – | 1.2 | 3.2 | 0.9 | **5.3** |
| PFNN | – | – | – | – | – | – | 9.3 |

**Bear 场景（最能体现 LMM 价值）**：MM **995.6 MB** → LMM **7.1 MB**，约 **140× 压缩**。

> 注：Y 中速度信息改用有限差分实时算、不变动的关节位移只存一次以省内存；网络权重还可压成 16-bit int 再降 2×（无明显精度/性能损失）。

## 运行时性能（µs/帧，Locomotion）

| 方法 | D | S | P | **Frame** |
|---|---|---|---|---|
| MM | – | – | – | **90** |
| DMM | 85 | – | 90 | 94 |
| LMM | 85 | 100 | 127 | **197** |
| PFNN | – | – | – | **1370** |
| MANN | – | – | – | **2440** |

- LMM 比 MM 慢（多了三网络推理），但**远快于 PFNN/MANN**
- P 每 N=10 帧才跑一次，均摊后对帧时间影响小
- blending 约 20µs/帧；MM 还要算搜索（含 AABB 加速）

## 训练时间（小时）

| | D | S | P | **总** |
|---|---|---|---|---|
| Locomotion | 8.7 | 6.1 | 3.3 | **14.7** |
| Bear | 20.4 | 6.5 | 8.9 | **29.3** |

- S 与 P **并行**训练，总时间取较长者（表内 Total 已是并行后）
- D 必须先训，再 S/P 并行
- PFNN/MANN 需 GPU；LMM 全 CPU
- MM 无训练（预处理好即用）→ 迭代期可用 MM，定稿后切 LMM 当后处理

## 取舍小结

| 维度 | 最优 | 说明 |
|---|---|---|
| 内存 | **LMM** ≫ DMM > MM | 数据量越大优势越明显（Bear 140×） |
| 运行时速度 | MM > DMM > LMM ≫ PFNN > MANN | LMM 仍达产线预算 |
| 质量 | MM ≈ LMM > PFNN/MANN | 用户研究：难辨 MM 与 LMM |
| 训练成本 | MM(0) < LMM < PFNN/MANN | LMM 几小时~过夜，CPU 即可 |
| 灵活性/可控 | MM ≈ LMM > 端到端 | LMM 保留 MM 行为 |

## 实操建议（论文 §6.2）

- 数据库小（如 Chair）或求帧时间 → 单用 [[Decompressor]]（DMM）做折中
- 大数据量 + 内存敏感 → 完整 LMM
- 迭代期 → 用 MM 实时改，满意后切 LMM 作 post-process
