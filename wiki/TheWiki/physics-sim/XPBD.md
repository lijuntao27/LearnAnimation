---
type: concept
title: XPBD（Extended Position Based Dynamics，扩展的基于位置的动力学）
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, xpbd, pbd, compliance, lagrange-multiplier, real-time]
summary: Macklin 2016（NVIDIA，MIG）对 PBD 的物理化扩展：为约束引入柔度 α（刚度之逆）与总拉格朗日乘子 λ，从隐式时间离散推出与 PBD 几乎同形的更新式，使刚度与迭代数/时间步解耦、对应明确弹性势能并免费输出约束力；每约束仅多存一个标量，开销 <2%。
---

# XPBD（Extended PBD）

> 来源论文：[[RawSource/articles/Macklin-2016-XPBD|Macklin et al. 2016 索引卡]] · [[RawSource/articles/Macklin-2016-XPBD-中文翻译|中文译本]]
> 谱系：[[PositionBasedDynamics]]（被改造对象）→ **本页** → Small Steps 2019（子步化）；理论源头 Servin et al. 2006 柔顺约束

## 定义

**XPBD 是把"柔顺约束（compliant constraint）"引入 PBD 的扩展**：约束不再对应无限刚硬的几何投影，而是对应物理弹性势能 $U(x) = \frac{1}{2} C(x)^T \alpha^{-1} C(x)$（柔度 $\alpha$ = 刚度之逆）。由此刚度成为**真正的物理参数**——与迭代数和时间步无关——且求解过程中自然产出约束力估计。

## 它解决了 PBD 的什么病

PBD 把刚度 $k \in [0,1]$ 当投影缩放系数直接乘，后果（论文 §1）：

1. **迭代数依赖且非线性**：迭代越多越硬，无法按迭代数简单缩放刚度；
2. **传染性**：软硬体同场时，为一个对象提高迭代数会改变场景**所有**对象的行为，资产无法复用；
3. **无约束力**：触觉设备、力触发效果（可断关节）没有着力点。

## 核心推导（从隐式离散到 PBD 同形）

从 $M\ddot{x} = -\nabla U^T(x)$ 出发做隐式位置层离散，引入拉格朗日乘子并把时间步折进柔度（$\tilde\alpha = \alpha/\Delta t^2$——解耦时间步的关键一步），取两个近似（$K \approx M$ 省掉约束 Hessian；$g(x_i, \lambda_i) = 0$），Schur 补后得每约束的 Gauss-Seidel 更新：

$$\Delta\lambda_j = \frac{-C_j(x_i) - \tilde\alpha_j \lambda_i^j}{\nabla C_j M^{-1} \nabla C_j^T + \tilde\alpha_j}$$

要点：

- **$\alpha_j = 0$ 时精确退化为 PBD 的缩放因子 $s_j$**——原 PBD 原来一直在解"无限刚硬约束的增量乘子"，XPBD 只是给分母分子各加了一项正则化；
- $\lambda_i^j$ 是约束 $j$ 的**总**乘子，须跨迭代存储累加（每约束一个标量，实测开销 <2%）；
- 总乘子 $\lambda$ **就是约束力估计**——触觉、可断关节直接可用。

阻尼用 Rayleigh 耗散势 $\beta$ 并入同一更新式（对角情形 $\gamma_j = \tilde\alpha_j \tilde\beta_j / \Delta t$）。

## 验证结果（论文 §6）

| 实验 | 结论 |
|---|---|
| 谐振子（$\alpha = 0.001$ 距离约束） | PBD 1/5/10 迭代周期各异且全错；XPBD 任意步长/迭代数贴合解析解 |
| 20 粒子链 | 顶端约束力最大相对误差 6%（50 迭代）/ 2%（100）/ 0.5%（1000） |
| 悬臂梁（St Venant-Kirchhoff 三角 FEM，$E=10^5$, $\nu=0.3$） | 20 迭代与 Newton 参考解视觉无差 |
| 布料（64×64 粒子，24k 约束） | 迭代数 20→160，PBD 越来越硬越闷；XPBD 定性不变 |

FEM 接入路径：三角单元应变向量（Voigt 记法）当约束函数、柔度矩阵 = Lamé 参数刚度矩阵之逆——正确耦合应变分量（Poisson 效应），可直接用实验材料参数。

## 优点与缺点

**优点**：
- 刚度物理化：$\alpha$ 独立于迭代数/时间步，资产可复用（论文图 1 的核心卖点）
- 约束力免费：$\lambda$ 即力估计
- 实现代价极小：PBD 循环加 3 行（初始化 $\lambda$、算 $\Delta\lambda$、累加 $\lambda$）
- 保留 PBD 的简单与鲁棒（Gauss-Seidel/Jacobi 均可）

**缺点**（论文自认）：
- $\alpha = 0$ 时等同 $k=1$ PBD——要刚硬结果同样需要迭代数，**收敛不比 PBD 快**
- 迭代未收敛就停 → 人工柔度（与 PBD 同病）
- 只是隐式 Euler 的近似，要精度保证选传统方法（对比 Projective Dynamics 的精确形式）
- 全局法（PD/ADMM）预分解矩阵更高效——但撕裂/断裂拓扑变化时须重分解，XPBD 局部迭代对此免疫

## 在知识库中的体现

- [[PositionBasedDynamics]]：直接上游，$\alpha_j = 0$ 极限
- [[PyrrhaPhysics]]：网易自研方案明确以"XPBD + 杆模拟"为主打——本概念在本项目裙摆/服饰物理落地的直接体现
- [[RawSource/articles/SpringDecomposedSkinning-2025|SDS 2025]]：用经典 PBD（论文引用 [MHHR07]），可对照看 XPBD 能为其刚度调参带来什么
- 后续阅读方向：Small Steps 2019（子步化）、Unified Particle Physics 2014（粒子统一表示）

## 阅读指引

1. 先读 [[RawSource/articles/Macklin-2016-XPBD-中文翻译|中文译本]] §1（病根）与 §4（推导主线，重点吃透式 (18)）
2. §4.1 的"$\alpha_j = 0$ 退化为 $s_j$"段落是理解 PBD/XPBD 关系的钥匙
3. §6 结果四连（谐振子/链/悬臂梁/布料）建立量化直觉
4. 想深挖可顺藤 Servin 2006（柔顺约束源头）与 Goldenthal 2007（Fast Projection 变分视角）
