---
type: index
title: 布料动画 索引
updated: 2026-09-09
---

# 布料动画（Cloth Animation）

> 神经网络驱动的宽松服装（裙摆、披风）动画方法研究：虚拟骨骼表示、rope chain 物理、QNN 准静态形状推断、2026 业界实时线（自监督蒸馏 / 准静态调制）。

## 论文总览

| 页面 | 类型 | 简介 |
|------|------|------|
| [[Pan2022-VirtualBones]] | concept | Pan 2022 方法总览：SSDR 虚拟骨骼 + GRU 低频 + GNN 高频 + RBF 参数集成 |
| [[Jin2024-RopeChainQNN]] | concept | Jin 2024 方法总览：rope chain 物理 + neural skinning + QNN + 解析 SDF |
| [[HyperBones2026]] | concept | 2026 Meta 业界线（动态）：FPS 虚拟骨骼 + 教师蒸馏动态 + UV Conv-MLP，零仿真数据，GPU 实时 |
| [[PhySkin2026]] | concept | 2026 Meta 业界线（准静态）：FPS 节点 + 双层调制 + 物理自监督，单线程 CPU 微秒、拓扑无关 |
| [[Compare-Pan2022-vs-Jin2024]] | comparison | Pan 2022 vs Jin 2024：SSDR vs rope chain、GRU vs QNN、数据需求、适用场景 |
| [[Compare-BoneDrivenCloth-Lineage]] | comparison | 四代谱系 Pan2022→Jin2024→HyperBones/PhySkin 五维对比与两支独立谱系 |
| [[BoneDrivenCloth-CitationMap]] | survey | 引文图谱：从 Pan2022 与 SSDR 引文出发的扩展文献清单（锚点线/蒙皮分解线/弹簧二级动画线/压缩线），含 SkirtQNN 阅读优先级 |

## 项目实践

| 页面 | 类型 | 简介 |
|------|------|------|
| [[SkirtQNN-Design]] | concept | 本项目移动端时装裙摆方案设计笔记：Pyrrha+Kawaii 模拟→美术手K微调→离线骨骼动画为训练权威数据，QNN 填补混搭 gap，运行时双锚点 + 同一套二级物理接力；持续更新 |
| [[PyrrhaPhysics]] | concept | 网易艺设三部TA组自研骨骼级物理方案：XPBD+杆模拟主打裙摆/复杂服饰，动态碰撞器与层ID防自穿，被评估为 SkirtQNN 运行时二级物理的候选下游 |

## 核心概念

| 页面 | 类型 | 简介 |
|------|------|------|
| [[VirtualBones]] | concept | 虚拟骨骼：SSDR 从变形网格反解的低自由度骨骼表示 |
| [[RopeChain]] | concept | 绳链：纵向连接虚拟骨骼的不可拉伸可缩短物理结构 |
| [[QNN]] | concept | 准静态神经网络：不含时序、逐帧独立，学状态本身省数据 |

## 原文解读

| 文件 | 内容 |
|------|------|
| [[RawSource/articles/Pan-2022-VirtualBones\|Pan 2022 全文解读]] | 论文通读记录，配套仓库 VirtualBones 实测 |
| [[RawSource/articles/Jin-2024-LooseFittingClothing\|Jin 2024 全文解读]] | 论文通读记录，含 rope chain/QNN 机制 |
| [[RawSource/articles/PyrrhaPhysics-KM\|Pyrrha Physics KM 原文]] | 内部 KM 文章归档：艺设三部 TA 组自研骨骼级物理方案，含对比表与联系方式 |
| [[RawSource/articles/HyperBones-2026\|HyperBones 2026 全文解读]] | 论文通读记录：模块[A][B][C]、L_phys/L_3D、6 服装结果与消融 |
| [[RawSource/articles/PhySkin-2026\|PhySkin 2026 全文解读]] | 论文通读记录：两层 LBS、双层调制、L_phys 四项、CPU 微秒数据 |
| [[RawSource/articles/AnchorDEF-2023\|AnchorDEF 2023 索引卡]] | CVPR 2023 锚点线：自适应锚点刚体变换混合+canonical 位移，L_consis/L_dir 约束，QEM 监督锚点摆放；含 [[RawSource/articles/AnchorDEF-2023-中文翻译\|中文译本]] |
| [[RawSource/articles/SpringDecomposedSkinning-2025\|Spring Decomposed Skinning 2025 索引卡]] | SGP 2025 弹簧骨骼蒙皮：固定/自由质量+PBD+运动学约束+RST，业界 jiggle bones 形式化；含 [[RawSource/articles/SpringDecomposedSkinning-2025-中文翻译\|中文译本]] |
| [[RawSource/articles/TailorNet-2020\|TailorNet 2020 索引卡]] | CVPR 2020：低频 MLP+高频窄带宽混合，首个 pose/shape/style 联合模型，Pan2022 高低频与 RBF 设计的出处；含 [[RawSource/articles/TailorNet-2020-中文翻译\|中文译本]] |

## 阅读路径

1. [[Pan2022-VirtualBones]] → [[Jin2024-RopeChainQNN]]（看演进：GRU → 物理+QNN）
2. [[Compare-Pan2022-vs-Jin2024]]（横向对比）
3. 2026 业界线：[[HyperBones2026]] → [[PhySkin2026]] → [[Compare-BoneDrivenCloth-Lineage]]（看 GRU 之后的两条独立解法：自监督蒸馏 / 准静态调制）
4. [[VirtualBones]] / [[RopeChain]] / [[QNN]]（核心概念）
5. 原文解读（深入细节）
6. 项目落地：[[SkirtQNN-Design]]（把以上四代方案的取舍应用到本项目 300+ 时装的实际约束）
