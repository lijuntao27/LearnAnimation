---
type: index
title: 布料动画 索引
updated: 2026-09-02
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
| [[RawSource/articles/HyperBones-2026\|HyperBones 2026 全文解读]] | 论文通读记录：模块[A][B][C]、L_phys/L_3D、6 服装结果与消融 |
| [[RawSource/articles/PhySkin-2026\|PhySkin 2026 全文解读]] | 论文通读记录：两层 LBS、双层调制、L_phys 四项、CPU 微秒数据 |

## 阅读路径

1. [[Pan2022-VirtualBones]] → [[Jin2024-RopeChainQNN]]（看演进：GRU → 物理+QNN）
2. [[Compare-Pan2022-vs-Jin2024]]（横向对比）
3. 2026 业界线：[[HyperBones2026]] → [[PhySkin2026]] → [[Compare-BoneDrivenCloth-Lineage]]（看 GRU 之后的两条独立解法：自监督蒸馏 / 准静态调制）
4. [[VirtualBones]] / [[RopeChain]] / [[QNN]]（核心概念）
5. 原文解读（深入细节）
