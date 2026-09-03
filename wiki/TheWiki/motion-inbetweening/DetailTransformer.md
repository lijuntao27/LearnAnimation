---
type: entity
title: Detail Transformer（阶段二）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, transformer, detail-model]
summary: 两阶段框架第二级，结构同 Context 但无 Keyframe PE、注意力掩码更宽松；输入 Context 输出精修细节并预测 4 维足部接触供 IK 后处理，约 10.1M 参数。论文 §3.4。
---

# Detail Transformer（阶段二）

> [[MotionInbetweeningTwoStage|论文]] 两阶段框架的第二级。论文 §3.4，代码 `packages/motion_inbetween/model/__init__.py` 的 `DetailTransformer` 类、`train/detail_model.py`。

## 职责

输入 [[ContextTransformer|Context Transformer]] 的输出，**精修运动细节**（补足被 Context 抹平的高频细节、预测足部接触供 IK 消除脚滑）。不负责从零生成过渡。

## 网络结构

与 [[ContextTransformer]] 几乎相同，但有三处关键差异：

| 差异点 | Context | Detail | 原因 |
|---|---|---|---|
| Keyframe PE | 有 | **无** | 精修任务下"距关键帧距离"不再关键 |
| 注意力掩码 | 严：过渡帧不能作 key | **宽**：上下文/过渡/目标帧可自由互相 attend | 输入已无占位值，全是真实或 Context 预测值 |
| 输出维度 | 135 | **139**（135 + 4 足部接触） | 额外预测足部接触 |

结构同前：

```
x_det ∈ R^(T×136)            ← 输入 = Context 输出（过渡部分）+ 真值（非过渡/约束部分）
  │ encoder MLP (136→512→512, PReLU)
  │ （无 Keyframe PE）
  │ 6 层 Transformer 层（掩码多头自注意力 + FFN，相对位置编码，pre-norm）
  │ 最终 LayerNorm
  │ decoder MLP (512→512→139)
  ▼
y_det ∈ R^(T×139)            ← 输出：135 姿态 + 4 足部接触
```

约 **10.1M 参数**。

## 输入构造

- **过渡部分的值 = Context 的输出 `y_ctx`**（粗略过渡）
- **非过渡帧**：回填真值（在 Context 推理后处理时已做）
- **约束姿态**：只复制未约束关节的值、约束关节保留真值（`reset_constrained_values`）
- 注意力掩码更宽松：上下文帧、过渡帧、目标帧可自由互相 attend（论文 Figure 5b）——因输入已无占位值

> 实现细节：`detail_model.get_model_input` 返回 `(state, delta)`，但 `delta`（帧间差分）算出来了却**从未喂给模型**，属遗留代码。详见 [[RawSource/articles/Motion-In-betweening-2022|原文 §12.5]]。

## 足部接触输出

额外输出 `ĉ ∈ R^(T×4)`（左右脚 + 脚趾，1=触地）：

```
ĉ = sigmoid(C(y_det))    ← C 提取接触维度（135:139）
```

`sigmoid` 把输出压到 0~1。可配合 IK 消除脚滑（触地时脚速度应≈0）。

## 损失

`L_det = β_pos L_pos + β_rot L_rot + β_c L_c + β_foot L_foot`，权重 `β_pos=0.02, β_rot=1.0, β_c=0.05, β_foot=0.05`：

- `L_rot`、`L_pos`：同 Context
- `L_c`：足部接触重建 L1（`cal_c_loss`）
- `L_foot`：脚滑损失 `‖ĉ_t·v̂_t‖₁`（触地时脚/脚趾全局速度应≈0，`cal_f_loss`）

**关键：`L_foot` 必须切断 `ĉ` 的梯度回传**（代码 `cal_f_loss` 里 `c_out.detach()`）。否则优化会让 `ĉ` 恒为 0 来作弊（接触全 0 则脚滑损失变 0）。这是 [[Backpropagation|反向传播]]时"哪条路该断"的实例。

## 训练

- **两阶段顺序训练**：先训完并冻结 [[ContextTransformer|Context Transformer]]，再用其预测作输入训 Detail
- 每迭代随机采样过渡长度与受约束帧（同 Context）
- [[TrainingMechanics|Adam + Noam]]（warmup 8000），`lr=0.025`
- 不用 dropout（`dropout=0.0`；`update_dropout_p` 余弦退火逻辑存在但配置为 0，不生效，与论文结论一致）
- **随机姿态约束采样对 Detail 有效**（附录 C）：缓解过拟合、改善验证指标
- 产出 Table 1 数值的迭代数：1430k（比 Context 多近 8 倍，因精修更难）

## 推理

完整两阶段推理流程（`eval_detail_model.py`）：

```
load 两个模型 checkpoint
取单个 clip → 切片到 window_len = context + trans + 2
→ 对齐坐标系（to_start_centered_data）
→ Context 推理 + 后处理
→ Detail 推理（可选后处理）
→ 反 z-score → FK
→ 输出预测与 GT 的 JSON（positions/rotations/foot_contact/parents）供 Maya 加载
```

`det_mdl.evaluate` 调用 `ctx_mdl.evaluate` 得到粗略过渡，再跑 Detail 精修。

## 与论文的差异

1. `delta` 算了未用（见上）
2. `Discriminator` 类与 `dw=0.0` 权重存在但未使用——疑为早期 GAN 训练残留
3. dropout 余弦退火逻辑存在但配置为 0.0

详见 [[RawSource/articles/Motion-In-betweening-2022|原文 §12.5]]。

## 相关页面

- [[MotionInbetweeningTwoStage]] —— 论文总览
- [[ContextTransformer]] —— 第一级
- [[InbetweeningPositionalEncodings]] —— Detail 仍用 Learned Relative PE，但无 Keyframe PE
- [[Transformer]] / [[SelfAttention]] / [[Backpropagation]] / [[TrainingMechanics]] —— 底层概念
