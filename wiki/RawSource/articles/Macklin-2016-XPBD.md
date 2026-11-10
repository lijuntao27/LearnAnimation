---
type: raw-article
title: XPBD: Position-Based Simulation of Compliant Constrained Dynamics (Macklin et al., MIG 2016)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, xpbd, pbd, compliance, lagrange-multiplier, mig-2016, real-time]
summary: Macklin、Müller、Chentanez（NVIDIA）MIG 2016 的 XPBD 论文：从隐式时间离散出发为 PBD 引入柔度矩阵 α 与总拉格朗日乘子 λ，使约束刚度与迭代数/时间步解耦、对应明确弹性势能并可输出约束力（触觉、可断关节）；对 PBD 求解器仅加 3 行（每约束多存一个标量，开销 <2%），悬臂梁 FEM 20 次迭代即与 Newton 参考解视觉无异。中文译本见 [[Macklin-2016-XPBD-中文翻译]]。
---

# XPBD: Position-Based Simulation of Compliant Constrained Dynamics (MIG 2016)

> 原文 PDF：`Macklin-2016-XPBD.pdf`（本目录，ACM 数字图书馆正式版）；提取文本：`Macklin-2016-XPBD.txt`。
> 中文译本：[[Macklin-2016-XPBD-中文翻译]]。
> 谱系定位：[[XPBD]]（本论文的概念页）· 直系上游 [[Muller-2006-PositionBasedDynamics|PBD]]（Müller 2006）· 理论源头 Servin et al. 2006 柔顺约束。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Miles Macklin, Matthias Müller, Nuttapong Chentanez |
| 机构 | NVIDIA |
| 发表 | Motion in Games (MiG '16), 2016, Burlingame, CA；DOI 10.1145/2994258.2994272 |
| 验证 | 与基于 Cholesky 的非线性 Newton 参考求解器对比 |

## 核心机制一句话

**把柔顺约束（compliant constraint）塞进 PBD**：从隐式离散的运动方程 $M\ddot{x} = -\nabla U^T(x)$ 出发，把弹性势能写成 $U(x) = \frac{1}{2} C(x)^T \alpha^{-1} C(x)$（柔度 $\alpha$ = 刚度倒数），引入总拉格朗日乘子 $\lambda$ 后取两个近似（$K \approx M$ 省掉约束 Hessian、$g(x_i, \lambda_i) = 0$），得到与 PBD 几乎同形的 Gauss-Seidel 更新——但刚度成为物理参数。

## 关键设计

- **病根定位**：PBD 把刚度 $k$ 当投影缩放系数直接乘，有效刚度随迭代数与时间步变化且非线性——软硬体同场时调一个对象全体遭殃，资产无法复用
- **柔度折叠**：$\tilde\alpha = \alpha / \Delta t^2$ 把时间步折进柔度矩阵——刚度与时间步解耦的关键一步
- **核心更新式**（每约束一个标量乘子）：

  $$\Delta\lambda_j = \frac{-C_j(x_i) - \tilde\alpha_j \lambda_i^j}{\nabla C_j M^{-1} \nabla C_j^T + \tilde\alpha_j}$$

  $\alpha_j = 0$ 时精确退化为原 PBD 的缩放因子 $s_j$——PBD 原来是"无限刚硬约束的增量乘子"
- **实现代价**：相对原 PBD 算法只加第 4、7、9 行（初始化 $\lambda$、算 $\Delta\lambda$、累加 $\lambda$）；每约束存一个标量；实测开销 <2%
- **约束力免费获得**：总乘子 $\lambda$ 即约束力估计——触觉设备、力触发效果（可断关节）直接可用
- **阻尼**：Rayleigh 耗散势 $D = \frac{1}{2} \dot{C}^T \beta \dot{C}$，与弹性乘子合并为单个更新式（$\gamma_j = \tilde\alpha_j \tilde\beta_j / \Delta t$）
- **FEM 接入**：三角形线弹性单元的应变向量当约束函数、柔度矩阵 = Lamé 参数刚度矩阵之逆——正确耦合各应变分量（Poisson 效应），可用实验材料参数

## 主要结果

- **谐振子**：$\alpha = 0.001$ 距离约束，PBD 1/5/10 次迭代振荡周期与阻尼各不相同且全错；XPBD 任意时间步/迭代数下贴合解析解
- **20 粒子链**：顶端约束力 100 帧最大相对误差 6%（50 迭代）/ 2%（100）/ 0.5%（1000）
- **悬臂梁**（St Venant-Kirchhoff 三角 FEM，$E=10^5$，$\nu=0.3$）：20 次迭代与 Newton 参考解视觉无差
- **布料**：64×64 粒子 24k 距离约束，迭代数 20→160 时 PBD 越来越硬越来越闷，XPBD 行为定性不变（故意用低 $k=0.01$ 放大对比）

## 局限（论文自认）

- $\alpha = 0$ 极限下等同 $k=1$ 的 PBD——要刚硬结果同样需要迭代数，收敛不比 PBD 快
- 迭代未收敛就停止 → 人工柔度（与 PBD 同病）
- 只是隐式 Euler 的近似（非 Projective Dynamics 那样的精确形式），要精度保证的应用仍应选传统方法
- 未来方向：用上一帧的 $\lambda$ 热启动求解（时间相干性）、Chebyshev 加速

## 谱系与关联

- **上游**：[[Muller-2006-PositionBasedDynamics|PBD]]（被改造对象）；Servin et al. 2006（柔顺约束公式来源）；Goldenthal 2007 Fast Projection（变分视角，XPBD = 其柔顺版本）
- **同期对比**：Projective Dynamics (Bouaziz 2014)、ADMM (Narain 2016)——全局法预分解矩阵，撕裂/断裂拓扑变化时要重分解且实现复杂；XPBD 以局部迭代换取简单与鲁棒
- **后代**：Small Steps 2019（子步化把"迭代数"也变成物理量）；[[PyrrhaPhysics]] 等业界实现如今默认 XPBD
