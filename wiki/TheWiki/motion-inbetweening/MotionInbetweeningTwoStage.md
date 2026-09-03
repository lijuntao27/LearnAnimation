---
type: concept
title: Motion In-betweening via Two-stage Transformers
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, transformer, animation, siggraph-asia-2022]
summary: Qin 等 2022 论文方法，用 Context+Detail 两级 Transformer Encoder 一次性生成变长补间动画；掩码自注意力+Keyframe/Relative 位置编码实现超长泛化，LAFAN1 上以更少参数超越 RNN/CNN/Transformer 各流派 SOTA，落地 Maya 工具提速约 2 倍。
---

# Motion In-betweening via Two-stage Transformers

> 论文：Qin/Zheng/Zhou, ACM TOG 41(6) Article 184, SIGGRAPH Asia 2022。DOI 10.1145/3550454.3555454。
> 配套仓库 `motion_inbetweening/`。原文全文存 [[RawSource/articles/Motion-In-betweening-2022|RawSource/articles/Motion-In-betweening-2022]]。

## 一句话

先粗后细、两段式、纯 [[Transformer|Transformer Encoder]] + 掩码注意力的补间框架：[[ContextTransformer|Context Transformer]] 负责全局走向、[[DetailTransformer|Detail Transformer]] 负责运动细节，靠 [[InbetweeningPositionalEncodings|Keyframe PE + Learned Relative PE]] 实现超长过渡泛化，输出足底接触供 IK 后处理，最终在 LAFAN1 上以更少参数超越各流派 SOTA，落地 Maya 生产工具。

## 问题定义

**Motion in-betweening（运动补间）**：给定若干**上下文帧**（角色过往姿态）+ 一个**目标帧**（最终姿态），生成中间缺失的**过渡动画**；过渡中间可插入用户指定的姿态约束（全约束或部分关节约束）。

## 动机：已有方法的局限

| 方法 | 局限 |
|---|---|
| 传统样条插值（Slerp） | 只产生平滑过渡、缺乏运动细节；手工调整劳动强度大 |
| [[RecurrentNetworks\|RNN]] 类（ERD-QV, Harvey 2020） | 逐帧自回归、训练推理慢；缺乏全局规划，"预测阶段→逼近目标阶段"交界处突变 |
| CNN 类（FCN） | 感受野限制长程建模；输入尺寸常需 2 的幂，变长需 padding，浪费或不稳 |
| 已有 Transformer（MC-Trans / Δ-Interp） | 参数多；对超训练长度的过渡泛化差（固定位置编码表所致） |

## 两阶段框架

```
输入: 上下文帧 + 目标帧 + (可选)过渡内姿态约束
        ↓
Stage I:  [[ContextTransformer|Context Transformer]]  → 生成"粗略"过渡（把握全局走向）
        ↓
Stage II: [[DetailTransformer|Detail Transformer]]   → 精修细节（消除穿模/脚滑等瑕疵）
```

**经验动机**：单网络容易把过渡"抹平"；加第二级能恢复运动细节、修复 foot sliding。

**两个关键决策**：
1. **一次输出整段过渡**（非自回归）+ 全局 self-attention → 全局规划，避免逐帧突变
2. 训练时随机采样过渡长度 + 特殊位置编码 → 泛化到训练未见过的更长过渡

## 数据表示

- 骨架：类人 22 关节（LAFAN1）。每帧姿态 = 所有关节**局部旋转** `r_t ∈ R^{J×6}` + **根节点世界位置** `p_t ∈ R^3`。局部位置固定（保证骨骼长度一致）
- 旋转用 **6D 旋转表示**（Zhou 2019：取旋转矩阵前两列，Gram-Schmidt 还原）。相比四元数**唯一且连续**，利于训练
- 维度 `D = J×6 + 3 = 135`。最后 **z-score 归一化**：`x_t = (x̃_t − μ_x)/σ_x`，统计量用滑动窗口 50 帧、offset 20 帧算

## 网络结构（概要）

两个 [[Transformer|Transformer Encoder]]，详细见 [[ContextTransformer]] 与 [[DetailTransformer]]。共同结构：

```
姿态 → encoder MLP → +Keyframe PE → 6 层(掩码多头自注意力 + FFN) → 最终 LayerNorm → decoder MLP → 输出
```

每层用带相对位置编码的掩码多头自注意力（`RelMultiHeadedAttention`）+ 位置式 FFN，pre-norm。超参 `d_model=512, n_layer=6, n_head=8, d_head=64`。附录 D 超参扫描：`d=1024`/`L>6` 在 30 帧内与 `d=512/L=6` 相当，但 45 帧显著劣化，故选此。

详见 [[SelfAttention]]、[[InbetweeningPositionalEncodings]]、[[Transformer]]。

## 两种位置编码（核心创新）

见 [[InbetweeningPositionalEncodings]]。要点：

- **Keyframe PE**：MLP 把每帧"距上下文帧"和"距目标帧"的距离映射成编码。因输入是距离值而非固定索引，不限过渡长度
- **Learned Relative PE**：MLP 把任意两帧相对距离映射成表，经 skew 注入注意力分数。不限长度

两者用 MLP 生成（非固定查找表）是能泛化到超长过渡的根本。消融（Table 2）：两者齐全时 45 帧 L2P 从正弦绝对的 3.39 → 1.94，是 Transformer 超越 RNN 的关键。

## 损失函数

**Context**（重全局真实感）：`L_rot`（局部旋转 L1）+ `L_pos`（全局位置 L1，经 FK，隐式加权关节朝向）+ `L_smooth`（相邻帧全局位置差 L1，保证两端平滑）
`L_ctx = α_pos L_pos + α_rot L_rot + α_smooth L_smooth`，权重 `0.01/1.0/0.005`

**Detail**（精修，追加）：`+ L_c`（足部接触重建 L1）+ `L_foot`（脚滑损失 `‖ĉ_t·v̂_t‖₁`，**对 `ĉ` 做 `.detach()` 切断梯度**，否则网络把接触学成恒 0 来作弊）
`L_det = β_pos L_pos + β_rot L_rot + β_c L_c + β_foot L_foot`，权重 `0.02/1.0/0.05/0.05`

`L_pos` 大幅改善 L2P（3.86→1.05）；`L_smooth` 改善 NPSS；`L_c+L_foot` 联合缓解脚滑（0.93→0.89）。

## 训练

- 窗口 `T = t_ctx + t_max_trans + 2`：多出 2 帧，1 帧留给最大长度目标帧，1 帧用于后处理算目标帧曲率（等价提供目标帧速度）
- 预处理：最后上下文帧移到原点（仅 XZ，Y 不变）并让根节点朝 X+，使所有过渡起点一致
- 每迭代随机采样过渡长度（5~30）与受约束帧（候选 = 10% 过渡长度，每帧 50% 概率启用）
- mini-batch 32；[[TrainingMechanics|Adam]]（β₁=0.9,β₂=0.999,ε=1e-8）+ Noam（warmup 8000）
- **两阶段顺序训练**：先训 Context 冻结，再用其预测作输入训 Detail。迭代数 Context 181k、Detail 1430k
- 最多 200 epoch，早停。**不用 dropout**（加了反而出现高频噪声般不连续；随机过渡长度采样已足够防过拟合）。随机姿态约束采样对 Detail 有效

## 后处理 ψ

Context 输出进 Detail 前，非过渡帧回填真值，可能在两端引入小间隙。定义逐通道偏移 `δ*` 加到整段过渡，使最后一个上下文帧与目标帧处**运动曲线曲率最小**。`κ` 用相邻线段夹角近似。**只在 Context 推理时应用**。代码 `train/utils.anim_post_process`。

## 实验（LAFAN1 benchmark）

**数据集**：LAFAN1（Ubisoft 开源动捕，~496,672 帧、30Hz、5 演员、BVH 格式，刻意剔除随机短动作故适合补间）。训练 Subject 1-4（窗口 42、offset 1 → 404,195 clip），测试 Subject 5（窗口 65、offset 40 → 2,232 clip）。统一 `t_ctx=10`、训练最大过渡 30 帧，评估 5/15/30/45。

**指标**：L2P（全局位置 L2）、L2Q（全局四元数旋转 L2）、NPSS（频域归一化功率谱相似度，衡量整体运动真实性）。

**定量对比**（详见 [[Compare-Inbetweening-Methods]]）：

| 方法 | 参数量 | L2P@30 | L2P@45 | NPSS@30 |
|---|---|---|---|---|
| ERD-QV（RNN） | — | 1.28 | 2.24 | 0.1328 |
| Δ-Interp（Enc-Dec） | 40.4M | 1.00 | 3.23 | 0.1217 |
| Ours（T_ctx only, 后处理） | 10.4M | 1.04 | 1.94 | 0.1210 |
| **Ours（完整两阶段）** | **10.4+10.1M** | **0.89** | **1.68** | **0.1124** |

正常长度平均提升 16%、超长（45 帧）提升 55%；参数不到 Δ-Interp 一半。

**四足泛化**（Zhang 2018 数据集，训练 60 帧过渡）：T_det 在 30/60/90 帧 L2P 0.35/0.63/1.27，验证方法不限于人形。

**速度**（1080Ti）：单 clip 30 帧推理 12.1ms vs ERD-QV 33.1 vs Δ-Interp 32.5（约 2.7×），且时间几乎不随过渡长度增长（并行处理）。

**Maya 工具与用户研究**：集成到 Autodesk Maya，支持全约束/部分约束。3 位专业动画师 3 个任务平均提速 86%~112%（约 2 倍）。约束功能还可作"管理关键姿态"的手段。

## 局限

1. 输入上下文模糊或与训练分布差异大时不理想（可加姿态约束解决）
2. 即使有 Detail Transformer 脚滑仍可能残留，需接触信息 + IK 后处理
3. 过渡超过训练最大长度 **1.5 倍**后质量快速下降
4. 姿态约束作用于关节**局部**位置/旋转，但动画师更想约束**末端效应器**（手、脚）——需模型处理 IK，留作未来工作

## 配套代码库

`motion_inbetweening/`。关键：configs（`lafan1_context_model.json` / `lafan1_detail_model.json`）、`packages/motion_inbetween/`（model/train/data/visualization）、scripts（train_/eval_ 各阶段 + baseline）。论文概念 ↔ 代码位置、复现命令、代码与论文的 7 处差异（部分约束未完整实现、注意力缩放用 `1/√d_model` 而非 `1/√d_K`、`delta` 算了未用、`Discriminator` 残留等）详见 [[RawSource/articles/Motion-In-betweening-2022|RawSource 原文 §12]]。

## 相关页面

- [[ContextTransformer]] / [[DetailTransformer]] —— 两个网络细节
- [[InbetweeningPositionalEncodings]] —— 两种位置编码
- [[Compare-Inbetweening-Methods]] —— 与 ERD-QV/MC-Trans/Δ-Interp 横向对比
- [[Summary-MotionInbetweeningTwoStage]] —— 论文级摘要与导览
- [[Transformer]] / [[SelfAttention]] / [[Backpropagation]] / [[TrainingMechanics]] —— 底层概念（deep-learning 领域）
