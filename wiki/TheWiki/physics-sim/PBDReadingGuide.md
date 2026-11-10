---
type: concept
title: PBD 论文导读（Position Based Dynamics 阅读路线图）
created: 2026-10-21
updated: 2026-10-22
tags: [physics-sim, pbd, xpbd, reading-guide, survey]
summary: PBD 方法族从 2001 到 2019 的论文阅读路线：Jakobsen 游戏技巧 → Müller 2006 奠基 → 扩展期（层级化/长程约束/流体/统一粒子）→ XPBD 物理化 → Small Steps 子步化。每篇标注核心贡献、必读章节与库内原文位置，并给出整体演进逻辑主线。
---

# PBD 论文导读

> 本页是 physics-sim 领域的**总导读**：按阅读顺序组织 PBD 方法族的核心论文，标注每篇解决什么、怎么解决、与上下游的关系。读完本页应能对"这条技术线为什么这样演进"有完整把握。
> 各论文详细内容见 [[TheWiki/physics-sim/_index|physics-sim 索引]] 的原文解读区。

## 演进主线（一图流）

```text
Jakobsen 2001        Verlet + 距离约束松弛（游戏技巧，无理论）
    │
Müller 2006          PBD 正式化：一般约束 + 位置投影框架（奠基）
    │
┌───┴───────────────┬──────────────────┬─────────────────┐
HPBD 2008        LRA 2012         PBF 2013        Flex 2014
层级化加速        布料不可拉伸       流体密度约束      粒子统一表示
    │               │                │                │
    └───────────────┴───────┬────────┴────────────────┘
                            │
XPBD 2016             柔度 α + 乘子 λ → 刚度物理化（质变）
                            │
Fratarcangeli 2018         XPBD 刚体化（数千刚体堆叠）
                            │
Small Steps 2019           子步化 → "迭代数"也变成物理量
                            │
（前沿）能量化约束：Neo-Hookean 约束化 2021、Physically Based
Shape Matching 2022 → PBD 与 FEM/牛顿求解器理论上统一
```

**演进逻辑**：每一代都在消灭上一代的一个"非物理"参数——
1. Jakobsen→Müller：消灭"只支持距离约束"的局限；
2. Müller→扩展期：消灭"单一材质/单一物态"的局限；
3. 扩展期→XPBD：消灭**刚度对迭代数/时间步的依赖**（最重要的质变）；
4. XPBD→Small Steps：消灭**迭代数本身**这个非物理旋钮——切成小子步、每步 1 次迭代，让时间离散回归物理。

## 阅读顺序与每篇要点

### 第 1 篇 · Jakobsen 2001 — 建立直觉（半天）

**Advanced Character Physics** — Thomas Jakobsen（IO Interactive），GDC 2001 / Game Developer

- 《杀手 47》的角色物理：Verlet 积分 + 逐点松弛满足距离约束
- **为什么先读**：PBD 的全部直觉都在这里——速度隐含在前后位置差里、约束就是"把点拉回该在的地方"、不需要解任何线性系统。没有公式推导，纯粹工程视角
- 库内位置：[[RawSource/articles/Jakobsen-2001-AdvancedCharacterPhysics|索引卡]]（原 PDF 镜像已 404，全文取自 gotoandplay.it 2005 转载，正文提取为 .txt）
- 阅读重点：Verlet 积分一节、粒子系统与约束松弛的实现描述、§5 刚体的 λ 推导（PBD 式 (9) 的原型）

### 第 2 篇 · Müller 2006 — 奠基论文（精读，1-2 天）

**Position Based Dynamics** — Müller, Heidelberger, Hennix, Ratcliff（AGEIA），VRIPHYS 2006

- 正式确立 PBD 框架：一般约束函数 $C_j$ + Gauss-Seidel 位置投影 + 逆质量加权
- 概念页：[[PositionBasedDynamics]] · 原文：[[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|中文译本]]
- **必读章节**：§3.1 算法总览（17 行伪代码）、§3.3 约束投影推导（式 1–11）、§3.5 刚体模式阻尼、§4 布料全套约束
- 读完后应能：不看论文独立写出一个 2D 布料 demo（距离约束 + 弯曲约束 + 鼠标拖拽）

### 第 3 篇 · 综述 — 系统化（1 周，可当参考手册）

**Position-Based Methods for the Simulation of Solid Objects in Computer Graphics** — Bender, Müller, Otaduy, Teschner，EG 2013 STAR
（同组更新的教程讲义：**Position-Based Simulation Methods in Computer Graphics 2017**，EG 2017 课程）

- PBD 放在与力基方法、冲量方法的对比中系统讲述；布料/软体/流体/刚体分章
- 库内位置：[[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes|2017 教程讲义索引卡]]（31 页，含代码片段，比 STAR 更工程化）；全文通读可用 [[RawSource/articles/Bender-2017-PBDTutorial-CourseNotes-中文翻译|中文译本]]
- 用法：不必通读——第 2 篇读完后**带着问题查**：想实现弯曲看其弯曲章、想理解体积约束看其软体章

### 第 4 篇 · XPBD — 质变（精读，1-2 天）

**XPBD: Position-Based Simulation of Compliant Constrained Dynamics** — Macklin, Müller, Chentanez（NVIDIA），MIG 2016

- 柔度 $\alpha$ + 总拉格朗日乘子 $\lambda$：刚度与迭代数/时间步解耦，对应明确弹性势能，免费输出约束力
- 概念页：[[XPBD]] · 原文：[[RawSource/articles/Macklin-2016-XPBD-中文翻译|中文译本]]
- **必读章节**：§1（病根，理解动机）、§4 推导主线（吃透式 18）、§4.1（"$\alpha_j = 0$ 退化为 PBD 的 $s_j$"——理解 PBD/XPBD 关系的钥匙）、§6 四大验证实验
- 读完后应能：把第 2 篇的布料 demo 改造成 XPBD，体会"同一 $\alpha$ 在 10 次和 100 次迭代下同样的硬度"

### 第 5 篇 · Small Steps — 现代范式（1 天）

**Small Steps in Physics Simulation** — Macklin, Storey, Lu, Terdiman, Chentanez, Jeschke, Müller（NVIDIA），SCA 2019

- 核心发现：**大时间步 + $n$ 次迭代 < 小时间步 × $n$（每步 1 次迭代）**。把时间切成大量子步，每步只做 1 次约束求解
- 为什么：每次迭代内约束可独立求解（可并行）、收敛更快、刚性材料更稳——"迭代数"这个非物理参数被消灭，取而代之的是物理的时间离散
- 库内位置：[[RawSource/articles/Macklin-2019-SmallSteps|索引卡]]
- 阅读重点：摘要与引言的观察、求解策略对比图、GPU 实现一节
- 读完对照：如今业界实时布料/毛发引擎（含 [[PyrrhaPhysics]] 类方案）普遍采用子步范式

### 第 6-8 篇 · 按兴趣选修

| 篇目 | 场景 | 何时读 |
|---|---|---|
| **Hierarchical PBD**（Müller 2008）| 多分辨率层次加速约束传播 | 想优化大规模布料性能时 |
| **Long Range Attachments**（Kim, Chentanez, Müller 2012）| 顶点到根节点长程约束解决不可拉伸 | 实现游戏布料必读——纯 Gauss-Seidel 传播不到远端，这是工程上最重要的补丁 |
| **Position Based Fluids**（Macklin & Müller 2013）| 密度约束 + 人工压力项做流体 | 想把 PBD 扩到流体时；NVIDIA Flex 的源头之一 |
| **Unified Particle Physics / Flex**（Macklin et al. 2014）| 粒子+约束统一表示流体/布料/刚体/毛发 | 想看"一个求解器统治所有物态"的集大成设计 |

### 拓展线（未入库，遇到再补）

- **Fratarcangeli 2018**（Detailed Rigid Body Simulation with XPBD）：XPBD 刚体化，数千刚体稳定堆叠
- **Macklin & Müller 2021**（Constraint-based Neo-Hookean）：FEM 能量写成 XPBD 约束
- **Müller et al. 2022**（Physically Based Shape Matching）：shape matching 的能量化正名

## 阅读装备

- **数学底座**：[[IterativeSolvers]]——Jacobi/GS 迭代的完整数学原理（分裂法、收敛性三路线、PBD 对应表），读 PBD 论文 §3 求解器一节前建议过一遍；Verlet 积分推导（Taylor 展开消速度项）暂记于该页"配套概念"，需要时独立成页
- **代码库**：[PositionBasedDynamics（Jan Bender 维护）](https://github.com/InteractiveComputerGraphics/PositionBasedDynamics)——已克隆至 `H:\Animation\PositionBasedDynamics`（2026-10-21），实现了上述几乎所有方法（XPBD/PBF/弹性杆/FEM-PBD），pip 可装 `pyPBD`。读论文配这份代码是最高效路径
- **视频**：Matthias Müller 的 Ten Minute Physics（YouTube）——从零手写每个 demo，XPBD 实现细节的最好入门
- 每篇读完建议在对应索引卡的"局限"节旁记下你自己的验证（跑 demo / 改参数），这是知识库[[TheSchema/ingest-guide|录入规则]]倡导的"3 个月后还能看懂"标准

## 与其他领域的接口

- [[TheWiki/cloth-animation/_index|cloth-animation]]：PBD 族是 [[RawSource/articles/SpringDecomposedSkinning-2025|SDS]]、[[PyrrhaPhysics]] 等服饰物理方案的求解器底座
- [[SkirtQNN-Design]]：本项目裙摆管线中"QNN 准静态锚点 + 下游 XPBD 弹簧物理"的分工即建立在第 4-5 篇的现代范式上
