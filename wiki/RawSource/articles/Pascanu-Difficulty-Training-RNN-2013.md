---
type: raw-article
title: On the difficulty of training Recurrent Neural Networks (2013)
created: 2026-09-04
updated: 2026-09-04
tags: [deep-learning, rnn, gradient-vanishing, bptt, icml-2013]
summary: Pascanu/Mikolov/Bengio 2013 ICML 论文，用雅可比特征值分析 RNN 沿时间反向传播的梯度消失/爆炸，给出裁剪与正则对策；是本 Wiki [[RecurrentNetworks]] 长程遗忘数学根因的出处。
---

# On the difficulty of training Recurrent Neural Networks (2013)

> 原文 PDF：`Pascanu-Difficulty-Training-RNN-2013.pdf`（本目录）；提取文本：`Pascanu-Difficulty-Training-RNN-2013.txt`。
> arXiv: 1211.5063 · 参考索引卡，可后续展开为完整通读笔记。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | On the difficulty of training Recurrent Neural Networks |
| 作者 | Razvan Pascanu, Tomas Mikolov, Yoshua Bengio |
| 发表 | ICML 2013 |
| arXiv | 1211.5063 |
| 机构 | Université de Montréal |

## 核心贡献

用 **雅可比矩阵特征值** 严格分析 RNN 沿时间反向传播（BPTT）的梯度行为：循环权重 $W_{hh}$ 的最大特征值 $>1$ 致梯度爆炸、$<1$ 致梯度消失，且二者在长序列上呈指数级放大。给出两项工程对策——**梯度裁剪（gradient clipping）** 处理爆炸、**正则项** 鼓励雅可比特征值靠近 1 缓解消失。

## 与本 Wiki 的对应

本 Wiki [[RecurrentNetworks]] §BPTT 与梯度消失一节的数学根因即出自本文：

| 本文概念 | 本 Wiki 页面 |
|---|---|
| $\partial L/\partial \theta \propto \prod W_{hh}$，特征值连乘 | [[RecurrentNetworks]] §BPTT 与梯度消失 |
| 特征值 $<1$ → 消失，$>1$ → 爆炸 | [[RecurrentNetworks]] §BPTT 与梯度消失 |
| 梯度裁剪 | [[TrainingMechanics]]（若涉及） |
| 固定大小隐状态 = 有损压缩 | [[RecurrentNetworks]] §BPTT 与梯度消失 |

## 历史定位

- 从数学上坐实了 RNN "记不住久远信息"的根因，为 LSTM 门机制的合理性提供理论支撑
- 也是 [[Transformer]] 用自注意力（路径长度恒为 1）彻底绕开此问题的动机之一

## 一句话定位

RNN 长程遗忘的"诊断书"——把"梯度消失/爆炸"从经验现象变成雅可比特征值的数学结论。
