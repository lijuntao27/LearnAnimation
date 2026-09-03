---
type: concept
title: PhySkin 2026 — 准静态调制 + CPU 微秒骨骼驱动 Draping
created: 2026-09-02
updated: 2026-09-02
tags: [cloth-animation, skinning, quasi-static, self-supervised, arxiv-2026]
summary: Meta 2026 准静态 CPU 骨骼驱动 draping：FPS 节点+可训练蒙皮权+双层调制(Transformer 身体/MLP 姿态)，物理自监督(StVK/弯曲/碰撞/重力)，单线程微秒级，服装拓扑无关、无动态。
---

# PhySkin 2026 — 准静态调制 + CPU 微秒骨骼驱动 Draping

> Srivastava et al., arXiv:2603.27013, 2026（Meta Reality Labs + IIIT-Hyderabad）。原文全文：[[RawSource/articles/PhySkin-2026]]（PDF 在 `RawSource/articles/`）。

## 一句话

**主动放弃动态**，做"节点减空间 + 准静态 draping"：FPS 采样节点 + 可训练蒙皮权 + 双层调制，网格能量自监督（零仿真数据），**单线程 CPU 微秒级（~1.7 万 FPS）**，且服装/身体拓扑全无关——面向低算力设备的极致实时。

## 问题定位

日常穿着大部分时刻动态可忽略，故砍掉时序换 CPU 极致实时。与 [[HyperBones2026]] 同程序、不同取舍：HyperBones 学动态上 GPU，PhySkin 准静态上 CPU。

## 方法（三层网络）

```
每 identity 一次: Hyper Modulator ξ_hyper（Transformer 点编码身体形状）
每帧:            Pose Modulator MLP_θ → 调 Node Deformer MLP_χ → 两层 LBS 蒙皮
```

| 组件 | 结构 | 职责 |
|---|---|---|
| Hyper Modulator | 身体网格采 4096 点+法线→PE→Transformer(4 head)→Self-Attention pooling→MLP | 读身体形状，FiLM 调制 MLP_θ；点 token → **身体拓扑无关** |
| Pose Modulator | MLP 4×512 | 姿态 $\theta\in\mathbb{R}^{120}$ → 调制信号 |
| Node Deformer | MLP 4×128（被调制） | 输入 LBS 后节点变换 $\{\chi^{skin}_i\}$，出修正 $\Delta\chi$ |

## 关键机制

### 骨骼驱动是基底，不是装饰（先读这节）

> ⚠️ **两个 128 别混**——论文里有两个"128"，性质完全不同：

| | 128 根**身体骨骼** bones | 128 个**服装节点** garment nodes |
|---|---|---|
| 是什么 | 美术层级 humanoid rig（§3.1 写死）| canonical 服装面 FPS 采样点 |
| 性质 | 真骨骼（有层级、有语义）| **虚拟骨骼**（无层级、无语义，见 [[VirtualBones]]）|
| 作用 | 决定**人的 pose**（$\theta\in\mathbb{R}^{120}$ 是其关节状态）| 服装/裙摆的**控制点** |
| 128 来源 | rig 固定 | 表 3 消融（32/64/128/256）选出的最优 |

数字巧合，不是一回事。

**完整数据流（三层骨骼驱动嵌套）**：

```
───── identity 线（每体型一次，与骨骼驱动正交）─────
canonical 身体面采 4096 点 → Hyper Modulator → 形状码 z
  → FiLM (γ,β) → 调制 MLP_θ          ※只调网络参数，不驱动裙摆

───── pose 线（每帧）─────
θ (120维 = 128 身体骨骼的关节状态)
  │
  ├─[LBS·身体蒙皮权]────→ 身体网格 B(t)              ①身体被骨骼驱动
  │
  └─[LBS·节点→骨骼权 w̃]─→ 服装节点初始位 χ^skin      ②节点被身体骨骼驱动（第一层 LBS）
                             │
 MLP_θ(θ) → MLP_χ → Δχ                              ※网络只学修正量
                             │
                        χ = χ^skin + Δχ
                             │
                   [LBS·服装节点权 w，可训练]         ③裙摆被虚拟骨骼驱动（第二层 LBS）
                             ▼
                       服装/裙摆顶点
```

**关键结论**：裙摆顶点**不是**被身体骨骼直接 LBS 驱动，而是被 128 个**虚拟骨骼（服装节点）**驱动；这些虚拟骨骼又先被身体骨骼 LBS 带到初始位。4096 点属**正交的 identity 编码线**，不参与骨骼驱动链条。

骨骼驱动在本方法的**五个层面**：
1. **输入即骨骼**——驱动信号是 128-bones rig 的关节状态 $\theta$，非顶点位移/视频/latent；
2. **两层 LBS 嵌套**——身体骨骼→服装节点→裙摆顶点；
3. **蒙皮权可训练**——传统骨骼驱动权重固定（美术刷/SSDR 反解），此处第二层权重进训练（表 3 证明收益 > 加节点）；
4. **网络只学修正**——骨骼 LBS 闭式算掉"跟身体走"的大变换，网络只补 $\Delta\chi$；**这是"微秒级"比 FiLM 摊销更底层的速度根基**；
5. **路线选择**——选骨骼驱动（LBS 闭式、快）而非物理仿真（积分、慢）；传统骨骼驱动快但物理差，本方法 = 保留其快 + 神经修正补物理。

去掉骨骼驱动，"微秒级 + 物理合理"的整个设计立不住。命名沿用 [[Pan2022-VirtualBones]] 的 "bone-driven" 词族。

### 减空间 + 两层 LBS
$m$ 个仿射节点 FPS 采样，蒙皮权 **heat method** 初始化并**进训练**：

$$\mathrm{LBS}_\chi(x;w,\chi)=\sum_{i=1}^{m}w_i\,\chi_i\,\bar\chi_i^{-1}x,\qquad w_{ij}=\exp\!\bigl(-d_{ij}/\sqrt{n/m}\bigr)$$

身体→节点先做一次 LBS（$\tilde w$ 为每节点到最近骨骼的逆平方距离）得到 $\chi^{skin}$，网络只学修正量 $\Delta\chi$——避免直接学 canonical→posed 大变换。

### 双层调制 = 运行时零成本身份/姿态解耦
- **身份（身体形状）**只经 Hyper Modulator 一次性算，不随帧重复——这是网络能缩到"微秒"的关键工程决策（对比：monolithic 网络每帧都要再读一遍身体形状，白耗算力）。
- **姿态**只经小网络；Node Deformer 极轻（4×128）。
- 运行时（表 2）：MLP_χ 12.5µs + MLP_θ 46.7µs，**单线程 C++ 16,891 FPS**（不含蒙皮；body encoder 逐 body 摊销）。

### 物理自监督（网格只用于训练期算能量）
$$L_{phys}=\lambda_1 E_{StVK}+\lambda_2 E_{dihedral}+\lambda_3 E_{collision}+\lambda_4 E_{grav}$$

- $E_{StVK}$ 面内拉伸、$E_{dihedral}$ 二面角弯曲、$E_{collision}$ 距身体 0.3cm 内平方穿入罚、$E_{grav}=\sum g\,m_k y_k$。
- 推理表示**可换成 Gaussian 点云**（蒙皮权可迁移即可）→ 拓扑无关。
- 姿态采样从 0.1× 关节极限逐步放大（每 100 epoch +0.1×），防初始 draping 差的接触罚带崩训练。

## 训练产物与运行时闭环

### 训练输出 = 四组参数（§3.4）

| 训练输出 | 是什么 |
|---|---|
| **$w$** | 服装节点→顶点蒙皮权（heat method 初始化 + **进训练**）|
| **MLP_χ** | Node Deformer 权重（4×128）|
| **MLP_θ** | Pose Modulator 权重（4×512）|
| **ξ_hyper** | Hyper Modulator 权重（Transformer + residual-MLP）|

**不训练/不输出**：$\tilde w$（节点→最近骨骼逆平方距离，固定）、$\bar\chi$（FPS 位置给定、线性部为单位阵）、身体蒙皮权（rig 自带）。不输出网格/仿真数据——**只输出参数**。官方仓库**未发布预训练 checkpoint**，须自训。

### 推理两阶段（§3.5）

**阶段 A｜每 identity 一次（摊销）**：canonical 身体网格采 4096 点 → ξ_hyper → 形状码 $z$ → FiLM $(\gamma_\ell,\beta_\ell)$ → **固化进 MLP_θ**。不参与每帧前向。

**阶段 B｜每帧（CPU ~59µs）**：
```
θ ─①[LBS·身体蒙皮权]→ 身体网格 B(t)        （碰撞参考 + 渲染）
  └②[LBS·节点→骨骼权 w̃]→ χ^skin           第一层 LBS，骨骼级
     ③ MLP_θ(θ) → 调制信号
     ④ MLP_χ(χ^skin, 被③调制) → Δχ         ※修正在【节点】层，非顶点层
     ⑤ χ = χ^skin + Δχ
     ⑥ [LBS·服装节点权 w]→ 服装顶点         第二层 LBS，顶点级
```
①②⑥ 均 LBS 闭式解；③④ 两个小 MLP = 12.5+46.7µs。

### 训练 vs 推理

| | 训练 | 推理 |
|---|---|---|
| 物理 loss | ✅ 反向传播 | ❌ **不算**（物理只作训练监督）|
| 表示 | 网格（算能量梯度）| 可换 Gaussian/点云（$w$ 可迁即可）|
| body encoder | 每步跑 | 每 identity 一次 |
| 设备 | A100 GPU | **CPU 单线程** |

## UE 落地映射

> ⚠️ **不是 Morph Target / BlendShape**。修正作用在**骨骼 transform 层**（$\Delta\chi$ 是 4×4 仿射），不是顶点偏移；且 MLP 在**两层 LBS 之间**，不是"LBS 出网格后补顶点偏移"。修正经第二层 LBS 蒙皮权自然平滑传播到顶点——这既是速度根基（128 节点 vs 几万顶点）也是物理平滑性来源。

| PhySkin 概念 | UE 实现 |
|---|---|
| 128 服装节点（虚拟骨骼）| Skeleton 内新增 128 根"服装骨骼" |
| $\tilde w$（只跟最近一根骨骼）| 每根服装骨骼 **parent = 最近身体骨骼**（单骨骼绑定，UE 原生）|
| 第一层 LBS（身体→节点）| UE 父子骨骼自动跟随 → $\chi^{skin}$ |
| MLP_θ + MLP_χ → $\Delta\chi$ | **UE NNE**（Neural Network Engine, 5.3+）或 ONNX runtime |
| $\chi=\chi^{skin}+\Delta\chi$ | 服装骨骼 LocalTransform offset |
| 第二层 LBS（节点→顶点，权 $w$）| **UE 标准 skeletal mesh skinning**（服装顶点绑服装骨骼，刷 $w$）|

落地形态：服装骨骼作为身体骨骼子骨骼（实现 $\tilde w$ 单骨骼跟随），每帧在 Animation Blueprint / Control Rig 跑 NNE 推理出 $\Delta\chi$，apply 到服装骨骼 transform offset，其余交给 UE 原生 skinning。**无需顶点级后处理、无需 Morph Target**。

概念对应物是 **Control Rig 骨骼修正 + 标准蒙皮**，而非 Morph Target：

| | UE BlendShape (Morph Target) | PhySkin $\Delta\chi$ |
|---|---|---|
| 修正对象 | 顶点偏移 | **骨骼 4×4 变换** |
| shape 来源 | 预定义（美术/PCA）| **MLP 实时生成** |
| 传播 | 直接加顶点 | 经 LBS 蒙皮权传播 |
| 自由度 | 几万顶点×3 | 128 节点×(4×4) |

（"加性修正 + shape 由 MLP 生成"这层类比成立；"改顶点"这层不成立。）

## 数据与结果

- 128 节点最优（256 几乎无增益）；**蒙皮权可训练**收益 > 单纯加节点。
- 表 4：单步 $\epsilon_e$3.07 / $\epsilon_c$**0.119（穿入率最低）**，质量与 HOOD 10 步 roll-out 相当，但快几个数量级。
- 表 1：**唯一**同时满足 自监督 ✓ + 服装拓扑无关 ✓ + 身体拓扑无关 ✓ + CPU >9000fps ✓。
- 重网格/降分辨率鲁棒（HOOD 换分辨率即崩）。
- 自监督 ≈ 用 1k pose 逐顶点 Adam 仿真伪数据监督的结果（§4.3）。

## 局限（原文 §5 只列两点）

- **完全无动态**（快速运动甩摆需另接动态管线，future work）；
- 每服装单独训练、不跨服装零样本（点名想接生成式服装管线）。
- 注：$E_{collision}$ 只针对身体面，未显式建模布料 self-collision——属方法描述层面，**非作者自述局限**。

## 与方案/谱系的关系

- "学状态本身而非状态转移"与 [[QNN]] 准静态思想同源，只是用**节点减空间 + 物理能量自监督**，连 5000 帧仿真数据都不需要；
- 无旋转骨骼信息让 LBS/neural skinning 是唯一出路（同 [[Jin2024-RopeChainQNN]] 的判断）；
- 动态孪生：[[HyperBones2026]]。

## 关联

- 原文：[[RawSource/articles/PhySkin-2026]]
- 对比：[[Compare-BoneDrivenCloth-Lineage]]
- 概念：[[VirtualBones]] · [[QNN]] · [[RopeChain]] · 孪生：[[HyperBones2026]]
