---
type: raw-article
title: Position Based Dynamics (Müller et al., VRIPHYS 2006)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, cloth-simulation, constraints, gauss-seidel, vriphys-2006, real-time]
summary: Müller、Heidelberger、Hennix、Ratcliff（AGEIA）VRIPHYS 2006 的 PBD 奠基论文：跳过力与速度层直接操作位置——显式 Euler 预测位置后，把一般约束 C_j=0 当几何投影按 Gauss-Seidel 逐个迭代修正，动量守恒由沿 ∇C 修正自动保证；无条件稳定，统一处理布料拉伸/弯曲/自碰撞/撕裂/气球过压，是游戏实时布料的标准求解器。中文译本见 [[Muller-2006-PositionBasedDynamics-中文翻译]]。
---

# Position Based Dynamics (VRIPHYS 2006)

> 原文 PDF：`Muller-2006-PositionBasedDynamics.pdf`（本目录，VRIPHYS 2006 workshop 版）；提取文本：`Muller-2006-PositionBasedDynamics.txt`。
> 中文译本：[[Muller-2006-PositionBasedDynamics-中文翻译]]。
> 谱系定位：[[PositionBasedDynamics]]（本论文的概念页）· 直系后代 [[Macklin-2016-XPBD|XPBD]]（Macklin 2016）· 前身 Jakobsen 2001《Advanced Character Physics》。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Matthias Müller, Bruno Heidelberger, Marcus Hennix, John Ratcliff |
| 机构 | AGEIA（物理加速卡厂商，后并入 NVIDIA） |
| 发表 | 3rd Workshop in Virtual Reality Interactions and Physical Simulation (VRIPHYS), 2006；期刊版 *J. Vis. Comun. Image Represent.* 18(2), 2007, 109–118 |
| 应用 | 游戏物理库中的实时布料模拟器 |

## 核心机制一句话

**把约束求解从力层搬到位置层**：每个时间步先用显式 Euler 预测所有顶点的新位置 $p_i = x_i + \Delta t\, v_i$，然后迭代地按 Gauss-Seidel 方式逐个把约束函数 $C_j$ 沿其梯度 $\nabla C_j$ 投影修正（本质是对单个约束做一步 Newton-Raphson），最后用位置差反算速度 $v_i = (p_i - x_i)/\Delta t$——速度层被完全绕开，位置即状态。

## 关键设计

- **一般约束框架**：约束 = 基数 $n_j$ + 函数 $C_j: \mathbb{R}^{3n_j} \to \mathbb{R}$ + 顶点索引集 + 刚度 $k_j$ + 等式/不等式类型；距离、弯曲（二面角）、体积、碰撞、附着全部统一表达
- **动量守恒**：修正量 $\Delta p_i \propto -w_i \nabla_{p_i} C$（$w_i = 1/m_i$ 逆质量加权）自动满足线/角动量守恒（内部约束），避免"幽灵力"拖拽旋转物体
- **Gauss-Seidel vs Jacobi**：逐个约束投影、修正立即可见——压力波一个求解器步内穿透整个材质，收敛远快于 Jacobi；代价是对求解顺序敏感（过约束时须固定顺序）
- **刚度修正 $k'$**：直接乘 $k$ 的效果随迭代数 $n_s$ 非线性（残余误差 $(1-k)^{n_s}$）；改乘 $k' = 1-(1-k)^{1/n_s}$ 使误差与 $n_s$ 无关——但仍依赖时间步（XPBD 后来根治）
- **碰撞即约束**：连续碰撞（射线 $x_i \to p_i$ 进物体 → 入射点不等式约束）+ 静态回退（最近表面点）；约束生成在求解循环外，快但可能漏碰（经验上伪影可忽略）；摩擦/恢复系数在速度更新步处理
- **刚体模式阻尼**：算出系统全局线速度/角速度，只阻尼各顶点相对全局运动的偏差——$k_{damping}=1$ 时退化为刚体，全局运动永不受影响
- **布料特化**：弯曲约束用相邻三角形二面角 $C_{bend} = \arccos(n_1 \cdot n_2) - \phi_0$，与拉伸独立（只依赖角度不依赖边长）；气球 = 全网格体积约束 + 过压系数；撕裂 = 边拉伸超阈值时沿垂直平面分裂顶点

## 主要结果

- Pentium 4 3 GHz 上：布条双way交互场景 >380 fps；1364 顶点/2562 三角形含自碰撞布料 ~30 fps；4264 顶点撕裂场景 ~47 fps
- 弯曲与拉伸刚度独立可调（同一布袋 $(k_{stretch}, k_{bend})$ 从 $(1,1)$ 到 $(1/100, 0)$ 的完整行为谱）
- 无条件稳定：积分步不盲目外推，而是移动到求解器算出的物理有效构型；唯一不稳定源是 Newton-Raphson 求解器自身，且与时间步长无关

## 局限（论文自认 + 后续揭示）

- 刚度参数仍依赖时间步（论文承认，实时固定步长下不构成问题）
- 约束力无明确定义（XPBD 引入拉格朗日乘子解决，可用于触觉/可断关节）
- 未处理刚体（Future Work；Macklin 2014 Unified Particle Physics 与 Fratarcangeli 2018 完成）
- 纯 Gauss-Seidel 顺序执行，不利并行（后续 Jacobi 变体与 GPU 求解器补足）

## 谱系与关联

- **前身**：Jakobsen 2001（Verlet + 距离约束，无一般框架）；Provot 1995 / Desbrun 1999（质点弹簧过拉伸后处理）；Müller 2005 Shape Matching（最接近的积分方法）
- **直系后代**：[[Macklin-2016-XPBD|XPBD]]（物理正确化）；Hierarchical PBD 2008（多分辨率）；Long Range Attachments 2012（不可拉伸布料）；Position Based Fluids 2013（密度约束）
- **知识库应用对照**：[[SpringDecomposedSkinning-2025]]（SDS 用 PBD 模拟弹簧骨骼自由质量，明确引用本文）、[[PyrrhaPhysics]]（UE 抖动骨骼插件，PBD 约束投影的实现谱系）
