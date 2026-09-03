---
type: comparison
title: 骨骼驱动布料四代谱系 Pan2022→Jin2024→HyperBones/PhySkin
created: 2026-09-02
updated: 2026-09-02
tags: [cloth-animation, comparison, lineage]
summary: Pan2022→Jin2024→HyperBones2026/PhySkin2026 四代骨骼驱动布料五维对比：骨骼来源、动态由谁产生、数据需求(4万→0)、碰撞、runtime(交互→CPU微秒)；两支独立谱系——HyperBones/PhySkin 只引 Pan 未引 Jin。
---

# 骨骼驱动布料四代谱系 Pan2022 → Jin2024 → HyperBones/PhySkin

> 从 [[Pan2022-VirtualBones]]（SIGGRAPH 2022）到 [[Jin2024-RopeChainQNN]]（arXiv 2024）再到 2026 Meta 业界线的 [[HyperBones2026]] / [[PhySkin2026]]。每代都在纠正上一代的具体毛病。

## 总览表

| 维度 | [[Pan2022-VirtualBones\|Pan 2022]] | [[Jin2024-RopeChainQNN\|Jin 2024]] | [[HyperBones2026\|HyperBones]] | [[PhySkin2026\|PhySkin]] |
|---|---|---|---|---|
| 骨骼来源 | SSDR **从仿真数据反解** | 手动/SSDR，纵向串成绳链 | **FPS 在 canonical 服装面采样**（不反解） | **FPS 采样** + heat-method 蒙皮权 |
| 骨骼规模 | 80（裙摆） | 234（26链×9骨） | **128** | **128** |
| 骨骼 DOF | 旋转+平动（全局刚体） | **只平动** | 旋转(6D)+平动 | 仿射修正 Δχ |
| 动态来源 | **GRU 学**（body→bone 回归） | **rope chain 物理**产生 | **MeshGraphNet 教师蒸馏**（训练期积分，推理丢弃） | **无**（准静态，主动放弃） |
| 高频细节 | GRU‖EdgeConv-GNN | neural skinning + QNN | **UV Conv-MLP** | 并入 Node Deformer 修正 |
| 数据/监督 | 40000 帧×10 参数组（需仿真） | 5000 帧（需仿真） | **零仿真数据**（AMASS+物理loss） | **零仿真数据** |
| 身份/参数适配 | RBF 插值 K=8 pivot | 无 | **FiLM 形状码 z**（零逐帧开销） | **双层调制**（点编码身体/MLP 姿态） |
| 碰撞 | garment-body loss；无 self-collision | 解析 SDF + history push-out（骨骼级） | L_collision(教师)+L_interp(学生几何罚) | 距身体 0.3cm 平方穿入罚 |
| Runtime | 交互速率（未标） | 实时 | **GPU ~1ms/帧**，25× HOOD | **单线程 CPU ~59µs**，~1.7万 FPS |
| 泛化 | 单服装/体型，跨参数插值 | 单服装 | 固定 6 服装，跨体型/姿态 | **服装/身体拓扑无关**，最强 |
| 明显局限 | LBS artifact、无 self-collision、费数据 | 准静态假设、OOD 穿透 | 固定服装集、非零样本新服装 | 完全无动态 |

## 逐维演进

### 骨骼来源：从"反解"到"直接采样"
Pan 需先有仿真数据才能 SSDR 反解骨骼；2026 两篇跳过反解，在 canonical 服装面上 **FPS 直接采点**，初始蒙皮权用几何距离（测地/heat method），只学"修正量"。去掉"先仿真→再反解"的循环依赖，是自监督管线成立的前提。

### 动态来源：对同一个病根（GRU 学时序、费数据、泛化差）的三种药
1. **Jin 2024**：动态整个搬给物理（rope chain），网络只学准静态形状（QNN）；
2. **HyperBones**：不放弃网络学时序，但换训练信号来源——训练期物理积分教师（MeshGraphNet）产出物理轨迹，蒸馏给无状态 3 帧窗口学生。每帧独立被教师纠偏，避免 GRU 长程累积误差，**又保留滞后/惯性**；
3. **PhySkin**：判定"日常大多时刻动态可忽略"，干脆不要动态 → 换来最快、最泛化的 runtime。

一句话：Jin 用物理造动态、HyperBones 用教师蒸馏动态、PhySkin 不要动态。

### 数据需求的台阶
40000×10 → 5000 → **0** → **0**。从"少要一点仿真数据"到"完全不要仿真数据"是质变，靠**可微物理能量自监督**（拉伸/弯曲/碰撞/重力/惯性），不再依赖 Houdini Vellum 预生成。

### 碰撞
四篇都只有 garment-body，**均无 self-collision**（2022→2026 没人解决服装自身穿插）。惩罚形式从 loss → 解析 SDF+push-out → 顶点到身体有符号距离。

### Runtime 量级跃升
交互 → 实时 → GPU 毫秒 → CPU 微秒。省下的不是骨骼层，而是"动态产生 + 身份适配"的开销——**调制把身份计算挪到逐 identity 一次性**（FiLM / 双层调制）是 2026 两篇共同的工程关键，Pan 的 RBF 是运行时算、无此概念。

## 谱系结构观察（重要）

- **HyperBones / PhySkin 的参考文献都只引 [[Pan2022-VirtualBones]]、完全不引 [[Jin2024-RopeChainQNN]]**（已核对两篇 References）。
- 因此"骨骼驱动布料"2024 后分叉为**两支独立谱系**：
  - **Stanford/Fedkiw 支**：[[Jin2024-RopeChainQNN]]——rope chain 物理产动态（作者与 Pan 2022 不重叠）；
  - **Meta Reality Labs 支**：[[HyperBones2026]] / [[PhySkin2026]]——自监督蒸馏 / 准静态。
- 两支**互相没引用**，是各自独立诊断 Pan 病根后给出的不同处方——典型的"研究收敛"：同一问题被不同团队独立分头治疗。

## 关联

- 起点：[[Pan2022-VirtualBones]] · 原文对比：[[Compare-Pan2022-vs-Jin2024]]
- 概念：[[VirtualBones]] · [[RopeChain]] · [[QNN]]
- 索引：[[cloth-animation/_index]]
