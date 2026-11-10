---
type: raw-article
title: Spring Decomposed Skinning - Real-Time Secondary Animation (2025)
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, spring-bones, pbd, skinning, helper-bones, secondary-animation, sgp-2025]
summary: Akyürek 与 Sahillioğlu（METU）SGP 2025 论文：SDS 把二级动态物理放在骨骼层——每根弹簧骨骼=头端固定质量+尾端自由质量的 Hooke 弹簧，PBD 模拟自由质量、运动学约束恢复骨骼连通与长度、RST 从模拟骨骼反推蒙皮变换；无需四面体化、复杂度只随骨骼数，比 Wu&Umetani 四面体方案快约 10 倍，业界 jiggle bones 的学术形式化。
---

# Real-Time Secondary Animation with Spring Decomposed Skinning (2025)

> 原文 PDF：`SpringDecomposedSkinning-2025.pdf`（本目录，作者主页预印本）；提取文本：`SpringDecomposedSkinning-2025.txt`。
> 中文译本：[[SpringDecomposedSkinning-2025-中文翻译]]。
> CGF Vol.44 No.5（SGP 2025）· 代码：https://github.com/bartuakyurek/Spring-Decomposed-Skinning · 主页：https://bartuakyurek.github.io/publications/SDS/
> 谱系定位：[[BoneDrivenCloth-CitationMap]] §3c 弹簧/物理二级动画线 · 项目对照：[[SkirtQNN-Design]]（下游 Pyrrha/Kawaii 弹簧物理的学术近亲）

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Bartu Akyürek, Yusuf Sahillioğlu |
| 机构 | 中东技术大学（METU），土耳其 |
| 对比对象 | Wu & Umetani 2023（四面体 PBD+IK）、Fast Complementary Dynamics 2023、Velocity Skinning 2021 |

## 核心机制一句话

**把物理模拟限制在变形句柄（骨骼）范围内**：每根弹簧骨骼 = 骨骼头端"固定质量"（由关键帧/运动学约束驱动，不参与模拟）+ 尾端"自由质量"（PBD 模拟 Hooke 弹簧力），模拟后用运动学约束恢复骨骼树连通与原长（fixed_scale 选项），再用 RST（旋转-缩放-平移）仿射分解从"静止骨骼→模拟后骨骼"反推蒙皮变换矩阵，喂给 LBS。

## 关键设计

- **固定质量保层级**：第一质量固定、保持相对父级朝向——经典质点-弹簧网络全部粒子都更新，SDS 靠此保留骨架父子结构
- **点弹簧骨骼**：固定/自由质量都放尾端、零静止长度 → 挤压拉伸（squash & stretch）全局动态
- **约束投影位置**：运动学约束放 PBD 循环**外**（放循环内会突然振动）；fixed_scale 缩回原长 + 子骨骼头对齐父骨骼尾
- **主骨骼=全局动态，辅助骨骼链=局部动态**（软组织抖动、布料）；辅助骨骼建议与主运动骨骼**近正交**摆放（Complementary Dynamics 思想）
- 布料用**平行弹簧链**近似表面动态（单条链失败）；热扩散权重自动绑定
- 无重力/接触力/碰撞检测（留作 future work）；3-DOF 平移、无扭转弯曲

## 主要结果

- 逐帧 4.7–14.1 ms（顶点数几乎无关：Duck 12932 顶点 9.50ms < Monstera 4971 顶点 14.11ms，因骨骼更少）
- 比 Wu & Umetani 快约 10 倍（1.98 vs 20.73ms 等）；精细 rig 下四面体方案出现裂纹/自相交/刚度异常，SDS 动态与网格分辨率基本无关
- 互补动态（正交约束）不能表达"与一级运动同方向的抖动"；SDS 点弹簧可同向夸张

## 局限

- Hooke 弹簧参数敏感、位置驱动力可能爆炸；链越长越难调参；无碰撞检测易自相交；参数手工设置
- 引用 [JHG*22] Jin/Fedkiw 零静止长度弹簧（[[Jin2024-RopeChainQNN|Jin 2024]] 前身）—— 与 rope-chain 线同源

## 关联

- 思想同源：[[Jin2024-RopeChainQNN]]（[JHG*22]）、Capell 2002 骨骼驱动动态变形、Wu & Umetani 2023、Complementary Dynamics
- 业界对应：God of War Ragnarök GDC 2023 [Wan23]、UE 动态骨骼、Blender/Unity jiggle bones
- 项目对照：[[SkirtQNN-Design]]（"准静态锚点 + 下游弹簧物理"分工的学术正名；固定质量/自由质量设计与"QNN 锚点驱动 Pyrrha/Kawaii"结构同构）
