---
type: survey
title: 骨骼驱动布料引文图谱 — 从 Pan2022 与 SSDR 出发的扩展文献
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, citation-map, survey, reading-list]
summary: 以 Semantic Scholar 引文图谱调研：Pan2022 的上游引文（PSD/example-skinning/SSDR/布料数据驱动四条源头线）、Pan2022 的被引（2023-2026 骨骼/锚点/蒙皮驱动布料直系谱系精选）、SSDR 的被引（蒙皮分解→辅助骨骼→压缩→弹簧二级动画工程线）；并对每条线标注与 SkirtQNN 项目的关联和优先级。
---

# 骨骼驱动布料引文图谱 — 从 Pan2022 与 SSDR 出发的扩展文献

> 调研方法（2026-10-09）：Semantic Scholar Graph API 拉取 [[Pan2022-VirtualBones|Pan 2022]] 的引用列表（references）与被引列表（citations），以及 SSDR（Le & Deng 2012, DOI 10.1145/2366145.2366218）的被引列表；arXiv API 核实 AnchorDEF / UNIC 等摘要。SSDR 自身的引文列表被出版商在 S2 上隐去，其上游经 SSDR 原文叙述与 Pan2022 引文交叉确认。标注「待读」= 仅有题录未读原文。

## 一、Pan2022 的上游：四条源头线

### 1a. 骨架驱动变形的老根（PSD / example skinning / 物理二级动态）

| 论文 | 出处 | 对本谱系的意义 |
|---|---|---|
| Lewis et al., *Pose Space Deformation: A Unified Approach to Shape Interpolation and Skeleton-Driven Deformation* | SIGGRAPH 2000 | PSD 鼻祖：骨架驱动之上按姿态插值修正。后来一切「pose-conditioned 修正量 $\Delta\chi$」设计（DeePSD、PBNS、[[PhySkin2026\|PhySkin]] 两层 LBS）的源头 |
| Capell et al., *Interactive Skeleton-Driven Dynamic Deformations* | TOG 2002 | **最早的「骨骼定形 + 物理层补动态」混合管线**（分层变形 + 动力学）——SkirtQNN「QNN 定形 + 下游物理接力」的 24 年前原型 |
| Mohr & Gleicher, *Building Efficient, Accurate Character Skins from Examples* | SIGGRAPH 2003 | 从动画数据学蒙皮权重（example-based skinning）——SSDR 任务定义的先声 |
| Kavan et al., *Automatic Linearization of Nonlinear Skinning* | I3D 2009 | 把任意非线性变形近似为 LBS |
| **Le & Deng, SSDR**（*Smooth Skinning Decomposition with Rigid Bones*） | SIGGRAPH Asia 2012, TOG | 虚拟骨骼反解核心，概念页 [[VirtualBones]]。其直接前身是 James & Twigg, *Skinning Mesh Animations*（SIGGRAPH 2005）——把动画网格分解为骨骼+蒙皮权重 |
| Kim et al., *Near-exhaustive precomputation of secondary cloth effects* | TOG 2013 | 「把二级布料动态全部离线预烘焙+插值」的极端方案，与「运行时省动画资产」动机同源 |

### 1b. 布料数据驱动线（Pan2022 的直接基线与引文）

| 论文 | 出处 | 一句话定位 |
|---|---|---|
| Santesteban et al., *Learning-Based Animation of Clothing for Virtual Try-On* | EG 2019 | 逐顶点 pose 回归（紧身衣） |
| Gundogdu et al., *GarNet* / *GarNet++* | ICCV 2018 / TPAMI 2020 | 两流网络快速静态 draping |
| Patel et al., *TailorNet* | CVPR 2020 | pose+shape+style 分解的服装预测 |
| **Bertiche et al., DeePSD** | ICCV 2020 | **自动深度蒙皮 + PSD**：自编码器从仿真数据学骨骼/蒙皮 + 姿态空间修正——与 SSDR 反解同任务但可微可学 |
| **Bertiche et al., PBNS** | TOG 2021 | **物理自监督 PSD 布料**（零仿真数据）——[[PhySkin2026\|PhySkin]] 自监督 loss 的直系前身；SkirtQNN 方法2（物理松弛精修）可参考其能量设计 |
| Zhang et al., *Dynamic Neural Garments* (DNG) | TOG 2021 | 动态神经服装（Pan 的对比基线） |
| Li et al., *N-Cloth* | CGF 2021 | mesh-based 网络预测布料变形 |
| Santesteban et al., *Self-Supervised Collision Handling via Generative 3D Garment Models* | CVPR 2021 | **自监督防穿模**——与「不能穿模」硬约束直接相关 |
| Wang et al., *Learning an intrinsic garment space for interactive authoring of garment animation* | TOG 2019 | 服装动画的交互式作者化 |
| Habermann et al., *Real-time deep dynamic characters* | TOG 2021 | 全身实时深度动画（非服装专用） |
| Ma et al., *SCALE* / *The Power of Points*；Tiwari et al., *Neural-GIF* | CVPR/ICCV 2021 | 点/隐式衣人表示（重建向） |

## 二、Pan2022 的被引：骨骼/锚点/蒙皮驱动布料直系（2023–2026 精选）

> 已入库的直系后代不在本表重复：[[Jin2024-RopeChainQNN]]、[[HyperBones2026]]、[[PhySkin2026]]。

### 2a. 高优先级（与本谱系方法结构直接同构）

| 论文 | 出处 | 定位与关联 |
|---|---|---|
| **AnchorDEF** — Fang Zhao et al., *Learning Anchor Transformations for 3D Garment Animation* | CVPR 2023, arXiv 2304.00761 | 「骨骼」换成 mesh 面上的 FPS **锚点**：学锚点 SE(3) 刚体变换（位置/法向/方向一致性约束保持物理可解释性）+ canonical space 顶点非线性位移；自适应锚点重摆放。**专为宽松服装设计，[[HyperBones2026\|HyperBones]] 骨骼来源（FPS 采样）的先声**。待读重点：锚点摆放策略、canonical 回归、与美术骨骼的映射 |
| **D-Cloth** — Yu-Di Li et al., *Skinning-based Cloth Dynamic Prediction with a Three-stage Network* | CGF 2023 | 蒙皮式三阶段布料动态预测（与 N-Cloth 同组）。待读 |
| **GSNet** — Tao Peng et al., *Generating 3D garment animation via graph skinning network* | Graphical Models 2023 | 图蒙皮网络生成服装动画。待读 |
| **LFGarNet** — Peng Zhang et al., *Loose-Fitting Garment Animation With Multi-Attribute-Aware Graph Network* | CAVW 2025 | 宽松服装多属性感知图网络。待读 |
| Yifu Liu et al., *Towards Loose-Fitting Garment Animation via Generative Model of Deformation Decomposition* | arXiv 2312.14619, 2023 | 生成式**形变分解**：latent 分离静态 drape 与动态 offset，对抗细节增强——与 Pan 高低频分解同构的生成式版本 |
| **UNIC** — Chengfeng Zhao et al., *Neural Garment Deformation Field for Real-time Clothed Character Animation* | arXiv 2603.25580, 2026 | 实例特定神经形变场：**只泛化到新动作、不泛化到新服装**（训练更容易），点→位移偏移量，拓扑无关、实时——与 PhySkin「只学修正」哲学一致，载体从骨骼换成连续场 |
| **CPSN** — Tao Peng et al., *Collision-Overlap and Physics-Based Self-Supervised Neural Cloth Simulation* | CAVW 2026 | 碰撞+物理自监督神经布料——自监督线的最新一环。待读 |
| Junqi Diao et al., *Combating Spurious Correlations in Loose-fitting Garment Animation Through Joint-Specific Feature Learning* | CGF 2023 | 宽松服装动画的伪相关问题——与 QNN 泛化/关联偏差直接相关。待读 |
| Xipeng Chen et al., *Parametric Linear Blend Skinning Model for Multiple-Shape 3D Garments* | TVCG 2024 | 多形状服装的参数化 LBS——与「300+ 时装能否共享模型」的问题相关。待读 |
| **SAGS-GNN** — Kexuan Ban et al., *GNN for self-collision and anisotropy in dynamic garment simulation* | Computers & Graphics 2025 | **自碰撞**+各向异性——四代谱系都没解决的 self-collision，在 GNN 模拟侧有对应工作。待读 |
| AniDress — Beijia Chen et al., *Animatable Loose-Dressed Avatar from Sparse Views Using Garment Rigging Model* | arXiv 2024 | garment rigging model 做可动画宽松 avatar（重建线） |
| Zaiqiang Wu et al., *Real-Time Per-Garment Virtual Try-On with Temporal Consistency for Loose-Fitting Garments* | CGF 2025 | 逐服装 VTO + 时序一致性。待读 |
| PICA — Bo Peng et al., *Physics-Integrated Clothed Avatar* | TVCG 2024 | 物理集成衣人 avatar |
| Shichong Peng et al., *Latent Dynamics for Full Body Avatar Animation* | arXiv 2026 | 全身 avatar latent dynamics。待读 |
| Yidi Shao et al., *Learning 3D Garment Animation from Trajectories of A Piece of Cloth* | NeurIPS 2025 | 从单块布轨迹学服装动画 |

### 2b. 多层服装（裙+底裤分层穿模相关）

- ClothCombo — Dohae Lee et al., *Modeling Inter-Cloth Interaction for Draping Multi-Layered Clothes*, TOG 2023
- ISP — Ren Li et al., *Multi-Layered Garment Draping with Implicit Sewing Patterns*, NeurIPS 2023
- Yidi Shao et al., *Towards Multi-Layered 3D Garments Animation*, ICCV 2023

### 2c. 细节增强/时序网络一族（选读，与本项目「动态交给下游物理」的分工不同）

SmPhy（ICME 2024）、TDGar-Ani（Visual Computer 2024）、GarTemFormer（Graphical Models 2024）、GarTrans（CVM 2025）、StyleGarNet（CAVW 2025）、SwinGar（TVCG 2023）、Manifold-Aware Transformers（CGF 2024）、Structure-Preserving Multilevel（PACMCGIT 2024）、Hybrid multi-scale graphs（ASC 2026）——Tian-Xing Li / Jiazhe Miao 系的时序细节增强线。

### 2d. 低优先级（重建/生成/渲染向）

PhysAvatar（ECCV 2024）、SimAvatar（CVPR 2024）、ReLoo（ECCV 2024）、GaussiAnimate（2026）、Neural Garment Dynamic Super-Resolution（SIGGRAPH Asia 2024）、Garment Animation NeRF（CGF 2024）、DrapeNet（CVPR 2022）、HOOD（CVPR 2022，GNN 布料模拟，HyperBones 的对比基线）、GenSim/GarSim（CVPRW/WACV 2023，Tiwari）。

## 三、SSDR 的被引：蒙皮分解 → 辅助骨骼 → 压缩 → 弹簧二级动画

### 3a. 通用蒙皮分解/逼近线（「Pan2022 的通用版本」其实早有）

| 论文 | 出处 | 定位 |
|---|---|---|
| **Bailey & Sifakis, *Fast and deep deformation approximations*** | TOG 2018 | SSDR 式分解 + 深度网络逼近**任意**网格动画（低频骨骼 + 逐点修正，与 Pan2022 同构，早 4 年）；续作 *Fast and deep facial deformations*（TOG 2020）用于表情 |
| **Mukai, *Efficient dynamic skinning with low-rank helper bone controllers*** | TOG 2016 | **辅助骨骼（helper bones）+ 低秩控制器**——把动画序列转成带动态的骨骼 rig；续作 *Sampling-based Rig Conversion into Non-rigid Helper Bones*（PACMCGIT 2018） |
| Le, *Direct Delta Mush skinning and variants* / *DDM compression* | TOG 2019 / TOG 2021 | 一作 Le 后续：DDM 蒙皮权重平滑与压缩 |
| ARAPLBS — Thiery et al. | CGF 2018 | ARAP 弹性优化蒙皮权重+骨架关节 |
| Dodik et al., *Robust Biharmonic Skinning Using Geometric Fields* | TOG 2024 | 双调和蒙皮权重 |
| Chun-Liang Li et al., *LBS Autoencoder* | CVPR 2019 | 自监督 LBS 拟合点云 |
| RigNet（Xu, TOG 2020）/ Morig（Xu, SIGGRAPH Asia 2022） | — | 神经自动 rigging——Pan 一作后续 ACT（TOG 2026）同属此线 |
| 蒙皮权重工具线：EasySkinning（2024）、SkinCells（CGF 2025）、Geodesic Cut-Cell Neural Skinning（2026）、Moutafidou Temporal Parameter-free Deep Skinning（CGI 2021）/ Deep fusible skinning（2023）、DSNet（CGI 2021） | — | 权重生成/动画序列蒙皮拟合工具箱 |

### 3b. 压缩线（「省包体/省传输」动机的学术印证）

- Kavan et al., *Compressed Skinning for Facial Blendshapes*, SIGGRAPH 2024
- Fedotov et al., *QMF-Blend: Quantized Matrix Factorization for Efficient Blendshape Compression*, SIGGRAPH Asia 2025
- **Pietri et al., *Real-time live compression of dynamic 3D meshes for client-side GPU cloud gaming using skinning decomposition*, Visual Computer 2025**——用蒙皮分解把动态网格压成「骨骼变换+静态网格」做实时压缩传输，与 SkirtQNN「QNN 权重顶替裙摆动画曲线省包体」的动机**同构**

### 3c. 弹簧/物理二级动画工程线（SkirtQNN 分工的最近亲）

| 论文 | 出处 | 定位 |
|---|---|---|
| **Akyürek et al., *Real-Time Secondary Animation with Spring Decomposed Skinning*** | CGF 2025 | 蒙皮 + 弹簧分解做实时二级动画——与「准静态网络锚点 + 下游弹簧物理」分工最接近的图形学论文。**待读，高优先级** |
| **Sin (Blizzard), *An Animator-Friendly Chain Simulator for In-Game Animations at Blizzard Entertainment*** | SIGGRAPH Talks 2023 | 工业界游戏内链条/绳裙物理（美术友好、游戏帧预算内）——[[PyrrhaPhysics\|Pyrrha]] 的工业同类 |
| Casas et al., *Learning Nonlinear Soft-Tissue Dynamics for Interactive Avatars* | PACMCGIT 2018 | 学软组织二级动态——Kawaii 类「身体抖动」职责的学术对应 |
| Enhanced rig-space simulation | CAVW 2016 | rig-space 模拟（在骨骼空间做物理） |

## 四、与 SkirtQNN 的映射（阅读优先级）

1. **AnchorDEF（CVPR 2023）**——锚点 SE(3) 变换 + canonical 位移回归 + 自适应锚点摆放：直接影响「美术骨骼 → 锚点/虚拟骨骼映射」与蒙皮修正设计；HyperBones 的 FPS 骨骼来源即此思路的延续。
2. **Spring Decomposed Skinning（CGF 2025）+ Blizzard chain simulator（2023）+ Mukai helper bones（2016/2018）**——「准静态蒙皮 + 弹簧/物理二级动态」的既有学术与工程对应物，为「QNN 锚点 + Pyrrha/Kawaii 接力」提供先例、参数设计与美术工作流参考。
3. **PBNS（TOG 2021）/ DeePSD（ICCV 2020）**——物理自监督准静态布料、可学习的蒙皮/骨骼（替代 SSDR 反解）；方法 2（物理松弛精修标签）的能量设计可对照。
4. **SAGS-GNN（2025）+ 多层服装三篇（ClothCombo/ISP/Multi-Layered）**——自碰撞与层间碰撞：谱系内没人解决、而美术硬约束里的「不自穿」需要外部证据。
5. **Pietri 2025（蒙皮分解网格压缩）**——「骨骼表示省包体」动机的工业/学术印证，可用于体积论证。
6. 选读：D-Cloth、GSNet、LFGarNet、UNIC、CPSN、Deformation Decomposition 生成模型、Combating Spurious Correlations。

## 关联

- 谱系总览：[[Compare-BoneDrivenCloth-Lineage]] · 起点 [[Pan2022-VirtualBones]]
- 项目落地：[[SkirtQNN-Design]]
- 概念：[[VirtualBones]] · [[QNN]] · [[RopeChain]]
- 索引：[[cloth-animation/_index]]
