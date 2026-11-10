---
type: raw-article
title: TailorNet - Predicting Clothing in 3D as a Function of Human Pose, Shape and Garment Style (2020)
created: 2026-10-09
updated: 2026-10-09
tags: [cloth-animation, pose-space, mixture-of-experts, high-low-frequency, pca-style-space, cvpr-2020]
summary: Patel 等（MPI）CVPR 2020 论文：首个联合建模 pose/shape/style 的数据驱动服装模型。核心发现"任何组合训练样本的尝试都会抹平高频"，解法是高低频分解——低频单一 MLP 回归三因子、高频用 K=20 个 style-shape 专属 pose 模型经窄带宽 RBF 核凸组合；1782 静态姿态×9 形状，55800 帧公开数据，1-2ms/帧；skirt 挂根关节处理。
---

# TailorNet: Predicting Clothing in 3D as a Function of Human Pose, Shape and Garment Style (2020)

> 原文 PDF：`TailorNet-2020.pdf`（本目录）；提取文本：`TailorNet-2020.txt`。
> 中文译本：[[TailorNet-2020-中文翻译]]。
> arXiv: 2003.04583v2 · CVPR 2020 · 模型/数据：https://virtualhumans.mpi-inf.mpg.de/tailornet/
> 谱系定位：[[Pan2022-VirtualBones|Pan 2022]] 的**高低频分解与 RBF 混合两个核心设计的直接出处**；被 Pan2022/AnchorDEF 作为对比基线 · 全景见 [[BoneDrivenCloth-CitationMap]]

## 论文信息

| 项 | 内容 |
|---|---|
| 作者 | Chaitanya Patel*, Zhouyingcheng Liao*, Gerard Pons-Moll |
| 机构 | 马普信息学研究所（MPI Informatics） |
| 数据 | Marvelous Designer 模拟；T 恤/Shirt/Pants/Skirt 四类；1782 姿态 × 9 形状 × 25 款式 → 55800 帧（公开） |
| 对比基线 | Santesteban 2019（VTO [46]）、DRAPE [17]、GarNet [18]、DeepWrinkles [29] |

## 核心机制一句话

服装变形 = SMPL 去摆位空间下的位移 $D(\theta,\phi)$，拆成**低频**（单一 MLP 直接回归 pose+shape+style，平滑但准）+ **高频**（$K=20$ 个 style-shape 专属 pose-MLP 的输出，经窄带宽 RBF 核按"合身度相似度"凸组合）——高频模型只混合相近原型，避免"组合样本抹平皱纹"。

## 关键设计

- **核心假设**：任何（哪怕非线性）组合训练样本的尝试都会抹平高频细节 → 直接回归 pose+shape+style 定量准、定性平滑失真
- **高低频分解**：拉普拉斯平滑（扩散流，λ=0.15、80 迭代）拆 $\hat{G}_{LF}/\hat{G}_{HF}$；不分解直接混合原型输出同样丢细节（图 6 消融）
- **混合核**：$\Psi(\phi,\phi_k)=\exp(-\mathrm{dist}(g(\phi),g(\phi_k))/\sigma_x)$，$g$ 是 style-shape→A-pose 位移的浅 MLP——**用"合身度"代理 style-shape 相似度**，免去对每个组合模拟数据；$\sigma_x$ 小带宽只混相近原型
- **K=20 原型贪心选取**：以凸组合近似误差最大者迭代入池
- **款式空间**：真实数字衣柜 PCA + 模拟交替 2 轮（保物理约束），前 2 主成分=尺码/袖长
- **MLP 够用**：固定 style-shape 下 MLP（14.5/10.1mm）优于 UV Decoder（15.9/11.4）与 Graph CNN（16.1/11.7）

## 数据协议（对准静态路线最重要）

- 1782 静态 SMPL 姿态串成一个序列：canonical 出发贪心转移最近姿态；**插入线性插值中间姿态 + 每段松弛数帧——显式避免动态效应，全数据集即"准静态标签"**
- **移除自穿透帧**（PBS 失败清洗）
- 尽管只有静态姿态训练，AMASS 序列上输出时序连贯（"不建模动态，结果依然连贯"）

## 主要结果与局限

- 混合模型 vs 直接回归基线：定量略好（10.2 vs 10.6mm），定性显著好（保皱纹）
- 平均误差 11mm 级；宽松款最难（14.5mm）
- 1–2ms/帧（GTX 1060），比 PBS 快 1000×；可微
- **Skirt 的处理**：不跟随 SMPL 拓扑 → 模板挂 SMPL 根关节——这正是 [[Pan2022-VirtualBones|Pan 2022]] 批评其"连衣裙只剩轮廓、丢褶皱"的原因；所有指标上被 Pan2022 与 [[RawSource/articles/AnchorDEF-2023|AnchorDEF]] 反超
- 无物理属性/软组织敏感性；无碰撞保证（仅后处理去相交）

## 关联

- 下游直接继承：[[Pan2022-VirtualBones]]（拉普拉斯高低频 ← 本文扩散流；K=8 参数 pivot RBF ← 本文 K=20 style-shape RBF 混合）
- 批评者：[[RawSource/articles/AnchorDEF-2023|AnchorDEF]]（"插值借来的身体蒙皮权重丢细节"）、Pan 2022
- 项目对照：[[SkirtQNN-Design]]——本文的"插值中间姿态+松弛帧"准静态数据协议与方法 0/2（静止段挖掘/物理松弛精修）同构；"静态训练却时序连贯"是准静态赌注的早期实证
