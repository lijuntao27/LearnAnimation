---
type: concept
title: Motion Matching（基础算法）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, animation, nearest-neighbor]
summary: Clavet & Büttner 2015 提出的动画算法，在动画库里做最近邻搜索找最贴合当前上下文与用户轨迹的帧并回放，工业界主流但内存随数据线性增长。
---

# Motion Matching（基础算法）

> 论文 §3。本页描述论文给出的「自家 AAA 产线实现」，与原始 Clavet 2016 版本细节略有差异。

## 定义

Motion Matching（MM）由 Clavet 与 Büttner（2015，为 *For Honor* 开发）提出：在一个动画数据库里**搜索最贴合当前上下文与用户期望的帧**，回放该帧。本质是 Motion Fields 算法（Lee 2010）的贪心近似，也可视为带「匹配成本」的图搜索特例——goal cost 与 pose cost 合并、固定间隔可跳转、贪心遍历。

## 高层流程（每 N 帧）

1. 构造查询特征向量 `x̂`（当前上下文 + 用户意图，见 [[FeatureAndPoseVector]]）
2. 在 Matching Database `X` 上最近邻搜索：`k* = argmin_k ‖x̂ - x_k‖²`
3. 若 `k* ≠ 当前帧 i`，插过渡（[[InertializationBlending]]），`i := k*`
4. 每帧 `i := i+1`，从 Animation Database `Y` 查 `y_i` 回放

## 两个数据库

- `X = [x_0,...,x_{n-1}]` —— **Matching Database**，特征向量
- `Y = [y_0,...,y_{n-1}]` —— **Animation Database**，姿态向量

详见 [[FeatureAndPoseVector]]。

## 加速结构

暴力搜索太慢。论文用**自定义两层 AABB BVH**（16 帧一组、64 帧一组，Appendix B）：顺序遍历缓存友好；到 AABB 的平方距离可按维度累加便于 early-out；内存开销极小。也可选 KD-Tree、voxel lookup、clustering。

## 归一化与权重

每特征按数据集标准差缩放，再乘用户权重调整搜索重要度（默认 1 即够）。可对 X 与 x̂ 做 PCA 降维（非必须，保留足够维数则距离不变）。

## 二值特征（步态/标签）

两种处理：one-hot 编码进 `x`；或预算数据库范围，搜索限定在该范围。

## 优点与局限

| 优点 | 局限 |
|---|---|
| 简单、灵活、可控、易调试 | **内存（及部分性能）随数据量线性增长** |
| 直接回放原始数据 → 质量保真 | 难配合数据增强等手段 |
| 预处理极快、可实时改 → 迭代快 | 多样性与内存预算间恒定权衡 |
| 工具可追踪调试 | |

## 工业界采用

*For Honor*、*EA Sports UFC 3*、*The Last of Us 2* 等多款 AAA；学术上也被用作物理动画的轻量运动学引导（Bergamin 2019 DReCon、Hong 2019）。

## 在本项目中的体现

MM 是 [[LearnedMotionMatching]] 的拆解基础——LMM 把 Projection/Stepping/Decompression 三阶段分别交给 [[Projector]] / [[Stepper]] / [[Decompressor]]。要理解 LMM 必先理解本算法。

## Clip 结构（Appendix A）

实际是多 clip 打包进 X/Y，并存 `(start, end, anim_index)` 元组。有限差分、训练窗口需跨 clip 处理；clip 末触发搜索；禁止返回接近 clip 末尾的帧（避免频繁重触发）。
