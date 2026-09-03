---
type: summary
title: Learned Motion Matching — 论文拆解摘要
created: 2026-08-27
updated: 2026-08-27
source: RawSource/articles/Learned-Motion-Matching-2020.md
tags: [motion-matching, learned-motion-matching, paper-summary]
summary: Learned Motion Matching 论文（Holden 2020）的录入摘要与导览，串起 motion-matching 领域全部知识页，原始全文存 RawSource/articles/。
---

# Learned Motion Matching — 论文拆解摘要

> 本页是 [[RawSource/articles/Learned-Motion-Matching-2020|论文原文]] 的派生摘要，作为 motion-matching 领域知识页的导览入口。

## 论文元信息

| 项 | 内容 |
|---|---|
| 标题 | Learned Motion Matching |
| 作者 | Daniel Holden, Oussama Kanoun（Ubisoft La Forge）；Maksym Perepichka, Tiberiu Popa（Concordia University） |
| 发表 | ACM Trans. Graph. (SIGGRAPH 2020), 39(4), Article 1, 2020-07 |
| DOI | 10.1145/3386569.3392440 |
| 定位 | 工业界 + 学术界联合，工业落地的 character animation 论文 |

## 一句话核心

把 [[MotionMatching]] 拆成**投影 / 步进 / 解压**三阶段，各用一个专门小神经网络替代——既保留 MM 的质量与可控性，又获得神经网络的可扩展性（内存不随数据线性增长）。

## 动机

- **MM**：灵活、可预测、预处理快、质量高、可控易调试 → 工业界主流；但内存随数据**线性增长**，难配数据增强
- **神经网络生成模型**（PFNN/MANN）：内存小可扩展、运行时快；但难控制、不可预测、训练久、质量常低于训练集、长序列「dying-out」
- 目标：鱼与熊掌兼得。关键洞察是 MM 由三个可分离操作组成，可**逐个替换**而非整体端到端重做

## 三阶段 → 三网络（核心知识页）

| MM 阶段 | 替代网络 | 消除的库 | 详见 |
|---|---|---|---|
| Projection（投影） | [[Projector]] | X | 抗噪训练模拟最近邻，兼有分布映射 + 防 die-out |
| Stepping（步进） | [[Stepper]] | X、Z 查表 | 自回归预测 (x,z) 增量，承担全部动力学 |
| Decompression（解压） | [[Decompressor]]（+Compressor C） | Y | autoencoder 学隐变量 z∈R³²，FK+速度损失 |

## 数据表示

[[FeatureAndPoseVector]]：特征向量 `x∈R²⁷`（轨迹/足位/足速/髋速）+ 姿态向量 `y`（关节位姿/速度/根速度/任务输出），按标准差归一化 + 用户权重，构成 X、Y 两库。

## 训练

D 必须先训（出 Z），之后 S/P 并行；PyTorch+RAdam，batch 32，lr 0.001×0.99/1k iter，最多 500k iter，单线程 CPU，过夜最优。

## 结果与评估

7 个场景（Locomotion / Terrain / Chair / Interaction / Dog / Bear / User-Study），Bear 用 carpet unrolling 把 3581 帧增强到 ~70 万帧。完整数值对比见 [[Compare-MM-vs-LMM]]：
- 内存：Bear MM 995.6MB → LMM 7.1MB（约 **140×**）
- 速度：LMM 197µs/帧，远快于 PFNN(1370) / MANN(2440)
- 精度：Decompressor 关节误差均值 1.4cm

## 用户研究（§6.4）

34 人在 AAA 制作中实测。受试者难分辨 MM 与 LMM，连动画总监都分不清；32/34 认为可互相替代。LMM「transitions 更平滑」，MM 更「dynamic/snappier」。

## 讨论（§7）

- **多网络而非端到端**：可独立训练/调试/toggle；DMM 单独即折中；姿态与动力学解耦更稳更易扩展（变时间步只缩放 S 输出）；Projector 双重作用
- **主动不泛化**：Projector 模拟最近邻 → 不插值生成新动画，靠数据覆盖 + 增强 + 特征权重控制弥补；状态总贴近训练态

## 局限与未来（§8）

[[Projector]] 精度是质量瓶颈（最大最耗内存，加深层超预算）；未来用 memory layers 提精度、探索端到端训练。

## 附录要点

- **A** Clip 结构：多 clip 打包 + `(start,end,anim_index)` 元组
- **B** 搜索加速：两层 AABB（16/64 帧），C++ 与 Cython 双实现
- **C** 正向运动学递归公式
- **D** 地形拟合：最近邻回归 xy→z 拟合「stepped」地形

## 领域索引

全部知识页见 [[TheWiki/motion-matching/_index|motion-matching 领域索引]]。
