---
type: raw-article
title: Position Based Fluids (Macklin & Müller, SIGGRAPH 2013)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, fluids, sph, density-constraint, siggraph-2013, real-time]
summary: Macklin 与 Müller（NVIDIA）SIGGRAPH 2013 论文：把 SPH 不可压缩性改写成 PBD 密度约束 C_i = ρ_i/ρ_0 − 1 = 0 逐粒子迭代投影，继承 PBD 无条件稳定可用大时间步（传统 SPH/PCISPH 需小步长）；人工压力项改进粒子分布兼产表面张力并降低邻居数要求，涡度约束恢复能量损失；128k 粒子 10ms/帧，NVIDIA Flex 源头之一。
---

# Position Based Fluids (SIGGRAPH 2013)

> 原文 PDF：`Macklin-2013-PositionBasedFluids.pdf`（本目录，ACM TOG 32(4) 预印本）；提取文本：`Macklin-2013-PositionBasedFluids.txt`。
> 谱系定位：[[PositionBasedDynamics]] 扩展到流体 · 阅读顺序见 [[PBDReadingGuide]] 选修篇目。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Miles Macklin, Matthias Müller |
| 机构 | NVIDIA |
| 发表 | ACM Transactions on Graphics (SIGGRAPH 2013) 32(4), Article 104 |

## 核心机制一句话

**把"不可压缩"写成 PBD 约束**：对每个粒子 $i$ 建立密度约束

$$C_i(\mathbf{p}_1, ..., \mathbf{p}_n) = \frac{\rho_i}{\rho_0} - 1 = 0$$

其中 $\rho_i$ 是 SPH 核加权的邻域密度，$\rho_0$ 是静止密度——然后用标准 PBD 位置投影迭代求解，密度超标就把邻域粒子按梯度推开。不可压缩性从"刚性状态方程算压力"变成"几何投影"。

## 关键设计

- **为何选 PBD 当宿主**：SPH 的病在刚性状态方程（EOS）产生大力限制时间步；PCISPH 2009 用预测-校正迭代缓解但仍需小步；PBD 无条件稳定 + 大时间步——正对实时需求
- **自由面粒子缺失问题**：表面粒子邻居不足、密度估计偏低，直接投影会把表面粒子错误吸向内部——论文给出按邻居缺失修正缩放因子的处理
- **人工压力项（artificial pressure）**：给密度约束加一个小量 $s_{corr}$——一石三鸟：改善粒子分布（防成串/成团）、产生**表面张力**（粒子间距小于平衡距时相互吸引）、**降低邻居数要求**（传统 SPH 需要密集邻居，PBF 稀疏邻居也稳）
- **涡度约束（vorticity confinement）**：位置投影和隐式积分一样有数值耗散，把小尺度漩涡抹掉——速度后处理阶段向漩涡重心注入恢复力
- **XSPH 黏性**：邻域速度平滑实现人工黏性

## 主要结果

- 兔子洗澡场景：128k 粒子、2 子步、每帧 3 次密度迭代，平均 **10ms/帧**
- 与 PCISPH 对比：同等不可压缩度与收敛速度，但时间步可大幅放大、无刚性 EOS 调参
- 椭球泼溅渲染实时出表面

## 局限

- 密度迭代是 Jacobi 式（粒子级并行），但每帧须重建邻居表——邻居查找是主要成本
- 人工压力与涡度参数是"艺术旋钮"而非物理参数（XPBD 精神在流体上的彻底化要等后续工作）
- 大时间步下 CFL 限制仍存在（粒子每步移动不超过半径的某比例）

## 谱系与关联

- **上游**：[[Muller-2006-PositionBasedDynamics|PBD 2006]]；Müller 2003 交互 SPH（同作者前身）、PCISPH 2009
- **下游**：[[Macklin-2014-UnifiedParticlePhysics|Unified Particle Physics 2014]] 把 PBF 吸收为统一框架的流体模块（两篇是同组连续工作）；NVIDIA Flex SDK 的流体即 PBF
- **与 [[Macklin-2019-SmallSteps|Small Steps 2019]] 的接口**：后者用 PBF 做流体实验（936k 粒子水柱），子步化让密度约束收敛更好、可用更大 CFL——两篇连读能看到"迭代 vs 子步"在流体上的直接对比
