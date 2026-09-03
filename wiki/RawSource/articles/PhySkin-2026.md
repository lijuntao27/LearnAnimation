---
type: raw-article
title: "PhySkin: Physics-based Bone-driven Neural Garment Simulation (2026)"
created: 2026-09-02
updated: 2026-09-02
tags: [cloth-animation, skinning, quasi-static, self-supervised, arxiv-2026]
summary: Meta Reality Labs 2026 论文通读：准静态 CPU 级骨骼驱动 draping，FPS 节点 + heat-method 可训练蒙皮权 + 双层调制（Transformer 点编码身体、MLP 调姿态），物理自监督（StVK/弯曲/碰撞/重力），单线程 ~59µs、~1.7 万 FPS，服装/身体拓扑无关、无动态。原文 PDF 与全文提取稿已入仓（见下）。
---

# PhySkin: Physics-based Bone-driven Neural Garment Simulation

> 论文通读记录。原文 PDF：`PhySkin-2026.pdf`；全文提取稿：`PhySkin-2026.txt`（均在本目录）。arXiv:2603.27013v1, 2026-03-27。
> 精炼总览：[[PhySkin2026]]

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | PhySkin: Physics-based Bone-driven Neural Garment Simulation |
| 作者 | A. Srivastava (IIIT-H，实习于 Meta)、Hsiao-yu Chen、Ryan Goldade、Philipp Herholz、Zhongshi Jiang、Gene Wei-Chin Lin、Lingchen Yang、Nikolaos Sarafianos、Tuur Stuyck、Egor Larionov（Meta Reality Labs） |
| 发表 | arXiv:2603.27013v1 (cs.GR)，2026-03-27 |
| 定位 | 准静态版"减空间神经布料"，面向低算力/单线程 CPU；与 HyperBones 同程序，对应头发作 Quaffure [54]（同作者，CVPR 2025） |
| 身体模型 | **in-house SMPL-like** 参数化身体（比标准 SMPL 更丰富的体型多样性 + 肌肉等表面细节），§4.2；128 根层级人形骨骼 rig |
| 服装数据 | GarmentCodeData [28]；另在 3DBiCar [40]（卡通角色）上验证多样体型 |

## 一句话

主动放弃动态，做"减空间准静态 draping"：节点驱动 + 双层调制 + 网格能量自监督，单线程 CPU 微秒级，且服装/身体拓扑全无关。

## 核心管线（三层网络）

```
每 identity 一次:  Hyper Modulator ξ_hyper
                    canonical 身体网格采 4096 点+法线 → 位置编码(PE)
                    → standard Transformer encoder(4 head) 点 token
                    → Self-Attention pooling → 全局身体形状码
                    → residual-MLP → FiLM(scale/shift) 调 MLP_θ 激活

每帧:
  姿态 θ(120 维) → Pose Modulator MLP_θ(4×512) → 调制信号
                → Node Deformer MLP_χ(4×128)   ← 被 MLP_θ 调制
                   输入: LBS 后全部节点变换 {χ^skin_i}
                   输出: 每节点修正 Δχ
  两层 LBS 蒙皮出网格
```

## 关键机制

### 节点减空间 + 两层 LBS
- $m$ 个仿射节点 $\chi_i\in\mathbb{R}^{4\times4}$，**FPS 采样在每个 canonical-pose 服装面上**（§3.2 写死）；$\bar\chi$ 的构造：FPS 采样位置给平移列（最后一列），线性部分设**单位阵**。
- 蒙皮权 heat method 初始化 $w_{ij}=\exp(-d_{ij}/\sqrt{n/m})$（$d_{ij}$ 为 node→vertex 测地距离），且**权重本身进训练**（表 3：128 节点无权重训练 $\epsilon_e$=4.266 → 有权重训练 3.074）。
- 服装顶点变换
$$\mathrm{LBS}_\chi(x;w,\chi)=\sum_{i=1}^m w_i\,\chi_i\,\bar\chi_i^{-1}\,x,\qquad \sum_i w_i=1$$
- **身体→节点**先做一次 LBS：$\chi^{skin}=\mathrm{LBS}_\theta(\chi;\tilde w,\theta)$，$\tilde w_{i,k}$ 为每节点 $i$ 到**最近**骨骼 $k$ 的逆平方距离、其余骨骼为 0——避免直接学 canonical→posed 大变换，只学修正量 $\Delta\chi$。
- 128 服装节点为最终选择（表 3：256 几乎无增益但 LBS 开销翻倍 +9.03→+18.11µs）。注意：身体骨骼也是 128 根，两者数字巧合但含义不同。

### 双层调制 = 运行时零成本的身份/姿态解耦
- **Hyper Modulator**：身体网格**点采样**（不是 PCA 混形）→ 点 token 与**拓扑无关**。§4.2 明确："unlike existing methods, for model-agnosticism, we only sample the surface shape information for body shape encoding, and do not rely on PCA blendshapes"——这是 body topology independent 的根基。
- **Pose Modulator** 只在姿态变化时跑；Node Deformer 极小（4×128）。两 MLP 合计 ~59µs/帧、16,891 FPS（表 2：MLP_χ 12.5µs + MLP_θ 46.7µs，C++ 单线程，AMD Ryzen Threadripper PRO 7995WX）。body encoder 逐 body 摊销，不计入每帧。
- **FiLM**：身体形状码经 residual-MLP 出 scale/shift，"scale and shift the activation layers of MLP_θ"（§3.3）。identity 不进数据流，而是改写姿态网络的激活分布——每帧零增量成本。

### Hyper Modulator 的内部边界（原文写到哪、没写到哪）
- **[明确]** 采 4096 点 + 法线 → PE → standard transformer encoder(4 head) → Self-Attention pooling → residual-MLP → FiLM 参数。
- **[原文未详述]** PE 的**具体形式与频率**——原文只写 "apply positional encoding"；引 [43] Modulated Periodic Activations (Mehta ICCV 2021)，暗示方向为 periodic/Fourier features 类，但超参未给。
- **[原文未详述]** Transformer 的 **model dim 与层数**——只写 "standard transformer encoder layer with 4 attention heads"。
- **[原文未明写]** body 采 4096 点的**具体算法**——节点那边明确 FPS，body 采样未明说，最可能同源（FPS/Poisson 类均匀采样），但需以原文/附录为准。

### 物理自监督（无需仿真数据）
- 网格仅用于训练期算能量；推理表示**可换成 Gaussian/点云**（蒙皮权可迁移即可）→ 服装拓扑无关。
$$L_{phys}=\lambda_1 E_{StVK}+\lambda_2 E_{dihedral}+\lambda_3 E_{collision}+\lambda_4 E_{grav}$$
  - $E_{StVK}$：St. Venant-Kirchhoff 面内拉伸 [3]；
  - $E_{dihedral}$：二面角弯曲（Discrete Shells [15]）；
  - $E_{collision}$：距身体面 **0.3cm** 内按穿入距离平方罚；
  - $E_{grav}=\sum_k g\,m_k\,y_k$，$g=9.81\,\mathrm{m/s^2}$，$m_k$ 由织物密度算；
  - $\lambda_1\dots\lambda_4$ 基于标准材料属性**预计算**（§3.4，具体值未给）。
- **canonical pose = A-pose**（§3.4 "centered around the A-pose"）。姿态采样从 $0.1\times$ 关节极限起，每 100 epoch 放大 $0.1\times$，到真实范围——避免初始 draping 差时巨大接触罚带崩训练。

## 训练与数据
- 实现：PyTorch + NVIDIA Warp [41]；训练 ~32 小时 / 单 NVIDIA A100。
- body：in-house SMPL-like 参数化身体（更丰富体型 + 肌肉表面）；**不用 PCA blendshapes**，只用表面点采样编码身体形状。
- poses：AMASS 数据集，沿用 HOOD [14] 的 split（§3.4 课程采样）。
- garments：GarmentCodeData [28]；无任何仿真服装数据。
- 另在 3DBiCar [40]（卡通/任意比例体型）上单独训练验证 canonical pose 下的多样体型适配（因 3DBiCar 无动作数据，不训练动作序列）。
- 自定义 C++ 实现 MLP_θ/MLP_χ 前向以换 CPU 高效推理。

## 结果
- **128 节点最优**（表 3）；蒙皮权可训练 > 单纯加节点。
- 服装/身体拓扑无关 + 重网格鲁棒（图 5：HOOD 换分辨率即崩，它没事）；换 body topology 也 OK。
- 表 4（drape quality，AMASS CMU test split）：
  | 方法 | $\epsilon_e$↓ | $\epsilon_a$↓ | $\epsilon_c$↓ |
  |---|---|---|---|
  | GAPS [8] (1 step) | 4.231 | 4.655 | 0.153 |
  | HOOD [14] (1 step) | 6.482 | 5.993 | 0.182 |
  | HOOD [14] (10 steps) | 2.987 | 3.061 | 0.125 |
  | Ours (1 step) | 3.074 | 3.193 | **0.119** |
  - **穿入率全场最低**，strain 与 HOOD 10 步 roll-out 相当，但快几个数量级。
- 特征对比（表 1）：同时满足 自监督✓ + 服装拓扑无关✓ + 身体拓扑无关✓ + CPU>9000fps✓ 的唯一方法。
- 自监督 vs 监督（§4.3）：用 1k pose 逐顶点 Adam 1000 步最小化 Eq.1 仿真伪数据做 L2 监督，结果与自监督几乎一致（图 3 Hausdorff 距离示差异很小）——证明物理自监督等价且免仿真。

## 局限（原文 §5 只列两点）
- **Generalization**：每服装需单独训练，不跨服装零样本（点名想接生成式服装自动管线）。
- **Dynamics**：无动态，游戏/VR 快速运动需动态（future work，称可借 [4,36,50] 扩展 pose modulator）。
- 注：方法中 $E_{collision}$ 只针对身体面，未显式建模布料 self-collision；但这是方法描述层面，非作者自述局限。

## 关联
- 精炼总览：[[PhySkin2026]]
- 前驱：[[RawSource/articles/Pan-2022-VirtualBones]]（骨骼驱动源头，[47]）
- 并行支线：[[RawSource/articles/Jin-2024-LooseFittingClothing]]（互不引用）
- 概念：[[VirtualBones]]、[[QNN]]（准静态思想同源：学"状态本身"）、[[RopeChain]]、[[PCA]]
- 同作者头发前作：Quaffure [54]（real-time quasi-static neural hair，CVPR 2025）
- 兄弟作：HyperBones（动态版）见 [[HyperBones2026]]
