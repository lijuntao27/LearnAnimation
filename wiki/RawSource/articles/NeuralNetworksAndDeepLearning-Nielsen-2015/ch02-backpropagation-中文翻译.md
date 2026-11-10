---
type: raw-book-translation
title: "反向传播算法如何工作（第2章 中文译本）"
book: Neural Networks and Deep Learning
author: Michael A. Nielsen
source: ch02-backpropagation.md
created: 2026-09-07
tags: [deep-learning,neural-networks,book,michael-nielsen,backpropagation,translation]
summary: Nielsen《Neural Networks and Deep Learning》第2章中文学习译本：反向传播的完整推导——加权输入记号、Hadamard 积、四个基本方程(BP1-BP4)及其证明、算法伪代码与 Python 实现、大图景直觉。公式与图沿用原文，英文原章见 [[ch02-backpropagation]]。
---

# 反向传播算法如何工作（第2章 中文译本）

> [!info] 译本说明
> 本文件是 Michael A. Nielsen《Neural Networks and Deep Learning》(2015) 第 2 章的中文学习译本。本章是全书写得最重的数学章（需多元微积分与线性代数基础）；正文忠实传达原文论述，关键术语首次出现保留英文对照，公式用 LaTeX（`$...$` / `$$...$$`），方程编号与英文原章一致便于溯源，图片为本地 `images/` 相对引用。英文原章见 [[ch02-backpropagation]]。
>
> **官方阅读建议**：若数学推导读得吃力，可直接跳到本文「反向传播算法」一节的算法清单，或略读本章、把反向传播当黑盒继续读后续章节。书的后文会在少数几处引用本章结论，但那些位置即使不跟全部推理也能理解主要结论。

在[上一章](ch01-handwritten-digits-中文翻译.md)我们看到了神经网络如何用梯度下降算法学习权重与偏置。然而我们的解释有一个缺口：没讨论如何计算代价函数的梯度。这是个不小的缺口！本章我将解释一种快速计算这类梯度的算法——**反向传播（backpropagation）**。

反向传播算法最初于 1970 年代提出，但其重要性直到 [David Rumelhart](http://en.wikipedia.org/wiki/David_Rumelhart)、[Geoffrey Hinton](http://www.cs.toronto.edu/~hinton/) 与 [Ronald Williams](http://en.wikipedia.org/wiki/Ronald_J._Williams) 的一篇[1986 年著名论文](http://www.nature.com/nature/journal/v323/n6088/pdf/323533a0.pdf)才被充分认识。那篇论文描述了几种神经网络，其中反向传播远快于此前的学习方法，使神经网络得以解决以前无法解决的问题。今天，反向传播算法是神经网络学习的主力。

本章比书其余部分涉及更多数学。若你不热衷数学，可能想跳过本章，把反向传播当作细节可以忽略的黑盒。为什么要花时间研究这些细节？

原因当然是**理解**。反向传播的核心是代价函数 $C$ 对网络中任意权重 $w$（或偏置 $b$）的偏导数 $\partial C/\partial w$ 的表达式。这个表达式告诉我们改变权重与偏置时代价变化多快。虽然表达式有些复杂，它也有一种美感——每个元素都有自然、直观的解释。因此反向传播不只是快速的学习算法，它实际给了我们细致洞见：改变权重与偏置如何改变网络的整体行为。这非常值得细究。

话虽如此，若你想略读本章、或直接跳到下一章，也没问题。我把书其余部分写得即便把反向传播当黑盒也能读懂。后面书里当然有几处会引用本章结果，但在那些地方你仍能理解主要结论，即使没跟全部推理。

### 热身：一种快速基于矩阵的神经网络输出计算法

讨论反向传播前，先热身：用快速矩阵算法计算神经网络输出。我们其实在[上一章结尾](ch01-handwritten-digits-中文翻译.md#实现我们的数字分类网络)已简略见过这个算法，但当时讲得快，值得细看。在熟悉的语境里熟悉反向传播用的记号，尤其有好处。

先从能无歧义指代网络权重的记号开始。用 $w^l_{jk}$ 表示从第 $(l-1)$ 层第 $k$ 个神经元到第 $l$ 层第 $j$ 个神经元的连接权重。例如下图显示了网络中从第二层第四个神经元到第三层第二个神经元的连接权重：

![](images/tikz16.png)

这个记号起初笨拙，需要花功夫掌握；但稍加努力你会发现它变得容易而自然。记号的一个怪癖是 $j$ 与 $k$ 的顺序。你可能觉得用 $j$ 指输入神经元、$k$ 指输出神经元更合理，而不是实际这样反过来。下面我会解释这个怪癖的原因。

我们对网络偏置与激活用类似记号。明确地，用 $b^l_j$ 表示第 $l$ 层第 $j$ 个神经元的偏置，用 $a^l_j$ 表示第 $l$ 层第 $j$ 个神经元的**激活（activation）**。下图展示这些记号的使用示例：

![](images/tikz17.png)

有了这些记号，第 $l$ 层第 $j$ 个神经元的激活 $a^l_j$ 与第 $(l-1)$ 层激活的关系为（对比上一章式 (4) 及其周围讨论）：

$$
\begin{eqnarray} a^{l}_j = \sigma\left( \sum_k w^{l}_{jk} a^{l-1}_k + b^l_j \right), \tag{23}\end{eqnarray}
$$

其中求和覆盖第 $(l-1)$ 层的所有神经元 $k$。为把此式写成矩阵形式，为每层 $l$ 定义**权重矩阵（weight matrix）** $w^l$。$w^l$ 的元素就是连接到第 $l$ 层神经元的权重——第 $j$ 行第 $k$ 列的元素是 $w^l_{jk}$。类似地，每层 $l$ 定义**偏置向量（bias vector）** $b^l$。你大概能猜到——偏置向量的分量就是 $b^l_j$，第 $l$ 层每个神经元一个分量。最后定义**激活向量（activation vector）** $a^l$，其分量为激活值 $a^l_j$。
在·
把 (23) 写成矩阵形式所需的最后一块拼图，是把 $\sigma$ 这类函数**向量化（vectorizing）**的想法。上一章我们简略见过向量化，这里复述：我们想对向量 $v$ 的每个元素应用 $\sigma$ 这样的函数，用记号 $\sigma(v)$ 表示这种逐元素应用——即 $\sigma(v)$ 的分量就是 $\sigma(v)_j=\sigma(v_j)$。例如若有函数 $f(x)=x^2$，则 $f$ 的向量化形式效果为：

$$
\begin{eqnarray} f\left(\left[ \begin{array}{c} 2 \\ 3 \end{array} \right] \right) = \left[ \begin{array}{c} f(2) \\ f(3) \end{array} \right] = \left[ \begin{array}{c} 4 \\ 9 \end{array} \right], \tag{24}\end{eqnarray}
$$

即向量化的 $f$ 只是对向量每个元素平方。

心中有这些记号，式 (23) 可改写为优美紧凑的向量化形式：

$$
\begin{eqnarray} a^{l} = \sigma(w^l a^{l-1}+b^l). \tag{25}\end{eqnarray}
$$

这个表达式给了我们更全局的方式来看待一层激活与上一层激活的关系：只需把权重矩阵作用到激活上，加上偏置向量，最后应用 $\sigma$ 函数*。*（顺便说，正是这个表达式启发了前面提到的 $w^l_{jk}$ 记号怪癖。若用 $j$ 索引输入神经元、$k$ 索引输出神经元，式 (25) 里的权重矩阵就要换成它的转置。这是个小改动，但很烦人，而且我们会失去「把权重矩阵作用到激活上」这句（与这个想法）的简明性。）* 这种全局视角往往比我们此前逐神经元的视角更简单、更简洁（且涉及更少下标！）。把它当作逃离下标地狱的一种方式，同时对正在发生的事保持精确。这个表达式实践上也有用，因为多数矩阵库提供快速的矩阵乘法、向量加法与向量化实现。事实上，上一章的[代码](ch01-handwritten-digits-中文翻译.md#实现我们的数字分类网络)隐式使用了此式来计算网络行为。

用式 (25) 计算 $a^l$ 时，沿途会算中间量 $z^l\equiv w^l a^{l-1}+b^l$。这个量重要到值得命名：称 $z^l$ 为第 $l$ 层神经元的**加权输入（weighted input）**。本章后面会大量使用加权输入 $z^l$。式 (25) 有时用加权输入写成 $a^l=\sigma(z^l)$。还值得注意 $z^l$ 的分量为 $z^l_j=\sum_k w^l_{jk} a^{l-1}_k+b^l_j$，即 $z^l_j$ 正是第 $l$ 层神经元 $j$ 的激活函数的加权输入。

### 关于代价函数需要的两个假设

反向传播的目标是计算代价函数 $C$ 对网络中任意权重 $w$ 或偏置 $b$ 的偏导 $\partial C/\partial w$ 与 $\partial C/\partial b$。要让反向传播工作，需对代价函数的形式作两个主要假设。陈述假设前，先心里有个代价函数示例比较好。我们用上一章的二次代价函数（参见式 (6)）。用上一节记号，二次代价形如：

$$
\begin{eqnarray} C = \frac{1}{2n} \sum_x \|y(x)-a^L(x)\|^2, \tag{26}\end{eqnarray}
$$

其中：$n$ 是训练样本总数；求和覆盖单个训练样本 $x$；$y=y(x)$ 是对应的期望输出；$L$ 表示网络层数；$a^L=a^L(x)$ 是输入 $x$ 时网络输出的激活向量。

那么，为应用反向传播，我们需要对代价函数 $C$ 作哪些假设？**第一个假设**是代价函数可写成各训练样本代价 $C_x$ 的平均 $C=\frac{1}{n}\sum_x C_x$。二次代价满足：单个样本代价 $C_x=\frac12\|y-a^L\|^2$。本书后面遇到的所有其他代价函数也都满足此假设。

需要这个假设的原因是：反向传播实际能算的是**单个训练样本**的偏导 $\partial C_x/\partial w$ 与 $\partial C_x/\partial b$；通过对训练样本求平均，才恢复 $\partial C/\partial w$ 与 $\partial C/\partial b$。事实上，怀着此假设，我们假设训练样本 $x$ 已固定，并省去下标 $x$，把代价 $C_x$ 写作 $C$。后面会把 $x$ 加回来，但眼下它只是个最好隐去的记号累赘。

**第二个假设**是代价可写成神经网络输出的函数：

![](images/tikz18.png)

例如二次代价函数满足此要求，因为单个训练样本 $x$ 的二次代价可写成：

$$
\begin{eqnarray} C = \frac{1}{2} \|y-a^L\|^2 = \frac{1}{2} \sum_j (y_j-a^L_j)^2, \tag{27}\end{eqnarray}
$$

因而是输出激活的函数。当然，此代价函数也依赖期望输出 $y$，你或许疑惑为何不把代价也看作 $y$ 的函数。记住：输入训练样本 $x$ 固定，所以输出 $y$ 也是固定参数。特别地，它不是我们能靠改变权重与偏置来修改的东西——即不是神经网络学习的东西。因此把 $C$ 看作仅输出激活 $a^L$ 的函数是合理的，$y$ 只是帮助定义该函数的参数。

### Hadamard 积，$s\odot t$

反向传播算法基于常见的线性代数运算——向量加法、向量乘矩阵等。但其中一种运算没那么常用。设 $s$ 与 $t$ 是两个同维向量，用 $s\odot t$ 表示两向量的**逐元素（elementwise）乘积**。于是 $s\odot t$ 的分量就是 $(s\odot t)_j=s_jt_j$。例如：

$$
\begin{eqnarray} \left[\begin{array}{c} 1 \\ 2 \end{array}\right] \odot \left[\begin{array}{c} 3 \\ 4\end{array} \right] = \left[ \begin{array}{c} 1 * 3 \\ 2 * 4 \end{array} \right] = \left[ \begin{array}{c} 3 \\ 8 \end{array} \right]. \tag{28}\end{eqnarray}
$$

这种逐元素乘法有时叫 **Hadamard 积**或 **Schur 积**。我们称它 Hadamard 积。好的矩阵库通常提供 Hadamard 积的快速实现，实现反向传播时会派上用场。

### 反向传播的四个基本方程

反向传播研究的是：改变网络中的权重与偏置如何改变代价函数。归根结底，即计算偏导 $\partial C/\partial w^l_{jk}$ 与 $\partial C/\partial b^l_j$。但为计算它们，先引入一个中间量 $\delta^l_j$——第 $l$ 层第 $j$ 个神经元的**误差（error）**。反向传播将给我们计算误差 $\delta^l_j$ 的过程，然后把 $\delta^l_j$ 联系到 $\partial C/\partial w^l_{jk}$ 与 $\partial C/\partial b^l_j$。

为理解误差如何定义，想象我们神经网络里有个小恶魔：

![](images/tikz19.png)

恶魔坐在第 $l$ 层第 $j$ 个神经元处。当神经元输入进入时，恶魔捣乱：它给神经元的加权输入加一个小改动 $\Delta z^l_j$，于是神经元输出 $\sigma(z^l_j+\Delta z^l_j)$ 而非 $\sigma(z^l_j)$。这个改动传播经过网络后层，最终使总代价改变 $\frac{\partial C}{\partial z^l_j}\Delta z^l_j$。

这恶魔是只好恶魔，想帮你改进代价——即找使代价变小的 $\Delta z^l_j$。假设 $\frac{\partial C}{\partial z^l_j}$ 值很大（正或负）。则恶魔选与 $\frac{\partial C}{\partial z^l_j}$ 异号的 $\Delta z^l_j$ 就能大幅降低代价。反之若 $\frac{\partial C}{\partial z^l_j}$ 接近零，恶魔靠扰动加权输入 $z^l_j$ 几乎无法改进代价。就恶魔所能判断的，这神经元已相当接近最优*。*（当然，这只对小改动 $\Delta z^l_j$ 成立。我们假设恶魔只能做这样的小改动。）* 所以有一种启发式意义下，$\frac{\partial C}{\partial z^l_j}$ 是神经元误差的一种度量。

受这个故事启发，我们把第 $l$ 层神经元 $j$ 的误差 $\delta^l_j$ 定义为：

$$
\begin{eqnarray} \delta^l_j \equiv \frac{\partial C}{\partial z^l_j}. \tag{29}\end{eqnarray}
$$

按我们通常的约定，用 $\delta^l$ 表示第 $l$ 层关联的误差向量。反向传播将给我们逐层计算 $\delta^l$ 的方法，再把这些误差联系到真正关心的量 $\partial C/\partial w^l_{jk}$ 与 $\partial C/\partial b^l_j$。

你或许疑惑恶魔为何改加权输入 $z^l_j$。想象恶魔改输出激活 $a^l_j$、从而用 $\frac{\partial C}{\partial a^l_j}$ 作为误差度量，岂不更自然？事实上这样做与下面讨论结果相当类似，但会让反向传播的呈现更代数复杂些。所以我们坚持用 $\delta^l_j=\frac{\partial C}{\partial z^l_j}$ 作为误差度量*。*（在 MNIST 这类分类问题里，"误差"一词有时指分类失败率。例如网络正确分类 96.0% 数字，则误差是 4.0%。显然这与我们的 $\delta$ 向量含义颇为不同。实践中你不难辨别任何具体语境指的是哪个意思。）*

**进攻计划：** 反向传播围绕四个基本方程。这些方程合起来给我们计算误差 $\delta^l$ 与代价函数梯度的方法。下面陈述四个方程。但要提醒：别指望瞬间消化这些方程——这种预期只会带来失望。事实上反向传播方程如此丰富，透彻理解需要相当的时间与耐心，逐步深挖。好消息是这份耐心会有数倍回报。所以本节讨论只是开端，帮你在透彻理解方程的路上前行。

预告本章后面深入方程的方式：我会[给出方程的一个简短证明](ch02-backpropagation.md#proof_of_the_four_fundamental_equations_\(optional\))，帮助解释它们为何为真；把方程[以算法形式重述](ch02-backpropagation.md#the_backpropagation_algorithm)成伪代码，并[看伪代码如何实现为真实可运行的 Python 代码](ch02-backpropagation.md#the_code_for_backpropagation)；在[本章最后一节](ch02-backpropagation.md#backpropagation_the_big_picture)会建立反向传播方程含义的直观图景，以及人如何从零发现它们。一路我们会反复回到四个基本方程，随理解加深，这些方程会变得熟悉，甚至优美而自然。

**输出层误差 $\delta^L$ 的方程：** $\delta^L$ 的分量由下式给出：

$$
\begin{eqnarray} \delta^L_j = \frac{\partial C}{\partial a^L_j} \sigma'(z^L_j). \tag{BP1}\end{eqnarray}
$$

这是个非常自然的表达式。右边第一项 $\partial C/\partial a^L_j$ 衡量代价作为第 $j$ 个输出激活的函数变化多快。例如若 $C$ 不太依赖某个输出神经元 $j$，则 $\delta^L_j$ 会小，正如我们所料。右边第二项 $\sigma'(z^L_j)$ 衡量激活函数 $\sigma$ 在 $z^L_j$ 处变化多快。

注意 (BP1) 里一切都很容易计算。特别是计算网络行为时就算出了 $z^L_j$，算 $\sigma'(z^L_j)$ 只多一点点开销。$\partial C/\partial a^L_j$ 的具体形式当然取决于代价函数形式，但只要代价函数已知，算 $\partial C/\partial a^L_j$ 应没什么困难。例如用二次代价时 $C=\frac12\sum_j(y_j-a^L_j)^2$，于是 $\partial C/\partial a^L_j=(a_j^L-y_j)$，显然容易计算。

式 (BP1) 是 $\delta^L$ 的逐分量表达式。它本身很好，但不是反向传播想要的矩阵形式。不过把它写成矩阵形式很容易：

$$
\begin{eqnarray} \delta^L = \nabla_a C \odot \sigma'(z^L). \tag{BP1a}\end{eqnarray}
$$

这里 $\nabla_a C$ 定义为分量是偏导 $\partial C/\partial a^L_j$ 的向量。可以把 $\nabla_a C$ 想成表达 $C$ 相对输出激活的变化率。容易看出 (BP1a) 与 (BP1) 等价，因此从现在起我们用 (BP1) 同时指代两个方程。作为例子，二次代价情形下 $\nabla_a C=(a^L-y)$，于是 (BP1) 的完全矩阵化形式变为：

$$
\begin{eqnarray} \delta^L = (a^L-y) \odot \sigma'(z^L). \tag{30}\end{eqnarray}
$$

如你所见，这个表达式里一切都有漂亮的向量形式，用 Numpy 这类库很容易计算。

**用下一层误差 $\delta^{l+1}$ 表示本层误差 $\delta^l$ 的方程：** 即：

$$
\begin{eqnarray} \delta^l = ((w^{l+1})^T \delta^{l+1}) \odot \sigma'(z^l), \tag{BP2}\end{eqnarray}
$$

其中 $(w^{l+1})^T$ 是第 $(l+1)$ 层权重矩阵 $w^{l+1}$ 的转置。此式看似复杂，但每个元素都有不错的解释。假设已知第 $l+1$ 层的误差 $\delta^{l+1}$。应用转置权重矩阵 $(w^{l+1})^T$ 时，直觉上可看作把误差**向后**穿过网络移动，给出第 $l$ 层输出处误差的某种度量。然后取 Hadamard 积 $\odot\sigma'(z^l)$。这把误差向后穿过第 $l$ 层的激活函数，得到第 $l$ 层加权输入处的误差 $\delta^l$。

结合 (BP2) 与 (BP1) 可计算网络中任意层的误差 $\delta^l$：先用 (BP1) 算 $\delta^L$，再用式 (BP2) 算 $\delta^{L-1}$、$\delta^{L-2}$，如此一路反穿回网络。

**代价对网络中任意偏置变化率的方程：** 即：

$$
\begin{eqnarray} \frac{\partial C}{\partial b^l_j} = \delta^l_j. \tag{BP3}\end{eqnarray}
$$

即误差 $\delta^l_j$ **恰好等于**变化率 $\partial C/\partial b^l_j$。这是好消息，因为 (BP1) 与 (BP2) 已告诉我们怎么算 $\delta^l_j$。可把 (BP3) 简写为：

$$
\begin{eqnarray} \frac{\partial C}{\partial b} = \delta, \tag{31}\end{eqnarray}
$$

其中约定 $\delta$ 在与偏置 $b$ 相同的那个神经元上求值。

**代价对网络中任意权重变化率的方程：** 即：

$$
\begin{eqnarray} \frac{\partial C}{\partial w^l_{jk}} = a^{l-1}_k \delta^l_j. \tag{BP4}\end{eqnarray}
$$

这告诉我们如何用已会计算的量 $\delta^l$ 与 $a^{l-1}$ 来算偏导 $\partial C/\partial w^l_{jk}$。此式可改写为下标不那么密集的形式：

$$
\begin{eqnarray} \frac{\partial C}{\partial w} = a_{\rm in} \delta_{\rm out}, \tag{32}\end{eqnarray}
$$

其中约定 $a_{\rm in}$ 是权重 $w$ 输入端神经元的激活，$\delta_{\rm out}$ 是权重 $w$ 输出端神经元的误差。放大只看权重 $w$ 与其连接的两个神经元，可图示为：

![](images/tikz20.png)

式 (32) 的一个漂亮推论：当激活 $a_{\rm in}$ 很小时（$a_{\rm in}\approx0$），梯度项 $\partial C/\partial w$ 也往往很小。此时我们说该权重**学得慢**——指它在梯度下降中变化不大。换言之，(BP4) 的一个推论是：从低激活神经元输出的权重学得慢。

沿此思路还可从 (BP1)–(BP4) 得到其他洞见。先看输出层。考虑 (BP1) 里的 $\sigma'(z^L_j)$。回顾[上一章 sigmoid 函数图像](ch01-handwritten-digits-中文翻译.md#从感知机到-sigmoid)，当 $\sigma(z^L_j)$ 约为 0 或 1 时 $\sigma$ 函数变得很平。此时 $\sigma'(z^L_j)\approx0$。于是教训是：**若输出神经元处于低激活（$\approx0$）或高激活（$\approx1$），最终层的权重将学得慢**。这种情况常称输出神经元已**饱和（saturated）**，其结果是权重停止学习（或学得慢）。输出神经元的偏置也有类似说法。

对更早的层可得类似洞见。特别注意 (BP2) 里的 $\sigma'(z^l)$ 项：这意味着若神经元接近饱和，$\delta^l_j$ 很可能变小，进而任何输入到饱和神经元的权重都会学得慢*。*（若 ${w^{l+1}}^T\delta^{l+1}$ 元素大到足以补偿 $\sigma'(z^l_j)$ 的小，此推理不成立。但我说的是总体倾向。）*

总结：我们学到，**若输入神经元低激活、或输出神经元饱和（即高或低激活），权重会学得慢。**

这些观察都不太出人意料。但它们帮助改进我们对神经网络学习过程的心理模型。此外，我们可以把这套推理反着用。四个基本方程对**任何**激活函数都成立，不只是标准 sigmoid（因为——稍后会看到——证明没用 $\sigma$ 的任何特殊性质）。于是我们可以用这些方程来**设计**具有特定期望学习性质的激活函数。给你个例子感受一下：假设我们选一个（非 sigmoid）激活函数 $\sigma$，使 $\sigma'$ 恒正、且从不接近零。那就能防止普通 sigmoid 神经元饱和时出现的学习变慢。本书后面会看到对激活函数做这类修改的例子。心里记住 (BP1)–(BP4) 四个方程，有助于解释为何要尝试这类修改、以及它们能有什么影响。

![](images/tikz21.png)

#### 问题

- **反向传播方程的另一种呈现：** 我用 Hadamard 积陈述了反向传播方程（特别是 (BP1) 与 (BP2)）。若你不习惯 Hadamard 积，这种呈现可能让人不安。还有一条基于常规矩阵乘法的替代路线，部分读者可能觉得有启发性。(1) 证明 (BP1) 可改写为：

$$
\begin{eqnarray} \delta^L = \Sigma'(z^L) \nabla_a C, \tag{33}\end{eqnarray}
$$

其中 $\Sigma'(z^L)$ 是对角线元素为 $\sigma'(z^L_j)$、非对角线元素为零的方阵。注意此矩阵以常规矩阵乘法作用于 $\nabla_a C$。(2) 证明 (BP2) 可改写为：

$$
\begin{eqnarray} \delta^l = \Sigma'(z^l) (w^{l+1})^T \delta^{l+1}. \tag{34}\end{eqnarray}
$$

(3) 结合观察 (1) 与 (2)，证明：

$$
\begin{eqnarray} \delta^l = \Sigma'(z^l) (w^{l+1})^T \ldots \Sigma'(z^{L-1}) (w^L)^T \Sigma'(z^L) \nabla_a C \tag{35}\end{eqnarray}
$$

对熟悉矩阵乘法的读者，此式可能比 (BP1) 与 (BP2) 更容易理解。我聚焦 (BP1) 与 (BP2) 是因为那种方法数值实现更快。

### 四个基本方程的证明（可选）

现在证明四个基本方程 (BP1)–(BP4)。四者都是多元微积分**链式法则**的推论。若你熟悉链式法则，强烈建议先自己尝试推导，再读下去。

先证式 (BP1)——输出误差 $\delta^L$ 的表达式。回忆定义：

$$
\begin{eqnarray} \delta^L_j = \frac{\partial C}{\partial z^L_j}. \tag{36}\end{eqnarray}
$$

应用链式法则，可把上式用对输出激活的偏导重写：

$$
\begin{eqnarray} \delta^L_j = \sum_k \frac{\partial C}{\partial a^L_k} \frac{\partial a^L_k}{\partial z^L_j}, \tag{37}\end{eqnarray}
$$

其中求和覆盖输出层所有神经元 $k$。当然，第 $k$ 个神经元的输出激活 $a^L_k$ 只在 $k=j$ 时依赖第 $j$ 个神经元的加权输入 $z^L_j$。所以 $k\neq j$ 时 $\partial a^L_k/\partial z^L_j$ 为零。因此可化简上式：

$$
\begin{eqnarray} \delta^L_j = \frac{\partial C}{\partial a^L_j} \frac{\partial a^L_j}{\partial z^L_j}. \tag{38}\end{eqnarray}
$$

回忆 $a^L_j=\sigma(z^L_j)$，右边第二项可写为 $\sigma'(z^L_j)$，于是：

$$
\begin{eqnarray} \delta^L_j = \frac{\partial C}{\partial a^L_j} \sigma'(z^L_j), \tag{39}\end{eqnarray}
$$

正是 (BP1) 的分量形式。

接着证 (BP2)——用下一层误差 $\delta^{l+1}$ 表示 $\delta^l$ 的方程。为此要把 $\delta^l_j=\partial C/\partial z^l_j$ 用 $\delta^{l+1}_k=\partial C/\partial z^{l+1}_k$ 重写。用链式法则：

$$
\begin{eqnarray} \delta^l_j & = & \frac{\partial C}{\partial z^l_j} \tag{40}\\ & = & \sum_k \frac{\partial C}{\partial z^{l+1}_k} \frac{\partial z^{l+1}_k}{\partial z^l_j} \tag{41}\\ & = & \sum_k \frac{\partial z^{l+1}_k}{\partial z^l_j} \delta^{l+1}_k, \tag{42}\end{eqnarray}
$$

最后一行我们对调了右边两项并代入了 $\delta^{l+1}_k$ 的定义。为求最后一行的第一项，注意：

$$
\begin{eqnarray} z^{l+1}_k = \sum_j w^{l+1}_{kj} a^l_j +b^{l+1}_k = \sum_j w^{l+1}_{kj} \sigma(z^l_j) +b^{l+1}_k. \tag{43}\end{eqnarray}
$$

求导得：

$$
\begin{eqnarray} \frac{\partial z^{l+1}_k}{\partial z^l_j} = w^{l+1}_{kj} \sigma'(z^l_j). \tag{44}\end{eqnarray}
$$

代回 (42)：

$$
\begin{eqnarray} \delta^l_j = \sum_k w^{l+1}_{kj} \delta^{l+1}_k \sigma'(z^l_j). \tag{45}\end{eqnarray}
$$

这正是 (BP2) 的分量形式。

最后两个要证的方程是 (BP3) 与 (BP4)，同样由链式法则推出，方式与上面两个方程类似。留作练习。

#### 练习

- 证明式 (BP3) 与 (BP4)。

四个基本方程证毕。证明看似复杂，其实只是小心应用链式法则的结果。说得不那么简练些：可把反向传播看作通过系统应用多元微积分链式法则来计算代价函数梯度的方法。反向传播的全部本质就是这样——其余都是细节。


### 反向传播算法

反向传播方程给了我们计算代价函数梯度的方法。把它显式写成算法：

1. **输入 $x$：** 为输入层设置对应激活 $a^{1}$。
2. **前向传播：** 对每个 $l=2,3,\ldots,L$ 计算 $z^{l}=w^l a^{l-1}+b^l$ 与 $a^{l}=\sigma(z^{l})$。
3. **输出误差 $\delta^L$：** 计算向量 $\delta^{L}=\nabla_a C\odot\sigma'(z^L)$。
4. **反向传播误差：** 对每个 $l=L-1,L-2,\ldots,2$ 计算 $\delta^{l}=((w^{l+1})^T\delta^{l+1})\odot\sigma'(z^{l})$。
5. **输出：** 代价函数的梯度由 $\frac{\partial C}{\partial w^l_{jk}}=a^{l-1}_k\delta^l_j$ 与 $\frac{\partial C}{\partial b^l_j}=\delta^l_j$ 给出。

审视算法就能明白为何叫**反向**传播：我们从最后一层开始，**向后**计算误差向量 $\delta^l$。反穿网络可能显得奇怪，但想想反向传播的证明——向后移动是「代价是网络输出的函数」这一事实的推论。要理解代价如何随更早的权重与偏置变化，需要反复应用链式法则，从后往前穿过各层才能得到可用的表达式。

#### 练习

- **带单个被修改神经元的反向传播** 假设修改前馈网络中单个神经元，使其输出为 $f(\sum_j w_j x_j+b)$，其中 $f$ 是某个非 sigmoid 的函数。这种情况下应如何修改反向传播算法？
- **线性神经元上的反向传播** 假设全网络把通常的非线性 $\sigma$ 换成 $\sigma(z)=z$。为这种情况重写反向传播算法。

如上所述，反向传播算法计算的是**单个训练样本** $C=C_x$ 的代价梯度。实践中常把反向传播与随机梯度下降这类学习算法结合，为许多训练样本计算梯度。特别地，给定 $m$ 个样本的小批量，下面的算法基于该小批量施加一步梯度下降学习：

1. **输入一组训练样本。**
2. **对每个训练样本 $x$：** 设置对应输入激活 $a^{x,1}$，并执行以下步骤：
   - **前向传播：** 对每个 $l=2,3,\ldots,L$ 计算 $z^{x,l}=w^l a^{x,l-1}+b^l$ 与 $a^{x,l}=\sigma(z^{x,l})$。
   - **输出误差 $\delta^{x,L}$：** 计算向量 $\delta^{x,L}=\nabla_a C_x\odot\sigma'(z^{x,L})$。
   - **反向传播误差：** 对每个 $l=L-1,L-2,\ldots,2$ 计算 $\delta^{x,l}=((w^{l+1})^T\delta^{x,l+1})\odot\sigma'(z^{x,l})$。
3. **梯度下降：** 对每个 $l=L,L-1,\ldots,2$ 按规则 $w^l\rightarrow w^l-\frac{\eta}{m}\sum_x\delta^{x,l}(a^{x,l-1})^T$ 更新权重，按规则 $b^l\rightarrow b^l-\frac{\eta}{m}\sum_x\delta^{x,l}$ 更新偏置。

当然，实践中实现 SGD 还需一个生成训练样本小批量的外层循环，以及一个遍历多个训练回合的外层循环。为简洁我省略了它们。

### 反向传播的代码

在抽象层面理解了反向传播后，现在可以理解上一章实现反向传播所用的代码。回忆[那一章](ch01-handwritten-digits-中文翻译.md#实现我们的数字分类网络)的代码在 `update_mini_batch` 与 `backprop` 两个 Network 类方法里。这些方法的代码就是上面算法的直接翻译。特别地，`update_mini_batch` 方法通过计算当前小批量训练样本的梯度来更新网络权重与偏置：

```
class Network(object):
...
    def update_mini_batch(self, mini_batch, eta):
        """使用反向传播对单个小批量应用梯度下降，
        更新网络权重与偏置。"mini_batch" 是 "(x, y)" 元组列表，
        "eta" 是学习率。"""
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

大部分工作由 `delta_nabla_b, delta_nabla_w = self.backprop(x, y)` 这行完成——它用 backprop 方法算出偏导 $\partial C_x/\partial b^l_j$ 与 $\partial C_x/\partial w^l_{jk}$。backprop 方法紧跟上一节的算法。有一处小改动——用略微不同的方式索引各层。这改动为利用 Python 的一个特性：负数列表索引可从列表末尾倒数计数，例如 `l[-3]` 是列表 `l` 的倒数第三个元素。backprop 代码如下，连同几个辅助函数——用于计算 $\sigma$ 函数、其导数 $\sigma'$、以及代价函数的导数。有了这些你应能自足地读懂代码。若有卡住之处，可参考[代码的原始描述（与完整清单）](ch01-handwritten-digits-中文翻译.md#实现我们的数字分类网络)。

```
class Network(object):
...
   def backprop(self, x, y):
        """返回元组 "(nabla_b, nabla_w)"——代价函数 C_x 的梯度。
        "nabla_b" 与 "nabla_w" 是逐层的 numpy 数组列表，
        类似 "self.biases" 与 "self.weights"。"""
        nabla_b = [np.zeros(b.shape) for b in self.biases]
        nabla_w = [np.zeros(w.shape) for w in self.weights]
        # 前向传播
        activation = x
        activations = [x] # 逐层存储所有激活值的列表
        zs = [] # 逐层存储所有 z 向量的列表
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
        # 注意下面循环中变量 l 的用法与书中第 2 章记号略有不同。
        # 这里 l=1 表示最后一层神经元，l=2 是倒数第二层，依此类推。
        # 这是书中记号的重新编号，以便利用 Python 列表负数索引。
        for l in xrange(2, self.num_layers):
            z = zs[-l]
            sp = sigmoid_prime(z)
            delta = np.dot(self.weights[-l+1].transpose(), delta) * sp
            nabla_b[-l] = delta
            nabla_w[-l] = np.dot(delta, activations[-l-1].transpose())
        return (nabla_b, nabla_w)

...

    def cost_derivative(self, output_activations, y):
        """返回输出激活值对代价函数 C_x 的偏导向量。"""
        return (output_activations-y)

def sigmoid(z):
    """sigmoid 函数。"""
    return 1.0/(1.0+np.exp(-z))

def sigmoid_prime(z):
    """sigmoid 函数的导数。"""
    return sigmoid(z)*(1-sigmoid(z))
```

#### 问题

- **小批量上反向传播的完全矩阵化方法** 我们的 SGD 实现遍历小批量中的训练样本。可以修改反向传播算法，使它同时计算小批量中所有训练样本的梯度。思路是：不从单个输入向量 $x$ 开始，而是从矩阵 $X=[x_1\,x_2\,\ldots\,x_m]$ 开始，其各列是小批量中的向量。前向传播时乘以权重矩阵、加上合适的偏置矩阵、处处应用 sigmoid；反向传播也类似。请为这种反向传播方法显式写出伪代码，并修改 network.py 使用这种完全矩阵化方法。此方法的优势是充分利用现代线性代数库，因此可能比遍历小批量快不少。（例如在我笔记本上，对上一章那类 MNIST 分类问题，加速约两倍。）实践中所有正经的反向传播库都用这种完全矩阵化方法或其变体。

### 反向传播在何种意义上是一种快速算法？

反向传播在何种意义上是快速算法？为回答这个问题，考虑另一种计算梯度的方法。想象神经网络研究的早期——也许是 1950、60 年代，你是世上第一个想到用梯度下降学习的人！但要让这想法成立，你需要计算代价函数梯度的方法。回想你的微积分知识，决定试试能否用链式法则算梯度。但摆弄一阵后代数看着复杂，你气馁了。于是试着找别的办法。你把代价看作仅权重的函数 $C=C(w)$（偏置稍后再说），给权重编号 $w_1,w_2,\ldots$，想对某个特定权重 $w_j$ 算 $\partial C/\partial w_j$。一个显然的办法是用近似：

$$
\begin{eqnarray} \frac{\partial C}{\partial w_{j}} \approx \frac{C(w+\epsilon e_j)-C(w)}{\epsilon}, \tag{46}\end{eqnarray}
$$

其中 $\epsilon>0$ 是很小的正数，$e_j$ 是第 $j$ 个方向的单位向量。换言之，可通过对 $w_j$ 的两个略不同值各算一次代价 $C$，再套式 (46) 来估计 $\partial C/\partial w_j$。同样想法也能算对偏置的偏导 $\partial C/\partial b$。

这个方法看起来很有前途：概念简单，实现极易，只要几行代码。当然看起来比用链式法则算梯度有前途得多！

不幸的是，虽然此法看似有前途，真正实现代码时却极其缓慢。为理解原因，假设网络有一百万个权重。那么对每个不同的权重 $w_j$，为算 $\partial C/\partial w_j$ 都需计算 $C(w+\epsilon e_j)$。这意味着算梯度需把代价函数算一百万次——每个训练样本都要一百万次网络前向传播。还需要算 $C(w)$ 本身，所以总共一百万零一次网络前向传播。

反向传播的聪明之处在于：它只需**一次前向传播加一次反向传播**就能同时算出**所有**偏导 $\partial C/\partial w_j$。粗略地说，反向传播的计算代价与前向传播相当*。*（这应当可信，但要说得严谨需一些分析。可信是因为前向传播的主导计算代价是乘以权重矩阵，反向传播则是乘以权重矩阵的转置，二者计算代价显然相近。）* 于是反向传播的总代价大致等于只做两次网络前向传播。对比基于式 (46) 的方法需要一百万零一次前向传播！所以即便反向传播表面看比 (46) 法复杂，实际却快得多得多。

这个加速 1986 年首次被充分认识，极大扩展了神经网络能解决的问题范围，进而引发人们蜂拥使用神经网络。当然反向传播不是万灵药。即便 1980 年代末人们也撞上极限，尤其是试图用反向传播训练**深度**神经网络——即含许多隐藏层的网络——的时候。本书后面会看到现代计算机与一些巧妙新想法如今如何使训练这类深度神经网络成为可能。

### 反向传播：大图景

如我所述，反向传播带来两个谜团。第一，这算法到底在做什么？我们已经建立了误差从输出被反向传播的图景。但能否更深一层，为所有这些矩阵与向量乘法建立更多直觉？第二个谜团是人当初怎么可能发现反向传播？跟随算法步骤、甚至跟随算法有效的证明是一回事；但那不意味着你把问题理解到足以**最初自己发现**算法的程度。是否存在一条合情合理的推理链能引导你发现反向传播算法？本节我回应这两个谜团。

为改进算法直觉，假设我们对网络中某权重 $w^l_{jk}$ 做了小改动 $\Delta w^l_{jk}$：

![](images/tikz22.png)

该权重改动会引起对应神经元输出激活的变化：

![](images/tikz23.png)

那又会引起下一层**所有**激活的变化：

![](images/tikz24.png)

这些变化又引起再下一层的变化，一路直到最终层，然后到代价函数：

![](images/tikz25.png)

代价的变化 $\Delta C$ 与权重的变化 $\Delta w^l_{jk}$ 通过下式关联：

$$
\begin{eqnarray} \Delta C \approx \frac{\partial C}{\partial w^l_{jk}} \Delta w^l_{jk}. \tag{47}\end{eqnarray}
$$

这提示计算 $\frac{\partial C}{\partial w^l_{jk}}$ 的一个可能途径：仔细追踪 $w^l_{jk}$ 的小变化如何传播引起 $C$ 的小变化。若能如此，沿途小心用易计算的量表达一切，就能算出 $\partial C/\partial w^l_{jk}$。

让我们尝试实行。$\Delta w^l_{jk}$ 引起第 $l$ 层第 $j$ 个神经元激活的小变化 $\Delta a^l_j$。这个变化由下式给出：

$$
\begin{eqnarray} \Delta a^l_j \approx \frac{\partial a^l_j}{\partial w^l_{jk}} \Delta w^l_{jk}. \tag{48}\end{eqnarray}
$$

激活变化 $\Delta a^l_j$ 又会引起下一层（第 $(l+1)$ 层）**所有**激活的变化。我们先只看其中单个激活如何受影响，比如 $a^{l+1}_q$：

![](images/tikz26.png)

事实上它会引起如下变化：

$$
\begin{eqnarray} \Delta a^{l+1}_q \approx \frac{\partial a^{l+1}_q}{\partial a^l_j} \Delta a^l_j. \tag{49}\end{eqnarray}
$$

代入式 (48) 的表达式得：

$$
\begin{eqnarray} \Delta a^{l+1}_q \approx \frac{\partial a^{l+1}_q}{\partial a^l_j} \frac{\partial a^l_j}{\partial w^l_{jk}} \Delta w^l_{jk}. \tag{50}\end{eqnarray}
$$

当然，$\Delta a^{l+1}_q$ 又会引起下一层激活的变化。事实上可以想象一条从 $w^l_{jk}$ 一路穿到 $C$ 的路径，每个激活变化引起下一个激活变化，最终在输出端引起代价变化。若路径经过激活 $a^l_j,a^{l+1}_q,\ldots,a^{L-1}_n,a^L_m$，则结果表达式为：

$$
\begin{eqnarray} \Delta C \approx \frac{\partial C}{\partial a^L_m} \frac{\partial a^L_m}{\partial a^{L-1}_n} \frac{\partial a^{L-1}_n}{\partial a^{L-2}_p} \ldots \frac{\partial a^{l+1}_q}{\partial a^l_j} \frac{\partial a^l_j}{\partial w^l_{jk}} \Delta w^l_{jk}, \tag{51}\end{eqnarray}
$$

即每多穿过一个神经元就累积一个 $\partial a/\partial a$ 型因子，末尾还有 $\partial C/\partial a^L_m$ 项。这表示 $C$ 因激活沿这条网络特定路径变化而产生的变化。当然，$w^l_{jk}$ 的变化可通过**许多**路径传播影响代价，而我们只考虑了一条。要算 $C$ 的总变化，合情合理的做法是对权重与最终代价之间的**所有可能路径**求和：

$$
\begin{eqnarray} \Delta C \approx \sum_{mnp\ldots q} \frac{\partial C}{\partial a^L_m} \frac{\partial a^L_m}{\partial a^{L-1}_n} \frac{\partial a^{L-1}_n}{\partial a^{L-2}_p} \ldots \frac{\partial a^{l+1}_q}{\partial a^l_j} \frac{\partial a^l_j}{\partial w^l_{jk}} \Delta w^l_{jk}, \tag{52}\end{eqnarray}
$$

其中对路径上中间神经元的所有可能选择求和。与 (47) 比较得：

$$
\begin{eqnarray} \frac{\partial C}{\partial w^l_{jk}} = \sum_{mnp\ldots q} \frac{\partial C}{\partial a^L_m} \frac{\partial a^L_m}{\partial a^{L-1}_n} \frac{\partial a^{L-1}_n}{\partial a^{L-2}_p} \ldots \frac{\partial a^{l+1}_q}{\partial a^l_j} \frac{\partial a^l_j}{\partial w^l_{jk}}. \tag{53}\end{eqnarray}
$$

式 (53) 看起来很复杂。但它有漂亮的直观解释。我们在计算 $C$ 对网络中某权重变化率。方程告诉我们：网络中两个神经元之间的每条边都关联一个**速率因子**——一个神经元激活对另一个神经元激活的偏导。从初始权重到第一个神经元的边速率因子为 $\partial a^l_j/\partial w^l_{jk}$。一条路径的速率因子就是路径上各速率因子的乘积。而总变化率 $\partial C/\partial w^l_{jk}$ 就是初始权重到最终代价之间所有路径速率因子之和。此过程对单条路径图示如下：

![](images/tikz27.png)

至此我提供的是启发式论证——一种思考「扰动网络权重时发生什么」的方式。让我勾勒一条你可用来进一步发展此论证的思路。首先，你可推出式 (53) 中所有单个偏导的显式表达式——用点微积分很容易做到。做完后，可试着把所有下标求和写成矩阵乘法。这会很繁琐、需要些毅力，但不需非凡洞见。全部做完并尽可能化简后，你会发现最终得到的**恰好是反向传播算法**！于是你可以把反向传播算法看作提供了一种计算所有这些路径速率因子之和的方法。或者换个说法，反向传播算法是一种巧妙的方式，追踪权重（与偏置）的小扰动如何穿过网络、到达输出、然后影响代价。

这里我不会完整走一遍——它很乱，需相当细心处理所有细节。你若想接受挑战，也许会乐在其中。即便不想，我也希望这条思路能让你对反向传播在做些什么有所洞见。

另一个谜团呢——人最初怎么可能发现反向传播？事实上，若你跟随我刚勾勒的方法，会得到一个反向传播的证明。可惜这个证明比本章前面那个长得多也复杂得多。那么那个简短（但更神秘）的证明是怎么被发现的？你把长证明的细节全部写出时会发现，事后看有若干明显的化简扑面而来。你做这些化简，得到一个更短的证明，写出来。然后又有几个明显化简跳出来，于是再来一轮。反复几次后得到的就是我们前面看到的证明*。*（有一个巧妙步骤不可或缺。式 (53) 中中间变量是 $a_q^{l+1}$ 这类激活。巧妙的想法是把中间变量换成加权输入 $z^{l+1}_q$ 之类。若没有这个想法、继续用激活 $a^{l+1}_q$，得到的证明会比本章前面那个略复杂。）* ——短，但有点晦涩，因为通往其构造的所有路标都被移除了！当然，我是要你相信我，但前面那个证明的起源真没什么大秘密，不过是把本节勾勒的证明大量化简的苦功罢了。
