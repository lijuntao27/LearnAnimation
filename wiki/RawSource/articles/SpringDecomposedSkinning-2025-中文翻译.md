---
type: translation
title: Spring Decomposed Skinning - Real-Time Secondary Animation with Spring Decomposed Skinning (2025) 中文译本
source: RawSource/articles/SpringDecomposedSkinning-2025.pdf
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, spring-bones, pbd, skinning, helper-bones, secondary-animation, sgp-2025, translation]
summary: Akyürek 与 Sahillioğlu SGP 2025 原论文中文学习译本：SDS 在骨骼层面接入物理——每根弹簧骨骼=固定质量+自由质量两点 Hooke 弹簧，PBD 模拟自由质量，运动学约束恢复骨骼连通与静止长度，RST 从模拟骨骼反推蒙皮变换；无四面体化、复杂度只随骨骼数，比 Wu&Umetani 四面体方案快约 10 倍。原始英文见 [[SpringDecomposedSkinning-2025]]。
---

# Real-Time Secondary Animation with Spring Decomposed Skinning — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），算法用代码块，正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[SpringDecomposedSkinning-2025|原索引卡]] 所指的 `SpringDecomposedSkinning-2025.txt`。
>
> 论文为作者主页公开的预印本（SGP 2025 / Computer Graphics Forum Vol.44 No.5），版权归 Eurographics / John Wiley & Sons 所有，本译本按学术学习使用处理。
>
> **关键术语对照**：spring bone=弹簧骨骼；fixed/free mass=固定质量/自由质量；helper bone=辅助骨骼；kinematic constraints=运动学约束；RST=Rotate-Scale-Translate（旋转-缩放-平移）仿射分解；jiggle bones=抖动骨骼（业界惯称）；secondary motion/dynamics=二级运动/二级动态；fixed_scale=保持骨骼原长的用户选项。
>
> 与本谱系的关系：本文是"蒙皮+弹簧"分工的学术正名之作（业界 jiggle bones 的形式化），引用了 [[Jin2024-RopeChainQNN|Jin 2024]] 的前身零静止长度弹簧（Jin et al. SIGGRAPH 2022）；与 SkirtQNN"QNN 准静态锚点 + 下游弹簧物理"分工的对照见 [[SkirtQNN-Design]]，谱系定位见 [[BoneDrivenCloth-CitationMap]]。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Real-Time Secondary Animation with Spring Decomposed Skinning |
| 作者 | Bartu Akyürek, Yusuf Sahillioğlu |
| 机构 | 中东技术大学（METU）计算机工程系，土耳其 |
| 发表 | Eurographics Symposium on Geometry Processing (SGP) 2025；Computer Graphics Forum, Vol. 44 (2025), No. 5 |
| 代码 | https://github.com/bartuakyurek/Spring-Decomposed-Skinning |
| 主页 | https://bartuakyurek.github.io/publications/SDS/ |

> **图 1**：我们的方法覆盖广泛的动态，包括复合刚体的二级运动（如植物叶片）、弹性体的拉伸压缩（牛、橡皮鸭）、软体抖动（脂肪组织）、全局二级运动（振动的大象）以及布料动画。

## 摘要

我们提出一个把二级运动集成到现有动画管线中的框架。蒙皮（skinning）为实时动画提供快速计算与直观的变形控制；尽管有这些好处，传统蒙皮方法缺少二级动态，例如脂肪组织的抖动。我们通过**用弹簧力物理模拟变形句柄（deformation handles）**来解决蒙皮的刚性问题。多数研究通过在**体网格顶点**上做 FEM 模拟把二级运动引入蒙皮，使其计算复杂度与网格分辨率耦合。与这些方法不同，我们**不需要任何体网格输入**；我们的方法通过**直接模拟变形句柄**扩展到更高的网格分辨率。被模拟的句柄——即**弹簧骨骼（spring bones）**——为刚体、弹性体、软组织与布料模拟等各类对象丰富了刚体蒙皮变形，产出发散的二级动画。本质上，我们在变形句柄的范围内利用物理模拟的好处，在广泛对象上实现**可控制的实时动态**，同时与现有蒙皮管线保持兼容。我们的方法避免四面体重网格化，与基于 FEM 的体网格模拟相比显著更快。

**CCS 概念**：计算方法 → 动画；物理模拟。

## 1. 引言

自然界的刚体常表现出振荡运动，称为**二级动态（secondary dynamics）**，作为对初始**一级运动（primary motion）**的响应。软体在受力下抖动时也有类似运动。尽管蒙皮方法在动画中广泛使用，它仍然太僵硬，无法在软体与刚体上表现这类动态。为模拟动态变形，研究者一直在发展物理模拟技术：蒙皮方法缺乏真实感的问题已通过**在网格顶点上做物理模拟**来补偿；但把物理并入网格变形可能涉及复杂的实现。此外，许多物理变形方案**依赖网格分辨率**；FEM 模拟通常还需要四面体网格 [HMT∗12, IKNDP16, ARF15, ZZCB21, CGC∗02, HTC∗13, RF14, MZS∗11, KB18, KBB∗17]，因此这些方法在更精细分辨率下计算成本显著增加。另一个缺点是物理模拟结果的**可控性有限**，因此物理模拟变形方法相比蒙皮大多不够直观。

**在保留蒙皮一级直觉的前提下引入可信的动态效果**，仍然是一个开放的研究领域。Spring Decomposed Skinning（SDS）通过**把物理限制在蒙皮骨骼的范围内**来 tackling 这个问题。以此方式，SDS 把蒙皮"计算快、用户控制直观"的优势与物理模拟"动态可信"的优势结合起来。

我们用**弹簧力**模拟骨骼的动态运动，称为**弹簧骨骼**。由于正弦行为（用于抖动型二级运动）与计算效率，这些力用 **Hooke 弹簧**建模。然而尽管简单，Hooke 弹簧在时间步进方案下、尤其在大时间步时可能遭遇数值不稳定 [WB01]；且它们只是条件稳定，需要谨慎选择初始参数 [MHTG05]。为解决稳定性问题，我们采用**位置基动力学（PBD）框架** [MHHR07]，并进一步施加**运动学约束**以确保直观的骨骼运动。

我们的方法输入一个有关节的角色动画（静止姿态网格、其 rig、rig 的关键帧），允许用户指定哪些骨骼应作为弹簧骨骼模拟。可选地，我们利用**辅助骨骼（helper bones）**扩展主骨架，让用户把动态**局部化到更小的表面**。通过这一管线，我们在现有动画上引入动态，给传统蒙皮管线只增加很小开销。我们的方法简单，却能捕获发散的动态运动。蒙皮的核心是**把复杂表面变形分解为有限骨骼变换集合**；循此直觉，我们把动态网格表面变形分解为**弹簧运动的线性组合**。源代码公开于 https://github.com/bartuakyurek/Spring-Decomposed-Skinning 。

### 1.1. 贡献

弹簧骨骼是**动态的**，因为其运动依赖当前时间与构型；而现有几何蒙皮方法是**静态的**——要产出同样的运动需要额外的手动关键帧。即便加了关键帧，我们算出的动态也很难用手工复现。本质上，我们的方法通过 rig 提供了**可控制的物理模拟方案**。

与常见方法不同，我们的框架**不需要体网格输入或解剖学基元**，因此与现有动画管线高度兼容。体网格模拟通常依赖网格分辨率；相比之下，我们把物理模拟限制在 rig 句柄上，从而扩展到更高网格分辨率，快速计算动态运动。

主要贡献如下：

- 通过变形句柄为物理模拟提供**美术控制**。这些句柄在刚体、软组织、布面等多种对象上同时实现**全局与局部**二级运动。
- 把动态表面变形分解为**简单弹簧振荡的线性组合**，与蒙皮背后的核心直觉对齐。
- 在广泛 3D 模型上实现**实时**动态变形，无需计算复杂的四面体化。
- 一个计算动态摆位骨骼的**类逆运动学（IK-like）变换**的方法（译注：RST 分解）。
- **易于集成**到现有动画管线的框架，与传统蒙皮输入输出兼容。
- 围绕 "jiggle bones" 构造做**形式化**，弥合工业应用与学术研究的差距。

## 2. 相关工作

**蒙皮分解（Skinning Decomposition）** 一词由 [KSO10] 引入，指从示例姿态求骨骼变换与骨骼-顶点绑定权重。[LD12]（译注：SSDR，Le & Deng 2012）把一组示例姿态分解为刚体变换。尽管本文不聚焦于基于示例的分解，我们同样关心把动态姿态分解为一组骨骼变换。

几何蒙皮方法（如 LBS）普遍受角色运动刚性问题之苦，普遍缺少抖动、摇摆、肌肉隆起或弹性变形等动态效果 [RF16, WLP∗17, Muk15]。研究者持续发展解决方案，为蒙皮框架注入生命、提升动画表现力。

**Blendshape 变形**（[KBB∗17, LMR∗23, MWF∗12] 等诸多研究）在蒙皮表现力不足时有用，如表面高度复杂的面部表情。除蒙皮管线外，我们的框架也可与 blendshape 变形集成。

**解剖学模型**（[KBB∗17, LST09, RRC∗18, AS07]）利用骨骼与肌肉的底层解剖结构实现高度真实的变形；但这些方法实现复杂，且对给定网格可能没有现成的解剖学知识。

**基于示例的方法**在 [SZT∗08, MG03, Muk15] 等众多研究中被用来引入真实动态。大量论文 [ZZCB21, MMRH22, YWM∗21, SGOC20, CO18, PMRMB15, JHG∗22] 进一步用神经网络扩展基于示例的方法。若干研究 [SGX∗21, SGXT20, BRPMB17] 利用动作捕捉技术。尽管动态结果可信，动捕仍是昂贵的方案；此外基于示例方法的范围受限于数据集，动态的泛化是另一个挑战。相比之下，我们的框架无需数据集成本即可应用于多种对象。

**基于 FEM 的方法**（[CBC∗05, CGC∗02, HMT∗12] 等）用物理模拟计算外力（风、重力、碰撞）下发生的动态运动。很多论文 [HMT∗12, IKNDP16, ARF15, ZZCB21, CGC∗02, HTC∗13, RF14, MZS∗11, KB18, KBB∗17, TRPO21] 通过体四面体网格把 FEM 模拟并入蒙皮管线。尽管取得了动态效果，四面体化计算昂贵且对某些几何并非易事；时间复杂度也是瓶颈（尤其高网格分辨率），使高分辨率四面体 FEM 不适合实时应用。本工作中我们避免复杂四面体化，兼顾可扩展与可控的动态。

**互补动态（Complementary Dynamics）** [ZBLJ20, BZC∗23] 引入不会撤销几何方法所产变形的二级运动：在刚体运动与计算出的二级运动之间注入**正交约束**。类似地，若干论文也在动态变形中使用正交约束，如 [CBC∗05] 与 [WU23]。[TRPO21] 把软组织动态表示为对参数化骨架运动的偏离。互补动态的一个局限是没有完全覆盖二级运动的范围，尤其当物体在**与一级运动相同的方向**上出现额外抖动时——摇动花盆（图 9）会在互补空间和一级空间都产生抖动。就此而言，我们的二级动画既可产出互补运动，也可**夸张一级运动**；分别称之为局部与全局动态。

**质点-弹簧系统**用于绳、布料与服装动画 [LBOK13]、毛发 [SLF08]、软组织模拟 [Gol18] 与网格展开 [SK16]。若干工作 [NT98, LST09, KHS01, TT93, MHYH17] 在肌肉解剖结构上、或肌肉与皮肤组织之间用质点-弹簧网络做真实变形。[KB18] 在四面体边上用弹簧约束做基于物理的角色蒙皮；[LTW95] 用分层质点-弹簧网络模拟软组织以获得真实面部表情。**[JHG∗22]（译注：Jin et al. SIGGRAPH 2022，Jin 2024 rope-chain 工作的前身）用一个准静态神经网络增强解析积分的零静止长度弹簧，以捕获动态变形，特别是人体软组织动态。** 其它方法有 [vFTS07]（定义有限组质点-弹簧集合实现弹性变形）、[WLP∗17]（用质点-弹簧网络变形 2D 插画）。虽非质点-弹簧系统，[DJBDT10] 提出带质量与刚度参数的物理模拟动态曲线用于抖动的 2D 动画。[TN19] 中 Toothman 与 Neff 通过在主 rig 与网格顶点之间加辅助 rig 引入**弹簧 rig** 的概念——他们用弹簧力在后处理中**静态地**修正蒙皮伪影；相比之下我们把弹簧 rig 用于**骨骼子空间中的动态运动**。

**辅助骨骼**被 [CM24, Muk15, MK16, Muk18, MG03] 用来模拟直观 rig 无法实现的非线性变形，如肌肉隆起与组织抖动。受这些研究启发，我们用辅助骨骼拓宽动态范围。作为对"逐根看待辅助骨骼"研究的扩展，我们允许**辅助骨骼链**实现局部软组织抖动与布料动态；我们也展示辅助骨骼不仅对组织变形有用，对更广泛的动态也有用。

**Rig 级物理方法**提供由 rig 参数驱动的变形框架。处理 rig 级物理力的先驱工作之一是 Rohmer 等的 **Velocity Skinning** [RTK∗21]：骨骼按其速度关联一组额外权重用于底层蒙皮，这些额外权重允许动态的松软与挤压变形。[GBFP11] 也把物理变形精度与蒙皮句柄结合。基于 Velocity Skinning，[SRKZ24] 向蒙皮空间注入时变信号以捕获振荡二级运动——该信号建模为阻尼正弦，有效表示单个一维弹簧。我们的方法**直接把蒙皮骨骼模拟为弹簧**，使弹簧链能产出更复杂的二级运动。另一项统一物理与蒙皮变形的工作是 Wu 与 Umetani 的 [WU23]（2D 三角化表面或 3D 四面体网格）。我们的方法同样由 rig 参数驱动、统一物理模拟与蒙皮管线；SDS 的区别在于**直接模拟骨骼**，仅通过骨架就为物理变形提供美术控制。

基于物理的变形对**表面重建**也有用 [YS09, SY10, BL21]。若干体网格模拟工作也在物理变形管线中参数化骨骼运动：[CGC∗02] 用粗体网格引入由骨骼动画驱动的弹性变形；[HTC∗13] 把 rig 参数用作体网格物理模拟的自由度；类似地在 **Rig-Space Physics** [HMT∗12] 中，rig 参数决定底层 FEM 弹性变形的刚度参数，用骨骼动画增强物理模拟。缺点是这些框架可能需要繁琐的额外权重绘制与纹理映射，把已动画模型转成体输入。我们的方法避免此类重网格化步骤，可直接给现有动画角色加动态。

**Jiggle bones** 在业界已有一定知名度，最近的使用之一是游戏《战神：诸神黄昏》——开发者用辅助骨骼借助解剖学启发式产出组织抖动与肌肉隆起 [Wan23]。然而据我们所知，尽管存在 [MK16] 等类似工作，jiggle bones **从未在蒙皮文献中被直接正式介绍**。本工作中我们提出弹簧骨骼蒙皮的基础框架，同时利用物理模拟的真实感与传统蒙皮的速度——这是工业应用与学术研究之间的一个空白。方法虽简单但有效且流行，对游戏开发有正面影响。

## 3. 方法

给定一个已 rig 的模型及其动画关键帧，SDS 定义一个模拟 rig 骨骼的简单框架。初始时，用户选择特定 rig 骨骼作为**弹簧骨骼**；这些骨骼包括**主骨骼**与**辅助骨骼**。初始配置后，方法在动画序列中自动计算动态骨骼位置。主骨骼是原始 rig 中驱动主要运动的骨骼（如人体骨架的四肢）；辅助骨骼是用户引入的附加骨骼，用于把二级运动**局部化**到更小区域（典型用于软组织动态）。经验法则：想要**全局**二级运动（rig 主运动导致的振荡）时把主骨骼转为弹簧骨骼；需要**局部**二级运动（如肚皮脂肪抖动）时添加辅助弹簧骨骼。

如图 2 所示，我们把弹簧骨骼定义为**附在骨骼两端两个质点上的弹簧**。第一个质点即**固定质量（fixed mass）**位于骨骼头部，其位置在整个质点-弹簧模拟中保持不变，仅由运动学约束与用户关键帧更新。第二个质点即**自由质量（free mass）**位于骨骼尾部，模拟在每帧动态更新其位置。

用户可通过沿骨骼移动固定质量（从头到尾）调整**弹簧静止长度**。极端情况下，固定质量与自由质量都位于骨骼尾部，形成**点弹簧骨骼（point spring bone）**。把主骨骼转为点弹簧骨骼可实现挤压拉伸（squash & stretch）变形——二级运动增强一级运动的表现力（图 15）。

我们引入固定/自由质量的概念，以**符合骨骼动画直觉的方式**模拟骨骼动态。与经典质点-弹簧网络（所有粒子都在模拟中更新）不同，我们通过固定每根弹簧的第一个质量来**维持骨架的层级结构**：固定质量保持相对其父级的朝向，不在质点-弹簧模拟中更新；其位置由刚体蒙皮变换与运动学约束驱动（见后文）。这一设计保留了骨骼树的父子关系（除根外每根骨骼相对其父级定向）。

每帧我们模拟自由质量并相应更新 rig 骨骼朝向。设 $\mathcal{B}^0: \{B^0_i\}$ 为静止姿态的骨骼朝向集合 $B^0_i \in \mathbb{R}^{2\times3}$（含头尾位置）；类似地 $\mathcal{B}^R, \mathcal{B}^D$ 分别为刚体与动态骨骼朝向集合。前向运动学计算的刚体骨骼变换 $M_R: \mathcal{B}^0 \to \mathcal{B}^R$ 把静止姿态映射到刚体姿态。基于初始设置与每帧的 rig 姿态，框架动态更新 rig 姿态并计算动态骨骼变换 $M_D: \mathcal{B}^R \to \mathcal{B}^D$，供底层蒙皮方法使用。

> **图 2**：沿骨骼放置的质点-弹簧系统。固定质量由刚体动画关键帧驱动，自由质量由动态模拟在每个时间步更新。弹簧静止长度由固定质量位置决定；沿骨骼轴平移固定质量会缩短静止长度。极端配置下固定质量与自由质量都位于骨骼尾端，由零静止长度弹簧连接，形成点弹簧骨骼。

### 3.1. 模拟

我们用 PBD 框架 [MHHR07] 把弹簧骨骼模拟为简单 Hooke 弹簧。初始时用户确定质量-弹簧模拟参数：刚度 $k_s \in \mathbb{R}$、阻尼 $k_d \in \mathbb{R}$、质量 $m \in \mathbb{R}$。每个粒子 $p_i$ 初始位置 $x_i$ 按静止姿态设置，初始速度 $v^0_i = 0$。PBD 算法中同时存储逆质量 $w_i = 1/m_i$。

每帧所有质量先按传统蒙皮管线被前向运动学位移；然后质点-弹簧模拟器基于 Hooke 弹簧力与质量速度更新自由质量位置（图 5 步骤 3-4）。

设 $p_1, p_2 \in \mathbb{R}^3$ 为弹簧两端的粒子位置，$v_1, v_2 \in \mathbb{R}^3$ 为速度，$l_0$ 为静止长度。弹簧向量 $p_{1,2} = p_2 - p_1$，单位弹簧向量 $n = p_{1,2} / \|p_{1,2}\|$。作用在 $p_1$ 上的 Hooke 弹簧力与阻尼力为：

$$f_s = k_s \big( \tfrac{p_{1,2}}{\|p_{1,2}\|} - l_0 \big) n \tag{1}$$

（译注：原文如此；按标准 Hooke 定律应为 $k_s(\|p_{1,2}\| - l_0)n$，PDF 抽取的公式排版疑似有误，语义为"长度偏离静止长度产生沿 $n$ 的恢复力"。）

$$f_d = -k_d \,[ \cdot (v_2 + v_1)]\, n \tag{2}$$

（译注：方括号处按 Hooke 阻尼标准形式应为 $n \cdot (v_2 + v_1)$。）

对另一粒子 $p_2$，弹簧力取 $-f_s$。总外力 $f = f_s + f_d$ 用于 PBD 框架。注意公式 (1)(2) 不表示弹簧的**弯曲**——即弹簧骨骼只做线性轨迹的仿射运动；但使用**弹簧链**时弯曲现象会出现：父子关系允许链弯曲，子骨骼既继承祖先的旋转、也拥有自由质量经模拟推断后的自身旋转。

**算法 1（基于 PBD 的模拟）** 输入 rig 中所有质量位置 $\mathcal{X}: \{x_i \in \mathbb{R}^3\}$，单次迭代、时间步 $\Delta t$ 更新自由质量位置：

```
算法 1  PBD-based Simulation
Require: X : {xi ∈ R³}
1: for all particles i do
2:     if mi is 0: continue            ▷ 固定质量跳过
3:     vi ← vi + Δt·wi·fi
4:     vi ← ds·vi                      ▷ 速度阻尼，美术可调参数 ds
5: end for
6: for all particles i do pi ← xi + Δt·vi
7: if stretch_constraints
8:     p̃i ← projectStretchConstraints(p1,...,pN)
9: endif
10: for all particles i do
11:    vi ← (p̃i − xi)
12:    xi ← p̃i
13: end for
```

行 2：粒子固定（质量为零）时跳过模拟。行 4：平行于 PBD 算法，提供参数 $d_s$ 进一步阻尼速度 $v \leftarrow d_s v$——这是 3D 美术调节效果的参数；实验中 $0 < d_s < 1$ 得到可信动态。

外力可以包含重力与其它接触力，但实验中我们**通常省略重力加速度与接触力**，只关注弹簧力的效果。如图 3，较低刚度 $k_s$ 产生较慢运动，较高刚度让弹簧骨骼突然振动并更快回到静止长度（系统保持稳定）。

> **图 3**：弹簧刚度对收敛行为的影响。增大 $k_s$ 导致向静止长度更快收敛。（$k_s=10$ vs $k_s=300$）

由于只限弹簧力、避免其它外力，我们在 PBD 算法中**跳过碰撞约束**。替代地，行 7-9 可选地提供**拉伸约束**（见 [BMOT13]），在速度与粒子更新后把弹簧骨骼投回原长。行 8 的约束投影为 $\tilde{p}_i \leftarrow p_i + \Delta p_i$，$\Delta p_i$ 为校正向量。对单根弹簧，拉伸校正向量 $\Delta p_1, \Delta p_2$ 按 [BMOT13] 描述。实验中通常单次迭代做可选的拉伸约束投影。

对每个与 $p_j$ 以弹簧相连的粒子 $p_i$，校正向量为：

$$\Delta p_i = \frac{w_i}{w_i + w_j} \Big(\|p_{i,j}\| - l_0\Big) \frac{p_{i,j}}{\|p_{i,j}\|} \tag{3}$$

尽管 PBD 无条件稳定，投影拉伸约束常常产出**不可信的抖动骨骼**。为改善动态，我们在 PBD 模拟之后进一步投影约束（见 3.2 节）。

> **图 4**：约束对龟背竹 rig 运动的影响。蓝色为弹簧骨骼。一根刚性白骨左右旋转使骨骼抖动。(a) 拉伸约束产生圆弧运动。(b) 在 PBD 框架内投影我们的运动学约束导致振动、无法产生平滑抖动。(c) 在 PBD 框架外投影运动学约束产生更可信的抖动。

### 3.2. 运动学约束

算法 1 更新自由质量位置后，**骨骼尾端与其子骨骼头端的连接被打断**（因为固定质量不参与模拟，图 5 步骤 4）。因此为保证骨骼连通性，我们基于 rig 的运动学树投影附加约束。

当运动学约束在经典 PBD 框架**内**投影时，会导致突然振动（图 4b）；相反，在**模拟循环之后**施加这些约束产生更可信的抖动效果（图 4c）。此外只用拉伸约束会产生多余的圆弧运动（图 4a）。因此我们提供用户选项 **fixed_scale** 在模拟后保持骨骼静止长度。在 PBD 模拟之后投影这些约束，避免不直观的振动、得到更平滑的 rig 动态。

图 5 展示恢复骨骼连通性与静止长度的步骤：步骤 1-4 用前向运动学摆 rig 并模拟质点-弹簧系统；步骤 4→5 若启用 fixed_scale，把弹簧骨骼缩回静止长度（算法 2 行 2-6），然后把后代骨骼平移回其静止朝向（算法 2 行 7-11）——即若原本全连通，子骨骼被移到父骨骼尾部；若原本不连通，则带偏移移动（行 9）。

```
算法 2  Kinematic Constraints
Require: B⁰, KB, B
1: for all bones i do
2:     if fixed_scale then
3:         l = ∥B¹i − B²i∥
4:         d = (B¹i − B²i) / l
5:         B²i ← (l − l₀)·d            ▷ 骨骩缩回静止长度
6:     end if
7:     for all children j ∈ KB(i) do
8:         Bⱼ = B²ⱼ − B¹ⱼ              ▷ 子骨髂向量
9:         B¹ⱼ ← B²i + tⱼ             ▷ 子骨髂头对齐父骨髂尾（或带偏移）
10:        B²ⱼ ← B¹ⱼ + Bⱼ
11:    end for
12: end for
```

算法 2 中 $\mathcal{B}^0$ 为静止骨骼位置，$B_i = (B^1_i, B^2_i)$ 为第 $i$ 根骨骼（头 $B^1_i$、尾 $B^2_i \in \mathbb{R}^3$）；$\mathcal{K}_B: \{(B_i, B_j)\}$ 为运动学关系（每个元组中 $B_i$ 是 $B_j$ 的父骨骼）；$\mathcal{B}$ 为摆位空间中的骨骼（本例中为模拟后的骨骼朝向）。

> **图 5**：玩具案例中保持静止长度与连通性的步骤。第一个框为静止姿态；虚线框中用户旋转经前向运动学旋转骨骼链；最后一框为动态摆位后的骨骼链（保持原长与连通）。

### 3.3. 蒙皮变换

蒙皮中，底层骨骼变换决定网格变形。模拟弹簧骨骼后，需要推断交给蒙皮的骨骼变换。本文采用最广泛使用的**线性混合蒙皮（LBS）**。设 $v^0_j \in \mathbb{R}^{4\times1}$ 为齐次坐标下的静止姿态顶点，$M_i \in \mathbb{R}^{4\times4}$ 为骨骼 $i$ 的变换矩阵，$w_{ij}$ 为骨骼 $i$ 与顶点 $j$ 的绑定权重。变形后顶点位置：

$$v_j = \sum_{i=0}^{N} w_{ij} M_i v^0_j \tag{4}$$

传统蒙皮管线中骨骼变换 $M_i$ 在用户摆好骨架后由前向/逆向运动学直接计算。本例中用户输入的变换在弹簧骨骼被模拟后**被改变**。因此问题变为求新变换 $M^*_i: \mathcal{B}^0_i \to \mathcal{B}^D_i$，把静止姿态骨骼摆到动态位置。

为计算动态骨骼变换，我们把矩阵 $M^*_i$ 分解为单独的仿射变换：**旋转-缩放-平移（RST）**。图 7 展示求仿射变换的步骤。最终为公式 (4) 计算每根骨骼的变换矩阵 $M_i$。计算的中间步骤见附录 A。

> **图 7**：计算链式仿射变换的过程。第一框给定静止骨骼（源）与摆位骨骼（目标），目标是找变换 $M$ 把源线段映射到目标线段；最后一框中源线段经 RST 仿射变换后与目标对齐。

### 3.4. 实现

方法输入传统蒙皮输入：表面多边形网格、rig、rig 的关键帧姿态。给定关键帧姿态，用线性插值获得每帧的 rig 姿态。用户决定哪些骨骼是弹簧骨骼并指定质点-弹簧参数。另外，类似 [MK16]，我们也用**辅助骨骼**扩展主 rig 模拟局部动态——计算上辅助骨骼被当作普通骨骼处理，只是静止朝向由美术决定。

遵循 Complementary Dynamics [ZBLJ20] 的正交力概念，辅助弹簧骨骼设计为**补足现有刚体 rig**：多数辅助骨骼被放置得与负责一级运动的骨骼近似正交。

```
算法 3  Overall flow of our pipeline
Require: V⁰, W, B⁰, KB, θ, D, γ
1: initialize_simulator(B⁰, D, γ)
2: for all frames t do
3:     θt ← θ(t)                       ▷ 取当前帧骨髂姿态
4:     BR, MR ← forward_kinematics(B⁰, KB, θt)
5:     B* ← simulate(BR)               ▷ PBD 模拟自由质量
6:     BD ← kinematic_constraints(B*, KB)  ▷ 恢复连通与长度
7:     MD ← get_bone_transforms(BD)   ▷ RST 变换
8:     M ← compose_transforms(MD, MR, D)   ▷ 弹簧骨髂用 MD，刚体骨髂用 MR
9:     V ← skinning(V⁰, W, M)
10: end for
```

算法 3 中，弹簧骨骼索引 $D$、父子关系 $\mathcal{K}_B$、质点-弹簧参数 $\gamma: (k_s, k_d, d_s, m)$ 与静止形状 $V^0$、绑定权重 $W$、静止骨骼 $B^0$、动画关键帧 $\theta$ 由用户在初始化阶段给出。行 3 取当前帧骨骼姿态，行 4 前向运动学；模拟并投影运动学约束后，行 7 计算 RST 变换；行 8 组合最终骨骼变换——刚体骨骼保持前向运动学的 $M_R$，**只有弹簧骨骼**用 $M_D$ 更新。初始 rig 与质点-弹簧参数设置好后，管线自动计算动态骨骼朝向 $\mathcal{B}^D$ 与顶点位置 $V$。

## 4. 结果

弹簧骨骼在多种对象上产出发信的动态运动，包括刚体、弹性体、软组织与服装动画。例如花盆是刚体复合物：运动中叶片可以旋转平移但不允许缩放——用 fixed_scale 选项模拟附着叶片的骨骼、保持骨骼原长即可。

> **图 9**：摇晃的龟背竹花盆蒙皮。上：LBS 变形，白骨为刚体骨骼。中：蓝色弹簧骨骼实现的动态变形，只留一根负责摇晃的刚体白骨在根部。下：我们的变形——叶片左右摇晃而花盆保持刚体（通过给刚性根节点刷 1.0 权重实现）。

注意图 9、10、11 的植物/鸭子/布动画中，**只有根骨骼被用户旋转**；前向运动学使所有后代骨骼继承同一旋转——这个单一旋转在 LBS 中产生预期的僵硬运动。给定该输入，我们的方法**一次性**自动推断动态运动。无论底层几何蒙皮管线如何，我们产出的动态都很难手工复现——美术得为大量帧逐根骨骼 K 旋转关键帧。主要结果使用相近的刚度与阻尼参数；参数细节与更多二级动画贡献见补充视频。

结果用"刚体变形顶点与我们的变形的欧氏距离差"做**颜色编码**：红色区域表示与 LBS 差异最大，蓝色表示零差异（与 LBS 完全一致）。静止姿态在各图左侧，白骨为刚体、蓝骨为弹簧骨骼。

> **图 10**：橡皮鸭模型（弹性体）来回旋转。上：LBS。中：我们的。下：相对 LBS 的颜色编码。不模拟任何接触力，我们的方法获得仿佛鸭子漂浮在水面的平滑动态。
> **图 11**：SDS 应用于一块来回旋转的布料。平行的弹簧骨骼链在布料变形时形成凹凸表面。

### 4.1. 辅助骨骼

把现有 rig 骨骼转为弹簧骨骼允许用户在整个 3D 角色上施加**全局**动态；反过来，给原始 rig 添加弹簧辅助骨骼允许施加**局部**动态（如腹部脂肪抖动），作为互补运动。

我们在流行的 SMPL 模型 [LMR∗23] 上测试脂肪组织抖动：用 DFAUST 数据集 [BRPMB17] 的姿态序列，先用 SMPL 得到这些姿态的刚体变形；然后建立扩展 SMPL 骨架的辅助骨骼；框架模拟辅助骨骼后，把模拟的运动**叠加**到 SMPL 变形上，实现软组织抖动（图 13-14）。与经典蒙皮不同，SMPL 定义在 blendshape 上；因此 SMPL 演示中我们直接把骨骼尾端平移逐帧加到 blendshape 网格上：骨骼 $i$ 的变换取 $B^D_{2i}(t) - B^D_{2i}(t-1)$，$t \in [1,...,N]$，首帧取静止姿态 $B^D_{2i}(0) = B^0_{2i}$。我们观察到骨骼尾端平移足以在局部区域产生可信抖动；但如图 8 所示，该方法在大平面上会产生伪影。

> **图 13**：辅助弹簧骨骼（蓝）作为主骨骼的补充模拟脂肪组织动态。上：SMPL blendshape 动画。下：模拟骨骼运动叠加在 blendshape 变形上。
> **图 14**：同一组辅助弹簧骨骼在不同动画序列上模拟脂肪组织动态。颜色编码表示顶点欧氏距离相对上行的从高到低变化。

辅助骨骼对**服装动画**也很顺手：添加**平行弹簧链**可以近似布料动态（图 11）。图 12 展示失败案例——**单条**弹簧链无法近似布料表面。两个布料 rig 都用**热扩散权重** [BP07] 自动绑定到布面上。

> **图 12**：单条弹簧链的 SDS 无法产出布料动态。

弹簧 rig 的一个优势是相比传统布料模拟给用户**更多控制**。传统上用户只能通过改变质量与刚度参数控制材料属性，计算结果高度依赖物理模拟；我们的管线中用户同时控制**物理模拟参数**（质量、刚度）与 **rig 参数**（主/辅助骨骼的摆放与绑定权重）——这些控制实现多样的动态并提升自由度。

### 4.2. 对比

图 18 提供与 Benchekroun 等 Fast Complementary Dynamics [BZC∗23] 的对比。我们也与 Wu 与 Umetani [WU23] 比较——该工作提供可控制蒙皮动态框架：用 PBD 模拟四面体网格（约束在互补子空间，受 Zhang 等 Complementary Dynamics [ZBLJ20] 启发），再用逆运动学推断动态句柄位置。与我们类似，Wu 与 Umetani 也想统一物理模拟与蒙皮管线；但其方法与我们的互补：**他们先模拟网格、再推断句柄位置；我们先模拟 rig、再基于模拟的骨骼变换推断动态变形顶点。**

> **图 15**：摇晃运动对比。(a) LBS。(b) [WU23]：固定句柄（红点）意在限制 PBD 模拟在受约束顶点上；尽管所有句柄都固定、腿部句柄动画中未移动，牛腿仍受四面体网格模拟影响（细节框），降低用户可控性。(c) 主句柄上用弹簧骨骼产生全局挤压拉伸动态。
> **图 16**：带辅助骨骼 rig 的对比。上三行为动画中 rig 的运动，下三行为 LBS、[WU23] 与我们的最终网格渲染。静止姿态在左侧。蓝色弹簧骨骼用于模拟大象耳朵与象鼻。

与 [WU23] 的对比中，变形基于**点句柄**；而经典蒙皮以骨架层级为特征。我们的 RST 算法为骨架骨骼设计（假设骨骼长度为正），因此这些对比中只用点句柄的**平移**做蒙皮变换。我们观察到：非平坦表面的局部动态中平移就足够（图 13 SMPL 演示亦然）。把这些点句柄转为点弹簧骨骼，形成零长度弹簧（图 2）。图 15 的对比展示点弹簧句柄产生全局二级运动；局部化软组织抖动需要额外的弹簧辅助骨骼（如图 16 大象耳朵与象鼻周围）。

模拟四面体网格可能在网格上引入**不可预测的运动**（图 15）；相比之下我们的方法不依赖网格顶点、保持可控。此外，受 [ZBLJ20] 启发，Wu 与 Umetani (2023) 引入与刚体运动**正交**的互补运动，但结果运动可能不符合期望动态：如图 15，奶牛 Spot 前后摇晃，而互补运动产出牛头左右旋转——完全不同的运动。相比之下，我们的点弹簧通过**同方向夸张**运动增强摇晃。

Wu 与 Umetani 四面体模拟的另一问题出现在**精细 rig** 上（图 16 第二、五行）：体模拟中塌陷蒙皮伪影常表现为相邻四面体间的裂缝；精细 rig 下这些伪影在邻近骨骼周围更频繁，导致自相交与多个弹簧上的快速体积损失。在时间步进方案下，该状况会造成邻域弹簧力的突变并最终放大弹簧力；其框架中 PBD 模拟与逆运动学有反馈回路，在这种条件下易严重不稳定。此外精细 rig 输入增大互补动力学的刚度（图 18），与"通过 rig 控制变形"的直觉矛盾。我们的方法基于 rig 模拟，动态基本**独立于网格分辨率**、可通过句柄控制。最后，四面体化应用于现有动画并非易事——需要重组绑定权重与纹理、容易过度耗费人工；我们的方法直接在壳网格上工作，容易用于现有已 rig 角色。简言之，弹簧骨骼避免四面体模拟的缺点，在不同网格与 rig 分辨率下保持可控与可扩展。

> **图 18**：与 Fast Complementary Dynamics [BZC∗23] 对比。(a) 单句柄。(b) 完整 rig（浅蓝骨骼）。(c) 我们的完整 rig（弹簧骨骼为蓝色）。注意 [BZC∗23] 加详细 rig 后，鸭身上绑定多根骨骼的部位动态严重减少、二级运动集中在绑定单根骨骼的底板上；我们的方法通过句柄控制二级运动——底板用单根刚体骨骼保持刚性、鸭子其余部分跟随弹簧骨骼动态运动。

### 4.3. 耗时

与 LBS 类似，我们的时间复杂度主要取决于 **rig 骨骼数**而非顶点数。如表 1，Duck 模型顶点数显著更多但骨骼更少，变形快于 Cloth 与 Monstera。SMPL 变形更快是因为其演示不含蒙皮计算（骨骼尾端平移直接叠加在 blendshape 上）。其余模型用 RST 算法计算骨骼变换。

| 模型 | \|V\| | \|Bs\|/\|B\| | LBS (ms) | Ours (ms) |
|---|---|---|---|---|
| Cloth | 2025 | 22/23 | 5.26 | 12.21 |
| Monstera | 4971 | 23/24 | 6.19 | 14.11 |
| Duck | 12932 | 5/6 | 4.51 | 9.50 |
| SMPL | 8541 | 9/25 | N/A | 4.67 |

> **表 1**：逐帧平均耗时。$|V|$ 顶点数；$|B_s|$ 弹簧骨骼数 / $|B|$ 总骨骼数。SMPL 基于 blendshape 蒙皮故无 LBS。不含渲染耗时。

我们的方法约比 [WU23] **快 10 倍**：实现主要依赖 rig 骨骼数、避免模拟四面体网格顶点。对比时（表 2）所有模型都加辅助骨骼，骨骼总数翻倍。

| 模型 | \|Vtet\| | \|B\| | LBS (ms) | Ours (ms) | [WU23] (ms) |
|---|---|---|---|---|---|
| Spot | 837 | 8 | 1.15 | 1.98 | 20.73 |
| Spot | 837 | 15 | 1.86 | 3.45 | 27.10 |
| Spot (HQ) | 1762 | 8 | 1.29 | 2.40 | 22.97 |
| Spot (HQ) | 1762 | 15 | 2.13 | 4.05 | 31.23 |
| Elephant | 8541 | 25 | 4.14 | 7.87 | 90.41 |
| Elephant | 8541 | 47 | 6.16 | 11.91 | 114.79 |

> **表 2**：带体网格输入的对比耗时（毫秒，500 帧平均）。$|V_{tet}|$ 四面体顶点数。一致性起见，我们的所有骨骼都作弹簧骨骼，[WU23] 都作固定骨骼。

实验环境：2.5 GHz 四核 Intel Core i7、16 GB RAM；Python 3.9.18；我们用 NumPy 1.26.4，Wu 与 Umetani 用 Taichi 1.7.1。

## 5. 讨论

### 5.1. 工业界方法

工业方法用 jiggle bones 模拟二级运动，专注动物尾巴、马尾辫毛发或肌肉抖动。[Wan23]（《战神：诸神黄昏》GDC 演讲）把弹簧运动**约束在锥形范围内**，为肌肉抖动创建可控范围；Unreal Engine 也提供类似 [Wan23] 的动态骨骼。据我们所知，现有应用**不考虑骨骼缩放**；相比之下我们也提供缩放骨骼的选项，实现全局的挤压拉伸二级运动（类似 Velocity Skinning，图 15）。固定/自由质量及其运动学约束的引入，让我们可以定义缩放、或可选地保持原始骨骼长度。在工业应用普遍采用的半隐式 Euler 之上，我们用带可选约束的 PBD——这对捕获局部动态（如腹部脂肪抖动，图 13、14）很有用。总体上该框架允许定义**局部与全局**二级动态，泛化到广泛的应用。

Unity、Blender 等工具也为质点-弹簧模拟提供布料动态，但它们典型是**直接作用于网格顶点**的经典模拟；相比之下，我们的工作引入**基于蒙皮的框架**，把动态网格变形表示为**少量弹簧的线性组合**而非逐顶点独立模拟，把二级运动泛化。虽然现有工具里可以通过给顶点组指定父子关系创建 jiggle-bone 效果，但 SDS 把这一过程**形式化为统一框架**，为未来二级运动系统的增强提供基础。

### 5.2. 创作工作流

框架设计为易于集成到现有蒙皮管线、可快速应用于常见 3D 动画软件：在 Blender 等软件中创建的已 rig 动画序列可导出到本框架；用户选好弹簧骨骼、设置模拟参数，即可**即插即用**地获得动态变形。补充视频中的参数设置在不同模型间基本一致。必要时用户可加辅助骨骼驱动局部二级运动——这需要重算绑定权重，但多数情况下**自动的顶点-骨骼绑定方法已足够**；少量权重绘制可用于施加刚性材料约束（如把植物花盆绑到刚性骨骼、图 9；鸭子底板、图 18）。展示的结果主要依赖自动**热扩散权重**，避免大量手工权重绘制，提供快速原型。

### 5.3. 局限与未来工作

迄今为止我们研究 Hooke 弹簧骨骼，因其正弦性质与简单性。尽管结果有前景，系统继承了 Hooke 弹簧力的主要局限：[MHTG05] 讨论过，质点-弹簧系统对参数敏感，错误设置可能导致不稳定；即便有 PBD 这样无条件稳定的框架，**爆炸的力**仍然存在——也因为系统是**基于位置而非基于力**的：用户直接确定骨骼位置而非施加影响力。未来可探索更鲁棒的弹簧系统改善稳定性。

弹簧骨骼限于 **3 自由度**：只在 3D 空间沿 x/y/z 平移，不扭曲不弯曲。为获得更大自由度与扭曲运动，可以像 [GBFP11] 那样考虑**角速度**。RST 算法中我们直接用矩阵旋转向量；对 DQS 等蒙皮方法可采用**四元数**旋转，缓解旋转矩阵潜在的体积塌陷。RST 算法的适用范围也值得进一步评估。

我们还观察到**更长的辅助骨骼链更难控制与调参**——也因为无碰撞检测时可能自相交。碰撞检测与整合其它外力留作未来工作。最后，模拟参数目前为达到期望二级动画而**手工设置**；尽管参数常在不同例子间共享，SDS 框架的**自动化**也是开放的研究领域。

## 6. 结论

Spring Decomposed Skinning 提出一个简单而有效的框架，在现有几何蒙皮管线上模拟实时、多样的二级动画。SDS 动态的范围覆盖刚体、弹性体、软组织与服装动画。与常见 FEM 方法不同，SDS 不需要网格四面体化，显著改善时间复杂度、易于集成到现有动画管线。此外 SDS **直接模拟骨骼**，为物理变形提供美术控制。与直接模拟网格顶点的体网格模拟相反，本方法通过把复杂表面动态分解为简单弹簧运动的线性组合，与蒙皮背后的核心直觉对齐。最终，SDS 展示了实现实时、可控、多样动态的有前景方向。

## 致谢

感谢匿名评审的建设性意见，感谢 Arcturus Studios 的 Devin Horsman 带来的早期启发讨论。本工作受 TUBITAK 项目 EEEAG-124E183 资助。

## 附录 A：RST 的计算

初始时，把静止骨骼与变形骨骼两条线段都移到**骨骼空间**（骨骼头位于原点）。预计算阶段，为每根骨骼 $i$ 存储偏移矩阵及其逆：

$$B_o = \begin{bmatrix} I & -B^0_{1i} \\ 0 & 1 \end{bmatrix}, \qquad B_o^{-1} = \begin{bmatrix} I & B^0_{1i} \\ 0 & 1 \end{bmatrix}$$

其中 $I \in \mathbb{R}^{3\times3}$ 为单位矩阵。然后用偏移 $B_o$ 把静止骨骼 $i$ 与模拟后骨骼 $i$ 都平移到骨骼空间（图 7 步骤 1）。骨骼空间中，RST 矩阵的平移部分为：

$$t^*_i = B^D_{1i} - B^0_{1i} \tag{5}$$
$$T = \begin{bmatrix} I & t^*_i \\ 0 & 1 \end{bmatrix} \tag{6}$$

把目标骨骼平移到原点后，其尾端位置 $B^{D'}_{2i}$ 与骨骼空间中静止骨骼尾端 $B^{0'}_{2i}$ 为：

$$B^{D'}_{2i} = B^D_{2i} - B^0_{1i} - t^*_i \tag{7}$$
$$B^{0'}_{2i} = B^0_{2i} - B^0_{1i} \tag{8}$$

此时问题变为求对齐旋转 $R: B^{0'}_{2i} \to B^{D'}_{2i}$。设 $\theta$ 为两单位向量 $p, q \in \mathbb{R}^3$（$\|p\| = \|q\| = 1$）之间的角，$c = \cos\theta = p \cdot q$，$s = \sin\theta = \|p \times q\|$；旋转轴 $r = [r_x, r_y, r_z] = p \times q / \|p \times q\|$，$\kappa = 1 - c$。把 $q$ 对齐到 $p$ 的旋转矩阵 $R^*$ 用 Rodrigues 公式（矩阵形式）计算：

$$R^* = \begin{bmatrix} \kappa r_x^2 + c & \kappa r_x r_y - r_z s & \kappa r_x r_z + r_y s \\ \kappa r_x r_y + r_z s & \kappa r_y^2 + c & \kappa r_y r_z - r_x s \\ \kappa r_x r_z - r_y s & \kappa r_y r_z + r_x s & \kappa r_z^2 + c \end{bmatrix} \tag{9}$$

$$R = \begin{bmatrix} R^* & 0 \\ 0 & 1 \end{bmatrix} \tag{10}$$

弹簧模拟会缩放弹簧骨骼。为匹配静止骨骼（源）与动态摆位骨骼（目标）的缩放，用旋转矩阵 $R^*$ 旋转静止骨骼尾端 $B^{0'}_{2i}$。两对齐向量间的缩放为：

$$s = \frac{\|B^{D'}_{2i}\|}{\|R^* B^{0'}_{2i}\|} \tag{11}$$

$$S = \begin{bmatrix} sI & 0 \\ 0 & 1 \end{bmatrix} \tag{12}$$

把公式 (10)、(12)、(6) 的旋转、缩放、平移矩阵组合为：

$$M^* = T S R \tag{13}$$

最后把骨骼从骨骼空间变换回世界空间，整体仿射可链为单一变换：

$$M^D_i = B_o^{-1} M^* B_o \tag{14}$$

用它把静止姿态骨骼变换到摆位空间，供底层蒙皮使用。实验中公式 (4) 的变换矩阵 $M_i$ 即用动态骨骼变换 $M^D_i$。

## 参考文献（保留英文原题与引用键，压缩格式）

[KSO10] Kavan et al. Fast and efficient skinning of animated meshes. CGF 2010. · [LD12] **Le & Deng. Smooth Skinning Decomposition with Rigid Bones. TOG 2012.** · [MG03] Mohr & Gleicher. Building efficient, accurate character skins from examples. TOG 2003. · [Muk15] Mukai. Building helper bone rigs from examples. I3D 2015. · [MK16] Mukai & Kuriyama. Efficient dynamic skinning with low-rank helper bone controllers. TOG 2016. · [Muk18] Mukai. Sampling-based rig conversion into non-rigid helper bones. PACMCGIT 2018. · [CM24] Cao & Mukai. Skeleton-aware skin weight transfer for helper joint rigs. EG Short 2024. · [CGC∗02] **Capell et al. Interactive skeleton-driven dynamic deformations. TOG 2002.** · [CBC∗05] Capell et al. Physically based rigging for deformable characters. SCA 2005. · [HTC∗13] Hahn et al. Efficient simulation of secondary motion in rig-space. SCA 2013. · [HMT∗12] Hahn et al. Rig-space physics. TOG 2012. · [TRPO21] Tapia et al. Parametric skeletons with reduced soft-tissue deformations. CGF 2021. · [RTK∗21] Rohmer et al. Velocity Skinning for real-time stylized skeletal animation. CGF 2021. · [SRKZ24] Salem et al. Art-directable Expressive Oscillation Behavior for Rigged Characters. SCA Posters 2024. · [ZBLJ20] Zhang et al. Complementary Dynamics. TOG 2020. · [BZC∗23] Benchekroun et al. Fast Complementary Dynamics via skinning eigenmodes. TOG 2023. · [WU23] **Wu & Umetani. Two-way coupling of skinning transformations and position based dynamics. PACMCGIT 2023.** · [JHG∗22] **Jin, Han, Geng, Teran, Fedkiw. Analytically integratable zero-restlength springs for capturing dynamic modes unrepresented by quasistatic neural networks. SIGGRAPH 2022.**（译注：[[Jin2024-RopeChainQNN|Jin 2024]] 的前身） · [ZZCB21] Zheng et al. A deep emulator for secondary motion of 3D characters. CVPR 2021. · [CO18] Casas & Otaduy. Learning nonlinear soft-tissue dynamics for interactive avatars. PACMCGIT 2018. · [SGOC20] Santesteban et al. SoftSMPL. CGF 2020. · [MZS∗11] McAdams et al. Efficient elasticity for character skinning with contact and collisions. SIGGRAPH 2011. · [KB18] Komaritzan & Botsch. Projective skinning. PACMCGIT 2018. · [KBB∗17] Kozlov et al. Enriching facial blendshape rigs with physical simulation. CGF 2017. · [RF14/RF16] Rumman & Fratarcangi. Position based skinning / State of the art in skinning techniques. · [AS07] Angelidis & Singh. Kinodynamic skinning using volume-preserving deformations. SCA 2007. · [RRC∗18] Rousselleet et al. Dynamic implicit muscles for character skinning. C&G 2018. · [LST09] Lee, Sifakis, Terzopoulos. Biomechanical upper body modeling. TOG 2009. · [GBFP11] Gilles et al. Frame-based elastic models. TOG 2011. · [LBOK13] Liu et al. Fast simulation of mass-spring systems. TOG 2013. · [SLF08] Selle et al. A mass spring model for hair simulation. SIGGRAPH 2008. · [TN19] Toothman & Neff. Spring rigs for skinning. MIG 2019. · [Wan23] Wang. Joint-based skin deformation in God of War Ragnarök. GDC 2023. · [MHHR07] Müller et al. Position Based Dynamics. 2007. · [MHTG05] Müller et al. Meshless deformations based on shape matching. TOG 2005. · [BMOT13] Bender et al. Position-based methods for the simulation of solid objects. EG STAR 2013. · [BP07] Baran & Popović. Automatic rigging and animation of 3D characters（热扩散权重）. TOG 2007. · [LMR∗23] Loper et al. SMPL. 2023. · [BRPMB17] Bogo et al. Dynamic FAUST. CVPR 2017. · [SHR17] Sorkine-Hornung & Rabinovich. Least-squares rigid motion using SVD. 2017. · 其余：[NT98] 面部质量-弹簧肌肉；[TT93] 弹性表面层模型；[LTW95] 面部质点-弹簧；[KHS01] 面部几何肌肉建模；[MHYH17] 肌骨+软组织皮肤模拟；[Gol18] 软组织混合质点-弹簧；[vFTS07] 向量场积分弹性二级变形；[DJBDT10] 稳定逆动态曲线；[WLP∗17] 2D 动画二级运动；[MMRH22] SkinningNet；[YWM∗21] S3 神经形状/骨架/蒙皮场；[SGXT20/SGX∗21] PhysCap/物理感知动捕；[PMRMB15] Dyna；[IKNDP16] 用户特定体积面部 rig；[ARF15] 软关节角色位置蒙皮；[Witkin & Baraff 2001] 物理建模课程；[BL21] 网格同化；[YS09/SY10/SK16] 形状重建与网格展开相关。
