---
type: raw-article
title: "HyperBones: Realtime Bone-driven Neural Garment Simulation with Hypernetwork Conditioning (2026)"
created: 2026-09-02
updated: 2026-09-02
tags: [cloth-animation, virtual-bones, self-supervised, distillation, arxiv-2026]
summary: Meta Reality Labs 2026 论文通读：bone-driven 自监督实时布料，[A] 运行时粗→细（Bone-Net+UV Conv-MLP）/ [B] canonical 身体-服装编码 / [C] MeshGraphNet 教师蒸馏；AMASS 姿态零仿真数据，6 服装 GPU ~1.1ms/帧。
---

# HyperBones: Realtime Bone-driven Neural Garment Simulation with Hypernetwork Conditioning

> 论文通读记录。原文 PDF：`HyperBones-2026.pdf`；全文提取稿 `HyperBones-2026.txt`（均在本目录）。
> 精炼总览：[[HyperBones2026]]

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | HyperBones: Realtime Bone-driven Neural Garment Simulation with Hypernetwork Conditioning |
| 作者 | A. Srivastava (IIIT-H), Hsiao-yu Chen, Ryan Goldade, Philipp Herholz, Zhongshi Jiang, Gene Wei-Chin Lin, Lingchen Yang, Nikolaos Sarafianos, Tuur Stuyck, Doug Roble (Meta Reality Labs), Avinash Sharma, Egor Larionov |
| 发表 | arXiv:2605.20460v3 (cs.GR)，2026-05 |
| 定位 | 动态版"减空间神经布料"；同程序 Quaffure/NeuralLocks（头发）的布料对应 |

## 一句话

把"大网格神经积分"蒸馏成"骨骼驱动粗形 + UV 卷积补细形"，零仿真数据自监督训练，GPU ~1ms/帧，比 HOOD 快 25×+。

## 核心管线（三模块）

| 模块 | 时机 | 作用 |
|---|---|---|
| [A] Pose-Conditioned Deformation | **每帧**（唯一运行时） | LBS(身体→骨骼)→ Bone-Net 修正骨骼 → LBS(骨骼→服装) → UV Conv-MLP 补残差 |
| [B] Body/Garment Encoding | 每 identity 一次 | canonical 身体-服装-骨骼图 → 图编码器 → 形状码 z / 蒙皮权修正 Δw / UV 特征 F_UV |
| [C] Temporal Integration | **仅训练**（训完丢弃） | MeshGraphNet(HOOD 式) 逐顶点物理积分，做 [A] 的教师 |

## 关键机制

### 虚拟骨骼
- canonical 穿好裙网格上 **FPS 采样**（在服装面上，尊重版片/褶皱轮廓），$B$ 根。
- Skeleton→Bones：每骨继承最近身体点的 SMPL 权重，先被身体刚性带跑
$$\tilde{\mathbf p}_b=\sum_{k=1}^K w^{\text{SMPL}}_{bk}\,\mathbf T_k\,\mathbf p^c_b$$
- Bones→Garment 权重初始为 geodesic 高斯 $\exp(-d_{vb}/\sigma)$（沿服装面测地，避免薄版片两侧穿过体内耦合）。

### [A] Stage1：Bone-Net（无 GRU）
- 输入：3 帧窗口骨骼位置 + 一阶速度差（服装参考系内，比关节角差分更强）。
- 输出：每骨修正 $\Delta\mathbf T_b$（旋转用 6D 表示 + 平移），及蒙皮权修正 $\hat w=\operatorname{softmax}(w^{(0)}+\Delta w)$。
- **FiLM 调制**（题名 hypernetwork conditioning）：形状码 $z$ 经 Shape Modulator → 每层 scale/shift
$$h^\ell\leftarrow\gamma_\ell(z)\odot h^\ell+\beta_\ell(z)$$
$z$ 逐 identity 预计算，**逐帧零成本**。选 FiLM 而非拼接/超网络：一个紧致码调所有层、参数最省。

### [A] Stage2：UV Conv-MLP（高频）
Stage1 网格投到固定 UV atlas 得位置图 $P^t_{UV}$，拼上静态特征图 $F_{UV}$（接缝/版片/区域刚度先验）+ 姿态嵌入 $\phi^t$ → Conv-MLP → 逐顶点残差 $\delta p_v$；$p_v=\hat p_v+\delta p_v$。

### [B] 编码
身体+服装 A-pose 建 Body-Garment-Bones 图（服装顶点连 k 近邻身体点）→ 消息传递图编码器（**与 [C] 共享权重**）→ 三个头出 z、Δw、F_UV。共享权重使 [C] 的物理梯度回流进 canonical 特征——编码的是"几何暗示的动态行为"而非纯几何。

### [C] 教师 + 为什么物理 loss 不能直接打 [A]
- MeshGraphNet 预测逐顶点加速度，**辛欧拉**积分 $v_v\leftarrow v_v+\Delta t\,a_v$、$p_v\leftarrow p_v+\Delta t\,v_v$，速度是**持久积分变量**。
- 自监督物理 loss：$L_{phys}=\lambda_s L_{stretch}+\lambda_b L_{bend}+\lambda_c L_{collision}+\lambda_i L_{inertia}+\lambda_f L_{friction}$，其中惯性项
$$L_{inertia}=\tfrac1{|V_c|}\sum_v\bigl\lVert p_v^{[C],t}-(2p_v^{[C],t-1}-p_v^{[C],t-2})\bigr\rVert^2$$
- **不能直接对 [A] 算 $L_{phys}$** 的原因：蒙皮下顶点位置是无状态函数 $\hat p_v=g_v(\boldsymbol\theta_t)$ → ① 布料速度是 $K$ 维关节速度的低秩混合，表达不了局部独立速度场；② 身体静止时表观速度恒 0，惯性梯度消失——恰在"急停振荡/沉降/阻尼摆动"（动态最强）时身体常静止。教师保持持久速度使惯性良定。
- 交叉一致性蒸馏：$L_{3D}=L_{MSE}+L_{lap}+L_{interp}$
  - MSE：学生对齐教师积分位置（动量/滞后传入无状态学生）；
  - Laplacian：压稀疏骨骼影响边界处的蒙皮锯齿（Pan 的 LBS artifact 同类问题）；
  - 穿透：$\tfrac1{|V_c|}\sum_v\max(0,\epsilon-d_v)^2$（$d_v$ 到身体有符号距离），对运行时输出独立施加。

### 训练
两阶段：warm-up 500 epoch 只训 [C]（否则初始输出任意、当不了教师）；再联合 ~99.5k epoch 训 $L=L_{phys}+L_{3D}$。数据只取 **AMASS 姿态序列**，**无任何预计算仿真数据**。

## 结果
- 6 件 VTO 服装（Tshirt/Tshirt-Unzipped/Pants/Long-Skirt/Dress/Hooded-Tight-Dress），SMPL 多样体型，holdout AMASS+VTO。
- $\epsilon_e/\epsilon_a$（边/面误差，GAPS 指标）全场最低；紧身衣 $\epsilon_c$（穿入%）略输 HOOD（其图碰撞更强）。
- 推理 1.07–1.31 ms/帧（Table 2）vs HOOD 25–50 ms → **~25×**；论文标题称 300+ FPS。
- 消融：128 骨骼最优（32 穿透、64 尚可）；**Conv-MLP 缺不得**（纯骨骼自由度不足→穿透+三角形退化）；$L_{lap}$ 关键；直接对 [A] 施物理监督早期崩（蒙皮穿入→几何退化）。
- GAPS（GRU 运动态）长序列累积漂移；HOOD 稳但贵；本方法稳且快。

## 局限（作者自述）
- 只支持**固定 6 服装**的单模型，不能对新生成服装零样本（Limitations 点名想接 text/scan 生成服装的自动管线）。

## 关联
- 精炼总览：[[HyperBones2026]]
- 前驱：[[RawSource/articles/Pan-2022-VirtualBones]]（骨骼驱动源头，其 GRU 路线被本方法用教师蒸馏替代）
- 并行支线：[[RawSource/articles/Jin-2024-LooseFittingClothing]]（rope chain 物理产动态，两线互不引用）
- 概念：[[VirtualBones]]、[[QNN]]、[[RopeChain]]
- 兄弟作：PhySkin（准静态版）见 [[PhySkin2026]]
