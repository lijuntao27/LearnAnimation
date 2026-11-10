---
type: translation
title: AnchorDEF - Learning Anchor Transformations for 3D Garment Animation (2023) 中文译本
source: RawSource/articles/AnchorDEF-2023.pdf
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, anchor, skinning, lbs, qem, cvpr-2023, translation]
summary: Zhao 等 CVPR 2023 原论文中文学习译本：AnchorDEF 用网格表面自适应锚点的刚体变换混合 + canonical 空间逐顶点位移表示服装变形，位置/法向/方向一致性约束保证锚点变换的物理含义，QEM 网格简化监督把锚点推向褶皱与边界；宽松裙装与紧身 T 恤的 RMSE/Hausdorff/STED 全面优于 TailorNet 与 VirtualBones。原始英文见 [[AnchorDEF-2023]]。
---

# Learning Anchor Transformations for 3D Garment Animation — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），结果表用 Markdown 表格，正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[AnchorDEF-2023|原索引卡]] 所指的 `AnchorDEF-2023.txt`。
>
> 论文为 arXiv 开放获取（2304.00761），版权归 CVPR 2023（IEEE/CVF）所有，本译本按学术学习使用处理。
>
> **抽取工具说明**：`.txt` 用 PyMuPDF 抽取（本目录），PDF 内公式被拆散到多行，译文中已按论文语义重组为 LaTeX；个别符号（如下标、撇号）如与原 PDF 图例有出入，以 PDF 为准。
>
> **关键术语对照**：anchor=锚点；blend weight=（蒙皮）混合权重；canonical space=典型（canonical）空间，指未摆姿态的模板空间；garment-body interpenetration=服装-身体相互穿透；mesh simplification=网格简化；QEM=Quadric Error Metric，二次误差度量。
>
> 与本谱系的关系：本文是 [[Pan2022-VirtualBones]] 之后、[[HyperBones2026]] 骨骼来源（FPS 面采样）之前的**锚点线**关键节点，对比分析见 [[BoneDrivenCloth-CitationMap]]，对 SkirtQNN 的落地讨论见 [[SkirtQNN-Design]]。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Learning Anchor Transformations for 3D Garment Animation |
| 作者 | Fang Zhao¹, Zekun Li¹, Shaoli Huang¹*, Junwu Weng¹, Tianfei Zhou², Guo-Sen Xie³, Jue Wang¹, Ying Shan¹（*通讯作者） |
| 机构 | 1 腾讯 AI Lab；2 ETH Zurich；3 南京理工大学 |
| 发表 | CVPR 2023 |
| arXiv | 2304.00761v1 [cs.CV]，2023-04-03 |
| 代码/主页 | https://semanticdh.github.io/AnchorDEF（演示页） |

## 摘要

本文提出一种**基于锚点的变形模型**，即 **AnchorDEF**，从身体运动序列预测 3D 服装动画。它通过**刚体变换的混合**加上**额外的非线性位移**来变形服装网格模板。我们在网格表面周围引入一组**锚点（anchors）**来指导刚体变换矩阵的学习。一旦锚点变换确定，服装模板的逐顶点非线性位移就可以在一个 **canonical 空间**中回归，这降低了变形空间学习的复杂度。通过显式约束**变换后的锚点**满足**位置、法向和方向的一致性**，所学锚点变换在空间中的物理含义得到保证，从而获得更好的泛化。此外，本文提出**自适应锚点更新**（adaptive anchor updating），通过感知局部网格拓扑来优化锚点位置，以学习有代表性的锚点变换。在不同类型服装上的定性与定量实验表明，AnchorDEF 在运动中的 3D 服装变形预测上取得了业界最佳性能，**尤其是对宽松服装**。

## 1. 引言

3D 服装动画在 3D 内容生成、数字人、虚拟试穿、视频游戏等领域有广泛应用。现有的 3D 服装动画管线通常依赖**基于物理的模拟（PBS）**，它需要大量计算资源和时间成本，高质量 PBS 方法尤其如此。一些数据驱动或基于学习的方法已被提出，用较低的计算复杂度从静态姿态或运动序列快速产出 3D 服装变形 [4,5,9,11,12,17,26–29,32,41]。然而它们中的许多把服装模板挂接（attach）到人体骨架上来建模服装的关节运动，这只对紧身服装（如 T 恆、裤子）有效，而对宽松服装（如连衣裙、裙子）处理得很差。

> **图 1**：用所提 AnchorDEF 根据身体运动预测的 3D 服装变形。借助锚点变换，AnchorDEF 能够真实地变形服装网格，尤其是宽松服装，如连衣裙。

在这种情况下，服装的拓扑与人体不同，因此使用身体的蒙皮混合权重会在变形后的服装网格上产生**不连续**。若干方法 [27,35] 通过**平滑混合权重**来缓解不连续，但对宽松服装的大变形可能丢失形状细节，因为这类服装并不紧贴身体运动。

为此，我们提出基于锚点的变形模型 AnchorDEF，从身体运动序列预测 3D 服装变形。它利用**刚体锚点变换的混合**来表示服装的主要旋转和平移，把服装的关节运动**从身体骨架上解耦**，同时保留身体运动先验；随后非线性位移可以在 canonical 空间中相对容易地回归出来。如图 1 所示，我们的方法能利用锚点变换真实地变形服装网格，尤其是连衣裙等宽松服装。

具体地，给定包含姿态和平移的身体运动序列，我们首先估计服装网格周围一组锚点的刚体变换。使用线性混合蒙皮（LBS），服装网格模板由锚点变换的加权组合来变形；与此同时，网格模板的**逐顶点位移**被回归出来，以修正混合刚体变换产生的伪影。为了让学到的锚点变换具有物理含义以获得更好的泛化，我们强制**变换后的锚点**与目标在**位置和法向**上保持一致。此外，我们采用一个**相对方向约束**来减少服装-身体的相互穿透，由于锚点的稀疏性这一约束非常高效。为了让学到的锚点变换能有效表达服装变形，本文进一步引入**自适应锚点更新**，利用**网格简化**作为监督来优化锚点位置：它用相邻网格顶点上的**局部注意力掩码**来参数化锚点位置，把锚点推向通常决定变形方式的**褶皱和网格边界**。

我们的主要贡献可总结为：1）提出一个基于锚点的变形模型，在统一框架内学习一组锚点变换和混合权重来表示 3D 服装变形，尤其是宽松服装；2）提出通过位置与法向一致性以及相对方向约束来学习锚点变换，以获得更好的泛化和更少的服装-身体穿透；3）引入以网格简化为监督的自适应锚点更新，优化锚点位置以学习有代表性的锚点变换。

## 2. 相关工作

多数服装动画与变形工作可分为**基于物理的模拟**与**基于学习的模型**两类。

**基于物理的模拟。** PBS 方法已发展数十年并取得了令人印象深刻的精度，它们利用物理定律与稳定精确的求解器 [1,13,19]、碰撞检测 [24,30,36] 以及干摩擦接触 [20,22]。但其高昂的计算成本——长时间模拟与高性能设备——在日常生活中通常难以承受。

**基于学习的模型。** 相比之下，基于学习的模型利用从 PBS 或数字扫描获得的服装动画数据来学习变形，通常能以更低的计算成本给出足够满意的动画视觉效果 [27,32]，尽管没有显式的物理精度保证。早期研究者提出了许多依赖人体运动的方法，如线性条件模型 [8,11] 或**次级运动图**（secondary motion graphs）[14,15]，它们都依赖一个高分辨率网格构成的数据库。近期工作 [4,27,35] 假设服装是蒙皮多人线性模型（SMPL）的子网格，用 LBS 把身体形状迁移到布料形状。在单帧静态结果之外，采用**循环神经网络**处理衣物动态皱纹被广泛使用 [26,32,35]。例如 [32]（译注：VTO / Santesteban 2019）为若干体型和运动的着装角色准备了 PBS 数据库，并用该数据库学习"体型+动态"的函数来建模布料垂坠和皱纹。虽然基于学习的模型效率高，其性能严重依赖大量高质量 PBS 结果或扫描数据，这些通常很昂贵 [2,31,35]。因此一些工作 [3,33] 提出以某种物理定律约束模型的**自监督训练**方法来缓解该问题。此外，为保障模拟的泛化性，组合若干预训练的 **pivot motion 网络**可以处理不同的未见服装风格 [27] 与模拟参数 [26]。

但多数基于学习的方法局限于紧身服装，不适合连衣裙或更自由的衣物，因为它们假设服装与 SMPL 互补，即变形与身体**形状**强相关而非与**运动**强相关。为处理动态问题，一些工作提出神经场以保留高频细节 [40]、处理困难区域 [23]、处理多层服装 [34]，以及把服装从 canonical 姿态扭曲到变形形状 [6]。不利用身体形状参数建立服装-身体对齐模型，[38]（译注：Wang 2019 intrinsic garment space）学习一个内蕴服装形状描述子嵌入空间，用不同的运动参数引导服装 latent 可得到不同的形状表示。为消除衣物动态与身体顶点间的对齐，**VirtualBones** [26] 从服装动画中提取一组刚体骨骼，用身体运动预测骨骼变换来插值服装形状。不基于网格学习，[39]（译注：Zakharkin 点云 Point-Based Modeling）用点 Transformer 预测各种穿搭的点云，对多种人体姿态与形状都能工作，获得了更多拓扑灵活性以及把衣物与身体分开建模的能力。

**VirtualBones [26] 是与我们最相关的工作**，它需要以 SSDR [18] 做服装骨架提取作为数据预处理。与 [26] 不同，我们用一组**自适应锚点**分解服装变形，锚点的刚体变换和混合权重在**统一框架**内联合学习。

## 3. 我们的方法

我们目标是学习一个动画模型，给定身体运动序列即可预测动态 3D 服装变形。本节介绍 AnchorDEF——一个基于锚点的变形模型。它利用网格表面周围的一组**自适应锚点**，以**刚体锚点变换的混合 + canonical 空间中附加的逐顶点位移**来变形服装网格模板；锚点变换由位置、法向和方向的一致性学习得到。总览见图 2。

> **图 2**：AnchorDEF 总览。给定身体运动序列和服装网格模板，服装变形由"刚体锚点变换的加权平均 + canonical 空间逐顶点位移"预测。我们显式约束变换后的锚点满足位置、法向和方向的一致性，以保证所学锚点变换在空间中的物理含义，获得更好泛化。锚点从网格顶点聚类中心出发，通过相邻网格顶点上的局部注意力图、以网格简化为监督来更新位置。模型所有参数可在统一框架内联合优化。

### 3.1 用锚点做变形

人体的 3D 动画模型（如 SMPL [21]）通常用 blendshape 与蒙皮变形身体网格模板 $\mathcal{T}$：

$$V = W(\mathcal{T} + B(\theta), \mathcal{J}, \theta, w), \tag{1}$$

其中 $W$ 是蒙皮函数（如线性混合蒙皮），输入为骨架 $\mathcal{J}$、姿态参数 $\theta$ 和混合权重 $w$；$V$ 是摆好姿态的顶点；$B$ 是与身体姿态 $\theta$ 对应的 blendshape。由于服装是一种软体，很难从身体运动序列直接预测每个顶点的坐标，这里我们同样把服装变形分解为两部分：**刚体变换的加权平均**与**非线性位移**。

多数数据驱动服装模型直接把服装与人体骨架关联 [4,27,32]，即直接借用邻近身体网格顶点的混合权重来蒙皮动画化服装——这隐含假设服装紧贴身体运动。然而对宽松服装而言，服装变形通常包含比"可视为刚体的人体"复杂得多的**非刚体效应**，服装的行为与身体显著不同，把服装挂接到身体骨架上并不合理。

为解决该问题，我们在网格表面周围引入一组锚点 $\{p_n\}$ 来引导服装变形，**把服装从身体骨架解耦，同时保留身体运动先验**。给定由姿态和平移构成的身体运动序列，首先用门控循环单元（GRU）[7] 估计刚体锚点变换——旋转 $\{R_n\}$ 与平移 $\{T_n\}$（网络结构见补充材料）。随后，第 $m$ 个变形后服装顶点 $v_m$ 可表示为锚点变换的加权平均：

$$v_m = \sum_{n=1}^{N} w_{mn} (R_n v^c_m + T_n), \qquad w_{mn} \ge 0 \text{ 且 } \sum_{n=1}^{N} w_{mn} = 1, \tag{2}$$

其中 $v^c_m$ 是服装网格模板的第 $m$ 个顶点，$w_{mn}$ 是 $v_m$ 关于第 $n$ 个锚点的混合权重，$N$ 是锚点数。为确定锚点-顶点关系并维持 $\{w_n\}$ 的**稀疏性**，我们把锚点初始化为网格模板顶点的**聚类中心**；然后每个顶点只受其 $N'$（$N' \ll N$）个最近锚点的混合权重影响，其余权重在训练中保持为零。与直接预测变形网格顶点坐标相比，刚体锚点变换以少得多的预测参数为复杂变形提供了更泛化的表示。

针对**宽松**与**紧身**两类服装，我们设计了两种锚点变换配置策略。对宽松型，为利用全局人体朝向先验 $R_{global}$，我们预测**相对于身体根关节** $J_{root}$ 的变换：

$$R^{loose}_n = R_{global} R_n([\theta, t]), \tag{3}$$
$$T^{loose}_n = R_{global} \big( T_n([\theta, t]) - R_n([\theta, t]) J_{root} \big) + J_{root},$$

其中 $\theta$ 和 $t$ 分别是身体姿态与平移。对紧身型，服装大体跟随身体运动，因此我们预测在**身体顶点旋转 $R_{body}$ 与偏移 $T_{body}$ 基础上的增量**：

$$R^{tight}_n = R_{body} R_n([\theta, t]), \tag{4}$$
$$T^{tight}_n = R_{body} T_n([\theta, t]) + T_{body},$$

其中 $R_{body}, T_{body}$ 是离锚点 $p_n$ 最近的身体网格顶点的变换。此外，对同时有紧身与宽松部分的服装（如连衣裙），可以按**锚点到身体关节的距离**分类锚点所属部分，组合这两种策略。

一旦服装变形中由关节运动引起的主要旋转与平移被公式 (2) 表达，**非线性残差变形**就可以与服装关节运动解耦，定义为 **canonical（未摆姿态）空间中的逐顶点位移** $D([\theta, t])$。利用公式 (1) 的蒙皮函数，整体服装变形为：

$$V_{pred} = W(\mathcal{T}_g + D([\theta, t]),\ G([\theta, t]),\ w_p), \tag{5}$$

其中 $\mathcal{T}_g$ 是服装网格模板，$w_p$ 是锚点-顶点混合权重，$G([\theta, t])$ 是由身体运动预测的锚点变换。我们认为这种按锚点的分解**极大降低了姿态相关的服装变形预测复杂度**：大变形已由稀疏锚点的变换处理，canonical 空间中只剩小位移需要回归。

> **图 3**：用局部注意力掩码 + 网格简化监督做自适应锚点更新的例子。蓝点是 k-means 初始化的锚点，橙点是更新后的锚点。

### 3.2 学习锚点变换

与 [26] 求助外部蒙皮分解算法 SSDR [18] 从平滑网格序列提取服装骨架不同，我们在统一框架内**联合学习**锚点的刚体变换、混合权重与位置。

首先对预测网格顶点施加 L2 损失以恢复变形网格形状：

$$\mathcal{L}_{rec} = \| V_{pred} - V_{gt} \|_2^2, \tag{6}$$

最小化预测顶点与真值变形顶点间的欧氏距离。然后使用保持两网格**图拉普拉斯**相似的损失项：

$$\mathcal{L}_{lap} = \| \triangle V_{pred} - \triangle V_{gt} \|_2^2. \tag{7}$$

再加一个碰撞项 [26] 以避免服装-身体碰撞。顶点上的损失为：

$$\mathcal{L}_{vert} = \mathcal{L}_{rec} + \beta_1 \mathcal{L}_{lap} + \beta_2 \mathcal{L}_{collision}, \tag{8}$$

其中 $\beta_1, \beta_2$ 是控制平滑与碰撞程度的权衡系数。

然而，**只优化顶点位置无法保证所学锚点变换在空间中的物理含义**，会导致泛化差。因此我们显式约束**变换后的锚点位于变形网格表面附近**。为此，锚点 $p$ 的位置由其相邻网格顶点重新计算：

$$p = \frac{1}{K} \sum_{v^c_k \in K(p, \mathcal{T}_g)} v^c_k, \tag{9}$$

其中 $K(p, \mathcal{T}_g)$ 是 $p$ 在网格模板上的 $K$ 近邻顶点集合。然后变换后锚点 $p_G$ 的**目标点** $p_{tgt}$ 用同一组顶点 $K(p, \mathcal{T}_g)$ 在**变形后网格**上的位置定义，变换一致性损失为：

$$\mathcal{L}_{consis} = \| p_G - p_{tgt} \|_2^2 + \gamma \, \| n(p_G) - n(p_{tgt}) \|_2^2, \tag{10}$$

其中 $n(\cdot)$ 是点的单位法向，$\gamma$ 是权重因子。通过鼓励变换后锚点 $p_G$ 与 $p_{tgt}$ 在位置和法向上相似，所学锚点变换就表示了 $p$ 的 $K$ 近邻顶点的**平均变换**。它可以看作对变换幅度与方向的**正则化**，用于发现变形中的本质变换。

我们进一步强制 $p_{tgt}$ 指向 $p_G$ 的方向与 $n(p_{tgt})$ 的方向一致，最小化如下惩罚：

$$\mathcal{L}_{dir} = 1 - \cos(p_G - p_{tgt},\ n(p_{tgt})), \tag{11}$$

其中 $\cos(\cdot,\cdot)$ 是余弦相似度。直观上，该惩罚使 $p_G$ 倾向于位于 $p_{tgt}$ 形成的平面**外侧**，以减少服装-身体穿透。

> **图 4**：去掉 (a) 变换一致性 $\mathcal{L}_{consis}$ 与 (b) 自适应锚点更新 $\mathcal{L}_{anch}$ 的定性对比。无 $\mathcal{L}_{consis}$ 时模型无法准确预测变形网格形状、丢失部分表面几何细节；无 $\mathcal{L}_{anch}$ 时锚点变换表达不足，无法产出真实的服装变形。

**自适应锚点更新。** 仅有物理含义的锚点变换还不够，我们还需要**几何上有表达力**的变换来更好地表达服装变形。我们观察到，**褶皱与边界区域**通常对表现服装形状特征起重要作用，相比平滑区域包含更多变形信息。为了让锚点反映服装形状的内蕴结构，我们提出用**网格简化**提供监督信号来寻找有代表性的锚点。这里用**二次误差度量（QEM）**[10] 通过折叠边获得简化网格顶点，同时保持服装拓扑。由于锚点 $\{p_n\}$ 与简化顶点 $\{\tilde{v}_m\}$ 之间没有一一对应，采用 **Chamfer 距离**作为优化目标：

$$\mathcal{L}_{anch} = \sum_{p} \min_{\tilde{v}} \| p - \tilde{v} \|_2^2 + \sum_{\tilde{v}} \min_{p} \| p - \tilde{v} \|_2^2. \tag{12}$$

为使锚点位置可更新，我们用可学习权重 $\alpha$ 把公式 (9) 重写为：

$$p = \frac{\sum_{k=1}^{K} e^{\alpha_k} v^c_k}{\sum_{j=1}^{K} e^{\alpha_j}}. \tag{13}$$

这实际上等于为锚点学习一个**局部注意力掩码**，决定相邻顶点如何贡献其位置。图 3 表明：以简化顶点为监督优化权重 $\alpha$ 后，锚点被推向网格的**褶皱拐点与边界**，从而能更好地通过插值它们的变换矩阵来表达变形网格顶点。注意我们是在**整个数据集**上而非只在网格模板上优化锚点位置，因为模板上的一些平滑区域摆完姿态后可能出现新褶皱。另一方面，若用公式 (13) 把变换后锚点 $p_G$ 重写为：

$$p_G = Rp + T = \frac{\sum_{k=1}^{K} e^{\alpha_k} (R v^c_k + T)}{\sum_{j=1}^{K} e^{\alpha_j}}, \tag{14}$$

则锚点的变换学习过程（公式 (10)）可以解读为：按从网格简化学到的权重（重要性）为其相邻网格顶点找一个合适的变换——权重越大，该顶点变换对锚点变换学习的影响越大。

> **图 5**：(a) 训练中只优化网格顶点时 vs (b) AnchorDEF 的锚点变换可视化。

最后，组合公式 (8)、(10)–(12)，模型总体目标函数为：

$$\mathcal{L} = \mathcal{L}_{vert} + \lambda_1 \mathcal{L}_{consis} + \lambda_2 \mathcal{L}_{dir} + \lambda_3 \mathcal{L}_{anch}, \tag{15}$$

其中 $\lambda$ 为损失权重。

## 4. 实验

我们在 **VTO 数据集** [32]（译注：Santesteban 2019，含 dress 与 T-shirt 两类服装）上评估 AnchorDEF。遵循 [26]，采用 RMSE（均方根误差）、Hausdorff 距离和 STED（时空边差异）[37] 评估变形网格质量：前两者度量预测与真值网格的距离，STED 反映两者**动态**差异。我们还与其它 3D 服装变形方法 [26,27] 对比。

**实现细节。** 为捕获运动动态，用 2 层 GRU（隐藏尺寸 256）作为身体运动编码器；两个 2 层 MLP（尺寸 256）分别回归锚点变换与逐顶点位移。每个网格顶点关联的锚点数设为 8（可得到期望的光滑表面），计算锚点位置的近邻顶点数设为 128。对 VTO 的服装序列，随机选 4 个 clip 测试、其余作训练集，训练序列长度为 30。训练与推理的更多细节见补充材料。

### 4.1 消融研究

我们验证 AnchorDEF 各主要组件的有效性。首先评估锚点变换一致性 $\mathcal{L}_{consis}$（公式 10）的重要性。如表 1 所示，保持所学锚点变换的一致性能全面改善所有指标，得益于更好的泛化。图 4 (a) 给出定性对比：没有 $\mathcal{L}_{consis}$ 时模型无法准确预测变形网格形状、丢失部分表面几何细节。

| 方法 | RMSE ↓ | Hausdorff ↓ | STED ↓ |
|---|---|---|---|
| w/o $\mathcal{L}_{consis}$ | 17.21 | 75.01 | 0.0595 |
| w/o $\mathcal{L}_{anch}$ | 16.49 | 74.53 | 0.0526 |
| AnchorDEF（完整） | **16.05** | **74.20** | **0.0493** |

> **表 1**：AnchorDEF 不同组件的定量结果：去掉锚点变换一致性（w/o $\mathcal{L}_{consis}$）、去掉自适应锚点更新（w/o $\mathcal{L}_{anch}$）、完整模型。

为评估自适应锚点更新的影响，表 1 还给出不使用 $\mathcal{L}_{anch}$（公式 12）的结果：只用 k-means 聚类中心作锚点时性能下降。原因在于：与聚类相比，网格简化的监督能提供**关键拓扑信息**（如褶皱、边界），对学习有代表性的锚点变换有用。定性结果见图 4 (b)：无 $\mathcal{L}_{anch}$ 时锚点变换表达不足，无法产出真实的服装变形。![[mcv2-fullcorpus-conclusion]]

图 5 (a)(b) 分别可视化"训练中只优化网格顶点"与 AnchorDEF 变换出的锚点。图 5 (a) 中，没有对锚点变换的一致性约束，许多变换后锚点**不在变形网格表面上**——空间上缺乏清晰物理含义、容易过拟合；相反，AnchorDEF 强制锚点变换落到变形网格表面，一定程度上防止过拟合，测试时泛化更好。此外，如图 5 (b)，得益于按简化网格顶子做的自适应锚点更新，变换后锚点主要分布在网格的**褶皱与边界**，能更好地表达网格顶点（尤其是离锚点远的顶点）的变形。这样更有效地降低了服装变形空间的复杂度、保留更多网格形状细节。

> **图 6**：有无 $p_{tgt} \to p_G$ 方向惩罚 $\mathcal{L}_{dir}$ 的定性对比。

接着考察惩罚 $\mathcal{L}_{dir}$（公式 11）对"目标锚点到变换锚点方向"的影响。如图 6，有 $\mathcal{L}_{dir}$ 时模型能减少服装-身体穿透：通过强制变换后锚点位于目标法向的正方向，受这些锚点影响的服装网格顶点能被有效推到身体表面外侧，同时因沿目标法向移动而保持变形精度。

> **图 7**：不同锚点数量的定性对比。160 个锚点比其它数量产出更精细、更准确的变形。
> **图 8**：AnchorDEF 运动中动态服装变形的例子。第一行为 PBS [25]（译注：ARCSim）生成的真值，第二行为 AnchorDEF 结果。能产出自然真实的衣物动态，如跳跃产生的皱纹、跑步引起的裙摆摆动。

我们进一步评估锚点数量的影响。表 2 给出 20/80/160/320 个锚点的结果。可见用更多锚点引导网格变形在三指标上都更好，因为用刚体锚点变换分解网格变形后，非线性变形可以更容易地预测为 canonical 空间中的小位移，无需再考虑身体姿态引起的大旋转平移（已被锚点变换处理）。但随着锚点继续增多性能**下降**，因为过多变换参数导致测试期预测不稳定。图 7 的定性对比显示：160 个锚点产出更精细准确的变形；20 个只能恢复变形网格轮廓；80 和 320 个无法获得真实变形。

| 锚点数 | 20 | 80 | 160 | 320 |
|---|---|---|---|---|
| RMSE ↓ | 17.78 | 16.82 | **16.05** | 17.52 |
| Hausdorff ↓ | 75.30 | 74.71 | **74.20** | 75.13 |
| STED ↓ | 0.0620 | 0.0539 | **0.0493** | 0.0625 |

> **表 2**：不同锚点数的定量结果。

### 4.2 与相关方法比较

我们在不同类型服装上与 TailorNet [27]、VirtualBones [26] 比较。如表 3，AnchorDEF 在所有情形取得最佳结果。图 9 给出定性对比：TailorNet 能恢复 T 恤上的褶皱，但无法处理与人体拓扑不同的连衣裙变形；通过插值蒙皮混合权重，它缓解了从 SMPL 身体借来的权重不连续，但这使其对宽松服装的大变形只能产出轮廓形状、丢失部分褶皱并出现伪影。VirtualBones 用 SSDR 提取服装虚拟骨骼，能估计连衣裙的大变形，但它倾向在变形网格上**保持裙装模板原有的褶皱**，且无法很好估计 T 恤上由身体运动产生的**新褶皱**。VirtualBones 需要 SSDR 预先提取服装骨架和骨骼混合权重，并在网络训练期间**固定**骨骼位置与权重；此外 SSDR 是在经拉普拉斯平滑处理的**低频**网格序列上进行的（因为它无法处理高频服装变形），因此学到的虚拟骨骼未必最优。相反，通过联合学习锚点的刚体变换、混合权重与位置，AnchorDEF 不仅恢复连衣裙的大变形，还能在 T 恤上产出精细表面细节。更多定性结果见补充材料。

| 方法                | Dress RMSE ↓ | Dress Hausdorff ↓ | Dress STED ↓ | T-shirt RMSE ↓ | T-shirt Hausdorff ↓ | T-shirt STED ↓ |
| ----------------- | ------------ | ----------------- | ------------ | -------------- | ------------------- | -------------- |
| TailorNet [27]    | 22.95        | 76.80             | 0.0757       | 9.90           | 27.02               | 0.0418         |
| VirtualBones [26] | 19.91        | 83.39             | 0.0722       | 10.52          | 31.51               | 0.0452         |
| AnchorDEF         | **16.05**    | **74.20**         | **0.0493**   | **6.25**       | **26.31**           | **0.0262**     |

> **表 3**：不同类型服装上各方法的定量结果。

> **图 9**：TailorNet / VirtualBones / AnchorDEF 与真值的定性对比。借助学到的锚点变换，我们的方法既恢复连衣裙的大变形，也在 T 恤上产出精细表面细节。

## 5. 结论

我们提出基于锚点的变形模型 AnchorDEF，给定身体运动序列预测 3D 服装动画。借助网格表面周围的一组锚点，服装变形可分解为"刚体锚点变换的混合 + canonical 空间中附加的逐顶点位移"。变换一致性学习保证所学锚点变换在空间中的物理含义；以网格简化为监督，我们进一步通过注意力掩码让锚点感知局部网格拓扑，优化锚点位置以学习有代表性的锚点变换。在不同类型服装上的大量实验表明，AnchorDEF 能有效预测紧身与宽松服装在运动中的 3D 服装变形。

**局限。** 与 PBS 方法相比，我们的方法仍缺乏细粒度动态。更好地在网络架构与监督信号中利用时序信息是未来工作的好方向。

**致谢。** 本工作部分受国家自然科学基金（Nos. 62276134）资助。

## 参考文献（保留英文原题，压缩格式）

[1] Baraff & Witkin. Large steps in cloth simulation. SIGGRAPH 1998. · [2] Bertiche et al. Cloth3D: Clothed 3D Humans. ECCV 2020. · [3] Bertiche et al. PBNS: Physically Based Neural Simulator for Unsupervised Garment Pose Space Deformation. arXiv 2020. · [4] Bertiche et al. DeePSD: Automatic Deep Skinning and Pose Space Deformation for 3D Garment Animation. ICCV 2021. · [5] Chen et al. Synthesizing cloth wrinkles by CNN-based geometry image superresolution. CAVW 2018. · [6] Chi & Song. GarmentNets. ICCV 2021. · [7] Cho et al. GRU. arXiv 2014. · [8] De Aguiar et al. Stable spaces for real-time clothing. TOG 2010. · [9] Feng et al. A deformation transformer for real-time cloth animation. TOG 2010. · [10] Garland & Heckbert. Surface simplification using quadric error metrics. SIGGRAPH 1997. · [11] Guan et al. DRAPE: Dressing any person. TOG 2012. · [12] Gundogdu et al. GarNet. ICCV 2019. · [13] Harmon et al. Asynchronous contact mechanics. SIGGRAPH 2009. · [14] Kim et al. Near-exhaustive precomputation of secondary cloth effects. TOG 2013. · [15] Kim & Vendrovsky. DriveShape. SCA 2008. · [16] Kingma & Ba. Adam. ICLR 2015. · [17] Lähner et al. DeepWrinkles. ECCV 2018. · [18] Le & Deng. Smooth Skinning Decomposition with Rigid Bones. TOG 2012. · [19] Li et al. P-Cloth. TOG (SIGGRAPH Asia) 2020. · [20] Li et al. DiffCloth. TOG 2022. · [21] Loper et al. SMPL. TOG 2015. · [22] Ly et al. Projective dynamics with dry frictional contact. TOG 2020. · [23] Ma et al. Neural point-based shape modeling of humans in challenging clothing. 3DV 2022. · [24] Müller et al. Air meshes. TOG 2015. · [25] Narain et al. Adaptive anisotropic remeshing for cloth simulation. TOG 2012. · [26] **Pan et al. Predicting Loose-Fitting Garment Deformations Using Bone-Driven Motion Networks. SIGGRAPH 2022.** · [27] Patel et al. TailorNet. CVPR 2020. · [28] Pfaff et al. Learning Mesh-Based Simulation with Graph Networks. ICLR 2021. · [29] Pons-Moll et al. ClothCap. TOG 2017. · [30] Provot. Collision and self-collision handling in cloth design. 1997. · [31] Pumarola et al. 3DPeople. ICCV 2019. · [32] Santesteban et al. Learning-Based Animation of Clothing for Virtual Try-On. CGF (Eurographics) 2019. · [33] Santesteban et al. SNUG: Self-Supervised Neural Dynamic Garments. CVPR 2022. · [34] Santesteban et al. ULNeF: Untangled Layered Neural Fields. NeurIPS 2022. · [35] Santesteban et al. Self-supervised collision handling via generative 3D garment models. CVPR 2021. · [36] Tang et al. CAMA. CGF (Eurographics) 2016. · [37] Vása & Skala. A perception correlated comparison method for dynamic meshes. TVCG 2010. · [38] Wang et al. Learning an intrinsic garment space for interactive authoring of garment animation. TOG 2019. · [39] Zakharkin et al. Point-based modeling of human clothing. ICCV 2021. · [40] Zhang et al. Dynamic Neural Garments. TOG 2021. · [41] Zhao et al. Learning anchored unsigned distance functions... ICCV 2021.

## 补充材料

### A. 训练与推理

训练采用 Adam 优化器 [16]，初始学习率 1e-3，batch size 8，epoch 数 50；30 epoch 后学习率降至 1e-4。总体目标函数权重 $\lambda_1, \lambda_2, \lambda_3$ 经验地设为 1、0.01、100；变换一致性损失中的权重因子 $\gamma$ 设为 0.1；顶点损失中的拉普拉斯与碰撞系数设为 0.2 与 1。训练开始时不加碰撞项与方向惩罚项，在**最后 10 个 epoch** 加入这两项，并根据新锚点位置**重新计算**锚点-顶点关系。

> **图 10**：估计锚点变换与 canonical 空间逐顶点位移的网络结构。

图 10 展示网络结构。对第 $i$ 帧，身体姿态 $\theta_i$ 与平移 $t_i$ 拼接后输入 GRU 层；锚点旋转平移 $[R_i; T_i]$ 与逐顶点位移 $D_i$ 分别由两个"FC×2"的 MLP 产出，第一个 FC 层后接 PReLU 激活。锚点旋转由网络预测为**欧拉角**再转为旋转矩阵。每层 GRU 初始状态 $h_0$ 从均值 0、标准差 0.1 的正态分布采样；**推理时初始状态设为零**，模型可应用于任意长度的运动序列。

图 11 展示 QEM [10] 简化的网格，用作自适应锚点更新的监督。这些网格保留了表面的内蕴几何结构（褶皱、边界），为学习有代表性的锚点提供关键拓扑信息。

所有对比模型按其论文所述超参训练。TailorNet [27] 使用作者提供的训练代码；VirtualBones [26] 只公开了推理代码，我们按其推理代码**复现**了训练代码。

### B. 更多结果

更多动态变形示例与定性对比见 https://semanticdh.github.io/AnchorDEF 。演示视频中，给定训练中未见的身体运动序列，我们的方法能产出自然真实的衣物动态，且比其它方法更接近真值。

> **图 12**：一些极端姿态下的失败案例，可能出现服装-身体穿透。

图 12 展示失败案例：某些极端姿态下可能出现服装-身体穿透。防止或减少这类穿透是未来的研究方向。
