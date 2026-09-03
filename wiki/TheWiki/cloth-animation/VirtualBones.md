---
type: concept
title: Virtual Bones（虚拟骨骼）
created: 2026-08-31
updated: 2026-08-31
tags: [cloth-animation, virtual-bones, skinning]
summary: 用 SSDR 从变形网格序列反解的低自由度骨骼表示，非层级、无语义、自由旋转平移；把高维顶点动画降为骨骼动画，是布料降维和补间的表示底座。
---

# Virtual Bones（虚拟骨骼）

> 布料动画中的低自由度表示概念，源于 SSDR（Le & Deng 2012），[[Pan2022-VirtualBones|Pan 2022]] 首篇用于 loose garment。

## 定义

虚拟骨骼 = 用 SSDR 从变形网格序列反解出的一组刚体骨骼 + 蒙皮权重，使原网格动画可由骨骼变换经 LBS 重建。

## 与美术 rig 骨骼的区别

| | 美术 rig 骨骼 | 虚拟骨骼 |
|---|---|---|
| 层级 | 有（父子）| 无（非层级）|
| 语义 | 有（大腿、小腿）| 无 |
| 运动 | local 旋转（受父级约束）| 全局刚体变换（自由）|
| 来源 | 美术手动 | SSDR 自动提取 |

## 为什么用于布料

布料顶点动画 DOF 极高（裙摆 18546 顶点×3），用虚拟骨骼降到 80×6=480 DOF，降约 77×。且骨骼动画序列与 [[MotionInbetweeningTwoStage|motion in-betweening]] 输入同构，可接补间网络。

## 提取方法

SSDR（Smooth Skinning Decomposition with Rigid Bones）：输入网格序列，输出静态网格 P + 蒙皮权重 W + 每帧骨骼变换 [R|T]。LBS 重建误差可控。

## 关联

- [[Pan2022-VirtualBones]]（首篇用于 loose garment）
- [[Jin2024-RopeChainQNN]]（rope chain 也是虚拟骨骼 + 纵向绳约束）
- 代码：`VirtualBones/src/SSDRLBS.py`
