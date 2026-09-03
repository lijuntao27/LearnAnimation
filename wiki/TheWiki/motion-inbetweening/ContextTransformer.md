---
type: entity
title: Context Transformer（阶段一）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, transformer, context-model]
summary: 两阶段框架第一级，Transformer Encoder+掩码自注意力，输入上下文/目标帧+占位过渡输出粗略过渡；含 Keyframe PE、Learned Relative PE、encoder/decoder MLP，约 10.4M 参数。论文 §3.3。
---

# Context Transformer（阶段一）

> [[MotionInbetweeningTwoStage|论文]] 两阶段框架的第一级。论文 §3.3，代码 `packages/motion_inbetween/model/__init__.py` 的 `ContextTransformer` 类、`train/context_model.py`。

## 职责

输入上下文帧 + 目标帧 + 占位过渡帧，**生成粗略过渡动画**（把握全局走向）。不负责细节精修（那是 [[DetailTransformer]] 的事）。

## 网络结构

[[Transformer|Transformer Encoder]] + 掩码自注意力，外面包 encoder/decoder MLP：

```
x_ctx ∈ R^(T×135)            ← 输入（含占位）
  │ 拼 mask m_in → R^(T×136)
  │ encoder MLP: Linear(136→512)→PReLU→Linear(512→512)→PReLU
  ▼
R^(T×512)
  │ + Keyframe Positional Encoding
  ▼
R^(T×512)
  │ 6 层 Transformer 层:
  │   每层 = 掩码多头自注意力(带相对位置编码) + 位置式 FFN
  │   pre-norm（LayerNorm 在子层输入前）
  │   残差连接
  ▼
R^(T×512)
  │ 最终 LayerNorm
  │ decoder MLP: Linear(512→512)→PReLU→Linear(512→135)
  ▼
y_ctx ∈ R^(T×135)            ← 输出粗略过渡
```

> 论文称 encoder-decoder 结构，但"decoder"只是两层 MLP，不是自回归 seq2seq decoder。详见 [[RecurrentNetworks|Encoder-Decoder 术语陷阱]]。

## 超参

`d_model=512, n_layer=6, n_head=8, d_head=64, d_pff_inner=512, d_encoder_in=136, d_out=135, dropout=0.0, pre_lnorm=true`。约 **10.4M 参数**。

## 输入构造（关键）

- **缺失帧占位**：根位置在已知帧间**线性插值**（`set_placeholder_root_pos`）、关节旋转**置零**（6D 下即单位旋转）
- **输入掩码 `m_in`**：每帧一个二值通道（`d_mask=1`），拼接在输入后 → 136 维。指示哪些值有效。区分"真值"与"占位值"
- **注意力掩码 `m_atten`**：只有上下文帧、受约束帧、目标帧能被 attend（作为 key），占位过渡帧不能（论文 Figure 5a）

```
Attention(Q,K,V) = softmax((QK^T + S_rel)/√d + m_atten)·V
```

注意力掩码的实现：`context_model.get_attention_mask`，对被屏蔽位置填 `-inf`，softmax 后权重 → 0。详见 [[SelfAttention|掩码做补间的机制]]。

## 两种位置编码（都在这层）

- **Keyframe Positional Encoding**：加到 encoder 输出上。详见 [[InbetweeningPositionalEncodings]]
- **Learned Relative Positional Encoding**：注入注意力分数（`S_rel`，经 skew 机制）。详见 [[InbetweeningPositionalEncodings]]

## 损失

`L_ctx = α_pos L_pos + α_rot L_rot + α_smooth L_smooth`，权重 `α_pos=0.01, α_rot=1.0, α_smooth=0.005`。三项均在过渡帧上评估：

- `L_rot`：局部旋转 L1
- `L_pos`：全局位置 L1（经 FK，隐式加权关节朝向）
- `L_smooth`：相邻帧全局位置差 L1，保证两端平滑

详见 [[MotionInbetweeningTwoStage|论文 §3.5/§3.6]]。

## 推理时的后处理 ψ

输出进 [[DetailTransformer]] 前，非过渡帧回填真值，并应用后处理 ψ：定义逐通道偏移 `δ*` 加到整段过渡，使最后一个上下文帧与目标帧处**运动曲线曲率最小**。`κ` 用相邻线段夹角近似。**只在 Context 推理时应用**。代码 `train/utils.anim_post_process`。

## 训练

- 先训练 Context，训完**冻结参数**，再用其预测作输入训 [[DetailTransformer]]
- 每迭代随机采样过渡长度（5~30）与受约束帧（候选 10%，每帧 50% 启用）
- [[TrainingMechanics|Adam + Noam]]（warmup 8000），`lr=0.0625`
- 不用 dropout
- 产出 Table 1 数值的迭代数：181k

## 完整前向数据流（公式级）

见 [[Transformer]] 的"完整前向数据流"节，标注了每步形状（30×135 → 30×135）。

## 与论文的差异（代码阅读注意）

1. **部分姿态约束未完整实现**：论文称支持 `m_in ∈ R^{T×D}` 逐维部分约束，但代码 `d_mask=1`、`constrained_slices=[[null,null]]`，实际只有"整帧全约束/无约束"
2. **注意力缩放用 `1/√d_model`（1/√512）而非论文式的 `1/√d_K`（1/√64）**：见 `RelMultiHeadedAttention.atten_scale`
3. Context 训练窗口 41 vs 论文公式的 42：小不一致，不影响（后处理只在推理用，推理窗口是 `context+trans+2`）

## 相关页面

- [[MotionInbetweeningTwoStage]] —— 论文总览
- [[DetailTransformer]] —— 第二级
- [[InbetweeningPositionalEncodings]] —— 两种位置编码
- [[Transformer]] / [[SelfAttention]] / [[Backpropagation]] / [[TrainingMechanics]] —— 底层概念
