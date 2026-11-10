---
type: schema-guide
title: 数学公式与符号约定
created: 2026-09-07
updated: 2026-09-07
---

# 数学公式与符号约定

本知识库数学相关内容的**唯一格式标准**。所有 TheWiki 页面、RawSource 摘要、会话沉淀中的数学内容均须遵守。

## 1. 公式渲染格式（必守）

数学公式一律使用 Obsidian 原生 LaTeX 标记（MathJax 渲染）：

- **行内公式**用 `$...$`：正文里提到的单个符号或简短关系。如 `权重矩阵 $W$`、`更新式 $w_k \to w_k - \eta \frac{\partial C}{\partial w_k}$`。
- **块级公式**用 `$$...$$`（单独成行、公式独占一个段落）：核心定义、损失函数、递推式、推导链条。如

  ```text
  $$\sigma(z) \equiv \frac{1}{1+e^{-z}}$$
  ```

**禁止**：

- ❌ 把公式写进代码块（```` ``` ````）或反引号里
- ❌ 用纯文本 / ASCII / Unicode 字符手拼公式（如 `` `h_t = φ(W_hh·h_{t-1}+...)` ``）
- ❌ 下标/上标/希腊字母用手写体（`h_t`、`W_hh`、`φ`），一律用 LaTeX（`h_{t}`、`W_{hh}`、`\phi`）

反例 vs 正例：

- ❌ `` `h_t = φ(W_hh · h_{t-1} + W_xh · x_t + b)` ``（反引号包整条公式）
- ✅ 块级 `$$h_t = \phi(W_{hh}\, h_{t-1} + W_{xh}\, x_t + b)$$`

## 2. 公式与「结构示意图」的边界（豁免规则）

代码块（```` ``` ````）**不是禁用区**——它仍是**结构示意图 / 数据流图 / 伪代码**的合适载体。判定标准：

| 内容 | 本质 | 载体 |
|---|---|---|
| 数学关系式、定义、等式、不等式 | 公式 | `$...$` / `$$...$$` |
| 网络/数据流结构、张量形状变换、逐帧示意图 | 示意图 | 代码块 |

**混合情况**：结构示意图（代码块）内部如果包含数学关系（如 `W1·x+b1`、`softmax(Q·K^T)`），该部分仍应写成 LaTeX 行内公式，而不是手拼符号。示意图优先保证可读性，允许 `Linear`、`+`、`→` 等说明性文字。

- ✅ 示例（结构图 + 行内公式混排）：
  ````text
  ```
  x ($d_{in}$) → Linear: $W_1 x + b_1$ → 激活 $\phi$ → h ($d_{h}$)
  ```
  ````
- ❌ 示例（公式手拼塞进代码块）：`` ``` `` `` y = φ(W·x + b) `` `` ``` ``

## 3. 符号约定表

知识库统一使用以下符号，**各页不得另起炉灶**。来源约定：Nielsen《Neural Networks and Deep Learning》(2015) + 既有 TheWiki 页面惯例（MotionMatching / 深度学习系）。

### 3.1 标量、向量、矩阵

| 符号 | 含义 | 备注 |
|---|---|---|
| $x$ | 标量 / 输入向量（视上下文） | 向量默认**列向量** |
| $\boldsymbol{W}$ 或 $W$ | 权重矩阵 | 行数 = 本层神经元数，列数 = 上一层神经元数 |
| $w_{jk}$ | 权重矩阵元素 | Nielsen 约定：**连接第 $k$ 个源神经元 → 第 $j$ 个目标神经元**（便于无转置写出 $a'=\sigma(Wa+b)$） |
| $b$ | 偏置向量 | 每层一个，形状 = 本层神经元数 |
| $\phi$ / $\sigma$ | 激活函数 | $\sigma$ 特指 sigmoid/logistic；$\phi$ 泛指（ReLU/PReLU/…） |
| $a^l$ | 第 $l$ 层激活向量 | $a^1$ = 输入层 |
| $z^l$ | 第 $l$ 层加权输入 | $z^l \equiv W^l a^{l-1}+b^l$，激活前 |
| $\Theta$ | 全部参数集合（有时记 $\theta$） | |

### 3.2 常用算子

| 记号 | 含义 | LaTeX |
|---|---|---|
| $a \cdot b$ | 点积 / 内积 | `a \cdot b` |
| $\odot$ | Hadamard 积（逐元素乘） | `\odot` |
| $A^T$ | 转置 | `A^T` |
| $\|v\|$ | L2 范数（欧氏长度） | `\|v\|` |
| $\nabla C$ | 梯度（对所有权重/偏置） | `\nabla C` |
| $\sum_j$ / $\prod_j$ | 求和 / 连乘 | |
| $\mathbb{E}[X]$ | 期望 | `\mathbb{E}` |

### 3.3 神经网络学习相关

| 记号 | 含义 | 备注 |
|---|---|---|
| $C(w,b)$ | 代价函数（cost/loss/objective） | 本书二次代价：$C=\frac{1}{2n}\sum_x\|y(x)-a\|^2$ |
| $C_x$ | 单样本代价 | |
| $y = y(x)$ | 期望输出（监督信号） | 分类常为 one-hot 向量 |
| $\eta$ | 学习率 learning rate | |
| $n$ | 训练样本总数 | mini-batch 大小另记 $m$ |
| $m$ | mini-batch 大小 | |
| $w_k \to w_k - \eta\frac{\partial C}{\partial w_k}$ | 梯度下降更新 | 负梯度方向下降 |
| epoch | 完整遍历训练集一遍 | |
| $\Delta v$、$\Delta C$ | 小扰动记号 | 一阶近似 $\Delta C\approx\nabla C\cdot\Delta v$ |

### 3.4 数据形状速记

| 记号 | 含义 |
|---|---|
| $d_{in}$ | 输入维 |
| $d_{out}$ | 输出维 |
| $d_{model}$ / $d_{h}$ | 隐藏维 / 模型宽 |
| $d_k$ | attention 的 key/query 维 |
| `(n, 1)` 等 | 具体张量形状用代码字体（numpy 风格） |

### 3.5 易混符号警示

- `d_in × d_out`（纯文本乘号）→ 一律 `$d_{in} \times d_{out}$`；矩阵乘不写 `×` 而并置（$Wx$）
- `φ` 与 `ϕ`、`σ` 与 `Σ`（和号）严格区分
- `T` 转置须上标：`$x^T$`，不是正文大写 T 带过
- 指数一律 LaTeX：`e^{-z}`、`e^{z_i}`，不写 `exp(-z)` 手拼（块级公式内可保留 `\exp` 排版）

## 4. frontmatter summary 的例外

`summary` 字段是纯文本摘要（供检索与表格展示），**不强制**套用 LaTeX：`y = φ(Wx+b)` 这类可读写法在 summary 里允许，正文中禁止。修正正文时若 summary 已含清晰描述，可保留原写法。

## 参考

- 本书系来源：Michael A. Nielsen, *Neural Networks and Deep Learning* (2015)，`RawSource/articles/NeuralNetworksAndDeepLearning-Nielsen-2015/`。
- 本规范由 `TheSchema/changelog.md` 记录变更；执行修正时的基准页：`TheWiki/deep-learning/RecurrentNetworks.md`。
