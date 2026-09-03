---
type: summary
title: Motion In-betweening via Two-stage Transformers 论文摘要与导览
source: RawSource/articles/Motion-In-betweening-2022.md
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, summary, paper-summary]
summary: Motion In-betweening 论文（Qin 2022）的录入摘要与导览，串起 motion-inbetweening 领域全部知识页，原始全文存 RawSource/articles/。
---

# Motion In-betweening via Two-stage Transformers 论文摘要与导览

> 本文是 [[MotionInbetweeningTwoStage|论文主概念页]]的派生摘要，串起 motion-inbetweening 领域全部知识页。原始全文存 [[RawSource/articles/Motion-In-betweening-2022|RawSource/articles/Motion-In-betweening-2022]]。

## 论文信息

- **标题**：Motion In-betweening via Two-stage Transformers
- **作者**：Jia Qin（浙大 + 网易伏羲 AI 实验室）、Youyi Zheng、Kun Zhou
- **发表**：ACM TOG 41(6) Article 184, SIGGRAPH Asia 2022
- **代码**：github.com/victorqin/motion_inbetweening

## 一句话

用 Context + Detail 两级 [[Transformer|Transformer Encoder]] + 掩码自注意力，一次性生成变长补间动画；靠 [[InbetweeningPositionalEncodings|Keyframe PE + Learned Relative PE]] 实现超长过渡泛化；[[Compare-Inbetweening-Methods|LAFAN1 上]]以约一半参数超越 RNN/CNN/Transformer 各流派 SOTA，落地 Maya 工具提速约 2 倍。

## 论文要点速览

| 维度 | 内容 |
|---|---|
| **问题** | 给定上下文帧 + 目标帧，生成中间过渡动画（支持全/部分姿态约束） |
| **框架** | 两阶段：[[ContextTransformer|Context Transformer]]（粗）→ [[DetailTransformer|Detail Transformer]]（精） |
| **数据表示** | 22 关节 6D 旋转 + 根位置 = 135 维，z-score 归一化 |
| **核心创新** | [[InbetweeningPositionalEncodings\|两种 MLP 生成的位置编码]]（不限长度 → 泛化超长） |
| **损失** | L_rot / L_pos / L_smooth / L_c / L_foot（脚滑损失对接触 detach） |
| **训练** | 两阶段顺序训练，[[TrainingMechanics\|Adam+Noam]]，不用 dropout |
| **实验** | [[Compare-Inbetweening-Methods\|LAFAN1]]：正常长度+16%、超长+55%，推理 12ms |
| **落地** | Maya 工具，动画师提速约 2 倍 |

## 知识页导览

| 页面 | 角色 |
|---|---|
| [[MotionInbetweeningTwoStage]] | 论文总览（问题、框架、损失、训练、实验、局限） |
| [[ContextTransformer]] | 阶段一：粗略过渡网络细节 |
| [[DetailTransformer]] | 阶段二：精修 + 足部接触预测 |
| [[InbetweeningPositionalEncodings]] | 两种位置编码（泛化超长的核心） |
| [[Compare-Inbetweening-Methods]] | 与 ERD-QV/MC-Trans/Δ-Interp 横向对比 + LAFAN1/指标说明 |
| [[RawSource/articles/Motion-In-betweening-2022]] | 原文全文（含代码结构与差异） |

## 底层概念支撑（deep-learning 领域）

理解本论文需要以下 [[TheWiki/deep-learning/_index|deep-learning 基础概念]]：

- [[NeuralNetworkBasics]] —— 神经元、[[NeuralNetworkBasics|MLP]]（论文各处的砖块）、PReLU
- [[RecurrentNetworks]] —— RNN（ERD-QV 的本体）、Encoder-Decoder 范式（Δ-Interp 用）、术语陷阱
- [[Transformer]] —— 核心架构、完整前向数据流公式
- [[SelfAttention]] —— Q/K/V 手算全程、掩码做补间、W_Q 维度
- [[InbetweeningPositionalEncodings]] 与 [[Transformer]] —— 位置编码为何必需
- [[Backpropagation]] —— 从 Loss 传回 W_Q、雅可比
- [[TrainingMechanics]] —— LayerNorm、Adam vs SGD、Noam warmup

## 阅读路径建议

1. [[MotionInbetweeningTwoStage]] —— 先读论文总览
2. [[ContextTransformer]] → [[DetailTransformer]] —— 两个网络细节
3. [[InbetweeningPositionalEncodings]] —— 核心创新（泛化超长）
4. [[Compare-Inbetweening-Methods]] —— 实验结果与对比
5. 若需底层概念补充 → 跳 [[TheWiki/deep-learning/_index|deep-learning 领域]]
6. 若需代码细节 → [[RawSource/articles/Motion-In-betweening-2022|RawSource 原文 §12]]
