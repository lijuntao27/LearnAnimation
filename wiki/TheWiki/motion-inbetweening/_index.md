---
type: index
title: motion-inbetweening 领域索引
created: 2026-08-27
updated: 2026-08-27
---

# motion-inbetweening 领域索引

> 角色动画补间技术研究知识域。两阶段 Transformer 补间方法、两种自定义位置编码、网络组件、横向对比。
> 论文原文：[[RawSource/articles/Motion-In-betweening-2022|Motion In-betweening via Two-stage Transformers (2022)]]。

## 知识页

| 页面 | 类型 | 派生? | Summary |
|---|---|---|---|
| [[MotionInbetweeningTwoStage]] | concept | 原生 | Qin 等 2022 论文方法，用 Context+Detail 两级 Transformer Encoder 一次性生成变长补间动画；掩码自注意力+Keyframe/Relative 位置编码实现超长泛化，LAFAN1 上以更少参数超越 RNN/CNN/Transformer 各流派 SOTA，落地 Maya 工具提速约 2 倍。 |
| [[ContextTransformer]] | entity | 原生 | 两阶段框架第一级，Transformer Encoder+掩码自注意力，输入上下文/目标帧+占位过渡输出粗略过渡；含 Keyframe PE、Learned Relative PE、encoder/decoder MLP，约 10.4M 参数。 |
| [[DetailTransformer]] | entity | 原生 | 两阶段框架第二级，结构同 Context 但无 Keyframe PE、注意力掩码更宽松；输入 Context 输出精修细节并预测 4 维足部接触供 IK 后处理，约 10.1M 参数。 |
| [[InbetweeningPositionalEncodings]] | concept | 原生 | 论文提出两种位置编码：Keyframe PE 用 MLP 把每帧距上下文/目标帧的距离映射成编码注入特征流；Learned Relative PE 用 MLP+skew 把两帧相对距离注入注意力分数；都用 MLP 生成而非固定表，故能泛化到训练未见的超长过渡。 |
| [[Compare-Inbetweening-Methods]] | comparison | 原生 | 横向对比 ERD-QV(RNN)/MC-Trans(Transformer+1Dconv)/Δ-Interp(Enc-Dec+delta)/本文(Enc-only 两阶段)；本文以约一半参数在正常长度提升 16%、超长 45 帧提升 55%，推理 12.1ms vs 33ms。含 LAFAN1 数据集与 L2P/L2Q/NPSS 指标说明。 |
| [[Summary-MotionInbetweeningTwoStage]] | summary | 派生 | Motion In-betweening 论文（Qin 2022）的录入摘要与导览，串起 motion-inbetweening 领域全部知识页，原始全文存 RawSource/articles/。 |

## 关系速览

```
RawSource/articles/Motion-In-betweening-2022
        │ (派生摘要)
        ▼
Summary-MotionInbetweeningTwoStage ──导览──> 领域全部知识页
        │
        ▼
MotionInbetweeningTwoStage ──两阶段──> ContextTransformer + DetailTransformer
        │                                   │
        │ 核心创新                            │ 精修+足部接触
        ▼                                   ▼
InbetweeningPositionalEncodings      (Detail 仍用 Relative PE，但无 Keyframe)
        │
        ▼
Compare-Inbetweening-Methods ──对比──> ERD-QV(RNN) / MC-Trans / Δ-Interp(Enc-Dec)
```

## 阅读路径建议

1. [[MotionInbetweeningTwoStage]] —— 论文总览
2. [[ContextTransformer]] → [[DetailTransformer]] —— 两个网络细节
3. [[InbetweeningPositionalEncodings]] —— 核心创新（泛化超长）
4. [[Compare-Inbetweening-Methods]] —— 实验结果与对比
5. [[Summary-MotionInbetweeningTwoStage]] —— 回到论文级摘要
6. 底层概念 → [[TheWiki/deep-learning/_index|deep-learning 领域]]
