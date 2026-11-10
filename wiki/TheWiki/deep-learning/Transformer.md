---
type: concept
title: Transformer
created: 2026-08-27
updated: 2026-08-27
tags: [deep-learning, transformer, attention, architecture]
summary: 用自注意力替换循环的并行序列网络；核心是"每帧从所有帧按相关性取加权平均，相关性从数据现算"；每层=多头自注意力(跨帧混合)+FFN(逐帧变换)+残差+LayerNorm，堆 L 层；输入输出同形，反复精炼。
---

# Transformer

## 出发点：用"注意力"替换"循环"

[[RecurrentNetworks|RNN]] 的病根是"逐步循环"——记不住久远、不能并行。Transformer 的核心革命：**不要循环，让每个位置直接和所有其他位置建立联系**——这就是 [[SelfAttention|自注意力]]。

## 一句话核心

> **每个输出帧 = 所有关键帧的加权平均，而"加权多少"是从数据本身算出来的。**

三层意思：

1. **加权平均**：输出帧不是凭空生成，而是把别的帧内容按比例掺和。这个动作天然能"把相关信息拉过来"。
2. **所有帧**：不像 RNN 只能找相邻的，这里帧 i 可直接从帧 0、帧 2、帧 30 任意一帧拿信息——不管隔多远，路径长度永远是 1。**这就是它治"长程遗忘"的方式。**
3. **权重从数据算出**：不像"取平均"那种每帧权重一样的死板做法，权重是**动态的**——当前输入是什么，权重就变成什么。

对比三种网络处理 30 帧动画：

| 网络 | 生成帧 i 输出的方式 | 毛病 |
|---|---|---|
| [[NeuralNetworkBasics|MLP]] | 只看帧 i 自己 | 完全看不到别的帧 |
| RNN | 看"传了 i 次之后的记忆" | 远的帧传过来早走样 |
| **Transformer** | **直接从所有帧按权重取平均** | 没有上面两个毛病 |

## 注意力怎么算权重 → 见 [[SelfAttention]]

权重具体怎么算？需要一个**相关性分数**：对任意两帧 (i, j)，问"j 对 i 有多相关"。Q/K/V 就是"算这个分数"的标准做法（图书馆比喻 + 手算全程见 [[SelfAttention]]）。

**关键：权重是"活的"。** 死矩阵 `y = M·x` 里 `M` 固定不变；attention 里权重 `A = softmax(Q·K^T)` 是从输入算出来的（`Q=X·W_Q`, `K=X·W_K`），输入变权重变。这就是它和"一个矩阵"的根本区别，也是它能做全局规划的原因。

## 一个 Transformer 层的完整结构

一层 = 两个子层，每个带**残差连接 + [[TrainingMechanics|LayerNorm]]**（论文用 pre-norm：LayerNorm 在子层输入前，更稳）：

```
输入 x  (batch, seq, d_model=512)
  │
  ├─ 子层1: Masked Multi-Head Self-Attention
  │    h  = LayerNorm(x)            ← pre-norm
  │    a  = MHA(h, mask)            ← 跨帧混合，详见 [[SelfAttention]]
  │    x' = x + a                   ← 残差
  │
  ├─ 子层2: Position-wise FFN（就是个 MLP）
  │    h' = LayerNorm(x')
  │    f  = FFN(h')                 ← 逐帧变换，不混合帧
  │    x''= x' + f                  ← 残差
  ▼
输出 x''  (shape 不变)
```

### 两个子层分工

| 子层 | 干的事 | 跨不跨帧 |
|---|---|---|
| attention | "我该听谁的、把谁的信息搬过来"（路由） | 跨帧 |
| FFN | "听完之后，怎么加工这些信息"（计算） | 逐帧 |

光有 attention：信息能搬来搬去但没人消化。光有 FFN：每帧能加工但帧间不通气。**两者必须配**，每层交替做"通气+加工"。

### 为什么 attention 必须配 FFN

attention 输出是 `A·V`——一个**加权平均**。对固定 A，`V → A·V` 是线性的。attention 能把别处特征"搬"到这帧来，但搬来之后不做非线性计算。

想算"如果特征 a 大且 b 小，就输出 a−b"这种**逐帧非线性判断**，attention 干不了，FFN 干。所以一层必须 = attention + FFN，缺一不可。

## FFN 的具体结构

逐帧独立的 [[NeuralNetworkBasics|MLP]]（论文里 `512→512→512`，ReLU 激活），每帧独立跑、权重共享、**不跨帧**。代码 `PositionwiseFeedForward`：

```
Linear(512→512) → ReLU → Dropout → Linear(512→512) → Dropout
+ 残差 + LayerNorm(pre-norm)
```

## 多头

把 `d_model=512` 拆成 `H=8` 个头，每个 `d_head=64`（`8×64=512`）。每个头独立做一套 Q/K/V 投影 + attention，最后拼回 512：

```
                 ┌─ head0 (d_head=64): Q0,K0,V0 → attention → out0
                 ├─ head1: ...
d_model=512 ────┤  ...
                 └─ head7: ...
                                          ↓ concat
                                    512 → W_O → 512
```

不同头能学不同关系：一个头看"时间近不近"，一个头看"像不像目标帧"。论文 `n_head=8, d_head=64`。

> "一个头关注什么"是训练后**涌现的结果**，不是写死的：[[SelfAttention|PE 提供位置信号]]，`W_Q`/`W_K` 学会利用它，SGD 在任务上调出"这个头恰好对近邻帧打分高"。不同头有不同 W 故学到不同相关性。

## 堆叠 + 外壳

- 把一层堆 `L=6` 次（论文 `n_layer=6`），每层独立参数
- **前面**接 encoder MLP（`136→512→512`，PReLU）：姿态投影到特征空间
- **后面**接 decoder MLP（`512→512→135`）：投影回姿态维度
- **注入位置信息**：加 [[SelfAttention|位置编码]]

这就是 [[ContextTransformer|论文 ContextTransformer.forward]] 的全貌：

```
x ─encoder(MLP)─► +keyframe_pe ─► 6×[RelAttention+FFN] ─► LayerNorm ─► decoder(MLP) ─► 输出
```

论文每层用 `RelMultiHeadedAttention`（带相对位置编码的 attention），核心算术同 [[SelfAttention]] 手算那套，只是分数里多加一项相对位置。

## 输入输出形状不变（关键性质）

```
输入: (seq 帧数, d_model=512)
输出: (seq 帧数, d_model=512)   ← 同形
```

**形状不变，它只是把每帧特征"重新精炼"**——每帧都掺入了别的帧的相关信息。既然形状不变，就可反复套（堆 6 层 = 6 轮精炼）。

## 位置编码：为什么必须有

attention 本身是**集合操作**——把输入当无序集合处理，打乱顺序喂入结果只是行换位置、内容不变。它**看不出**"帧 0 在帧 1 前面"。

但动画里顺序至关重要。所以必须**额外**注入顺序信息——这就是位置编码（PE）的作用。原版用正弦/余弦；论文用 MLP 学出来两种自定义 PE（见 [[InbetweeningPositionalEncodings]]），能泛化到超长过渡。

## 三种用法

| 变体 | 代表 | 典型任务 |
|---|---|---|
| Encoder-only | BERT | 理解/完形填空（补全挖掉的部分） |
| Encoder-Decoder | 原版 Transformer、T5 | 翻译（输入输出不同序列） |
| Decoder-only | GPT | 生成（自回归续写） |

### 论文为何用 Encoder-only 就能做补间

直觉上"生成缺失中间帧"好像需要 Decoder。但论文用 **Encoder-only + 掩码**，借了 BERT 的"完形填空"思路：

- 输入 = 上下文帧 + 占位帧（过渡） + 目标帧，长度完整
- **注意力掩码**：让占位帧不能作为 Key 被别人关注，只有真实帧能被关注 → 占位帧只能去"看"真实帧，把自己填上
- 输出 = 整段序列，占位位置被预测值替换

**不需要 Decoder 也能补间**。两阶段则是：Context 先粗填一遍，Detail 再精修（补细节、预测足部接触）。配合自定义位置编码实现超长泛化——这是论文超越其他 Transformer 方法的关键。

## 完整前向数据流（30×135 → 30×135）

以 Context Transformer 为例（对应代码 `model/__init__.py` forward）：

**记号**：`T=30`, `d=135`, `d_model=512`, `H=8`, `d_head=64`, `L=6`。

```
X_raw ∈ R^(30×135)           ← 输入姿态（132 旋转 + 3 根位置）
m_in  ∈ R^(30×1)             ← 每帧"是否已知"
p_kf  ∈ R^(30×2)             ← 每帧 [距上下文帧, 距目标帧]
M_att ∈ {0,1}^(30×30)        ← 注意力掩码

Step 0: z-score     X = (X_raw − μ)/σ ∈ R^(30×135)
Step 1: 拼 mask     X_in = [X | m_in] ∈ R^(30×136)
Step 2: encoder MLP  H = PReLU(W2·PReLU(W1·X_in+b1)+b2) ∈ R^(30×512)
Step 3: +Keyframe PE  PE_kf = MLP(p_kf) ∈ R^(30×512);  H ← H + PE_kf
Step 4: 相对PE表     E_rel = MLP([-29..29]) ∈ R^(59×64)   ← 6 层共用
Step 5: 循环 ℓ=1..6:
   子层A 掩码多头自注意力:
     Ĥ = LayerNorm(H_{ℓ-1})
     对 h=1..8:  Q_h=Ĥ·W_Q^(h)  K_h=Ĥ·W_K^(h)  V_h=Ĥ·W_V^(h)  (各 30×64)
       S_h = Q_h·K_h^T  +  skew(Q_h·E_rel^T)        (30×30)
       S_h ← S_h/√512,  S_h.masked_fill(M_att, −∞)
       A_h = softmax(S_h)                              (30×30)
       O_h = A_h·V_h                                    (30×64)
     O = [O_1|...|O_8]·W_O  ∈ R^(30×512)
     H' = H_{ℓ-1} + O                                  ← 残差
   子层B FFN:
     F = W2_ff·ReLU(W1_ff·LayerNorm(H')+b1)+b2 ∈ R^(30×512)
     H_ℓ = H' + F                                      ← 残差
Step 6: 最终 LN     H_final = LayerNorm(H_6)
Step 7: decoder MLP  Y = W2_dec·PReLU(W1_dec·H_final+b1)+b2 ∈ R^(30×135)

输出 Y ∈ R^(30×135)
```

**核心观察**：姿态从 135 维被抬到 512 维特征空间，在 512 维里经 6 轮"跨帧混合 + 逐帧变换"，最后投影回 135 维。**形状始终是 30×N，只是 N 在 135↔512 间切换**；attention 不改形状，只把每帧的 512 维特征用别的帧的信息"刷新"。

> 参数 vs 运行时算的：学出来的参数有 `W_Q,W_K,W_V,W_O`（每层 ×6）、FFN 的 W、encoder/decoder MLP、PE 生成 MLP、LayerNorm γ/β；运行时算的（随输入变）有 `Q,K,V`、`S=Q·K^T`、`A=softmax(S)`、输出。`W_Q` 固定，`A` 随输入变——input 相关性全来自 `H` 流入 `Q`/`K`，参数本身不动。详见 [[SelfAttention]]、[[Backpropagation-Transformer]]（W_Q 梯度实例）与 [[Backpropagation]]（通用机制）。

## 在本项目中的体现

- [[ContextTransformer]]、[[DetailTransformer]] 是论文的两个 Transformer（Encoder-only + 掩码 + 自定义 PE）
- [[SelfAttention]] 是核心运算
- [[InbetweeningPositionalEncodings]] 是论文的两种自定义位置编码
- [[TrainingMechanics]] 含其训练用的 LayerNorm/Adam/Noam 细节
- 对比方法 ERD-QV 是 [[RecurrentNetworks|RNN]]，Δ-Interp 是 Encoder-Decoder Transformer，见 [[Compare-Inbetweening-Methods]]
