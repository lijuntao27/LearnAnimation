---
type: raw-article
title: A Survey on Position Based Dynamics (Bender, Müller, Macklin, EG 2017 教程讲义)
created: 2026-10-21
updated: 2026-10-23
tags: [physics-sim, pbd, xpbd, survey, tutorial, eurographics-2017]
summary: Bender（RWTH Aachen）、Müller、Macklin（NVIDIA）EG 2017 教程讲义（31 页）：PBD/XPBD 官方系统综述——从运动方程与时间积分的基础讲起，覆盖求解器对比（GS/Jacobi/变分隐式 Euler+柔顺约束）、弹性杆/布料/体积软体/刚体/流体的全套约束、各向异性与塑性等复杂效应、性能优化；2013 EG STAR 的工程化更新版，适合作参考手册按需查阅。
---

# A Survey on Position Based Dynamics（EG 2017 教程讲义）

> 原文 PDF：`Bender-2017-PBDTutorial-CourseNotes.pdf`（本目录，Eurographics 2017 Tutorial Course Notes）；提取文本：`Bender-2017-PBDTutorial-CourseNotes.txt`；**中文译本：[[Bender-2017-PBDTutorial-CourseNotes-中文翻译]]（全文翻译，2026-10-23）**。
> 谱系定位：PBD 方法族的官方综述 · 阅读顺序见 [[PBDReadingGuide]] 第 3 篇（当参考手册用）。
> 注：同组更早的 EG 2013 STAR《Position-Based Methods for the Simulation of Solid Objects in Computer Graphics》（Bender, Müller, Otaduy, Teschner）版权在 Wiley/EG diglib、无公开 PDF，本讲义是其 2017 年的更新替代，内容覆盖同一谱系且更工程化（含代码片段与 XPBD）。

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Jan Bender (RWTH Aachen), Matthias Müller, Miles Macklin (NVIDIA PhysX Research) |
| 发表 | EUROGRAPHICS 2017 Tutorial；配 slides 两份（本目录未收录） |
| 同组代码 | [PositionBasedDynamics 库](https://github.com/InteractiveComputerGraphics/PositionBasedDynamics)（Bender 维护，实现讲义中几乎所有方法） |

## 内容结构（31 页讲义目录）

1. Tutorial 详情 / 引言
2. Background：运动方程、时间积分、约束
3. **PBD 核心**：算法 + 求解器（GS/Jacobi、与变分形式隐式 Euler+柔顺约束的对比——即 XPBD 的定位）
4. 收敛改进方法
5. **Specific Constraints**：拉伸 / 剪切 / 弯曲 / 等距弯曲 / 体积……逐个给公式
6. **弹性杆（Elastic Rods）**： bend-twist / stretch-shear / Cosserat 杆
7. **布料**：完整布料管线
8. **体积可变形体**：FEM-based PBD（应变约束）、shape matching
9. **刚体系统**：接触、摩擦、关节
10. **流体**：PBF 及统一求解
11. 复杂效应：各向异性、塑性
12. 性能优化 + 展望

## 使用方式建议

- **不通读**：第 2 篇奠基论文读完后带着问题查——想实现弯曲约束看 §5、想接 FEM 看 §8、想理解 XPBD 在全景中的位置看 §3 的变分对比
- 与 [[RawSource/articles/Muller-2006-PositionBasedDynamics-中文翻译|PBD 译本]]、[[RawSource/articles/Macklin-2016-XPBD-中文翻译|XPBD 译本]] 互为补充：两篇论文是纵向深读，本讲义是横向速查；也可直接读 [[Bender-2017-PBDTutorial-CourseNotes-中文翻译|本讲义中文译本]] 全文通览
- 讲义中每个主题都给出关键公式与伪代码，配合 Bender 的开源库源码食用最佳

## 谱系与关联

- **前身**：EG 2013 STAR（Bender, Müller, Otaduy, Teschner）——无公开 PDF，未入库
- **作者三人**分别代表 PBD 谱系的三个支点：Bender（学术界：PBD 库与后续多篇论文）、Müller（奠基者）、Macklin（XPBD/Small Steps 作者）
- 2017 时点：XPBD 已发表、Small Steps 未出——讲义的收敛改进章不含子步化，读时注意时间差
