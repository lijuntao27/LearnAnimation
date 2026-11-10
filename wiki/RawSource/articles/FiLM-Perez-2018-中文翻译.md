---
type: translation
title: FiLM - Visual Reasoning with a General Conditioning Layer (2018) 中文译本
source: RawSource/articles/FiLM-Perez-2018.pdf
created: 2026-09-14
updated: 2026-09-14
tags: [deep-learning, conditioning, film, modulation, visual-reasoning, clevr, aaai-2018, translation]
summary: Perez 等 2018 AAAI 原论文的中文学习译本：提出通用条件化层 FiLM，用条件输入生成的逐特征仿射参数 γ/β 调制网络中间激活；CLEVR 达 97.7% 首超人类，并证明调制不依赖归一化。原始英文见 [[FiLM-Perez-2018]]。
---

# FiLM: Visual Reasoning with a General Conditioning Layer — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），四张结果表与图表的数值用 Markdown 表格渲染，正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[FiLM-Perez-2018|原索引卡]] 所指的 `FiLM-Perez-2018.txt`。
>
> 论文为 arXiv 开放获取（1709.07871），版权归 AAAI 2018 所有，本译本按学术学习使用处理。
>
> **抽取工具说明**：本目录的 `.txt` 用 PyMuPDF 抽取，而非既有文章惯用的 `pdftotext`。原因是 `pdftotext` 会把本文的核心符号 $\gamma$ / $\beta$ 整个丢弃（全文 0 个 $\gamma$，公式退化成"output i,c and i,c"），信息损失不可接受。PyMuPDF 保住了 $\gamma$（63 处）与 $\beta$（40 处），代价是公式会被拆到单独行上。
>
> 本 Wiki 概念页：[[FiLM]]（机制推导、记号辨析、梯度链条、在布料动画中的落地）。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | FiLM: Visual Reasoning with a General Conditioning Layer |
| 作者 | Ethan Perez, Florian Strub, Harm de Vries, Vincent Dumoulin, Aaron Courville |
| 发表 | AAAI 2018（第 32 届 AAAI 人工智能大会） |
| arXiv | 1709.07871v2 [cs.CV]，2017-12-18 |
| 机构 | MILA / Université de Montréal；Rice University；CIFAR Fellow；Univ. Lille, CNRS, Centrale Lille, Inria, UMR 9189 CRIStAL France |
| 关键词 | Deep Learning, Language and Vision |
| 代码 | https://github.com/ethanjperez/film |

## 摘要

我们提出一种通用的神经网络条件化方法，称为 **FiLM：Feature-wise Linear Modulation（特征级线性调制）**。FiLM 层基于条件信息，通过一个简单的、逐特征的仿射变换来影响神经网络的计算。我们证明 FiLM 层对**视觉推理**极为有效——即回答与图像相关、需要多步高层过程的问题，而这类任务对不显式建模推理的标准深度学习方法一直很困难。具体而言，我们在视觉推理任务上表明 FiLM 层：1) 将 CLEVR 基准的最优误差**减半**；2) 以**连贯一致**的方式调制特征；3) 对消融与架构修改具有**鲁棒性**；4) 能从少量样本甚至**零样本**泛化到有挑战性的新数据。

> **图 1**：CLEVR 样例与 FiLM 模型的答案。
> (a) 问：既是小的紫色物体、又是黄色橡胶物体的圆柱体有多少个？答：2
> (b) 问：与那个大的棕色哑光物体形状相同的另一个物体是什么颜色？答：棕色

## 1 引言

对日常视觉输入进行推理的能力，是人类智能的一块基石。有人主张，要让智能体学会这一复杂、有结构的过程，必须在架构中内置推理的某些方面，例如组合性（compositionality，Hu et al. 2017; Johnson et al. 2017b）或关系计算（relational computation，Santoro et al. 2017）。然而，如果一个由**通用组件**构成的模型能够学会视觉推理，这样的架构很可能会在更广泛的领域中得到应用。

为弄清是否存在这样的通用架构，我们利用新近提出的 CLEVR 数据集（Johnson et al. 2017a），它通过问答来测试视觉推理，样例见图 1。视觉问答（就图像提问这一通用任务）有自己的数据集谱系（Malinowski and Fritz 2014; Geman et al. 2015; Antol et al. 2015），它们通常关注在图像上提出一组多样的、更简单的问题，往往一眼即可作答。从这些数据集中，涌现出若干有效的通用深度学习模型（Malinowski, Rohrbach, and Fritz 2015; Yang et al. 2016; Lu et al. 2016; Anderson et al. 2017）。然而，在 CLEVR 上的测试表明，这些通用深度学习方法难以学会**结构化、多步的推理**（Johnson et al. 2017a）。特别地，这些方法倾向于利用数据中的偏置，而不是捕捉推理背后的复杂底层结构（Goyal et al. 2017）。

在本工作中，我们表明一种通用的模型架构能够取得强视觉推理能力，所用方法即我们提出的 FiLM：Feature-wise Linear Modulation。一个 FiLM 层对一个神经网络的**中间特征**执行简单的、逐特征的仿射变换，其条件是一个**任意输入**。在视觉推理场景中，FiLM 层使一个作用于输入问题的循环神经网络（RNN）能够影响一个作用于图像的卷积神经网络（CNN）的计算。该过程自适应地、剧烈地改变 CNN 的行为，使其成为输入**问题的函数**，从而让整个模型能够执行从计数到比较等多种推理任务。FiLM 可以看作 **Conditional Normalization（条件归一化）** 的推广——后者已在图像风格化（Dumoulin, Shlens, and Kudlur 2017; Ghiasi et al. 2017; Huang and Belongie 2017）、语音识别（Kim, Song, and Bengio 2017）和视觉问答（de Vries et al. 2017）中被证明非常成功，这也展示了 FiLM 的广泛适用性。

本文扩展自一份更短的报告（Perez et al. 2017）。我们的关键贡献是：通过在视觉推理任务上展示以下几点，说明 FiLM 是一种强力的条件化方法：

1. FiLM 模型在多种视觉推理任务上达到 state-of-the-art，且往往优势明显。
2. FiLM 以**连贯**的方式运作。它学到一种复杂的底层结构，并有选择性地操纵被条件化网络的特征。它还使 CNN 能够正确地定位问题所指的对象。
3. FiLM 是**鲁棒**的；许多变体化的 FiLM 模型仍然优于此前的最优方法。特别地，我们发现归一化与"条件化仿射变换之所以成功"之间并没有紧密联系——这是一个此前未被触碰的假设。因此，我们放宽了该方法可被应用的条件。
4. FiLM 模型能从少量数据泛化到比训练时更复杂和/或差异显著的数据。我们还提出了一种新的、基于 FiLM 的**零样本泛化**方法，进一步提升并验证了 FiLM 的泛化能力。

## 2 方法

我们的模型用 FiLM 处理"问题-图像"输入，如图 2 所示。我们先解释 FiLM，再描述我们用于视觉推理的具体模型。

### 2.1 Feature-wise Linear Modulation（特征级线性调制）

FiLM 学习基于某个输入、对一个神经网络的中间特征施加仿射变换（即 FiLM），从而自适应地影响该网络的输出。更形式化地，FiLM 学习函数 $f$ 和 $h$，它们以输入 $x_i$ 为自变量输出 $\gamma_{i,c}$ 和 $\beta_{i,c}$：

$$\gamma_{i,c} = f_c(x_i) \qquad \beta_{i,c} = h_c(x_i) \tag{1}$$

其中 $\gamma_{i,c}$ 与 $\beta_{i,c}$ 通过一个逐特征的仿射变换，调制神经网络的激活 $F_{i,c}$（下标分别指第 $i$ 个输入的第 $c$ 个特征或特征图）：

$$\mathrm{FiLM}(F_{i,c} \mid \gamma_{i,c}, \beta_{i,c}) = \gamma_{i,c} F_{i,c} + \beta_{i,c} \tag{2}$$

$f$ 和 $h$ 可以是任意函数，例如神经网络。对目标神经网络处理的调制，既可以基于该网络自身的输入，也可以基于另一个输入——后者正是多模态或条件化任务的情形。对 CNN 而言，$f$ 和 $h$ 因此基于 $x_i$ 调制**逐特征图的激活分布**，而与空间位置无关。

实践中，把 $f$ 和 $h$ 视为"输出一个 $(\gamma, \beta)$ 向量的单一函数"更方便，因为例如在 $f$ 与 $h$ 之间共享参数往往更利于高效学习。我们把这单一函数称为 **FiLM generator（FiLM 生成器）**。我们同样把施加 FiLM 层的网络称为 the FiLM-ed network（被 FiLM 作用的网络，即"特征级线性调制网络"）。

FiLM 层使 FiLM 生成器能够操纵目标网络（被 FiLM 作用网络）的特征图：把它们整体放大或缩小、取反、关闭、选择性阈值化（当后接 ReLU 时），等等。**每个特征图被独立地条件化**，使 FiLM 生成器在每个 FiLM 层对激活拥有中等细粒度的控制。

由于 FiLM 每个被调制的特征图只需两个参数，它是一种可扩展且计算高效的条件化方法。特别地，**FiLM 的计算开销不随图像分辨率增长**。

> **图 2**：一个作用于 CNN 的 FiLM 层。点号表示 Hadamard 积。$\gamma$ 与 $\beta$ 的不同组合能以多种方式调制各个特征图。

### 2.2 模型

我们的 FiLM 模型由一条"**生成 FiLM 的语言流水线**"和一条"**被 FiLM 作用的视觉流水线**"组成，如图 3 所示。FiLM 生成器用一个含 4096 个隐单元的 GRU 网络（Chung et al. 2014）处理问题 $x_i$，其输入是学到的 200 维词嵌入。GRU 的最终隐状态即问题嵌入，模型据此对第 $n$ 个残差块通过仿射投影预测 $(\gamma_{i^n,\cdot}, \beta_{i^n,\cdot})$。

视觉流水线从缩放为 $224 \times 224$ 的图像输入中提取 128 张 $14 \times 14$ 的图像特征图，使用的要么是**从零训练的 CNN**，要么是**固定的预训练特征提取器**加一个可学习的 $3 \times 3$ 卷积层。从零训练的 CNN 由 4 层、每层 128 个 $4 \times 4$ 卷积核、ReLU 激活与批归一化组成，与此前 CLEVR 工作类似（Santoro et al. 2017）。固定特征提取器输出在 ImageNet（Russakovsky et al. 2015）上预训练的 ResNet-101（He et al. 2016）的 conv4 层，以对齐此前 CLEVR 工作（Johnson et al. 2017a; 2017b）。图像特征被若干个（我们的模型是 4 个）**被 FiLM 作用的残差块**（FiLM-ed ResBlock）处理，每块 128 个特征图，最后接一个分类器。分类器由 $1 \times 1$ 卷积到 512 个特征图、全局最大池化、以及一个含 1024 个隐单元的两层 MLP 组成，输出最终答案上的 softmax 分布。

每个被 FiLM 作用的 ResBlock 以一个 $1 \times 1$ 卷积开始，后接一个 $3 \times 3$ 卷积，架构如图 3。我们将**紧邻 FiLM 层之前的批归一化层的参数关掉**。借鉴此前 CLEVR（Hu et al. 2017; Santoro et al. 2017）与视觉推理（Watters et al. 2017）的工作，我们把两张表示相对 $x$ / $y$ 空间位置（缩放到 $-1$ 到 $1$）的坐标特征图，与图像特征、每个 ResBlock 的输入、以及分类器的输入拼接起来，以促进空间推理。

我们用 Adam（Kingma and Ba 2015）（学习率 3e-4）、权重衰减（1e-5）、批大小 64，并在整个被 FiLM 作用的网络中通篇使用批归一化和 ReLU，端到端从零训练。模型只使用训练集中的"图像-问题-答案"三元组，不做数据增广。我们基于验证准确率做早停，最多训练 80 个 epoch。更多模型细节见附录。经验上我们发现 FiLM 容量很大，因此许多架构与超参选择都是为了增加正则化。

我们强调，我们的模型**仅**依靠逐特征仿射条件化，就让问题信息影响视觉流水线的行为以回答问题。这与经典的视觉问答流水线不同——后者通过逐元素积、拼接、注意力，和/或更高级的方法把图像与语言信息融合为单一嵌入（Yang et al. 2016; Lu et al. 2016; Anderson et al. 2017）。

> **图 3**：我们模型中的 FiLM 生成器（左）、被 FiLM 作用的网络（中）与残差块架构（右）。

## 3 相关工作

FiLM 可以视为 **Conditional Normalization（CN，条件归一化）** 方法的推广。CN 把归一化层中典型的逐特征仿射变换的参数（最初由 Ioffe and Szegedy 2015 引入），替换为某个条件信息的学习函数。多种形式的 CN 在若干领域被证明极为有效：Conditional Instance Norm（Dumoulin, Shlens, and Kudlur 2017; Ghiasi et al. 2017）与 Adaptive Instance Norm（Huang and Belongie 2017）用于图像风格化，Dynamic Layer Norm 用于语音识别（Kim, Song, and Bengio 2017），Conditional Batch Norm 用于复杂场景（如 VQA、GuessWhat?!）上的通用视觉问答（de Vries et al. 2017）。本工作与它们互补：我们试图证明逐特征仿射条件化对**多步推理**有效，并理解其成功背后的底层机制。

值得注意的是，此前的 CN 工作**没有考察"仿射变换是否必须紧跟在归一化之后"**——它们把归一化写进方法名，或是出于教学目的，或是出于实现细节。我们研究了 FiLM 与归一化的联系，发现仿射变换**并非必须**紧跟归一化。因此，我们通过 FiLM 为上述所有方法提供了一个统一框架，并给出该方法的**无归一化松弛版本**，从而能被更广泛地应用。

除 CN 外，FiLM 与其它条件化方法之间还有诸多联系。一种常见做法（例如用于 Conditional DCGAN，Radford, Metz, and Chintala 2016）是把条件信息的**常量特征图**与卷积层输入拼接。这种做法虽然参数效率不高，但结果不过是产生一个**逐特征的条件偏置**。同样地，把条件信息与全连接层输入拼接，也等价于一个逐特征的条件偏置。另一些方法如 WaveNet（van den Oord et al. 2016a）与 Conditional PixelCNN（van den Oord et al. 2016b）则直接加上一个逐特征的条件偏置。这些方法都等价于 **$\gamma = 1$ 的 FiLM**，我们在实验部分将 FiLM 与之比较。在强化学习中，FiLM 的一种变体表述被用于训练单个"游戏条件化"的深度 Q 网络来玩十款 Atari 游戏（Kirkpatrick et al. 2017），不过 FiLM 既不是该工作的重点，也未被当作主要组件分析。

还有一些方法是对**输入自身的特征**做门控，而不是基于一个独立的**条件输入**。这类方法包括用于序列建模的 LSTM（Hochreiter and Schmidhuber 1997）、用于机器翻译的 Convolutional Sequence to Sequence（Gehring et al. 2017），甚至包括 ImageNet 2017 的夺冠模型 Squeeze-and-Excitation Networks（Hu, Shen, and Sun 2017）。这种做法相当于一个**逐特征的条件缩放**，且被限制在 0 到 1 之间；而 FiLM 同时包含缩放与平移，且二者都不受限。在实验部分，我们展示了把 FiLM 的缩放限制在 0 到 1 之间的效果。我们认为值得注意的是：这种"特征调制"的一般性思路在多种设定与架构中都有效。

FiLM 与其它方法之间还有更宽泛的关联。例如，FiLM 可以看成"用一个网络生成另一个网络的参数"，因此是**超网络**（hypernetwork，Ha, Dai, and Le 2016）的一种形式。此外，FiLM 与**条件计算**、**专家混合**（mixture of experts）方法有潜在联系——后者按样本激活专门的子网络部件（Jordan and Jacobs 1994; Eigen, Ranzato, and Sutskever 2014; Shazeer et al. 2017）；我们稍后会给出证据，表明 FiLM 学会基于条件信息选择性地凸显或抑制特征图。区别在于：那些方法在**子网络**层面做选择，而 FiLM 在**特征图**层面做选择。

在视觉推理领域，一种领先方法是 **Program Generator + Execution Engine（PG+EE）** 模型（Johnson et al. 2017b）。它由一个序列到序列的 Program Generator 组成：输入一个问题，输出一个对应"可组合神经模块树"的序列，每个模块是一个两三层残差块。这棵神经模块树被组装成 Execution Engine，再从图像预测答案。这种模块化做法属于神经模块网络方法谱系（Andreas et al. 2016a; 2016b; Hu et al. 2017），其中 End-to-End Module Networks（Hu et al. 2017）也已在视觉推理上被测试。这些模型通过显式建模推理的组合性、并用额外程序标签（即"如何正确回答该问题"的逐步真值指令）训练，从而引入了强先验。End-to-End Module Networks 更进一步，通过为特定功能手工设计逐模块的神经网络架构来内置模型偏置。**我们的方法直接从视觉与文本输入学习，不需要额外线索，也不需要专门架构。**

**Relation Networks（RN，关系网络）** 是视觉推理的另一条领先路线（Santoro et al. 2017）。RN 通过显式内置"基于比较"的先验而成功。RN 用一个 MLP 对图像上提取的卷积特征的每个位置做两两比较，并把 LSTM 提取的问题特征作为该 MLP 的输入之一；然后对所得比较向量做逐元素求和，形成另一个向量，由最终分类器据此预测答案。我们注意到，RN 的计算开销随空间分辨率**平方增长**，而 FiLM 的开销与空间分辨率无关。特别地，由于 RN 把问题特征与 MLP 输入拼接——即前文所说的"逐特征条件偏置"的一种形式——它的条件化方式与 FiLM 是相关的。

## 4 实验

首先，我们在 CLEVR 任务上测试模型，并用训练好的 FiLM 模型分析 FiLM 学到了什么。其次，我们用 CLEVR-Humans 任务考察模型对更有挑战性问题的泛化能力。最后，我们用 CLEVR Compositional Generalization Test 考察 FiLM 在少样本与零样本泛化设定下的表现。附录中我们提供了模型的错误分析。代码见 https://github.com/ethanjperez/film 。

### 4.1 CLEVR 任务

CLEVR 是一个合成数据集，含 70 万条（图像、问题、答案、程序）四元组（Johnson et al. 2017a）。图像包含各种形状、材质、颜色和大小的 3D 渲染物体。问题本质上是多步且组合性的，如图 1 所示：从计数问题（"有多少个绿色物体与那个绿色金属块大小相同？"）到比较问题（"小的黄色圆柱体是否比黄色金属立方体少？"），长度可达 40 词以上。答案是从 28 个可能答案中选出的一个词。程序是额外的监督信号，由逐步指令组成（例如 `filter shape[cube]`、`relate[right]`、`count`），说明如何回答该问题。

**基线** 我们对比以下方法（详见相关工作一节）：

- **Q-type baseline**：仅基于问题的类别做预测。
- **LSTM**：仅用问题做预测。
- **CNN+LSTM**：对 CNN 提取的图像特征与 LSTM 提取的问题特征做 MLP 预测。
- **Stacked Attention Networks（CNN+LSTM+SA）**：对 CNN 图像特征与 LSTM 问题特征做线性预测，二者通过两轮软空间注意力结合（Yang et al. 2016）。
- **End-to-End Module Networks（N2NMN）与 Program Generator + Execution Engine（PG+EE）**：由分离的神经网络学习分离的子函数，并组装成依赖问题的结构（Hu et al. 2017; Johnson et al. 2017b）。
- **Relation Networks（CNN+LSTM+RN）**：内置对空间位置的两两比较，以显式建模推理的关系性质（Santoro et al. 2017）。

**结果** 如表 1 所示，FiLM 在 CLEVR 上取得了新的总体 state-of-the-art，超过了人类和此前的方法——包括那些使用显式推理模型、程序监督和/或数据增广的方法。

| 模型 | 总体 | Count | Exist | Compare Numbers | Query Attribute | Compare Attribute |
|---|---|---|---|---|---|---|
| Human (Johnson et al. 2017b) | 92.6 | 86.7 | 96.6 | 86.5 | 95.0 | 96.0 |
| Q-type baseline (Johnson et al. 2017b) | 41.8 | 34.6 | 50.2 | 51.0 | 36.0 | 51.3 |
| LSTM (Johnson et al. 2017b) | 46.8 | 41.7 | 61.1 | 69.8 | 36.8 | 51.8 |
| CNN+LSTM (Johnson et al. 2017b) | 52.3 | 43.7 | 65.2 | 67.1 | 49.3 | 53.0 |
| CNN+LSTM+SA (Santoro et al. 2017) | 76.6 | 64.4 | 82.7 | 77.4 | 82.6 | 75.4 |
| N2NMN* (Hu et al. 2017) | 83.7 | 68.5 | 85.7 | 84.9 | 90.0 | 88.7 |
| PG+EE (9K prog.)* (Johnson et al. 2017b) | 88.6 | 79.7 | 89.7 | 79.1 | 92.6 | 96.0 |
| PG+EE (700K prog.)* (Johnson et al. 2017b) | 96.9 | 92.7 | 97.1 | 98.7 | 98.1 | 98.9 |
| CNN+LSTM+RN†‡ (Santoro et al. 2017) | 95.5 | 90.1 | 97.8 | 93.6 | 97.9 | 97.1 |
| **CNN+GRU+FiLM** | **97.7** | 94.3 | 99.1 | 96.8 | 99.1 | 99.1 |
| **CNN+GRU+FiLM‡** | 97.6 | 94.3 | 99.3 | 93.4 | 99.3 | 99.3 |

> **表 1**：各基线、竞争方法与 FiLM 在 CLEVR 上的准确率（总体与按问题类型）。(*) 表示使用了程序标签形式的额外监督。(†) 表示使用了数据增广。(‡) 表示从原始像素训练。

对不使用额外监督的方法而言，FiLM 大致把 state-of-the-art 的误差**减半**（从 4.5% 降到 2.3%）。注意，使用预训练图像特征作为输入本身可以看作一种数据增广，但 FiLM 在使用原始像素输入时表现同样好。有趣的是，原始像素模型在较低级问题上（即查询与比较属性）似乎更好，而图像特征模型在较高级问题上（即比较物体数量）似乎更好。

### 4.2 FiLM 层学到了什么？

为理解 FiLM 如何进行视觉推理，我们可视化激活以观察 FiLM 层的净效果；同时用直方图与 t-SNE（van der Maaten and Hinton 2008）在学到的 FiLM $\gamma$、$\beta$ 参数本身中寻找规律。附录图 14 与图 15 中，我们可视化了 FiLM 在单个特征图层面的效果。

**激活可视化** 图 4 可视化了"被全局池化的特征"所对应位置的分布，模型最终分类器中的 MLP 正是用这些特征来预测答案。这些图像显示，FiLM 模型使用**答案相关或问题相关物体**附近区域的特征来做预测，这也与 CLEVR 上的高准确率相符。这一发现凸显出：**恰当的特征调制会间接导致空间调制**——因为带有问题相关特征的区域会有大激活，其它区域则不会。这一观察或许解释了为什么 FiLM 能如此显著地（21%）超过 Stacked Attention 这个"下一个最好的、且未为推理显式设计"的方法：FiLM 似乎兼具空间注意力的许多好处，同时还影响了特征表示。

图 4 还表明，被 FiLM 作用的网络**在其流水线全程都在进行推理**。在顶部例子中，被 FiLM 作用的网络在 MLP 分类器之前就已经定位到了答案所指的对象本身；在底部例子中，它为 MLP 分类器保留了**问题所指、但答案未指**的物体特征。后一个例子提供的证据是：最终那个 MLP 自身在承担部分推理，它利用 FiLM 抽取相关特征来支撑自己的推理。

> **图 4**：可视化模型用于全局最大池化特征的位置分布，其最终 MLP 据此预测。
> 问："…紫色物体是什么形状？"答：立方体；"…蓝色物体？"答：球体；"…蓝色物体右侧的红色物体？"答：球体；"…蓝色物体左侧的红色物体？"答：立方体。
> 问："有多少个青色物体…"答：位于灰色立方体右侧 3 个；位于小立方体左侧 2 个；既在灰色立方体右侧又在小立方体左侧 1 个；在灰色立方体右侧或在（P: 3）小立方体左侧 4 个。
> FiLM 能正确定位答案所指对象（上排）或全部问题所指对象（下排），但在答错时定位精度下降（最右下方）。所用问题与图像与 Johnson et al. (2017b) 一致。

**FiLM 参数直方图** 为在更低层面分析 FiLM 如何利用问题来条件化视觉流水线，我们画出在验证集上预测出的 $\gamma$ 与 $\beta$ 值，见图 5（更详细的见附录图 16–18）。$\gamma$ 与 $\beta$ 的取值都利用了相当大的范围，分别从 $-15$ 变到 $19$、从 $-9$ 变到 $16$。$\gamma$ 值在 0 处有一个**尖峰**，说明 FiLM 学会用问题来**关闭或显著抑制整片特征图**。与此同时，FiLM 学会用高幅值的 $\gamma$ 上调一组**选择性得多**的其它特征图。此外，$\gamma$ 值中有很大比例（36%）为负；由于我们的模型在 FiLM 之后使用 ReLU，$\gamma < 0$ 会让**与 $\gamma > 0$ 时相当不同的一组激活**通过 ReLU 传到下游层。另外，$\beta$ 值中有 76% 为负，说明 FiLM 也用 $\beta$ 来选择哪些激活能通过 ReLU。我们稍后会展示 FiLM 的成功在很大程度上与架构无关，但考察某个具体模型仍能揭示 FiLM 学到的施加影响的方式。

> **图 5**：全部 FiLM 层的 $\gamma_{i,c}$（左）与 $\beta_{i,c}$（右）在验证集上的取值直方图。

**FiLM 参数的 t-SNE 图** 在图 6 中，我们用 t-SNE 可视化 3000 个随机验证点的 FiLM 参数向量 $(\gamma, \beta)$。我们分析更深的 6-ResBlock 版本（其验证准确率与 4-ResBlock 模型相近），以便更好地考察层级中不同深度的 FiLM 层如何表现。第一层与最后一层的 FiLM $(\gamma, \beta)$ 分别按**回答 CLEVR 问题所需的低级推理函数**与**高级推理函数**聚类。例如，`equal color` 与 `query color` 的 FiLM 参数在第一层彼此接近、在最后一层则分开；`shape`、`size`、`material` 的问题同理。反过来，`equal shape`、`equal size`、`equal material` 的 FiLM 参数在最后一层聚在一起、在第一层却分开了——其它高级分组（如整数比较、查询）亦然。这些发现表明，**FiLM 层学出了某种"基于函数的模块性"，而且没有架构先验**。仅靠端到端训练，FiLM 就学会不仅以不同方式处理不同**类型的问题**，也以不同方式处理不同**类型的问题子部分**；FiLM 模型按"从低级过程到高级过程"的顺序工作，这是正确的做法。对于 FiLM 层更少的模型，这类模式也会出现，但不够清晰——这些模型不得不更早开始高级推理。

> **图 6**：一个 6 层 FiLM 网络中第一层（左）与最后一层（右）$(\gamma, \beta)$ 的 t-SNE 图。FiLM 参数在第一层按低级推理函数聚类，在最后一层按高级推理函数聚类。

### 4.3 消融

我们在验证集上对最优模型做消融研究，以理解 FiLM 如何学会视觉推理。测试期消融结果见图 7，架构消融见表 2，不同模型深度见表 3。**未做任何超参调优**的情况下，大多数架构消融与模型深度仍超过此前只用"图像-问题-答案"三元组训练的 state-of-the-art，这支持了 FiLM 的整体鲁棒性。表 3 还表明（基于验证集）我们的结果具有统计显著性。

| 模型 | 总体 |
|---|---|
| **Restricted $\gamma$ or $\beta$** | |
| FiLM with $\beta := 0$ | 96.9 |
| FiLM with $\gamma := 1$ | 95.9 |
| FiLM with $\gamma := \sigma(\gamma)$ | 95.9 |
| FiLM with $\gamma := \tanh(\gamma)$ | 96.3 |
| FiLM with $\gamma := \exp(\gamma)$ | 96.3 |
| **Moving FiLM within ResBlock** | |
| FiLM after residual connection | 96.6 |
| FiLM after ResBlock ReLU-2 | 97.7 |
| FiLM after ResBlock Conv-2 | 97.1 |
| FiLM before ResBlock Conv-1 | 95.0 |
| **Removing FiLM from ResBlocks** | |
| No FiLM in ResBlock 4 | 96.8 |
| No FiLM in ResBlock 3-4 | 96.5 |
| No FiLM in ResBlock 2-4 | 97.3 |
| No FiLM in ResBlock 1-4 | 21.4 |
| **Miscellaneous** | |
| $1 \times 1$ conv only, with no coord. maps | 95.3 |
| No residual connection | 94.0 |
| No batch normalization | 93.7 |
| Replace image features with raw pixels | 97.6 |
| **Best Architecture** | 97.4±.4 |

> **表 2**：消融的 CLEVR 验证集准确率。基于最优架构训练，仅改动指定部分。最优模型准确率的标准差取自 5 次运行。

> **图 7**：FiLM 参数对测试期噪声的鲁棒性分析。水平线对应把 $\gamma$ 或 $\beta$ 设为各自训练集均值的水平。

**$\gamma$ 与 $\beta$ 各自的作用** 为分别测试 $\gamma$ 与 $\beta$ 的效果，我们训练了一个 $\gamma = 1$ 恒定的模型和另一个 $\beta = 0$ 的模型。这两个模型分别掉 1.5% 和 0.5% 准确率；说明 FiLM **仅靠偏置或仅靠缩放**都能学会为视觉推理条件化 CNN，只是不如二者同时使用。这个结果也说明 **$\gamma$ 比 $\beta$ 更重要**。

为进一步比较 $\gamma$ 与 $\beta$ 的重要性，我们在最优的、完全训练好的模型上做了一系列**测试期消融**（图 7）。首先，把 $\beta$ 替换为训练集上的 $\beta$ 均值。这个消融实际上在测试期抹掉了 $\beta$ 参数中的全部条件信息（模型训练时是同时使用 $\gamma$ 与 $\beta$ 的）。此时准确率只掉 **1.0%**；而对 $\gamma$ 做同样操作则掉 **65.4%**。这一巨大差异说明，实践中 FiLM 主要通过 $\gamma$ 而非 $\beta$ 来条件化。接着，我们在测试期向最优模型的 FiLM 参数加入越来越多的高斯噪声。$\gamma$ 上的噪声对性能的伤害显著更大，表明 FiLM 对 $\gamma$ 的变化比 $\beta$ 更敏感，也印证了 $\gamma$ 相对更重要。

> [!note] 两组消融测的不是同一件事
> 训练期消融（$\gamma := 1$ 掉 1.5%、$\beta := 0$ 掉 0.5%）测的是**必要性**：网络可以从零适应"没有 $\gamma$ 的条件化"。测试期消融（把 $\gamma$ 换成训练集均值掉 65.4%）测的是**依赖度**：一个已经学会依赖逐样本 $\gamma$ 的网络，突然被抽掉这个信号会崩。两者数值相差百倍并不矛盾——前者有重训的机会，后者没有。论文用后者的巨大落差（65.4% vs 1.0%）论证 $\gamma$ 是主要条件化通道。

**限制 $\gamma$** 为理解 $\gamma$ 的哪个方面最有效，我们训练了一个用 sigmoid 把 $\gamma$ 限制到 $(0,1)$ 的模型——很多使用逐特征乘性门控的模型都是这么做的。同样地，我们也用 $\tanh$ 把 $\gamma$ 限制到 $(-1,1)$。两种限制都损害性能，程度大致相当于用 $\gamma = 1$ 训练来彻底移除 $\gamma$ 上的条件化。因此，**FiLM 以大幅值缩放特征的能力**似乎是其成功的一个来源。用 $\exp$ 把 $\gamma$ 限制到 $(0,\infty)$ 同样损害性能，这验证了 **FiLM 取反与清零特征图的能力**的价值。

**条件归一化** 我们对 FiLM 的**放置位置**做消融，以评估 Conditional Normalization 一系所假设的"归一化与 FiLM 之间的关系"。遗憾的是，仅靠"训练一个不带归一化的对应模型"很难把 FiLM 的效果与归一化的效果准确解耦，因为归一化会显著加速、正则化并改善神经网络学习（Ioffe and Szegedy 2015）——但为完整起见我们仍列出这些结果。不过我们发现，把 FiLM 层挪到模型 ResBlock 的不同位置，性能**没有实质性下降**；把 FiLM 放在 ResBlock 中归一化之后的 ReLU 后面时，甚至达到最优模型性能区间的上端。因此，我们把**名称与归一化解绑**，以便更清楚地说明该方法的根本有效性来自何处。通过证明这种条件化机制与归一化并无紧密联系，我们为在归一化不太常用的场景中的应用打开了大门，例如 RNN 与强化学习——这些都是 FiLM 值得期待的未来方向。

**重复条件化** 为理解"重复条件化"对 FiLM 模型成功的贡献，我们训练了 FiLM 层数逐级减少的模型。FiLM 层更少的模型——哪怕只有**一层** FiLM——与最优模型的性能相差都不大，这表明模型**只调制一次特征**就能成功推理并回答多样的问题。这一观察凸显了哪怕单个 FiLM 层的容量：也许一层 FiLM 就能把足够的问题信息传给 CNN，使其能在网络更靠后的位置执行推理，从而替代更深 FiLM 模型似乎使用的那种更分层的条件化。我们把对这个问题的更深入研究留待未来工作。

**空间推理** 为考察 FiLM 模型如何处理空间推理，我们训练了一版最优架构（基于图像特征），只用 $1 \times 1$ 卷积，并且不向模型输入表示相对空间位置的坐标特征图。由于模型末端有全局最大池化，这个模型**无法跨空间位置传递信息**。值得注意的是，它仍达到 95.3% 的高准确率，说明 FiLM 模型仅凭固定图像特征在**单个位置**所含的空间信息，就能对空间做推理。

**残差连接** 去掉残差连接导致一次较大的准确率下降。由于网络末端有全局最大池化，这一发现说明最优模型学会主要使用那些"在推理的低层与高层中**反复重要**的位置"的特征来做最终决策。FiLM 调制位于残差连接**内部**的模型比调位于残差连接**外部**的模型准确率更高，支持了这一假设。

**模型深度** 表 3 显示按 ResBlock 数量划分的模型性能。FiLM 对深度变化是鲁棒的，但只有 1 个 ResBlock 时较差，这支持了"被 FiLM 作用的网络在其流水线全程都在推理"这一早先的推论。

| 模型 | 总体 | 模型 | 总体 |
|---|---|---|---|
| 1 ResBlock | 93.5 | 6 ResBlocks | 97.7 |
| 2 ResBlocks | 97.1 | 7 ResBlocks | 97.4 |
| 3 ResBlocks | 96.7 | 8 ResBlocks | 97.6 |
| 4 ResBlocks | 97.4±.4 | 12 ResBlocks | 96.9 |
| 5 ResBlocks | 97.4 | | |

> **表 3**：按 FiLM 模型深度划分的 CLEVR 验证集准确率。

### 4.4 CLEVR-Humans：人类提出的问题

为评估视觉推理模型对更真实、更复杂、更自由形式问题的泛化能力，CLEVR-Humans 数据集被提出（Johnson et al. 2017b）。该数据集包含人类就 CLEVR 图像提出的问题及对应答案。样本数有限：训练 1.8 万、验证 7 千、测试 7 千。这些问题由 Amazon Mechanical Turk 上的工人收集，被要求提出"对聪明机器人来说可能很难回答"的问题。因此，CLEVR-Humans 的问题使用更丰富的词汇与更复杂的概念。

**方法** 为在 CLEVR-Humans 上测试 FiLM，我们取最好的 CLEVR 训练好的 FiLM 模型，**仅微调其"生成 FiLM 的语言流水线"**。与此前工作（Johnson et al. 2017b）类似，我们不在 CLEVR-Humans 上更新视觉流水线，以缓解在小训练集上的过拟合。

| 模型 | 训练于 CLEVR | 训练于 CLEVR，在 human 上微调 |
|---|---|---|
| LSTM | 27.5 | 36.5 |
| CNN+LSTM | 37.7 | 43.2 |
| CNN+LSTM+SA+MLP | 50.4 | 57.6 |
| PG+EE (18K prog.) | 54.0 | 66.6 |
| **CNN+GRU+FiLM** | **56.6** | **75.9** |

> **表 4**：CLEVR-Humans 测试准确率：微调前（左）与在 CLEVR-Humans 数据上微调后（右）。

**结果** 如表 4 所示，我们的模型在微调前后都取得 CLEVR-Humans 上的 state-of-the-art 泛化，表明 FiLM 很适合处理更复杂、更多样的问题。图 8 给出了 CLEVR-Humans 的样例与 FiLM 模型的答案。微调之前，FiLM 领先此前方法的幅度较小；微调之后，FiLM 达到了显著改善的最终准确率。特别地，FiLM 微调带来的准确率增益比其它模型高出 50% 以上；FiLM 利用小小的 CLEVR-Humans 数据集实现了**数据高效**的自适应。

值得注意的是，微调后 FiLM 超过此前的最优方法 PG+EE 达 9.3%。PG+EE 的先前工作解释说，这种神经模块网络方法在"无法用其模块库存很好地近似"的问题上表现挣扎（Johnson et al. 2017b）。相比之下，FiLM 有自由以**新颖的方式**调制已有特征图——一种相当灵活且细粒度的操作——来推理新概念。这些结果因此为 FiLM 通用本性的优势提供了一些证据。

> **图 8**：CLEVR-Humans 的样例，其中引入了新词（下划线标出）与新概念。
> 问：草是什么颜色？答：圆柱体。问：哪些形状的物体部分被遮挡？答：球体。问：最靠右的哑光物体是什么颜色？答：棕色。问：在大立方体里反射的是什么形状？答：圆柱体。问：若移除所有立方体形状的物体，剩下最多的是什么形状的物体？答：球体（P: 橡胶）。
> 在 CLEVR-Humans 上微调后，一个在 CLEVR 上训练的模型能推理遮挡、最高级与反射，但仍难以应付假设性情景（最右）。它还学会了人类偏好——主要以形状来标识物体（最左）。

### 4.5 CLEVR 组合泛化测试

为测试模型学到的组合性概念泛化得如何，CLEVR-CoGenT 被提出（Johnson et al. 2017a）。该数据集用与 CLEVR 相同的方式合成，但包含两个条件：在 **Condition A** 中，所有立方体是灰、蓝、棕或黄，所有圆柱体是红、绿、紫或青；在 **Condition B** 中，立方体与圆柱体**互换调色板**。两个条件都包含各种颜色的球体。因此 CLEVR-CoGenT 能指示一个模型**如何**回答 CLEVR 问题：是靠**记忆**性状的组合，还是学到了**解耦的/通用的表示**。

**结果** 我们在 Condition A 上训练最优架构，报告在 Condition A 与 B、以及在 B 上微调前后的准确率，见图 9。结果表明，FiLM 在学习通用概念上超过其它视觉推理模型。FiLM 学到的组合泛化甚至比 PG+EE 更好——后者显式建模了组合性，并用**明确包含过滤颜色与过滤形状**的程序级监督训练。

**样本效率与灾难性遗忘** 我们在图 9 中给出样本效率与遗忘曲线。FiLM 用 **1/3 的微调数据**就达到了此前 state-of-the-art 的准确率。然而，我们的 FiLM 模型在微调后仍然遭受灾难性遗忘。

**零样本泛化** FiLM 在 Condition A 上的准确率远高于 B，说明 FiLM 在一定程度上**记忆了属性组合**。例如，模型学到了"立方体不是青色"这一偏置，因为学这个训练集偏置有助于最小化训练损失。

为克服这一偏置，我们提出一种新的、基于 FiLM 的零样本泛化方法。受词嵌入操作启发（例如 "King" − "Man" + "Woman" = "Queen"，Mikolov et al. 2013），我们检验线性操作是否能延伸到用 FiLM 做推理。我们通过 FiLM **参数空间**中问题的线性组合，计算 "How many cyan cubes are there?" 的 $(\gamma, \beta)$：

$$\text{"How many cyan spheres?"} + \text{"How many brown cubes?"} - \text{"How many brown spheres?"}$$

用这个 $(\gamma, \beta)$，我们的模型能正确计数青色立方体。图 10 给出了该方法的另一个例子。

| 方法 | Train A: A | Train A: B | Fine-tune B: A | Fine-tune B: B |
|---|---|---|---|---|
| CNN+LSTM+SA | 80.3 | 68.7 | 75.7 | 75.8 |
| PG+EE (18K prog.) | 96.6 | 73.7 | 76.1 | 92.7 |
| CNN+GRU+FiLM | 98.3 | 75.6 | 80.8 | 96.9 |
| **CNN+GRU+FiLM 0-Shot** | 98.3 | **78.8** | 81.1 | 96.9 |

> **图 9**：CoGenT 结果。表中为在 ValB 上（去掉 3 万微调样本）报告的 FiLM ValB 准确率；图（曲线）为在 3 万 ValB 上微调前后的准确率。

> **图 10**：一个 CLEVR-CoGenT 例子。
> 问题：那个蓝色大圆柱体是什么材质？
> (1) 换形状：那个蓝色大**球体**是什么材质？
> (2) 换颜色：那个**绿色**大圆柱体是什么材质？
> (3) 换形状+颜色：那个**绿色**大**球体**是什么材质？
> 概念组合"蓝色"与"圆柱体"不在训练集中。我们的零样本方法通过另外三个问题的 FiLM 参数的线性组合算出原问题的 FiLM 参数：(1) + (2) − (3)。该方法把模型的答案从"橡胶"纠正为"金属"。

我们用解析器自动生成正确的问题组合，在验证集 B 上评估该方法。我们用该方法测试此前报告的 CLEVR-CoGenT FiLM 模型，结果见图 9。在 A 上训练、B 上做零样本泛化时，该方法带来 **3.2% 的总体准确率提升**。然而这个方法只能用于 B 中 **1/3** 的问题。对这些问题，模型准确率从 71.5% 起步，跃升到 80.7%。在 B 上微调之前，零样本方法与原方法在 A 上的准确率完全相同；微调之后在 B 上亦然。我们注意到，两种方法预测出的 FiLM 参数差异可忽略，这很可能是性能相近的原因。

我们**没有**为"零样本泛化"专门训练模型，就取得了这些改进。我们的方法只是让 FiLM 得以利用 CNN 训练后已有的**概念解耦**。我们还观察到，FiLM 参数的凸组合——即在 "How many cyan things are there?" 与 "How many brown things are there?" 之间——常常**单调地插值**于两个端点问题的答案之间。这些结果在有限程度上凸显了 FiLM 参数用于有意义操作的灵活性。

就其当前实现而言，该方法有许多局限。不过，词嵌入、表示学习与零样本学习中的方法都可以用来直接优化 $(\gamma, \beta)$ 以做类比生成（Bordes et al. 2013; Guu, Miller, and Liang 2015; Oh et al. 2017）。被 FiLM 作用的网络可以直接通过反向传播用该过程训练。一个学习到的模型也可以替代那个解析器。我们认为这些方向对未来工作很有希望。

## 5 结论

我们表明，一个模型可以仅靠通用的 **Feature-wise Linear Modulation 层**就取得强视觉推理能力。通过 FiLM 层以选择性、有意义的方式高效操纵神经网络的中间特征，一个 RNN 能够有效地**用语言去调制一个 CNN**，从而在图像上执行多样、多步的推理任务。我们的消融研究表明，FiLM 对架构修改、测试期消融，乃至对 FiLM 层本身的限制都有韧性。值得注意的是，我们提供了证据表明 **FiLM 的成功与归一化并无紧密联系**，这与此前假设相反。因此，我们为把该方法应用到归一化不太常用的场景（如 RNN 与强化学习）打开了大门。我们的发现还表明，FiLM 模型能更好地泛化、更样本高效，甚至能零样本迁移到陌生或更有挑战性的数据。总体而言，我们在视觉推理场景中对 FiLM 的考察结果，与更广泛的文献中"FiLM 类技术在许多领域都成功"这一结论相互印证，支持了这样一个论断：FiLM 的优势不仅限于单一领域，而是一种**通用、多才多艺**的方法。

## 6 致谢

我们感谢 PyTorch（pytorch.org）的开发者，以及 Johnson et al. (2017b) 提供的开源代码——我们的实现基于它。我们感谢 Mohammad Pezeshki、Dzmitry Bahdanau、Yoshua Bengio、Nando de Freitas、Hugo Larochelle、Laurens van der Maaten、Joseph Cohen、Joelle Pineau、Olivier Pietquin、Jérémie Mary、César Laurent、Chin-Wei Huang、Layla Asri、Max Smith 与 James Ough 的有益讨论，以及 Justin Johnson 的 CLEVR 测试评估。我们感谢 NVIDIA 捐赠用于本工作的 DGX-1 计算机。我们还感谢 FRQNT（通过 CHIST-ERA IGLU 项目）、Collège Doctoral Lille Nord de France、以及 CPER Nord-Pas de Calais/FEDER DATA Advanced data science and technologies 2015-2020 对本工作的资助。最后，我们感谢 acronymcreator.net 造出了 FiLM 这个缩写。

## 7 附录

### 7.1 错误分析

我们考察模型所犯的错误，以理解模型在何处失败、以及失败时如何表现。错误样例如图 12 与图 13 所示。

**遮挡** 许多模型错误源于部分遮挡。这类错误很可能可以用工作在**更高分辨率**下的 CNN 修复——这是可行的，因为 FiLM 的计算开销与分辨率无关。

**计数** 计数错误中有 **96.1% 是差一错误（off-by-one）**，说明 FiLM 学到了计数背后的底层概念，例如相近数字之间的紧密关系。

**逻辑一致性** 模型有时会犯人类不会犯的奇怪推理错误。例如我们发现一个案例：模型正确数出一个灰色物体和两个青色物体，却同时回答"灰色物体与青色物体数量相同"。事实上，它回答灰色物体的数量**既少于又等于**黄色块的数量。这类错误可以通过直接最小化逻辑不一致性来避免——这是一个与 FiLM 正交、值得研究的未来方向。

> **图 12**：一些模型预测错误的图像-问题对。我们观察到的大多数错误源于部分遮挡的物体，如前三个例子所示。
> 问：有没有一个与绿色物体形状相同的大棕色物体？答：有（P: 无）。
> 问：有多少个其它物体与那个大灰色圆柱体材质相同？答：6（P: 5）。
> 问：与那个小圆柱体颜色相同的大金属物体是什么形状？答：圆柱体（P: 球体）。
> 问：有多少个其它物体与那个微小球体材质相同？答：3（P: 2）。

> **图 13**：一个有趣的失败样例：模型计数正确，但比较计数时出错——它的第三个答案不正确，且与其它答案不一致。
> 问：有多少个灰色物体？答：1
> 问：有多少个青色物体？答：2
> 问：灰色物体是否与青色物体一样多？答：是
> 问：灰色物体是否比青色物体多？答：否
> 问：灰色物体是否比青色物体少？答：是

### 7.2 模型细节

我们不直接输出 $\gamma_{i,c}$，而是输出 $\Delta\gamma_{i,c}$，其中：

$$\gamma_{i,c} = 1 + \Delta\gamma_{i,c} \tag{3}$$

因为若 $\gamma_{i,c}$ 初始以 0 为中心，会把 CNN 特征图的激活清零，进而把梯度清零。在我们的实现中，我们选择输出 $\Delta\gamma_{i,c}$ 而非 $\gamma_{i,c}$，但为简单起见，本文通篇用 $\gamma_{i,c}$ 来解释 FiLM。不过这一改动在 CLEVR 上似乎**不会统计显著地**影响模型性能。

我们在图 11 中给出基于图像特征训练的最优模型的训练与验证曲线。我们观察到准确率起初快速上升，随后缓慢稳定地增长，直到最佳验证准确率 97.84%，此时训练准确率为 99.53%。我们基于图像特征在 CLEVR 上训练 80 个 epoch，使用 1 块 NVIDIA TITAN Xp GPU 耗时 4 天。出于实际考虑，我们在 80 个 epoch 后停止训练，但我们观察到此后准确率仍会缓慢继续上升。

> **图 11**：最优模型的训练与验证曲线。

### 7.3 FiLM 层学到了什么（附录补充）

我们在图 14 与图 15 中可视化 FiLM 对单个任意特征图的效果。我们还在图 16、17、18 中分别给出逐层 $\gamma_{i,c}$ 值直方图、逐层 $\beta_{i,c}$ 值直方图，以及逐通道 FiLM 参数统计量。

> **图 14**：对第一个 ResBlock 中某个任意特征图，FiLM 前后激活（缩放到 0–1）的可视化。这个特定特征图似乎在检测灰色与棕色。有趣的是，FiLM 对**特定颜色的问题**改动了特定颜色物体的激活，而对**与颜色无关的问题**则不动激活。注意因为这是第一个 FiLM 层，所有问题的 FiLM 前激活（第 1、3 行）完全相同，FiLM 后激活（第 2、4 行）的差异完全来自 FiLM 对问题信息的利用。
> 样例："那个大的橡胶圆柱体是什么颜色？"答：青色 / "那个大的橡胶球体是什么颜色？"答：灰色 / "那个立方体是什么颜色？"答：黄色 / "有多少个圆柱体？"答：4。

> **图 15**：对最后一个 ResBlock 中某个任意特征图，FiLM 影响的可视化。这个特征图似乎关注空间特征（即前/后或左/右）。注意因为这是最后一个 FiLM 层，第一行的激活已经通过若干 FiLM 层受到问题信息的影响。
> 样例："有多少个青色物体在灰色球体后面？"答：2 / "…在灰色球体前面？"答：1 / "…在灰色球体左边？"答：2 / "…在灰色球体右边？"答：1。

> **图 16**：每个 FiLM 层（从左到右为第 1–4 层）的 $\gamma_{i,c}$ 值直方图，在 CLEVR 验证集上计算。各图按相同尺度绘制。FiLM 层显得逐渐更具选择性、方差更大。

> **图 17**：每个 FiLM 层（从左到右为第 1–4 层）的 $\beta_{i,c}$ 值直方图，在 CLEVR 验证集上计算。各图按相同尺度绘制。$\beta_{i,c}$ 在第一层呈现出与后面各层不同的、方差更大的分布。

> **图 18**：逐通道 $\gamma_c$ 与 $\beta_c$ 统计量（均值与标准差）的直方图，在 CLEVR 验证集上计算。从左到右依次为：$\gamma_c$ 均值、$\gamma_c$ 标准差、$\beta_c$ 均值、$\beta_c$ 标准差。不同特征图被 FiLM 以不同模式调制；有些常被清零、另一些则很少，有些被稳定地缩放或平移相近的值、另一些则方差很大，等等。

## 参考文献

原文参考文献共 40 余条，此处不逐条翻译。与本文核心脉络直接相关的几篇：

| 文献 | 关系 |
|---|---|
| Perez et al. 2017, *Learning visual reasoning without strong priors* (MLSLP Workshop at ICML) | 本文的短版前身 |
| Ioffe and Szegedy 2015, *Batch Normalization* (ICML) | 归一化层的原始引入；CN 系方法的起点 |
| Dumoulin, Shlens, and Kudlur 2017, *A learned representation for artistic style* (ICLR) | Conditional Instance Norm（图像风格化） |
| Huang and Belongie 2017, *Arbitrary style transfer in real-time with adaptive instance normalization* (ICCV) | AdaIN |
| Kim, Song, and Bengio 2017, *Dynamic layer normalization* (InterSpeech) | Dynamic LayerNorm |
| de Vries et al. 2017, *Modulating early visual processing by language* (NIPS) | Conditional Batch Norm；本文的直接近亲 |
| van den Oord et al. 2016a, *WaveNet*；2016b, *Conditional PixelCNN* | 等价于 $\gamma = 1$ 的 FiLM |
| Hu, Shen, and Sun 2017, *Squeeze-and-Excitation Networks* | 限幅在 $(0,1)$ 的自门控缩放 |
| Ha, Dai, and Le 2016, *Hypernetworks* (ICLR) | FiLM 是超网络的一种紧凑特例 |
| Kirkpatrick et al. 2017, *Overcoming catastrophic forgetting* | 强化学习中 FiLM 变体的应用 |
| Johnson et al. 2017a (CLEVR, CVPR) / 2017b (PG+EE, ICCV) | 数据集与最强对手 |
| Santoro et al. 2017, *A simple neural network module for relational reasoning* | Relation Networks |
| Hu et al. 2017, *Learning to reason: End-to-end module networks* (ICCV) | N2NMN |
| Mikolov et al. 2013, *Distributed representations of words* (NIPS) | 零样本参数类比的启发来源 |
