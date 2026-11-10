---
type: raw-article
title: Neural Machine Translation by Jointly Learning to Align and Translate (2014)
created: 2026-09-04
updated: 2026-09-04
tags: [deep-learning, rnn, attention, encoder-decoder, neural-machine-translation, iclr-2015]
summary: Bahdanau 等 2014 ICLR 论文，给 RNN seq2seq decoder 加"可回看 encoder 各位置"的加性注意力，解固定瓶颈向量问题；是 RNN 到 Transformer 注意力机制的演化桥梁。
---

# Neural Machine Translation by Jointly Learning to Align and Translate (2014)

> 原文 PDF：`Bahdanau-Attention-2014.pdf`（本目录）；提取文本：`Bahdanau-Attention-2014.txt`。
> arXiv: 1409.0473 · 参考索引卡，可后续展开为完整通读笔记。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Neural Machine Translation by Jointly Learning to Align and Translate |
| 作者 | Dzmitry Bahdanau, Kyunghyun Cho, Yoshua Bengio |
| 发表 | ICLR 2015（arXiv 2014-09） |
| arXiv | 1409.0473 |
| 机构 | Jacobs University Bremen / Université de Montréal |

## 核心贡献

在 RNN Encoder-Decoder 之上引入 **加性注意力（additive attention）**：decoder 生成每个目标词时，不再只看 encoder 压出的一个固定向量，而是动态地"回看" encoder 各隐状态并按相关性加权取用。这解除了早期 seq2seq 把整句压成定长向量的 **bottleneck**，在长句翻译上显著改善。

## 历史定位（RNN → Transformer 的桥）

- [[RecurrentNetworks]] §Encoder-Decoder 的 bottleneck 问题正是本文要解的
- 本文的"decoder 回看 encoder"注意力，演化成 [[SelfAttention]] 的"每位置看所有位置"——后者把注意力从"单向 decoder→encoder"推广到"任意位置两两相关"
- 本文仍是 RNN-based（attention 套在 RNN 外），到 [[Attention-Is-All-You-Need-2017]] 才彻底弃循环

## 与本 Wiki 的对应

| 本文概念 | 本 Wiki 页面 |
|---|---|
| 固定瓶颈向量 → 注意力解瓶颈 | [[RecurrentNetworks]] §瓶颈问题 |
| 加性注意力打分 $a_{ij} = v^T \tanh(W_s s_{i-1} + W_h h_j)$ | [[SelfAttention]]（对比点积打分 $Q K^T$） |
| 双向 RNN encoder | [[RecurrentNetworks]] |

## 一句话定位

注意力的"出生证"——把"动态回看"引入 seq2seq，是从 RNN 走向 Transformer 的关键一跃。
