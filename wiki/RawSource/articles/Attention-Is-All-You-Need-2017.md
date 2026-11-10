---
type: raw-article
title: Attention Is All You Need (2017)
created: 2026-09-04
updated: 2026-09-04
tags: [deep-learning, transformer, attention, neural-machine-translation, neurips-2017]
summary: Vaswani 等 2017 NeurIPS 原论文，提出纯自注意力 Transformer，弃循环卷积、任意两位置路径为 1、可并行；本 Wiki [[Transformer]]/[[SelfAttention]] 的结构超参与数据流均直接出自此文。
---

# Attention Is All You Need (2017)

> 原文 PDF：`Attention-Is-All-You-Need-2017.pdf`（15 页，本目录）；提取文本：`Attention-Is-All-You-Need-2017.txt`。
> arXiv: 1706.03762 · 本文为参考索引卡，可后续展开为完整通读笔记（参考 `Motion-In-betweening-2022.md` 的粒度）。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Attention Is All You Need |
| 作者 | Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N. Gomez, Łukasz Kaiser, Illia Polosukhin |
| 发表 | NeurIPS 2017（31st Conference on Neural Information Processing Systems） |
| arXiv | 1706.03762 |
| 机构 | Google Brain / Google Research / University of Toronto |

## 核心贡献

提出 **Transformer**——首个完全基于自注意力、不依赖循环或卷积的序列转换架构。通过多头自注意力让任意两位置间信息路径长度为 1，实现并行训练与长程依赖建模；在 WMT 2014 英德/英法翻译上达到当时 SOTA，同时训练显著快于基于 RNN/CNN 的方法。

## 与本 Wiki 的对应

本 Wiki 中 Transformer 的结构、超参与数据流均出自本文：

| 本文概念 | 本 Wiki 页面 |
|---|---|
| Scaled Dot-Product Attention、Q/K/V | [[SelfAttention]] |
| 多头注意力（H=8, d_head=64） | [[Transformer]] §多头 |
| 位置式前馈网络 FFN | [[Transformer]] §FFN |
| 正弦绝对位置编码 | [[Transformer]] §位置编码（补间论文改用自定义 PE，见 [[InbetweeningPositionalEncodings]]） |
| 残差 + LayerNorm（原版 post-norm） | [[Transformer]] §一层结构（补间论文改 pre-norm） |
| Encoder-Decoder 变体 | [[Transformer]] §三种用法 |

## 阅读路径建议

- §2 背景：自注意力 vs 循环/卷积的动机（对应 [[RecurrentNetworks]] 的 RNN 病根讨论）
- §3 Scaled Dot-Product Attention + Multi-Head Attention 公式（对应 [[SelfAttention]] 手算全过程）
- §4 训练：Noam warmup、Adam β₂=0.98、dropout（对应 [[TrainingMechanics]]）
- §5 定量结果与可视化注意力
- 配合 Harvard NLP《The Annotated Transformer》逐行代码阅读效果最佳

## 一句话定位

Transformer 的"宪法"——本 Wiki 整个 deep-learning 子库关于注意力与并行序列建模的一切结论，追根溯源都到这一篇。
