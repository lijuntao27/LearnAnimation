---
type: raw-book-translation
title: "深度学习（第6章 中文译本）"
book: Neural Networks and Deep Learning
author: Michael A. Nielsen
source: ch06-deep-learning.md
created: 2026-09-07
tags: [deep-learning,neural-networks,book,michael-nielsen,cnn,translation]
summary: Nielsen《Neural Networks and Deep Learning》第6章中文学习译本：深度卷积网络（CNN）原理（局部感受野/共享权重/池化）与 MNIST 实战（98.78%→99.67%），network3.py 完整代码，图像识别近年进展，RNN/LSTM/DBN 简述与展望。英文原章见 [[ch06-deep-learning]]。
---

# 深度学习（第6章 中文译本）

> [!info] 译本说明
> 本文件是 Michael A. Nielsen《Neural Networks and Deep Learning》(2015) 第 6 章的中文学习译本。正文忠实传达原文论述，关键术语保留英文对照；公式用 LaTeX，方程编号与英文原章一致；图片为本地 `images/` 相对引用；Python/Theano 代码保持原样、注释与文档字符串中译。英文原章见 [[ch06-deep-learning]]。
>
> 结构提示：本章很长，各节松散耦合，可跳读。代码基于 Theano 库（2015 时代），重在理解结构而非运行。

在[上一章](ch05-deep-networks-hard-to-train-中文翻译.md)我们学到深度神经网络往往比浅层网络难训得多。这很遗憾，因为我们有充分理由相信*若*能训练深度网络，它们会比浅层强大得多。但上章的消息虽令人沮丧，我们不会让它挡住去路。本章我们将发展可用于训练深度网络的技术并付诸实践；也看更广的图景，简要回顾深度网络在图像识别、语音识别与其他应用的近期进展；并对神经网络与人工智能的未来做些简短、思辨性的展望。

本章很长。为助你导航，先导览。各节只松散耦合，只要有神经网络基本熟悉，可跳到最感兴趣处。

本章[主体](#convolutional_networks)介绍应用最广的深度网络类型之一：**深度卷积网络**。我们会详细走一个例子——代码与全部——用卷积网络解决 MNIST 手写数字分类：

![](images/digits.png)

我们从本书早先用来攻此问题的浅层网络开始，经过多次迭代搭出越来越强大的网络。一路上探索许多强大技术：卷积、池化、用 GPU 做远超浅层网络的训练、算法扩展训练数据（减少过拟合）、dropout 技术（也减少过拟合）、网络集成（ensemble）等。结果是一个接近人类性能的系统：10,000 张 MNIST 测试图（训练时没见过的图！）中，系统正确分类 **9,967** 张。下面是 33 张被误分类的图——注意正确分类在右上角、我们程序的分类在右下角：

![](images/ensemble_errors.png)

其中许多连人类都难分类。例如顶行第三张：在我看来它更像"9"而非官方分类的"8"，我们的网络也认为是"9"。这种"错误"至少可以理解，甚至也许值得称赞。我们以对近期[图像识别惊人进展](#recent_progress_in_image_recognition)（用网络尤其是卷积网络做图像识别）的综述结束图像识别讨论。

本章其余部分从更广、更不细致的视角讨论深度学习。我们[简略综述其他神经网络模型](#things_we_didn't_cover_but_which_you'll_eventually_want_to_know)——如循环神经网络与长短时记忆单元，以及这类模型如何应用于语音识别、自然语言处理等领域；并[思辨神经网络与深度学习的未来](#on_the_future_of_neural_networks)——从意图驱动界面到深度学习在 AI 中的角色。

本章建立在前面章节之上，使用并整合反向传播、正则化、softmax 函数等想法。但读本章不必详细读过前面所有章节——读过[第 1 章](ch01-handwritten-digits-中文翻译.md)（神经网络基础）会有帮助；用到第 2 至 5 章概念时我给链接供必要时熟悉。

值得说明本章不是什么。它不是最新最棒神经网络库的教程，也不会训练几十层的深度网络去解决最前沿问题。重点是理解深度神经网络背后的一些核心原理、并在 MNIST 这个简单易懂的语境里应用。换言之：本章不会把你带到最前沿，而是与前面章节一样聚焦基础、为你理解当前广泛工作做准备。

### 介绍卷积网络

前面章节我们教会神经网络相当好地识别手写数字图像：

![](images/digits.png)

用的是相邻网络层**全连接**的网络——每个神经元都连到相邻层每个神经元：

![](images/tikz41.png)

特别是，对输入图像每个像素，我们把像素强度编码为输入层对应神经元的值。对我们一直用的 $28\times28$ 像素图，意味着网络有 $784(=28\times28)$ 个输入神经元。然后训练网络权重与偏置，让网络输出——我们希望！——正确识别输入图：'0','1','2',...,'8','9'。

我们早先的网络工作得相当好：用 [MNIST 手写数字集](ch01-handwritten-digits-中文翻译.md#_用梯度下降学习)的训练与测试数据[得到超过 98% 的分类准确率](ch03-improving-learning-中文翻译.md#_用交叉熵分类-mnist-数字)。但细想，用全连接层网络分类图像很奇怪——这种网络架构没有考虑图像的空间结构。例如它对相距远与相距近的输入像素一视同仁。这种空间结构概念必须改从训练数据推断。但与其用*白板（tabula rasa）*式网络架构，何不用尝试利用空间结构的架构？本节我描述*卷积神经网络（convolutional neural networks）**。*（卷积神经网络的起源可追溯至 1970 年代。但奠定现代卷积网络主题的开创性论文是 Yann LeCun、Léon Bottou、Yoshua Bengio 与 Patrick Haffner 1998 年的 ["Gradient-based learning applied to document recognition"](http://yann.lecun.com/exdb/publis/pdf/lecun-98.pdf)。LeCun 后来对卷积网络术语做过[有趣评论](https://www.facebook.com/yann.lecun/posts/10152348155137143)："卷积网络这类模型中\[生物\]神经启发非常微弱。所以我叫它们'卷积网络'而非'卷积神经网络'，我们把节点叫'单元'而非'神经元'。"尽管有此评论，卷积网络使用许多与我们研究的神经网络相同的想法——反向传播、梯度下降、正则化、非线性激活函数等——所以我们遵循惯例、视其为一种神经网络。我将互换使用"卷积神经网络"与"卷积网络"，也互换使用"\[人工\]神经元"与"单元"。）*这些网络用特别适合分类图像的特殊架构，使卷积网络训练快，进而帮我们训练非常擅长分类图像的深层多层网络。如今，深度卷积网络或其近亲变体用于多数图像识别神经网络。

卷积神经网络用三个基本想法：*局部感受野（local receptive fields）*、*共享权重（shared weights）*与*池化（pooling）*。逐一来看。

**局部感受野：** 早先所示的全连接层里，输入画成垂直一列神经元。卷积网络里，把输入想成 $28\times28$ 神经元方阵更有帮助——其值对应我们用作输入的 $28\times28$ 像素强度：

![](images/tikz42.png)

照例，把输入像素连到一层隐藏神经元。但不把每个输入像素连到每个隐藏神经元，而只在输入图像的小局部区域做连接。

更精确地，第一隐藏层每个神经元只连到输入神经元的一小区域——比如 $5\times5$ 区域，对应 25 个输入像素。于是对某特定隐藏神经元，连接可能长这样：

![](images/tikz43.png)

输入图像中那区域叫该隐藏神经元的*局部感受野*——输入像素上的小窗口。每条连接学一个权重；隐藏神经元也学整体偏置。可把该特定隐藏神经元想成在学分析它的特定局部感受野。

然后让局部感受野滑过整个输入图像。每个局部感受野对应第一隐藏层一个不同隐藏神经元。具体说明：从左上角一个局部感受野开始：

![](images/tikz44.png)

再把局部感受野向右滑一个像素（即一个神经元），连到第二个隐藏神经元：

![](images/tikz45.png)

如此继续，搭起第一隐藏层。注意：若输入图像 $28\times28$、局部感受野 $5\times5$，则隐藏层有 $24\times24$ 个神经元——因为局部感受野只能横向（或纵向）移动 23 个神经元就会撞上输入图像右侧（或底部）。

我演示的是局部感受野每次移动一个像素。事实上有时用不同*步长（stride length）*。例如局部感受野每次向右（或向下）移 2 像素，就说用步长 2。本章主要用步长 1，但值得知道人们有时实验不同步长*。*（如前面章节做的，若想试不同步长可用验证数据挑性能最好的步长。详见[前面讨论](ch03-improving-learning-中文翻译.md#如何选择神经网络的超参数)。同样方法也可用来选局部感受野大小——当然用 $5\times5$ 感受野没什么特殊，一般输入图明显大于 MNIST 的 $28\times28$ 时较大感受野有帮助。）*

**共享权重与偏置：** 我说过每个隐藏神经元有偏置与连到局部感受野的 $5\times5$ 权重。还没说的是：我们要对 $24\times24$ 个隐藏神经元都用*相同*权重与偏置。换言之，对第 $j,k$ 个隐藏神经元，输出是：

$$
\begin{eqnarray} \sigma\left(b + \sum_{l=0}^4 \sum_{m=0}^4 w_{l,m} a_{j+l, k+m} \right). \tag{125}\end{eqnarray}
$$

$\sigma$ 是神经激活函数——也许是前面章节用的 [sigmoid 函数](ch01-handwritten-digits-中文翻译.md#sigmoid-神经元)。$b$ 是共享偏置值，$w_{l,m}$ 是 $5\times5$ 共享权重数组。最后用 $a_{x,y}$ 表示位置 $x,y$ 的输入激活。

这意味着第一隐藏层所有神经元检测*完全相同的特征*，只是在不同输入位置*。*（我没精确定义"特征"。非正式地，把隐藏神经元检测的特征想成会让神经元激活的输入模式类型：也许是图像里的边，或其他某种形状。）*为看清为何合理，假设权重偏置使隐藏神经元能挑出（比如）某局部感受野里的垂直边。这能力在图像其他地方多半也有用，所以在图像各处应用相同特征检测器很有用。更抽象地说，卷积网络很适合图像的平移不变性：把猫图挪一点，仍是猫图*。*（事实上对我们研究的 MNIST 分类问题，图像居中且尺寸归一，所以 MNIST 的平移不变性比"野生"图像少。不过边与角这类特征很可能在输入空间大部分地方有用。）*

因此有时把输入层到隐藏层的映射叫*特征映射（feature map）*。把定义特征映射的权重叫*共享权重*，偏置叫*共享偏置*。共享权重与偏置常被说成定义*核（kernel）*或*滤波器（filter）*。文献里人们用这些词的方式略有不同，所以我不更精确；稍后看具体例子。

至今描述的网络结构只能检测一种局部特征。做图像识别需要不止一个特征映射，所以完整卷积层由几个不同特征映射组成：

![](images/tikz46.png)

示例中有 3 个特征映射，每个由一组 $5\times5$ 共享权重与单个共享偏置定义。结果网络能检测 3 种不同特征，每种都能跨整幅图像检测。

我只画 3 个特征映射以保持上图简单。实践中卷积网络会用更多（也许多得多）特征映射。早期卷积网络之一 LeNet-5 用 6 个特征映射（各关联 $5\times5$ 局部感受野）识别 MNIST 数字，所以上面示例其实很接近 LeNet-5。本章后面开发的例子用 20 与 40 个特征映射的卷积层。快速看一眼学到的一些特征*。*（图示特征映射来自我们训练的最后卷积网络，见[此处](#final_conv)。）*：

![](images/net_full_layer_0.png)

20 张图对应 20 个不同特征映射（或滤波器/核）。每张映射表示成 $5\times5$ 块图，对应局部感受野中的 $5\times5$ 权重。白块 = 较小（通常更负）权重，特征映射对相应输入像素响应弱；暗块 = 较大权重，响应强。粗略说，上图显示卷积层响应的特征类型。

从这些特征映射能得出什么？显然这里有超出随机的空间结构：许多特征有明暗清晰子区域，说明网络确实在学与空间结构相关的东西。但除此之外，很难看出这些特征检测器在学什么——当然没在学（比如）许多传统图像识别方法用的 [Gabor 滤波器](http://en.wikipedia.org/wiki/Gabor_filter)。事实上现在有大量工作研究更好理解卷积网络学到的特征。想跟进的话，建议从 Matthew Zeiler 与 Rob Fergus (2013) 的 [Visualizing and Understanding Convolutional Networks](http://arxiv.org/abs/1311.2901) 开始。

共享权重与偏置的一大好处：大幅减少卷积网络涉及的参数。每个特征映射需 $25=5\times5$ 个共享权重加单个共享偏置，即每特征映射 26 参数。20 个特征映射共 $20\times26=520$ 参数定义卷积层。对比：假设全连接第一层——$784=28\times28$ 输入神经元、相当适中的 30 隐藏神经元（如本书早先许多例子）——共 $784\times30$ 权重加 30 偏置，共 23,550 参数。换言之，全连接层参数是卷积层的 40 多倍。

当然不能真直接比较参数数量，因为两模型本质上不同。但直觉上，卷积层用平移不变性很可能减少"达到全连接模型同等性能所需"的参数。进而让卷积模型训练更快，最终帮我们用卷积层搭深度网络。

顺便，"卷积（convolutional）"之名来自式 (125) 的操作有时称*卷积（convolution）*。更精确些，人们有时把该式写成 $a^1=\sigma(b+w*a^0)$，其中 $a^1$ 表示一个特征映射的输出激活集、$a^0$ 是输入激活集、$*$ 叫卷积操作。我们不会深入用卷积数学，不必太担心这联系，但至少值得知道名字从哪来。

**池化层：** 除刚描述的卷积层，卷积神经网络还含*池化层（pooling layers）*。池化层通常紧接在卷积层后，作用是简化卷积层输出里的信息。

细节上，池化层取卷积层输出的每个特征映射*。*（这里命名用得较随意：我用"特征映射"不是指卷积层计算的函数，而是指该层输出的隐藏神经元激活。这种轻微滥用命名在文献里相当常见。）*并准备一个浓缩的特征映射。例如池化层每个单元可概括上一层（如 $2\times2$ 神经元）的区域。具体例子：一种常见池化程序叫*最大池化（max-pooling）*——池化单元直接输出 $2\times2$ 输入区域的最大激活，如下图：

![](images/tikz47.png)

注意卷积层输出 $24\times24$ 神经元，池化后得到 $12\times12$ 神经元。

如上所述，卷积层通常不止单个特征映射。我们对每个特征映射分别应用最大池化。若有 3 个特征映射，卷积 + 最大池化层组合起来长这样：

![](images/tikz48.png)

可把最大池化想成网络在问"某特征是否出现在图像某区域的任何地方"，然后丢掉精确位置信息。直觉：一旦找到某特征，其精确位置不如它相对其他特征的粗略位置重要。大好处是池化后特征少得多，帮助减少后面层所需参数。

最大池化不是唯一池化技术。另一种常见方法叫 *L2 池化*：不取 $2\times2$ 神经元区域的最大激活，而取区域激活平方和的平方根。细节不同但直觉与最大池化类似——浓缩卷积层信息。实践两种技术都被广泛使用，有时也用其他池化操作。真想优化性能可用验证数据比较几种池化方法、选最佳，但我们不操心那种细节优化。

**整合：** 现在把所有想法整合成完整卷积神经网络。它类似我们刚看的架构，但加了 10 个输出神经元的输出层，对应 MNIST 数字的 10 种值（'0','1','2' 等）：

![](images/tikz49.png)

网络从 $28\times28$ 输入神经元开始，编码 MNIST 图像像素强度；接着用 $5\times5$ 局部感受野与 3 特征映射的卷积层，得到 $3\times24\times24$ 隐藏特征神经元；下一步是对 3 个特征映射各做 $2\times2$ 区域最大池化，得到 $3\times12\times12$ 隐藏特征神经元。

网络最后一层连接是全连接层——把最大池化层*每个*神经元连到 10 个输出神经元每个。这种全连接架构与我们前面章节用的一样。注意上图我为简洁只画一个箭头而非全部连接，你当然容易想象连接。

这种卷积架构与前面章节架构颇不同，但整体图景相似：由许多简单单元组成的网络，行为由权重偏置决定；总体目标仍相同——用训练数据训练权重偏置，让网络做好输入数字分类。

特别是，如本书前面，我们用 SGD 与反向传播训练网络，过程与前面章节基本相同。但需对反向传播程序做几处修改——因为我们早先的[反向传播推导](ch02-backpropagation-中文翻译.md)是针对全连接层的网络。幸运的是，为卷积与最大池化层修改推导很直接。想理解细节，请做下面问题。警告：除非真正内化了[早先反向传播推导](ch02-backpropagation-中文翻译.md)（那样则容易），这问题要花些时间。

#### 问题

- **卷积网络中的反向传播** 全连接层网络反向传播的核心方程是 (BP1)–(BP4)（[链接](ch02-backpropagation-中文翻译.md#_输出层误差)。假设网络含卷积层、最大池化层、全连接输出层（如上面讨论的网络）。反向传播方程如何修改？

### 卷积网络实践

我们已看过卷积神经网络的核心想法。现在通过实现一些卷积网络、应用于 MNIST 分类问题，看它们实践中如何工作。用来做这的程序叫 network3.py，是前面章节 network.py 与 network2.py 的改进版*。*（注意 network3.py 也吸收了 Theano 库卷积神经网络文档的想法——尤其 [LeNet-5](http://deeplearning.net/tutorial/lenet.html) 的实现——以及 Misha Denil 的 [dropout 实现](https://github.com/mdenil/dropout) 与 [Chris Olah](http://colah.github.io) 的想法。）*想跟着做的话代码在 [GitHub](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/network3.py)。注意我们下一节才走 network3.py 自己的代码；本节把 network3.py 当库来搭卷积网络。

network.py 与 network2.py 用 Python 与矩阵库 Numpy 实现，从第一性原理深入反向传播、SGD 等细节。但如今我们理解那些细节，network3.py 改用机器学习库 **Theano** *。*（James Bergstra、Olivier Breuleux、Frederic Bastien、Pascal Lamblin、Razvan Pascanu、Guillaume Desjardins、Joseph Turian、David Warde-Farley 与 Yoshua Bengio (2010) 的 [Theano: A CPU and GPU Math Expression Compiler in Python](http://www.iro.umontreal.ca/~lisa/pointeurs/theano_scipy2010.pdf)。Theano 也是流行 [Pylearn2](http://deeplearning.net/software/pylearn2/) 与 [Keras](http://keras.io/) 库的基础。当时其他流行神经网络库还有 [Caffe](http://caffe.berkeleyvision.org) 与 [Torch](http://torch.ch)。）*用 Theano 让卷积神经网络实现反向传播变容易，因为它自动计算涉及的全部映射；Theano 也比早先代码快不少（早先代码为易懂而非为快而写），使训练更复杂网络变得实际。尤其 Theano 一大特性是能在 CPU 或（若有）GPU 上跑代码——GPU 运行提供大幅加速，也让训练复杂网络更实际。

想跟着做，需让 Theano 在系统上跑起来。安装按项目[主页](http://deeplearning.net/software/theano/)说明。下面例子用 Theano 0.6 运行*。*（我发布本章时 Theano 当前版已到 0.7。我实际在 Theano 0.7 下重跑过例子，结果与文中报告极相似。）*；有些在 Mac OS X Yosemite（无 GPU）跑、有些在 Ubuntu 14.04（NVIDIA GPU）跑、有些实验两种都跑。要让 network3.py 跑起来，需在源码把 GPU 标志设为 True 或 False（视情况）。此外让 Theano 在 GPU 运行可参考[这里说明](http://deeplearning.net/software/theano/tutorial/using_gpu.html)，网上也有教程（Google 易找）。本地无 GPU 可考虑 [Amazon Web Services](http://aws.amazon.com/ec2/instance-types/) EC2 G2 spot 实例。注意即便有 GPU 代码也要跑些时间——许多实验从几分钟到几小时；CPU 上最复杂实验可能跑几天。如前面章节建议，让程序跑着继续读、不时回看输出。用 CPU 的话可为较复杂实验减少训练回合或干脆省略。

为得基线，先从单隐藏层 100 隐藏神经元的浅层架构开始。训练 60 回合、学习率 $\eta=0.1$、mini-batch 大小 10、无正则化。开始*。*（本节实验代码见[此脚本](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/conv.py)，脚本代码只是复制并平行本节讨论。另注意全节我显式指定训练回合数——为清晰说明如何训练；实践值得用[提前停止](ch03-improving-learning-中文翻译.md#_用提前停止确定训练回合数)——跟踪验证集准确率、确信验证准确率停止提高时停止训练。）*

我得到最佳分类准确率 **97.80%**——这是在 validation_data 上取得最佳分类准确率的训练回合、对 test_data 评估的分类准确率。用验证数据决定何时评估测试准确率有助避免过拟合到测试数据（见[前面讨论](ch03-improving-learning-中文翻译.md#_为何用验证数据而非测试数据)）。下面都遵循此做法。你的结果可能略异，因为网络权重偏置随机初始化*。*（事实上此实验我做了三次独立运行训练此架构网络，报告与三次中最佳验证准确率对应的测试准确率。多次运行减少结果差异，比较许多架构时有用。下面我都遵循此做法，另有注明除外。实践中这对所得结果影响很小。）*

97.80% 接近[第 3 章](ch03-improving-learning-中文翻译.md#_用交叉熵分类-mnist-数字)用类似架构与超参得的 98.04%。两例都用含 100 隐藏神经元的单隐藏层浅层网络、都训 60 回合、mini-batch 10、学习率 $\eta=0.1$。

但早先网络有两处不同。第一，我们[正则化](ch03-improving-learning-中文翻译.md#_过拟合与正则化)了早先网络减少过拟合。正则化当前网络确实提高准确率，但增益小，所以先不操心正则化。第二，早先网络最后一层用 sigmoid 激活 + 交叉熵代价，当前网络用 softmax 末层 + 对数似然代价。如[第 3 章](ch03-improving-learning-中文翻译.md#softmax)所释这不是大改动。我没特别深刻理由做此切换——主要是 softmax + 对数似然代价在现代图像分类网络更常见。

用更深网络架构能比这些结果更好吗？

先在网络最开头插一个卷积层：用 $5\times5$ 局部感受野、步长 1、20 特征映射；再插一个最大池化层，用 $2\times2$ 池化窗口组合特征。整体网络架构很像上节讨论的，但多一个全连接层：

![](images/simple_conv.png)

此架构中，可把卷积与池化层想成学习输入训练图像的局部空间结构，后面全连接层在更抽象层面学习、整合跨整幅图像的全局信息。这是卷积神经网络的常见模式。

训练这种网络看表现如何*。*（这里继续用 mini-batch 10。事实上如我们[早先讨论](ch03-improving-learning-中文翻译.md#mini-batch-大小)可用更大 mini-batch 加速训练。继续用同 mini-batch 大小主要为了与前面章节实验一致。）*：

```
>>> net = Network([
        ConvPoolLayer(image_shape=(mini_batch_size, 1, 28, 28),
                      filter_shape=(20, 1, 5, 5),
                      poolsize=(2, 2)),
        FullyConnectedLayer(n_in=20*12*12, n_out=100),
        SoftmaxLayer(n_in=100, n_out=10)], mini_batch_size)
>>> net.SGD(training_data, 60, mini_batch_size, 0.1,
            validation_data, test_data)
```

这把我们带到 **98.78%** 准确率——比之前任何结果都大幅改进。确实，错误率降低了超过三分之一，是巨大改进。

指定网络结构时我把卷积与池化层当作单层处理。它们算单独层还是单层某种程度是品味问题；network3.py 视为单层因为它让代码更紧凑。不过要的话也容易改 network3.py 让两层可分别指定。

#### 练习

- 若省略全连接层、只用卷积-池化层与 softmax 层，得到多少分类准确率？加入全连接层有帮助吗？

能否在 98.78% 基础上改进？

试试插第二个卷积-池化层——插在现有卷积-池化层与全连接隐藏层之间。仍用 $5\times5$ 局部感受野、$2\times2$ 区域池化。看用与之前类似超参训练时发生什么：

```
>>> net = Network([
        ConvPoolLayer(image_shape=(mini_batch_size, 1, 28, 28),
                      filter_shape=(20, 1, 5, 5),
                      poolsize=(2, 2)),
        ConvPoolLayer(image_shape=(mini_batch_size, 20, 12, 12),
                      filter_shape=(40, 20, 5, 5),
                      poolsize=(2, 2)),
        FullyConnectedLayer(n_in=40*4*4, n_out=100),
        SoftmaxLayer(n_in=100, n_out=10)], mini_batch_size)
>>> net.SGD(training_data, 60, mini_batch_size, 0.1,
            validation_data, test_data)
```

又见改进：现在 **99.06%** 分类准确率！

此刻有两个自然问题。第一：应用第二卷积-池化层到底意味着什么？其实可把第二卷积-池化层想成输入是 $12\times12$"图像"，其"像素"表示原始输入图中特定局部特征的存在（或缺失）。即可把这层想成输入是原始输入图的一个版本——被抽象与浓缩、但仍有大量空间结构——所以用第二卷积-池化层有意义。

这观点令人满意，却引出第二问题。上一层输出涉及 20 个独立特征映射，所以第二卷积-池化层有 $20\times12\times12$ 个输入——仿佛给这层输入 20 张独立图，而非第一层那样的单张图。第二卷积-池化层的神经元该如何响应这些多输入图？事实上，我们允许这层每个神经元从其局部感受野里的*全部* $20\times5\times5$ 输入神经元学习。更非正式地说：第二卷积-池化层的特征检测器能访问*所有*上一层特征，但只在其特定局部感受野内*。*（若输入图像是彩色的，这问题在第一层就会出现——每像素 3 个输入特征对应 RGB 通道，特征检测器可访问全部颜色信息、但只在给定局部感受野内。）*

#### 问题

- **用 tanh 激活函数** 本书前面几次提到 [tanh 函数](ch03-improving-learning-中文翻译.md#其他人工神经元模型)可能比 sigmoid 更好的论证，但我们从没照做——sigmoid 已取得大量进展。现在用 tanh 做激活函数实验。试着在卷积与全连接层用 tanh 激活训练网络*。*（注意可把 activation_fn=tanh 作为参数传给 ConvPoolLayer 与 FullyConnectedLayer 类。）*先与 sigmoid 网络相同超参，但只训 20 回合而非 60。网络表现如何？续到 60 回合呢？试着画 tanh 与 sigmoid 网络到 60 回合的逐回合验证准确率。若结果与我类似，会发现 tanh 网络训稍快、但最终准确率很相似。能解释为何 tanh 网络可能训更快吗？能否改学习率或重缩放让 sigmoid 也得到类似训练速度*。*（回想 $\sigma(z)=(1+\tanh(z/2))/2$ 也许会给你灵感。）*？试半打学习超参或架构迭代，寻找 tanh 可能优于 sigmoid 的方式。*注：这是开放式问题。个人而言我没发现换 tanh 有多大好处（虽未穷尽实验，也许你能找到）。总之稍后我们会发现换修正线性激活函数有好处，所以不再深入用 tanh。*


**用修正线性单元（ReLU）：** 我们目前发展的网络其实是 1998 年开创性论文*。*（Yann LeCun、Léon Bottou、Yoshua Bengio 与 Patrick Haffner (1998) 的 ["Gradient-based learning applied to document recognition"](http://yann.lecun.com/exdb/publis/pdf/lecun-98.pdf)。细节差异很多，但大致说我们的网络与该文描述的 LeNet-5 颇相似。）*引入 MNIST 问题时所用网络的变体——LeNet-5。它是进一步实验、建立理解与直觉的好基础。特别是有许多改网络的方法可尝试改进结果。

首先，把神经元换成[修正线性单元](ch03-improving-learning-中文翻译.md#其他人工神经元模型)——用激活函数 $f(z)\equiv\max(0,z)$。训练 60 回合、学习率 $\eta=0.03$。我还发现用点 [l2 正则化](ch03-improving-learning-中文翻译.md#_正则化)有帮助，正则化参数 $\lambda=0.1$：

```
>>> from network3 import ReLU
>>> net = Network([
        ConvPoolLayer(image_shape=(mini_batch_size, 1, 28, 28),
                      filter_shape=(20, 1, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        ConvPoolLayer(image_shape=(mini_batch_size, 20, 12, 12),
                      filter_shape=(40, 20, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        FullyConnectedLayer(n_in=40*4*4, n_out=100, activation_fn=ReLU),
        SoftmaxLayer(n_in=100, n_out=10)], mini_batch_size)
>>> net.SGD(training_data, 60, mini_batch_size, 0.03,
            validation_data, test_data, lmbda=0.1)
```

我得到 **99.23%** 分类准确率——比 sigmoid 结果 (99.06%) 小有改进。但在我所有实验里，基于 ReLU 的网络一贯优于基于 sigmoid 激活的网络。对这个题目转向 ReLU 似乎有真实收益。

什么让 ReLU 激活函数优于 sigmoid 或 tanh？目前我们对答案理解很差。确实，ReLU 只在过去几年才开始被广泛使用——被采纳的原因是经验的：一些人试了 ReLU（常基于预感或启发式论证*。*（常见理由是 $\max(0,z)$ 在大 $z$ 极限不饱和，不像 sigmoid 神经元，这帮助 ReLU 继续学习。这论证就其本身没问题，但谈不上详细理由，更多是"就这么回事"的故事。注意我们早在[第 2 章](ch02-backpropagation-中文翻译.md)讨论过饱和问题。）*），在基准数据集取得好结果，做法蔓延开来。理想世界我们该有理论告诉我们为哪个应用选哪个激活函数，但目前离那种世界还很远。若靠更好的激活函数选择能获得更大改进，我一点也不惊讶；也预期未来几十年会发展出强有力的激活函数理论。今天仍须依赖理解很差的经验法则。

**扩展训练数据：** 另一条可能改进结果的路是算法扩展训练数据。简单方式是把每张训练图平移一个像素——上移、下移、左移或右移一像素。可运行 expand_mnist.py 程序做*。*（expand_mnist.py 代码见[此处](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/src/expand_mnist.py)。）*：

```
$ python expand_mnist.py
```

此程序取 50,000 张 MNIST 训练图，准备含 250,000 张图的扩展训练集。然后用这些图训练网络——用与上面相同、带 ReLU 的网络。我最初实验减少训练回合数（合理，因为我们用 5 倍数据训练）；但扩展数据实际大幅减少过拟合，所以实验一番后最终回到 60 回合训练。总之，训练：

用扩展训练数据我得到 **99.37%** 训练准确率。这个几乎微不足道的改动带来分类准确率的实质提升。事实上，如我们[早先讨论](ch03-improving-learning-中文翻译.md#其他正则化技术)的，算法扩展数据这想法可更进一步。提醒早先讨论里一些结果的味道：2003 年 Simard、Steinkraus 与 Platt*。*（Patrice Simard、Dave Steinkraus 与 John Platt (2003) 的 [Best Practices for Convolutional Neural Networks Applied to Visual Document Analysis](http://dx.doi.org/10.1109/ICDAR.2003.1227801)。）*用与我们非常相似的网络把 MNIST 性能提到 99.6%——两个卷积-池化层后接 100 神经元全连接隐藏层。他们架构有几处细节差异（比如没有 ReLU 的优势），但改进性能的关键是扩展训练数据——靠旋转、平移、倾斜 MNIST 训练图。他们还发展了"弹性畸变"过程——模拟人手写字时肌肉的随机振荡。结合所有这些过程大幅增加训练数据有效规模，从而达 99.6%。

#### 问题

- 卷积层的想法是在图像间以不变方式行动。那么，当我们只平移输入数据网络却能学更多，似乎令人惊讶。能解释为何这其实相当合理吗？

**插入额外全连接层：** 能做得更好吗？一种可能是完全用上面相同程序、但扩大全连接层规模。我试了 300 与 1,000 神经元，分别得 99.46% 与 99.43%——有趣，但不算对早先结果 (99.37%) 的有说服力胜利。

加额外全连接层呢？试着插入额外全连接层，使有两个 100 隐藏神经元全连接层：

```
>>> net = Network([
        ConvPoolLayer(image_shape=(mini_batch_size, 1, 28, 28),
                      filter_shape=(20, 1, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        ConvPoolLayer(image_shape=(mini_batch_size, 20, 12, 12),
                      filter_shape=(40, 20, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        FullyConnectedLayer(n_in=40*4*4, n_out=100, activation_fn=ReLU),
        FullyConnectedLayer(n_in=100, n_out=100, activation_fn=ReLU),
        SoftmaxLayer(n_in=100, n_out=10)], mini_batch_size)
>>> net.SGD(expanded_training_data, 60, mini_batch_size, 0.03,
            validation_data, test_data, lmbda=0.1)
```

如此得 99.43% 测试准确率——扩展网络帮助也不大。用 300 与 1,000 神经元全连接层做类似实验分别得 99.48% 与 99.47%——令人鼓舞但仍差真正决定性胜利。

这是怎么回事？是扩展或额外全连接层对 MNIST 真没帮助？还是网络有做得更好的能力、但我们学习方式不对？例如也许可用更强正则化技术减少过拟合倾向——一种可能是[第 3 章](ch03-improving-learning-中文翻译.md#其他正则化技术)引入的 [dropout](ch03-improving-learning-中文翻译.md#dropout) 技术。回顾 dropout 基本想法：训练时随机移除单个激活，使模型对单条证据丢失更鲁棒、不太可能依赖训练数据特定怪癖。试试把 dropout 应用到最终全连接层：

```
>>> net = Network([
        ConvPoolLayer(image_shape=(mini_batch_size, 1, 28, 28),
                      filter_shape=(20, 1, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        ConvPoolLayer(image_shape=(mini_batch_size, 20, 12, 12),
                      filter_shape=(40, 20, 5, 5),
                      poolsize=(2, 2),
                      activation_fn=ReLU),
        FullyConnectedLayer(
            n_in=40*4*4, n_out=1000, activation_fn=ReLU, p_dropout=0.5),
        FullyConnectedLayer(
            n_in=1000, n_out=1000, activation_fn=ReLU, p_dropout=0.5),
        SoftmaxLayer(n_in=1000, n_out=10, p_dropout=0.5)],
        mini_batch_size)
>>> net.SGD(expanded_training_data, 40, mini_batch_size, 0.03,
            validation_data, test_data)
```

如此得 **99.60%** 准确率——比早先结果实质改进，尤其对比主要基准（100 隐藏神经元网络达 99.37%）。

有两处改动值得注意。第一，我把训练回合数减到 40——dropout 减少过拟合，所以学得更快。第二，全连接隐藏层有 1,000 神经元而非早先 100——dropout 训练时实际省略许多神经元，所以一些扩展在意料之中。事实上我试了 300 与 1,000 隐藏神经元，1000 神经元验证性能（略）更好。

**用网络集成（ensemble）：** 进一步改善性能的简单方法：建几个神经网络，让它们投票定最佳分类。假设按上面处方训练 5 个不同网络，每个准确率接近 99.6%。虽各网络准确率相似，但因随机初始化不同，它们很可能犯不同错误。合理预期 5 个网络投票可能比任何单个网络分类更好。

这听好得难以置信，但这种集成是神经网络与其他机器学习技术的常用技巧，也确实带来进一步改进：最终达 **99.67%** 准确率——即我们的网络集成正确分类 10,000 张测试图中除 33 张外的全部。

测试集剩余错误如下。右上标签是 MNIST 数据的正确分类，右下是集成网络输出的标签：

![](images/ensemble_errors.png)

值得细看。前两个数字（6 与 5）是集成真正的错误，但也是可理解的错误——人类也可能犯那种错：那 6 确实很像 0、5 很像 3。第三张（据称 8）在我看来更像 9，所以我站网络集成这边：它比最初写这数字的人干得更好。另一方面，第四张（6）似乎确实被网络分得很糟。

如此等等。多数情况下网络选择至少看起来合理，有些甚至比原写字的人分类更好。总体我们的网络提供出色性能——尤其考虑到它们正确分类了未显示的 9,967 张图。此背景下，这里少数明显错误相当可理解。即使细心的人类也偶犯错。所以我预期只有极其仔细、有条理的人类才能好得多——我们的网络正接近人类表现。

**为何我们只对全连接层应用 dropout：** 细看上面代码，我们只对网络全连接部分应用 dropout、没对卷积层。原则上可对卷积层应用类似程序，但事实上没必要——卷积层有相当的内在抗过拟合能力：共享权重意味着卷积滤波器被迫从整幅图学习，使它们不太可能抓住训练数据的局部怪癖，所以更不需要 dropout 等其他正则化器。

**更进一步：** MNIST 性能还能再提高。Rodrigo Benenson 编了[信息量大的总结页](http://rodrigob.github.io/are_we_there_yet/build/classification_datasets_results.html)展示历年进展并链到论文。许多论文用与我们所使用类似的深度卷积网络。挖论文会发现许多有趣技术，你可能乐在其中实现一些。若实现，明智做法是从能快速训练的网络开始，帮你更快理解发生了什么。

大体上我不综述这些近期工作，但忍不住破一例：Cireșan、Meier、Gambardella 与 Schmidhuber 2010 年论文*。*（Dan Claudiu Cireșan、Ueli Meier、Luca Maria Gambardella 与 Jürgen Schmidhuber (2010) 的 [Deep, Big, Simple Neural Nets Excel on Handwritten Digit Recognition](http://arxiv.org/abs/1003.0358)。）*我喜欢它够简单。网络是多层神经网络、只用全连接层（无卷积）。他们最成功的网络隐藏层分别含 2,500、2,000、1,500、1,000、500 神经元；用与 Simard *et al* 类似的想法扩展训练数据；除此之外几乎没用其他技巧——没有卷积层，是朴素的普通网络——那种只要足够耐心、在 1980 年代就能训练（若 MNIST 数据集当时存在）、给足算力的话。他们达 99.65% 分类准确率，与我们的几乎相同。关键是用非常大、非常深的网络，并用 GPU 加速训练，让他们能训很多回合；也利用长训练时间把学习率从 $10^{-3}$ 逐渐降到 $10^{-6}$。用他们那样架构试着追平结果是好玩练习。

**我们为何能训练？** 我们在[上一章](ch05-deep-networks-hard-to-train-中文翻译.md)看到深度多层网络训练有根本障碍——梯度相当不稳定：从输出层到前层，梯度倾向消失（消失梯度问题）或爆炸（爆炸梯度问题）。既然梯度是我们训练用的信号，这会造成麻烦。

我们如何避开那些结果？

当然，答案是我们没避开——而是做了几件帮助继续推进的事。特别是：(1) 用卷积层大幅减少那些层参数数量，让学习问题容易得多；(2) 用更强正则化技术（尤其 dropout 与卷积层）减少过拟合——否则在更复杂网络更成问题；(3) 用 ReLU 而非 sigmoid 神经元加速训练——经验上常快 3–5 倍；(4) 用 GPU 且愿意长时间训练——尤其最后实验用比原始 MNIST 训练数据大 5 倍的数据集训 40 回合；本书前面多用原始数据训 30 回合。结合 (3)(4)，好比我们训练时间比之前长约 30 倍。

你的回应也许是"就这？训练深度网络就只需做这些？有什么好大惊小怪的？"

当然，我们也用了其他想法：用足够大数据集（帮助避免过拟合）；用对代价函数（[避免学习变慢](ch03-improving-learning-中文翻译.md#_交叉熵代价函数)）；用[好权重初始化](ch03-improving-learning-中文翻译.md#_权重初始化)（也避免由神经元饱和导致的学习变慢）；[算法扩展训练数据](ch03-improving-learning-中文翻译.md#其他正则化技术)。前面章节讨论过这些与其他想法，本章多半不加评论地复用。

话虽如此，这真是一组相当简单的想法——简单但合用起来强大。开始深度学习竟如此容易！

**这些网络到底有多深？** 把卷积-池化层算单层，我们最终架构有 4 个隐藏层。这网络真配叫*深度*网络吗？4 隐藏层当然比早先研究的浅层网络多得多——那些网络大多只有 1 隐藏层、偶尔 2 层。另一方面，截至 2015 年最先进深度网络有时有几十隐藏层。我偶尔听到人采取"比你深"态度，认为隐藏层数跟不上趟就不算真做深度学习。我不赞同这态度——部分因为它把深度学习定义变成依赖"当下结果"的东西。深度学习的真正突破是认识到超越 2000 年代中期主导的 1–2 隐藏层浅层网络是可行的——那真是重大突破，开辟了更具表达力模型的探索。但除此之外，层数不是首要根本关注点；用更深网络是帮助达成其他目标（如更好分类准确率）的工具。

**关于流程的话：** 本节我们顺畅地从单隐藏层浅层网络走到多层卷积网络——看起来都那么容易！我们做个改动、多半就得到改进。若你开始实验，我保证事情不会总这么顺——因为我呈现的是清洗过的叙事，省略了许多实验（包括许多失败实验）。这清洗过的叙事希望能帮你弄清基本想法，但也有传达不完整印象的风险。做出好用网络可能涉及大量试错与偶尔挫折。实践应预期做相当多实验。为加速可重读第 3 章[如何选超参](ch03-improving-learning-中文翻译.md#如何选择神经网络的超参数)的讨论，也许再看那节建议的延伸阅读。

### 卷积网络的代码

好，看 network3.py 程序代码。结构上类似第 3 章发展的 network2.py，但因用 Theano 细节不同。先看 FullyConnectedLayer 类——与本书前面研究的层类似。代码如下（讨论在下方）*。*（2016 年 11 月补注：几位读者注意到初始化 self.w 的行里我设 scale=np.sqrt(1.0/n_out)，而第 3 章论证提示更好的初始化可能是 scale=np.sqrt(1.0/n_in)。这是我方的失误。理想世界我会用正确代码重跑本章所有例子，但我已转向其他项目，就让这错误过去吧。）*

\_\_init\_\_ 方法多半不言自明，但几句说明有助澄清代码。照例我们用合适标准差的正常随机变量随机初始化权重偏置；做这事的行看着有点吓人，但大部分复杂只是把权重偏置载入 Theano 所谓共享变量——确保这些变量可在 GPU（若有）上处理。我们不太深入这些细节，感兴趣可挖 [Theano 文档](http://deeplearning.net/software/theano/index.html)。注意此权重偏置初始化是为 sigmoid 激活函数设计的（如[前面讨论](ch03-improving-learning-中文翻译.md#_权重初始化)）；理想上对 tanh 与修正线性函数等激活函数该略不同地初始化权重偏置，这在下面问题里进一步讨论。\_\_init\_\_ 方法以 `self.params = [self.w, self.b]` 结尾——把层关联的全部可学习参数打包的便利方式。后面 Network.SGD 方法用 params 属性找出 Network 实例里哪些变量可学习。

set_inpt 方法用来设置层输入并计算对应输出。我用名字 inpt 而非 input，因为 input 是 Python 内建函数——乱动内建函数倾向导致不可预测行为与难诊断 bug。注意我们实际以两种方式设置输入：self.inpt 与 self.inpt_dropout。这样做因为训练时可能想用 dropout——若如此要移除 self.p_dropout 比例的神经元，正是 set_inpt 倒数第二行 dropout_layer 函数所做的。所以 self.inpt_dropout 与 self.output_dropout 训练时用，self.inpt 与 self.output 用于其他所有目的（如在验证与测试数据上评估准确率）。

ConvPoolLayer 与 SoftmaxLayer 类定义与 FullyConnectedLayer 类似——近到我不在此节录代码，感兴趣可看本节稍后 network3.py 完整清单。

但有几点细节差异值得提。最明显：ConvPoolLayer 与 SoftmaxLayer 都按该层类型合适的方式计算输出激活——幸好 Theano 让这容易，提供内建操作算卷积、最大池化与 softmax。不太明显的：我们[介绍 softmax 层](ch03-improving-learning-中文翻译.md#softmax)时从没讨论如何初始化其权重偏置。别处我们论证对 sigmoid 层该用合适参数化的正常随机变量初始化权重——但那启发式论证是 sigmoid 神经元特有（稍加修正也适用 tanh），没特别理由该适用于 softmax 层。所以没有*先验*理由再次应用那初始化。与其那样，我选择把所有权重偏置初始化为 0——相当*临时*的程序，但实践工作得够好。

好，已看所有层类。Network 类呢？先看 \_\_init\_\_ 方法：

```
class Network(object):

    def __init__(self, layers, mini_batch_size):
        """取 \`layers\` 列表（描述网络架构）与 \`mini_batch_size\` 值
        （SGD 训练时使用）。"""
        self.layers = layers
        self.mini_batch_size = mini_batch_size
        self.params = [param for layer in self.layers for param in layer.params]
        self.x = T.matrix("x")
        self.y = T.ivector("y")
        init_layer = self.layers[0]
        init_layer.set_inpt(self.x, self.x, self.mini_batch_size)
        for j in xrange(1, len(self.layers)):
            prev_layer, layer  = self.layers[j-1], self.layers[j]
            layer.set_inpt(
                prev_layer.output, prev_layer.output_dropout, self.mini_batch_size)
        self.output = self.layers[-1].output
        self.output_dropout = self.layers[-1].output_dropout
```

这多半不言自明或几乎如此。`self.params = [param for layer in...]` 把每层参数捆进单列表；如前述，Network.SGD 用 self.params 找出 Network 中哪些变量可学习。`self.x = T.matrix("x")` 与 `self.y = T.ivector("y")` 定义名为 x、y 的 Theano 符号变量，用来表示网络输入与期望输出。

这不是 Theano 教程，所以不太深入"符号变量"意味着什么*。*（[Theano 文档](http://deeplearning.net/software/theano/index.html)提供好入门；卡住可看网上其他教程，如[此教程](http://nbviewer.ipython.org/github/craffel/theano-tutorial/blob/master/Theano%20Tutorial.ipynb)覆盖许多基础。）*但粗略想法是：它们表示数学变量而非显式值。可对这些变量做一切常做的事——加减乘、应用函数等。事实上 Theano 提供许多操作此类符号变量的方式，做卷积、最大池化等。但最大赢点是能用非常一般形式的反向传播算法做快速符号微分——这对把 SGD 应用到各种网络架构极其有用。特别地，接下来几行代码定义网络符号输出。先设初始层输入：

```
init_layer.set_inpt(self.x, self.x, self.mini_batch_size)
```

注意输入一次设一个 mini-batch（所以有 mini_batch_size）。也注意我们把输入 self.x 传了两次——因为可能以两种方式用网络（有无 dropout）。for 循环随后把符号变量 self.x 前向穿过 Network 各层，让我们定义最终 output 与 output_dropout 属性——符号表示 Network 输出。

理解 Network 如何初始化后，看它如何用 SGD 方法训练。代码看着长，但结构其实相当简单。解释注释在代码后。

```
def SGD(self, training_data, epochs, mini_batch_size, eta,
            validation_data, test_data, lmbda=0.0):
        """用小批量随机梯度下降训练网络。"""
        training_x, training_y = training_data
        validation_x, validation_y = validation_data
        test_x, test_y = test_data

        # 计算训练、验证、测试的小批量数
        num_training_batches = size(training_data)/mini_batch_size
        num_validation_batches = size(validation_data)/mini_batch_size
        num_test_batches = size(test_data)/mini_batch_size

        # 定义（正则化）代价函数、符号梯度与更新
        l2_norm_squared = sum([(layer.w**2).sum() for layer in self.layers])
        cost = self.layers[-1].cost(self)+\
               0.5*lmbda*l2_norm_squared/num_training_batches
        grads = T.grad(cost, self.params)
        updates = [(param, param-eta*grad)
                   for param, grad in zip(self.params, grads)]

        # 定义训练一个小批量的函数、以及计算验证与测试小批量准确率的函数
        i = T.lscalar() # mini-batch 索引
        train_mb = theano.function(
            [i], cost, updates=updates,
            givens={
                self.x:
                training_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                training_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        validate_mb_accuracy = theano.function(
            [i], self.layers[-1].accuracy(self.y),
            givens={
                self.x:
                validation_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                validation_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        test_mb_accuracy = theano.function(
            [i], self.layers[-1].accuracy(self.y),
            givens={
                self.x:
                test_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                test_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        self.test_mb_predictions = theano.function(
            [i], self.layers[-1].y_out,
            givens={
                self.x:
                test_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        # 实际训练
        best_validation_accuracy = 0.0
        for epoch in xrange(epochs):
            for minibatch_index in xrange(num_training_batches):
                iteration = num_training_batches*epoch+minibatch_index
                if iteration % 1000 == 0:
                    print("Training mini-batch number {0}".format(iteration))
                cost_ij = train_mb(minibatch_index)
                if (iteration+1) % num_training_batches == 0:
                    validation_accuracy = np.mean(
                        [validate_mb_accuracy(j) for j in xrange(num_validation_batches)])
                    print("Epoch {0}: validation accuracy {1:.2%}".format(
                        epoch, validation_accuracy))
                    if validation_accuracy >= best_validation_accuracy:
                        print("This is the best validation accuracy to date.")
                        best_validation_accuracy = validation_accuracy
                        best_iteration = iteration
                        if test_data:
                            test_accuracy = np.mean(
                                [test_mb_accuracy(j) for j in xrange(num_test_batches)])
                            print('The corresponding test accuracy is {0:.2%}'.format(
                                test_accuracy))
        print("Finished training network.")
        print("Best validation accuracy of {0:.2%} obtained at iteration {1}".format(
            best_validation_accuracy, best_iteration))
        print("Corresponding test accuracy of {0:.2%}".format(test_accuracy))
```

前几行直接——把数据集分成 x、y 分量、计算每数据集小批量数。接下来几行更有趣，展示 Theano 好玩之处。显式节录：

```
# define the (regularized) cost function, symbolic gradients, and updates
        l2_norm_squared = sum([(layer.w**2).sum() for layer in self.layers])
        cost = self.layers[-1].cost(self)+\
               0.5*lmbda*l2_norm_squared/num_training_batches
        grads = T.grad(cost, self.params)
        updates = [(param, param-eta*grad)
                   for param, grad in zip(self.params, grads)]
```

这些行符号性设置正则化对数似然代价函数、在梯度函数中计算相应导数与参数更新——Theano 只用这几行就让我们做到这一切。唯一隐藏的是计算代价涉及调用输出层的 cost 方法（代码在 network3.py 别处，但那段代码又短又简单）。定义好这一切后，舞台就绪可定义 train_mb 函数——一个 Theano 符号函数，给定 mini-batch 索引用 updates 更新 Network 参数。类似地 validate_mb_accuracy 与 test_mb_accuracy 计算 Network 在任意给定验证/测试小批量上的准确率；对这些函数平均，就能计算整个验证与测试数据集的准确率。

SGD 方法其余部分不言自明——只是迭代回合、反复在训练 mini-batch 上训练网络并计算验证与测试准确率。


好，现在已理解 network3.py 最重要的代码部分。快速看整个程序——不必细读，但可愉快略读、甚至深入任何感兴趣片段。真正理解它的最好方式当然是修改它、加功能或重构任何你认为能做得更优雅的部分。代码后有些问题含起步建议。代码在此*。*（在 GPU 上使用 Theano 可能有点棘手：容易犯把数据从 GPU 拉下来的错、拖慢很多。我尽量避免了。话虽如此，此代码当然还能靠仔细优化 Theano 配置进一步加速，详见 Theano 文档。）*：

```
"""network3.py
~~~~~~~~~~~~~~

基于 Theano 的程序，用于训练与运行简单神经网络。

支持多种层类型（全连接、卷积、最大池化、softmax）与激活函数
（sigmoid、tanh 与修正线性单元，且易添加更多）。

在 CPU 上运行时此程序比 network.py 与 network2.py 快得多。
但与 network.py、network2.py 不同，它也可在 GPU 上运行，更快。

因代码基于 Theano，代码在许多方面与 network.py、network2.py 不同。
不过只要可能我尽量与早期程序保持一致——尤其 API 类似 network2.py。
注意我聚焦让代码简单、易读、易改；未优化，省略许多理想特性。

本程序吸收 Theano 卷积神经网络文档的想法（尤其
http://deeplearning.net/tutorial/lenet.html ）、Misha Denil 的 dropout
实现（https://github.com/mdenil/dropout ）与 Chris Olah
（http://colah.github.io ）的想法。

为 Theano 0.6 与 0.7 编写，用于更新版本需一些改动。

"""

#### 库
# 标准库
import cPickle
import gzip

# 第三方库
import numpy as np
import theano
import theano.tensor as T
from theano.tensor.nnet import conv
from theano.tensor.nnet import softmax
from theano.tensor import shared_randomstreams
from theano.tensor.signal import downsample

# 神经元的激活函数
def linear(z): return z
def ReLU(z): return T.maximum(0.0, z)
from theano.tensor.nnet import sigmoid
from theano.tensor import tanh

#### 常量
GPU = True
if GPU:
    print "Trying to run under a GPU.  If this is not desired, then modify "+\
        "network3.py\nto set the GPU flag to False."
    try: theano.config.device = 'gpu'
    except: pass # it's already set
    theano.config.floatX = 'float32'
else:
    print "Running with a CPU.  If this is not desired, then the modify "+\
        "network3.py to set\nthe GPU flag to True."

#### 加载 MNIST 数据
def load_data_shared(filename="../data/mnist.pkl.gz"):
    f = gzip.open(filename, 'rb')
    training_data, validation_data, test_data = cPickle.load(f)
    f.close()
    def shared(data):
        """把数据放入共享变量。这让 Theano 可把数据复制到 GPU（若有）。"""
        shared_x = theano.shared(
            np.asarray(data[0], dtype=theano.config.floatX), borrow=True)
        shared_y = theano.shared(
            np.asarray(data[1], dtype=theano.config.floatX), borrow=True)
        return shared_x, T.cast(shared_y, "int32")
    return [shared(training_data), shared(validation_data), shared(test_data)]

#### 用于构造与训练网络的主类
class Network(object):

    def __init__(self, layers, mini_batch_size):
        """取 \`layers\` 列表（描述网络架构）与 \`mini_batch_size\` 值
        （SGD 训练时使用）。"""
        self.layers = layers
        self.mini_batch_size = mini_batch_size
        self.params = [param for layer in self.layers for param in layer.params]
        self.x = T.matrix("x")
        self.y = T.ivector("y")
        init_layer = self.layers[0]
        init_layer.set_inpt(self.x, self.x, self.mini_batch_size)
        for j in xrange(1, len(self.layers)):
            prev_layer, layer  = self.layers[j-1], self.layers[j]
            layer.set_inpt(
                prev_layer.output, prev_layer.output_dropout, self.mini_batch_size)
        self.output = self.layers[-1].output
        self.output_dropout = self.layers[-1].output_dropout

    def SGD(self, training_data, epochs, mini_batch_size, eta,
            validation_data, test_data, lmbda=0.0):
        """用小批量随机梯度下降训练网络。"""
        training_x, training_y = training_data
        validation_x, validation_y = validation_data
        test_x, test_y = test_data

        # 计算训练、验证、测试的小批量数
        num_training_batches = size(training_data)/mini_batch_size
        num_validation_batches = size(validation_data)/mini_batch_size
        num_test_batches = size(test_data)/mini_batch_size

        # 定义（正则化）代价函数、符号梯度与更新
        l2_norm_squared = sum([(layer.w**2).sum() for layer in self.layers])
        cost = self.layers[-1].cost(self)+\
               0.5*lmbda*l2_norm_squared/num_training_batches
        grads = T.grad(cost, self.params)
        updates = [(param, param-eta*grad)
                   for param, grad in zip(self.params, grads)]

        # 定义训练一个小批量的函数、以及计算验证与测试小批量准确率的函数
        i = T.lscalar() # mini-batch 索引
        train_mb = theano.function(
            [i], cost, updates=updates,
            givens={
                self.x:
                training_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                training_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        validate_mb_accuracy = theano.function(
            [i], self.layers[-1].accuracy(self.y),
            givens={
                self.x:
                validation_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                validation_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        test_mb_accuracy = theano.function(
            [i], self.layers[-1].accuracy(self.y),
            givens={
                self.x:
                test_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size],
                self.y:
                test_y[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        self.test_mb_predictions = theano.function(
            [i], self.layers[-1].y_out,
            givens={
                self.x:
                test_x[i*self.mini_batch_size: (i+1)*self.mini_batch_size]
            })
        # 实际训练
        best_validation_accuracy = 0.0
        for epoch in xrange(epochs):
            for minibatch_index in xrange(num_training_batches):
                iteration = num_training_batches*epoch+minibatch_index
                if iteration % 1000 == 0:
                    print("Training mini-batch number {0}".format(iteration))
                cost_ij = train_mb(minibatch_index)
                if (iteration+1) % num_training_batches == 0:
                    validation_accuracy = np.mean(
                        [validate_mb_accuracy(j) for j in xrange(num_validation_batches)])
                    print("Epoch {0}: validation accuracy {1:.2%}".format(
                        epoch, validation_accuracy))
                    if validation_accuracy >= best_validation_accuracy:
                        print("This is the best validation accuracy to date.")
                        best_validation_accuracy = validation_accuracy
                        best_iteration = iteration
                        if test_data:
                            test_accuracy = np.mean(
                                [test_mb_accuracy(j) for j in xrange(num_test_batches)])
                            print('The corresponding test accuracy is {0:.2%}'.format(
                                test_accuracy))
        print("Finished training network.")
        print("Best validation accuracy of {0:.2%} obtained at iteration {1}".format(
            best_validation_accuracy, best_iteration))
        print("Corresponding test accuracy of {0:.2%}".format(test_accuracy))

#### 定义层类型

class ConvPoolLayer(object):
    """用于创建卷积与最大池化层的组合。更精细的实现会把两者分开，
    但对我们用途总是一起用，且合并简化代码，所以合并合理。"""

    def __init__(self, filter_shape, image_shape, poolsize=(2, 2),
                 activation_fn=sigmoid):
        """\`filter_shape\` 是长度 4 的元组：滤波器数、输入特征映射数、
        滤波器高、滤波器宽。

        \`image_shape\` 是长度 4 的元组：mini-batch 大小、输入特征映射数、
        图像高、图像宽。

        \`poolsize\` 是长度 2 的元组：y 与 x 的池化大小。"""
        self.filter_shape = filter_shape
        self.image_shape = image_shape
        self.poolsize = poolsize
        self.activation_fn=activation_fn
        # 初始化权重与偏置
        n_out = (filter_shape[0]*np.prod(filter_shape[2:])/np.prod(poolsize))
        self.w = theano.shared(
            np.asarray(
                np.random.normal(loc=0, scale=np.sqrt(1.0/n_out), size=filter_shape),
                dtype=theano.config.floatX),
            borrow=True)
        self.b = theano.shared(
            np.asarray(
                np.random.normal(loc=0, scale=1.0, size=(filter_shape[0],)),
                dtype=theano.config.floatX),
            borrow=True)
        self.params = [self.w, self.b]

    def set_inpt(self, inpt, inpt_dropout, mini_batch_size):
        self.inpt = inpt.reshape(self.image_shape)
        conv_out = conv.conv2d(
            input=self.inpt, filters=self.w, filter_shape=self.filter_shape,
            image_shape=self.image_shape)
        pooled_out = downsample.max_pool_2d(
            input=conv_out, ds=self.poolsize, ignore_border=True)
        self.output = self.activation_fn(
            pooled_out + self.b.dimshuffle('x', 0, 'x', 'x'))
        self.output_dropout = self.output # 卷积层不用 dropout

class FullyConnectedLayer(object):

    def __init__(self, n_in, n_out, activation_fn=sigmoid, p_dropout=0.0):
        self.n_in = n_in
        self.n_out = n_out
        self.activation_fn = activation_fn
        self.p_dropout = p_dropout
        # 初始化权重与偏置
        self.w = theano.shared(
            np.asarray(
                np.random.normal(
                    loc=0.0, scale=np.sqrt(1.0/n_out), size=(n_in, n_out)),
                dtype=theano.config.floatX),
            name='w', borrow=True)
        self.b = theano.shared(
            np.asarray(np.random.normal(loc=0.0, scale=1.0, size=(n_out,)),
                       dtype=theano.config.floatX),
            name='b', borrow=True)
        self.params = [self.w, self.b]

    def set_inpt(self, inpt, inpt_dropout, mini_batch_size):
        self.inpt = inpt.reshape((mini_batch_size, self.n_in))
        self.output = self.activation_fn(
            (1-self.p_dropout)*T.dot(self.inpt, self.w) + self.b)
        self.y_out = T.argmax(self.output, axis=1)
        self.inpt_dropout = dropout_layer(
            inpt_dropout.reshape((mini_batch_size, self.n_in)), self.p_dropout)
        self.output_dropout = self.activation_fn(
            T.dot(self.inpt_dropout, self.w) + self.b)

    def accuracy(self, y):
        "返回小批量的准确率。"
        return T.mean(T.eq(y, self.y_out))

class SoftmaxLayer(object):

    def __init__(self, n_in, n_out, p_dropout=0.0):
        self.n_in = n_in
        self.n_out = n_out
        self.p_dropout = p_dropout
        # 初始化权重与偏置
        self.w = theano.shared(
            np.zeros((n_in, n_out), dtype=theano.config.floatX),
            name='w', borrow=True)
        self.b = theano.shared(
            np.zeros((n_out,), dtype=theano.config.floatX),
            name='b', borrow=True)
        self.params = [self.w, self.b]

    def set_inpt(self, inpt, inpt_dropout, mini_batch_size):
        self.inpt = inpt.reshape((mini_batch_size, self.n_in))
        self.output = softmax((1-self.p_dropout)*T.dot(self.inpt, self.w) + self.b)
        self.y_out = T.argmax(self.output, axis=1)
        self.inpt_dropout = dropout_layer(
            inpt_dropout.reshape((mini_batch_size, self.n_in)), self.p_dropout)
        self.output_dropout = softmax(T.dot(self.inpt_dropout, self.w) + self.b)

    def cost(self, net):
        "返回对数似然代价。"
        return -T.mean(T.log(self.output_dropout)[T.arange(net.y.shape[0]), net.y])

    def accuracy(self, y):
        "返回小批量的准确率。"
        return T.mean(T.eq(y, self.y_out))

#### 杂项
def size(data):
    "返回数据集 \`data\` 的大小。"
    return data[0].get_value(borrow=True).shape[0]

def dropout_layer(layer, p_dropout):
    srng = shared_randomstreams.RandomStreams(
        np.random.RandomState(0).randint(999999))
    mask = srng.binomial(n=1, p=1-p_dropout, size=layer.shape)
    return layer*T.cast(mask, theano.config.floatX)
```

#### 问题

- 目前 SGD 方法要求用户手动选训练回合数。本书前面讨论过自动选回合数的办法——[提前停止](ch03-improving-learning-中文翻译.md#_用提前停止确定训练回合数)。修改 network3.py 实现提前停止。
- 加 Network 方法返回任意数据集上的准确率。
- 修改 SGD 方法允许学习率 $\eta$ 是回合号的函数。*提示：做一阵此题后，看[此链接](https://groups.google.com/forum/#!topic/theano-users/NQ9NYLvleGc)的讨论会有帮助。*
- 本章前面我描述了用（小）旋转、倾斜、平移扩展训练数据的技术。修改 network3.py 纳入所有这些技术。*注：除非内存极大，否则显式生成整个扩展数据集不实际，应考虑替代方法。*
- 给 network3.py 加加载与保存网络的能力。
- 当前代码的缺点：提供很少诊断工具。能想到加哪些诊断、让理解网络过拟合程度更容易吗？加上它们。
- 我们给 ReLU 用了与 sigmoid（与 tanh）神经元相同的初始化程序。我们对那初始化的[论证](ch03-improving-learning-中文翻译.md#_权重初始化)是 sigmoid 函数特有。考虑完全由 ReLU 构成的网络（含输出）。证明把网络所有权重乘以正常数 $c>0$ 只是把输出乘以 $c^{L-1}$（$L$ 是层数）。若最后一层是 softmax 会怎样改变？你如何看待对 ReLU 用 sigmoid 初始化程序？能想到更好初始化程序吗？*注：这是非常开放的问题，没有简单自足答案。但思考它会帮你更好理解含 ReLU 的网络。*
- 我们对不稳定梯度问题的[分析](ch05-deep-networks-hard-to-train-中文翻译.md#什么导致梯度消失问题)针对 sigmoid 神经元。对由 ReLU 构成的网络，分析如何改变？能想到修改这种网络使其不遭受不稳定梯度问题的好办法吗？*注：第二部分"好"字让这成为研究问题。想修改方法其实容易，但我没深究到知道真正好技术。*

### 图像识别的近期进展

1998 年 MNIST 面世那年，训练当时最先进的工作站达到比我们用 GPU 与不到一小时训练差得多的准确率，需时数周。所以 MNIST 不再是推动现有技术极限的问题，训练速度让它成为适合教学学习的题目。与此同时研究焦点已转移，现代工作涉及更具挑战性的图像识别问题。本节简述一些用神经网络做图像识别的近期工作。

本节与本书大部分不同。全书写到如今我聚焦于可能持久有意思的想法——反向传播、正则化、卷积网络——尽量避免我写作时时髦但长期价值未知的结果。科学里这类结果多半是昙花一现、影响甚微。鉴于此，怀疑论者会说："可图像识别的近期进展难道不正是这类昙花？再过两三年事情又会变。这些结果岂不只对少数想在绝对前沿竞争的专家有意思？何必讨论？"

怀疑论者说对了一点：近期论文的一些细枝末节会逐渐显得不那么重要。但过去几年用深度网络攻极难图像识别任务取得非凡进展。想象公元 2100 年写计算机视觉的科学史家：他们会把 2011 到 2015（也许再往后几年）标为深度卷积网络驱动的巨大突破期。这不意味 2100 年还用深度卷积网络（更不用说 dropout、ReLU 等细节想法），但确实意味着此刻、在思想史上，一个重要转变正在发生——有点像目睹原子发现或抗生素发明：历史规模的发明与发现。所以虽不深挖细节，值得了解正在做出的激动人心的发现。

**2012 LRMD 论文：** 从 2012 年一篇论文*。*（Quoc Le、Marc'Aurelio Ranzato、Rajat Monga、Matthieu Devin、Kai Chen、Greg Corrado、Jeff Dean 与 Andrew Ng (2012) 的 [Building high-level features using large scale unsupervised learning](http://research.google.com/pubs/pub38115.html)。注意论文所用网络架构与我们研究的深度卷积网络在许多细节上不同；但大致说 LRMD 基于许多类似想法。）*说起，作者是斯坦福与谷歌的研究者。我按前四作者姓氏称它为 LRMD。LRMD 用神经网络分类 [ImageNet](http://www.image-net.org) 图像——极具挑战性的图像识别问题。他们用的 2011 ImageNet 数据含 1,600 万张全彩图、2 万类别，图从开放网络爬取、由 Amazon Mechanical Turk 工人分类。下面是一些 ImageNet 图*。*（这些来自 2014 数据集，与 2011 略不同；但定性极相似。ImageNet 细节见原始论文：Jia Deng、Wei Dong、Richard Socher、Li-Jia Li、Kai Li 与 Li Fei-Fei (2009) 的 [ImageNet: a large-scale hierarchical image database](http://www.image-net.org/papers/imagenet_cvpr09.pdf)。）*：

![](images/imagenet1.jpg) ![](images/imagenet2.jpg) ![](images/imagenet3.jpg) ![](images/imagenet4.jpg)

它们分别属于嵌线刨、褐根腐菌、烫伤奶与常见蛔虫类别。想挑战的话，鼓励你访问 ImageNet 的[手动工具](http://www.image-net.org/synset?wnid=n03489162)清单——区分嵌线刨、平刨、斜边刨与约十几种其他刨。我不知道你如何，反正我没法自信区分所有这些工具类型。这显然比 MNIST 挑战得多的图像识别任务！LRMD 网络正确分类 ImageNet 图得到可观的 **15.8%** 准确率——听来不咋样，但相对此前最佳 9.3% 是巨大改进。这飞跃提示神经网络可能为 ImageNet 这类极难图像识别任务提供强大方法。

**2012 KSH 论文：** LRMD 工作之后是 Krizhevsky、Sutskever 与 Hinton (KSH) 2012 年论文*。*（Alex Krizhevsky、Ilya Sutskever 与 Geoffrey E. Hinton (2012) 的 [ImageNet classification with deep convolutional neural networks](http://www.cs.toronto.edu/~fritz/absps/imagenet.pdf)。）*KSH 用 ImageNet 受限子集训练与测试深度卷积网络。他们用的子集来自流行机器学习竞赛——ImageNet 大规模视觉识别挑战（ILSVRC）。用竞赛数据集给了他们把方法与领先技术比较的好途径。ILSVRC-2012 训练集约 120 万张 ImageNet 图、来自 1,000 类别；验证与测试集分别 5 万与 15 万张、来自同样 1,000 类别。

跑 ILSVRC 竞赛的困难之一是许多 ImageNet 图含多个物体。假设图显示拉布拉多追足球——所谓"正确"ImageNet 分类可能是拉布拉多。若算法标成足球该受罚吗？因这歧义，只要真实 ImageNet 分类在算法认为最可能的 5 个分类里就算正确。按此 top-5 标准，KSH 深度卷积网络达 **84.7%** 准确率——远超次优参赛者（73.8%）。用更严格"精确对上标签"的度量，KSH 网络达 63.3%。

值得简述 KSH 网络，因为它启发了大量后续工作，也与我们本章早先训练的网络密切相关（虽更精致）。KSH 用深度卷积神经网络、在两块 GPU 上训练。用两块 GPU 因他们所用 GPU 型号（NVIDIA GeForce GTX 580）片上内存不足存整个网络——所以把网络分两部分、跨两块 GPU。

KSH 网络有 7 层隐藏神经元：前 5 隐藏层是卷积层（一些带最大池化），后 2 层全连接；输出层是 1,000 单元 softmax 层，对应 1,000 图像类。下图是取自 KSH 论文的网络草图*。*（感谢 Ilya Sutskever。）*，细节下文解释。注意许多层分成 2 部分对应 2 块 GPU。

![](images/KSH.jpg)

输入层含 $3\times224\times224$ 神经元，表示 $224\times224$ 图的 RGB 值。如前所述，ImageNet 图分辨率不一——这是问题，因神经网络输入层通常固定大小。KSH 处理：每图重缩放使短边长 256，再裁出重缩放图中心 $256\times256$ 区域，最后从 $256\times256$ 图提取随机 $224\times224$ 子图（与水平镜像）。他们做随机裁剪作为扩展训练数据、减少过拟合的方式——这对 KSH 这种大网络尤其有用。正是这些 $224\times224$ 图用作网络输入；多数情况下裁剪图仍含未裁剪图的主体。

转到 KSH 网络隐藏层：第一隐藏层是带最大池化步的卷积层，用 $11\times11$ 局部感受野、4 像素步长，共 96 特征映射——分两组各 48，前 48 在一 GPU、后 48 在另一 GPU。这层与后面层最大池化在 $3\times3$ 区域做，但池化区允许重叠、只隔 2 像素。

第二隐藏层也是带最大池化的卷积层，用 $5\times5$ 局部感受野、共 256 特征映射分各 128 到每 GPU。注意特征映射只用 48 输入通道、非上一层的全部 96 输出（通常情形）——因任何单特征映射只用同一 GPU 的输入。此意义上网络偏离本章早先描述的卷积架构，不过显然基本想法仍同。

第三、四、五隐藏层是卷积层，但与前层不同、不含最大池化。参数分别：(3) 384 特征映射、$3\times3$ 局部感受野、256 输入通道；(4) 384 特征映射、$3\times3$ 局部感受野、192 输入通道；(5) 256 特征映射、$3\times3$ 局部感受野、192 输入通道。注意第三层涉及一些 GPU 间通信（图中所示），使特征映射用全部 256 输入通道。

第六、七隐藏层是全连接层，各 4,096 神经元。输出层是 1,000 单元 softmax 层。

KSH 网络利用许多技术。不用 sigmoid 或 tanh，KSH 用 ReLU，显著加速训练。KSH 网络约 6,000 万可学习参数，即便大训练集也易过拟合——为克服，他们用前面讨论的随机裁剪策略扩展训练集，还用 [l2 正则化](ch03-improving-learning-中文翻译.md#_正则化)变体与 [dropout](ch03-improving-learning-中文翻译.md#dropout) 进一步应对过拟合。网络本身用[基于动量](ch03-improving-learning-中文翻译.md#其他随机梯度下降变体)的 mini-batch SGD 训练。

这是 KSH 论文许多核心想法的概览。我省略了一些细节，想看应读原论文。也可看 Alex Krizhevsky 的 [cuda-convnet](https://code.google.com/p/cuda-convnet/)（及其后继）——含实现许多想法的代码。还发展了基于 Theano 的实现*。*（Weiguang Ding、Ruoyan Wang、Fei Mao 与 Graham Taylor (2014) 的 [Theano-based large-scale visual recognition with multiple GPUs](http://arxiv.org/abs/1412.2302)，代码见[此处](https://github.com/uoguelph-mlrg/theano_alexnet)。）*代码显然与本章发展的类似，只是多 GPU 使用复杂了些。Caffe 框架也含 KSH 网络版本，见其 [Model Zoo](http://caffe.berkeleyvision.org/model_zoo.html)。

**2014 ILSVRC 竞赛：** 2012 年后快速进展持续。看 2014 ILSVRC 竞赛——与 2012 一样，训练集 120 万图、1,000 类别，衡量标准是 top-5 预测是否含正确类别。冠军队伍（主要在 Google*。*（Christian Szegedy、Wei Liu、Yangqing Jia、Pierre Sermanet、Scott Reed、Dragomir Anguelov、Dumitru Erhan、Vincent Vanhoucke 与 Andrew Rabinovich (2014) 的 [Going deeper with convolutions](http://arxiv.org/abs/1409.4842)。）*）用 22 层神经元的深度卷积网络，命名 GoogLeNet 致敬 LeNet-5。GoogLeNet 达 top-5 准确率 **93.33%**——较 2013 冠军（Clarifai，88.3%）与 2012 冠军（KSH，84.7%）巨大改进。

GoogLeNet 的 93.33% 到底多好？2014 年一组研究者写 ILSVRC 竞赛综述*。*（Olga Russakovsky、Jia Deng、Hao Su、Jonathan Krause、Sanjeev Satheesh、Sean Ma、Zhiheng Huang、Andrej Karpathy、Aditya Khosla、Michael Bernstein、Alexander C. Berg 与 Li Fei-Fei (2014) 的 [ImageNet large scale visual recognition challenge](http://arxiv.org/abs/1409.0575)。）*他们讨论的问题之一是人类在 ILSVRC 表现多好。为此建了让人分类 ILSVRC 图的系统。作者之一 Andrej Karpathy 在[信息量大的博文](http://karpathy.github.io/2014/09/02/what-i-learned-from-competing-against-a-convnet-on-imagenet/)解释：费了很大劲才让人类达到 GoogLeNet 的性能：

> ……从 1000 类中给图标 5 个标签的任务很快显出极难，即使对实验室里研究 ILSVRC 及其类别一段时间的同事也如此。先以为可放上 [Amazon Mechanical Turk]，又想可雇付费本科生。然后我在实验室里组织了只限（专家标注者）的高强度标注派对。接着我开发了改进接口，用 GoogLeNet 预测把类别数从 1000 修剪到约 100。仍太难——人们不断漏类别、错误率达 13–15%。最后我意识到要逼近 GoogLeNet，最有效的是自己坐下来经历痛苦漫长的训练过程与后续仔细标注……标注速度约每分钟 1 张、随时间下降……有些图易认，有些（如细粒度犬种、鸟或猴的图）要几分钟专注努力。我变得很擅长识别犬种……基于我处理的样本，GoogLeNet 分类错误 6.8%……我最终错误 5.1%，约好 1.7%。

换言之，专家人类煞费苦心、费大力才勉强胜过深度神经网络。事实上 Karpathy 报告第二位人类专家（在较小样本上训练）只达 top-5 错误率 12.0%、显著低于 GoogLeNet；约一半错误源于专家"没发现并考虑真实标签作为选项"。

这些是惊人结果。事实上自那以后几个团队报告 top-5 错误率实际*优于* 5.1% 的系统——媒体有时报道为系统具备超人视觉。结果确令人兴奋，但有许多保留让"系统有超人视觉"的想法误导：ILSVRC 挑战在许多方面相当受限（开放网络爬取不一定代表应用中遇到的图像！），top-5 标准当然相当人为。我们离解决图像识别或更广计算机视觉还很远。不过，看到如此难的题在短短几年取得这么多进展，极令人鼓舞。

**其他活动：** 我聚焦 ImageNet，但用神经网络做图像识别还有大量其他活动。简述几个有趣的近期结果，给当前工作的一些味道。

一组令人鼓舞的实用结果来自 Google 团队——他们把深度卷积网络用于识别 Google 街景图像中的街道号码*。*（Ian J. Goodfellow、Yaroslav Bulatov、Julian Ibarz、Sacha Arnoud 与 Vinay Shet (2013) 的 [Multi-digit Number Recognition from Street View Imagery using Deep Convolutional Neural Networks](http://arxiv.org/abs/1312.6082)。）*论文报告检测并自动转录近 1 亿街道号码、准确率接近人类操作员。系统快：不到一小时转录完法国全部街景街道号码图！他们说："这个新数据集显著提高了 Google Maps 在几个国家的地理编码质量，尤其那些原本没有其他良好地理编码来源的国家。"并进一步断言："我们相信用这模型已为许多应用解决了短字符序列的\[光学字符识别\]。"

我可能给了"全是令人鼓舞结果的游行"的印象。当然，一些最有趣的工作报告的是我们还不理解的根本事物。例如 2013 年论文*。*（Christian Szegedy、Wojciech Zaremba、Ilya Sutskever、Joan Bruna、Dumitru Erhan、Ian Goodfellow 与 Rob Fergus (2013) 的 [Intriguing properties of neural networks](http://arxiv.org/abs/1312.6199)。）*展示深度网络可能遭受实质上的盲点。看下面几行图：左是他们的网络正确分类的 ImageNet 图，右是略扰动图（扰动在中间）——被网络*错误*分类。作者发现对几乎每张样本图都存在这类"对抗"图，不只是少数特别的。

![](images/adversarial.jpg)

这是令人不安的结果。论文用的网络基于与 KSH 相同代码——即正被越来越广泛使用的网络类型。虽然这类神经网络原则上计算连续函数，这类结果提示实践中它们很可能计算几乎不连续的函数；更糟，不连续方式违反我们对合理行为的直觉。这令人担忧；而且尚未很好理解什么导致不连续——是损失函数？激活函数？网络架构？还是别的？我们还不知道。

不过这些结果没听来那么糟。虽然这类对抗图常见，实践也罕见。如论文所述：

> 对抗负样本的存在似乎与网络的高泛化能力矛盾。确实，若网络能很好泛化，怎会被这些与常规样本难分彼此的对抗负样本弄混？解释是：对抗负样本集合概率极低、因此测试集中从不（或极少）出现，却又稠密（很像有理数），所以几乎每个测试用例附近都能找到。

然而，我们如此不了解神经网络、以致这类结果竟是近期才发现，这令人痛心。当然，这结果的一大好处是刺激了大量后续工作。例如最近一篇论文*。*（Anh Nguyen、Jason Yosinski 与 Jeff Clune (2014) 的 [Deep Neural Networks are Easily Fooled: High Confidence Predictions for Unrecognizable Images](http://arxiv.org/abs/1412.1897)。）*展示给定训练好的网络，能生成人类看着像白噪声、网络却以很高置信度分为已知类别的图像——又一次展示我们理解神经网络及其在图像识别中的使用还有很长的路要走。

尽管有这类结果，整体图景令人鼓舞。我们看着 ImageNet 这类极难基准快速进展，也看着街景号码识别等真实世界问题快速进展。但仅见基准甚至真实应用的改进不够——仍有理解得很差的基本现象（如对抗图存在）。这类基本问题仍在被发现（更别说解决）时，说"我们接近解决图像识别"为时过早；同时这类问题是进一步工作的激动刺激。

### 深度神经网络的其他方法

贯穿本书我们聚焦单个问题：分类 MNIST 数字。这是多汁的问题、迫使我们理解许多强大想法——SGD、反向传播、卷积网络、正则化等。但它也是窄问题。若读神经网络文献，会遇到许多我们没讨论的想法：循环神经网络、玻尔兹曼机、生成模型、迁移学习、强化学习…… 没完没了！神经网络是广阔领域。不过许多重要想法是我们已讨论想法的变体、稍加努力可理解。本节瞥一眼这些未见的风景。讨论不细致也不全面——那会大幅扩书——而是印象式的：试图唤起领域的概念丰富性、并把其中一些联系到我们已见的。全节我会给些别处来源链接作为深入学习入口。当然许多链接很快会过时，你可能想搜更新的文献；尽管如此，我预期许多底层想法持久有意思。

**循环神经网络（RNN）：** 我们一直用的前馈网络里，单个输入完全决定其余层所有神经元激活——非常静态的图景：网络里一切都固定、带着冻结晶体的品质。但假设允许网络元素以动态方式持续变化：隐藏神经元行为也许不仅由前层激活决定，还由更早时刻激活决定——甚至神经元激活可能部分由其自身更早激活决定（前馈网络当然不这样）；或隐藏与输出神经元激活不只看当前输入、还看更早输入。

带这种时变行为的神经网络叫*循环神经网络*或 *RNN*。把上段非正式描述数学形式化的方式很多，瞥一眼 [RNN 维基](http://en.wikipedia.org/wiki/Recurrent_neural_network)可感受一些模型——我写时该页列出不下 13 种模型。但撇开数学细节，大致想法是 RNN 是网络中存在某种随时间动态变化的神经网络。毫不意外，它们对分析随时间变化的数据或过程特别有用——这类数据与过程天然出现在语音或自然语言等问题。

RNN 当前的一种用法是把神经网络与传统算法思考方式更紧密相连——图灵机与（常规）编程语言等概念。[2014 年一篇论文](http://arxiv.org/abs/1410.4615)发展了一个 RNN，能输入（非常非常简单的！）Python 程序逐字符描述、用描述预测输出——非正式说，网络在学"理解"某些 Python 程序。[另一篇也 2014](http://arxiv.org/abs/1410.5401) 用 RNN 作起点发展所谓神经图灵机（NTM）——整个结构可用梯度下降训练的通用计算机。他们训练 NTM 推断几个简单问题的算法，如排序与复制。

就现状看，这些是极简单的玩具模型——学会执行 `print(398345+42598)` 不把网络变成完整 Python 解释器！不清楚这些想法能推多远。但结果引人入胜：历史上神经网络在常规算法方法困难处做得好的模式识别上擅长；反之常规算法擅长神经网络不太擅长的问题——今天没人用神经网络实现 web 服务器或数据库程序！若能发展整合两者优势的统一模型就太好了，RNN 与受 RNN 启发的想法可能帮我们做到。

RNN 近年也用于攻许多其他问题，尤其在语音识别特别有用。基于 RNN 的方法[创下音素识别准确率纪录](http://arxiv.org/abs/1303.5778)；也被用来发展[人们说话时所用语言的改进模型](http://www.fit.vutbr.cz/~imikolov/rnnlm/thesis.pdf)——更好的语言模型帮助消解听来相似的语音，好模型会告诉我们 "to infinity and beyond" 比 "two infinity and beyond" 更可能，尽管两短语听来相同。RNN 已用于创某些语言基准新纪录。

顺便，这工作属于各类深度网络（不只 RNN）在语音识别的更广使用。例如基于深度网络的方法在[大词汇连续语音识别](http://arxiv.org/abs/1309.1501)取得出色结果；另一基于深度网络的系统已部署在 [Google 安卓操作系统](http://www.wired.com/2013/02/android-neural-network/)（相关技术工作见 [Vincent Vanhoucke 2012–2015 论文](http://research.google.com/pubs/VincentVanhoucke.html)）。

我说了 RNN 能做什么，却没怎么说它们如何工作。你大概不惊讶：前馈网络用的许多想法也能用于 RNN——尤其可用梯度下降与反向传播的直接修改训练 RNN。从前馈网络用的许多其他想法——正则化技术到卷积再到激活与代价函数——在循环网络也有用。所以本书发展的许多技术可改造用于 RNN。

**长短时记忆单元（LSTM）：** 影响 RNN 的一个挑战是早期模型极难训练，比深度前馈网络更难。原因是[第 5 章](ch05-deep-networks-hard-to-train-中文翻译.md)讨论的不稳定梯度问题——回想其通常表现是梯度随反向传播穿层越来越小，使前层学习极慢。这问题在 RNN 更糟：梯度不只反向穿层、还反向穿时间；网络跑得久会让梯度极不稳定、难学。幸运的是，可把叫*长短时记忆单元（LSTM）*的想法纳入 RNN。[Hochreiter 与 Schmidhuber 1997 年](http://dx.doi.org/10.1162/neco.1997.9.8.1735)引入这些单元，明确目的就是帮助应对不稳定梯度问题。LSTM 让训练 RNN 得到好结果容易得多，许多近期论文（包括我上面链的许多）用 LSTM 或相关想法。

**深度信念网络、生成模型与玻尔兹曼机：** 现代深度学习兴趣始于 2006 年——那年的论文解释如何训练叫*深度信念网络（DBN）*的神经网络*。*（见 Geoffrey Hinton、Simon Osindero 与 Yee-Whye Teh (2006) 的 [A fast learning algorithm for deep belief nets](http://www.cs.toronto.edu/~hinton/absps/fastnc.pdf)，以及 Geoffrey Hinton 与 Ruslan Salakhutdinov (2006) [Reducing the dimensionality of data with neural networks](http://www.sciencemag.org/content/313/5786/504.short) 的相关工作。）*DBN 有几年影响很大，此后流行度下降，前馈与循环网络成时尚。尽管如此，DBN 有几处有趣特性。

DBN 有趣的原因之一是它们是所谓*生成模型（generative model）*的例子。前馈网络里我们指定输入激活、它们决定后面特征神经元激活；生成模型（如 DBN）可类似使用，但也可指定某些特征神经元值、然后"反向运行网络"生成输入激活值。具体说，在手写数字图上训练的 DBN（可能、并小心地）也能用来生成看似手写数字的图——即 DBN 在某种意义上学"写字"。这点上生成模型很像人脑：不仅能读数字，还能写。用 Geoffrey Hinton 令人难忘的话，[要识别形状，先学生成图像](http://www.sciencedirect.com/science/article/pii/S0079612306650346)。

DBN 有趣的第二个原因是能做无监督与半监督学习。例如用图像数据训练时，即便训练图未标注，DBN 也能学习理解其他图像有用的特征。做无监督学习的能力对根本科学原因与——若能做得够好——实际应用都极有趣。

既有这些诱人特性，DBN 为何作为深度学习模型流行度下降？部分原因是前馈与循环网络取得许多耀眼结果（如图像与语音基准的突破），这些模型获得大量关注不足为奇且很对。但有不幸推论：思想市场常赢者通吃，几乎全部注意力都流向当下时尚。即便想法显然有真实长期价值，人们研究暂不时髦的想法也变得极难。我个人观点是 DBN 与其他生成模型多半该受比当前更多的关注；DBN 或相关模型有朝一日超越当前时尚模型，我不会惊讶。DBN 入门见[此综述](http://www.scholarpedia.org/article/Deep_belief_networks)；我也觉得[此文](http://www.cs.toronto.edu/~hinton/absps/guideTR.pdf)有用——它主要不是讲 DBN，但含大量关于受限玻尔兹曼机（DBN 关键组件）的有用信息。

**其他想法：** 神经网络与深度学习还有什么？有大量其他迷人工作。活跃研究领域包括用神经网络做[自然语言处理](http://machinelearning.org/archive/icml2008/papers/391.pdf)（也见[此信息丰富综述](http://arxiv.org/abs/1103.0398)）、[机器翻译](assets/MachineTranslation.pdf)，以及也许更意外的应用如[音乐信息学](http://yann.lecun.com/exdb/publis/pdf/humphrey-jiis-13.pdf)。当然还有许多其他领域。多数情况下读完本书你应能开始跟进近期工作（当然需补预设背景知识缺口）。

让我提一篇特别有趣的论文结束本节。它结合深度卷积网络与叫强化学习的技术，学会[玩好电子游戏](http://www.cs.toronto.edu/~vmnih/docs/dqn.pdf)（也见[此后续](http://www.nature.com/nature/journal/v518/n7540/abs/nature14236.html)）。想法是用卷积网络简化游戏画面像素数据成更简单特征集、用来决定采取什么行动——"左走""下走""开火"等。特别有趣的是单个网络学会玩 7 个不同经典游戏玩得不错、在 3 个游戏上胜过人类专家。这听着像噱头，无疑论文营销好（标题 "Playing Atari with reinforcement learning"）。但撇开表面光泽，想想这系统取原始像素数据——甚至不知游戏规则！——并从中学习在几个非常不同、非常对抗、各有复杂规则的环境做高质量决策。相当不错。

### 论神经网络的未来

**意图驱动用户界面：** 有个老笑话：不耐烦的教授告诉困惑的学生："别听我说什么；听我*意思*什么。"历史上电脑常像困惑学生、不明白用户意思。但这在改变。我仍记得第一次拼错 Google 搜索词、Google 说 "Did you mean \[纠正的查询\]?" 并提供相应搜索结果时的惊讶。Google CEO Larry Page [曾把完美搜索引擎描述为：完全理解你的\[查询\]意思、并给你完全想要的](http://googleblog.blogspot.ca/2012/08/building-search-engine-of-future-one.html)。

这是*意图驱动用户界面*的愿景：搜索用机器学习取模糊用户输入、辨明精确意图、据此行动，而非响应用户字面查询。

意图驱动界面想法可比搜索应用广得多。未来几十年，成千上万公司会建用机器学习做能容忍不精确、同时辨明并作用于用户真实意图的用户界面。已见这类意图驱动界面的早期例子：Apple 的 Siri、Wolfram Alpha、IBM 的 Watson、能[标注照片与视频](http://arxiv.org/abs/1411.4555)的系统等等。

这些产品多数会失败——好的界面设计难，我预期许多公司会用强大机器学习技术建平庸用户界面。世界上最好的机器学习在你界面概念很烂时也帮不上。但会有成功产品留存，随时间深刻改变我们与电脑的关系。不久前的 2005 年，用户默认多数与电脑交互需要精确；确实，计算机素养很大程度上意味着内化"电脑极其字面"——一个放错的分号可能完全改变与电脑交互的性质。但未来几十年我预期会发展许多成功的意图驱动界面，深刻改变我们与电脑交互时的期望。

**机器学习、数据科学与创新良性循环：** 当然，机器学习不只用于建意图驱动界面。另一显著应用在数据科学——用机器学习找数据中隐藏的"已知未知"。这已是时尚领域、写得很多，所以不多说。但想提这时尚一个不常被说的后果：长期看机器学习最大突破也许不是任何单一概念突破，而是机器学习研究通过数据科学等应用变得*可盈利*。若公司投 1 美元机器学习研究能较快拿回 1 美元 10 美分，大量钱会进入机器学习研究。换言之，机器学习是驱动多个新市场与科技增长领域创造的引擎——结果是拥有深度领域专长与非凡资源的大团队，把机器学习推得更远、创造更多市场与机会，形成创新良性循环。

**神经网络与深度学习的角色：** 我一直在宽泛谈机器学习作为科技新机会的创造者。神经网络与深度学习在其中具体角色是什么？

为回答，看历史有帮助。1980 年代神经网络有大量兴奋与乐观，尤其反向传播广为人知后。那兴奋消退，1990 年代机器学习接力棒传给支持向量机等其他技术。今天神经网络再次高飞、创各种纪录、在许多问题上击败所有挑战者。但谁说明天不会发展出新方法再次扫走神经网络？或者神经网络进展停滞、一时没有东西顶上？

因此，宽泛思考机器学习未来比特定思考神经网络容易得多。部分问题是我们对神经网络理解很差：为何神经网络能泛化这么好？参数那么多、为何能避免过拟合？为何 SGD 工作得这么好？数据集扩大时神经网络表现如何——若 ImageNet 扩 10 倍，神经网络性能改善会比别的机器学习技术多还是少？这些都是简单根本问题，而目前我们对答案理解很差。既如此，很难说神经网络在机器学习未来扮演什么角色。

我做一预测：**相信深度学习会留下**。学习概念层级、搭建多层抽象的能力似乎对理解世界是根本的。这不意味明天的深度学习不会与今天截然不同——组成单元、架构或学习算法可能大变，变化大到我们不再把结果系统想成神经网络；但它们仍是在做深度学习。

**神经网络与深度学习会很快通向人工智能吗？** 本书我们聚焦用神经网络做具体任务（如分类图像）。扩大抱负问：通用思维电脑呢？神经网络与深度学习能帮我们解决（一般）人工智能（AI）问题吗？若能，鉴于深度学习近期快速进展，能期待很快出现一般 AI 吗？

全面回答这些问题要另一本书。取而代之，给一观察——基于叫 [Conway 定律](http://en.wikipedia.org/wiki/Conway%27s_law)的想法：

> 设计系统的任何组织……都不可避免地产生其结构是该组织沟通结构之复制的设计。

例如 Conway 定律提示波音 747 的设计会镜像波音及承包商设计 747 时的扩展组织结构。简单具体例子：建复杂软件应用的公司——若应用仪表盘该与某机器学习算法集成，建仪表盘的人最好在与公司机器学习专家沟通。Conway 定律只是把这观察放大。

初听 Conway 定律，许多人回应"这不平凡显然吗？"或"这不错吗？"先处理"错"的反对：作为例子，问波音会计部门出现在 747 设计哪里？清洁部门？内部餐饮？答案是组织这些部分多半不显式出现在 747 任何地方——所以应理解 Conway 定律只指组织明确关注设计与工程的部分。

"平凡显然"的反对呢？也许对，但我不认为——因为组织太常无视 Conway 定律行事。建新产品的团队常塞满遗留雇员、或反之缺某个关键专长的人。想想所有带无用复杂功能的产品、或带明显重大缺陷（如可怕用户界面）的产品——两类问题常由"产出好产品所需团队与实际组装的团队"错配造成。Conway 定律也许显然，但不意味人们不例行无视它。

Conway 定律适用于我们一开始对可能组成部件及如何构建有相当好理解的系统设计与工程。它不能直接用于 AI 发展，因为 AI 还（尚）不是这种问题：我们不知组成部件是什么——甚至不确定该问什么基本问题。换言之，此刻 AI 更是科学问题而非工程问题。想象不知喷气发动机或空气动力学原理就开始设计 747——你不知道该雇什么专家。如 Wernher von Braun 所说，"基础研究就是我在不知道自己做什么时做的事"。有没有 Conway 定律版本适用于"更多科学而非工程"的问题？

为洞悉此问题，看医学史。早期医学是 Galen、Hippocrates 等从业者的领域——研究整个身体。但知识增长迫人专精。我们发现了许多深刻新想法*。*（抱歉过度使用"深"。我不精确定义"深刻想法"，大致指作为丰富研究领域基础的想法——反向传播算法与疾病细菌学说都是好例子。）*——疾病细菌学说、抗体如何工作的理解、心肺静脉动脉构成完整心血管系统的理解。这些深刻洞见形成流行病学、免疫学、心血管周围互联领域群等子领域的基础。于是我们的知识结构塑造了医学的社会结构——这在免疫学尤其惊人：意识到免疫系统存在且值得研究是极不平凡的洞见，于是整个医学领域——专家、会议、甚至奖项——围绕一个不仅不可见、甚至可说根本不是一个独立事物的东西组织。

这是许多成熟科学反复出现的普遍模式：不只医学，物理、数学、化学等也如此。领域开始是整体的、只有几个深刻想法，早期专家能掌握全部；随时间整体性改变——发现太多新深刻想法、多到任何一人无法真正掌握，结果领域社会结构围绕想法重组分化。不再是整体，而是领域里的领域里的领域——复杂、递归、自我指涉的社会结构，其组织镜像我们最深刻洞见间的联系。*于是知识结构塑造科学的社会组织；但那社会形态反过来约束并帮助决定我们能发现什么。*这是 Conway 定律的科学类比。

这与深度学习或 AI 何干？

自 AI 早期就有这类争论，一边说"嘿，不会那么难，我们有\[超级武器\]"，另一边说"\[超级武器\]不够"。深度学习是我听到的用于这种争论的最新超级武器*。*（有趣的是常不是深度学习顶尖专家——他们相当克制；见 Yann LeCun [这则深思的帖子](https://www.facebook.com/yann.lecun/posts/10152348155137143)。这与该论证许多早期化身不同。）*；论证早期版本用逻辑、Prolog、专家系统或当时最强技术。这论证的问题：它不给你任何好方法说任何给定候选超级武器到底多强。当然我们刚花一章回顾深度学习能解决极难问题的证据，它看着确非常激动、有前途——但 Prolog、[Eurisko](http://en.wikipedia.org/wiki/Eurisko) 或专家系统当年也如此。所以"一组想法看着非常有前途"的事实本身没多大意义。怎么判断深度学习是否真与这些早期想法不同？有没有办法度量一组想法多强多有望？Conway 定律提示：作为粗略启发式代理指标，可评估与那些想法关联的社会结构复杂性。

于是有两个问题要问。第一，按此社会复杂性指标，深度学习关联的一组想法有多强？第二，要能建立一般 AI，我们需要多强的理论？

第一问：看今日深度学习——是令人兴奋快节奏但也相对整体的领域。有少数深刻想法与几个主要会议（几个会议间大量重叠）；一篇篇论文利用同一组基本想法：用 SGD（或近亲）优化代价函数。这些想法如此成功太棒了。但还没见到大量发展良好的子领域各自探索自己的深刻想法、把深度学习推向许多方向。所以按社会复杂性指标，深度学习仍是（恕我双关）相当浅的领域——一个人仍可能掌握领域里多数最深刻想法。

第二问：获得 AI 需要多复杂多强的一组想法？当然答案没人确定知道。但在[附录](sai.html)我审视此问题的一些现有证据，结论是：即便相当乐观，建 AI 也要许多许多深刻想法。于是 Conway 定律提示要到那一步，必然看到许多互联学科的出现，带复杂而惊人的结构、镜像我们最深刻洞见的结构。我们在神经网络与深度学习的使用里还没见到这丰富社会结构——因此我相信，**距用深度学习发展一般 AI 至少还有几十年。**
