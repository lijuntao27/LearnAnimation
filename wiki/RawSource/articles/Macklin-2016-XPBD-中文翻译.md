---
type: translation
title: XPBD: Position-Based Simulation of Compliant Constrained Dynamics（Macklin et al. 2016）中文译本
source: RawSource/articles/Macklin-2016-XPBD.pdf
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, xpbd, pbd, compliance, lagrange-multiplier, mig-2016, real-time, translation]
summary: Macklin、Müller、Chentanez（NVIDIA）MIG 2016 XPBD 论文中文学习译本：从隐式时间离散出发为 PBD 引入柔度 α 与总拉格朗日乘子 λ，刚度与迭代数/时间步解耦、对应明确弹性势能并输出约束力；对 PBD 循环仅加 3 行、每约束多存一个标量，开销 <2%。原始英文见 [[Macklin-2016-XPBD]]。
---

# XPBD: Position-Based Simulation of Compliant Constrained Dynamics — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），算法用代码块，正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[Macklin-2016-XPBD|原索引卡]] 所指的 `Macklin-2016-XPBD.txt`。
>
> 论文为 ACM正式出版物（MiG '16，© 2016 ACM，DOI 10.1145/2994258.2994272），本译本按学术学习使用处理。
>
> **关键术语对照**：compliance=柔度（刚度之逆）；compliant constraint=柔顺约束；Lagrange multiplier=拉格朗日乘子；iteration count=迭代数；ghost stiffness/人工柔度=artificial compliance；warm-start=热启动；Schur complement=舒尔补；Voigt notation=Voigt 记法；haptic=触觉。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | XPBD: Position-Based Simulation of Compliant Constrained Dynamics |
| 作者 | Miles Macklin, Matthias Müller, Nuttapong Chentanez |
| 机构 | NVIDIA |
| 发表 | Motion in Games (MiG '16), 2016-10-10–12, Burlingame, CA, USA |
| DOI | 10.1145/2994258.2994272 |

> **图 1**：此例展示改变体积保持与拉伸/剪切约束的相对刚度对可变形体的效果。与传统 PBD 不同，我们的方法让用户以**与时间步和迭代数无关**的方式控制可变形体的刚度，大幅简化资产创建。

## 摘要

我们解决基于位置的动力学（PBD）中长期存在的**约束刚度依赖迭代数与时间步**的问题。我们引入一个对 PBD 的简单扩展，使其能以隐式方式精确、高效地仿真**任意的弹性与耗散能量势**。此外，我们的方法提供**约束力估计**，使其适用于更广泛的应用，例如需要触觉用户反馈的场景。我们将算法与更昂贵的非线性求解器对比，发现它在保持 PBD 方法简单性与鲁棒性的同时产生视觉相似的结果。

**关键词**：物理仿真、受约束动力学、基于位置的动力学。
**概念**：计算方法 → 实时仿真；交互式仿真。

## 1. 引言

基于位置的动力学 [Müller et al. 2007] 是游戏与交互应用中实时可变形体仿真的流行方法，因其简单与鲁棒而特别有吸引力，近来在游戏之外的影视与医疗仿真应用中也流行起来。

随着其流行度上升，PBD 的局限也愈发棘手。一个众所周知的局限是：**PBD 的行为依赖仿真的时间步与迭代数** [Bender et al. 2014b]。具体而言，随迭代数增加或时间步减小，约束会变得任意刚硬。这种参数耦合在创建包含多种材质类型的场景时尤其麻烦，例如软体与近刚体交互的场景。此时，为一个对象提高迭代数以获得刚度，可能无意间改变场景中**所有其他对象**的行为——这常常迫使刚度系数被全局重新调参，让可复用的仿真资产极难创建。即使单一资产也受迭代数依赖之苦，例如设定布料模型中拉伸与弯曲约束的相对刚度。更糟的是，迭代数的影响是**非线性**的，难以直观调参，也难以把刚度值简单地按迭代数缩放了事。

近来虚拟现实的复兴催生了对更高保真、更具物理代表性的实时仿真的需求；与此同时，触觉反馈设备的广泛使用要求方法能提供**精确的力估计**。PBD 没有明确定义的约束力概念，因此大多局限于精度不如速度重要、仿真只作二级效果的应用。

本文提出**扩展的基于位置的动力学（XPBD）**算法。我们的方法通过引入对应明确定义弹性势能概念的新约束公式，解决迭代数与时间步依赖的刚度问题。我们从一个隐式时间离散出发推导方法，为 PBD 引入**总拉格朗日乘子（total Lagrange multiplier）**的概念。这提供了可用于驱动力相关效果与设备的约束力估计。

主要贡献总结如下：

- 扩展 PBD 约束，使其与明确定义的弹性与耗散能量势**直接对应**；
- 为 PBD 引入总拉格朗日乘子的概念，使约束求解**与时间步和迭代数无关**；
- 将算法与基于非线性 Newton 求解器的参考隐式时间步进方案做验证。

## 2. 相关工作

图形学界有大量求解受约束动力学仿真的工作 [Nealen et al. 2005]。我们的方法建立在 Müller et al. [2007] 的 PBD 算法之上——该算法以迭代 Gauss-Seidel 方式在**位置层**求解约束。Stam [2009] 提出了作用于速度的类似 Gauss-Seidel 约束求解器。这些方法局部地处理约束，因高效与易于实现而流行，但都受迭代数刚度依赖之苦。

许多作者把 PBD 用于可变形物体仿真。近期，Bender et al. [2014a] 提出把应变能直接作为约束函数的连续介质公式；同期 Müller et al. [2014] 提出约束 Green-St Venant 应变张量的分量，以按独立于模型离散化的方向控制应变。形状匹配 [Müller et al. 2005] 为适于 PBD 的可变形仿真引入几何约束——但所有这些方法都苦于刚度对迭代数的强依赖。相比之下，我们的柔顺约束公式与传统本构模型直接对应，并收敛到明确定义的解。

**Servin et al. [2006]** 提出了连续介质模型的柔顺约束公式与一个半隐式的速度层积分器，以与时间步无关的方式处理大范围刚度。我们的工作把他们的柔顺约束公式适配到 PBD，使我们在位置层求解约束且无需附加的稳定化项。

一系列工作使用**全局**方法求解约束。Goldenthal et al. [2007] 提出与 PBD 类似的不可拉伸约束投影方法，但约束系统被全局线性化、每次迭代用直接法求解——我们的方法可视为其 Fast Projection 算法的柔顺版本、配以迭代 Gauss-Seidel 求解器。Tournier et al. [2015] 用二阶导数信息扩展柔顺约束公式，以避免线性化导致的不稳定。与之不同，我们只用约束的一阶导数，并通过每时间步内反复的**局部**线性化避免全局线性化的稳定性问题。

Liu et al. [2013] 为质点-弹簧系统提出局部-全局求解器：局部解处理方向相关的非线性项，可预分解的全局解处理拉伸。其刚度大致与迭代数无关，且随迭代数增加解趋于 Newton 法。该思想被 Bouaziz et al. [2014] 推广为 Projective Dynamics。Wang [2015] 提出 Chebyshev 型方法，组合先前迭代的结果获得更好收敛。Narain et al. [2016] 把交替方向乘子法（ADMM）用于隐式时间积分，可处理非线性本构模型与硬约束，并证明 Projective Dynamics 是 ADMM 的特例。

这些全局方法通常依赖全局求解可高效完成——这常意味着对全局矩阵预分解。然而若约束拓扑在运行时变化（如撕裂或断裂），全局矩阵需要重新分解，计算代价高昂。它们的实现也远比 PBD 复杂。

## 3. 背景

我们简要描述 PBD 算法的核心，详情参阅 Bender et al. [2014b] 的综述。PBD 可视为一次 Strömer-Verlet 半隐式积分步，后接若干约束投影步。投影用各约束函数的局部线性化与质量加权的修正完成。PBD 约束求解器的主要步骤是计算每约束的位置修正量：

$$\Delta x = k_j s_j M^{-1} \nabla C_j(x_i). \tag{1}$$

下标 $i$ 表示迭代索引，$j$ 是约束索引，$k \in [0, 1]$ 是约束刚度——只是简单地乘在每个约束修正上。缩放因子 $s$ 由约束函数的单一 Newton 步推导而来：

$$s_j = \frac{-C_j(x_i)}{\nabla C_j M^{-1} \nabla C_j^T}. \tag{2}$$

简单地把位置变化乘以 $k$ 有一个不良副作用：**有效约束刚度现在同时依赖时间步与执行的约束投影次数**。Müller et al. [2007] 试图通过对刚度系数做指数缩放解决——但这没有考虑时间步，而且在多约束存在时不收敛到明确定义的解。

下一节我们发展一个使用**正则化约束**的方案——与明确定义的能量势直接对应——并展示如何以与时间步和迭代数无关的方式求解它们。

## 4. 我们的方法

我们从由能量势 $U(x)$ 导出的力的牛顿运动方程出发，推导扩展的基于位置的动力学算法（XPBD）：

$$M \ddot{x} = -\nabla U^T(x). \tag{3}$$

这里 $x = [x_1, x_2, \cdots, x_n]^T$ 是系统状态。在 PBD 中这通常就是粒子位置，但它可表示任意广义坐标模型（如刚体变换）。注意我们采用**梯度算子 $\nabla$ 是偏导数行向量**的约定。

对运动方程 (3) 做**位置层的隐式时间离散**，上标 $n$ 表示时间步索引：

$$M \left( \frac{x^{n+1} - 2x^n + x^{n-1}}{\Delta t^2} \right) = -\nabla U^T(x^{n+1}). \tag{4}$$

能量势 $U(x)$ 可进一步用约束函数向量 $C = [C_1(x), C_2(x), \cdots, C_m(x)]^T$ 表达为

$$U(x) = \frac{1}{2} C(x)^T \alpha^{-1} C(x), \tag{5}$$

其中 $\alpha$ 是对应**刚度之逆**（柔度）的分块对角柔度矩阵。弹性势的力由 $U$ 对 $x$ 的负梯度给出：

$$f_{elastic} = -\nabla_x U^T = -\nabla C^T \alpha^{-1} C. \tag{6}$$

遵循 Servin et al. [2006]，我们通过引入拉格朗日乘子把力分解为方向与标量分量，转换为柔顺约束公式：

$$\lambda_{elastic} = -\tilde\alpha^{-1} C(x). \tag{7}$$

这里 $\lambda_{elastic} = [\lambda_1, \lambda_2, \cdots, \lambda_m]^T$ 是约束乘子向量（此后省略下标）。注意我们把式 (4) 左侧的时间步**折进柔度矩阵**，定义了 $\tilde\alpha = \frac{\alpha}{\Delta t^2}$。

把 $\lambda$ 的表达式代回，得离散受约束运动方程：

$$M(x^{n+1} - \tilde{x}) - \nabla C(x^{n+1})^T \lambda^{n+1} = 0 \tag{8}$$

$$C(x^{n+1}) + \tilde\alpha \lambda^{n+1} = 0, \tag{9}$$

其中 $\tilde{x} = 2x^n - x^{n-1} = x^n + \Delta t\, v^n$ 称为**预测位置**（或惯性位置）。为求解这个非线性系统，我们设计一个基于 Newton 法的**不动点迭代**。以下省略时间步上标 $(n+1)$，用下标 $(i+1)$ 强调每次迭代的未知量。

把式 (8)(9) 分别记为 $g$ 与 $h$。目标是找到满足

$$g(x, \lambda) = 0 \tag{10}$$

$$h(x, \lambda) = 0 \tag{11}$$

的 $x$ 与 $\lambda$。对式 (10)(11) 线性化，得如下线性 Newton 子问题：

$$\begin{bmatrix} K & -\nabla C^T(x_i) \\ \nabla C(x_i) & \tilde\alpha \end{bmatrix} \begin{bmatrix} \Delta x \\ \Delta\lambda \end{bmatrix} = -\begin{bmatrix} g(x_i, \lambda_i) \\ h(x_i, \lambda_i) \end{bmatrix} \tag{12}$$

其中 $K = \frac{\partial g}{\partial x}$。该系统可解出 $\Delta x$ 与 $\Delta\lambda$，并相应更新位置与乘子：

$$\lambda_{i+1} = \lambda_i + \Delta\lambda \tag{13}$$

$$x_{i+1} = x_i + \Delta x. \tag{14}$$

这是一个对任意满足 $|x_{i+1} - x_i| \to 0$、$|\lambda_{i+1} - \lambda_i| \to 0$ 的序列都满足隐式运动方程 (8)(9) 的不动点迭代。该方法总体表现良好，但可能需要线性搜索策略才够鲁棒，且组装系统矩阵可能昂贵——特别地，我们想避免计算需要约束 Hessian 的 $K$。

现在引入**两个近似**，简化实现并与 PBD 重新接上联系。第一，用近似 $K \approx M$。这省略了几何刚度与约束 Hessian 项，引入 $O(\Delta t^2)$ 阶的局部误差。注意该近似可能改变收敛速率，但**不改变不动点迭代的全局误差或解**——它可以看作一个拟 Newton 方法。

第二，假设 $g(x_i, \lambda_i) = 0$。这一假设的合理性在于：初始化 $x_0 = \tilde{x}$、$\lambda_0 = 0$ 时它对第一次 Newton 迭代平凡成立；且若约束梯度变化缓慢，它会保持小值，并在梯度恒定时趋于零。此外，修改后的线性系统现在对应**从当前迭代 $x_i$ 出发到约束流形的质量加权最小化的最优性条件**——这一变分视角由 Goldenthal et al. [2007] 详细讨论，我们的方法可视为其 Fast Projection 算法的柔顺版本。

纳入这些近似后，更新的线性子问题为

$$\begin{bmatrix} M & -\nabla C^T(x_i) \\ \nabla C(x_i) & \tilde\alpha \end{bmatrix} \begin{bmatrix} \Delta x \\ \Delta\lambda \end{bmatrix} = -\begin{bmatrix} 0 \\ h(x_i, \lambda_i) \end{bmatrix}, \tag{15}$$

对 $M$ 取 **Schur 补**，得到关于未知量 $\Delta\lambda$ 的缩减系统：

$$\left[ \nabla C(x_i) M^{-1} \nabla C(x_i)^T + \tilde\alpha \right] \Delta\lambda = -C(x_i) - \tilde\alpha \lambda_i. \tag{16}$$

位置更新则直接由下式求值：

$$\Delta x = M^{-1} \nabla C(x_i)^T \Delta\lambda. \tag{17}$$

虽然我们方法返回的解不再能称为精确求解隐式运动方程，但实践中误差很小。第 6 节进一步考察精度。

### 4.1. Gauss-Seidel 更新

现在通过考虑线性方程组 (16) 的 Gauss-Seidel 解与 PBD 建立联系。若取索引 $j$ 的单个约束方程，可直接计算其拉格朗日乘子变化量：

$$\Delta\lambda_j = \frac{-C_j(x_i) - \tilde\alpha_j \lambda_i^j}{\nabla C_j M^{-1} \nabla C_j^T + \tilde\alpha_j}. \tag{18}$$

**这个公式是我们方法的核心。** 求解约束时，先对单个约束计算 $\Delta\lambda_j$，再用式 (13)(14) 更新系统位置与乘子。算法 1 总结了我们的方法——它与原 PBD 算法一致，只是增加了第 4、7、9 行（译注：乘子初始化、$\Delta\lambda$ 计算、$\lambda$ 累加）。

```text
Algorithm 1: XPBD simulation loop
1:  predict position    x̃ ⇐ x^n + Δt v^n + Δt² M^{−1} f_ext(x^n)
3:  initialize solve    x^0 ⇐ x̃
4:  initialize multipliers λ^0 ⇐ 0
5:  while i < solverIterations do
6:      for all constraints do
7:          compute Δλ using Eq (18)
8:          compute Δx using Eq (17)
9:          update λ^{i+1} ⇐ λ^i + Δλ
10:         update x^{i+1} ⇐ x^i + Δx
11:     end for
12:     i ⇐ i + 1
13: end while
15: update positions    x^{n+1} ⇐ x^i
16: update velocities   v^{n+1} ⇐ (x^{n+1} − x^n) / Δt
```

细察 $\Delta\lambda_j$ 的公式：$\alpha_j = 0$ 时它**精确对应**原 PBD 算法 (2) 中的缩放因子 $s_j$。从新定义看，$s_j$ 可理解为无限刚硬约束的**增量拉格朗日乘子变化量**。存在柔度（$\alpha_j \geq 0$）时，分子分母都多出一项——这些项起到**正则化**约束的作用：约束力受限，且对应式 (5) 的弹性势。

式 (18) 的分子包含 $\lambda_i^j$——约束 $j$ 在当前迭代 $i$ 的**总**拉格朗日乘子。除系统位置外，我们必须存储并更新这个变量。每约束多存一个标量是适度开销，而且提供了关于总约束力的有用信息，可用于驱动力相关效果（如可断关节）或触觉设备。

## 5. 阻尼

我们的方法从隐式时间步进方案推导而来，天然包含一些机械能耗散。不过，建模额外的约束阻尼仍有用处。为此我们定义 **Rayleigh 耗散势**：

$$D(x, v) = \frac{1}{2} \dot{C}(x)^T \beta \dot{C}(x) = \frac{1}{2} v^T \nabla C^T \beta \nabla C\, v, \tag{19, 20}$$

其中 $\beta$ 是对应约束阻尼系数的分块对角矩阵。注意 $\beta$ **不是**像柔度那样的逆参数，应按阻尼刚度设置。按拉格朗日动力学，耗散势的力由 $D$ 对速度的负梯度给出：

$$f_{damp} = -\nabla_v D^T = -\nabla C^T \beta \nabla C\, v. \tag{21}$$

与弹性情形类似，我们把力向量的标量分量分离为约束乘子：

$$\lambda_{damp} = -\tilde\beta \dot{C}(x) = -\tilde\beta \nabla C\, v, \tag{22}$$

其中通过定义 $\tilde\beta = \Delta t^2 \beta$ 把时间步折进阻尼参数。

可以单独求解 $\lambda_{damp}$，从而分别获得阻尼与弹性分量的约束力乘子。但多数情况下我们只关心总约束力，且由于阻尼力必须沿与弹性力相同的方向作用，可把两个乘子合并为单一方程：

$$\lambda = \lambda_{elastic} + \lambda_{damp} = -\tilde\alpha^{-1} C(x) - \tilde\beta \nabla C\, v. \tag{23}$$

把 (23) 重排为约束形式：

$$h(x, \lambda) = C(x) + \tilde\alpha \lambda + \tilde\alpha \tilde\beta \nabla C\, v = 0. \tag{24}$$

代入速度的离散近似 $v = \frac{1}{\Delta t}(x^{n+1} - x^n)$ 并对 $\lambda$ 线性化，得更新后的 Newton 步：

$$\left[ \left( I + \frac{\tilde\alpha \tilde\beta}{\Delta t} \right) \nabla C(x_i) M^{-1} \nabla C(x_i)^T + \tilde\alpha \right] \Delta\lambda = -h(x_i, \lambda_i). \tag{25}$$

写成单个约束方程的形式，Gauss-Seidel 更新为

$$\Delta\lambda_j = \frac{-C_j(x_i) - \tilde\alpha_j \lambda_i^j - \gamma_j \nabla C_j (x_i - x_n)}{(1 + \gamma_j) \nabla C_j M^{-1} \nabla C_j^T + \tilde\alpha_j}. \tag{26}$$

常见情形是 $\tilde\alpha$ 与 $\tilde\beta$ 为简单对角（而非分块对角），此时 $\gamma_j = \frac{\tilde\alpha_j \tilde\beta_j}{\Delta t}$，即约束的柔度与阻尼参数之积经时间步缩放。需要强调，这里所有附加项都容易且高效可算。

## 6. 结果

我们在多种可变形模型与约束类型上测试方法。2D 结果用 CPU 实现（Gauss-Seidel 式迭代），与直接作用于离散运动方程 (8)(9) 的传统非线性 Newton 求解器做验证对比；Newton 求解器的线性求解器用 Eigen 数学库 [2010] 的鲁棒 Cholesky 分解。3D 结果采集自基于 GPU 的 Jacobi 式迭代求解器（NVIDIA GTX1070）。碰撞处理与原 PBD 方法完全一致：接触采用零柔度假设，即接触约束无需存储拉格朗日乘子。

### 6.1. 弹簧

我们用建模为距离约束的简单谐振子测试：静止位置 $x = 1$、初始位置 $x_0 = 1.5$、$\alpha = 0.001$、质量 $m = 1$。如图 2 所示，PBD 无法再现正确的振荡周期——我们绘制了 1、5、10 次迭代的 PBD 仿真，说明其振荡周期与阻尼强烈依赖迭代数；而在这个简单例子中，XPBD 无论时间步与迭代数如何都紧密再现解析结果。注意此例中参考解的阻尼是我们对运动方程隐式离散的副作用。

> **图 2**：建模为距离约束（柔度 $\alpha = 0.001$）的简单谐振子。XPBD 紧贴解析解，而 PBD 的解严重依赖迭代数。

### 6.2. 链

> **图 4**：20 个粒子在重力下落的链的延时视图。左：参考 Newton 求解器；右：XPBD（50 次迭代）。

我们方法的一个关键收益是能返回约束力大小。那么一个重要问题是：返回的约束力有多精确？为量化 PBD 式更新引入的误差，我们仿真一条弱可拉伸的粒子链（图 4）：链由 20 个粒子组成，各 $m = 1.0$，以 $\alpha = 10^{-8}$ 的距离约束连接。仿真 100 帧，测量链顶端固定粒子处的约束力大小。

图 3a 比较参考 Newton 求解器与 XPBD 的结果。整个仿真过程中我们方法的最大相对误差为：50 次迭代 6%、100 次 2%、1000 次 0.5%。我们相信这对多数图形学应用是可接受的。

### 6.3. 悬臂梁

传统有限元方法（FEM）可在柔顺约束框架下重新公式化 [Servin et al. 2006]。对线弹性各向同性本构模型，把每个单元应变张量 $\epsilon$ 的分量当作约束函数向量（三角形单元的 Voigt 记法）：

$$C_{tri}(x) = \epsilon_{tri} = \begin{bmatrix} \epsilon_x \\ \epsilon_y \\ \epsilon_{xy} \end{bmatrix}. \tag{27}$$

柔度矩阵由刚度矩阵之逆给出，用 Lamé 参数表达为

$$\alpha_{tri} = K^{-1} = \begin{bmatrix} \lambda + 2\mu & \lambda & 0 \\ \lambda & \lambda + 2\mu & 0 \\ 0 & 0 & 2\mu \end{bmatrix}^{-1}. \tag{28}$$

这一公式相对 Strain-Based Dynamics [Müller et al. 2014] 的优势是：不仅对应传统材料参数，还**正确耦合各应变分量**以建模 Poisson 效应——这使 XPBD 能用实验数据获得的材料属性精确仿真。

我们用 St Venant-Kirchhoff 三角 FEM 离散的悬臂梁（图 5）评估算法精度：线弹性各向同性本构，Young's 模量 $E = 10^5$、Poisson 比 $\mu = 0.3$、$\Delta t = 0.008$。运行 50 帧，发现 **20 次 XPBD 迭代**就足以与参考 Newton 求解器视觉无差。残差随迭代的变化绘制于图 3b。

> **图 5**：三角 FEM 单元 + 线性各向同性本构建模的悬臂梁，仿真 50 帧。左：参考 Newton 求解器；右：XPBD——20 次迭代即与参考视觉无异。

### 6.4. 布料

我们用受重力的悬挂布料测试算法的迭代数无关性。布料模型为 64×64 粒子网格、24k 条距离约束。图 6 展示固定约束刚度、改变迭代数的效果：PBD 布料随迭代数增加逐渐变硬、阻尼变重；XPBD 的行为在不同迭代数下定性一致。

> **图 6**：悬挂布料，迭代数分别为 20、40、80、160（从左到右）。上排：PBD；下排：XPBD。PBD 刚度以非线性方式依赖迭代数，而 XPBD 行为定性不变。

为清晰展示 PBD 行为随迭代数的变化，我们使用了人为调低的约束刚度 $k = 0.01$。注意 **XPBD 并不比 PBD 收敛更快**——只是行为在不同迭代数下保持一致。事实上，柔度为零的极限下 XPBD 等价于 $k = 1$ 的 PBD。

我们测量 XPBD 对布料例子的性能影响：每迭代增加的成本通常**少于总仿真时间的 2%**。详见表 1。

| 迭代数 | 20 | 40 | 80 | 160 |
|---|---|---|---|---|
| PBD (ms) | 0.95 | 1.75 | 3.25 | 5.61 |
| XPBD (ms) | 0.97 | 1.78 | 3.34 | 5.65 |

**表 1**：悬挂布料例子在不同迭代数下的每步仿真时间（ms）。

### 6.5. 充气气球

我们把方法应用于充气气球：内部气压用全局体积约束建模，外表面为带拉伸、剪切、弯曲约束的布料网格（见图 1）。存储方面，内部体积约束只需**一个**辅助乘子，外加每个表面约束一个乘子。我们的方法让调整约束的相对刚度变得容易——例如必要时提高求解器迭代数获得更强的体积保持，而不影响表面刚度。

## 7. 局限与未来工作

$\alpha = 0$ 的极限下我们的方法与 $k = 1$ 的 PBD 完全相同，因此要获得刚硬结果需要同样多的迭代数。与 PBD 类似，收敛前就终止的低迭代数会产生**人工柔度**。收敛速度可以通过在每次迭代换用更精巧的线性求解器改进——聚焦加速迭代方法的近期工作（如 Wang et al. [2015] 的 Chebyshev 方法）可能适用。

我们的方法开启的一个有前途方向是**时间相干性**：既然现在有了总约束力的记录，就应该可以基于上一帧的拉格朗日乘子对约束求解做**热启动**（warm-start）。

与 Projective Dynamics [Bouaziz et al. 2014] 不同，我们的方法只是隐式 Euler 积分器的一个**近似**。对实时图形学应用我们相信误差可接受、多数情形不可分辨；但需要更强精度保证的应用可能更适合传统方法。

## 8. 结论

我们提出了对基于位置的动力学的一个简单扩展，解决其最著名的局限——刚度对时间步与迭代数的依赖。我们的方法只需每约束计算与存储**一个额外的标量**，却让 PBD 能仿真任意的弹性与耗散能量势，并提供可用于力相关效果的精确约束力估计。我们相信这拓宽了 PBD 在需要更高精度、与传统材料模型对应的应用中的适用范围。由于该方法只需对现有 PBD 求解器做平凡的修改，我们预期它会被工业界与从业者轻松采用（译注：此后业界实现如 NVIDIA Flex/PhysX、[[PyrrhaPhysics]] 等确实全面转向 XPBD）。

## 参考文献（选录）

- BENDER, KOSCHIER, CHARRIER, WEBER 2014. *Position-based simulation of continuous materials*. Computers & Graphics 44.
- BENDER, MÜLLER, OTADUY, TESCHNER, MACKLIN 2014. *A survey on position-based simulation methods in computer graphics*. CGF.
- BOUAZIZ, MARTIN, LIU, KAVAN, PAULY 2014. *Projective dynamics: fusing constraint projections for fast simulation*. TOG 33(4).
- GOLDENTHAL, HARMON, FATTAL, BERCOVIER, GRINSPUN 2007. *Efficient simulation of inextensible cloth*. TOG 26.
- LIU, BARGTEIL, O'BRIEN, KAVAN 2013. *Fast simulation of mass-spring systems*. TOG 32(6).
- MÜLLER, HEIDELBERGER, TESCHNER, GROSS 2005. *Meshless deformations based on shape matching*. SIGGRAPH '05.
- MÜLLER, HEIDELBERGER, HENNIX, RATCLIFF 2007. *Position based dynamics*. J. Vis. Comun. Image Represent. 18(2), 109–118.
- MÜLLER, CHENTANEZ, KIM, MACKLIN 2014. *Strain based dynamics*. SCA '14.
- NARAIN, OVERBY, BROWN 2016. *ADMM ⊇ projective dynamics: Fast simulation of general constitutive models*. SCA '16.
- SERVIN, LACOURSIERE, MELIN 2006. *Interactive simulation of elastic deformable materials*. Sigrad.
- STAM 2009. *Nucleus: Towards a unified dynamics solver for computer graphics*. CAD/Graphics '09.
- TOURNIER, NESME, GILLES, FAURE 2015. *Stable constrained dynamics*. TOG 34(4).
- WANG 2015. *A Chebyshev semi-iterative approach for accelerating projective and position-based dynamics*. TOG 34(6).
