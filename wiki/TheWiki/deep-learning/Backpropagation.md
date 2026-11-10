---
type: concept
title: 反向传播与雅可比矩阵
created: 2026-08-27
updated: 2026-09-07
tags: [deep-learning, backpropagation, jacobian, calculus, training]
summary: 反向传播=链式法则+局部性，梯度信号从损失倒流；矩阵乘法反向核心规则 ∂L/∂W=X^T·∂L/∂Y；Nielsen 四公式 BP1-BP4（误差 δ=∂C/∂z 逐层回传）为通用框架；雅可比是向量函数的导数，softmax 满雅可比须单独处理。项目 W_Q 反传实例见 Backpropagation-Transformer。
---

# 反向传播与雅可比矩阵

> 本页是**通用概念页**：反向传播的一般机制（链式法则、局部性、矩阵乘法反向规则、Nielsen 四公式）与雅可比矩阵。
> 在 Transformer 项目里的具体反传链（从 Loss 传回 $W_Q$）见 [[Backpropagation-Transformer]]。

## 反向传播 = 链式法则 + 局部性

### 链式法则

若 $L = f(g(h(x)))$，则：

$$
\frac{\partial L}{\partial x} = \frac{\partial L}{\partial f}\cdot\frac{\partial f}{\partial g}\cdot\frac{\partial g}{\partial h}\cdot\frac{\partial h}{\partial x}
$$

损失对最底层 $x$ 的梯度 = 一连串"局部导数"乘起来。

### 局部性（反向传播能高效算的关键）

每个运算**只需要知道两样东西**就能算出自己的梯度：

1. **上游传来的梯度** $g_{out} = \partial L/\partial(\text{我的输出})$ —— 从后一站收到
2. **我自己的局部导数** —— 输出怎么依赖输入

有了这两样，它能产出：
- **传给前面的梯度** $g_{in} = \partial L/\partial(\text{我的输入})$ —— 继续往前传
- **我自己参数的梯度** $\partial L/\partial(\text{我的参数})$ —— 留给 SGD/[[TrainingMechanics|Adam]] 更新

所以反向传播 = **一个"梯度信号"从损失倒着流，每经过一个运算就被变换一下、并顺便算出那个运算自己参数的梯度**。

## 唯一核心规则：矩阵乘法的反向

整个反向传播里 **80% 靠这一条**，和 [[NeuralNetworkBasics|MLP]] 一模一样：

> 若前向 $Y = X\cdot W$，则反向：
>
> $$
> \frac{\partial L}{\partial W} = X^T\cdot\frac{\partial L}{\partial Y}
> \qquad\text{（参数梯度）}
> $$
>
> $$
> \frac{\partial L}{\partial X} = \frac{\partial L}{\partial Y}\cdot W^T
> \qquad\text{（输入梯度，继续往下传）}
> $$

记住这条，后面只是反复套。softmax 那步稍特殊，需雅可比，单独给（见下文）。

## Nielsen 四公式框架（BP1–BP4）

> 来源：Michael Nielsen《Neural Networks and Deep Learning》Ch.2。与"矩阵乘法反向规则"是**同一件事的两种表述**：规则讲"梯度 $g$ 逐运算回传"，Nielsen 框架讲"误差 $\delta$ 逐层回传"——见下方对照。

### 核心量：误差 $\delta$（对加权输入的梯度）

先定义第 $l$ 层的**加权输入** $z^l \equiv w^l a^{l-1}+b^l$（激活 $a^l=\sigma(z^l)$ 之前的值）。误差就是代价对这个加权输入的偏导：

$$
\delta^l_j \equiv \frac{\partial C}{\partial z^l_j}
$$

**恶魔直觉**：小恶魔坐在神经元处，给加权输入加扰动 $\Delta z^l_j$，代价随之变化 $\frac{\partial C}{\partial z^l_j}\Delta z^l_j$。若 $\partial C/\partial z^l_j$ 很大（正或负），恶魔选异号的 $\Delta z^l_j$ 就能大幅降代价；若它接近零，该神经元已"接近最优"。所以 $\delta^l_j$ 是"该神经元误差"的天然度量。

### 四个基本方程

| 方程 | 内容 | 直觉 |
|---|---|---|
| **BP1** | $\delta^L_j = \frac{\partial C}{\partial a^L_j}\sigma'(z^L_j)$，矩阵形式 $\delta^L = \nabla_a C \odot \sigma'(z^L)$ | 输出层误差 = 代价对输出的梯度 × 激活函数斜率 |
| **BP2** | $\delta^l = \big((w^{l+1})^T \delta^{l+1}\big)\odot\sigma'(z^l)$ | 把下一层误差**反穿权重转置**（移过权重），再**反穿 $\sigma'$**（移过激活） |
| **BP3** | $\frac{\partial C}{\partial b^l_j} = \delta^l_j$ | 偏置梯度**恰好等于**误差 |
| **BP4** | $\frac{\partial C}{\partial w^l_{jk}} = a^{l-1}_k\delta^l_j$，简写 $\frac{\partial C}{\partial w}=a_{in}\delta_{out}$ | 权重梯度 = 输入端激活 × 输出端误差 |

用 BP1 算 $\delta^L$ → 反复用 BP2 算 $\delta^{L-1},\delta^{L-2},\ldots$ → 用 BP3/BP4 由 $\delta^l$ 得到偏置与权重的梯度。这就是 Ch.1 代码里 `backprop(x,y)` 的完整逻辑。

### 两个直接洞察

1. **低激活 ⟹ 学得慢**：BP4 中 $a^{l-1}_k\approx0$ 时梯度 $\approx0$。
2. **饱和 ⟹ 学得慢**：BP1/BP2 中的 $\sigma'(z^l)$——sigmoid 输出接近 0 或 1 时 $\sigma'$ 趋零，误差被"掐灭"。这直接引出 Ch.3 的动机：换交叉熵代价 + 换激活函数（$\sigma'$ 恒正）来避免饱和。

### 接口契约：为什么要求「代价 = 单样本代价的平均」

反向传播本质是**单样本梯度计算器**——BP1 的 $\nabla_a C_x$ 需要知道单个样本 $x$ 的 $y(x)$ 才能算。所以它要求：

- **假设一**：$C=\frac1n\sum_x C_x$（可分解成单样本代价的平均）→ 逐样本算 $\nabla C_x$ 再平均，才拼出 $\nabla C$；这也是 Ch.1 SGD 抽样平均的前提
- **假设二**：$C$ 是网络输出 $a^L$ 的函数（不依赖内部状态）→ 链式法则才能从输出往内层传

若代价含不可分解的全局项（如对比学习的三元组损失），反向传播就不适用，那类梯度必须绕道单独算。

### 与"矩阵乘法反向规则"的对照

| Nielsen | 通用规则 | 说明 |
|---|---|---|
| $\delta^l$ | $g_{z^l} = \partial L/\partial z^l$ | 同一个量：对第 $l$ 层加权输入的梯度 |
| BP2 的 $(w^{l+1})^T\delta^{l+1}$ | $\frac{\partial L}{\partial X} = \frac{\partial L}{\partial Y}\cdot W^T$ | 误差反穿权重 = 梯度乘以权重转置 |
| BP2 的 $\odot\sigma'(z^l)$ | 逐元素激活（如 ReLU 的 0/1 掩码） | 误差反穿激活函数 |
| BP4 的 $a_{in}\delta_{out}$ | $\frac{\partial L}{\partial W} = X^T\cdot\frac{\partial L}{\partial Y}$ | 权重梯度 = 输入 × 上游梯度 |

> 一句话：**Nielsen 的 $\delta$ 回传 = 通用的 $g$ 回传；BP2 是"反穿一层"的通用写法，矩阵乘法反向规则是它在 $y=xW$ 这一特例上的化简。**

---

## 雅可比矩阵是什么

### 雅可比 = "向量到向量函数"的导数

导数的三个层级：

| 函数形式 | "导数"叫什么 | 形状 |
|---|---|---|
| 标量→标量 $f(x)$ | 导数 $df/dx$ | 一个数 |
| 向量→标量 $f(x)$ | 梯度 $\nabla f$ | 向量 $[\partial f/\partial x_1,\ldots,\partial f/\partial x_n]$ |
| **向量→向量 $f(x)$** | **雅可比矩阵 $J$** | **矩阵，$J_{ij}=\partial f_i/\partial x_j$** |

**雅可比 $(i,j)$ 元 = "输出 $i$ 对输入 $j$ 的偏导"**，即"拨动输入 $j$ 一点点，输出 $i$ 变多少"。整张矩阵刻画"每个输出受每个输入多大影响"。

### 反向传播需要它

反向过每个运算本质上是在套链式法则：

$$
\frac{\partial L}{\partial x} = J^T\cdot\frac{\partial L}{\partial y}
\qquad\text{（} y=f(x)\text{，}J\text{ 是 }f\text{ 的雅可比）}
$$

**梯度过一个运算 = 上游梯度乘以这个运算的雅可比的转置。** 雅可比就是那个"局部导数"，是链式法则在向量函数上的具体化。

### 不同运算的雅可比长不一样

- **逐元素运算**（$y=x+5$、$y=\mathrm{relu}(x)$）：每个输出只依赖一个输入 → 雅可比是**对角阵**，反向平凡
- **矩阵乘法 $Y=X\cdot W$**：雅可比有特殊结构，能化简成 $\frac{\partial L}{\partial W}=X^T\cdot g$、$\frac{\partial L}{\partial X}=g\cdot W^T$。**不用真构造大雅可比矩阵**
- **softmax**：每个输出 $A_i$ 依赖**所有**输入 $S_j$（因分母是 $\sum\exp(S)$）→ 雅可比是**满矩阵**，没法化简，须正经算。**这就是反向推导里单独拎出 softmax 的原因。**

### 3 维 softmax 雅可比数值例子

$s=[1,2,3]$，$y=\mathrm{softmax}(s)=[0.090,0.245,0.665]$。雅可比公式 $J_{ij}=y_i\cdot(\delta_{ij}-y_j)$：

$$
J \approx
\begin{bmatrix}
0.082 & -0.022 & -0.060\\
-0.022 & 0.185 & -0.163\\
-0.060 & -0.163 & 0.223
\end{bmatrix}
$$

读出 softmax 的"性格"：

- **对角正**：增大 $s_i$ 增大 $y_i$（被 $(1-y_i)$ 削弱，因已接近饱和）
- **非对角负**：增大 $s_j$ 减小其他 $y_i$（权重和必须 $=1$，一个变大别人让）
- **每行和 $=0$**：$\sum y$ 恒为 1，整体平移 $s$ 不改 $y$

### 反向里怎么用

假设损失想让 $y_1$ 变大，即 $g_y=[1,0,0]$。反向 $g_s=J^T\cdot g_y$ = $J$ 第一行：

$$
g_s = [0.082,\,-0.022,\,-0.060]
$$

含义：要增大 $y_1$，应**增大 $s_1$、减小 $s_2$ 和 $s_3$**（把 $s_1$ 相对抬高）。

### 实际代码不构造这个矩阵

概念上"乘以 $J^T$"，但 softmax 雅可比有闭式化简，实际用公式（不建 $30\times30$ 矩阵再乘）：

$$
g_{s_j} = y_j\cdot\left(g_{y_j} - \sum_i y_i\,g_{y_i}\right)
$$

> **一句话**：雅可比矩阵 = 向量函数的导数，$J_{ij}$ 说"输出 $i$ 受输入 $j$ 多大影响"。反向传播过 softmax 需要它，是因为 softmax 每个输出都耦合依赖所有输入，不像逐元素运算那样可对角化简。

## 相关页

- [[NeuralNetworkBasics]] —— 神经元/MLP 基础与梯度下降/SGD 更新式
- [[Backpropagation-Transformer]] —— 项目视角：从 Loss 传回 $W_Q$ 的完整反传链
- [[TrainingMechanics]] —— 拿到梯度后的优化器（Adam）与归一化
