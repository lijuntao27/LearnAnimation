---
type: concept
title: 反向传播：Transformer 中的 W_Q 梯度
created: 2026-09-07
updated: 2026-09-07
tags: [deep-learning, backpropagation, transformer, self-attention, gradient]
summary: 项目视角的反向传播实例：梯度从 Loss 经 decoder/FFN/残差/W_O/A·V/softmax/Q·K^T 一路传回 W_Q，最终形式 ∂L/∂W_Q=z^T·g_Q 与 MLP 的 ∂L/∂W=x^T·g_y 完全同形；softmax 需满雅可比单独处理。通用机制见 Backpropagation。
---

# 反向传播：Transformer 中的 W_Q 梯度

> 本页是**项目应用页**：以 Transformer 的注意力权重 $W_Q$ 为例，走一遍"从 Loss 一路传回参数"的完整反传链。
> 通用机制（链式法则、局部性、矩阵乘法反向规则、Nielsen 四公式 BP1–BP4、雅可比矩阵）见 [[Backpropagation]]。

## 目标

算出 $\partial L/\partial W_Q$，用于 [[TrainingMechanics|Adam]] 更新 $W_Q$。核心工具就是 [[Backpropagation]] 里的**矩阵乘法反向规则**：

$$
\frac{\partial L}{\partial W} = X^T\cdot\frac{\partial L}{\partial Y}
\qquad\text{（对参数）}\qquad
\frac{\partial L}{\partial X} = \frac{\partial L}{\partial Y}\cdot W^T
\qquad\text{（往下传）}
$$

反复套用，softmax 处换成满雅可比即可。

## 前向数据流

记 $g_X=\partial L/\partial X$。单头简化（省略相对 PE/缩放/掩码——它们是逐元素加减乘，梯度平凡）。前向：

$$
z = \mathrm{LayerNorm}(H),\quad
Q = z\cdot W_Q,\quad
K = z\cdot W_K,\quad
V = z\cdot W_V
$$

$$
S = Q\cdot K^T\ (30\times30),\quad
A = \mathrm{softmax}(S)\ (30\times30),\quad
O = A\cdot V\ (30\times64)
$$

$$
O \to W_O \to +\text{残差} \to \text{FFN} \to \text{下层} \to \cdots \to \text{decoder} \to Y \to \text{Loss}
$$

形状（单头，$d_{head}=64$，$seq=30$）：$z:30\times512$，$W_Q:512\times64$，$Q:30\times64$，$K:30\times64$，$S,A:30\times30$，$V:30\times64$，$O:30\times64$。

## 倒着走

**起点**：$g_Y=\partial L/\partial Y$（L1 损失：$g_Y=\mathrm{sign}(Y-Y_{gt})$）。

**穿过 decoder MLP + 最终 LayerNorm + 5 层（残差+FFN+attention）**：每步用矩阵乘法反向规则，一路传到当前层 $\ell$ 的 attention 子层输出，带着 $g_{att}=\partial L/\partial(\text{该层 attention 输出})$。

**attention 子层内部倒退（单头）**：

(a) 穿过 $W_O$：

$$
g_O = g_{att}\cdot W_O^T
\qquad (30\times64)
$$

(b) $O = A\cdot V$：

$$
g_A = g_O\cdot V^T
\qquad (30\times64\cdot64\times30 = 30\times30)
$$

$$
g_V = A^T\cdot g_O
\qquad \text{（传给 } V \text{ 那条路）}
$$

(c) $A = \mathrm{softmax}(S)$：

$$
g_S = \mathrm{softmax\_jac}(g_A,\,A)
\qquad (30\times30)
\quad\text{（见 [[Backpropagation]] 的满雅可比）}
$$

(d) $S = Q\cdot K^T$：

$$
g_Q = g_S\cdot K
\qquad (30\times30\cdot30\times64 = 30\times64)
$$

$$
g_K = g_S^T\cdot Q
\qquad \text{（传给 } K \text{ 那条路）}
$$

(e) $Q = z\cdot W_Q$：

$$
\boxed{\,g_{W_Q} = z^T\cdot g_Q\,}
\qquad (512\times30\cdot30\times64 = 512\times64)
$$

$$
g_z = g_Q\cdot W_Q^T
\qquad \text{（传给 } z \text{，过 LayerNorm 回 } H \text{，与残差路汇合）}
$$

**$g_{W_Q}$ 就是 $\partial L/\partial W_Q$，梯度传到了。** 形状 $512\times64$ 与 $W_Q$ 完全一致 ✓

## 全链展开

$$
g_{W_Q}
= z^T\cdot g_Q
= z^T\cdot\big(g_S\cdot K\big)
= z^T\cdot\big(\mathrm{softmax\_jac}(g_A,A)\cdot K\big)
$$

$$
= z^T\cdot\big(\mathrm{softmax\_jac}(g_O\cdot V^T,\,A)\cdot K\big)
= z^T\cdot\big(\mathrm{softmax\_jac}((g_{att}\cdot W_O^T)\cdot V^T,\,A)\cdot K\big)
$$

$g_{att}$ 来自更下游（残差、FFN、下一层、……、decoder、$Y$、Loss）的一路反传。

> 多头：每个头 $h$ 独立走一遍，得 $g_{W_Q}^{(h)}=z^T\cdot g_Q^{(h)}$（各 $512\times64$）；8 个头沿输出维拼起来就是大 $W_Q$（$512\times512$）的完整梯度。$z$ 在所有头间共享。

## 关键洞察：和 MLP 一模一样

盯着最终结果：

$$
g_{W_Q} = z^T\cdot g_Q
\qquad\text{（attention 的 } W_Q \text{ 梯度）}
$$

对比 MLP 一层 $y=x\cdot W$：

$$
g_W = x^T\cdot g_y
\qquad\text{（MLP 的 } W \text{ 梯度）}
$$

**完全同形。** 因为对 $W_Q$ 而言，$Q=z\cdot W_Q$ 就是一个矩阵乘法层，和 MLP 的 $y=x\cdot W$ 没有本质区别。

attention 里那些"吓人"的部分（softmax、$Q\cdot K^T$），在反向传播里**只是变换梯度信号的中间运算**——它们把"对 attention 输出的梯度 $g_{att}$"，经 $W_O$、$A\cdot V$、softmax、$Q\cdot K^T$ 一路转成"对 $Q$ 的梯度 $g_Q$"。**到了 $Q=z\cdot W_Q$ 这一步，套用的就是最朴素的矩阵乘法反向规则。**

> 所以 attention 的 $W_Q$（及 $W_K$、$W_V$、$W_O$、FFN、PE MLP、$\gamma/\beta$）全部用和 MLP 完全一样的 SGD/反向传播训练。框架的 autograd 自动完成，没人手写。所谓"学 attention"就是链式法则把损失梯度传回这些 $W$。

## 更新（闭环）

拿到 $g_{W_Q}$ 后，[[TrainingMechanics|Adam]] 更新（所有参数同时各拿各的梯度）：

```
W_Q ← W_Q − lr · Adam(g_W_Q)
W_K ← W_K − lr · Adam(g_W_K)
...（W_V, W_O, FFN, encoder/decoder, PE MLP, γ/β 同理）
```

重复几十万次 → $W_Q$ 等参数被逐渐调成"让损失最低"的值 → attention 学会"该关注谁"。

**闭环**：前向 $Q=z\cdot W_Q$ 产生预测 → 算损失 → 反向算 $\partial L/\partial W_Q$ → Adam 调 $W_Q$ → 下次前向预测更好。

## 在本项目中的体现

- $\partial L/\partial W_Q$ 的完整反传链是 [[ContextTransformer]] / [[DetailTransformer]] 训练时的核心
- 训练用 [[TrainingMechanics|Adam]] + Noam 调度执行更新
- `L_foot` 脚滑损失对接触 $\hat{c}$ 做 `.detach()` 切断梯度，正是"用雅可比/反向规则时要小心哪条路该断"的实例（否则网络会把 $\hat{c}$ 学成恒 0 来作弊）
- 后处理 $\psi$（`anim_post_process`）在推理时用，不参与训练梯度

## 相关页

- [[Backpropagation]] —— 通用机制：矩阵乘法反向规则、Nielsen 四公式、雅可比矩阵
- [[NeuralNetworkBasics]] —— 神经元/MLP 基础
- [[SelfAttention]] —— 注意力前向（$Q/K/V$、softmax、掩码）
- [[TrainingMechanics]] —— Adam 更新与归一化
