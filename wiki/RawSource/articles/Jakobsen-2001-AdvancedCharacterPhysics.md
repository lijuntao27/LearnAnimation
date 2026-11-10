---
type: raw-article
title: Advanced Character Physics (Jakobsen, GDC 2001)
created: 2026-10-21
updated: 2026-10-21
tags: [physics-sim, pbd, verlet, hitman, game-physics, gdc-2001]
summary: Jakobsen（IO Interactive）GDC 2001 经典：为《杀手 47》开发的角色物理——Verlet 积分隐式存储速度、粒子+距离约束逐点松弛直接改位置，不解线性系统；布料=粒子网、刚体=四面体四粒子六约束、人体=棍图粒子骨架；碰撞摩擦靠投影+穿透深度修改切向速度。PBD 的直接祖先。原文 PDF 各镜像 404，全文取自 gotoandplay.it 2005 转载。
---

# Advanced Character Physics（GDC 2001）

> 原文状态：Game Developer/gamasutra 原链与各大学镜像均已 404（2026-10 核实）；**全文取自 [gotoandplay.it 2005 年转载](http://www.gotoandplay.it/_articles/2005/08/advCharPhysics.php)**（两页，HTML），提取文本：`Jakobsen-2001-AdvancedCharacterPhysics.txt`（正文 Abstract→References 完整覆盖 §1–9，代码块保留）。
> 中文译本：[[Jakobsen-2001-AdvancedCharacterPhysics-中文翻译]]。
> 谱系定位：[[PositionBasedDynamics]] 的直接前身 · 阅读顺序见 [[PBDReadingGuide]] 第 1 篇。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Thomas Jakobsen（IO Interactive 研究主管，丹麦技术大学数学博士） |
| 发表 | GDC 2001 / Game Developer Magazine |
| 应用 | 《杀手：代号 47》(Hitman: Codename 47) 的 Glacier 引擎物理系统 |

## 核心机制一句话

**Verlet 积分 + 粒子约束松弛**：每粒子只存当前位置 $\mathbf{x}$ 与上一位置 $\mathbf{x}^*$（速度隐含于 $\mathbf{x}-\mathbf{x}^*$）；每时间步先外推 $\mathbf{x}' = 2\mathbf{x} - \mathbf{x}^* + \mathbf{a}\Delta t^2$，然后反复执行约束松弛（把距离约束两端粒子直接推/拉回原距）——**不解线性系统、不显式算力，位置被改了速度就自动被改**。

## 关键设计（原文结构）

1. **§2 Verlet 积分**：分子动力学传统积分器；比 Euler 稳（速度位置不会失同步）；系数 2 改 1.99 还能白捡一点拖曳阻尼
2. **§3 投影处理碰撞**：穿透点直接沿法线投影出障碍物（恢复系数=0 时下一帧速度自动无法向分量）——"惩罚弹簧法要调刚度、回退时间法太慢，投影法两全"
3. **§4 松弛解多约束**：盒子约束（单向）+ 木棍约束（双向）逐个满足再整体迭代——"看似天真，实际收敛"；**布料** = 三角网格每边一条 stick 约束，多数布料 **1 次迭代就够**；平方根用静止长度处一阶 Taylor 展开近似（= 一次 Newton-Raphson），N 次开方化 N 次除法
4. **§5 刚体**：四面体 4 粒子 + 6 条边约束（$4 \times 3 - 6 = 6$ 自由度）；碰撞点 $\mathbf{p}$ 表为粒子位置线性组合 $\mathbf{p} = c_1\mathbf{x}_1 + c_2\mathbf{x}_2 + c_3\mathbf{x}_3 + c_4\mathbf{x}_4$，按系数分配修正量——"通常 3~4 次松弛就够；物体不会完全刚性——这其实是优点，世上本无完美刚体，尤其人体"
5. **§6 关节体**：两刚体共享 1 粒子=销关节、共享 2 粒子=铰链；角度限位用单向距离约束或点积约束；**Hitman 尸体实际更简单**：棍图（stick figure）粒子骨架，膝盖/手肘的弯曲朝向由骨骼动画系统强制——省掉绕肢体轴旋转的模拟，大幅提速
6. **§7 工程杂记**：IK=松弛循环里反复把粒子钉到目标位（invmass=0）；摩擦=投影后按穿透深度比例削减切向速度（改 $\mathbf{x}^*$）；连续碰撞用上一帧到当前帧的线段测试；"误差不累积——投影+Verlet 把复杂计算摊到多帧，动静小时误差也小"

## 历史地位

- **PBD 的全部直觉源头**：Müller 2006 的式 (10)(11) 正是本文 stick 约束公式的数学化；"看似天真的重复实际收敛"即 Gauss-Seidel 松弛；PBD 2006 补上了本文缺的一般约束框架与动量守恒分析
- 原文自曝数学根基：实为"微分包含的时间步进法 + 简式内点法"，投影即向约束流形投影、松弛即隐式解线性系统——但作者刻意隐藏数学以达到最广受众（"origins may appear somewhat magical"）
- 粒子级操作天然适合向量处理器（当年 PS2）——20 年后 GPU Jacobi 求解器（[[Macklin-2014-UnifiedParticlePhysics|Flex 2014]]）是同一论断的延续
- 粒子骨架模拟人体的思路在 [[RawSource/articles/SpringDecomposedSkinning-2025|SDS 2025]]、[[PyrrhaPhysics]] 等骨骼物理方案中再度登场

## 阅读指引

- 全文无公式推导、纯工程叙述 + C++ 伪代码——半天可通读，**适合作为整个 physics-sim 领域的第一篇**
- **中文译本**：[[Jakobsen-2001-AdvancedCharacterPhysics-中文翻译]]（全文翻译，文末带译读导航）
- 重点：§2 Verlet（与 [[Muller-2006-PositionBasedDynamics-中文翻译|PBD 2006 译本]] §3.1 的 17 行伪代码对照——同一循环的"无速度版"与"显速度版"）、§4 松弛与布料、§5 的 $\lambda$ 推导（PBD 2006 式 (9) 的原型）
