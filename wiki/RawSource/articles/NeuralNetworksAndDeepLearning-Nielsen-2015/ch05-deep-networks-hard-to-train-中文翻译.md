---
type: raw-book-translation
title: "为什么深度神经网络难以训练？（第5章 中文译本）"
book: Neural Networks and Deep Learning
author: Michael A. Nielsen
source: ch05-deep-networks-hard-to-train.md
created: 2026-09-07
tags: [deep-learning,neural-networks,book,michael-nielsen,vanishing-gradient,translation]
summary: Nielsen《Neural Networks and Deep Learning》第5章中文学习译本：深度网络难训的根源——梯度消失/爆炸/不稳定问题；推导 ∂C/∂b₁ 连乘式、|wσ'(z)|<1/4 的由来、单神经元链到多层网络的反向传播连乘。英文原章见 [[ch05-deep-networks-hard-to-train]]。
---

# 为什么深度神经网络难以训练？（第5章 中文译本）

> [!info] 译本说明
> 本文件是 Michael A. Nielsen《Neural Networks and Deep Learning》(2015) 第 5 章的中文学习译本。正文忠实传达原文论述，关键术语保留英文对照；公式用 LaTeX，方程编号与英文原章一致；图片为本地 `images/` 相对引用。英文原章见 [[ch05-deep-networks-hard-to-train]]。
>
> 与前几章衔接：Ch.3 讨论"输出层饱和 + 交叉熵"，本章揭示**隐藏层**同类问题的深层形态——梯度消失/爆炸。Ch.6 将给出实际解法。

想象你是个被要求从零设计一台计算机的工程师。某天你在办公室设计逻辑电路，摆弄 `AND` 门、`OR` 门等，老板带着坏消息走进来：客户刚加了个惊人的设计要求——整台计算机的电路只能**两层深**：

![](images/shallow_circuit.png)

你目瞪口呆，对老板说："客户疯了！"

老板回答："我也觉得他们疯了。但客户想要什么，就得到什么。"

事实上，在有限意义上客户并不疯。假设允许用特殊的逻辑门，能把任意多个输入 `AND` 在一起；也允许多输入 `NAND` 门——能对多输入 `AND` 再取反的门。有这些特殊门，事实证明可用仅两层深的电路计算任何函数。

但"可能"不等于"好主意"。实践中解决电路设计问题（或大多数算法问题）时，我们通常先弄清如何解子问题、再逐步整合解——即通过**多层抽象**搭建出解。

例如，假设设计一个两数相乘的逻辑电路。多半想用做"两数相加"之类操作的子电路搭起来；两数相加的子电路又由"两位相加"的子子电路搭成。粗略说电路长这样：

![](images/circuit_multiplication.png)

即最终电路至少含三层电路元件。实际上很可能超过三层，因为我们会把子任务拆得比我描述的更细。但你懂大意。

所以**深电路让设计过程更容易**。但它们不只对设计有帮助。事实上，有数学证明表明：对某些函数，很浅的电路需要指数级更多的电路元件来计算，远超深电路。例如 1980 年代初一系列著名论文*。*（历史相当复杂，我不给详细引用。见 Johan Håstad 2012 年论文 [On the correlation of parity and small-depth circuits](http://eccc.hpi-web.de/report/2012/137/) 对早期历史的叙述与引用。）*证明：计算一组比特的奇偶性若用浅电路需指数级多的门。反之用更深电路则容易用小电路算奇偶性——只需算成对比特的奇偶性、再用那些结果算成对成对比特的奇偶性，依此类推，快速搭建出整体奇偶性。**深电路因此可能本质上比浅电路强大得多。**

至今本书像那个疯客户一样对待神经网络：我们用的几乎所有网络都只有单隐藏层神经元（外加输入输出层）：

![](images/tikz35.png)

这些简单网络出奇有用——前面章节我们用这种网络以超过 98% 准确率分类手写数字！不过直觉上我们预期隐藏层多得多的网络更强大：

![](images/tikz36.png)

这种网络能用中间层搭建多层抽象，正如布尔电路里做的。例如做视觉模式识别时，第一层神经元可能学识别边缘，第二层学识别由边缘组成的更复杂形状（如三角形、矩形），第三层再识别更复杂形状，依此类推。这些多层抽象似乎很可能给深度网络学习解决复杂模式识别问题带来诱人优势。而且，正如电路情形，有理论结果表明深度网络本质上比浅层网络更强大*。*（对某些问题与网络架构，Razvan Pascanu、Guido Montúfar 与 Yoshua Bengio (2014) 的 [On the number of response regions of deep feed forward networks with piece-wise linear activations](http://arxiv.org/pdf/1312.6098.pdf) 证明了这点。另见 Yoshua Bengio (2009) [Learning deep architectures for AI](http://www.iro.umontreal.ca/~bengioy/papers/ftml_book.pdf) 第 2 节较非正式的讨论。）*

怎么训练这种深度网络？本章我们试着用主力学习算法——[随机梯度下降](ch01-handwritten-digits-中文翻译.md#_用梯度下降学习) + [反向传播](ch02-backpropagation-中文翻译.md)——训练深度网络。但会遇到麻烦：深度网络的表现（若有）不比浅层网络好多少。

鉴于上面讨论，这失败似乎惊人。我们不放弃深度网络，而是深挖、试着理解什么让深度网络难训。凑近看会发现：**深度网络的不同层以悬殊速度学习**。特别是，当网络较后层学得好时，较前层常在训练中卡住、几乎学不到任何东西。这种卡住不只是运气不好——会发现学习变慢有根本原因，与我们用基于梯度的学习技术有关。

更深地探究会发现相反现象也可能发生：前层学得好、后层卡住。事实上会发现：深层多隐层神经网络里用梯度下降学习有**内在不稳定性**。这种不稳定倾向让训练中要么前层、要么后层卡住。

这一切听着像坏消息。但深究这些困难，我们开始洞悉有效训练深度网络需要什么。因此这些调查是下一章的好准备——下一章将用深度学习攻图像识别问题。

### 梯度消失问题

那么，训练深度网络时到底哪里出错了？

为回答此问题，先重访单隐藏层网络情形。照例用 MNIST 数字分类问题作为学习与实验的游乐场*。*（我在[这里](ch01-handwritten-digits-中文翻译.md#_用梯度下降学习)与[这里](ch01-handwritten-digits-中文翻译.md#实现我们的数字分类网络)介绍 MNIST 问题与数据。）*

若愿意，可在你电脑上训练网络跟着做；当然只读也行。想跟做需 Python 2.7、Numpy 与代码副本——可用命令行克隆相关仓库获取：

```
git clone https://github.com/mnielsen/neural-networks-and-deep-learning.git
```

不用 git 可在此[下载数据与代码](https://github.com/mnielsen/neural-networks-and-deep-learning/archive/master.zip)。需进入 src 子目录。

然后在 Python shell 加载 MNIST 数据：

```
>>> import mnist_loader
>>> training_data, validation_data, test_data = \
... mnist_loader.load_data_wrapper()
```

建立网络：

```
>>> import network2
>>> net = network2.Network([784, 30, 10])
```

此网络输入层 784 神经元，对应输入图的 $28\times28=784$ 像素；30 隐藏神经元；10 输出神经元，对应 MNIST 数字的 10 种分类（'0','1','2',$\ldots$,'9'）。

试着训练网络 30 个完整回合，每次 mini-batch 10 个训练样本、学习率 $\eta=0.1$、正则化参数 $\lambda=5.0$。训练时监控 validation_data 分类准确率*。*（注意网络训练可能要几分钟，取决于机器速度。若在跑代码，可继续阅读稍后回看，不必等代码跑完。）*：

```
>>> net.SGD(training_data, 30, 10, 0.1, lmbda=5.0,
... evaluation_data=validation_data, monitor_evaluation_accuracy=True)
```

得到 96.48% 分类准确率（左右——每次运行略异），与早先类似配置结果相当。

现在加一个同样 30 神经元的隐藏层，用相同超参训练：

```
>>> net = network2.Network([784, 30, 30, 10])
>>> net.SGD(training_data, 30, 10, 0.1, lmbda=5.0,
... evaluation_data=validation_data, monitor_evaluation_accuracy=True)
```

得到改进的分类准确率 **96.90%**。令人鼓舞：多一点深度有帮助。再加一个 30 神经元隐藏层：

```
>>> net = network2.Network([784, 30, 30, 30, 10])
>>> net.SGD(training_data, 30, 10, 0.1, lmbda=5.0,
... evaluation_data=validation_data, monitor_evaluation_accuracy=True)
```

完全没帮助。事实上结果掉回 96.57%，接近最初浅层网络。假设再插一个隐藏层：

```
>>> net = network2.Network([784, 30, 30, 30, 30, 10])
>>> net.SGD(training_data, 30, 10, 0.1, lmbda=5.0,
... evaluation_data=validation_data, monitor_evaluation_accuracy=True)
```

分类准确率又降到 96.53%。这大概不是统计显著下降，但也不鼓舞。

这行为似乎奇怪。直觉上，额外隐藏层该让网络能学更复杂的分类函数、从而更好分类。当然事情不该变差——额外层最坏情况下可什么都不做*。*（见[后面这个问题](#identity_neuron)理解如何构造什么都不做的隐藏层。）*但事实并非如此。

那怎么回事？假设额外隐藏层原则上确实能帮忙、问题是学习算法没找到正确权重偏置。我们想弄清学习算法哪里出错、如何做得更好。

为洞悉什么出错了，可视化网络如何学习。下面我画了 $[784,30,30,10]$ 网络（两个隐藏层各 30 神经元）的一部分。图中每个神经元上有小条，表示网络学习时该神经元变化多快：大条 = 权重与偏置快速变化，小条 = 变化缓慢。更精确地，条表示每个神经元的梯度 $\partial C/\partial b$——代价对神经元偏置的变化率。早在[第 2 章](ch02-backpropagation-中文翻译.md#反向传播的四个基本方程)我们见过此梯度量不仅控制学习中偏置变化多快、也控制输入到该神经元的权重变化多快。不记得细节也没关系：只需记住这些条显示网络学习时每个神经元权重偏置变化多快。

为让图简单，我只显示两个隐藏层顶部六个神经元，省略输入神经元（它们没有要学的权重偏置）与输出神经元（我们做逐层比较，比较同神经元数的层最合理）。结果在训练最开始——网络刚初始化后——画出。如下*。*（图数据由程序 [generate_gradient.py](https://github.com/mnielsen/neural-networks-and-deep-learning/blob/master/fig/generate_gradient.py) 生成，本节后面引用的结果也用它。）*：

[图：两个隐藏层的学习速度条（初始化后）]

网络随机初始化，所以神经元学习速度差异大不奇怪。但仍有一处跳出来：第二隐藏层的条大多比第一隐藏层大得多。结果第二隐藏层神经元会比第一层学得快不少。这是巧合，还是第二隐藏层神经元一般会比第一层学得快？

为判断是否如此，需要有全局方式比较第一、二隐藏层的学习速度。为此把梯度记为 $\delta^l_j=\partial C/\partial b^l_j$——第 $l$ 层第 $j$ 个神经元的梯度*。*（[第 2 章](ch02-backpropagation-中文翻译.md#反向传播的四个基本方程)我们称之为误差，这里采用非正式称呼"梯度"。说"非正式"因为它当然不显式含代价对权重的偏导 $\partial C/\partial w$。）*可把梯度 $\delta^1$ 想成元素决定第一隐藏层学习多快的向量，$\delta^2$ 同理。然后用这些向量的长度作为（粗略的）层学习速度的全局度量：$\|\delta^1\|$ 测第一隐藏层学习速度，$\|\delta^2\|$ 测第二层。

有了这些定义、在与上面相同的配置下，发现 $\|\delta^1\|=0.07\ldots$、$\|\delta^2\|=0.31\ldots$。这印证了早先怀疑：第二隐藏层神经元确实比第一层学得快很多。

加更多隐藏层会怎样？若三个隐藏层（$[784,30,30,30,10]$ 网络），各层学习速度分别是 0.012、0.060、0.283。又是前层比后层慢得多。假设再加一个 30 隐藏神经元层：各层学习速度 0.003、0.017、0.070、0.285。模式成立：**前层学得比后层慢。**

我们一直在看训练开始（刚初始化后）的学习速度。训练过程中学习速度如何变化？回到只有两个隐藏层的网络，学习速度变化如下：

![](images/training_speed_2_layers.png)

为生成这些结果，我用批量梯度下降、只 1,000 张训练图、训练 500 回合。这与我们通常训练方式略不同——没用 mini-batch、只 1,000 图而非完整 50,000 训练集。我不是想耍花招或遮你眼——只是 mini-batch SGD 给更嘈杂的结果（平均掉噪声后很相似）。用我选的参数是平滑结果的简易方式，好让我们看清发生了什么。

总之，如你所见两层开始以悬殊速度学习（已知），然后两层速度都很快下降、再反弹。但自始至终，第一隐藏层比第二隐藏层慢得多。

更复杂的网络呢？这是类似实验的结果，这次三个隐藏层（$[784,30,30,30,10]$ 网络）：

![](images/training_speed_3_layers.png)

又是前层比后层慢得多。最后加第四隐藏层（$[784,30,30,30,30,10]$ 网络），看训练时发生什么：

![](images/training_speed_4_layers.png)

前层仍比后层慢得多。此情形第一隐藏层比最后隐藏层慢约 100 倍。难怪我们早先训练这些网络有麻烦！

这里有个重要观察：**至少在某些深度神经网络里，梯度随我们向后穿过隐藏层而变小。** 这意味着前层神经元比后层学得慢得多。虽然我们只在单个网络看到这点，许多神经网络发生此事有根本原因。此现象叫*梯度消失问题（vanishing gradient problem）*。*（见 Sepp Hochreiter、Yoshua Bengio、Paolo Frasconi 与 Jürgen Schmidhuber (2001) 的 [Gradient flow in recurrent nets: the difficulty of learning long-term dependencies](http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.24.7321)。该文研究循环网络，但本质现象与我们研究的前馈网络相同。另见 Sepp Hochreiter 更早的毕业论文 [Untersuchungen zu dynamischen neuronalen Netzen](http://www.idsia.ch/~juergen/SeppHochreiter1991ThesisAdvisorSchmidhuber.pdf) (1991，德文)。）*

为何发生梯度消失？有办法避免吗？训练深度神经网络时该如何应对？其实我们很快就会学到它并非不可避免，虽然替代也不太好——有时前层梯度会大得多！这是*梯度爆炸问题（exploding gradient problem）*，不比梯度消失好多少。更一般地，深度神经网络的梯度*不稳定*——倾向在前层爆炸或消失。这种不稳定是深度神经网络基于梯度学习的根本问题，是需要理解、可能的话采取措施应对的东西。

对消失（或不稳定）梯度的一种回应是怀疑它们是否真是问题。暂离神经网络，想象要数值最小化单变量函数 $f(x)$。若导数 $f'(x)$ 小，岂非好消息？岂非意味着已接近极值？类似地，深度网络前层梯度小也许意味着我们不需要对权重偏置做多少调整？

当然，事实并非如此。回想我们随机初始化网络权重偏置——初始权重偏置极不可能在我们想让它做的事上做得好。具体些，考虑 MNIST 问题 $[784,30,30,30,10]$ 网络的第一层权重：随机初始化意味着第一层丢掉输入图像的大部分信息。即便后层被充分训练，它们仍会发现识别输入图像极其困难——只因信息不够。所以"第一层不需要多少学习"绝不可能是事实。要训练深度网络，必须弄清如何应对梯度消失问题。

### 什么导致梯度消失问题？深度神经网络的不稳定梯度

为洞悉梯度消失为何发生，考虑最简单的深度神经网络：每层只有一个神经元。下面是三个隐藏层的网络：

![](images/tikz37.png)

$w_1,w_2,\ldots$ 是权重，$b_1,b_2,\ldots$ 是偏置，$C$ 是某代价函数。提醒工作原理：第 $j$ 个神经元输出 $a_j=\sigma(z_j)$，其中 $\sigma$ 是通常的 [sigmoid 激活函数](ch01-handwritten-digits-中文翻译.md#sigmoid-神经元)，$z_j=w_ja_{j-1}+b_j$ 是神经元的加权输入。我把代价 $C$ 画在末尾，强调代价是网络输出 $a_4$ 的函数：若网络实际输出接近期望输出则代价低，远离则代价高。

我们将研究第一个隐藏神经元关联的梯度 $\partial C/\partial b_1$。推出 $\partial C/\partial b_1$ 的表达式、研究它，从而理解梯度消失为何发生。

先直接展示 $\partial C/\partial b_1$ 的表达式。它看着吓人，其实结构简单，我马上描述。表达式如下（暂时忽略网络，注意 $\sigma'$ 是 $\sigma$ 函数的导数）：

![](images/tikz38.png)

表达式结构如下：网络中每个神经元对应乘积里一个 $\sigma'(z_j)$ 项；每个权重对应一个 $w_j$ 项；末尾一个 $\partial C/\partial a_4$ 项对应末端代价函数。注意我把每项放在网络相应部分上方——**网络本身就是表达式的助记符**。

你可以直接接受此表达式、跳到[它如何关联梯度消失问题的讨论](#discussion_why)。这样做无害——它是我们[早先反向传播讨论](ch02-backpropagation-中文翻译.md#反向传播的四个基本方程)的特例。但表达式为真也有简单解释，看看那解释有趣（也许有启发性）。

想象对偏置 $b_1$ 做小改动 $\Delta b_1$。这会在网络其余部分引发一连串级联变化。首先它引起第一个隐藏神经元输出变化 $\Delta a_1$；那又引起第二个隐藏神经元加权输入变化 $\Delta z_2$；然后第二个隐藏神经元输出变化 $\Delta a_2$……一路到输出端代价变化 $\Delta C$。我们有：

$$
\begin{eqnarray} \frac{\partial C}{\partial b_1} \approx \frac{\Delta C}{\Delta b_1}. \tag{114}\end{eqnarray}
$$

这提示我们可仔细追踪级联中每一步的效果来推出梯度 $\partial C/\partial b_1$ 的表达式。

为此想想 $\Delta b_1$ 如何引起第一个隐藏神经元输出 $a_1$ 变化。$a_1=\sigma(z_1)=\sigma(w_1a_0+b_1)$，所以：

$$
\begin{eqnarray} \Delta a_1 & \approx & \frac{\partial \sigma(w_1 a_0+b_1)}{\partial b_1} \Delta b_1 \tag{115}\\ & = & \sigma'(z_1) \Delta b_1. \tag{116}\end{eqnarray}
$$

那个 $\sigma'(z_1)$ 项应该眼熟：它是我们声称的梯度 $\partial C/\partial b_1$ 表达式中第一项。直觉上此项把偏置变化 $\Delta b_1$ 转成输出激活变化 $\Delta a_1$。变化 $\Delta a_1$ 又引起第二个隐藏神经元加权输入 $z_2=w_2a_1+b_2$ 的变化：

$$
\begin{eqnarray} \Delta z_2 & \approx & \frac{\partial z_2}{\partial a_1} \Delta a_1 \tag{117}\\ & = & w_2 \Delta a_1. \tag{118}\end{eqnarray}
$$

结合 $\Delta z_2$ 与 $\Delta a_1$ 的表达式，我们看到偏置 $b_1$ 的变化如何沿网络传播影响 $z_2$：

$$
\begin{eqnarray} \Delta z_2 & \approx & \sigma'(z_1) w_2 \Delta b_1. \tag{119}\end{eqnarray}
$$

又该眼熟：现在有了声称的梯度 $\partial C/\partial b_1$ 表达式头两项。

可继续这样追踪变化如何穿过网络其余部分传播。每个神经元处拾取一个 $\sigma'(z_j)$ 项，每个权重处拾取一个 $w_j$ 项。最终结果是联系末端代价变化 $\Delta C$ 与初始偏置变化 $\Delta b_1$ 的表达式：

$$
\begin{eqnarray} \Delta C & \approx & \sigma'(z_1) w_2 \sigma'(z_2) \ldots \sigma'(z_4) \frac{\partial C}{\partial a_4} \Delta b_1. \tag{120}\end{eqnarray}
$$

除以 $\Delta b_1$ 确实得到想要的梯度表达式：

$$
\begin{eqnarray} \frac{\partial C}{\partial b_1} = \sigma'(z_1) w_2 \sigma'(z_2) \ldots \sigma'(z_4) \frac{\partial C}{\partial a_4}. \tag{121}\end{eqnarray}
$$

**梯度消失为何发生：** 为理解梯度消失为何发生，把梯度完整表达式显式写出：

$$
\begin{eqnarray} \frac{\partial C}{\partial b_1} = \sigma'(z_1) \, w_2 \sigma'(z_2) \, w_3 \sigma'(z_3) \, w_4 \sigma'(z_4) \, \frac{\partial C}{\partial a_4}. \tag{122}\end{eqnarray}
$$

除最末项外，此表达式是 $w_j\sigma'(z_j)$ 形式项的乘积。为理解每项行为，看 $\sigma'$ 函数图：

[图：σ' 函数，峰值 1/4]

导数在 $\sigma'(0)=1/4$ 处最大。若用我们[标准方法](ch03-improving-learning-中文翻译.md#_权重初始化)初始化网络权重——均值 0、标准差 1 的高斯——权重通常满足 $|w_j|<1$。综合观察：项 $w_j\sigma'(z_j)$ 通常满足 $|w_j\sigma'(z_j)|<1/4$。取许多这样的项乘积时，乘积倾向指数衰减——**项越多，乘积越小**。这开始像梯度消失问题的可能解释了。

把这事说得更显式些：比较 $\partial C/\partial b_1$ 的表达式与对更后偏置（如 $\partial C/\partial b_3$）的梯度表达式。我们当然没显式推出 $\partial C/\partial b_3$ 的表达式，但它遵循上面 $\partial C/\partial b_1$ 描述的同样模式。两表达式对比如下：

![](images/tikz39.png)

两表达式共享许多项。但梯度 $\partial C/\partial b_1$ 多了两个 $w_j\sigma'(z_j)$ 形式项。如所见的，此类项幅度通常小于 $1/4$。于是梯度 $\partial C/\partial b_1$ 通常会比 $\partial C/\partial b_3$ 小因子 16（或更多）。这就是梯度消失问题的本质根源。

当然，这是非正式论证，不是梯度消失必会发生的严格证明。有几个可能逃生条款。特别是，我们可能疑惑训练中权重 $w_j$ 会不会变大。若会，乘积里 $w_j\sigma'(z_j)$ 项可能不再满足 $|w_j\sigma'(z_j)|<1/4$。事实上若项大到超过 1，就不会有梯度消失问题——反而梯度会随向后穿过层而指数增长。**不再是梯度消失问题，而是梯度爆炸问题。**

**梯度爆炸问题：** 看一个梯度爆炸发生的显式例子。例子有点人为：我把网络参数以恰好保证爆炸的方式固定。但即使人为，它有好处——坚定确立爆炸梯度不只是假想可能、真能发生。

引爆梯度有两步。第一，把网络所有权重选大，比如 $w_1=w_2=w_3=w_4=100$。第二，选偏置使 $\sigma'(z_j)$ 项不太小。这其实容易做：只需选偏置确保每个神经元加权输入 $z_j=0$（于是 $\sigma'(z_j)=1/4$）。例如想要 $z_1=w_1a_0+b_1=0$，设 $b_1=-100*a_0$ 即可；同样想法选其他偏置。这样做时所有 $w_j\sigma'(z_j)$ 项都等于 $100\times\frac14=25$。这些选择下得到爆炸梯度。

**不稳定梯度问题：** 这里的根本问题与其说是消失或爆炸梯度，不如说是：**前层梯度是来自所有后层的项的乘积。** 层多时这是本质不稳定局面。所有层以相近速度学习的唯一方式是那些项乘积都接近平衡。没有某种机制或根本原因让平衡发生，纯靠碰巧极不可能。简言之，真正问题是神经网络遭受*不稳定梯度问题*——结果用标准基于梯度学习技术时，网络不同层往往以悬殊速度学习。

#### 练习

- 梯度消失讨论里我们用到 $|\sigma'(z)|<1/4$。假设用导数能大得多的不同激活函数，会有助避免不稳定梯度问题吗？

**梯度消失问题的普遍性：** 我们看到梯度在深度网络前层可能消失或爆炸。事实上用 sigmoid 神经元时梯度通常会消失。为看清为何，再考虑表达式 $|w\sigma'(z)|$。要避免消失需 $|w\sigma'(z)|\geq1$。你也许觉得 $w$ 很大时容易发生——但比看起来难。原因是 $\sigma'(z)$ 项也依赖 $w$：$\sigma'(z)=\sigma'(wa+b)$，其中 $a$ 是输入激活。所以把 $w$ 调大时须小心别同时把 $\sigma'(wa+b)$ 调小。事实证明这是相当大的约束——因为 $w$ 大时 $wa+b$ 往往很大；看 $\sigma'$ 图可见这把我们放到 $\sigma'$ 函数的"翼部"（取值很小处）。唯一避免方式要求输入激活落在相当窄的值域内（这定性解释在下面第一问题里定量化）。有时碰巧发生；但更多时候不发生。所以一般情形会梯度消失。

#### 问题

- 考虑乘积 $|w\sigma'(wa+b)|$。假设 $|w\sigma'(wa+b)|\geq1$。(1) 论证这只有当 $|w|\geq4$ 才可能发生。(2) 假设 $|w|\geq4$，考虑使 $|w\sigma'(wa+b)|\geq1$ 的输入激活 $a$ 集合。证明满足该约束的 $a$ 集合覆盖区间宽度不超过：

$$
\begin{eqnarray} \frac{2}{|w|} \ln\left( \frac{|w|(1+\sqrt{1-4/|w|})}{2}-1\right). \tag{123}\end{eqnarray}
$$

(3) 数值上证明上面约束范围宽度的表达式在 $|w|\approx6.9$ 处最大，取值 $\approx0.45$。所以即便一切都完美排齐，能避免消失问题的输入激活范围仍相当窄。

- **恒等神经元：** 考虑单输入 $x$、对应权重 $w_1$、偏置 $b$、输出权重 $w_2$ 的神经元。证明适当选权重偏置可保证 $x\in[0,1]$ 时 $w_2\sigma(w_1x+b)\approx x$。这种神经元因此可用作一种*恒等神经元*——输出（差一个权重因子重缩放）等于输入。*提示：把 $x=1/2+\Delta$ 重写、假设 $w_1$ 小、用 $w_1\Delta$ 的泰勒级数展开会有帮助。*

### 更复杂网络的不稳定梯度

我们一直在研究每隐藏层单神经元的玩具网络。每隐藏层许多神经元的更复杂深度网络呢？

![](images/tikz40.png)

事实上这种网络也发生大体相同行为。在反向传播那章我们见过：$L$ 层网络第 $l$ 层的梯度[由下式给出](ch02-backpropagation-中文翻译.md#_反向传播方程的另一种呈现)：

$$
\begin{eqnarray} \delta^l = \Sigma'(z^l) (w^{l+1})^T \Sigma'(z^{l+1}) (w^{l+2})^T \ldots \Sigma'(z^L) \nabla_a C \tag{124}\end{eqnarray}
$$

$\Sigma'(z^l)$ 是对角矩阵，元素是第 $l$ 层加权输入的 $\sigma'(z)$ 值；$w^l$ 是各层权重矩阵；$\nabla_a C$ 是 $C$ 对输出激活的偏导向量。

这比单神经元情形的表达式复杂得多。但凑近看，本质形式很相似——大量 $(w^j)^T\Sigma'(z^j)$ 形式的对。而且矩阵 $\Sigma'(z^j)$ 对角线元素小、都不超过 $\frac14$。只要权重矩阵 $w^j$ 不太大，每个额外项 $(w^j)^T\Sigma'(z^l)$ 都倾向让梯度向量更小，导致梯度消失。更一般地，乘积里大量项倾向导致不稳定梯度，正如早先例子。实践中，经验上常发现 sigmoid 网络中梯度在前层指数级快速消失——结果那些层学习变慢。这变慢不只是意外或不便：它是我们采取的学习方法的根本后果。

### 深度学习的其他障碍

本章聚焦梯度消失——更一般地，不稳定梯度——作为深度学习的障碍。事实上不稳定梯度只是深度学习的一个障碍，虽是很重要的根本障碍。大量进行中研究旨在更好理解训练深度网络时可能出现的挑战。我不全面总结那些工作，只想简略提几篇论文，让你感受人们在问的问题的味道。

第一个例子，2010 年 Glorot 与 Bengio*。*（Xavier Glorot 与 Yoshua Bengio (2010) 的 [Understanding the difficulty of training deep feedforward neural networks](http://jmlr.org/proceedings/papers/v9/glorot10a/glorot10a.pdf)。另见 Yann LeCun、Léon Bottou、Genevieve Orr 与 Klaus-Robert Müller (1998) [Efficient BackProp](http://yann.lecun.com/exdb/publis/pdf/lecun-98b.pdf) 中更早讨论的 sigmoid 用法。）*发现证据提示 sigmoid 激活函数的使用会给训练深度网络带来问题。特别是他们发现 sigmoid 的使用会让最后隐藏层激活在训练早期饱和在 0 附近、大幅拖慢学习。他们建议了一些替代激活函数，似乎不那么受此饱和问题影响。

第二个例子，2013 年 Sutskever、Martens、Dahl 与 Hinton*。*（Ilya Sutskever、James Martens、George Dahl 与 Geoffrey Hinton (2013) 的 [On the importance of initialization and momentum in deep learning](http://www.cs.toronto.edu/~hinton/absps/momentum.pdf)。）*研究随机权重初始化与基于动量 SGD 中动量计划对深度学习的影响。两种情况中，做好选择都对训练深度网络能力有实质影响。

这些例子提示"什么让深度网络难训？"是复杂问题。本章聚焦深度网络中基于梯度学习关联的不稳定性。上两段结果提示：激活函数选择、权重初始化方式、甚至梯度下降学习的实现细节都起作用。当然，网络架构选择与其他超参也重要。所以许多因素都可能让深度网络难训，理解所有这些因素仍是在进行的研究课题。这一切听着相当悲观。但好消息是下一章我们将扭转局面，发展几种深度学习方法来一定程度克服或绕开所有这些挑战。
