---
type: translation
title: TailorNet - Predicting Clothing in 3D as a Function of Human Pose, Shape and Garment Style (2020) 中文译本
source: RawSource/articles/TailorNet-2020.pdf
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, pose-space, mixture-of-experts, high-low-frequency, pca-style-space, cvpr-2020, translation]
summary: Patel 等 CVPR 2020 原论文中文学习译本：TailorNet 把服装变形分解为低频（单一 MLP 直接回归 pose+shape+style）与高频（K=20 个 style-shape 专属 pose 模型经窄带宽 RBF 核凸组合）——首个联合建模 pose/shape/style 的数据驱动服装模型；指出"任何组合训练样本的尝试都会抹平高频"，训练数据用插入中间姿态+松弛帧的准静态协议生成（55800 帧，比 PBS 快 1000 倍）。原始英文见 [[TailorNet-2020]]。
---

# TailorNet: Predicting Clothing in 3D as a Function of Human Pose, Shape and Garment Style — 中文译本

> [!info] 译本说明
> 本文件是原论文的中文学习译本，供个人研读使用。公式用 LaTeX（`$...$` / `$$...$$`），结果表用 Markdown 表格，正文忠实传达原文论述，关键术语保留英文对照。原始英文抽取文本见 [[TailorNet-2020|原索引卡]] 所指的 `TailorNet-2020.txt`。
>
> 论文为 arXiv 开放获取（2003.04583v2），版权归 CVPR 2020（IEEE/CVF）所有，本译本按学术学习使用处理。
>
> **抽取工具说明**：`.txt` 用 PyMuPDF 抽取（本目录）。原 PDF 表 1 的 ✓/✗ 符号在抽取中丢失，译文中按论文正文的自述声明重构，个别单元格可能有出入（已注明）。
>
> **关键术语对照**：style=款式（指服装几何本身）；fit=合身度；un-posing=去摆位（反蒙皮）；diffusion flow=扩散流（拉普拉斯平滑）；mixture=混合（专家模型）；prototype=原型（style-shape 对）；bandwidth=核带宽。
>
> **谱系定位（译注）**：本文是 [[Pan2022-VirtualBones|Pan 2022]] 的两个核心设计的直接出处——(1) 高/低频分解（本文用扩散流，Pan 用拉普拉斯平滑）；(2) RBF 窄带宽混合（本文混合 K=20 个 style-shape pose 模型，Pan 混合 K=8 个模拟参数 pivot network）。同时本文的**准静态数据生成协议**（插入线性插值中间姿态+松弛帧）与 [[SkirtQNN-Design]] 方法 2"物理松弛精修"同构。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | TailorNet: Predicting Clothing in 3D as a Function of Human Pose, Shape and Garment Style |
| 作者 | Chaitanya Patel*, Zhouyingcheng Liao*, Gerard Pons-Moll（*同等贡献） |
| 机构 | 马普信息学研究所（MPI Informatics），德国 |
| 发表 | CVPR 2020 |
| arXiv | 2003.04583v2 [cs.CV]，2020-03-15 |
| 模型/数据 | https://virtualhumans.mpi-inf.mpg.de/tailornet/ （55800 帧数据集公开） |
| 模拟工具 | Marvelous Designer（商业布料设计与模拟软件） |

> **图 1**：TailorNet 从身体形状、姿态与服装款式预测带细节褶皱的 3D 服装变形。左起：前两个 avatar 是同一形状上两种款式；随后两个是另一形状上的同两款式；最后两个展示方法适用于不同服装类别。

## 摘要

本文提出 **TailorNet**——一个把 3D 服装变形预测为**三个因子（姿态 pose、形状 shape、款式 style 即服装几何）的函数**的神经模型，同时保留皱纹细节。这超越了先前或局限于单一款式与形状、或虽泛化到不同形状但结果平滑（且仍限单一款式）的模型。我们的假设是：**（哪怕是非线性的）训练样本组合会把细皱纹等高频成分抹平**，这使三个因子的联合学习变得困难。本文技术的核心是**把变形分解为高频与低频成分**：低频成分用 MLP 从 pose、shape、style 参数直接预测；高频成分用**一组 shape-style 专属的 pose 模型混合**预测。混合权重用**窄带宽核**计算，保证只组合高频模式相似的预测。款式变化通过在 canonical 姿态下计算一个满足物理约束（不互穿、垂坠在身体上）的**变形子空间**获得。TailorNet 产出的 3D 服装保留了其学习来源（PBS 物理模拟）的皱纹，同时**快 1000 倍以上**；与经典 PBS 相比，TailorNet 易用且**完全可微**——这对计算机视觉与学习算法至关重要。大量实验表明 TailorNet 比先前工作更真实，甚至在 AMASS [34] 数据集的序列上产出时序连贯的变形——尽管只用另一个数据集的静态姿态训练过。

## 1. 引言

给数字人穿衣服做动画在 3D 内容生产、游戏、娱乐与虚拟试穿中应用众多。主流方法仍是物理模拟（PBS）。但典型 PBS 管线需要在 2D 用版型设计服装、手动放置到数字角色上、调参到满意——费力、耗时、需要专家知识。此外高质量 PBS 计算昂贵、实现与控制复杂、且不可微。对计算机视觉任务而言，穿衣人的生成模型需要**可微、易部署、易集成进 CNN**、可拟合图像与视频数据。

为简化动画，若干工作从离线 PBS 学高效的近似模型。至少三个因子影响服装变形：身体姿态、形状与款式（style 指服装几何）。现有方法或对固定形状建模姿态变形 [10,29]、或对固定款式建模形状与姿态 [46,18]、或对固定姿态建模款式 [58]。这些方法启发了我们，但它们**没有联合建模**三个因子的影响，尽管三者相互交织：不同服装作为姿态与形状的函数变形方式不同；服装专属模型 [46,29,17,10]（单款式）用途有限。此外现有的 pose+shape 联合模型 [17,8,18] 常产出**过度平滑**的结果（即便固定款式），且未公开。因此现有方法没有一个落地成可用于解决计算机视觉与图形学问题的模型。

缺失的是一个**统一模型**：能生成不同服装款式、在任何身体形状与任何姿态上做动画、同时保留皱纹细节。为此我们提出 TailorNet——从物理模拟学到的**神经网络混合体**，把服装变形分解为 style、shape、pose，有效近似物理服装变形、允许直观控制合成动画。与身体的 SMPL [32] 同一精神，TailorNet 把变形学为 **canonical 姿态下服装模板的位移**，关节运动由蒙皮驱动。TailorNet 可接收真实服装作输入，或从头生成，再 draped 到任意形状与姿态的 SMPL 身体上。与 [17] 不同，我们的模型预测服装**在现实中的合身效果**：中号服装在大身体上预测为紧、在瘦身体上预测为松。

学习 TailorNet 需要解决若干技术挑战。为在静态姿态生成不同款式，我们用公开的真实静态服装数字衣柜 [6] 计算 **PCA 子空间**。为在满足服装-人体物理约束的同时生成更多款式变化，我们从 PCA 子空间采样、对每个样本跑 PBS、再重算 PCA，得到静态款式子空间。该子空间的样本在静态姿态下产生袖长、尺码与合身度的变化。为学 pose 与 shape 的变形函数，我们用 PBS 在 SMPL [32] 身体上动画化服装（真实款或款式子空间样本），覆盖静态 SMPL 姿态与不同形状，生成一个**半真实数据集**。

我们的**第一个观察**：对固定的款式（服装实例）与身体形状，**用 MLP 把高频服装变形预测为姿态的函数是可行的**——出乎意料地，实验表明对这个任务，简单 MLP 的表现与 Graph Neural Networks [31,28] 和 UV 空间图像解码器 [29] 相当甚至更好。与之尖锐对照的是，直接把变形预测为 style+shape+pose 的联合函数会得到**过度平滑、不真实**的结果。我们假设：**任何组合训练样本的尝试都会抹平高频成分**——这解释了为什么以前的模型 [46,17,18]（哪怕单一款式）缺乏细尺度皱纹与折叠。

这些关键观察催生了 TailorNet 的设计：**低频几何用简单 MLP 预测**；**高频几何用一组 style-shape 专属的混合模型预测**——每个专属模型是一个"pose→变形"的 MLP，混合权重由评估 style-shape 相似度的核计算。**很窄的核带宽**防止抹平细尺度皱纹。大量实验表明模型对全新姿态泛化良好、预测依赖身体形状的合身效果、保留皱纹细节、能产出服装类别的款式变化（如 T 恤的不同尺码/袖长/合身类型），且**实现与控制简单**。总结主要贡献：

- **首个**联合建模服装 style、pose、shape 变化的模型：简单、易部署、完全可微、易于集成深度学习。
- 一个简单有效的**网格变形高/低频分解**，与混合模型耦合，得以保留高频皱纹。
- 对不同方法（MLP、Graph NN、UV 空间图像翻译）预测姿态相关服装变形的**对比**。
- 为促进后续研究，公开**数据集**：20 件对齐的真实静态服装，1782 个姿态、9 种身体形状模拟，共 55800 帧。

多组实验表明模型泛化到 AMASS 数据集的全新姿态（尽管训练没用 AMASS），在比以前的单款式模型 [46,18] 更细节化的同时产出 pose/shape/style 的变化。此外，尽管只用静态姿态训练，TailorNet 产出平滑连续的动画。

## 2. 相关工作

服装动画有两大路线：物理模拟（PBS）与从离线 PBS 或真实捕获学到的高效数据驱动模型。

**基于物理的模拟（PBS）。** 超真实动画需要模拟数百万三角形 [53,47,22]，计算昂贵。许多工作聚焦效率：给低分辨率模拟加皱纹 [15,25,26,55]、用更简单的质点-弹簧模型 [43] 与位置基动力学（PBD）[38,37] 用速度换精度。调模拟参数是可能花数周的苦差，因此有作者尝试机械式推断物理参数 [35,57]、从多视角视频 [45,62,50] 或人类感知判断 [49] 推断——但只在受控场景有效、仍需跑 PBS。PBS 管线通常需要 2D 设计服装、放码、适配 3D 身体、调参——即便对训练有素的专家也要数小时到数周。

**数据驱动的布料模型。** 一条路是图像 [6,5,3,4,19]、动态扫描 [41,39] 或 RGBD [52,51] 捕获真人服装并重定向到新形状——但局限于把同一动作重播到新形状上。从真实捕获学姿态相关模型 [61,29,33] 是激动人心的路线，但准确捕获足够的真实数据仍是重大挑战，这些模型只展示了向训练数据相似动作的泛化。另一条路是用 PBS 生成数据、学高效数据驱动模型。早期方法依赖线性自回归或最近邻搜索预测姿态 [10,26,56]，或姿态与形状相关的服装变形 [17]。近期方法基于深度学习，按建模因子（style 或 pose+shape）、数据表示与架构区分：style 变化从用户草图的 VAE 预测 [58]（姿态固定）；单款式的 pose+shape 变化用 MLP 与 RNN 回归 [46,61] 或处理身体与服装的 Graph-NN [18]；姿态效果用 UV 空间的法线图 [29] 或位移图 [23] 预测。这些模型 [10,18,17] 倾向产出过度平滑的结果（[29] 除外，但其只训练单件服装与单个主体，且如前所述未演示对野外动作（如 CMU [11]）的泛化）。由于"什么表示与学习模型最适合该任务"没有共识，我们对固定 style 与 shape 对比了表示与架构，发现 **MLP 与更复杂的 Graph-NN 或 UV 空间图像翻译表现相当**。虽然受前人启发，但没有一个能像我们一样联合建模 style、pose、shape。

**基于像素的模型。** 3D 建模的替代是检索并扭曲图像与视频 [60,9]。近期方法用深度学习生成像素 [20,14,13,63,65,67,21,44,54,66]，常以 2D 扭曲或平滑 3D 模型 [30,64,59,48] 引导，或学从布料图像向 3D 模型转移纹理 [36]。它们（至多）在受控场景产出照片级真实图像，但**不捕获皱纹的 3D 形状**，难以控制运动、视角、光照、形状与款式。

## 3. 与 SMPL 对齐的服装模型

基础服装模板如 [41,6] 那样与 SMPL [32] 对齐。SMPL 把人体 $M(\cdot)$ 表示为 pose（$\theta$）与 shape（$\beta$）的参数化函数：

$$M(\beta, \theta) = W(T(\beta, \theta),\ J(\beta),\ \theta,\ \mathcal{W}) \tag{1}$$
$$T(\beta, \theta) = \bar{T} + B_s(\beta) + B_p(\theta) \tag{2}$$

由线性函数 $T(\beta,\theta)$（向 T-pose 基础网格顶点 $\bar{T} \in \mathbb{R}^{n\times3}$ 加位移）与学到的蒙皮 $W(\cdot)$ 组成：$B_p(\cdot)$ 加姿态相关变形，$B_s(\cdot)$ 加形状相关变形，$\mathcal{W}$ 是骨架 $J(\cdot)$ 的混合权重。

我们把服装模板拓扑取为 SMPL 模板的**子网格**（顶点 $T_G \in \mathbb{R}^{m\times3}$，模板顶点 $\bar{T}$）。形式上，指示矩阵 $I \in \mathbb{Z}^{m\times n}$：若服装顶点 $i$ 关联身体顶点 $j$ 则 $I_{i,j}=1$。 draped 在 $M(\beta,\theta)$ 上的特定服装款式在 0-pose 下编码为相对未摆位身体形状 $T(\theta,\beta)$ 的位移 $D$。因为服装与底层 SMPL 身体关联，先前工作 [6,41] 用其关联 SMPL 身体顶点的函数变形每个服装顶点。给定款式 $D$、形状 $\beta$、姿态 $\theta$，先用未摆位 SMPL 函数：

$$T_G(\beta, \theta, D) = I\, T(\beta, \theta) + D \tag{3}$$

再套公式 (1) 的 SMPL 蒙皮：

$$G(\beta, \theta, D) = W(T_G(\beta, \theta, D),\ J(\beta),\ \theta,\ \mathcal{W}) \tag{4}$$

由于 $D$ 固定，这**假设服装与身体以相同方式变形**——实用但显然过度简化。像先前工作一样，我们也把变形分解为非刚体成分（公式 3）与关节成分（公式 4），但**与前人不同，我们把非刚体变形 $D$ 学成 pose、style、shape 的函数**——即学到真实的服装变化，下一节详述。

## 4. 方法

本节描述我们把服装分解为非刚体变形（由 pose/shape/style 引起）与关节变形的方案——后者我们称为 **un-posing**（去摆位，§4.1）。模型的第一个组件是生成 A-pose 款式变化的**款式子空间**（§4.2）。如引言所述，固定 shape-style 的 pose 模型（§4.3）**能保留高频**，但组合不同 style 与 shape 的模型产出过度平滑的结果。因此技术核心是：**低频用简单 MLP 预测，高频用混合模型预测**（§4.2–4.4）。总览见图 2。

> **图 2**：模型总览。给定姿态 $\theta$、形状 $\beta$ 上的身体与款式 $\gamma$：低频变形用单一模型预测；$K$ 个原型 shape-style 对的姿态相关高频变形分别计算，用 RBF 核混合得到最终高频；低频+高频相加得到未摆位服装，再经标准蒙皮摆位得到服装。

| 方法 | 静/动 | Pose 变化 | Shape 变化 | Style 变化 | 模型公开 | 数据公开 |
|---|---|---|---|---|---|---|
| Santesteban et al. [46] | 动态 | ✓ | ✓ | ✗ | ✗ | ✗ |
| Wang et al. [58] | 静态 | ✗ | ✗ | ✓ | ✓* | ✗ |
| DeepWrinkles [29] | 动态 | ✓ | ✗ | ✗ | ✗ | ✗ |
| DRAPE [17] | 动态 | ✓ | ✓ | ✗ | ✗ | ✗ |
| GarNet [18] | 静态 | ✓ | ✓ | ✗ | ✗ | ✓* |
| Ours | 静态 | ✓ | ✓ | ✓ | ✓ | ✓ |

> **表 1**：与其它工作对比。我们首个把服装变形建模为 pose、shape、style 的函数，并公开模型与数据集。（译注：原 PDF 的 ✓/✗ 在文本抽取中丢失，本表按论文正文声明重构，标 \* 的两格不确定，请以原 PDF 为准。）

### 4.1. 去摆位（Un-posing）服装变形

给定对特定 pose $\theta$ 与 shape $\beta$ 模拟出的一组服装 $\mathcal{G}$，先用 un-posing 把非刚体变形与关节运动解耦——即对蒙皮函数 $W(\cdot)$ 求逆：

$$D = W^{-1}(\mathcal{G}, J(\beta), \theta, \mathcal{W}) - I\, T(\beta, \theta) \tag{5}$$

再减去 canonical 姿态的身体形状，得到作为相对身体位移的**未摆位非刚体变形 $D$**。由于关节 $J(\cdot)$ 已知，公式 (5) 的 $W^{-1}(\cdot)$ 对每个顶点计算 $\hat{v}_j = (\sum_k \mathcal{W}_{k,j} H_k(\theta, J(\beta)))^{-1} v_j$，其中 $H_k(\theta, J(\beta)) \in SE(3)$ 是部位变换矩阵，$\hat{v}_j \in \mathbb{R}^3$ 是去摆位顶点。未摆位空间的非刚体变形 $D$ 受身体姿态、形状与服装款式（尺码、袖长、合身度）影响。因此我们提出把变形学为 shape $\beta$、pose $\theta$、style $\gamma$ 的函数：$D(\beta, \theta, \gamma): \mathbb{R}^{|\theta|} \times \mathbb{R}^{|\beta|} \times \mathbb{R}^{|\gamma|} \mapsto \mathbb{R}^{m\times3}$。模型应真实、易控且可微。

### 4.2. 款式的参数化模型生成

我们在 A-pose 用公开 3D 服装 [6] 计算 PCA 子空间来生成款式变化 $\gamma$。尽管 Bhatnagar 等 [6] 的方法能把 3D 服装迁移到 canonical 姿态与形状（$\{G(\theta_0, \beta_0, D_i)\}$），物理约束可能被违反——服装有时轻微飘在身体上方、皱纹与 A-pose 不符。为在满足物理约束的同时生成款式变化，我们**交替**从 PCA 空间采样与对样本跑 PBS。交替 PBS 与 PCA 两轮已得到好结果，获得静态 A-pose 的款式变化子空间——PCA 系数即 $\gamma$。图 3 展示 T 恤的参数化款式空间。然而这只是固定 A-pose 与形状下的模型；不同款式作为 pose 与 shape 的函数变形不同。§4.3 讲我们的 style-shape 专属 pose 模型，§4.4 讲面向多样形状与款式的联合混合模型。

> **图 3**：T 恤款式空间总览。第一主成分（上）改变整体尺码，第二主成分（中）改变袖长。从该空间采样生成大范围 T 恤款式（下）。前两行从左到右对应成分取 $-1.5\sigma, 0, 1.5\sigma$。

### 4.3. 单一 Style-Shape 模型

对固定服装款式 $\gamma$ 与固定身体形状 $\beta$ 的对（记 $\phi = (\gamma, \beta) \in \mathcal{X}$），取按公式 (5) 算出的"身体姿态-服装位移"配对模拟数据 $\{(\theta_i, D^{(\phi)}_i)\}$，训练模型 $D_\phi: \mathbb{R}^{|\theta|} \to \mathbb{R}^{m\times3}$ 预测姿态相关位移。具体用多层感知机（MLP）训练，最小化预测位移 $D_\phi(\theta_i)$ 与真值 $D^{(\phi)}_i$ 的 **L1 损失**。我们观察到 $D_\phi$ 预测出合理的合身度与细尺度皱纹、对未见姿态泛化良好——但该模型专属特定 shape 与 style。

### 4.4. TailorNet

为简单起见， tempting 的做法是把所有非刚体服装变形直接回归为 pose、shape、style 的联合函数，即直接用 MLP 学 $D(\theta, \phi): \mathbb{R}^{|\theta|} \times \mathcal{X} \to \mathbb{R}^{m\times3}$。然而实验表明：**定量上准确、定性上过度平滑缺乏真实感**。我们假设：**任何组合大量高频（细皱纹）各异样本几何的尝试都会抹平细节**——这可能是相关工作 [46,18,17]（哪怕单款式）结果平滑的潜在解释。

我们的想法是把未摆位空间 $\hat{\mathcal{G}} = W^{-1}(\mathcal{G}, J(\beta), \theta, \mathcal{W})$ 中的服装网格顶点用**扩散流**分解为平滑低频形状 $\hat{\mathcal{G}}_{LF}$ 与高频形状 $\hat{\mathcal{G}}_{HF}$。设 $f(x,t): \mathcal{G} \mapsto \mathbb{R}$ 是服装表面上的函数，用扩散方程平滑：

$$\frac{\partial f(x,t)}{\partial t} = \lambda \triangle f(x,t) \tag{6}$$

即函数随时间变化量为标量扩散系数 $\lambda$ 乘以其空间拉普拉斯 $\triangle f$。为平滑网格几何，把扩散方程应用于顶点坐标 $g_i \in \hat{\mathcal{G}}$（解释为表面上的离散化函数）：

$$g_i = g_i + \lambda \triangle g_i \tag{7}$$

其中 $\triangle g_i$ 是作用在顶点 $g_i$ 的离散 Laplace-Beltrami 算子，$\lambda$ 与迭代次数控制平滑程度。公式 (7) 也即**拉普拉斯平滑** [12,7]。我们用 $\lambda = 0.15$、80 次迭代，得到平滑低频网格 $\hat{\mathcal{G}}_{LF}$ 与高频残差 $\hat{\mathcal{G}}_{HF} = \hat{\mathcal{G}} - \hat{\mathcal{G}}_{LF}$。然后如公式 (5) 减去身体形状 $I\,T(\beta,\theta)$，**分开预测**位移成分：

$$D(\theta, \phi) = D_{LF}(\theta, \phi) + \sum_{k=1}^{K} \Psi(\phi, \phi_k)\, D^{HF}_{\phi,k}(\theta) \tag{8}$$

其中低频成分用 MLP $D_{LF}(\theta, \phi)$ 预测（平滑但准确）；高频成分用一组 **style-shape $\phi=(\gamma,\beta)$ 专属的 pose 模型** $D^{HF}_\phi(\theta)$ 的**混合**预测——如实验所示，style-shape 专属高频模型**保留细节**。我们用专属模型的**凸组合**泛化到原型 $D^{HF}_{\phi_k}(\theta)$ 之外的新 shape-style。混合权重用**窄带宽核** $\Psi(\phi_1, \phi_2): \mathcal{X} \times \mathcal{X} \mapsto \mathbb{R}$ 计算，只组合**相似皱纹模式**：

$$\Psi(\phi, \phi_k) = \exp\Big(-\,\mathrm{dist}\big(g(\phi),\, g(\phi_k)\big)\, \big/ \,\sigma_x\Big) \tag{9}$$

其中 $g(\phi): \mathcal{X} \mapsto \mathbb{R}^{m\times3}$ 是一个浅层 MLP，把 style-shape 映射到 canonical A-pose 的服装位移 $D$。理想情况下核应直接度量 style 与 shape 的相似度，但那要求为每种可能的 pose-shape-style 组合模拟训练数据，低效且耗资源。我们的**关键简化假设**是：**两件服装在两个人身上，若它们相对各自身体的位移 $g(\phi)=D$ 相似，则变形方式相似**——这度量的是**合身度相似度**。虽是近似，实践中工作得很好。

带宽 $\sigma_x$ 是模型的自由参数，允许生成不同程度的高频细节。我们保持 $\sigma_x$ 较小（只组合邻近样本）获得定性上好的结果。我们对输出做后处理以移除服装与身体的相交。

**选择 K 个 Style-Shape 原型。** 对每个选定的 style-shape 对，需要模拟大量姿态——耗时。因此我们要选 $K$ 个原型 style-shape 对 $\phi$，使其覆盖 canonical A-pose 下静态位移 $D(\beta, \theta_0, \gamma)$ 的空间。具体地：选择 $K$ 个 style-shape 对，使任何其它 style-shape 都能用原型的凸组合以最小误差近似——这是一个不保证全局最优的非凸问题。迭代法如 K-SVD [2] 可能有更好覆盖，但这里用简单有效的**贪心法**：取一个数据集 $\mathcal{X}$ 件不同款式 $\gamma_i$ draped 在不同形状 $\beta_i$ 上的 canonical 姿态服装 $\{D(\theta_0, \phi)\}_{i=1...|\mathcal{X}|}$。从 9 个身体形状、同一公共款式的池子出发，尝试把其余每个 style-shape 对拟合为池子的凸组合；然后取**近似误差最高**的 style-shape 加入池子；重复直到得到 $K$ 个 style-shape 对。我们发现 $K \geqslant 20$ 时所有 style-shape 都能被很好近似，故取 $K=20$。

## 5. 数据集

我们从"参数-模拟服装"配对数据学习模型。用商业 3D 布料设计与模拟工具 **Marvelous Designer** [1] 模拟。服装不手动设计摆放，而用公开的真实数字衣柜 [6]（含款式、形状、姿态变化）。以 T 恤为例描述数据生成，方法对所有服装相同。

**5.1. 生成款式的服装。** 先把数字衣柜 [6] 的 3D 服装重定向 [41] 到 canonical 形状 $\beta_0$，缓慢模拟到 canonical 姿态 $\theta_0$，得到 43 件服装的 $\{G(\beta_0, \theta_0, D_i)\}$ 学款式空间（§4.2）。但学到的款式空间可能因数据变化有限而不直观，且因配准过程含不规则模式与扭曲。于是从该 PCA 模型采样 **300 个款式**、再模拟，得到扭曲更少、更大的数据集。**PCA 与模拟交替 2 轮**后，生成一致且有意义的款式变化。发现前 2 个 PCA 成分足以表示 $\gamma$ 参数（见图 3）。

**5.2. Shape、Pose 与 Style 变化。** 手动选 9 个形状：对前两个 shape 成分 $\beta$ 各在 4 个等距间隔采样（其余成分置零）得 $4 \times 2 = 8$ 个形状，加 canonical 零形状 $\beta_0$。类似地选 25 个款式——采样前两个 style 成分 $\gamma$。在 canonical 姿态模拟所有 shape×style 组合，得 $25 \times 9 = 225$ 个实例。用 §4.4 的方法从 225 个 style-shape 对中选 **20 个原型**作训练 style-shape 对；再随机选 20 个 style-shape 对作测试。

**5.3. 模拟细节。** 姿态变化用 **1782 个静态 SMPL 姿态**（含大量极端姿态）。对给定 style-shape $\phi_k$ 与姿态集 $\{\theta_i\}$，把它们模拟在**一个序列**里：身体从 canonical 姿态出发，贪心转移到最近的姿态直到遍历全部。**为避免动态效果（本文只关心准静态变形），我们插入线性插值的中间姿态、并让服装松弛数帧**。PBS 有时会失败，因此从数据集中**移除自穿透的帧**。训练 style-shape 对模拟含插值姿态的 SMPL 姿态；测试 style-shape 对模拟一部分 SMPL 姿态（已见与未见姿态混合）。最终得到 4 个切分：(1) train-train：20 个训练 style-shape 对、各 2600 个训练姿态；(2) train-test：20 个训练对、各 179 个未见姿态；(3) test-train：20 个测试对、各 35 个训练姿态；(4) test-test：20 个测试对、各 131 个未见姿态。

## 6. 实验

我们在 T 恤上定量定性评估并与基线及先前工作对比。基线与 TailorNet 用多个 MLP——每个含 2 个隐藏层（ReLU）+ dropout。超参先调基线至最优，再保持恒定训练所有其它 MLP（细节见补充材料）。

**6.1. 单一 Style-Shape 模型的结果。** 我们在 $K$ 个 style-shape 对上分别训练单一模型 $D^{HF}_\phi(\theta)$。各模型的逐顶点平均测试误差随特定 style-shape 的难度而变：**所有 $K$ 个模型平均误差 8.04 mm**，最差 14.50 mm（宽松合身），最好 6.56 mm（紧配合身）。工作早期我们对单一 style-shape 模型实验了不同 3D 网格学习模态：类似 [29] 的 UV 图、服装网格上的图卷积（Graph CNN）。UV 图学习训练一个"输入姿态参数→服装位移 UV 图"的解码器；Graph CNN [27] 把输入 $\theta$ 放在图的每个节点上。表 2 报告两个 style-shape 对的结果，显示**简单 MLP 足以学会固定 style-shape 的姿态相关变形**。

| Style-shape | MLP | UV Decoder | Graph CNN |
|---|---|---|---|
| 宽松（Loose-fit） | **14.5** | 15.9 | 16.1 |
| 紧身（Tight-fit） | **10.1** | 11.4 | 11.7 |

> **表 2**：MLP / UV Decoder / Graph CNN 在两个 style-shape 上姿态相关预测的逐顶点平均误差（mm）。

> **图 4**：未见 pose/shape/style 的结果。[46]（左）与我们的混合模型（右）。注意我们的混合模型保留了更有意义的折叠与细节。

**6.2. TailorNet 的结果。** 基线 $f_{BL}(\beta,\theta,\gamma): \mathbb{R}^{|\theta|+|\beta|+|\gamma|} \to \mathbb{R}^{m\times3}$ 实现为直接回归位移的 MLP。表 3 显示混合模型在 3 个测试切分上**小幅**优于基线。

| 切分 | style-shape 集 | pose 集 | 基线 | 混合模型 |
|---|---|---|---|---|
| 2 | train | test | 10.6 | **10.2** |
| 3 | test | train | 11.7 | **11.4** |
| 4 | test | test | 11.6 | **11.4** |

> **表 3**：§5.3 的 3 个测试切分上的逐顶点平均误差（mm）。混合模型定量上略好于基线，**定性上显著更好**。

定性上，TailorNet 胜过基线与先前工作：图 4 对比 [46] 与混合模型；图 5 对比 TailorNet 与同数据训练的基线——**基线把服装上的细节抹平了，TailorNet 保留了与 PBS 真值一样多的皱纹**。为验证高/低频分解的必要性，我们考察一个"各 MLP 直接预测 style-shape 相关位移、**不做分解**"的混合模型：图 6 显示它能近似整体形状但**丢失细皱纹**——分解让我们既预测合身度又保留细节，并对高频细节提供直观控制。

> **图 5**：基线（左）抹平服装细节；TailorNet（中）保留与 PBS 真值（右）一样多的皱纹。
> **图 6**：（左）不分解、直接混合原型输出会抹平细节；（右）高低频分解让我们预测出带细节的良好合身度。

**AMASS 序列 [34]：** TailorNet 泛化到训练分布之外的完全不同姿态（部分见图 7），且**尽管不建模动态，结果是时序连贯的**（见补充视频）。

> **图 7**：左：TailorNet 对多件服装与完全未见姿态的预测；右：带真实纹理的预测。

**6.3. 多件服装。** 为展示泛化性，我们对 Shirt、Pants、Skirt 三件服装分别训练。**由于裙（skirt）不跟随模板身体网格的拓扑，我们把裙模板挂到 SMPL 的根关节 [41]**。对每件服装，用公共款式在 9 个形状上模拟数据集并训练模型。图 7 展示详细预测。由于基础服装来自真实数字衣柜 [6]，我们还能把真实扫描的纹理迁移到预测上。

**6.4. 运行时性能。** 用 PyTorch [40] 实现。在游戏本（NVIDIA GTX 1060 + Intel i7）上每帧 **1–2 ms**，比训练来源 PBS **快 1000 倍**；纯 CPU 也比 PBS 快 100 倍。

## 7. 讨论与结论

TailorNet 是首个 pose、shape、style 的数据驱动服装模型。实验表明简单 MLP 以约 11 mm 精度近似服装变形，与更复杂的 graph-CNN 或 UV 空间位移预测相当；但与现有方法共享同一局限：**当用不同身体形状（本文还有不同款式）训练时，结果过度平滑、缺乏真实感**。为此我们提出分离预测低频与高频的模型。多组实验表明**窄带宽混合模型预测高频**比现有模型保留显著更多细节——尽管任务更难（把姿态相关变形联合建模为 shape 与 style 的函数）。未来工作：拟合扫描/图像/视频、自监督视频精化、真实 3D 布料捕获 [41]、让模型对布料物理属性与人体软组织 [42] 敏感。TailorNet 比 PBS 快 1000 倍，无需手工即可直观控制 style/shape/pose，且可微——这对计算机视觉与学习应用至关重要。我们将公开 TailorNet 与数据集，持续增加服装类别与款式。TailorNet 填补了 SMPL、ADAM [32,24] 等身体模型缺失的关键组件：真实服装——虚拟人动画、解释扫描/图像/视频中穿衣人复杂性的必需品。

**致谢。** 本工作部分受 DFG（德国科学基金会）- 409792180（Emmy Noether 计划，Real Virtual Humans 项目）与 Google 教研奖资助。

## 参考文献（保留英文原题，压缩格式）

[1] Marvelous Designer（商业布料模拟）. · [2] Aharon et al. K-SVD. IEEE TSP 2006. · [3–5] Alldieck et al. RGB 人体重建（HMD-Net / Video based / Tex2shape）. CVPR/ICCV 2018-19. · [6] **Bhatnagar et al. Multi-Garment Net: Learning to dress 3D people from images. ICCV 2019**（数字衣柜）. · [7] Botsch et al. Polygon Mesh Processing. 2010. · [8] Casas et al. 4D video textures. CGF 2014. · [9] Casas et al. 4D Video Textures. EG 2014. · [10] de Aguiar et al. Stable spaces for real-time clothing. TOG 2010. · [11] CMU multimodal activity database. 2009. · [12] Desbrun et al. Implicit fairing of irregular meshes using diffusion and curvature flow. SIGGRAPH 1999. · [13,14] Dong et al. 多姿态 VITON / FW-GAN. ICCV 2019. · [15] Gillette et al. Real-time dynamic wrinkling of coarse cloth. SCA 2015. · [16] Goldenthal et al. Efficient simulation of inextensible cloth. SIGGRAPH 2007. · [17] **Guan et al. DRAPE: Dressing any person. TOG 2012**. · [18] **Gundogdu et al. GarNet: A two-stream network for fast and accurate 3D cloth draping. 2018**. · [19] Habermann et al. LiveCap. TOG 2019. · [20] Han et al. VITON. CVPR 2018. · [21] Hsiao et al. Fashion++. ICCV 2019. · [22] Jiang, Gast, Teran. Anisotropic elastoplasticity. TOG 2017. · [23] Jin et al. A pixel-based framework for data-driven clothing. 2018. · [24] Joo et al. Total Capture (Adam). CVPR 2018. · [25] Kavan et al. Physics-inspired upsampling for cloth simulation in games. TOG 2011. · [26] **Kim et al. Near-exhaustive precomputation of secondary cloth effects. TOG 2013**. · [27] Kipf & Welling. Semi-supervised classification with GCN. ICLR 2017. · [28] Kolotouros et al. Convolutional mesh regression. CVPR 2019. · [29] **Lähner et al. DeepWrinkles. ECCV 2018**. · [30] Lassner et al. A generative model of people in clothing. ICCV 2017. · [31] Litany et al. Deformable shape completion with graph convolutional autoencoders. CVPR 2018. · [32] **Loper et al. SMPL. TOG 2015**. · [33] Ma et al. Learning to dress 3D people in generative clothing. CVPR 2020. · [34] Mahmood et al. AMASS. ICCV 2019. · [35] Miguel et al. Data-driven estimation of cloth simulation models. CGF 2012. · [36] Mir et al. Learning to transfer texture from clothing images to 3D humans. CVPR 2020. · [37,38] Müller. Hierarchical PBD 2008 / **PBD 2007**. · [39] Neophytou & Hilton. Layered body-garment deformation. 3DV 2014. · [40] Paszke et al. PyTorch. 2017. · [41] **Pons-Moll et al. ClothCap: Seamless 4D clothing capture and retargeting. TOG 2017**. · [42] Pons-Moll et al. Dyna. TOG 2015. · [43] Provot. Deformation constraints in a mass-spring model. 1995. · [44] Raj et al. SwapNet. ECCV 2018. · [45] Rosenhahn et al. Articulated tracking with clothing model. 2007. · [46] **Santesteban et al. Learning-based animation of clothing for virtual try-on. CGF (EG) 2019**（VTO）. · [47] Selle et al. Robust high-resolution cloth. TVCG 2009. · [48] Shysheya et al. Textured neural avatars. CVPR 2019. · [49] Sigal et al. A perceptual control space for garment simulation. TOG 2015. · [50] Stoll et al. Video-based reconstruction of animatable characters. TOG 2010. · [51,52] Tao et al. DoubleFusion / SimulCap. CVPR 2018/2019. · [53] Terzopoulos et al. Elastically deformable models. 1987. · [54] Wang et al. Characteristic-preserving try-on. ECCV 2018. · [55] Wang et al. Example-based wrinkle synthesis. TOG 2010. · [56] Wang et al. Example-based wrinkle synthesis（重复收录）. · [57] Wang et al. Data-driven elastic models for cloth. TOG 2011. · [58] **Wang et al. Learning a shared shape space for multimodal garment design. TOG 2018**. · [59] Weng et al. Photo wake-up. CVPR 2019. · [60] Xu et al. Video-based characters. TOG 2011. · [61] Yang et al. Analyzing clothing layer deformation statistics. ECCV 2018. · [62] Yang et al. Physics-inspired garment recovery. TOG 2018. · [63] Yu et al. VTNFP. ICCV 2019. · [64] Zanfir et al. Human appearance transfer. CVPR 2018. · [65–67] Zhu et al. / Zhao et al. Be your own prada. ICCV 2017.
