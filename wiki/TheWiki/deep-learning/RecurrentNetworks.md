---
type: concept
title: RNN 与 Encoder-Decoder 范式
created: 2026-08-27
updated: 2026-09-02
tags: [deep-learning, rnn, encoder-decoder, sequence-modeling]
summary: RNN 用隐藏状态在时间步间传记忆处理序列，展开即深层共享权重网络，BPTT 致梯度消失与长程遗忘，自回归逐帧生成慢且局部；Encoder-Decoder 是 seq2seq 结构范式，论文 Context Transformer 的 decoder 只是 MLP 非解码器，是术语陷阱。
---

# RNN 与 Encoder-Decoder 范式

## RNN 的诞生：为序列而生

普通 [[NeuralNetworkBasics|MLP]] 处理固定形状的单个输入。但很多数据是**序列**：一句话、一段动画——长度可变、顺序重要、前后相关。MLP 没有处理"时序"和"记忆"的机制。

RNN 的核心想法：**加一个跨时间步传递的"隐藏状态" $h$，充当记忆。**

## 递归方程

每一步，用"上一步记忆 + 当前输入"更新记忆，再从记忆产出：

$$h_t = \phi\big(W_{hh}\, h_{t-1} + W_{xh}\, x_t + b\big)$$

$$y_t = W_{hy}\, h_t$$

| 符号 | 含义 | 形状 |
|---|---|---|
| $x_t$ | 第 $t$ 步输入（第 $t$ 帧姿态） | $d_{in}$ 维 |
| $h_t$ | 第 $t$ 步隐藏状态（记忆） | $d_h$ 维 |
| $W_{xh}$ | 输入→隐藏 权重 | $d_h \times d_{in}$ |
| $W_{hh}$ | 旧记忆→新记忆 权重 | $d_h \times d_h$ |
| $W_{hy}$ | 隐藏→输出 权重 | $d_{out} \times d_h$ |

**关键**：$W_{xh}$、$W_{hh}$、$W_{hy}$ 在**所有时间步共享同一套数值**。无论序列多长，反复迭代这三组权重。

## 时间展开 = 深层共享权重网络

把 RNN 沿时间展开（3 帧序列）：

```
x_1 ─[Cell]─ x_2 ─[Cell]─ x_3 ─[Cell]─
       ↑             ↑             ↑
      h_0 ─→ h_1 ─→ h_2 ─→ h_3
                                    │
                                   y_3
```

每个 `[Cell]` 是同一个 RNN 单元（同一套 $W$）。所以：

- 展开后 RNN 在**时间方向上是一个很深的网络**（$T$ 帧即 $T$ 层）
- 每层用**同一组权重**

### 玩具例子（1 维姿态，2 维隐藏，3 帧）

设 $x = [0.5,\ 0.8,\ 0.2]$，$h_0 = [0,\ 0]$，逐帧递推：

- $h_1 = \tanh\big(W_{hh}\,h_0 + W_{xh}\cdot 0.5\big)$ —— 只受 $x_1$ 影响
- $h_2 = \tanh\big(W_{hh}\,h_1 + W_{xh}\cdot 0.8\big)$ —— 受 $x_1$（经 $h_1$）与 $x_2$ 影响
- $h_3 = \tanh\big(W_{hh}\,h_2 + W_{xh}\cdot 0.2\big)$ —— 受 $x_1,x_2,x_3$ 全部影响

$h_3$ 里 $x_1$ 的路径是 $x_1 \to h_1 \to h_2 \to h_3$，经过 $W_{hh}$ 两次连乘。30 帧序列则连乘 29 次。

## BPTT 与梯度消失：长程遗忘的根源

训练 RNN 用**沿时间反向传播（BPTT）**：从最后损失倒推梯度，沿时间往回走。梯度每往回一步都乘一次 $W_{hh}$。

- $W_{hh}$ 特征值 $<1$：连乘 29 次 → 梯度 $\to 0$（**梯度消失**）→ 早期帧学不动，被遗忘
- $W_{hh}$ 特征值 $>1$：连乘 29 次 → 梯度 $\to \infty$（**梯度爆炸**）

这就是 RNN **记不住久远信息**的数学根因。$h_t$ 本质是对过去所有帧的**有损压缩**——塞进一个固定大小向量，越早越模糊。

## 自回归生成

要生成序列，RNN 一帧一帧来：算出第 $t$ 帧输出 → 当输入算第 $t+1$ 帧 → …… 30 帧过渡要跑 30 次 LSTM。每步依赖上一步，**不能并行**，且误差逐帧累积。

## LSTM：给 RNN 打补丁

为缓解遗忘，出现 LSTM（ERD-QV 用的就是它）。在 $h_t$ 外加**细胞状态 $C_t$**，用三个**门**控制信息：

| 门 | 作用 |
|---|---|
| 遗忘门 $f_t$ | 决定从 $C_{t-1}$ 丢掉什么 |
| 输入门 $i_t$ | 决定把新信息写多少 |
| 输出门 $o_t$ | 决定 $C_t$ 暴露多少给 $h_t$ |

每个门是学出来的 $0\sim 1$ 数值（sigmoid），像可调阀门。LSTM 让"记住久远信息"变**可学**，是 RNN 的补丁，但**没改变"逐步循环、有损压缩、不能并行"三个根本问题**。

> 门控单元（LSTM/GRU）的递推公式与直觉详解见 [[RNN-LSTM-GRU]]。

## ERD-QV 的结构（论文的 RNN 对比基线）

ERD-QV（Robust Motion In-betweening, Harvey 2020）在 `model/rmi.py`：

```
每步输入:
  state  (当前帧状态)   ─┐
  offset (距目标偏移)   ─┼─→ 三个 encoder(MLP) ─concat→ LSTM → decoder(MLP) → 输出该帧
  target (目标帧)       ─┘
  ztta   (距目标还有几帧的编码, TimeToArrival embedding)
  ztarget(目标噪声, 可选)
                         上一步 hidden state
```

逐帧自回归。

## 为什么 RNN 不适合补间任务

| RNN 毛病 | 补间任务上的后果 |
|---|---|
| 逐步循环、不能并行 | 30 帧过渡串行跑 30 步，推理时间随长度线性增长 |
| 隐藏状态有损压缩 | 早期过渡帧主要靠"过去的记忆"生成、忽略目标；接近目标时才转向 → 两阶段交界处**突变** |
| 长程遗忘 | 远距上下文记不住 |
| 自回归误差累积 | 缺乏全局规划 |

这正对应 [[ContextTransformer|论文对 ERD-QV 的批评]]。

---

## Encoder-Decoder 范式

### 是什么

Encoder-Decoder 是一种**结构范式**，不是具体网络：

```
输入序列 ──[Encoder]──► 中间表示（context）──[Decoder]──► 输出序列
```

- **Encoder**：把输入序列"读懂"，压缩成中间表示
- **Decoder**：基于中间表示，生成输出序列（通常自回归）

经典场景：**机器翻译**。Encoder 读法语压成语义向量；Decoder 据此写英语。输入输出长度可不同。

### 瓶颈问题

早期 seq2seq 把整个输入压成**一个固定长度向量**再交给 Decoder。长输入时信息塞不进去，丢失严重——这叫 **bottleneck**。

后来注意力机制就是为解它而生的：让 Decoder 生成每个词时能回头看 Encoder 每个位置，动态决定关注哪些。这个"回头看"演化成了 [[SelfAttention|自注意力]]，是 [[Transformer]] 的基础。

### 论文里的术语陷阱

论文把 Context Transformer 写成"encoder-decoder 结构"，但**这里的 decoder 不是自回归 seq2seq decoder，只是两层 MLP**：

```
输入 → Encoder(MLP 投影) → 6层Transformer层 → Decoder(MLP 投影回 135 维) → 输出
```

输入和输出**长度相同、位置一一对应**，不是翻译。只是借用了"先把输入映射到隐空间、再映射回数据空间"的形状。别被命名误导。

论文的对比方法 **Δ-Interp 才是真正的 Transformer Encoder-Decoder**（类似翻译用原版 Transformer）：在 Slerp 插值之上预测 delta 动作。论文论证它更差：Slerp 只看两个关键帧、可能和上下文矛盾，delta 修正不过来，长过渡时急剧劣化。

## 对比表：三种网络处理同一序列

同一组 30 帧动画：

| 网络 | 怎么生成帧 i 的输出 | 毛病 |
|---|---|---|
| [[NeuralNetworkBasics|MLP]] | 拍成 4050 向量一次喂入 | 参数爆炸、不能变长、丢失顺序 |
| RNN | 一帧一帧喂，$h_0 \to \dots \to h_{30}$ | 能变长有序，但串行慢、长程遗忘、局部生成 |
| [[Transformer]] | 30 帧同时喂入，注意力让每帧直接看所有帧 | 能变长有序、并行快、全局视野、长程不遗忘 |

**一句话**：MLP 是"无结构的函数逼近器"，RNN 是"靠记忆向量传时序的串行网络"，Transformer 是"靠注意力直接跨帧通信的并行网络"。补间任务需要全局规划 + 变长 + 速度，所以 Transformer 胜出。

---

## 参考资料

- 中文全译本（Wikipedia「Recurrent neural network」词条，逐节翻译 + 公式 LaTeX）：[[Recurrent_neural_network-中文翻译]]
- 单元级原理（RNN/LSTM/GRU 门控详解，以机器翻译为例）：[[RNN-LSTM-GRU]]
- 英文原版 PDF（本地）：`RawSource/articles/Recurrent_neural_network.pdf`
