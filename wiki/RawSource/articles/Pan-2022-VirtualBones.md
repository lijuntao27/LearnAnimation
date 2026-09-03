---
type: raw-article
title: Predicting Loose-Fitting Garment Deformations Using Bone-Driven Motion Networks (2022)
created: 2026-08-31
updated: 2026-08-31
tags: [cloth-animation, virtual-bones, skinning, gru, siggraph-2022]
summary: Pan 等 2022 SIGGRAPH 论文全面解读，用 SSDR 从物理模拟提取虚拟骨骼，GRU 学身体动作→骨骼 motion，GNN 补高频细节，RBF 集成模拟参数；裙摆 18546 顶点降为 80 骨骼，首篇 bone-driven loose garment 方法，开源 VirtualBones。
---

# Predicting Loose-Fitting Garment Deformations Using Bone-Driven Motion Networks (2022)

> 本文是对论文的通读解读记录（原文 PDF：`Pan-2022-VirtualBones.pdf`，在本目录；提取文本：`Pan-2022-VirtualBones.txt`；配套仓库：`VirtualBones/`）。
> 作为 `TheWiki/cloth-animation/` 各知识页的原文来源。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Predicting Loose-Fitting Garment Deformations Using Bone-Driven Motion Networks |
| 作者 | Xiaoyu Pan, Jiaming Mai, Xinwei Jiang, Dongxue Tang, Jingxiang Li, Tianjia Shao, Kun Zhou, Xiaogang Jin, Dinesh Manocha |
| 机构 | 浙江大学 CAD&CG + NExT Studios + 马里兰大学 |
| 发表 | SIGGRAPH 2022 Conference Proceedings |
| DOI | 10.1145/3528233.3530709 |
| arXiv | 2205.01355 |
| 代码 | github.com/non-void/VirtualBones |

## 1. 问题定义与动机

**输入**：身体动作序列 M_b（关节旋转 + 身体平移）+ 模拟参数 θ（弯曲刚度、密度、时间尺度）
**输出**：loose-fitting garment 网格顶点序列 G ∈ R^{t×V×3}
**目标**：交互速率预测裙摆（dress）等宽松服装变形

**动机**：已有学习方法多针对紧身服装或静态姿态，对宽松服装的大幅变形（飞、甩、落）处理不好；且不处理模拟参数变化。本文首篇用虚拟骨骼学宽松服装。

## 2. 核心管线

离线 + 运行时两段：

```
[离线] Houdini Vellum 物理模拟 → 裙摆网格序列
         → Laplacian 平滑 → 低频 / 残差=高频
         → SSDR 分解低频 → 虚拟骨骼{B} + 蒙皮权重 W + 静态网格 P

[运行时] (K=8 pivot network, RBF 加权)
  身体动作 → [低频 D_LF] GRU: body→bone motion → LBS → 低频网格
           → [高频 D_HF] GRU(全局) ‖ EdgeConv-GNN(局部) → 高频位移
           → G = G_LF + G_HF
```

## 3. 虚拟骨骼生成（§4.1）

用 SSDR（Smooth Skinning Decomposition with Rigid Bones, Le & Deng 2012）把低频网格序列反解为 LBS 模型：静态网格 P + 蒙皮权重 W_skin + 每帧每骨骼刚体变换 [R|T]。

虚拟骨骼特点：**非层级、自由旋转平移、无语义**（"virtual"指无语义，"bone"指 SSDR 提取的刚体）。与美术 rig 的有层级骨架不同。

骨骼数实验：20/40/80/150/250/500，低频 40-80 最优、高频越多越好，选 80。裙摆 18546 顶点 → 80 骨骼，DOF 降约 77×。

## 4. 低频模块（§4.2）

GRU（1层）+ Linear。输入 body motion（20 关节×3 + 平移 = 63 维），输出 80 骨骼×6（3 欧拉角 + 3 平移 = 480 维）。

用 RNN 因 history-dependent：裙摆是次级动画，当前姿态不足以定形，需记忆历史。hidden 帧间传递。

loss（公式 2）：L2 位置 + Laplacian 一致性。

## 5. 高频模块（§4.3）

- 全局流：GRU(bone motion) → 每顶点全局特征
- 局部流：3 层 EdgeConv（DGCNN）在低频网格上 → 每顶点局部特征
- MLP(concat) → 高频位移

loss：位置 + 碰撞项（最近身体顶点法向穿透惩罚，防 garment-body 穿透）。

## 6. RBF 模拟参数集成（§4.4）

K=8 个 pivot motion network，每个对应一组模拟参数。RBF 核 exp(-||g(θ_i)-g(θ)||²/2σ²) 加权凸组合（g 是 MLP 投影参数到 latent）。支持未见模拟参数交互生成。

## 7. 数据与结果

Houdini Vellum 模拟；3 件 dress + MMD 舞蹈；10 组参数（8/2 训测），每组 40000 帧。

指标：RMSE / Hausdorff / STED。低频比 TailorNet/DNG/Santesteban2019：RMSE ↓~20%、Hausdorff/STED ↓~10%。

## 8. 局限（作者自述）

- LBS 线性 artifact（多骨骼区域，建议 DQS）
- 无 self-collision 处理
- 只 pose-driven

## 9. 仓库实测（VirtualBones/）

仓库结构：5 个 .py（`run_motionnet.py`、`run_sim_param_variation.py`、`src/models.py`、`src/SSDRLBS.py`、`src/obj_parser.py`）+ 2 件 dress 预计算资产 + 5 测试动作。

关键数字（dress02）：顶点 12273，骨骼 80，skin_weights (80,12273)，低频网络 63→600→480，高频 GRU+2 层 EdgeConv[3,8,16]。

仓库缺：checkpoints（需 Google Drive 下）、psbody 依赖、数据集（单独 Google Drive）。

推理流程（`run_motionnet.py`）：逐帧自回归，body motion → GRU → 80 骨骼 6DOF → `SSDRLBS.batch_pose`(LBS) → 低频网格 → GRU_GNN → 高频位移 → 相加 → 输出 obj。旋转用欧拉角（`euler2rotmat`）。

## 10. 与裙摆补间/驱动方案的接口

- 表示层可复用：SSDR 虚拟骨骼 + LBS + 高低频分解 + 碰撞 loss + RBF 参数集成
- 生成层要改：低频 GRU 是 body→bone 的确定性回归（pose-driven），若要 keyframe 驱动需换成 in-betweening 网络
- 虚拟骨骼 motion 序列与 motion in-betweening 输入格式同构，可接 Qin 2022 补间器
- 表示差异：虚拟骨骼全局刚体变换无层级，vs 人体骨骼有层级 local 旋转

## 11. 关联

- 精炼总览：[[Pan2022-VirtualBones]]
- 对比：[[Compare-Pan2022-vs-Jin2024]]
- 虚拟骨骼概念：[[VirtualBones]]
- 后续演进：[[Jin2024-RopeChainQNN]]（Jin 2024 直接批评本文 GRU 路线）
- SSDR 来源：Le & Deng 2012
