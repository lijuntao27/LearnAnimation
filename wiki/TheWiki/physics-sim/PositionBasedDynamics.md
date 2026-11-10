---
type: concept
title: Position Based Dynamics（PBD，基于位置的动力学）
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, constraints, cloth-simulation, real-time]
summary: Müller 2006 提出的实时仿真框架（VRIPHYS）：显式预测位置后把一般约束当几何投影按 Gauss-Seidel 逐个迭代修正，跳过力层直接操作位置、无条件稳定；统一处理布料拉伸/弯曲/自碰撞/撕裂，游戏实时布料的奠基求解器，前身是 Jakobsen 2001。
---

# Position Based Dynamics（PBD）

> 来源论文：[[RawSource/articles/Muller-2006-PositionBasedDynamics|Müller et al. 2006 索引卡]] · [[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|中文译本]]
> 谱系：前身 Jakobsen 2001（Verlet+松弛）→ **本页** → [[XPBD]]（2016 物理化）→ Small Steps（2019 子步化）

## 定义

**PBD 是一族绕过力与速度层、直接在位置上求解约束的实时物理仿真方法**。每个时间步：先按显式 Euler 预测所有顶点的新位置，再把每个约束（距离、二面角、体积、碰撞……）当作几何投影问题逐个迭代地把位置"拉回"满足约束的状态，最后用位置差反算速度。与力基方法（牛顿第二定律 $F = ma$ + 隐式积分）和冲量法（直接改速度）并列的第三条路线。

## 核心算法

```text
每个时间步 Δt：
  1. 预测     v ← v + Δt·f_ext/m ;  p ← x + Δt·v
  2. 生成碰撞约束（射线 x→p 与物体的交点 → 不等式约束）
  3. 迭代 n_s 次（Gauss-Seidel，逐个约束）：
       对每个约束 C_j(p_i1...p_in) = 0：
         沿梯度投影修正，见下方公式
  4. 提交     v ← (p − x)/Δt ;  x ← p
```

单约束投影公式（对单个约束做一步 Newton-Raphson）：

$$\Delta p_i = -s\, w_i \nabla_{p_i} C_j, \qquad s = \frac{C_j}{\sum_k w_k |\nabla_{p_k} C_j|^2}$$

其中 $w_i = 1/m_i$ 是逆质量——无限质量的点（$w_i = 0$）不移动，这是 PBD 处理附着/钉死的天然机制。

**动量守恒是免费的**：修正沿 $\nabla C_j$ 方向（垂直于刚体模式）且按逆质量加权，内部约束自动满足线/角动量守恒，不会引入"幽灵力"拖拽旋转物体。

**刚度的本质缺陷**：原始 PBD 把刚度 $k$ 直接当投影缩放系数，多轮迭代下有效刚度随迭代数 $n_s$ 非线性变化；论文用 $k' = 1-(1-k)^{1/n_s}$ 修正迭代依赖，但**时间步依赖仍在**——这是 [[XPBD]] 要根治的问题。

## 约束类型（布料应用实例）

| 约束 | 函数 | 用途 |
|---|---|---|
| 拉伸 | $C(p_1,p_2) = \|p_1 - p_2\| - l_0$ | 布料边不可伸长 |
| 弯曲 | $C = \arccos(n_1 \cdot n_2) - \phi_0$ | 相邻三角形二面角恢复（与拉伸独立，只依赖角度） |
| 自碰撞 | $C(q,p_1,p_2,p_3) = (q-p_1)\cdot n - h$ | 顶点保持在三角形上方布料厚度 $h$ 处 |
| 体积 | $C = \sum_i (p_{t_i^1} \times p_{t_i^2}) \cdot p_{t_i^3} - k_{pressure} V_0$ | 气球过压 |
| 碰撞 | $C(p) = (p - q_c) \cdot n_c$（不等式） | 与静态/动态物体的接触 |

撕裂 = 边拉伸超阈值时过顶点作垂直边方向的分裂平面分裂顶点。

## 优点与缺点

**优点**：
- 无条件稳定——积分步不外推，而是移动到求解器算出的物理有效构型；唯一不稳定源（Newton-Raphson）与时间步长无关
- 速度极快、实现极简（2006 年 Pentium 4 跑 4k 顶点撕裂场景 47 fps）
- 位置级直接可控：钉顶点、附着运动学对象、完全消除穿透，都是改位置即可
- 约束统一：上述五类约束一个求解循环全处理

**缺点**：
- 刚度非物理参数：依赖迭代数与时间步，资产不可复用（[[XPBD]] 解决）
- 无约束力概念：触觉、力触发效果做不了（XPBD 的 $\lambda$ 解决）
- Gauss-Seidel 顺序执行不利并行；过约束下对求解顺序敏感
- 不是真实的弹性系统：迭代有限时材料"发橡皮"

## 在知识库中的体现

- [[XPBD]]：正下游，物理正确化改造
- [[RawSource/articles/SpringDecomposedSkinning-2025|SDS 2025]]：弹簧骨骼蒙皮直接用 PBD 模拟自由质量——PBD 作二级动画求解器的当代应用
- [[PyrrhaPhysics]]：网易自研骨骼级物理即 XPBD+杆模拟，PBD 谱系的工业实现
- [[SkirtQNN-Design]]：本项目裙摆方案中"下游弹簧物理"的求解器即属此谱系

## 阅读指引

1. 先读 [[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|中文译本]] §3（算法+投影推导，全文核心）
2. §3.3 动量守恒 + §3.5 刚体模式阻尼（两个最优雅的设计）
3. 附录 A 弯曲约束梯度推导（实现布料弯曲必读）
4. 接 [[XPBD]] 看它如何被物理化
