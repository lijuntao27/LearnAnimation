---
type: raw-article
title: Sequence to Sequence Learning with Neural Networks (2014)
created: 2026-09-04
updated: 2026-09-04
tags: [deep-learning, rnn, encoder-decoder, seq2seq, neurips-2014]
summary: Sutskever/Vinyals/Le 2014 NeurIPS 论文，建立 RNN Encoder-Decoder seq2seq 范式：encoder 把输入压成定长向量、decoder 自回归生成；是后续 bottleneck 与注意力研究的对照基线。
---

# Sequence to Sequence Learning with Neural Networks (2014)

> 原文 PDF：`Seq2Seq-Sutskever-2014.pdf`（本目录）；提取文本：`Seq2Seq-Sutskever-2014.txt`。
> arXiv: 1409.3215 · 参考索引卡，可后续展开为完整通读笔记。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Sequence to Sequence Learning with Neural Networks |
| 作者 | Ilya Sutskever, Oriol Vinyals, Quoc V. Le |
| 发表 | NeurIPS 2014 |
| arXiv | 1409.3215 |
| 机构 | Google |

## 核心贡献

确立 **Encoder-Decoder seq2seq** 范式：一个 LSTM 把输入序列读完后压成**一个固定长度向量**，另一个 LSTM 据此向量自回归生成输出序列。在 WMT 14 英法翻译上用深层 LSTM + 输入反转达到接近 SOTA。配套提出端到端直接训练、对长句用翻转输入的小技巧。

## 历史定位

- 这是 [[RecurrentNetworks]] §Encoder-Decoder 范式与 §自回归生成的原型实现
- 其"固定长度瓶颈向量"的缺陷，直接催生 [[Bahdanau-Attention-2014]] 的注意力机制
- 与后续 [[Attention-Is-All-You-Need-2017]] 对照：seq2seq 用 RNN 串行、Transformer 用自注意力并行

## 与本 Wiki 的对应

| 本文概念 | 本 Wiki 页面 |
|---|---|
| Encoder 压成固定向量 → bottleneck | [[RecurrentNetworks]] §瓶颈问题 |
| Decoder 自回归逐 token 生成 | [[RecurrentNetworks]] §自回归生成 |
| 输入输出长度可不同 | [[RecurrentNetworks]] §Encoder-Decoder |
| 与 Transformer 对照 | [[Transformer]] §三种用法 |

## 一句话定位

seq2seq 的"出生证"——把 RNN 编成 encoder-decoder 对，也把"固定瓶颈向量"的病根留给后来者去治。
