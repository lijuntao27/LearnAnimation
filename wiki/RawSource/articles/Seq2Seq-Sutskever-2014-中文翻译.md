---
type: translation
title: Sequence to Sequence Learning with Neural Networks (2014) 中文译本
source: RawSource/articles/Seq2Seq-Sutskever-2014.pdf
created: 2026-09-10
updated: 2026-09-10
tags: [deep-learning, rnn, lstm, encoder-decoder, seq2seq, neurips-2014, translation]
summary: Sutskever 等 2014 NeurIPS 原论文的中文学习译本：RNN Encoder-Decoder 用定长向量桥接输入输出、LSTM 自回归生成，倒转源句是关键 trick。忠实传达原文，公式 LaTeX、结果表 Markdown 渲染。原始英文见 [[Seq2Seq-Sutskever-2014]]。
---

# Sequence to Sequence Learning with Neural Networks — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），两张结果表用 Markdown 表格渲染。正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[Seq2Seq-Sutskever-2014|原索引卡]] 所指的 `Seq2Seq-Sutskever-2014.txt`。
>
> 论文为 arXiv 开放获取（1409.3215），本译本按学术学习使用处理。表 3 中的法语例句在抽取文本里存在重音符号乱码，本译本已按法语原文补正。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Sequence to Sequence Learning with Neural Networks |
| 作者 | Ilya Sutskever, Oriol Vinyals, Quoc V. Le |
| 发表 | NeurIPS 2014（第 28 届神经信息处理系统大会） |
| arXiv | 1409.3215 |
| 机构 | Google |

## 摘要

深度神经网络（DNN）是强大的模型，在许多困难的学习任务上取得了优异表现。尽管只要有大量带标注的训练数据，DNN 就能工作得很好，但它们**无法用于"序列到序列"的映射**。本文提出一种通用的端到端序列学习方法，对序列结构只做最少的假设：我们的方法用一个多层长短期记忆网络（LSTM）把输入序列映射为一个固定维度的向量，再用另一个深层 LSTM 从该向量解码出目标序列。

主要结果是：在 WMT'14 数据集的英法翻译任务上，LSTM 产生的译文在整个测试集上取得 **34.8** 的 BLEU 分数，其中 LSTM 的 BLEU 分数因词表外（out-of-vocabulary，OOV）单词而受到惩罚。此外，LSTM 在长句上没有困难。作为对照，基于短语的 SMT（统计机器翻译）系统在同一数据集上取得 33.3 的 BLEU 分数。当我们用 LSTM 对上述 SMT 系统产生的 1000 个假设做重排序（rerank）时，BLEU 分数提升到 **36.5**，接近该任务此前的最高结果。LSTM 还学到了合理的短语与句子表示，它们对词序敏感，且相对不受主动语态与被动语态的影响。

最后我们发现，把**所有源句**（而非目标句）中单词的顺序颠倒，能显著提升 LSTM 的表现——因为这样做在源句与目标句之间引入了许多短程依赖，使优化问题变得更容易。

## 1 引言

深度神经网络（DNN）是极其强大的机器学习模型，在语音识别 `[13, 7]`、视觉目标识别 `[19, 6, 21, 20]` 等困难问题上取得了优异表现。DNN 之所以强大，是因为它能在不多的步骤内完成任意的并行计算。一个令人惊讶的例子是：DNN 只用 2 个规模为平方级的隐层，就能对 N 个 N 位数进行排序 `[27]`。因此，尽管神经网络与传统统计模型相关，它学到的却是一种精巧的计算。此外，只要带标注训练集包含足够信息来确定网络参数，大型 DNN 就能用有监督的反向传播来训练。因此，如果存在一个大型 DNN 的参数设定能取得好的结果（例如因为人类能极快地解决该任务），有监督的反向传播就能找到这些参数并解决问题。

尽管 DNN 灵活而强大，它只能应用于那些"输入与目标都能被合理地编码为固定维度向量"的问题。这是一个显著局限，因为许多重要问题最好用"长度事先未知的序列"来表达。例如，语音识别和机器翻译就是序列问题；同样地，问答也可以看作把"表示问题的词序列"映射为"表示答案的词序列"。因此很明显，一种与领域无关、能学会"序列到序列"映射的方法将很有用。

序列对 DNN 构成挑战，因为它要求输入输出的维度已知且固定。本文表明，**直接应用**长短期记忆网络（LSTM）架构 `[16]` 就能解决通用的序列到序列问题。想法是：用一个 LSTM 一次一个时间步地读入输入序列，得到一个大的固定维度向量表示；再用另一个 LSTM 从该向量中提取出输出序列（图 1）。第二个 LSTM 本质上是一个循环神经网络语言模型 `[28, 23, 30]`，只是它额外以输入序列为条件。LSTM 能成功学习具有长程时序依赖的数据，这一能力使它成为该应用的天然选择——因为输入与对应输出之间存在相当大的时间滞后（图 1）。

已有多项相关工作尝试用神经网络解决通用的序列到序列学习问题。我们的方法与 Kalchbrenner 和 Blunsom `[18]` 密切相关——他们率先把整个输入句映射为向量；也与 Cho 等人 `[5]` 相关，尽管后者只被用于对基于短语的系统产生的假设做重打分。Graves `[10]` 提出了一种新颖的**可微注意力机制**，使神经网络能够聚焦于输入的不同部分；Bahdanau 等人 `[2]` 成功地把这一想法的优雅变体应用于机器翻译。连接主义时序分类（Connectionist Sequence Classification）是另一种用神经网络做序列到序列映射的流行技术，但它假设输入与输出之间存在单调对齐 `[11]`。

> **图 1**：我们的模型读入输入句 "ABC"，输出句为 "WXYZ"。模型在输出句尾标记（end-of-sentence token）后停止预测。注意 LSTM 是**倒序**读入输入句的，因为这样做在数据中引入了许多短程依赖，使优化问题大为简化。

本工作的主要结果如下：在 WMT'14 英法翻译任务上，我们用 5 个深层 LSTM 的集成（每个含 3.84 亿参数、8000 维状态），配合简单的从左到右 beam search 解码器直接提取译文，取得 **34.81** 的 BLEU 分数。这是迄今用大型神经网络直接翻译所取得的最好结果。作为对照，SMT 基线在该数据集上的 BLEU 分数是 33.30 `[29]`。34.81 的 BLEU 分数是由一个仅含 8 万词表的 LSTM 取得的，因此只要参考译文含有这 8 万词之外的词，分数就会受到惩罚。这一结果表明：一个相对未优化、小词表、还有很大改进空间的神经网络架构，已经超越了基于短语的 SMT 系统。

最后，我们用 LSTM 对同一任务上公开可用的 SMT 基线 1000-best 列表做重打分 `[29]`，取得 **36.5** 的 BLEU 分数，比基线提升 3.2 个 BLEU 点，接近该任务此前已发表的最好结果（37.0 `[9]`）。

令人惊讶的是，LSTM 在很长的句子上没有吃力，尽管其他研究者近期用类似架构遇到了困难 `[26]`。我们之所以能在长句上表现良好，是因为我们在训练集和测试集中倒转了源句（而非目标句）的词序。这样做引入了许多短程依赖，使优化问题大为简化（见第 2 节与第 3.3 节）。结果，SGD 能学出对长句毫无困难的 LSTM。**"倒转源句词序"这一简单技巧是本工作的关键技术贡献之一**。

LSTM 的一个有用性质是：它能学会把变长的输入句映射为固定维度的向量表示。鉴于译文往往是源句的转述，翻译目标会促使 LSTM 找到能捕捉句义的句子表示——语义相近的句子彼此靠近，语义不同的句子彼此远离。定性评估支持这一论断：我们的模型能感知词序，且相对不受主动/被动语态影响。

## 2 模型

循环神经网络（RNN）`[31, 28]` 是前馈神经网络向序列的自然推广。给定输入序列 $(x_1, \dots, x_T)$，标准 RNN 通过迭代以下方程计算输出序列 $(y_1, \dots, y_T)$：

$$h_t = \sigma\!\left(W^{hx} x_t + W^{hh} h_{t-1}\right)$$
$$y_t = W^{yh} h_t$$

（其中 $\sigma$ 为 sigmoid，原文记作 `sigm`。）

只要输入与输出的对齐事先已知，RNN 就能轻松完成序列到序列的映射。然而，当输入输出序列长度不同、且关系复杂且非单调时，如何应用 RNN 并不清楚。

通用序列学习最简单的策略是：用一个 RNN 把输入序列映射为固定大小的向量，再用另一个 RNN 把该向量映射为目标序列（Cho 等人 `[5]` 也采用了这一方法）。原则上它或许可行——因为 RNN 获得了所有相关信息——但由于由此产生的长程依赖（图 1），训练 RNN 会很困难 `[14, 4, 16, 15]`。不过，长短期记忆网络（LSTM）`[16]` 以能学习长程时序依赖问题著称，因此 LSTM 或许能在这种设定下成功。

LSTM 的目标是估计条件概率 $p(y_1, \dots, y_{T'} \mid x_1, \dots, x_T)$，其中 $(x_1, \dots, x_T)$ 是输入序列，$y_1, \dots, y_{T'}$ 是相应的输出序列，其长度 $T'$ 可能不同于 $T$。LSTM 计算该条件概率的方式是：先取 LSTM 的**最后一个隐状态**作为输入序列 $(x_1, \dots, x_T)$ 的固定维度表示 $v$，再用标准的 LSTM-LM（语言模型）公式计算 $y_1, \dots, y_{T'}$ 的概率，其初始隐状态被设为 $x_1, \dots, x_T$ 的表示 $v$：

$$p(y_1, \dots, y_{T'} \mid x_1, \dots, x_T) = \prod_{t=1}^{T'} p(y_t \mid v, y_1, \dots, y_{t-1}) \tag{1}$$

在该方程中，每个 $p(y_t \mid v, y_1, \dots, y_{t-1})$ 分布都由词表中所有词上的 softmax 表示。我们采用 Graves `[10]` 的 LSTM 公式。注意我们要求每个句子以特殊的句尾符号 "<EOS>" 结尾，这使模型能够定义所有可能长度序列上的分布。整体方案见图 1：图中 LSTM 先计算 "A"、"B"、"C"、"<EOS>" 的表示，再用该表示计算 "W"、"X"、"Y"、"Z"、"<EOS>" 的概率。

> [!note] 这就是"固定长度瓶颈向量"的出处
> 原文第 2 节把"整个输入句压成最后一个隐状态 $v$"确立为范式核心——无论多长的输入都挤进一个定长向量，长句信息必然丢失。这正是 [[RecurrentNetworks]] §瓶颈问题 所说的"病根"，也是 [[Bahdanau-Attention-2014]] 的注意力机制要治的病：让解码器不再只盯着这一个 $v$，而是直接回看输入的每个位置。

我们的实际模型与上述描述有三点重要不同。**第一**，我们用了两个不同的 LSTM：一个处理输入序列（"编码器"），另一个处理输出序列（"解码器"）——因为这样做在几乎不增加计算开销的情况下增多了模型参数，还使 LSTM 能自然地同时在多个语言对上训练 `[18]`。**第二**，我们发现深层 LSTM 显著优于浅层 LSTM，因此选择了四层的 LSTM。**第三**，我们发现倒转输入句词序极有价值。

例如，不是把句子 a, b, c 映射到句子 α, β, γ，而是让 LSTM 把 c, b, a 映射到 α, β, γ，其中 α, β, γ 是 a, b, c 的译文。这样一来，a 与 α 彼此靠近，b 与 β 相当接近，依此类推——这一事实让 SGD 容易在输入与输出之间"建立通信"。我们发现这种简单的数据变换能大幅提升 LSTM 的性能。

## 3 实验

我们以两种方式把方法应用于 WMT'14 英法机器翻译任务：其一是不借助参考 SMT 系统直接翻译输入句；其二是对 SMT 基线的 n-best 列表做重打分。我们报告这些翻译方法的准确率、给出样例译文，并可视化所得的句子表示。

### 3.1 数据集细节

我们使用 WMT'14 英法数据集，在 1200 万句的子集上训练模型，其中含 3.48 亿法语词与 3.04 亿英语词——这是来自 `[29]` 的一个干净的"精选"子集。我们之所以选择该翻译任务和这个特定的训练子集，是因为它公开提供了分词后的训练集与测试集，以及基线 SMT 的 1000-best 列表 `[29]`。

由于典型的神经语言模型依赖每个词的向量表示，我们对两种语言都采用了固定词表：源语言取最频繁的 16 万个词，目标语言取最频繁的 8 万个词。每个词表外的词都被替换为特殊的 "UNK" 标记。

### 3.2 解码与重打分

实验的核心是在大量句对上训练一个大型深层 LSTM。训练方式是最大化"给定源句 $S$ 时正确译文 $T$ 的对数概率"，因此训练目标为

$$\frac{1}{|\mathcal{S}|} \sum_{(T, S) \in \mathcal{S}} \log p(T \mid S)$$

其中 $\mathcal{S}$ 是训练集。训练完成后，我们通过寻找 LSTM 认为最可能的译文来生成翻译：

$$\hat{T} = \arg\max_{T} p(T \mid S) \tag{2}$$

我们用简单的从左到右 beam search 解码器来搜索最可能的译文，该解码器维护少量 $B$ 个部分假设（partial hypothesis），其中每个部分假设是某个译文的前缀。在每个时间步，我们把 beam 中的每个部分假设用词表中的每个可能单词做扩展。这使假设数量急剧增加，因此我们按模型对数概率只保留最可能的 $B$ 个假设，其余全部丢弃。一旦 "<EOS>" 符号被追加到某个假设，该假设就从 beam 中移除，并加入"完整假设"集合。这个解码器虽然是近似的，但实现简单。有趣的是，即便 beam 大小仅为 1，我们的系统也表现良好；而 beam 大小为 2 时已能提供 beam search 的大部分收益（表 1）。

我们还用 LSTM 对基线系统 `[29]` 产生的 1000-best 列表做重打分。对 n-best 列表重打分时，我们用 LSTM 计算每个假设的对数概率，并将其与原分数取**等权平均**。

### 3.3 倒转源句

虽然 LSTM 能够解决长程依赖问题，但我们发现：当源句被倒转（目标句不倒转）时，LSTM 学得好得多。这样做之后，LSTM 的测试困惑度从 5.8 降到 4.7，其解码译文的测试 BLEU 分数从 25.9 升到 30.6。

虽然我们无法完全解释这一现象，但我们认为它是由"向数据集引入许多短程依赖"导致的。通常，当把源句与目标句拼接在一起时，源句中的每个词都离它在目标句中的对应词很远，因此该问题有较大的"最小时滞"（minimal time lag）`[17]`。倒转源句词序后，源语与目标语中对应词之间的**平均距离不变**；但源语开头几个词现在与目标语开头几个词非常接近，于是问题的**最小时滞大大缩短**。因此，反向传播更容易在源句与目标句之间"建立通信"，进而带来整体性能的显著提升。

> [!note] 倒转源句的本质：缩短最小时滞
> 倒转源句不改变对应词的平均距离，却把反向传播要跨越的"最小时滞"大幅缩短——这是"用数据变换制造短程依赖来软化优化问题"的经典例子。它与 [[RecurrentNetworks]] 里"长程依赖难学（梯度消失/爆炸）"是同一枚硬币的两面：变换数据，让难学的长程依赖变成易学的短程依赖。

起初我们以为，倒转输入句只会让目标句靠前部分的预测更有把握、靠后部分的预测更没把握。然而，在倒转源句上训练的 LSTM 在长句上的表现远好于在原始源句上训练的 LSTM（见 3.7 节），这表明倒转输入句能让 LSTM 更充分地利用记忆。

### 3.4 训练细节

我们发现 LSTM 模型相当容易训练。我们使用 4 层的深层 LSTM，每层 1000 个单元、1000 维词嵌入，输入词表 16 万、输出词表 8 万。因此深层 LSTM 用 8000 个实数来表示一个句子。我们发现深层 LSTM 显著优于浅层 LSTM，每增加一层困惑度就降低近 10%，这可能源于其大得多的隐状态。我们在每个输出处对 8 万词做朴素的 softmax。所得 LSTM 有 3.84 亿参数，其中 6400 万是纯循环连接（"编码器" LSTM 与"解码器" LSTM 各 3200 万）。完整训练细节如下：

- 我们用 $[-0.08, 0.08]$ 之间的均匀分布初始化 LSTM 的所有参数。
- 我们使用不带动量的随机梯度下降（SGD），固定学习率 0.7。5 个 epoch 之后，每半个 epoch 把学习率减半。模型总共训练 7.5 个 epoch。
- 我们用 128 个序列的 batch 计算梯度，并除以 batch 大小（即 128）。
- 尽管 LSTM 一般不遭受梯度消失问题，但可能出现**梯度爆炸**。因此我们对梯度范数施加硬约束 `[10, 25]`：当范数超过阈值时对其缩放。对每个训练 batch，我们计算 $s = \|g\|^2$，其中 $g$ 是除以 128 后的梯度。若 $s > 5$，则令 $g \gets \dfrac{5g}{s}$。
- 不同句子长度不同。多数句子较短（如长度 20–30），少数句子较长（如长度 > 100），因此随机抽取的 128 句 minibatch 会包含许多短句和少量长句，导致 minibatch 中大量计算被浪费。为解决这个问题，我们确保同一 minibatch 中的所有句子长度大致相同，从而获得 2 倍加速。

> [!note] 梯度裁剪 + 等长分桶
> 这两处是原版最早给出、后来成为 RNN 训练标配的工程细节：按范数裁剪梯度（对应 [[TrainingMechanics]] 的梯度裁剪），以及"按长度分桶"避免短句 padding 浪费（本 Wiki 补间论文的数据管线沿用同一思想）。

### 3.5 并行化

采用上一节配置的深层 LSTM 的 C++ 实现，在单块 GPU 上处理速度约为每秒 1700 词。这对我们的目标而言太慢，于是我们用一台 8-GPU 机器对模型做并行化：LSTM 的每一层在不同的 GPU 上运行，激活值一经算出就立即传给下一块 GPU / 下一层。我们的模型有 4 层 LSTM，每层驻留在独立的 GPU 上。其余 4 块 GPU 用于并行化 softmax，每块 GPU 负责乘以一个 $1000 \times 20000$ 的矩阵。最终实现达到了每秒 6300 词（英法合计）的速度，minibatch 大小为 128。用这一实现训练大约花了十天。

### 3.6 实验结果

我们用区分大小写的 BLEU 分数 `[24]` 评估译文质量，并在分词后的预测与真值上用 `multi-bleu.pl` 计算 BLEU 分数。这种评估 BLEU 的方式与 `[5]`、`[2]` 一致，也能复现 `[29]` 的 33.3 分。不过，若用这种方式评估最好的 WMT'14 系统 `[9]`（其预测可从 statmt.org/matrix 下载），我们得到 37.0，高于 statmt.org/matrix 报告的 35.8。

结果见表 1 与表 2。我们最好的结果由 LSTM 集成取得，这些 LSTM 仅在随机初始化与 minibatch 随机顺序上不同。尽管 LSTM 集成的解码译文并未超过最好的 WMT'14 系统，但这是**首次**一个纯神经翻译系统在大规模机器翻译任务上以可观幅度超过基于短语的 SMT 基线——尽管它无法处理词表外单词。若用 LSTM 对基线系统的 1000-best 列表做重打分，则它距最好的 WMT'14 结果只差 0.5 个 BLEU 点。

**表 1：LSTM 在 WMT'14 英法测试集（ntst14）上的表现。** 注意：5 个 LSTM 的集成配 beam 大小 2，比单个 LSTM 配 beam 大小 12 更便宜。

| 方法 | 测试 BLEU 分数 (ntst14) |
|---|---|
| Bahdanau 等人 `[2]` | 28.45 |
| 基线系统 `[29]` | 33.30 |
| 单个正向 LSTM，beam 大小 12 | 26.17 |
| 单个倒转 LSTM，beam 大小 12 | 30.59 |
| 5 个倒转 LSTM 集成，beam 大小 1 | 33.00 |
| 2 个倒转 LSTM 集成，beam 大小 12 | 33.27 |
| 5 个倒转 LSTM 集成，beam 大小 2 | 34.50 |
| 5 个倒转 LSTM 集成，beam 大小 12 | 34.81 |

**表 2：在 WMT'14 英法测试集（ntst14）上，与 SMT 系统结合使用神经网络的方法。**

| 方法 | 测试 BLEU 分数 (ntst14) |
|---|---|
| 基线系统 `[29]` | 33.30 |
| Cho 等人 `[5]` | 34.54 |
| 最好 WMT'14 结果 `[9]` | 37.0 |
| 单个正向 LSTM 重打分基线 1000-best | 35.61 |
| 单个倒转 LSTM 重打分基线 1000-best | 35.85 |
| 5 个倒转 LSTM 集成重打分基线 1000-best | 36.5 |
| 基线 1000-best 列表的 Oracle 重打分 | 45 |

### 3.7 长句上的表现

我们惊讶地发现 LSTM 在长句上表现良好，图 3 定量地展示了这一点。表 3 给出若干长句及其译文的示例。

> **图 3**：左图展示了系统性能随句子长度的变化，x 轴对应"按长度排序的测试句"并标注了实际序列长度。长度小于 35 词的句子没有性能下降，最长的句子上只有轻微下降。右图展示了 LSTM 在"稀有词逐渐增多"的句子上的表现，x 轴对应按"平均词频排名"排序的测试句。

**表 3：LSTM 生成的长译文示例与真值对照。** 读者可用 Google 翻译验证这些译文是否合理。

| 示例 | 我们的模型（法语输出） | 真值（法语） |
|---|---|---|
| 1 | Ulrich UNK, membre du conseil d'administration du constructeur automobile Audi, affirme qu'il s'agit d'une pratique courante depuis des années pour que les téléphones portables puissent être collectés avant les réunions du conseil d'administration afin qu'ils ne soient pas utilisés comme appareils d'écoute à distance. | Ulrich Hackenberg, membre du conseil d'administration du constructeur automobile Audi, déclare que la collecte des téléphones portables avant les réunions du conseil, afin qu'ils ne puissent pas être utilisés comme appareils d'écoute à distance, est une pratique courante depuis des années. |
| 2 | Les téléphones cellulaires, qui sont vraiment une question, non seulement parce qu'ils pourraient potentiellement causer des interférences avec les appareils de navigation, mais nous savons, selon la FCC, qu'ils pourraient interférer avec les tours de téléphone cellulaire lorsqu'ils sont dans l'air, dit UNK. | Les téléphones portables sont véritablement un problème, non seulement parce qu'ils pourraient éventuellement créer des interférences avec les instruments de navigation, mais parce que nous savons, d'après la FCC, qu'ils pourraient perturber les antennes-relais de téléphonie mobile s'ils sont utilisés à bord, a déclaré Rosenker. |
| 3 | Avec la crémation, il y a un « sentiment de violence contre le corps d'un être cher », qui sera « réduit à une pile de cendres » en très peu de temps au lieu d'un processus de décomposition « qui accompagnera les étapes du deuil ». | Il y a, avec la crémation, « une violence faite au corps aimé », qui va être « réduit à un tas de cendres » en très peu de temps, et non après un processus de décomposition, qui « accompagnerait les phases du deuil ». |

### 3.8 模型分析

我们模型的一个吸引人之处是它能把词序列转化为固定维度的向量。图 2 可视化了一些学到的表示。该图清楚地显示：表示对**词序敏感**，同时对"主动语态换成被动语态"相当不敏感。二维投影由 PCA 得到。

> **图 2**：该图展示了 LSTM 处理图中短语后所得隐状态的 2 维 PCA 投影。短语按语义聚类，在这些例子中语义主要是词序的函数，这是词袋模型难以捕捉的。注意两个聚类都有相似的内部结构。

## 4 相关工作

已有大量工作研究神经网络在机器翻译中的应用。迄今为止，把 RNN 语言模型（RNNLM）`[23]` 或前馈神经网络语言模型（NNLM）`[3]` 应用于机器翻译任务最简单也最有效的方式，是对强 MT 基线的 n-best 列表做重打分 `[22]`，这能可靠地提升翻译质量。

近期，研究者开始探索把源语言信息纳入 NNLM 的方法。例如 Auli 等人 `[1]` 把 NNLM 与输入句的主题模型相结合，从而提升了重打分性能。Devlin 等人 `[8]` 采用了类似思路，但把 NNLM 融入了 MT 系统的解码器，并利用解码器的对齐信息为 NNLM 提供输入句中最有用的词。他们的方法非常成功，相对其基线取得了大幅提升。

我们的工作与 Kalchbrenner 和 Blunsom `[18]` 密切相关——他们率先把输入句映射为向量再映射回句子，尽管他们用的是卷积神经网络，会丢失词序信息。与本工作类似，Cho 等人 `[5]` 用类 LSTM 的 RNN 架构把句子映射为向量再映射回来，尽管他们的主要关注点是把神经网络集成进 SMT 系统。Bahdanau 等人 `[2]` 也尝试了用带注意力机制的神经网络直接翻译，以克服 Cho 等人 `[5]` 在长句上遇到的性能不佳问题，并取得了令人鼓舞的结果。同样，Pouget-Abadie 等人 `[26]` 试图通过"分块翻译源句、并让译文平滑拼接"的方式来应对 Cho 等人 `[5]` 的记忆问题，这与基于短语的方法类似。我们猜想，他们只需在倒转的源句上训练网络，就能取得类似的改进。

端到端训练也是 Hermann 等人 `[12]` 关注的重点，他们的模型用前馈网络表示输入与输出，并把它们映射到空间中相近的点。然而，他们的方法无法直接生成译文：要得到译文，需在预先计算好的句子数据库中查找最接近的向量，或对句子做重打分。

> [!note] 本文的历史坐标
> 本节基本把 seq2seq 的"前世今生"点全了：`[18]`（句子→向量→句子）、`[5]`（RNN encoder-decoder）、`[2]`（注意力治长句）是三条直接的前驱线；`[26]` 用"分块"绕开定长瓶颈。文中那句"他们只需在倒转的源句上训练即可"是事后被 `[2]` 的注意力真正解决——对应 [[RecurrentNetworks]] §Encoder-Decoder 与 [[Bahdanau-Attention-2014]] 的对照关系。

## 5 结论

本工作表明：一个词表有限、几乎不对问题结构做任何假设的大型深层 LSTM，能在大规模机器翻译任务上超越词表无限的、基于 SMT 的标准系统。我们基于 LSTM 的简单方法在机器翻译上的成功表明：只要有足够训练数据，它应该也能在许多其他序列学习问题上表现良好。

倒转源句词序带来的改进幅度令我们惊讶。我们的结论是：找到一种"拥有最多短程依赖"的问题编码很重要，因为短程依赖能使学习问题大为简化。特别地，虽然我们无法在未倒转的翻译问题上训练标准 RNN（见图 1），但我们相信，当源句被倒转后，标准 RNN 应当很容易训练（尽管我们没有通过实验验证这一点）。

LSTM 正确翻译很长句子的能力也令我们惊讶。起初我们确信，由于记忆有限，LSTM 会在长句上失败；其他研究者也报告了与我们类似的模型在长句上的不佳表现 `[5, 2, 26]`。然而，在倒转数据集上训练的 LSTM 在翻译长句时几乎没有困难。

最重要的是，我们证明了：一种简单、直接、相对未优化的方法就能超越 SMT 系统，因此后续工作很可能带来更高的翻译准确率。这些结果表明，我们的方法很可能也能在其他有挑战性的序列到序列问题上表现良好。

## 6 致谢

感谢 Samy Bengio、Jeff Dean、Matthieu Devin、Geoffrey Hinton、Nal Kalchbrenner、Thang Luong、Wolfgang Macherey、Rajat Monga、Vincent Vanhoucke、Peng Xu、Wojciech Zaremba 以及 Google Brain 团队的有益评论与讨论。

## 参考文献

原文 31 条参考文献均为英文文献条目，本译本不转译标题。其中与本 Wiki 直接相关的：

- `[16]` Hochreiter & Schmidhuber, LSTM → [[RecurrentNetworks]] / [[RNN-LSTM-GRU]]
- `[2]` Bahdanau attention → [[Bahdanau-Attention-2014]]
- `[10]` Graves, 用 RNN 生成序列（RNN-LM 与可微注意力）→ [[RecurrentNetworks]]
- `[14, 15, 4]` 梯度消失 / 长程依赖难学 → [[RecurrentNetworks]] §瓶颈与长程依赖
- `[25]` Pascanu, 训练 RNN 的梯度裁剪 → [[TrainingMechanics]]
- `[28, 31]` 反向传播 → [[Backpropagation]]
- `[22]` Mikolov, RNNLM / NNLM → [[RNN-LSTM-GRU]]

完整条目可在 `Seq2Seq-Sutskever-2014.txt` 中查阅。
