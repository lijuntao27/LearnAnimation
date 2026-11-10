---
type: raw-article
title: Small Steps in Physics Simulation (Macklin et al., SCA 2019)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, xpbd, substepping, small-steps, sca-2019, real-time, gpu]
summary: Macklin 等 7 人（NVIDIA）SCA 2019 论文：大时间步 + n 次迭代不如 n 个小子步、每步 1 次 XPBD 迭代——外力位置误差 ∝ Δt²，子步化带来平方级误差削减；悬链 1:1e5 质量比下 100 子步误差比 100 迭代低两个数量级，FEM 悬臂梁与 PCG 相当且阻尼更小；碰撞检测每帧一次摊销成本，是现代实时引擎的标配范式。
---

# Small Steps in Physics Simulation (SCA 2019)

> 原文 PDF：`Macklin-2019-SmallSteps.pdf`（本目录，ACM 正式版）；提取文本：`Macklin-2019-SmallSteps.txt`。
> 谱系定位：[[XPBD]] 的直接后续——把"迭代数"也变成物理量 · 阅读顺序见 [[PBDReadingGuide]] 第 5 篇。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Miles Macklin, Kier Storey, Michelle Lu, Pierre Terdiman, Nuttapong Chentanez, Stefan Jeschke, Matthias Müller |
| 机构 | NVIDIA（Macklin 兼哥本哈根大学） |
| 发表 | SCA '19，Article 39，DOI 10.1145/3309486.3340247 |
| 实现 | CUDA，RTX 2080 Ti，并行 Jacobi 迭代 |

## 核心机制一句话

**把每个帧的时间步 $\Delta t_f$ 切成 $n$ 个子步 $\Delta t_s = \Delta t_f/n$，每个子步只做 1 次 XPBD 约束迭代**。依据是隐式离散方程里外力对位置的影响 $\propto \Delta t^2$：时间步减半、位置误差减四分之一——平方级削减，而单次迭代的计算量几乎不变。

## 关键设计

- **核心观察**：提高精度有两条路——小时间步 or 多迭代，计算量等价时哪个更有效？自 Baraff & Witkin 1998 以来共识是"大步+隐式"，本文实测**相反**：对 PGS 局部求解器，最优点在"预算内尽量多的小步 × 每步 1 次迭代"
- **单迭代的理论依据**：XPBD 的第一次迭代 = 作用在 backward Euler 方程上的 Newton 求解器第一步（Macklin 2016 已证）——所以单迭代方法计算上像显式积分，却带着隐式方法的全部稳定性（无限刚度/零柔度下依然稳定）
- **乘子简化**：每步迭代 1 次且 $\lambda$ 初始为 0，分子中 $\tilde\alpha_i \lambda_i$ 项消失，$\lambda$ 甚至可不存储（保留可输出力估计——单迭代下力估计依然准确，链顶 190N 真值贴合）
- **碰撞检测摊销**：每帧只做 1 次碰撞检测（用整帧 $\Delta t_f$ 预测轨迹、加 margin），子步间复用接触集——这是子步化可行的成本关键
- **接触去穿透限速**：子步化副作用是初始穿透 $d_0$ 被转成大分离速度 $d_0/\Delta t_s$（弹开伪影）；解法是约束加 $\max(d_0 - v_{max}\Delta t_s, 0)$ 项钳制分离速度
- **摩擦**：摩擦拉格朗日乘子钳制 $\Delta\lambda_f \leftarrow \min(\mu \Delta\lambda_n, \Delta\lambda_f)$ 满足 Coulomb 定律，隐式捕获粘滞/滑移切换

## 主要结果

| 场景 | 迭代 vs 子步 |
|---|---|
| 悬链（1:1e5 质量比） | 100 迭代误差 322.1m；100 子步 3.2m——**两个数量级** |
| 布料（k=1e7 N/m） | 1 子步 40 迭代有明显拉伸；40 子步 1 迭代显著更硬；半隐式 Euler 直接发散 |
| 150k 粒子 896k 弹簧布料 | 1 子步 30 迭代 12.4ms 有拉伸；30 子步 1 迭代 13.5ms 无拉伸 |
| 悬臂梁（12800 四面体 FEM，E=1e7） | 100 子步与 PCG 隐式 Euler 同刚度，阻尼更小；耗时 6ms vs 12ms |
| 936k 粒子流体（PBF） | 10 迭代体积明显损失；10 子步更硬、可用更大 CFL、运动更生动 |
| 刚体（1:1000 质量比关节链/胶囊堆叠） | 100 子步 6ms 稳定；500 迭代 17ms 仍拉伸 |

## 局限（论文自认）

- 速度相关项（阻尼力）误差只 ∝ $\Delta t$ 而非 $\Delta t^2$——高黏性材料建议位置约束后再对速度项做精确隐式后处理
- $\Delta t^2$ 太小时撞上单精度浮点下限（大坐标+小增量=无变化），子步数极多时可能需双精度
- Jacobi 版对顺序不敏感但收敛略慢；Gauss-Seidel 版残差依赖迭代顺序（可用 SSOR 缓解）

## 谱系与关联

- **上游**：[[Macklin-2016-XPBD|XPBD 2016]]（单迭代=Newton 第一步的理论出处）
- **下游影响**：当代实时布料/毛发/刚体引擎普遍转向子步范式；[[PyrrhaPhysics]] 类骨骼级物理亦受益
- **对照**：与显式积分的比较——同为"小步"计算模式，本文方法在无限刚度下稳定而半隐式 Euler 发散（Table 1：Small Steps 在四象限{稳定/效率/简单/能量}全占优）
