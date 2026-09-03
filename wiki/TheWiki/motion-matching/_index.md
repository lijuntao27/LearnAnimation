---
type: index
title: motion-matching 领域索引
updated: 2026-08-27
---

# motion-matching 领域索引

> 角色动画 Motion Matching 技术研究知识域。基础算法、学习方法、数据表示、网络组件、横向对比。
> 论文原文：[[RawSource/articles/Learned-Motion-Matching-2020|Learned Motion Matching (2020)]]。

## 知识页

| 页面 | 类型 | 派生? | Summary |
|---|---|---|---|
| [[MotionMatching]] | concept | 原生 | Clavet & Büttner 2015 提出的动画算法，在动画库里做最近邻搜索找最贴合当前上下文与用户轨迹的帧并回放，工业界主流但内存随数据线性增长。 |
| [[LearnedMotionMatching]] | concept | 原生 | Holden 等 2020 论文方法，把 Motion Matching 拆为投影/步进/解压三阶段各用一个专门小神经网络替代，消除动画库内存依赖，兼得 MM 质量可控与 NN 可扩展性。 |
| [[FeatureAndPoseVector]] | concept | 原生 | Motion Matching 的两套数据表示：x 描述上下文与用户轨迹（R27），y 完整描述一帧姿态；按标准差归一化加用户权重，构成 Matching/Animation Database。 |
| [[Decompressor]] | entity | 原生 | Learned Motion Matching 三网络之一，用 autoencoder 结构替代 Animation Database Y 的逐帧查表；输入 (x, z) 输出 pose y，是 DMM 单独可用形态。 |
| [[Stepper]] | entity | 原生 | Learned Motion Matching 三网络之一，自回归地预测 (x,z) 的帧间增量，替代 Matching/隐变量库的逐帧查表，承担系统全部动力学。 |
| [[Projector]] | entity | 原生 | Learned Motion Matching 三网络之一，用抗噪训练模拟最近邻搜索，从用户查询 x̂ 直接映射到 (x, z)，消除对 Matching Database X 的依赖；兼有分布映射与防递归漂移作用。 |
| [[Compare-MM-vs-LMM]] | comparison | 原生 | 以论文 Table 3 数值对比 MM/DMM/LMM/PFNN/MANN 的内存、运行时、训练时间，Locomotion 5.3MB vs 52.1MB、Bear 约 140× 压缩；LMM 比 MM 慢但远快于 PFNN/MANN。 |
| [[Summary-LearnedMotionMatching]] | summary | 派生 | Learned Motion Matching 论文（Holden 2020）的录入摘要与导览，串起 motion-matching 领域全部知识页，原始全文存 RawSource/articles/。 |

## 关系速览

```
RawSource/articles/Learned-Motion-Matching-2020
        │ (派生摘要)
        ▼
Summary-LearnedMotionMatching ──导览──> 领域全部知识页
                                        │
          ┌─────────────────────────────┼──────────────────┐
          ▼                             ▼                  ▼
   MotionMatching ──拆解──> LearnedMotionMatching      Compare-MM-vs-LMM
          │                        │                        ▲
          │ 数据表示                 │ 三网络                  │ 数值来源
          ▼                        ▼                        │
   FeatureAndPoseVector    Decompressor / Stepper / Projector
```

## 阅读路径建议

1. [[MotionMatching]] —— 先理解基础算法
2. [[FeatureAndPoseVector]] —— 再看数据表示
3. [[LearnedMotionMatching]] —— 核心思想与三阶段拆解
4. [[Decompressor]] / [[Stepper]] / [[Projector]] —— 三网络细节
5. [[Compare-MM-vs-LMM]] —— 数值对比与取舍
6. [[Summary-LearnedMotionMatching]] —— 回到论文级摘要
