---
type: comparison
title: Pan 2022 vs Jin 2024
created: 2026-08-31
updated: 2026-09-02
tags: [cloth-animation, comparison]
summary: 两篇 bone-driven 布料动画论文对比：Pan 2022 用 SSDR+GRU+GNN 一把抓，Jin 2024 用 rope chain 物理+QNN 分工；关键分歧在动态由谁产生——GRU 学时序（费数据泛化差）vs 物理产生（QNN 只学形状省数据）。
---

# Pan 2022 vs Jin 2024

> 两篇 [[cloth-animation/_index|布料动画]] 论文的横向对比。Pan 2022 首篇 bone-driven loose garment；Jin 2024 同研究线后续，直接批评 Pan 的 GRU 路线。

## 核心分歧：动态由谁产生

| | Pan 2022 | Jin 2024 |
|---|---|---|
| 动态来源 | GRU 学时序（body→bone motion 含动态）| [[RopeChain\|rope chain]] 物理产生 |
| 形状来源 | GRU 一并学 + GNN 补 | neural skinning + [[QNN\|QNN]] 学形状 |
| 时序 | 有 hidden state（RNN）| 逐帧独立（准静态）|

## 表示对比

| | Pan 2022 | Jin 2024 |
|---|---|---|
| 骨骼来源 | SSDR 从模拟反解 | 手动/SSDR + 纵向连成绳链 |
| 骨骼结构 | 非层级、全局刚体变换 | 非层级、绳链约束 |
| DOF | 旋转 + 平移 | 只平动（无旋转）|
| 蒙皮 | LBS | neural skinning（LBS/DQS 失败）|

## 数据需求对比

| | Pan 2022 | Jin 2024 |
|---|---|---|
| 训练数据 | 40000 帧/参数组 ×10 组 | 5000 帧 |
| 数据生成 | Houdini Vellum | Houdini Vellum |
| 泛化机制 | holdout + RBF 参数集成 | QNN 准静态 + PINN + PCA 正则 |

## 碰撞对比

| | Pan 2022 | Jin 2024 |
|---|---|---|
| 范围 | garment-body（无 self-collision）| body-virtual bone |
| SDF | 未明用解析 | 解析 SDF + history-based push out |
| loss | 碰撞项 | PINN-style（权重 1000）|

## 适用场景

| | Pan 2022 | Jin 2024 |
|---|---|---|
| 适合 | 有大量模拟数据、要参数变化 | 数据少、要实时、要无穿透 |
| 不适合 | 数据少、要 keyframe 控制 | 要美术 keyframe 控制（QNN 确定性）|

## 对本仓库其他研究的意义

- 两者都证明"裙摆→低自由度骨骼"可行，是 [[MotionInbetweeningTwoStage|motion in-betweening]] 接布料的表示底座
- Jin 的"物理管动态 + QNN 管形状"分工，启发了"keyframe 进训练数据 + 物理补动态"的工程方案

## 关联

- [[Pan2022-VirtualBones]] / [[Jin2024-RopeChainQNN]]
- 原文：[[RawSource/articles/Pan-2022-VirtualBones]] / [[RawSource/articles/Jin-2024-LooseFittingClothing]]

## 谱系澄清（2026-09 补）

- 本文对比的两篇**作者不重叠**：Pan 2022（浙大/NExT/马里兰）与 Jin 2024（Yongxu Jin / Fedkiw，Stanford 组为主）。"Jin 2024 承接并批评 Pan"是**研究线层面**的承接，不是原班作者的续作。
- 2026 Meta 业界线 [[HyperBones2026]] / [[PhySkin2026]] **只引 Pan 不引 Jin**，构成两支独立谱系（Stanford/Fedkiw 支 vs Meta Reality Labs 支）。详见 [[Compare-BoneDrivenCloth-Lineage]]。
