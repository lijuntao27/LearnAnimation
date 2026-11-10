---
type: raw-book-translation
title: "用神经网络识别手写数字（第1章 中文译本）"
book: Neural Networks and Deep Learning
author: Michael A. Nielsen
source: ch01-handwritten-digits.md
created: 2026-09-07
tags: [deep-learning,neural-networks,book,michael-nielsen,translation]
summary: Nielsen《Neural Networks and Deep Learning》第1章中文学习译本：感知机、sigmoid 神经元、梯度下降/SGD 与 74 行 MNIST 识别实现。公式与图沿用原文，英文原章见 [[ch01-handwritten-digits]]。
---

# 用神经网络识别手写数字（第1章 中文译本）

> [!info] 译本说明
> 本文件是 Michael A. Nielsen《Neural Networks and Deep Learning》(2015) 第 1 章的中文学习译本，供逐章通读使用。正文忠实传达原文论述，关键术语首次出现时保留英文对照；公式用 LaTeX（`$...$` / `$$...$$`），图片为本地 `images/` 相对引用，代码块中注释中译、Python 代码保持原样。英文原章见 [[ch01-handwritten-digits]]。

人类的视觉系统是世界的奇迹之一。请看下面这串手写数字：

![](images/digits.png)

大多数人能不费吹灰之力地认出那是 504192。这种轻松是种假象。我们大脑的每个半球都有一块初级视觉皮层 V1，包含 1.4 亿个神经元，其间的连接有数百亿条。而人类的视觉还远不止 V1——它牵涉一整套视觉皮层 V2、V3、V4、V5，逐级做越来越复杂的图像处理。我们脑子里装着一台超级计算机，经过数亿年进化调优，极其擅长理解视觉世界。识别手写数字并不容易，恰恰相反，我们人类在解读眼睛所见这件事上天赋异禀、好得惊人。但几乎所有工作都在无意识中完成，所以我们通常意识不到视觉系统所解决的问题有多难。

视觉模式识别的难度，在你试图写一个程序去识别类似上图那样的数字时会暴露无遗。我们自己做起来毫不费力的事，忽然变得极其困难。关于"我们如何识别形状"的朴素直觉——比如"9 的顶部有个圈、右下有竖笔"——一旦要表达成算法就变得不那么简单。当你试图把这些规则精确化，很快就会迷失在例外、前置条件与特例的泥沼里。这看起来毫无希望。

神经网络用另一种方式处理这个问题。思路是拿大量手写数字——即所谓的**训练样本（training examples）**：

![](images/mnist_100_digits.png)

然后开发一个能从这些训练样本中学习的系统。换句话说，神经网络利用样本自动推断识别手写数字的规则。更进一步，通过增加训练样本的数量，网络能学到更多关于笔迹的知识，从而提高准确率。所以上面虽然只展示了 100 个训练数字，但也许我们可以用几千、几百万甚至几十亿个训练样本来构建更好的识别器。

这一章我们会写一个实现神经网络的计算机程序，让它学会识别手写数字。程序只有 **74 行**，不用任何专门的神经网络库。但这个简短的程序能在无人干预下以超过 **96%** 的准确率识别数字。后面的章节里，我们会发展出能把准确率提升到 99% 以上的技术。事实上，最好的商用神经网络已经好到被银行用来处理支票、被邮局用来识别地址。

我们聚焦手写识别，因为它是一个了解神经网络的上佳原型问题。作为原型它正好落在甜点上：它**有挑战性**——识别手写数字绝非易事——但又不至于难到需要极其复杂的解法或海量算力。此外，它也是发展深度学习等更高级技术的绝佳途径。因此整本书会反复回到手写识别问题，并在后面章节讨论如何把这些想法用到计算机视觉的其他问题，以及语音、自然语言处理等领域。

当然，如果本章的目的只是写一个识别手写数字的程序，那篇幅会短得多！但一路上我们会发展出关于神经网络的许多关键思想，包括两类重要的人工神经元（**感知机 perceptron** 与 **sigmoid 神经元**），以及神经网络的标准学习算法——**随机梯度下降（stochastic gradient descent）**。整章我着力解释**为什么**要这样做，并构建你的神经网络直觉。这比只讲机制要冗长，但为了更深的理解是值得的。收获之一是，到本章结束时我们将有能力理解什么是深度学习、以及它为何重要。

## 感知机（Perceptrons）

什么是神经网络？为开始讲解，我先介绍一种叫**感知机**的人工神经元。感知机由科学家 **Frank Rosenblatt** 于 1950–60 年代提出，灵感来自 **Warren McCulloch** 与 **Walter Pitts** 更早的工作。如今更常用的是其他人工神经元模型——本书及许多现代神经网络工作中，主要使用的神经元模型叫 **sigmoid 神经元**。我们稍后会讲到它。但要理解 sigmoid 神经元为何如此定义，值得先花时间理解感知机。

那么感知机如何工作？感知机接收若干**二进制**输入 $x_1, x_2, \ldots$，产生单个二进制输出：

![](images/tikz0.png)

示例中感知机有 3 个输入 $x_1, x_2, x_3$。一般来说它可以有更多或更少的输入。Rosenblatt 提出一个简单的规则来计算输出。他引入**权重（weights）** $w_1, w_2, \ldots$——表示各输入对输出的重要性的实数。神经元的输出 0 或 1，取决于加权和 $\sum_j w_j x_j$ 是小于还是大于某个**阈值（threshold value）**。与权重一样，阈值也是神经元的实数参数。用更精确的代数语言表述：

$$
\begin{eqnarray} \mbox{output} & = & \left\{ \begin{array}{ll} 0 & \mbox{if } \sum_j w_j x_j \leq \mbox{ threshold} \\ 1 & \mbox{if } \sum_j w_j x_j > \mbox{ threshold} \end{array} \right. \tag{1}\end{eqnarray}
$$

感知机的工作原理就这些！

这就是基本数学模型。你可以把感知机想成一种**通过权衡证据来做决定**的装置。我举个不太真实但易于理解的例子，稍后会讲到更真实的例子。假设周末快到了，你听说城里要办奶酪节。你喜欢奶酪，正纠结去不去。做决定时你可能权衡三个因素：

1. 天气好不好？
2. 你的男/女朋友想不想陪你去？
3. 节日地点离公共交通近吗？（你没有车）

我们用三个二进制变量 $x_1, x_2, x_3$ 表示这三因素。比如天气好 $x_1=1$，天气差 $x_1=0$。类似地，男/女朋友想去 $x_2=1$，不想去 $x_2=0$。公共交通同理。

现在假设你酷爱奶酪，即使男/女朋友不感兴趣、交通又不便，你也乐意去。但也许你极厌恶坏天气，天气差就绝不去。你可以用感知机为这类决策建模。一种做法：给天气选权重 $w_1=6$，另外两个条件 $w_2=2$、$w_3=2$。$w_1$ 较大表示天气对你很重要，远超男/女朋友是否陪同或交通远近。最后，选阈值 5。有了这些选择，感知机就实现了你想要的决策模型：天气好就输出 1，天气差就输出 0。男/女朋友去不去、交通近不近，对输出毫无影响。

改变权重与阈值，就得到不同的决策模型。例如把阈值改成 3，感知机就会在天气好**或**（交通近**且**男/女朋友愿意陪）时决定去。也就是说，它变成另一种决策模型。降低阈值意味着你更愿意去。

显然，感知机不是人类决策的完整模型！但这个例子展示了感知机如何权衡各种证据来做决定。而且可以想见，一个复杂的感知机网络能够做出相当精妙的决定：

![](images/tikz1.png)

这个网络中，第一列感知机——我们称之为第一**层（layer）**——通过权衡输入证据做三个很简单的决定。第二层的感知机呢？其中每个都在权衡第一层决策的结果来做决定。这样，第二层的感知机能比第一层在更复杂、更抽象层面做决定。第三层感知机还能做更复杂的决定。如此，多层感知机网络能够从事老练的决策。

顺便说明，定义感知机时我说它只有一个输出。上面网络中的感知机看起来像有多个输出，其实它们仍是单输出。多条输出箭头只是种方便的画法，表示一个感知机的输出被用作多个其他感知机的输入——比画一条再分叉的输出线更简洁。

让我们简化感知机的记法。条件 $\sum_j w_j x_j > \mbox{threshold}$ 很笨拙，我们可以做两处记号改动来化简。第一处是把 $\sum_j w_j x_j$ 写成点积 $w \cdot x \equiv \sum_j w_j x_j$，其中 $w$ 与 $x$ 分别是权重与输入组成的向量。第二处是把阈值移到不等式另一边，换成感知机的**偏置（bias）** $b \equiv -\mbox{threshold}$。用偏置取代阈值，感知机规则可重写为：

$$
\begin{eqnarray} \mbox{output} = \left\{ \begin{array}{ll} 0 & \mbox{if } w\cdot x + b \leq 0 \\ 1 & \mbox{if } w\cdot x + b > 0 \end{array} \right. \tag{2}\end{eqnarray}
$$

你可以把偏置理解为「感知机输出 1 有多容易」的度量。用更生物学的话说，偏置是感知机**激活（fire）**有多容易的度量。偏置很大时，感知机极其容易输出 1；偏置很负时，则很难输出 1。显然，引入偏置只是描述感知机的一处小改动，但后面会看到它能带来进一步的记号简化。因此本书余下部分不用阈值，一律用偏置。

我已把感知机描述为权衡证据做决定的方法。感知机的另一种用途是计算我们通常视为计算基础的基本逻辑函数，如 `AND`、`OR`、`NAND`。例如，假设一个感知机有两个输入，权重都是 $-2$，整体偏置为 $3$：

![](images/tikz2.png)

我们看到输入 $00$ 产生输出 1，因为 $(-2)*0+(-2)*0+3 = 3$ 为正。这里我用 $*$ 使乘法显式。类似计算表明 $01$ 与 $10$ 都输出 1。但输入 $11$ 输出 0，因为 $(-2)*1+(-2)*1+3 = -1$ 为负。于是这个感知机实现了一个 `NAND` 门！

`NAND` 例子说明我们可以用感知机计算简单的逻辑函数。事实上，我们可以用感知机网络计算**任何**逻辑函数。原因是 `NAND` 门对计算是通用的——任何计算都能用 `NAND` 门搭建。例如，可以用 `NAND` 门搭一个对两个比特 $x_1, x_2$ 求和的电路：需要计算按位和 $x_1 \oplus x_2$，以及当 $x_1, x_2$ 都为 1 时置 1 的进位比特，即进位就是按位积 $x_1 x_2$：

![](images/tikz3.png)

要得到等价的感知机网络，把所有 `NAND` 门换成两个输入权重均为 $-2$、整体偏置为 3 的感知机。结果网络如下。注意我把右下角 `NAND` 门对应的感知机挪了一点，只是为了让箭头好画：

![](images/tikz4.png)

这个感知机网络有一处值得注意：最左感知机的输出被两次用作最下方感知机的输入。定义感知机模型时我没说明是否允许这种「同一输出分叉到同一处」。其实无关紧要。如果不想允许这种写法，可以简单把两条线合并成一条权重为 $-4$ 的连接，而不是两条权重为 $-2$ 的连接。（如果你不觉得显然，应当停下来自己证明二者等价。）改动后网络如下，所有未标权重都是 $-2$，所有偏置都是 3，只有一条标出的 $-4$：

![](images/tikz5.png)

到目前为止我一直把 $x_1$、$x_2$ 这样的输入画成悬浮在感知机网络左边的变量。实际上，习惯上会加一层特殊的感知机——**输入层（input layer）**——来编码输入：

![](images/tikz6.png)

这种输入感知机的记号——有输出但没有输入：

![](images/tikz7.png)

是一种简写。它并不真表示没有输入的感知机。要知道为什么，假设我们确实有一个没有输入的感知机，那么加权和 $\sum_j w_j x_j$ 恒为 0，感知机就会在 $b>0$ 时输出 1、$b\le 0$ 时输出 0——也就是输出固定值，而不是期望值（上例中的 $x_1$）。更好的理解是：把输入感知机看作根本不是感知机，而是特殊单元，直接定义为输出期望值 $x_1, x_2, \ldots$。

加法器例子展示了感知机网络如何模拟含许多 `NAND` 门的电路。因为 `NAND` 门对计算是通用的，所以感知机对计算也是通用的。

感知机的计算通用性既令人安心又令人失望。安心在于它告诉我们感知机网络可以和任何计算设备一样强大；失望则在于它让感知机看起来不过是新式 `NAND` 门——这算不上大新闻！

然而，情况比这更好。事实证明，我们可以设计**学习算法（learning algorithms）**来自动调整人工神经元网络的权重与偏置。这种调整响应外部刺激，无需程序员直接干预。这些学习算法让我们以一种与常规逻辑门截然不同的方式使用人工神经元：我们不必显式地铺排 `NAND` 等门的电路，神经网络可以直接**学习**解决问题——有时是些极难直接设计常规电路的问题。

## Sigmoid 神经元

学习算法听起来很棒。但我们如何为神经网络设计这样的算法？假设我们有一个想用来学习解决某问题的感知机网络。比如网络输入是手写数字扫描图像的原始像素，我们希望网络学到合适的权重与偏置，使输出能正确分类该数字。要看学习如何可能，设想我们对某个权重（或偏置）做一点小改动。我们希望这个权重小改动只引起网络输出的**小**改动。稍后我们会看到，这个性质将使学习成为可能。示意如下（显然这个网络太简单，做不了手写识别！）：

![](images/tikz8.png)

如果权重的微小变化真的只引起输出的微小变化，那我们就能利用这一点来调整权重与偏置，让网络表现得更好。例如网络把一张图误判为"8"而应为"9"，我们能算出如何微调权重与偏置，使网络向正确分类"9"靠近一点；然后不断重复——反复调整权重与偏置，让输出越来越好。网络就在学习了。

问题在于：当网络由感知机构成时，事情并非如此。事实上，网络中**任何单个感知机**的权重或偏置的小改动，有时会让该感知机的输出**整个翻转**（从 0 变 1）。这个翻转又可能以某种复杂方式让网络其余部分的行为完全改变。于是虽然你的"9"可能被正确分类了，网络对其他所有图像的行为却很可能以难以控制的方式彻底改变。这让「逐步调整权重使网络逼近期望行为」变得困难。也许有绕过此问题的巧妙办法，但怎么让感知机网络学习，并非显而易见的。

我们可以引入一种新的人工神经元——**sigmoid 神经元**——来克服此问题。Sigmoid 神经元与感知机相似，但经过修改，使**权重与偏置的小改动只引起输出的微小改动**。这正是能让 sigmoid 神经元网络学习的要害。

好，我来描述 sigmoid 神经元。我们画 sigmoid 神经元的方式与感知机相同：

![](images/tikz9.png)

与感知机一样，sigmoid 神经元有输入 $x_1, x_2, \ldots$。但这些输入不再限于 0 或 1，还可以取 $0$ 与 $1$ 之间的**任意值**。比如 $0.638\ldots$ 就是合法输入。同样地，每个输入有权重 $w_1, w_2, \ldots$ 与整体偏置 $b$，但输出不再是 0 或 1，而是 $\sigma(w \cdot x+b)$，其中 $\sigma$ 称为 **sigmoid 函数** *，定义如下：* *顺便一提，$\sigma$ 有时也叫 **logistic 函数**，这类新神经元叫 **logistic 神经元**。记住这些术语有好处，因为许多神经网络从业者会用到；不过本书沿用 sigmoid 的说法。*：

$$
\begin{eqnarray} \sigma(z) \equiv \frac{1}{1+e^{-z}}. \tag{3}\end{eqnarray}
$$

说得更明确些，一个输入为 $x_1,x_2,\ldots$、权重为 $w_1,w_2,\ldots$、偏置为 $b$ 的 sigmoid 神经元的输出是：

$$
\begin{eqnarray} \frac{1}{1+\exp(-\sum_j w_j x_j-b)}. \tag{4}\end{eqnarray}
$$

乍看之下，sigmoid 神经元与感知机差别很大。如果你还不熟悉 sigmoid 函数的代数形式，它可能显得晦涩吓人。事实上，sigmoid 神经元与感知机有许多相似之处，sigmoid 的代数形式更多是个技术细节，而非理解上的真正障碍。

为理解与感知机模型的相似性，假设 $z \equiv w \cdot x + b$ 是很大的正数。则 $e^{-z} \approx 0$，$\sigma(z) \approx 1$。也就是说，当 $z=w\cdot x+b$ 大且正时，sigmoid 神经元输出约为 1，正如感知机那样。反过来，若 $z=w\cdot x+b$ 很负，则 $e^{-z}\rightarrow\infty$，$\sigma(z)\approx 0$。所以 $w\cdot x+b$ 很负时，sigmoid 神经元行为也高度接近感知机。只有当 $w\cdot x+b$ 量级适中时，才与感知机模型有明显偏离。

那么 $\sigma$ 的代数形式该如何理解？事实上，$\sigma$ 的确切形式并不重要——真正重要的是函数**画出来的形状**：

这条曲线是阶跃函数的光滑版：

如果 $\sigma$ 真是阶跃函数，那么 sigmoid 神经元**就是**感知机——输出取决于 $w\cdot x+b$ 正负，取 1 或 0。* *实际上，当 $w\cdot x+b=0$ 时感知机输出 0，而阶跃函数输出 1。严格说需在那一点修改阶跃函数，但你明白意思。* 而采用真正的 $\sigma$ 函数，我们得到的是一个**光滑化的感知机**。的确，**$\sigma$ 的平滑性才是要害，而非其具体形式**。$\sigma$ 的平滑意味着权重小改动 $\Delta w_j$ 与偏置小改动 $\Delta b$ 只引起神经元输出的微小变化 $\Delta \mbox{output}$。事实上，微积分告诉我们 $\Delta \mbox{output}$ 可很好地近似为：

$$
\begin{eqnarray} \Delta \mbox{output} \approx \sum_j \frac{\partial \, \mbox{output}}{\partial w_j} \Delta w_j + \frac{\partial \, \mbox{output}}{\partial b} \Delta b, \tag{5}\end{eqnarray}
$$

其中求和覆盖所有权重 $w_j$，$\partial \, \mbox{output} / \partial w_j$ 与 $\partial \, \mbox{output} /\partial b$ 分别表示 output 对 $w_j$ 与 $b$ 的偏导数。如果你对偏导数不自在，别慌！上式虽满是偏导数看着复杂，它其实在说一件非常简单（且是大好消息）的事：**$\Delta \mbox{output}$ 是权重与偏置改动 $\Delta w_j$ 与 $\Delta b$ 的线性函数**。这种线性使我们容易选择权重与偏置的小改动，来实现任意期望的输出小改动。所以 sigmoid 神经元与感知机定性行为虽大半相同，却让「权重偏置改动如何影响输出」变得容易推演。

如果重要的是 $\sigma$ 的形状而非具体形式，那为什么要用式 (3) 的那个特定形式？事实上，本书后面偶尔会考虑输出为 $f(w \cdot x + b)$ 的神经元，其中 $f(\cdot)$ 是其他**激活函数（activation function）**。换用不同激活函数，主要改变式 (5) 中偏导数的具体值。后面计算这些偏导时，用 $\sigma$ 能简化代数——只因指数函数求导有美妙性质。总之 $\sigma$ 在神经网络工作中常用，也是本书最常用的激活函数。

如何解释 sigmoid 神经元的输出？显然，sigmoid 神经元与感知机的一大区别是不只输出 0 或 1，可以输出 $0$ 到 $1$ 间的任意实数，所以 $0.173\ldots$、$0.689\ldots$ 都是合法输出。这有时很有用，比如想用输出来表示输入图像像素的平均强度。但有时也麻烦：假设我们希望网络输出表示"输入图是 9"或"输入图不是 9"。显然，若输出像感知机那样是 0 或 1 最省事。实践中我们可以约定处理，例如约定输出 $\ge 0.5$ 视为"9"、$<0.5$ 视为"非 9"。用到这类约定时我会明说，不会造成混淆。

#### 练习

- **Sigmoid 神经元模拟感知机，第一部分** $\mbox{}$  
  假设把感知机网络中的所有权重与偏置乘以正常数 $c>0$。证明网络行为不变。
- **Sigmoid 神经元模拟感知机，第二部分** $\mbox{}$  
  假设与上题同样的感知机网络设置，且感知机网络的整体输入已选定（不需要实际输入值，只要求输入已固定）。设权重与偏置使得对网络中任一感知机的输入 $x$ 都有 $w\cdot x+b\neq 0$。现在把网络中所有感知机换成 sigmoid 神经元，并把权重与偏置乘以正常数 $c>0$。证明当 $c\rightarrow\infty$ 时，这个 sigmoid 神经元网络的行为与感知机网络完全一致。当某个感知机 $w\cdot x+b=0$ 时，这个结论为何会失效？

## 神经网络的架构

下一节我会介绍一个能把手写数字分类得不错的神经网络。为做准备，先解释一些给网络不同部分命名的术语。假设有如下网络：

![](images/tikz10.png)

如前所述，网络最左层叫**输入层（input layer）**，其中的神经元叫**输入神经元（input neurons）**。最右层叫**输出层（output layer）**，含**输出神经元（output neurons）**，此处是单个输出神经元。中间层叫**隐藏层（hidden layer）**，因为其中神经元既非输入也非输出。"隐藏"听着有点神秘——我第一次听到以为必有深奥的哲学或数学含义——其实它不过指"不是输入也不是输出"。上图只有一个隐藏层，有些网络有多个隐藏层。例如下面这个四层网络就有两个隐藏层：

![](images/tikz11.png)

有点令人困惑的是，出于历史原因，这种多层网络有时叫**多层感知机（multilayer perceptrons，MLP）**——尽管它们由 sigmoid 神经元而非感知机构成。本书不用 MLP 这个叫法（我觉得它容易混淆），但提醒你它的存在。

网络输入层与输出层的设计往往直截了当。例如，假设我们要判断手写图像是否为"9"。自然的设计是把图像像素强度编码进输入神经元：若是 $64\times64$ 灰度图，就有 $4096=64\times64$ 个输入神经元，强度缩放到 0 到 1 之间。输出层只含一个神经元，输出 $<0.5$ 表示"非 9"，$>0.5$ 表示"是 9"。

输入输出层设计常直白，隐藏层设计却颇有门道。特别是，隐藏层设计无法用几条简单经验法则概括。相反，神经网络研究者发展出许多隐藏层设计启发式，帮人们获得想要的网络行为。例如这些启发式可用于权衡隐藏层数量与训练时间。本书后面会遇到若干此类设计启发式。

到此我们讨论的都是「一层输出用作下一层输入」的网络，称为**前馈（feedforward）**神经网络。这意味着网络无回路——信息总是向前传，从不回馈。若有回路，就会出现 $\sigma$ 的输入依赖自身输出的情况，那将难以理解，所以不允许回路。

不过也有允许反馈回路的其他人工神经网络模型，称为**循环神经网络（recurrent neural networks）**。其思想是让神经元先激活一段有限时间再安静下来；激活会刺激其他神经元稍后也激活有限时间，再刺激更多神经元，于是随时间形成神经元激活的级联。此类模型中回路不成问题，因为神经元的输出只会在**稍后**影响其输入，而非瞬时。

循环神经网相对前馈网络影响力较小，部分因为循环网络的学习算法（至少迄今）较弱。但循环网络仍极为有趣：它们在精神上远比前馈网络接近大脑的工作方式，而且可能解决前馈网络极难解决的重要问题。不过为限定范围，本书专注于应用更广的前馈网络。

## 一个用于手写数字分类的简单网络

定义完神经网络，回到手写识别。我们可以把识别手写数字的问题分成两个子问题。第一，把含多个数字的图像切成一串单数字图像。例如把下图：

![](images/digits.png)

切成六张独立图像：

![](images/digits_separate.png)

我们人类轻松解决这个**分割问题（segmentation problem）**，但让计算机程序正确切分却很有挑战。切分之后，程序需要逐个分类数字。例如我们希望程序认出下面第一个数字：

![](images/mnist_first_digit.png)

是 5。

我们专注于写程序解决第二个问题，即分类单个数字。这样做是因为：一旦有了好的单数字分类器，分割问题其实没那么难解。分割有很多方法。一种是对图像尝试许多不同切法，用单数字分类器给每种试切打分：若分类器对每个切块都高置信度分类，试切得分就高；若一个或多个切块让分类器很为难，得分就低。思路是：若分类器在某处犯难，多半是切法选错了。这个思路的变体能相当好地解决分割问题。因此我们暂不担心分割，专注开发能解决更有趣也更困难问题——识别单个手写数字——的神经网络。

为识别单个数字，我们将用一个三层神经网络：

![](images/tikz12.png)

网络输入层包含编码输入像素值的神经元。下一节会讨论，训练数据由许多 $28\times28$ 像素的手写数字扫描图组成，因此输入层含 $784=28\times28$ 个神经元。上图我简化省略了大部分 784 个输入神经元。输入像素是灰度的：值 $0.0$ 表示白，$1.0$ 表示黑，中间值表示渐深的灰。

网络第二层是隐藏层。我们把隐藏层神经元数记为 $n$，会实验不同的 $n$。示例画了一个仅含 $n=15$ 个神经元的小隐藏层。

输出层含 10 个神经元。若第一个神经元激活（即输出 $\approx 1$），表示网络认为数字是 0；第二个神经元激活表示认为是 1；以此类推。更精确些：我们给输出神经元编号 0 到 9，看哪个神经元激活值最高。若最高的是 6 号神经元，网络就猜输入数字是 6，等等。

你可能疑惑为何用 10 个输出神经元。毕竟网络目标是告诉我们输入图对应哪个数字（0 到 9）。一个看似自然的做法是用 4 个输出神经元，把每个神经元按输出接近 0 还是 1 当作一个二进制位。4 个神经元足以编码答案，因为 $2^4=16$ 大于 10 种可能数字。为什么反而用 10 个神经元？岂不低效？最终理由是经验的：两种设计都试过，对这个特定问题，10 输出网络比 4 输出网络学得更好。但这让我们好奇**为何** 10 输出更好。是否有启发式能预先告诉我们该用 10 输出编码而非 4 输出编码？

为理解这一点，从第一性原理想想神经网络在做什么。先看 10 输出的情形。聚焦第一个输出神经元，它负责判断数字是否为 0。它通过权衡隐藏层神经元的证据来做决定。那些隐藏神经元在做什么？姑且假设（为了论证）隐藏层第一个神经元检测下图所示图像是否存在：

![](images/mnist_top_left_feature.png)

它可以通过对与图像重叠的输入像素给大权重、对其余输入给小权重来实现。类似地，姑且假设隐藏层第二、三、四个神经元分别检测下列图像是否存在：

![](images/mnist_other_features.png)

你大概猜到了：这四张图拼起来正是前面那串数字里的 0 图像：

![](images/mnist_complete_zero.png)

所以若这四个隐藏神经元都激活，就能推断数字是 0。当然这不是推断 0 的唯一证据——0 的写法还有很多（比如上述图像的平移、轻微形变）。但至少在此情形下，说输入是 0 是稳妥的。

假设神经网络确实这样运作，我们就能给出「为何 10 个输出比 4 个好」的合理解释。若只有 4 个输出，第一个输出神经元要判断数字的最高有效位是什么。而最高有效位与上图这类简单形状很难建立联系。很难想象数字的组成部件形状会与（比如）输出的最高有效位有某种自然的因果联系。

话虽如此，以上都只是启发式。没有任何东西规定三层网络必须按我描述的方式运作、隐藏神经元必须检测简单部件形状。也许某个聪明的学习算法能找到只用 4 个输出神经元的权重分配。但作为一种启发式，上述思考方式相当好用，能在设计良好神经网络架构时省下大量时间。

#### 练习

- 有一种方法能通过给上述三层网络**增加一层**来确定数字的按位表示。新层把前一层输出转成二进制表示，如下图。试找出新输出层的一组权重与偏置。假设前 3 层神经元使得第三层（即原输出层）的正确输出激活值 $\ge 0.99$，错误输出激活值 $<0.01$。

![](images/tikz13.png)

## 用梯度下降学习

设计好了网络架构，它如何学会识别数字？首先需要用来学习的数据集——即**训练数据集（training data set）**。我们用 **MNIST 数据集**，其中含数万张手写数字扫描图及正确分类。MNIST 得名于它是 NIST（美国国家标准与技术研究院）两个数据集的改良子集。下面是 MNIST 的一些图像：

![](images/digits_separate.png)

如你所见，这些数字其实与本章开头作为挑战展示的数字相同。当然，测试网络时我们会让它识别**不在**训练集里的图像！

MNIST 数据分两部分。第一部分 60,000 张图用作训练数据：250 人的手写扫描样本，一半是美国人口普查局雇员，一半是高中生；图像灰度、$28\times28$ 像素。第二部分 10,000 张用作测试数据，同样是 $28\times28$ 灰度图。我们用测试数据评估网络学得如何。为保证测试有效，测试数据来自**另一批** 250 人（虽仍是普查局雇员与高中生的混合）。这有助于让我们确信系统能识别训练时没见过的人的笔迹。

我们用记号 $x$ 表示训练输入。把每个训练输入 $x$ 看作 $28\times28=784$ 维向量会很方便，每项代表图中单个像素的灰度值。对应期望输出记为 $y=y(x)$，是 10 维向量。例如某训练图 $x$ 画的是 6，则 $y(x)=(0,0,0,0,0,0,1,0,0,0)^T$ 是期望输出。注意 $T$ 是转置操作，把行向量变成通常的（列）向量。

我们想要一个能找出权重与偏置、使网络输出对所有训练输入 $x$ 都逼近 $y(x)$ 的算法。为量化做得多好，定义**代价函数（cost function）** *（有时叫损失 loss 或目标 objective 函数。本书通篇用代价函数，但你要留意其他叫法，论文和讨论中常用。）*：

$$
\begin{eqnarray} C(w,b) \equiv \frac{1}{2n} \sum_x \| y(x) - a\|^2. \tag{6}\end{eqnarray}
$$

这里 $w$ 表示网络中所有权重的集合，$b$ 所有偏置，$n$ 训练输入总数，$a$ 是输入 $x$ 时网络的输出向量，求和覆盖所有训练输入 $x$。当然，输出 $a$ 依赖 $x$、$w$、$b$，但为记号简洁我没显式标出。$\|v\|$ 只是向量 $v$ 的通常长度。我们把 $C$ 称为**二次代价（quadratic cost）**函数，也称**均方误差（mean squared error）**或 **MSE**。审视二次代价的形式：$C(w,b)$ 非负，因为求和每项都非负。而且当代价 $C(w,b)$ 很小（$C(w,b)\approx0$）时，恰是所有训练输入 $x$ 的 $y(x)$ 都近似等于输出 $a$。所以若训练算法能找到使 $C(w,b)\approx0$ 的权重与偏置，它就干得很棒；反之 $C(w,b)$ 很大说明做得不好——意味着大量输入上 $y(x)$ 不接近输出 $a$。因此训练算法的目标就是最小化作为权重与偏置函数的代价 $C(w,b)$。换言之，找一组使代价尽可能小的权重与偏置。我们用一个叫**梯度下降（gradient descent）**的算法来做。

为何引入二次代价？毕竟我们主要关心的不是网络正确分类的图像数吗？为什么不直接最大化正确分类数，而要最小化一个代理指标？问题在于：正确分类的图像数不是权重与偏置的**平滑**函数。多数情况下，微调权重与偏置完全不会改变正确分类的训练图数，这使「如何改权重偏置以提升表现」难以推演。若改用二次代价这样的平滑函数，就容易推演出如何微调权重与偏置来改进代价。这就是我们先把重心放在最小化二次代价、之后才看分类准确率的原因。

即便要用平滑代价函数，你可能仍疑惑为什么选式 (6) 的二次函数。这岂不是很**临时拼凑（ad hoc）**？若换别的代价函数，会不会得到完全不同的最小化权重与偏置？这是合理的顾虑，后面我们会回到代价函数并做修改。不过对理解神经网络学习的基础，式 (6) 的二次代价完全够用，现在先沿用。

回顾：训练神经网络的目标是找使二次代价 $C(w,b)$ 最小化的权重与偏置。这是个适定问题，但当前提法含大量干扰结构——把 $w$、$b$ 解释为权重偏置、背景里的 $\sigma$ 函数、网络架构选择、MNIST 等等。事实证明，忽略其中大部分结构、只专注于最小化本身，我们能理解极多。所以暂时忘掉代价函数的具体形式、与神经网络的联系等。相反，想象我们只是被给了一个多变量函数并要最小化它。我们将发展一种叫**梯度下降**的技术来解决这类最小化问题，然后再回到神经网络那个要最小化的具体函数。

好，假设要最小化某个函数 $C(v)$。它可以是多变量 $v=v_1,v_2,\ldots$ 的任意实值函数。注意我把 $w$、$b$ 记号换成 $v$，以强调这里可以是任意函数——不再特指神经网络场景。为最小化 $C(v)$，不妨想象 $C$ 只是两个变量 $v_1, v_2$ 的函数：

![](images/valley.png)

我们想找 $C$ 的全局最小值。当然，上图这种函数可以直接肉眼看出最小值。就这点而言我画的函数或许**太**简单了！一般的 $C$ 可能是许多变量的复杂函数，通常无法靠肉眼找出最小值。

一种进攻方式是微积分：计算导数，试着找 $C$ 取极值的位置。若 $C$ 只依赖一两个变量，碰碰运气或许可行；但当变量很多时就是噩梦。而对神经网络，我们常需要**远远更多**的变量——最大的神经网络，其代价函数依赖数十亿权重与偏置，关系极其复杂。用微积分最小化它根本行不通！

（我刚断言「想象 $C$ 是两个变量会带来洞见」，又连着两段说"可若是更多变量呢？"抱歉。请相信：想象 $C$ 是两变量函数真的有用，只是有时这个图像会失效，上两段就在处理这种失效。好的数学思考常需同时摆弄多个直觉图像，并学会何时用哪个、何时不用。）

好，微积分不行。幸运的是，有个美妙类比暗示了一个相当好用的算法。先把函数想成一种山谷。眯眼看看上图，应该不难。再想象一个小球沿山坡滚下。日常经验告诉我们，球最终会滚到谷底。也许我们能借这个想法找函数最小值：随机选个（想象中）小球的起点，模拟球滚到谷底的运动。我们可以只靠计算 $C$ 的导数（也许还有二阶导数）来模拟——这些导数告诉我们山谷局部的全部"形状"，从而决定球该怎么滚。

读到这里你可能以为我们要写球的牛顿运动方程，考虑摩擦、重力等。其实我们不会把滚球类比看得那么认真——我们在设计最小化 $C$ 的算法，而不是精确模拟物理定律！球眼视角意在激发想象，而非约束思考。所以与其陷入物理的种种细节，不如问自己：若我们能当一天上帝、自定物理定律来规定球怎么滚，该选怎样的运动定律才能让球**总是**滚到谷底？

为让问题更精确，想想我们让球沿 $v_1$ 方向移动小量 $\Delta v_1$、沿 $v_2$ 方向移动小量 $\Delta v_2$ 时会发生什么。微积分告诉我们 $C$ 的变化为：

$$
\begin{eqnarray} \Delta C \approx \frac{\partial C}{\partial v_1} \Delta v_1 + \frac{\partial C}{\partial v_2} \Delta v_2. \tag{7}\end{eqnarray}
$$

我们要找一种选择 $\Delta v_1$、$\Delta v_2$ 的办法，使 $\Delta C$ 为负——即让球向谷内滚。为找出这种选法，定义 $\Delta v$ 为改动向量 $\Delta v\equiv(\Delta v_1,\Delta v_2)^T$，$T$ 仍是转置，把行向量变成列向量。再定义 $C$ 的**梯度（gradient）**为偏导向量 $\left(\frac{\partial C}{\partial v_1},\frac{\partial C}{\partial v_2}\right)^T$。梯度向量记为 $\nabla C$：

$$
\begin{eqnarray} \nabla C \equiv \left( \frac{\partial C}{\partial v_1}, \frac{\partial C}{\partial v_2} \right)^T. \tag{8}\end{eqnarray}
$$

稍后我们会用 $\Delta v$ 与梯度 $\nabla C$ 重写 $\Delta C$。在那之前，先澄清一个常让人卡住的地方。初次见 $\nabla C$ 记号时，人们常疑惑该怎么理解 $\nabla$ 符号。$\nabla$ 到底是什么意思？其实完全可以把 $\nabla C$ 当作一个数学对象——上面定义的向量——只不过用两个符号书写。这个观点下，$\nabla$ 只是记号上的旗语，告诉你"嘿，$\nabla C$ 是个梯度向量"。还有更高级的观点把 $\nabla$ 看作独立数学实体（例如微分算子），但我们用不到。

有了这些定义，式 (7) 的 $\Delta C$ 可改写为：

$$
\begin{eqnarray} \Delta C \approx \nabla C \cdot \Delta v. \tag{9}\end{eqnarray}
$$

这个方程有助于解释为何 $\nabla C$ 叫梯度向量：$\nabla C$ 把 $v$ 的变化联系到 $C$ 的变化，正如"梯度"该有的样子。但真正令人兴奋的是，它让我们看到如何选 $\Delta v$ 使 $\Delta C$ 为负。特别地，假设我们选：

$$
\begin{eqnarray} \Delta v = -\eta \nabla C, \tag{10}\end{eqnarray}
$$

其中 $\eta$ 是小的正常数，叫**学习率（learning rate）**。那么式 (9) 告诉我们 $\Delta C \approx -\eta \nabla C \cdot \nabla C = -\eta \|\nabla C\|^2$。因为 $\|\nabla C\|^2 \ge 0$，这保证 $\Delta C \le 0$——即按式 (10) 的处方改变 $v$，$C$ **总会下降、永不上升**。（当然，限于式 (9) 近似的范围内。）这正是我们想要的！于是我们取式 (10) 定义梯度下降算法中"球"的运动定律：用式 (10) 算出 $\Delta v$，再把球的位置 $v$ 移动这个量：

$$
\begin{eqnarray} v \rightarrow v' = v -\eta \nabla C. \tag{11}\end{eqnarray}
$$

然后用此更新规则再走一步。反复进行，就不断降低 $C$——但愿直到全局最小。

总结：梯度下降的工作方式就是反复计算梯度 $\nabla C$，然后向**相反**方向移动——"沿山坡下落"。可以这样想象：

![](images/valley_with_ball.png)

注意按此规则，梯度下降并不复现真实物理运动。现实中球有**动量**，动量可能让它滚过斜坡甚至（短暂）上坡；只有摩擦生效后球才保证滚入谷底。而我们的 $\Delta v$ 规则只说"现在，向下"。这仍是找最小值的相当好的规则！

要让梯度下降正确工作，需选足够小的学习率 $\eta$，使式 (9) 是好的近似。否则可能得到 $\Delta C>0$，那显然不妙！同时也不希望 $\eta$ 太小——那会让 $\Delta v$ 微小，梯度下降走得极慢。实际实现中常动态调整 $\eta$，使式 (9) 保持良好近似又不至于太慢。后面会看到如何做。

我解释的是 $C$ 为两变量函数的梯度下降。但实际上，$C$ 是更多变量函数时一切同样成立。特别地，设 $C$ 是 $m$ 个变量 $v_1,\ldots,v_m$ 的函数。小改动 $\Delta v=(\Delta v_1,\ldots,\Delta v_m)^T$ 引起的 $C$ 变化为：

$$
\begin{eqnarray} \Delta C \approx \nabla C \cdot \Delta v, \tag{12}\end{eqnarray}
$$

其中梯度 $\nabla C$ 是向量：

$$
\begin{eqnarray} \nabla C \equiv \left(\frac{\partial C}{\partial v_1}, \ldots, \frac{\partial C}{\partial v_m}\right)^T. \tag{13}\end{eqnarray}
$$

与两变量情形一样，可选：

$$
\begin{eqnarray} \Delta v = -\eta \nabla C, \tag{14}\end{eqnarray}
$$

且保证式 (12) 的（近似）$\Delta C$ 为负。这就给了我们一种即使 $C$ 是多变量函数也能循梯度到最小值的方法——反复应用更新规则：

$$
\begin{eqnarray} v \rightarrow v' = v-\eta \nabla C. \tag{15}\end{eqnarray}
$$

你可以把这个更新规则看作**定义了**梯度下降算法。它给了我们反复改变位置 $v$ 来找 $C$ 最小值的方法。此规则并非总有效——有几处可能出错、阻碍梯度下降找到 $C$ 的全局最小值，我们会在后面章节探讨。但实践中梯度下降常效果极佳，在神经网络里它是最小化代价函数的强力手段，能帮网络学习。

事实上，某种意义下梯度下降是搜索最小值的最优策略。假设我们想移动 $\Delta v$ 以尽可能降低 $C$。这等价于最小化 $\Delta C \approx \nabla C \cdot \Delta v$。限制移动大小为 $\|\Delta v\|=\epsilon$（$\epsilon>0$ 为小常数）——即要一个固定大小的小步，找最能降低 $C$ 的方向。可以证明，使 $\nabla C \cdot \Delta v$ 最小的 $\Delta v$ 是 $\Delta v=-\eta\nabla C$，其中 $\eta=\epsilon/\|\nabla C\|$ 由大小约束 $\|\Delta v\|=\epsilon$ 决定。所以梯度下降可看作：每一步朝「当下最能降低 $C$」的方向走一小步。

#### 练习

- 证明上一段的断言。*提示：若你还不熟悉 [Cauchy-Schwarz 不等式](http://en.wikipedia.org/wiki/Cauchy%E2%80%93Schwarz_inequality)，熟悉一下可能有帮助。*
- 我解释了 $C$ 为两变量函数、以及多于两变量函数时的梯度下降。$C$ 只是单变量函数时会发生什么？你能对一维情形的梯度下降给出几何解释吗？

人们研究过梯度下降的许多变体，包括更接近真实物理球的变体。这些"拟球"变体有些优点，但有个大缺点：需要计算 $C$ 的**二阶偏导数**，代价可能很高。为何昂贵？假设要算所有二阶偏导 $\partial^2 C/\partial v_j\partial v_k$。若有一百万个 $v_j$ 变量，就要算约一万亿（一百万平方）个二阶偏导。*（实际约五万亿的一半，因为 $\partial^2 C/\partial v_j\partial v_k=\partial^2 C/\partial v_k\partial v_j$。总之你懂意思。）* 那计算量很大。话虽如此，有些技巧能规避这类问题，寻找梯度下降的替代方案也是活跃的研究领域。但本书以梯度下降（及其变体）为神经网络学习的主要方法。

如何把梯度下降用到神经网络的学习上？思路是用梯度下降找使式 (6) 代价最小化的权重 $w_k$ 与偏置 $b_l$。为看清如何运作，把梯度下降更新规则用权重与偏置改写，即用权重偏置替换变量 $v_j$。换言之，我们的"位置"现在有分量 $w_k$ 与 $b_l$，梯度向量 $\nabla C$ 有相应分量 $\partial C/\partial w_k$ 与 $\partial C/\partial b_l$。把梯度下降更新规则按分量写开：

$$
\begin{eqnarray} w_k & \rightarrow & w_k' = w_k-\eta \frac{\partial C}{\partial w_k} \tag{16}\\ b_l & \rightarrow & b_l' = b_l-\eta \frac{\partial C}{\partial b_l}. \tag{17}\end{eqnarray}
$$

反复应用此更新规则，我们就能"滚下山"，有望找到代价函数最小值。换言之，这是能让神经网络学习的规则。

应用梯度下降规则有不少挑战，我们会在后面章节深入。但此刻我只想提一个问题。为理解这问题，回顾式 (6) 的二次代价。注意它形如 $C=\frac{1}{n}\sum_x C_x$——是各样本代价 $C_x\equiv\frac{\|y(x)-a\|^2}{2}$ 的平均。实践中，要计算梯度 $\nabla C$，需对每个训练输入 $x$ 分别计算梯度 $\nabla C_x$ 再取平均：$\nabla C=\frac{1}{n}\sum_x\nabla C_x$。不幸的是，训练输入极多时这会很慢，学习因此迟缓。

一个叫**随机梯度下降（stochastic gradient descent，SGD）**的想法可用来加速学习。其思路是用一小撮随机选出的训练输入来**估计**梯度 $\nabla C$——通过平均这小小样本，我们能快速得到真实梯度 $\nabla C$ 的好估计，从而加速梯度下降与学习。

为把这些想法讲精确，SGD 随机挑出 $m$ 个随机训练输入，记为 $X_1,X_2,\ldots,X_m$，称为一个**小批量（mini-batch）**。只要样本量 $m$ 够大，我们预期 $\nabla C_{X_j}$ 的平均大致等于所有 $\nabla C_x$ 的平均：

$$
\begin{eqnarray} \frac{\sum_{j=1}^m \nabla C_{X_{j}}}{m} \approx \frac{\sum_x \nabla C_x}{n} = \nabla C, \tag{18}\end{eqnarray}
$$

其中第二个求和覆盖整个训练集。换边得：

$$
\begin{eqnarray} \nabla C \approx \frac{1}{m} \sum_{j=1}^m \nabla C_{X_{j}}, \tag{19}\end{eqnarray}
$$

确认我们只需计算随机小批量的梯度即可估计整体梯度。

为把它与神经网络学习明确挂钩，设 $w_k$、$b_l$ 是网络中的权重与偏置。SGD 随机挑出一个小批量训练输入，并用它们训练：

$$
\begin{eqnarray} w_k & \rightarrow & w_k' = w_k-\frac{\eta}{m} \sum_j \frac{\partial C_{X_j}}{\partial w_k} \tag{20}\\ b_l & \rightarrow & b_l' = b_l-\frac{\eta}{m} \sum_j \frac{\partial C_{X_j}}{\partial b_l}, \tag{21}\end{eqnarray}
$$

其中求和覆盖当前小批量内的所有训练样本 $X_j$。然后另挑一个小批量继续训练，如此反复直到用完所有训练输入——这算完成一个**回合（epoch）**的训练。此时开始新的训练回合。

顺带一提，关于代价函数与权重偏置小批量更新的缩放，各家约定有差异。式 (6) 里我们把整体代价乘了 $\frac1n$。有人省略 $\frac1n$，直接对单个样本代价求和而非平均——当训练样本总数事先未知时特别有用（例如实时生成更多训练数据）。类似地，小批量更新规则 (20)(21) 有时省略和号前的 $\frac1m$。概念上差别很小，因为它等价于缩放学习率 $\eta$。但细读不同工作时值得留意。

可以把 SGD 想成政治民调：抽一小批量远比全批量梯度下降省事，正如做民调远比办完整选举容易。例如若训练集 $n=60,000$（MNIST），选小批量 $m=10$，估计梯度的速度就快 6,000 倍！当然，估计不会完美——会有统计涨落——但也不需要完美：我们在意的只是朝大致能降低 $C$ 的方向走，不需要精确梯度。实践中 SGD 是神经网络学习常用且强大的技术，也是本书大部分学习技术的基础。

#### 练习

- 梯度下降的一个极端版本是用大小为 1 的小批量。即给定训练输入 $x$，按规则 $w_k\rightarrow w_k'=w_k-\eta\partial C_x/\partial w_k$、$b_l\rightarrow b_l'=b_l-\eta\partial C_x/\partial b_l$ 更新权重偏置；然后选另一个训练输入再次更新，如此反复。这个过程叫**在线（online）、on-line 或增量（incremental）学习**——网络一次只从一个训练输入学习（正如人类）。与批量大小为 20 的 SGD 相比，说出在线学习的一个优点与一个缺点。

让我以一点常困扰新接触梯度下降的人收尾。神经网络里代价 $C$ 当然是许多变量（所有权重偏置）的函数，某种意义上在高维空间定义了一个曲面。有人会纠结："嘿，我得把所有这些额外维度可视化。"进而担心："我连四维都想象不了，更别说五维（或五百万维）。"是不是少了某种特殊能力——那种"真正的超级数学家"才有的能力？当然不是。即便多数专业数学家也不能特别好地可视化四维。他们用的技巧是发展**其他**方式来表征正在发生的事。这正是我们上面做的：用 $\Delta C$ 的代数（而非视觉）表征，推演出如何移动以降低 $C$。擅长高维思考的人，其心智库里有许多诸如此类的不同技巧；我们的代数技巧只是其一。这些技巧或许没有三维可视化的那种简单，但一旦建起这类技巧库，你就能相当擅长高维思考。这里我不展开，若你感兴趣，可读读[这篇讨论](http://mathoverflow.net/questions/25983/intuitive-crutches-for-higher-dimensional-thinking)——专业数学家用来思考高维空间的一些技巧。其中一些相当复杂，但最好的内容直觉而平易，任何人都能掌握。

## 实现我们的数字分类网络

好，让我们写个用 SGD 与 MNIST 训练数据学习识别手写数字的程序。这是个很短的 Python (2.7) 程序，仅 74 行！首先需要取得 MNIST 数据。若你用 git，可克隆本书代码仓库获得数据：

```
git clone https://github.com/mnielsen/neural-networks-and-deep-learning.git
```

不用 git 可在此[下载数据与代码](https://github.com/mnielsen/neural-networks-and-deep-learning/archive/master.zip)。

顺便说明，我之前讲 MNIST 时说是 60,000 训练图与 10,000 测试图，那是 MNIST 官方说法。实际上我们稍作不同的切分：测试图保持不变，但把 60,000 张 MNIST 训练集切成两部分——50,000 张用于训练神经网络，另 10,000 张作**验证集（validation set）**。本章不会用验证数据，但后面章节会用它来设定某些**超参数（hyper-parameters）**——像学习率等不由学习算法直接挑选的东西。虽然验证数据不是 MNIST 原规范的一部分，许多人这样用 MNIST，且验证集在神经网络中很常用。从现在起，当我说"MNIST 训练数据"，指的是我们的 50,000 张数据集，而非原 60,000 张。*（如前所述，MNIST 基于 NIST 收集的两个数据集。为构造 MNIST，Yann LeCun、Corinna Cortes 与 Christopher J. C. Burges 把 NIST 数据集精简并整理成更方便的格式，详见[此链接](http://yann.lecun.com/exdb/mnist/)。我仓库里的数据集形式便于在 Python 中加载与操作 MNIST。我从蒙特利尔大学 LISA 机器学习实验室获得这份数据[链接](http://www.deeplearning.net/tutorial/gettingstarted.html)。）*

除 MNIST 数据外，还需要 Python 库 **Numpy**（做快速线性代数）。若未装 Numpy，可到[这里](http://www.scipy.org/install.html)获取。

让我先解释神经网络代码的核心特性，再给出完整清单。核心是一个表示神经网络的 `Network` 类。初始化 Network 对象的代码如下：

```
class Network(object):

    def __init__(self, sizes):
        self.num_layers = len(sizes)
        self.sizes = sizes
        self.biases = [np.random.randn(y, 1) for y in sizes[1:]]
        self.weights = [np.random.randn(y, x) 
                        for x, y in zip(sizes[:-1], sizes[1:])]
```

这段代码中，列表 `sizes` 含各层神经元数。例如想建一个第一层 2 神经元、第二层 3 神经元、最后一层 1 神经元的 Network 对象：

```
net = Network([2, 3, 1])
```

Network 对象的偏置与权重都用 `np.random.randn` 随机初始化——生成均值 0、标准差 1 的高斯分布。这种随机初始化给 SGD 算法一个起点。后面章节会找更好的初始化方式，但现在这就够了。注意 Network 初始化代码假设第一层是输入层，因此不为那些神经元设偏置——偏置只用于计算后续层输出。

还要注意偏置与权重存为 Numpy 矩阵列表。`net.weights[1]` 是存连接第二层与第三层神经元权重的 Numpy 矩阵。（不是第一与第二层，因为 Python 列表索引从 0 开始。）`net.weights[1]` 太啰嗦，就记该矩阵为 $w$。$w_{jk}$ 表示连接第二层第 $k$ 个神经元到第三层第 $j$ 个神经元的权重。这个 $j$、$k$ 顺序看着奇怪——对调岂不更自然？这么排的一大优点是：第三层神经元的激活向量可写成：

$$
\begin{eqnarray} a' = \sigma(w a + b). \tag{22}\end{eqnarray}
$$

这个方程信息量很大，逐块拆解。$a$ 是第二层神经元的激活向量。为得 $a'$，把 $a$ 乘以权重矩阵 $w$，加上偏置向量 $b$；再对向量 $wa+b$ 的每个元素逐个应用 $\sigma$（这叫把函数 $\sigma$ **向量化（vectorizing）**）。容易验证式 (22) 与我们早先计算 sigmoid 神经元输出的规则式 (4) 结果一致。

#### 练习

- 把式 (22) 按分量写开，验证它与式 (4) 计算 sigmoid 神经元输出的规则一致。

有了这些，写计算 Network 实例输出的代码就很容易。先定义 sigmoid 函数：

```
def sigmoid(z):
    return 1.0/(1.0+np.exp(-z))
```

注意当输入 $z$ 是向量或 Numpy 数组时，Numpy 会自动对每个元素应用 sigmoid，即向量化形式。

然后给 Network 类加 `feedforward` 方法：给定网络输入 `a`，返回对应输出。*（假定输入 a 是 (n, 1) 的 Numpy ndarray，而非 (n,) 向量，n 为网络输入数。若用 (n,) 向量会得到奇怪结果。虽然 (n,) 向量看似更自然，用 (n,1) ndarray 便于日后把代码改成一次前向多个输入。二者各有用处。）* 该方法不过是对每层应用式 (22)：

```
def feedforward(self, a):
        """返回网络在输入 "a" 时的输出"""
        for b, w in zip(self.biases, self.weights):
            a = sigmoid(np.dot(w, a)+b)
        return a
```

当然，我们希望 Network 对象做的主要事情是**学习**。为此给它们加一个实现 SGD 的 `SGD` 方法。代码如下。有几处略神秘，但下面完整清单后会逐段拆解。

```
def SGD(self, training_data, epochs, mini_batch_size, eta,
            test_data=None):
        """使用小批量随机梯度下降训练神经网络。
        "training_data" 是 "(x, y)" 元组列表，表示训练输入与期望输出。
        其余非可选参数不言自明。若提供 "test_data"，网络将在每回合
        训练后评测一次并打印进度，便于跟踪，但会明显拖慢速度。"""
        if test_data: n_test = len(test_data)
        n = len(training_data)
        for j in xrange(epochs):
            random.shuffle(training_data)
            mini_batches = [
                training_data[k:k+mini_batch_size]
                for k in xrange(0, n, mini_batch_size)]
            for mini_batch in mini_batches:
                self.update_mini_batch(mini_batch, eta)
            if test_data:
                print "Epoch {0}: {1} / {2}".format(
                    j, self.evaluate(test_data), n_test)
            else:
                print "Epoch {0} complete".format(j)
```

`training_data` 是 `(x, y)` 元组列表，表示训练输入与对应期望输出。`epochs` 与 `mini_batch_size` 如你所料——训练回合数与小批量大小。`eta` 是学习率 $\eta$。若提供可选参数 `test_data`，程序会在每回合训练后评测网络并打印进度。这便于跟踪，但会明显拖慢运行。

代码工作方式：每回合先随机打乱训练数据，再切成适当大小的小批量——这是从训练数据随机抽样的简单办法。然后对每个 `mini_batch` 施加一步梯度下降，由 `self.update_mini_batch(mini_batch, eta)` 完成——它按单次梯度下降迭代更新网络权重与偏置，只用 `mini_batch` 里的训练数据。`update_mini_batch` 方法代码如下：

```
def update_mini_batch(self, mini_batch, eta):
        """用反向传播对单个小批量应用梯度下降，更新网络权重与偏置。
        "mini_batch" 是 "(x, y)" 元组列表，"eta" 是学习率。"""
        nabla_b = [np.zeros(b.shape) for b in self.biases]
        nabla_w = [np.zeros(w.shape) for w in self.weights]
        for x, y in mini_batch:
            delta_nabla_b, delta_nabla_w = self.backprop(x, y)
            nabla_b = [nb+dnb for nb, dnb in zip(nabla_b, delta_nabla_b)]
            nabla_w = [nw+dnw for nw, dnw in zip(nabla_w, delta_nabla_w)]
        self.weights = [w-(eta/len(mini_batch))*nw 
                        for w, nw in zip(self.weights, nabla_w)]
        self.biases = [b-(eta/len(mini_batch))*nb 
                       for b, nb in zip(self.biases, nabla_b)]
```

大部分工作由这行完成：

```
delta_nabla_b, delta_nabla_w = self.backprop(x, y)
```

它调用一种叫**反向传播（backpropagation）**的算法——快速计算代价函数梯度的方法。所以 `update_mini_batch` 的工作方式就是：为小批量中每个训练样本计算这些梯度，再相应更新 `self.weights` 与 `self.biases`。

我现在不展示 `self.backprop` 的代码，下一章会研究反向传播如何工作，包括 `self.backprop` 的实现。眼下只需假定它如声明般工作——为训练样本 x 对应的代价返回合适的梯度。

看完整程序（含我上面略去的文档字符串）。除 `self.backprop` 外程序不言自明——所有重活都在已讨论过的 `self.SGD` 与 `self.update_mini_batch` 里。`self.backprop` 方法用几个辅助函数计算梯度：`sigmoid_prime` 计算 $\sigma$ 函数的导数，`self.cost_derivative` 此处暂不描述。看代码与文档字符串就能大致明白（甚至细节）。我们下一章细看。注意程序看着长，其实大部分是便于理解的文档字符串。非空白非注释代码仅 74 行。全部代码见 GitHub[此处](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/network.py)。

```
"""
network.py
~~~~~~~~~~

为前馈神经网络实现随机梯度下降学习算法的模块。梯度用反向传播计算。
注意我聚焦于让代码简单、易读、易改——未做优化，且省略许多理想特性。
"""

#### 标准库
import random

#### 第三方库
import numpy as np

class Network(object):

    def __init__(self, sizes):
        """列表 ``sizes`` 含各层神经元数。例如 [2, 3, 1] 表示三层网络：
        第一层 2 神经元、第二层 3 神经元、第三层 1 神经元。网络的偏置与
        权重随机初始化——均值为 0、方差为 1 的高斯分布。注意第一层假定为
        输入层，按惯例不为那些神经元设偏置——偏置只在计算后续层输出时用到。"""
        self.num_layers = len(sizes)
        self.sizes = sizes
        self.biases = [np.random.randn(y, 1) for y in sizes[1:]]
        self.weights = [np.random.randn(y, x)
                        for x, y in zip(sizes[:-1], sizes[1:])]

    def feedforward(self, a):
        """返回网络对输入 ``a`` 的输出。"""
        for b, w in zip(self.biases, self.weights):
            a = sigmoid(np.dot(w, a)+b)
        return a

    def SGD(self, training_data, epochs, mini_batch_size, eta,
            test_data=None):
        """用小批量随机梯度下降训练神经网络。``training_data`` 是 "(x, y)"
        元组列表，表示训练输入与期望输出。若提供 ``test_data``，网络将每回合
        评测并打印进度，便于跟踪但拖慢速度。"""
        if test_data: n_test = len(test_data)
        n = len(training_data)
        for j in xrange(epochs):
            random.shuffle(training_data)
            mini_batches = [
                training_data[k:k+mini_batch_size]
                for k in xrange(0, n, mini_batch_size)]
            for mini_batch in mini_batches:
                self.update_mini_batch(mini_batch, eta)
            if test_data:
                print "Epoch {0}: {1} / {2}".format(
                    j, self.evaluate(test_data), n_test)
            else:
                print "Epoch {0} complete".format(j)

    def update_mini_batch(self, mini_batch, eta):
        """用反向传播对单个小批量施加梯度下降，更新权重与偏置。"""
        nabla_b = [np.zeros(b.shape) for b in self.biases]
        nabla_w = [np.zeros(w.shape) for w in self.weights]
        for x, y in mini_batch:
            delta_nabla_b, delta_nabla_w = self.backprop(x, y)
            nabla_b = [nb+dnb for nb, dnb in zip(nabla_b, delta_nabla_b)]
            nabla_w = [nw+dnw for nw, dnw in zip(nabla_w, delta_nabla_w)]
        self.weights = [w-(eta/len(mini_batch))*nw
                        for w, nw in zip(self.weights, nabla_w)]
        self.biases = [b-(eta/len(mini_batch))*nb
                       for b, nb in zip(self.biases, nabla_b)]

    def backprop(self, x, y):
        """返回元组 ``(nabla_b, nabla_w)``——代价函数 C_x 的梯度。
        ``nabla_b`` 与 ``nabla_w`` 是逐层的 numpy 数组列表，同
        ``self.biases`` 与 ``self.weights``。"""
        nabla_b = [np.zeros(b.shape) for b in self.biases]
        nabla_w = [np.zeros(w.shape) for w in self.weights]
        # 前向传播
        activation = x
        activations = [x]   # 逐层存储所有激活值的列表
        zs = []             # 逐层存储所有 z 向量的列表
        for b, w in zip(self.biases, self.weights):
            z = np.dot(w, activation)+b
            zs.append(z)
            activation = sigmoid(z)
            activations.append(activation)
        # 反向传播
        delta = self.cost_derivative(activations[-1], y) * \
            sigmoid_prime(zs[-1])
        nabla_b[-1] = delta
        nabla_w[-1] = np.dot(delta, activations[-2].transpose())
        # 注意下面循环中变量 l 的用法与书中第 2 章略有不同：这里 l=1 表示
        # 最后一层神经元，l=2 表示倒数第二层，依此类推。这是书中记号的重新
        # 编号，以便利用 Python 列表的负数索引。
        for l in xrange(2, self.num_layers):
            z = zs[-l]
            sp = sigmoid_prime(z)
            delta = np.dot(self.weights[-l+1].transpose(), delta) * sp
            nabla_b[-l] = delta
            nabla_w[-l] = np.dot(delta, activations[-l-1].transpose())
        return (nabla_b, nabla_w)

    def evaluate(self, test_data):
        """返回测试输入中被网络正确分类的数量。注意网络输出被认定为
        最后一层激活值最高的神经元下标。"""
        test_results = [(np.argmax(self.feedforward(x)), y)
                        for (x, y) in test_data]
        return sum(int(x == y) for (x, y) in test_results)

    def cost_derivative(self, output_activations, y):
        """返回输出激活值对代价函数 C_x 的偏导向量。"""
        return (output_activations-y)

#### 杂项函数
def sigmoid(z):
    """sigmoid 函数。"""
    return 1.0/(1.0+np.exp(-z))

def sigmoid_prime(z):
    """sigmoid 函数的导数。"""
    return sigmoid(z)*(1-sigmoid(z))
```

程序识别手写数字的效果如何？先加载 MNIST 数据。我用一个小的辅助程序 `mnist_loader.py` 来做（下面描述）。在 Python shell 中执行：

```
>>> import mnist_loader
>>> training_data, validation_data, test_data = \
... mnist_loader.load_data_wrapper()
```

当然也可以在独立 Python 程序中做；若你边读边练，在 Python shell 中最方便。

加载 MNIST 数据后，建一个含 30 个隐藏神经元的 Network。在此之前先导入上面那个叫 `network` 的 Python 程序：

```
>>> import network
>>> net = network.Network([784, 30, 10])
```

最后，用 SGD 在 MNIST 训练数据上学习 30 回合、小批量大小 10、学习率 $\eta=3.0$：

```
>>> net.SGD(training_data, 30, 10, 3.0, test_data=test_data)
```

注意若你边读边跑代码，会花些时间——对 2015 年的典型机器，可能要几分钟。建议你先让程序跑起来，继续阅读，不时查看输出。赶时间的话可减少回合数、隐藏神经元数或只用部分训练数据来加速。注意生产代码会快得多：这些 Python 脚本旨在帮你理解神经网络如何工作，而非高性能代码！当然，一旦训练好，网络几乎可在任何平台飞快运行。例如学到一组好权重偏置后，很容易移植到浏览器里的 Javascript 或移动端原生应用。下面是一次训练运行的输出节选，显示每回合后网络正确识别的测试图数。可见仅一回合后就达 9,129/10,000，且继续增长：

```
Epoch 0: 9129 / 10000
Epoch 1: 9295 / 10000
Epoch 2: 9348 / 10000
...
Epoch 27: 9528 / 10000
Epoch 28: 9542 / 10000
Epoch 29: 9534 / 10000
```

即训练后的网络分类率约 **95%**——峰值（第 28 回合）95.42%！作为首次尝试相当鼓舞人心。但提醒你：若自己跑，结果未必与我完全一样，因为我们用（不同的）随机权重与偏置初始化网络。本章结果我取了三次运行的最佳。

把上面实验的隐藏神经元数改成 100 再跑：

```
>>> net = network.Network([784, 100, 10])
>>> net.SGD(training_data, 30, 10, 3.0, test_data=test_data)
```

果然，结果提升到 **96.59%**。至少在此例中，更多隐藏神经元带来更好结果。*（读者反馈此实验结果差异颇大，有些运行效果差不少。用第 3 章的技术会大幅降低不同训练运行的性能差异。）*

当然，达到这些准确率我做了具体选择：训练回合数、小批量大小与学习率 $\eta$。如前所述，这些叫神经网络的**超参数（hyper-parameters）**——以区别于学习算法学到的参数（权重与偏置）。若超参选得差，结果会很糟。比如假设学习率选 $\eta=0.001$：

```
>>> net = network.Network([784, 100, 10])
>>> net.SGD(training_data, 30, 10, 0.001, test_data=test_data)
```

结果远没那么鼓舞人心：

```
Epoch 0: 1139 / 10000
Epoch 1: 1136 / 10000
Epoch 2: 1135 / 10000
...
Epoch 27: 2101 / 10000
Epoch 28: 2123 / 10000
Epoch 29: 2142 / 10000
```

不过你能看到网络性能随时间缓慢变好。这提示可以提高学习率，比如到 $\eta=0.01$。若如此，结果更好，又提示再提高。（若改动带来改善，就再多做一点！）这样试几次，最终学习率会到 1.0 左右（也许微调到 3.0），接近我们早先的实验。所以即便初始超参选得差，至少得到足够信息来改进选择。

一般说来，调试神经网络很有挑战。尤其当初始超参产生的结果不比随机噪声好时。假设我们试早先成功的 30 隐藏神经元架构，但学习率改为 $\eta=100.0$：

```
>>> net = network.Network([784, 30, 10])
>>> net.SGD(training_data, 30, 10, 100.0, test_data=test_data)
```

这时其实走过头了，学习率过高：

```
Epoch 0: 1009 / 10000
Epoch 1: 1009 / 10000
Epoch 2: 1009 / 10000
Epoch 3: 1009 / 10000
...
Epoch 27: 982 / 10000
Epoch 28: 982 / 10000
Epoch 29: 982 / 10000
```

现在想象我们头一回遇到这个问题。我们当然*知道*（从早先实验）该降低学习率。但若真是初次接触，输出里可没多少线索告诉我们怎么做。我们可能不仅担心学习率，还担心神经网络的其他每个方面：会不会初始化权重偏置的方式让网络难学？或者训练数据不够、学不到有意义的东西？也许回合数不够？又或者这种架构的神经网络根本学不会识别手写数字？也许是学习率太*低*？或者太高？头一回碰到问题时，你并不总是确定。

这里的教训是：调试神经网络绝非易事，正如普通编程，它是一门艺术。你需要学会这门调试艺术，才能在神经网络上获得好结果。更一般地，我们需要发展选择好超参与好架构的启发式。本书会详谈这些，包括我上面怎么选超参的。

#### 练习

- 试建一个只有两层——输入层与输出层、无隐藏层——的网络，分别为 784 与 10 神经元。用 SGD 训练。能达到多高的分类准确率？

前面我略过了 MNIST 数据加载的细节，其实相当直接。为完整起见附上代码。存储 MNIST 数据的结构在文档字符串中说明——就是些 ndarray 的元组与列表（不熟悉 ndarray 就把它想成向量）：

```
"""
mnist_loader
~~~~~~~~~~~~

加载 MNIST 图像数据的库。返回的数据结构细节见 ``load_data`` 与
``load_data_wrapper`` 的文档字符串。实践中 ``load_data_wrapper`` 是
神经网络代码通常调用的函数。
"""

#### 标准库
import cPickle
import gzip

#### 第三方库
import numpy as np

def load_data():
    """把 MNIST 数据作为元组返回，含训练数据、验证数据与测试数据。

    ``training_data`` 以含两项的元组返回。第一项是实际训练图像：numpy
    ndarray，含 50,000 项，每项是含 784 个值的 ndarray，代表单张 MNIST
    图的 28*28=784 个像素。第二项是含 50,000 项的 ndarray，内容是图中
    数字的值（0...9）。``validation_data`` 与 ``test_data`` 类似，但各
    只含 10,000 张图。这是不错的数据格式，但用于神经网络时最好改一下
    ``training_data`` 的格式，这由包装函数 ``load_data_wrapper()`` 完成。"""
    f = gzip.open('../data/mnist.pkl.gz', 'rb')
    training_data, validation_data, test_data = cPickle.load(f)
    f.close()
    return (training_data, validation_data, test_data)

def load_data_wrapper():
    """返回元组 ``(training_data, validation_data, test_data)``。基于
    ``load_data``，但格式更方便神经网络实现使用。特别地，``training_data``
    是含 50,000 个 2 元组 "(x, y)" 的列表：x 是含输入图像的 784 维
    numpy.ndarray；y 是表示正确数字 one-hot 向量的 10 维 numpy.ndarray。
    ``validation_data`` 与 ``test_data`` 是含 10,000 个 2 元组的列表，
    其中 y 是数字值（整数）而非 one-hot 向量。"""
    tr_d, va_d, te_d = load_data()
    training_inputs = [np.reshape(x, (784, 1)) for x in tr_d[0]]
    training_results = [vectorized_result(y) for y in tr_d[1]]
    training_data = zip(training_inputs, training_results)
    validation_inputs = [np.reshape(x, (784, 1)) for x in va_d[0]]
    validation_data = zip(validation_inputs, va_d[1])
    test_inputs = [np.reshape(x, (784, 1)) for x in te_d[0]]
    test_data = zip(test_inputs, te_d[1])
    return (training_data, validation_data, test_data)

def vectorized_result(j):
    """返回第 j 位为 1.0、其余为 0 的 10 维单位向量。用于把数字 (0...9)
    转成神经网络对应的期望输出。"""
    e = np.zeros((10, 1))
    e[j] = 1.0
    return e
```

上面我说程序结果相当好。这算什么意思？相对什么算好？拿些简单的（非神经网络）基线对比很有信息量，能帮我们理解「表现好」的含义。最朴素的基线当然是随机猜数字，正确率约 10%。我们比那好得多！

换个不那么平凡的基线。试个极简单的想法：看图像有多**暗**。比如数字 2 的图通常比 1 暗不少，只因更多像素被涂黑，如下例：

![](images/mnist_2_and_1.png)

这提示用训练数据计算每个数字 $0,1,2,\ldots,9$ 的平均暗度。来新图时算它的暗度，猜成平均暗度最接近的那个数字。这过程很简单、易编码，我就不显式写代码了——感兴趣的可在 [GitHub 仓库](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/mnist_average_darkness.py) 找。但比起随机猜它是大进步：10,000 测试图里对 2,225 张，即 **22.25%** 准确率。

不难找到其他能到 20–50% 的点子。再努努力能上 50%。但要高得多，就最好用成熟的机器学习算法。试试最有名的算法之一——**支持向量机（support vector machine，SVM）**。不熟 SVM 也没关系，我们不需要理解其细节，只用 Python 库 [scikit-learn](http://scikit-learn.org/stable/)——它提供对基于 C 的快速 SVM 库 [LIBSVM](http://www.csie.ntu.edu.tw/~cjlin/libsvm/) 的简单接口。

用默认设置跑 scikit-learn 的 SVM 分类器，10,000 测试图里对 9,435 张（代码见[此处](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/mnist_svm.py)）。比起按暗度分类的朴素方法是大进步，意味着 SVM 与我们的神经网络表现大致相当，只略差一点。后面章节会引入新技术，让神经网络远超 SVM。

不过故事还没完。9,435/10,000 是 scikit-learn 默认设置的 SVM 结果。SVM 有些可调参数，搜索参数可改进开箱性能。我不显式做搜索，而是请你参考 [Andreas Mueller](http://peekaboo-vision.blogspot.ca/) 的[这篇博文](http://peekaboo-vision.blogspot.de/2010/09/mnist-for-ever.html)。Mueller 展示花功夫优化 SVM 参数后，性能能到 98.5% 以上。即调好的 SVM 大约每 70 个数字才错 1 个。相当不错！神经网络能更好吗？

事实上能。目前设计良好的神经网络在 MNIST 上胜过包括 SVM 在内的一切技术。现有（2013）纪录是 10,000 张中正确分类 9,979 张，由 [Li Wan](http://www.cs.nyu.edu/~wanli/)、[Matthew Zeiler](http://www.matthewzeiler.com/)、Sixin Zhang、[Yann LeCun](http://yann.lecun.com/) 与 [Rob Fergus](http://cs.nyu.edu/~fergus/pmwiki/pmwiki.php) 完成。他们用的大部分技术我们会在后面章节见到。这个水平接近人类等效，甚至可说更好——因为不少 MNIST 图像连人类都难以自信识别，例如：

![](images/mnist_really_bad_images.png)

相信你会同意这些很难分类！MNIST 里有这类图，神经网络还能准确分类除 21 张外的全部 10,000 张测试图，实在了不起。通常编程时我们相信，解决像识别 MNIST 数字这样复杂的问题需要老练的算法。但 Wan 等人论文里的神经网络连同上文讨论的，都是相当简单的算法，是我们本章所见算法的变体。所有复杂性都是**从训练数据自动学来的**。某种意义上，我们结果与更老练论文结果的寓意是：对某些问题——

$$ \text{老练算法} \le \text{简单学习算法} + \text{好训练数据} $$

## 迈向深度学习

虽然我们的神经网络表现亮眼，这表现却有点神秘。网络里的权重与偏置是自动发现的，这意味着我们不能立即解释网络如何做到。能否找到某种方式理解网络分类手写数字的原则？若有了原则，能否做得更好？

把这些问得更尖锐些：假设几十年后神经网络催生了人工智能（AI）。我们能理解这种智能网络如何工作吗？也许网络对我们不透明——权重与偏置不明所以，因为它们是自动学来的。AI 研究早期，人们希望造 AI 的努力也能帮我们理解智能背后的原则，也许还有人类大脑的运作。但结局也许是：大脑没搞懂，人工智能怎么工作也没搞懂！

为回应这些问题，回到本章开头给人工神经元的解释——作为权衡证据的手段。假设我们要判断一张图是否为**人脸**：

Credits: 1. [Ester Inbar](http://commons.wikimedia.org/wiki/User:ST). 2. Unknown. 3. NASA, ESA, G. Illingworth, D. Magee, and P. Oesch (University of California, Santa Cruz), R. Bouwens (Leiden University), and the HUDF09 Team. 点击图片了解更多。

[![](images/Kangaroo.JPG)](http://commons.wikimedia.org/wiki/File:Kangaroo_ST_03.JPG) [![](images/Einstein_crop.jpg)](http://commons.wikimedia.org/wiki/File:Albert_Einstein_at_the_age_of_three_\(1882\).jpg) [![](images/hubble.jpg)](http://commons.wikimedia.org/wiki/File:The_Hubble_eXtreme_Deep_Field.jpg)

我们可以像对付手写识别一样对付这问题——把图像像素作为网络输入，网络输出是单个神经元，表示"是人脸"或"不是人脸"。

假设就这么做，但不用学习算法，而是**手工设计**网络，挑合适的权重与偏置。会怎么做？暂时忘掉神经网络，一个启发式是把问题拆成子问题：图左上有没有眼睛？右上有没有眼睛？中间有没有鼻子？下中有没有嘴？上面有没有头发？等等。

若其中几问答案是"是"，甚至只是"大概有"，就推断图很可能是人脸。反之大多答"否"，大概不是人脸。

当然这只是粗略启发式，有许多缺陷。也许人秃顶所以没头发；也许只看到脸的一部分，或脸有角度，部分五官被遮挡。不过这启发式提示：若能用神经网络解决子问题，或许就能组合各子问题的网络来搭人脸检测网络。下图是一种可行架构，矩形表示子网络。注意这不是处理人脸检测的现实方法，而是帮我们建立网络运作直觉。架构如下：

![](images/tikz14.png)

子网络也可再分解，这也很合理。假设考虑"左上角有没有眼睛？"可拆成"有没有眉毛？""有没有睫毛？""有没有虹膜？"等等。当然这些问法应包含位置信息——"眉毛是否在左上、且在虹膜上方？"——但先简化。回答"左上角有没有眼睛？"的网络现在可以分解：

![](images/tikz15.png)

这些问题还能继续拆，穿过更多层逐级分解。最终我们处理的是些简单到只需在单像素层面就能回答的子网络。那些问题或许关于图中特定位置有无极简形状，可由连接原始像素的单个神经元回答。

最终得到的网络把极复杂的问题——"这图是否显示人脸"——拆成能在像素级回答的极简单问题。它通过多层串联实现：浅层回答关于输入图像的简单具体问题，深层构建起越来越复杂抽象的概念层级。这种多层的网络（两个或更多隐藏层）称为**深度神经网络（deep neural networks）**。

当然，我还没说怎么自动做这种递归子网络分解。手工设计权重偏置显然不现实。相反，我们希望用学习算法让网络**自动**从训练数据学到权重偏置——从而学到概念层级。1980–90 年代研究者试过用 SGD 与反向传播训练深度网络。可惜除少数特殊架构外，运气不佳。网络会学，但极慢，实践中常慢到没用。

2006 年起，一系列让深度神经网络学习成为可能的技术被开发出来。这些深度学习技术基于 SGD 与反向传播，但引入新想法，使更深（更大）的网络得以训练——如今人们常规训练 5 到 10 个隐藏层的网络。事实证明在许多问题上，它们远比浅层网络（仅一个隐藏层）表现好。原因自然是深度网络能构建起复杂的概念层级。这有点像常规编程语言用模块化设计与抽象思想来构建复杂程序。拿深度网络对比浅层网络，有点像拿「能调用函数的语言」对比「被阉割到没法调函数的语言」。抽象在神经网络里的形式不同于常规编程，但同样重要。
