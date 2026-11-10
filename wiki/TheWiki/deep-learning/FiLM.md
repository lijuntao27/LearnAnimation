---
type: concept
title: FiLM（特征级线性调制）
created: 2026-09-14
updated: 2026-09-17
tags: [deep-learning, film, conditioning, modulation, feature-map, hypernetwork]
summary: FiLM=逐通道仿射调制 输出=γ×特征图+β，γ/β 由条件输入经生成器现算（是激活不是参数）；每通道仅 2 参数、与分辨率无关，且证明不依赖归一化。是 HyperBones/PhySkin 身份适配的出处。
---

# FiLM（Feature-wise Linear Modulation，特征级线性调制）

> 出处：Perez, Strub, de Vries, Dumoulin, Courville. *FiLM: Visual Reasoning with a General Conditioning Layer*. AAAI 2018（arXiv:1709.07871）。
> 原文索引卡：[[FiLM-Perez-2018]] · 中文全文译本：[[FiLM-Perez-2018-中文翻译]]（公式与四张结果表的原文出处）。

## 一句话

用一个**条件输入**现算出一组逐通道的增益与偏置 $(\gamma,\beta)$，去调制网络中间层的激活：

$$\mathrm{FiLM}(F_{i,c} \mid \gamma_{i,c}, \beta_{i,c}) = \gamma_{i,c}\,F_{i,c} + \beta_{i,c}$$

关键不在这个式子（它与 [[TrainingMechanics|LayerNorm]] 的仿射**形式完全同构**），而在于：**它不往数据流里塞条件信息，而是去改网络的状态。** 同一个网络，遇到不同条件就表现出不同行为。

## 记号：三个东西必须分清

原文没有给生成器权重起符号。本页沿用 [[TheSchema/math-notation]] §3.1「$\Theta$ 为全部参数集合」的约定，用**下标划分子集**，记生成器权重为 $\theta_g$：

$$(\underbrace{\gamma_i,\beta_i}_{\text{激活}})=\underbrace{g}_{\text{结构}}\big(\underbrace{x_i}_{\text{条件输入}};\ \underbrace{\theta_g}_{\text{参数}}\big)$$

| 符号 | 是什么 | 谁定的 | 何时变 |
|---|---|---|---|
| $g$ | 生成器的**函数形式**（FiLM 论文里是 GRU + 逐块仿射投影） | **人**（架构选择） | 永不 |
| $\theta_g$ | 生成器的**权重**：词嵌入 + GRU 门矩阵 + 每个目标层一套仿射投影 | **反向传播** | 每个 batch |
| $(\gamma_i,\beta_i)$ | 前向**算出的输出** | 由 $\theta_g$ 与 $x_i$ 决定 | **每个样本重算** |

> ⚠️ **最容易搞混的一条**：LayerNorm 的 $\gamma_{\mathrm{bn}}/\beta_{\mathrm{bn}}$ 是**学出来的参数**，FiLM 的 $\gamma/\beta$ 是**前向算出的激活**。同一个位置、同一个形式，身份相反。
>
> 这条区分有实际后果：正因为 $\gamma/\beta$ 是激活而非参数，它们才可以被**缓存**——[[HyperBones2026]] / [[PhySkin2026]] 的「逐 identity 预计算、逐帧零成本」全靠这一点。

原文对这条流水线的叫法是 **FiLM-generating linguistic pipeline**（语言流水线）与 **FiLM-ed visual pipeline**（视觉流水线）。$\theta_g$ 就是前者的全部权重。

## 在 CNN 上到底发生了什么

先对齐形状。CNN 一层的输出不是一个向量，而是一叠二维响应图：

```
Conv 层输出:  F ∈ R^{128 × 14 × 14}
                    ↑     ↑    ↑
                  通道   高    宽     = 128 张 14×14 的特征图
```

**通道维索引「检测的是什么」，$H,W$ 两维索引「在哪里检测到」**（详见 [[ConvolutionalNetworks]] 的局部感受野/共享权重/多核三思想）。

FiLM 的下标是 $\gamma_{i,c}$、$\beta_{i,c}$——**带 $i$（哪个样本）、带 $c$（哪个通道），唯独不带 $h,w$**。翻译过来就是：

> 一个 $\gamma_c$，同时乘在这张特征图**全部 196 个格子**上。

这就是原文 "per-feature-map"（逐特征图）的准确含义，也解释了为什么参数量是**每通道 2 个**、与图像分辨率完全无关。

## 机制一：把 ReLU 一起算进去，才是真正的门控

单独看 $\gamma\odot F+\beta$ 只是「放大/缩小/取反/平移」。**真正的开关语义来自紧跟在后的 ReLU**。合成两步：

$$\mathrm{ReLU}(\gamma z+\beta)=|\gamma|\cdot\mathrm{ReLU}\!\left(\mathrm{sign}(\gamma)\,z+\frac{\beta}{|\gamma|}\right)$$

三个自由度拆得干干净净：

| 分量 | 数学作用 | 对 CNN 的含义 |
|---|---|---|
| $\mathrm{sign}(\gamma)$ | 决定 $z$ 前的符号 | $\gamma>0$：**有 $z$ 就激活**；$\gamma<0$：变成**没有 $z$ 才激活**——同一个卷积核被翻转成它的补检测器 |
| $\beta/\|\gamma\|$ | 决定 ReLU **阈值位置** | 移动该通道的工作点，控制它多容易被点亮 |
| $\|\gamma\|$ | 输出幅度乘子 | 放大/压制该通道在下游的发言权 |

$\gamma_c=0$ 是特例：$F'_c\equiv\beta_c$，与图像内容**完全无关**，该通道被踢出竞争。

这解释了论文 §4.2 的直方图：$\gamma$ 在 0 处有**尖峰**（整片关闭通道）、另有高幅 $\gamma$（点名少数通道）、$\gamma$ 有 36% 为负（翻转极性）、$\beta$ 有 76% 为负（改阈值）。**若没有 ReLU，负 $\gamma$ 只是普通负缩放，这些现象都不成立。**

## 机制二：空间选择性是「间接」涌现的

$\gamma,\beta$ 不含 $(h,w)$，所以 FiLM 在空间上是**逐点**的——它**不搬运任何空间信息**。但分类器末端是全局最大池化：

$$\mathrm{pooled}_c=\max_{h,w}\ \mathrm{ReLU}\big(\gamma_c F_{c,h,w}+\beta_c\big)$$

- $\gamma_c\to0$：$\mathrm{pooled}_c\to\mathrm{ReLU}(\beta_c)$，一个**与图像无关的常数**——该通道即使某处响应强，也被压平、退出池化竞争；
- $\gamma_c$ 放大：该通道的响应在下游 MLP 输入里占据主导。

于是**通道层面的选择，经 max-pooling 变成了空间层面的选择**：被点亮的通道在哪里响应最强，池化向量就由哪个位置主导。这就是 FiLM 能比 Stacked Attention 高 21% 的机理——它拿走了空间注意力的好处，却从不真的做位置混合。

可证伪的推论（论文已验证）：切断 $1\times1$ 卷积之外的空间通路（去掉坐标特征图），模型仍有 95.3%——因为空间信息**本来就不是靠搬运**获得的。

> 边界：正因如此，FiLM 单独扛不住**精确定位**类需求。CLEVR 上够用是因为定位交给了下游 max-pool；需要显式空间分辨时得另配空间注意力。

## 机制三：多层为什么不会坍缩成一个仿射

若两层 FiLM 之间**没有**非线性：

$$F''=\gamma_2(\gamma_1F+\beta_1)+\beta_2=(\gamma_2\gamma_1)\,F+(\gamma_2\beta_1+\beta_2)$$

**还是一个仿射**。所以「重复条件化」的意义完全依赖 $\gamma,\beta$ 之间存在 ReLU（或卷积），使整体成为**分段线性**，每层才能在不同区域实现不同选择。

论文表 2 的插入点结果与此自洽：

| FiLM 插入点 | CLEVR 验证准确率 |
|---|---|
| FiLM after ResBlock ReLU-2 | **97.7** |
| FiLM after ResBlock Conv-2 | 97.1 |
| FiLM after residual connection | 96.6 |
| FiLM before ResBlock Conv-1 | 95.0 |

放在 ReLU **之后**（作用在非负、稀疏的信号上）最好，放在卷积**之前**最差。差距不大，与「FiLM 对位置、对归一化都不敏感」的结论一致。

## 训练：就是端到端反向传播

生成器与 CNN 是**一个**计算图、**一条**损失（答案分类）、**一次** `backward()`；$\theta_g$ 与 CNN 权重在同一个 `optimizer.step()` 里更新。**不是**「先训 CNN 再训生成器」。

梯度链条：

$$\frac{\partial\mathcal{L}}{\partial\gamma_{i,c}}=\sum_{h,w}\frac{\partial\mathcal{L}}{\partial y_{i,c,h,w}}\cdot F_{i,c,h,w}\quad\longrightarrow\quad\frac{\partial\mathcal{L}}{\partial\theta_g}$$

三个读数：

1. **$\partial\mathcal{L}/\partial\gamma_c$ 是「上游梯度 × 特征图」在整张图上的和**——即*这个问题需要的证据，在图上累积有多少*。这是一个**空间聚合后的匹配滤波信号**，从梯度层面解释了为什么 FiLM 参数能按「推理函数」聚类（t-SNE 结果）而非按像素位置。
2. **去掉 FiLM，$\partial\mathcal{L}/\partial\theta_g\equiv 0$。** 生成器之所以可训，全靠 FiLM 凿开的这条可微通路。**FiLM 本质上是一根梯度的管子。**
3. **$\gamma$ 同时乘在前向值与后向梯度上**（$\partial\mathcal{L}/\partial F=\gamma\cdot\partial\mathcal{L}/\partial y$），所以 $\gamma_c=0$ 会把该通道的前向激活**和梯度一起切断**。

> **$\gamma=1+\Delta\gamma$ 是重参数化，不是加容量**（原文附录式 3）。
> $\gamma=1+\Delta\gamma$ 与 $\gamma=\Delta\gamma$ **表达能力完全相同**。区别纯粹在优化：若直接输出 $\gamma$ 且初始以 0 为中心，则开局 $\gamma\approx0$ → 激活清零 → 梯度也清零，通路一上来就是堵的。那个 `+1` 是**硬编码的结构常量、不是参数**，作用是让初始状态落在恒等映射附近。

论文训练配置（说明其"普通"）：Adam 3e-4、权重衰减 1e-5、批大小 64、最多 80 epoch 早停、只用「图像-问题-答案」三元组（无程序标签、无数据增广）、1×TITAN Xp 训练 4 天。

**两套参数确实可分**：CLEVR-Humans 实验中，作者**冻结视觉流水线、只微调语言流水线**——CNN 一字不改，只重训"拧旋钮的人"。

## 与归一化的关系（本文最反直觉的结论）

Conditional Normalization 一系（AdaIN / CIN / Dynamic LayerNorm / CBN）默认：仿射变换**必须紧跟归一化**。原文明确反驳：

> *"we find it not strictly necessary for the affine transformation to occur directly after normalization"*

把 FiLM 挪到 ResBlock 各处（甚至归一化之后的 ReLU 后）性能不显著下降。论文因此**把名称与归一化解绑**，为在归一化不常用的场景（RNN、强化学习）中的应用开了门。

工程配套细节：论文**把紧邻 FiLM 之前的 BN 的仿射参数关掉**——避免同一个仿射做两遍。

## 与其它条件化方法的对照

| 方法 | 公式 | 在 FiLM 框架中的位置 |
|---|---|---|
| 拼接条件特征图 | $W_h h+W_x x$ | 等价 $\gamma=1$（只剩偏置），参数效率更低 |
| WaveNet / Conditional PixelCNN | $h+b(x)$ | 等价 $\gamma=1$ |
| LSTM 门控 / Squeeze-and-Excitation | $\sigma(Wx)\odot h$ | $\gamma\in(0,1)$、$\beta=0$、且 $x=h$（自门控） |
| BN 的 $\gamma/\beta$ | $\gamma\odot\hat h+\beta$ | 同形，但 $\gamma,\beta$ 与输入无关 |
| AdaIN / CBN | 同上，$\gamma,\beta$ 由条件算出 | FiLM 的特例（绑定在归一化后） |
| 超网络 hypernetwork | 一网络生成另一网络的 $W$ | FiLM 是其 $O(C)$ 参数的紧凑特例（且生成的是"伪参数"，见上文辨析） |
| 注意力 | $\mathrm{softmax}(QK^T)V$ | 跨位置/跨通道混合，FiLM **完全不做** |
| Mixture of Experts | 按样本激活子网络 | 前者在**子网络**级选择，FiLM 在**特征图**级选择 |

$\gamma$、$\beta$ 都可自由取值这一点，是 FiLM 与全部"门控类"方法的**唯一**区别：论文证明把 $\gamma$ 限制到 $(0,1)$（sigmoid）、$(-1,1)$（tanh）、$(0,\infty)$（exp）**全部掉分**——**"能静音、能反相"在扛分，不是"能缩放"。**

## 关键实验证据

| 证据 | 说明 |
|---|---|
| CLEVR **97.7%** | 首个超过人类（92.6%）的结果；不做程序监督、不做增广，把 SOTA 误差从 4.5% 减半到 2.3% |
| $\gamma$ 直方图 0 处尖峰 | 学会整片**关闭**通道 + 高幅上调少数通道 → 选择性 |
| t-SNE 参数聚类 | 浅层按**低级推理函数**（equal color / query color）聚、深层按**高级函数**（integer comparison）聚——无架构先验，纯端到端学出"函数级模块性" |
| **$\gamma$ 比 $\beta$ 重要** | 测试期把 $\gamma$ 换成训练集均值掉 **65.4%**，$\beta$ 换均值只掉 **1.0%** |
| 单层 FiLM 就够 | 只留一层 FiLM 几乎不掉分 → 它传的是"模式"（设置），不是"数据"（注入） |
| 零样本参数类比 | 在 FiLM 参数空间做线性组合（"cyan cubes" ≈ "cyan spheres" + "brown cubes" − "brown spheres"），CoGenT B 零样本总准确率 **+3.2%** |

> ⚠️ 两组消融测的不是同一件事：**训练期**消融（$\gamma:=1$ 掉 1.5%、$\beta:=0$ 掉 0.5%）测**必要性**——网络可从零适应"没有 $\gamma$ 的条件化"；**测试期**消融（$\gamma$ 换均值掉 65.4%）测**依赖度**——已学会依赖逐样本 $\gamma$ 的网络被突然抽掉信号会崩。数值相差百倍并不矛盾。

## 在本 Wiki 中的体现

FiLM 是 2026 年 Meta 两篇骨骼驱动布料工作**身份/形状适配**方案的直接出处（原文参考文献 `[PSDV+18]` 即本文）：

- [[HyperBones2026]] —— 形状码 $z$ 经 Shape Modulator 出每层 $(\gamma_\ell(z),\beta_\ell(z))$，**逐 identity 预计算、逐帧零成本**；原文明确"选 FiLM 而非拼接/超网络，因为一个紧致码能调所有层、参数最省"（即上表"$\gamma=1$ 等价拼接"那一条的工程翻版）。
- [[PhySkin2026]] —— **双层调制**：Hyper Modulator（Transformer 点编码身体）出形状码 → FiLM 调姿态网络 `MLP_θ` 的激活层；Pose Modulator 再调 Node Deformer。**identity 不进数据流**，每帧零增量成本。该页指出"网络只学修正"是比 FiLM 摊销更底层的速度根基——即 FiLM 之外还叠了一层 LBS 闭式。
- [[Compare-BoneDrivenCloth-Lineage]] —— 把"**调制把身份计算挪到逐 identity 一次性**"列为 2026 两篇共同的工程关键，与 Pan 2022 的 RBF 插值（运行时算）形成对照。
- [[SkirtQNN-Design]] —— 本项目移动端方案已把 [[PhySkin2026]] 的双层调制列为"架构最贴近的参照"。

## 相关页

- [[ConvolutionalNetworks]] —— 特征图 / 通道 / 卷积核的来源（理解"逐特征图"的前提）
- [[TrainingMechanics]] —— LayerNorm 的 $\gamma/\beta$：同形式、但属参数，是最佳对照物
- [[NeuralNetworkBasics]] —— 激活、特征表示、MLP 基础
- [[SelfAttention]] —— 反面对照：注意力做跨位置混合，FiLM 完全不做
- [[Backpropagation]] —— 链式法则/局部性；本页"梯度链条"一节的通用机制
- [[FiLM-Perez-2018]] / [[FiLM-Perez-2018-中文翻译]] —— 原文与全文译本
