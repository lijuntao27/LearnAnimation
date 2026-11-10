---
type: index
title: 物理仿真 索引
updated: 2026-10-26
---

# 物理仿真（Physics Simulation）

> 实时物理仿真中基于位置的动力学（PBD）方法族研究：位置级约束投影框架、XPBD 物理化扩展、Small Steps 子步化范式，及其在布料/流体/刚体/骨骼级二级动画中的应用谱系。

## 论文导读（从这里开始）

| 页面 | 类型 | 简介 |
|------|------|------|
| [[PBDReadingGuide]] | concept | **PBD 方法族阅读路线图**：2001→2019 演进主线、每篇的核心贡献与必读章节、阅读装备 |

## 概念页

| 页面 | 类型 | 简介 |
|------|------|------|
| [[PositionBasedDynamics]] | concept | Müller 2006 PBD 奠基框架：预测位置后把一般约束当几何投影 Gauss-Seidel 迭代修正，无条件稳定，游戏实时布料的奠基求解器 |
| [[XPBD]] | concept | Macklin 2016 对 PBD 的物理化扩展：柔度 α + 总拉格朗日乘子 λ，刚度与迭代数/时间步解耦、可输出约束力，开销 <2% |
| [[SmallSteps]] | concept | Macklin 2019 子步化范式：n 个小子步 × 每步 1 次 XPBD 迭代取代大步 × n 迭代，误差 ∝ Δt² 平方级削减，现代实时引擎标配 |
| [[IterativeSolvers]] | concept | Jacobi/Gauss-Seidel 迭代数学原理：分裂法框架、线性收敛三路线（对角占优归纳收缩/谱半径/SPD 能量单调）、三角剥皮 O(n²)；§4 非线性侧——PBD 循环投影的收敛性（三层"收敛"区分、POCS 凸集全局保证、KM 平均=k<1 的定理解释、非凸横截角局部线性收敛、XPBD=能量单调 BCD、约束逐个过堂表与翻车模式对照）；全部概念落到 PBD 对应点，含手算基准例与更正记录 |

## 原文解读（按阅读顺序）

| 文件 | 内容 |
|------|------|
| [[RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics\|Jakobsen 2001 索引卡]] | GDC 2001：《杀手 47》角色物理——Verlet + 约束松弛，PBD 前身（原 PDF 失联，全文取自 2005 转载）；含 [[RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics-中文翻译\|中文译本]] |
| [[RawSource/articles/Muller-2006-PositionBasedDynamics\|Müller 2006 PBD 索引卡]] | VRIPHYS 2006 奠基论文：约束投影推导、布料全套约束、阻尼/碰撞/撕裂；含 [[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译\|中文译本]] |
| [[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes\|Bender 2017 综述讲义索引卡]] | EG 2017 教程：PBD/XPBD 官方系统综述（31 页），求解器/全套约束/五类物态，当参考手册按需查；含 [[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes-中文翻译\|中文译本]]（全文翻译） |
| [[RawSource/articles/Macklin-2016-XPBD\|Macklin 2016 XPBD 索引卡]] | MIG 2016：隐式离散推导、核心更新式、FEM 接入、四大验证实验；含 [[RawSource/articles/Macklin-2016-XPBD-中文翻译\|中文译本]] |
| [[RawSource/articles/Macklin-2019-SmallSteps\|Macklin 2019 Small Steps 索引卡]] | SCA 2019：子步化范式——n 个小子步 × 每步 1 次 XPBD 迭代优于大步 × n 迭代，误差 ∝ Δt² 平方级削减 |
| [[RawSource/articles/Kim-2012-LongRangeAttachments\|Kim 2012 LRA 索引卡]] | SCA 2012：长程附着约束——每粒子到附着点的单向距离约束，压力波单步全局直达、保褶皱，<1ms/帧 |
| [[RawSource/articles/Muller-2008-HierarchicalPBD\|Müller 2008 HPBD 索引卡]] | VRIPHYS 2008：多网格层级加速 PBD 收敛，非线性 Gauss-Seidel 当层内求解器 |
| [[RawSource/articles/Macklin-2013-PositionBasedFluids\|Macklin 2013 PBF 索引卡]] | SIGGRAPH 2013：密度约束 + 人工压力 + 涡度约束，SPH 不可压缩性的 PBD 化，Flex 源头 |
| [[RawSource/articles/Macklin-2014-UnifiedParticlePhysics\|Macklin 2014 Flex 索引卡]] | SIGGRAPH 2014：粒子+约束统一表示五类物态，双向耦合免费，NVIDIA Flex SDK 出处 |

## 阅读路径

1. [[PBDReadingGuide]]（总导读，先看演进主线）
2. [[PositionBasedDynamics]] → [[XPBD]] → [[SmallSteps]]（三块理论基石的概念页）
3. 原文按导读顺序：PBD 译本 → 2017 综述（手册式查阅）→ XPBD 译本 → Small Steps → 选修（LRA/HPBD/PBF/Flex）
4. 与其他子域的交汇：
   - [[TheWiki/cloth-animation/_index|cloth-animation]]：[[RawSource/articles/SpringDecomposedSkinning-2025\|SDS 2025]] 用 PBD 做弹簧骨骼、[[PyrrhaPhysics]] 以 XPBD+杆模拟为主打——PBD 族在布料/服饰动画的实际落地
5. 待补方向：Fratarcangeli 2018（Wiley 无 OA，需机构订阅）、Macklin 2021 Neo-Hookean、Müller 2022 Shape Matching、EG 2013 STAR 原版（Wiley/EG diglib 无公开 PDF，以 [[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes\|2017 讲义]] 替代）
