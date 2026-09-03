---
type: concept
title: 两阶段论文的位置编码（Keyframe PE + Learned Relative PE）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, transformer, positional-encoding]
summary: 论文提出两种位置编码：Keyframe PE 用 MLP 把每帧距上下文/目标帧的距离映射成编码注入特征流；Learned Relative PE 用 MLP+skew 把两帧相对距离注入注意力分数；都用 MLP 生成而非固定表，故能泛化到训练未见的超长过渡。论文 §3.3.1/§3.3.2。
---

# 两阶段论文的位置编码（Keyframe PE + Learned Relative PE）

> [[MotionInbetweeningTwoStage|论文]]的核心创新，论文 §3.3.1 / §3.3.2。底层概念见 [[Transformer|位置编码为何必需]]与[[SelfAttention]]。

## 为什么需要位置编码

[[SelfAttention|自注意力]]本质是**集合操作**——把输入当无序集合处理，打乱顺序喂入结果只是行换位置、内容不变。它**看不出**"帧 0 在帧 1 前面"。

但补间里顺序至关重要。所以必须**额外**注入顺序信息。原版 [[Transformer]] 用正弦/余弦；论文用 **MLP 学出来**两种自定义编码，能泛化到超长过渡——这是论文超越其他 Transformer 方法的关键。

## 问题：已有位置编码在补间任务上的局限

| 方案 | 局限 |
|---|---|
| 正弦绝对编码（vanilla） | 过渡长度超训练最大值时急剧失效（Table 2：45 帧 L2P 3.39） |
| 两端各起一个绝对编码拼接 | 超训练长度几帧，质量迅速劣化 |
| 固定尺寸 lookup table（Gehring 2017 / Shaw 2018） | 表有多长就支持多长，**限制推理长度** |

论文要的是：**只与"距已知帧的相对距离"相关，且能学、能泛化到任意长度**。

---

## Keyframe Positional Encoding

### 思想

对补间任务，一个缺失帧最该知道的是"我离已知帧多远"，而不是"我是整段第 17 帧"。缺失帧与**已知帧的相对距离**才是关键（近则相关强，远则弱）——这无法从绝对位置编码直接推出。

### 实现

`p_kf ∈ R^(T×2)` 是整数张量，描述每帧**相对最后一个上下文帧**和**相对目标帧**的位置。过单隐层 512 的 MLP 映射成 `pe_kf ∈ R^(T×d_model)`，加到 encoder 输出上：

```
pe_kf = φ(p_kf W_kf^(1) + b_kf^(1)) W_kf^(2) + b_kf^(2)
h_ctx = h̃_ctx + pe_kf
```

代码：`ContextTransformer.keyframe_pos_layer`（`Linear(2→512)→PReLU→Linear(512→512)`）+ `get_keyframe_pos_indices`。

### 为什么能泛化到超长过渡

输入是**距离值**（2 维：距上下文帧、距目标帧），不是固定索引。MLP 吃任意距离数值都能吐出编码 → 45 帧（训练未见）也能算 → 泛化。

> **只用在 [[ContextTransformer]]**，不在 [[DetailTransformer]]——精修任务下"距关键帧距离"不再关键。

---

## Learned Relative Positional Encoding

### 思想

相对位置编码捕捉序列内任意两项的**成对距离**。长过渡时任意两帧的相对距离大概率训练中见过，故伪影少。

### 实现

用两层 MLP 把相对距离 `p_rel ∈ R^{(2T−1)×1}` 映射成变长查找表 `E_rel ∈ R^{(2T−1)×d_K}`，再通过 **skew 机制**注入注意力分数：

```
E_rel = φ(p_rel W_rel^(1) + b_rel^(1)) W_rel^(2) + b_rel^(2)
S_rel = skew(Q · E_rel^T)          ← skew 把 (T, 2T-1) 对齐成 (T, T)
```

`S_rel` 加进注意力分数（论文式 8）：`Attention = softmax((QK^T + S_rel)/√d + m_atten)·V`

`skew(·)` 是 Music Transformer（Huang 2018）的省内存机制：把相对距离表按 query 位置错位对齐，避免 `O(T³)` 内存。

代码：`ContextTransformer.rel_pos_layer`（`Linear(1→64)→PReLU→Linear(64→64)`，生成 64 维表，所有层共用）+ `RelMultiHeadedAttention.skew()`。

### 为什么能泛化

同样因用 MLP 生成（非固定查找表），不限长度。`rel_pos_layer` 的 W 是学出来的，但输入 `p_rel` 是任意相对距离值。

> **两个 Transformer 都用**（Context 与 Detail 共用同一个 `rel_pos_layer` 实例，每次 forward 生成一次表）。

---

## 消融（Table 2）

对比 Context Transformer 不同位置编码（L2P/L2Q/NPSS，30 与 45 帧）：

| 位置编码 | L2P@30 | L2P@45 | L2Q@45 | NPSS@45 |
|---|---|---|---|---|
| pe_sin_abs（正弦绝对） | 1.08 | 3.39 | 1.41 | 0.7045 |
| pe_sin_rel（正弦相对，Huang 2018） | 1.07 | 2.66 | 1.09 | 0.3947 |
| pe_sin_rel + pe_kf | 1.05 | 2.08 | 0.96 | 0.3527 |
| pe_rel（学习相对） | 1.07 | 2.00 | 0.93 | 0.3674 |
| **pe_rel + pe_kf（完整）** | **1.04** | **1.94** | **0.91** | **0.3349** |
| ERD-QV（RNN 参照） | 1.28 | 2.24 | 0.94 | 0.3311 |
| ERD-QV w/ pe_kf | 1.23 | 2.15 | 0.97 | 0.3755 |

**两种编码是 Transformer 超越 RNN 的关键。** 相对编码（正弦或学习）大幅改善 45 帧；Keyframe PE 在相对编码之上再改善；两者齐全最优。

### pe_kf 移植到 RNN 收效甚微

把 pe_kf 替换 ERD-QV 的 time-to-arrival embedding（`z_tta`），结果与原 ERD-QV 相当。原因：RNN 自回归，隐藏状态已携带过去上下文，"距上下文距离"不重要；而"距目标距离"`z_tta` 已有，故 pe_kf 对 RNN 非必需。**Transformer 一次输出整段，相对距已知帧才关键**——这是架构差异带来的。

---

## 与代码的对应

| 论文概念 | 代码位置 |
|---|---|
| Keyframe PE 生成 | `ContextTransformer.keyframe_pos_layer`（`Linear(2→512)→PReLU→Linear(512→512)`） |
| Keyframe PE 索引 | `context_model.get_keyframe_pos_indices`（返回 `(1, T, 2)`） |
| Learned Relative PE 生成 | `ContextTransformer.rel_pos_layer`（`Linear(1→64)→PReLU→Linear(64→64)`） |
| 相对 PE 表生成 | `ContextTransformer.get_rel_pos_emb`（`pos_idx = [-T+1..T-1]`） |
| skew 机制 | `RelMultiHeadedAttention.skew()` |
| 注入注意力分数 | `RelMultiHeadedAttention.forward`：`atten_score = atten_score + self.skew(qpos, mem_len)` |

## 相关页面

- [[ContextTransformer]] —— 使用两种 PE 的第一级
- [[DetailTransformer]] —— 只用 Learned Relative PE（无 Keyframe）
- [[Transformer]] —— 位置编码为何必需（底层概念）
- [[SelfAttention]] —— 相对 PE 如何加进 `Q·K^T`
