---
type: raw-article
title: Hierarchical Position Based Dynamics (Müller, VRIPHYS 2008)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, multigrid, convergence, hpbd, vriphys-2008]
summary: Müller（NVIDIA）VRIPHYS 2008 论文：PBD 的 Gauss-Seidel 逐约束求解收敛慢（拉伸信息逐跳传播），HPBD 用多网格层级加速——粗层级先用非线性 Gauss-Seidel（即 PBD 求解器本身）松弛、解延拓回细层级，回答了不等式约束与非线性约束如何限制/延拓到粗层两个关键问题；保持 PBD 全部优点（非线性、单向约束、稳定性）同时显著加速收敛。
---

# Hierarchical Position Based Dynamics (VRIPHYS 2008)

> 原文 PDF：`Muller-2008-HierarchicalPBD.pdf`（本目录，VRIPHYS 2008 workshop 版，PDF-1.2 老格式）；提取文本：`Muller-2008-HierarchicalPBD.txt`。
> 谱系定位：[[PositionBasedDynamics]] 的第一个收敛性优化 · 阅读顺序见 [[PBDReadingGuide]] 选修篇目。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Matthias Müller |
| 机构 | NVIDIA |
| 发表 | Workshop on Virtual Reality Interaction and Physical Simulation (VRIPHYS) 2008 |

## 核心机制一句话

**把多重网格（multigrid）思想嫁接到 PBD**：普通 multigrid 每层用 Jacobi/Gauss-Seidel 解线性系统；HPBD 把层内求解器直接换成 PBD 的**非线性** Gauss-Seidel——粗层级快速消掉低频误差（全局拉伸），细层级消高频误差（局部细节），得到一个非线性多重网格求解器。

## 关键设计

- **要回答的两个问题**（论文自述的核心贡献）：
  1. 单向（不等式）约束如何限制（restrict）到粗网格——不等式在粗层如何表示
  2. 非线性约束函数如何翻译到粗层级——细层的距离约束在粗层变成什么
- **解法**：粗层粒子 = 细层粒子簇（位置取质心、质量取和）；细层约束按簇映射生成粗层约束；解经延拓（prolongation）写回细层
- **为何不用全局法**：Newton-Raphson + PCG 贵且复杂，无法处理不等式约束（会变成 QP/LCP 问题）；multigrid 本为线性系统设计，同样难处理不等式——但 multigrid "每层用简单迭代"的结构恰好允许层内求解器换成非线性的 PBD
- **论文背景吐槽值得一看**：学术方法越来越数学化（为了电影级效果），游戏业大量物理仍是刚体+程序化假效果——PBD 系列的定位就是"填这条沟"

## 主要结果

- 收敛速度显著快于原始 PBD（多层 vs 单层 Gauss-Seidel），同等视觉质量下迭代数大幅下降
- 保持 PBD 全部特性：无条件稳定、非线性/不等式约束直接处理、位置直接可控

## 局限

- 层级数据结构复杂、内存开销大——**限制并行与引擎集成**（这是 LRA 2012 论文放弃层级路线改走单向长程约束的直接原因，见 [[Kim-2012-LongRangeAttachments|LRA]] 相关工作节）
- 粗层约束的生成本身有工程复杂度（撕裂等拓扑变化时层级须重建）

## 谱系与关联

- **上游**：[[Muller-2006-PositionBasedDynamics|PBD 2006]]
- **同期竞争**：Goldenthal 2007 Fast Projection（全局线性化+直接法）；WOR10 等层级布料法
- **后续**：加速收敛这条线后来发展出 Chebyshev 加速（Wang 2015）、[[Macklin-2019-SmallSteps|子步化 2019]]（Small Steps 论文明确引用 HPBD 为布料拉伸问题的先前方案之一）
- **定位提示**：现代引擎实现中 HPBD 被引用得多、用得少——子步化 + Jacobi 并行 + GPU 成了更实用的路线；读它主要为理解"收敛慢"这个 PBD 核心病及其解法谱系
