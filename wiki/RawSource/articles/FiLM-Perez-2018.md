---
type: raw-article
title: FiLM - Visual Reasoning with a General Conditioning Layer (2018)
created: 2026-09-14
updated: 2026-09-14
tags: [deep-learning, conditioning, film, modulation, visual-reasoning, clevr, aaai-2018]
summary: Perez 等 2018 AAAI 论文，提出通用条件化层 FiLM：用条件输入生成的逐特征仿射参数 γ/β 调制网络中间激活；在 CLEVR 上首超人类，并证明调制不依赖归一化。
---

# FiLM: Visual Reasoning with a General Conditioning Layer (2018)

> 原文 PDF：`FiLM-Perez-2018.pdf`（本目录）；提取文本：`FiLM-Perez-2018.txt`。
> 中文译本：[[FiLM-Perez-2018-中文翻译]]。
> 本 Wiki 概念页：[[FiLM]]（机制推导、与归一化的关系、在布料动画中的落地）。
> arXiv: 1709.07871 · AAAI 2018 · 代码：https://github.com/ethanjperez/film

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | FiLM: Visual Reasoning with a General Conditioning Layer |
| 作者 | Ethan Perez, Florian Strub, Harm de Vries, Vincent Dumoulin, Aaron Courville |
| 发表 | AAAI 2018（第 32 届 AAAI 人工智能大会） |
| arXiv | 1709.07871（v2, 2017-12-18） |
| 机构 | MILA / Université de Montréal、Rice University、CIFAR、Univ. Lille (Inria CRIStAL) |
| 关键词 | Deep Learning, Language and Vision |

## 核心贡献

提出 **FiLM（Feature-wise Linear Modulation，特征级线性调制）**——一种通用的神经网络条件化层。核心是一个逐特征仿射变换：

$$\mathrm{FiLM}(F_{i,c} \mid \gamma_{i,c}, \beta_{i,c}) = \gamma_{i,c} F_{i,c} + \beta_{i,c}$$

其中 $\gamma_{i,c} = f_c(x_i)$、$\beta_{i,c} = h_c(x_i)$ 由**条件输入** $x_i$ 现算（学习中称为 FiLM generator）。在视觉推理场景里，一个 GRU 读取问题句生成 $(\gamma,\beta)$，去调制处理图像的 CNN 激活——让语言**改写 CNN 的行为**，而非把语言拼接进数据流。

在 CLEVR 上达到 97.7%（overall），首个超过人类（92.6%）的方法；不做程序监督、不做数据增广，把 SOTA 误差从 4.5% 减半到 2.3%。

## 主要结论

1. **效果**：CLEVR 97.7%、CLEVR-Humans 微调后 75.9%（超 PG+EE 9.3%）、CLEVR-CoGenT 上的组合泛化优于显式建模组合性的 PG+EE。
2. **机制**：$\gamma$ 直方图在 0 处有尖峰 → FiLM 学会用问题**整片关闭**特征图，同时高幅上调少数选择性特征图；t-SNE 显示浅层参数按**低级推理函数**聚类（equal color / query color），深层按**高级推理函数**聚类（integer comparison）——没有架构先验，纯端到端学出了"函数级模块性"。
3. **$\gamma$ 比 $\beta$ 重要**：测试期把 $\gamma$ 换成训练集均值掉 65.4%，把 $\beta$ 换成均值只掉 1.0%；限制 $\gamma$ 取值域（sigmoid / tanh / exp）均掉分，说明**大倍数缩放与取反/清零能力**是成功关键。
4. **与归一化解耦**（本文最反直觉的一点）：把 FiLM 挪到 ResBlock 不同位置，甚至放在归一化之后的 ReLU 后面，性能不显著下降。此前 Conditional Normalization 一系默认仿射变换必须紧跟归一化，本文证明**这个假设不必要**，因此方法可推广到归一化不常用的场景（RNN、强化学习）。
5. **零样本泛化**：仿词嵌入类比运算（King − Man + Woman = Queen），在 FiLM 参数空间做线性组合——"How many cyan cubes?" ≈ "How many cyan spheres?" + "How many brown cubes?" − "How many brown spheres?"，纠正了训练集偏置导致的错误答案，CoGenT B 上零样本总准确率 +3.2%。

## 与其它条件化方法的关系

| 方法 | 形式 | 与 FiLM 的关系 |
|---|---|---|
| Conditional Normalization（AdaIN / CIN / Dynamic LayerNorm / CBN） | 替换归一化层的仿射参数 | FiLM 是它的**推广**；FiLM 证明不必绑定归一化 |
| 拼接条件特征图 / 拼接进全连接层 | 逐特征条件**偏置** | 等价于 $\gamma = 1$ 的 FiLM（参数效率更低） |
| WaveNet / Conditional PixelCNN | 直接加逐特征条件偏置 | 等价于 $\gamma = 1$ 的 FiLM |
| LSTM 门控 / Squeeze-and-Excitation | 同一输入自身门控，缩放限 $(0,1)$ | FiLM 是"缩放+平移，且都不受限"的一般化 |
| 超网络 hypernetwork | 用一网络生成另一网络的参数 | FiLM 是其一种紧凑特例 |
| Mixture of Experts / 条件计算 | 按样本激活子网络 | 前者在**子网络**级选择，FiLM 在**特征图**级选择 |

## 历史定位

FiLM 是把"条件化"从具体任务技巧抽象成通用层的那篇，此后成为多模态与生成模型里的标准组件。在本 Wiki 的语境中，它是 2026 年布料动画两篇 Meta 工作身份适配方案的直接出处：

- [[HyperBones2026]] —— 形状码 $z$ 经 Shape Modulator 出每层 $(\gamma_\ell(z), \beta_\ell(z))$，逐 identity 预计算、逐帧零成本；原文明确"选 FiLM 而非拼接/超网络，因为一个紧致码能调所有层、参数最省"，参考文献即本文 `[PSDV+18]`。
- [[PhySkin2026]] —— "双层调制"：Hyper Modulator 出身体形状码 → FiLM 调姿态网络激活层；Pose Modulator 再调 Node Deformer。identity 不进数据流、每帧零增量成本。
- [[Compare-BoneDrivenCloth-Lineage]] —— 把"调制把身份计算挪到逐 identity 一次性"列为 2026 两篇共同的工程关键。

## 与本 Wiki 的对应

| 本文概念 | 本 Wiki 页面 |
|---|---|
| 全文机制推导、记号辨析、梯度链条 | [[FiLM]] |
| 逐特征仿射调制 $\gamma \odot h + \beta$ | [[TrainingMechanics]] §LayerNorm（同形式，但那里的 $\gamma/\beta$ 是与输入无关的学习参数） |
| 条件信息不进数据流、只改写激活分布 | [[NeuralNetworkBasics]]（激活/权重共享视角） |
| 自注意力是"混合通道"的对比项 | [[SelfAttention]] |
| FiLM 在骨骼驱动布料中的落地 | [[HyperBones2026]]、[[PhySkin2026]] |

## 一句话定位

FiLM 的"出生证"——把 $\gamma \odot h + \beta$ 这一个仿射式子确立为通用条件化层，并顺手拆掉了"必须依附归一化"这颗螺栓。
