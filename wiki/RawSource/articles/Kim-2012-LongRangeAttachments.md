---
type: raw-article
title: Long Range Attachments (Kim, Chentanez, Müller, SCA 2012)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, cloth-simulation, inextensibility, lra, sca-2012, games]
summary: Kim、Chentanez、Müller（NVIDIA PhysX Research）SCA 2012 论文：游戏角色服装必有附着点（肩/颈），对每个自由粒子预计算到附着点的初始距离 r_i，超过即投影回以附着点为球心、r_i 为半径的球面——单向约束让拉伸压力波单步直达全局、又完全不影响局部褶皱形成；晚礼服 1500 顶点 <1ms/帧，PBD 不可拉伸布料的工程补丁。
---

# Long Range Attachments — 不可拉伸服装模拟（SCA 2012）

> 原文 PDF：`Kim-2012-LongRangeAttachments.pdf`（本目录）；提取文本：`Kim-2012-LongRangeAttachments.txt`。
> 谱系定位：[[PositionBasedDynamics]] 的扩展期工程补丁 · 阅读顺序见 [[PBDReadingGuide]] 选修篇目。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Tae-Yong Kim, Nuttapong Chentanez, Matthias Müller-Fischer |
| 机构 | NVIDIA PhysX Research |
| 发表 | Eurographics/ACM SIGGRAPH Symposium on Computer Animation (SCA) 2012 |

## 核心机制一句话

**给每个布料自由粒子加一条连到附着点的"看不见的长绳"**：预计算粒子 $i$ 到附着点的初始路径距离 $r_i$；模拟中粒子在半径 $r_i$ 的球内自由运动，一旦超出就投影回球面——单向（只在拉伸时激活）距离约束。

## 关键设计

- **病根**：游戏布料预算只有约 5ms/帧的零头，PBD 低迭代数下 Gauss-Seidel 局部投影传播慢——拉伸压力波要几十次迭代才能从肩传到裙摆，中间全被拉长；加迭代又把褶皱抹平（PBD 迭代多→阻尼大→细褶皱消失）
- **LRA 的妙处**：
  - 压力波**瞬时全局传播**——模拟"完全绷紧的不可拉伸布料中压力波速度无限"的物理直觉
  - **单向性**保褶皱——布料受压弯曲时（褶皱形成）LRA 完全不干预，粒子自由运动；只有拉伸路径绷直才激活
  - 附着粒子被角色动画运动学驱动、无限质量，本来就主导拉伸能量传播——LRA 只是顺应这个结构
- **与常规约束配合**：LRA 只管全局拉伸路径，相邻粒子间的拉伸/剪切/弯曲仍用常规双边约束——但所需迭代数大幅下降
- **实现成本**：每粒子一个预计算标量 + 一次距离检查，可直接嵌入任何现有 PBD 引擎

## 主要结果

- 晚礼服角色（1500 顶点）：单核 CPU **<1ms/帧**；GPU 版同帧率跑 16 个角色
- 同角色对比：LRA 5 迭代 vs PBD 5 迭代（明显拉伸）vs PBD 15 迭代（褶皱被抹平）——LRA 同时赢两头

## 局限

- 只适用于**有附着点**的服装（披风、裙、衬衫）——自由落体的桌布没有附着点可用
- LRA 不约束相邻自由粒子间的局部拉伸/压缩，须与常规约束联用
- 距离按初始路径长度取，极端大变形场景（服装滑脱）不适用

## 谱系与关联

- **上游**：[[Muller-2006-PositionBasedDynamics|PBD 2006]]（宿主框架）
- **同期方案**：Goldenthal 2007 Fast Projection（全局线性解，贵）、[[Muller-2008-HierarchicalPBD|HPBD 2008]]（层级法，数据结构复杂限制并行）——LRA 以最简数据结构换最大并行度，是游戏引擎视角的答案
- **下游**：[[Macklin-2019-SmallSteps|Small Steps 2019]] 从另一角度（子步化）同样解决"迭代预算不足则拉伸"问题，两者可叠加
- **知识库关联**：游戏角色服饰物理与 [[SkirtQNN-Design]]（裙摆附着腰点）同构——LRA 思路可直接借用到裙摆锚点设计
