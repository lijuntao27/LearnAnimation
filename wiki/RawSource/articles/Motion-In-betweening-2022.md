---
type: raw-article
title: Motion In-betweening via Two-stage Transformers (2022)
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, transformer, animation, siggraph-asia-2022]
summary: Qin/Zheng/Zhou 2022 ACM TOG 论文的全面解读记录，含两阶段 Transformer 框架、两种自定义位置编码、损失设计、LAFAN1 实验结果与配套开源代码库的结构对应与实现差异。
---

# Motion In-betweening via Two-stage Transformers (2022)

> 本文是对论文的通读解读记录（原文 PDF：`Motion-In-betweening-2022.pdf`，16 页，在本目录；提取文本：`Motion-In-betweening-2022.txt`；配套仓库：`motion_inbetweening/`）。
> 作为 `TheWiki/motion-inbetweening/` 各知识页的原文来源。

## 论文信息

| 项 | 内容 |
|---|---|
| 标题 | Motion In-betweening via Two-stage Transformers |
| 作者 | Jia Qin（浙江大学 CAD&CG 国家重点实验室 + 网易伏羲 AI 实验室）、Youyi Zheng、Kun Zhou |
| 发表 | ACM TOG 41(6), Article 184, SIGGRAPH Asia 2022 |
| DOI | 10.1145/3550454.3555454 |
| 代码 | github.com/victorqin/motion_inbetweening |

## 1. 问题定义与动机

**Motion in-betweening（运动补间）**：给定若干**上下文帧**（角色过往姿态）+ 一个**目标帧**（最终姿态），生成中间缺失的**过渡动画**；过渡中间可插入用户指定的姿态约束（全约束或部分关节约束）。

痛点与已有方法局限：

- **传统样条插值（Slerp）**：只能产生平滑过渡，缺乏运动细节，动画师手工调整劳动强度大。Ciccone 2019 能优化插值曲线切线，但仍需加额外关键帧。
- **RNN 类（ERD-QV / Robust Motion In-betweening, Harvey 2020）**：逐帧自回归，训练慢推理慢；缺乏全局规划，在"运动预测阶段→逼近目标阶段"交界处产生突变。
- **CNN 类（FCN 全卷积自动编码器，Hernandez 2019 / Kaufmann 2020 / Zhou 2020）**：感受野限制长程依赖建模；输入尺寸常需 2 的幂，变长过渡需 padding，浪费计算或质量不稳。
- **已有 Transformer 方法**：MC-Trans（Duan 2021）用 BERT encoder + 1D 卷积；Δ-Interp（Oreshkin 2022）用完整 Encoder-Decoder 预测 Slerp 之上的 delta。参数多，且对超出训练长度的过渡泛化差。

**本文应对**：只用 Transformer Encoder + 掩码自注意力，两阶段生成，提出两种自定义位置编码，实现轻量、可并行、能泛化到超长过渡。

**贡献**：1) 证明 Transformer Encoder-based 两阶段框架配合 Keyframe PE 与 Learned Relative PE 能鲁棒生成变长补间；2) LAFAN1 上报告 SOTA，超越 RNN/CNN/Transformer 各流派。

## 2. 总体框架（两阶段）

```
输入: 上下文帧 + 目标帧 + (可选)过渡内姿态约束
        ↓
Stage I:  Context Transformer  → 生成"粗略"过渡（把握全局走向）
        ↓
Stage II: Detail Transformer   → 精修细节（消除穿模/脚滑等瑕疵）
```

动机（经验观察）：**单网络容易把过渡"抹平"**（smooth out）；加第二级能恢复运动细节、修复 foot sliding。

两个关键决策：
1. **一次输出整段过渡**（非自回归）+ 全局 self-attention → 具备**全局规划**能力，避免逐帧生成的突变。
2. 训练时随机采样过渡长度 + 特殊位置编码 → 泛化到训练未见过的更长过渡。

## 3. 数据表示

- 骨架：类人 22 关节（LAFAN1）。每帧姿态 = 所有关节**局部旋转** `r_t ∈ R^{J×6}` + **根节点世界位置** `p_t ∈ R^3`。局部位置固定（保证骨骼长度一致）。
- 旋转用 **6D 旋转表示**（Zhou et al. 2019：取旋转矩阵前两列，Gram-Schmidt 正交化还原）。相比四元数，6D 表示**唯一且连续**，利于神经网络训练。
- 维度 `D = J×6 + 3 = 135`。最后做 **z-score 归一化**：`x_t = (x̃_t − μ_x)/σ_x`，统计量用滑动窗口 50 帧、offset 20 帧算。

## 4. Context Transformer（阶段一）

结构：**encoder(MLP) → L 层带相对位置编码的 Transformer Encoder 层 → decoder(MLP)**。注意论文称 encoder-decoder，但这里的 "decoder" 只是两层 MLP，不是自回归解码器。

1. 输入 `x_ctx ∈ R^{T×D}`，缺失帧用占位值填充（根位置在已知帧间线性插值、关节旋转置零）。
2. 拼接**输入掩码** `m_in`（指示哪些值有效）→ 输入维度 135+1=136。
3. encoder：两层全连接 + PReLU（隐藏 512）→ `d_model=512`。
   `h̃_ctx = φ(φ([x_ctx, m_in]W_E^(1)+b_E^(1))W_E^(2)+b_E^(2))`
4. 加 **Keyframe Positional Encoding**：`h_ctx = h̃_ctx + pe_kf`
5. 6 层 Transformer 层，每层 = 掩码多头自注意力（MHSA，带相对位置编码）+ 位置式前馈网络（PFFN），**pre-norm**（LayerNorm 在子层输入前，提高训练稳定性，Chen 2018 / Wang 2019）：
   `h̃_l = h_{l−1} + MHSA(LayerNorm(h_{l−1}))`
   `h_l = h̃_l + PFFN(LayerNorm(h̃_l))`
6. **注意力掩码** `m_atten`：只有上下文帧、受约束帧、目标帧能被 attend（作为 key），占位过渡帧不能（论文 Figure 5a）。
   `Attention(Q,K,V) = softmax((QK^T + S_rel)/√d_K + m_atten)·V`
   `S_rel` 是 Transformer-XL（Dai 2019）式相对位置注入。
7. decoder：两层 MLP → 输出 135 维。`y_ctx = φ(h_L W_D^(1)+b_D^(1))W_D^(2)+b_D^(2)`
8. 从 `y_ctx` 提取预测旋转 `r̂_ctx = R(y_ctx)` 与根位置 `p̂_ctx = P(y_ctx)`。

超参：`d_model=512`, `L=6`, `H=8`, `d_head=64`。附录 D 的超参扫描显示 `d=512` 与 `L=6` 在 45 帧（超训练长度）上最优——`d=1024`/`L>6` 虽在 30 帧内相当但 45 帧显著劣化。

## 5. Detail Transformer（阶段二）

与 Context 结构几乎相同，但：

- **没有 Keyframe PE**——精修任务下"距关键帧距离"不再关键。
- **输入 = Context 的输出**：过渡部分用 `y_ctx` 填充；有约束姿态时只复制未约束关节的值、约束关节保留真值。
- **注意力掩码更宽松**：上下文帧、过渡帧、目标帧可自由互相 attend（Figure 5b）——因为输入里已无占位值。
- **额外输出足部接触** `ĉ ∈ R^{T×4}`（左右脚 + 脚趾，1=触地），`ĉ = sigmoid(C(y_det))`。输出维度 139 = 135+4。可配合 IK 消除脚滑。

## 6. 两种位置编码（核心创新）

**问题**：vanilla Transformer 的正弦绝对位置编码在"过渡长度超训练最大长度"时急剧失效；传统固定尺寸 lookup table 会限制推理长度。

### 6.1 Keyframe Positional Encoding

观察：补间任务中，缺失帧与**已知帧的相对距离**才是关键（近则相关强，远则弱）；这无法从绝对位置编码直接推出。早期实验中"两端各起一个绝对编码再拼接"的方案，一旦过渡超训练长度几帧，质量就迅速劣化。

实现：`p_kf ∈ R^{T×2}` 是整数张量，描述每帧**相对最后一个上下文帧**和**相对目标帧**的位置；过单隐层 512 的 MLP 映射成 `pe_kf ∈ R^{T×d}`：

```
pe_kf = φ(p_kf W_kf^(1) + b_kf^(1)) W_kf^(2) + b_kf^(2)
```

因为输入是**距离值**而非固定索引，任意过渡长度都适用（不用固定查找表，故不限长度）。

### 6.2 Learned Relative Positional Encoding

相对位置编码捕捉序列内任意两项的成对距离，在机器翻译（Shaw 2018）、音乐生成（Huang 2018）上被证明有效。长过渡时任意两帧的相对距离大概率训练中见过，故伪影少。

同样不用固定表，而用两层 MLP 把相对距离 `p_rel ∈ R^{(2T−1)×1}` 映射成变长查找表 `E_rel ∈ R^{(2T−1)×d_K}`：

```
E_rel = φ(p_rel W_rel^(1) + b_rel^(1)) W_rel^(2) + b_rel^(2)
S_rel = skew(Q E_rel^T)
```

`skew(·)` 是 Music Transformer（Huang 2018）的省内存机制。

### 6.3 消融（Table 2）

| 位置编码 | L2P@30 | L2P@45 | L2Q@45 | NPSS@45 |
|---|---|---|---|---|
| pe_sin_abs（正弦绝对） | 1.08 | 3.39 | 1.41 | 0.7045 |
| pe_sin_rel（正弦相对） | 1.07 | 2.66 | 1.09 | 0.3947 |
| pe_sin_rel + pe_kf | 1.05 | 2.08 | 0.96 | 0.3527 |
| pe_rel（学习相对） | 1.07 | 2.00 | 0.93 | 0.3674 |
| **pe_rel + pe_kf（完整）** | **1.04** | **1.94** | **0.91** | **0.3349** |
| ERD-QV（RNN 参照） | 1.28 | 2.24 | 0.94 | 0.3311 |
| ERD-QV w/ pe_kf | 1.23 | 2.15 | 0.97 | 0.3755 |

两种编码是 Transformer 超越 RNN 的关键。把 pe_kf 移植到 ERD-QV 收效不大——因为 RNN 自回归，隐藏状态已携带过去上下文，"距上下文距离"不重要；而"距目标距离"其 `z_tta` 已有。

## 7. 损失函数

**Context Transformer**（`T̃ = t_tgt − t_ctx` 为过渡长度，损失仅在过渡帧上评估）：

- `L_rot`：所有关节局部旋转 L1 —— `(1/T̃)Σ‖r_t − r̂_t‖₁`
- `L_pos`：全局关节位置 L1（经 FK） —— `(1/T̃)Σ‖g_t − ĝ_t‖₁`, `g_t = FK(p_t, r_t)`。因骨架是层级结构，根关节误差比叶关节影响大，评估全局位置可**隐式加权关节朝向**（Pavllo 2020）。
- `L_smooth`：相邻帧全局位置差 L1 —— `(1/(T̃+1))Σ‖ĝ_t − ĝ_{t−1}‖₁`，保证两端平滑
- `L_ctx = α_pos L_pos + α_rot L_rot + α_smooth L_smooth`，权重 `α_pos=0.01, α_rot=1.0, α_smooth=0.005`

**Detail Transformer**：

- `L_c`：足部接触重建 L1
- `L_foot`：脚滑损失 —— `(1/T̃)Σ‖ĉ_t v̂_t‖₁`（触地时脚/脚趾全局速度应≈0）。**必须切断 `ĉ` 的梯度回传**，否则优化会让 `ĉ` 恒为 0 来作弊。
- `L_det = β_pos L_pos + β_rot L_rot + β_c L_c + β_foot L_foot`，权重 `β_pos=0.02, β_rot=1.0, β_c=0.05, β_foot=0.05`

**消融（Table 3）**：`L_pos` 大幅改善 L2P（3.86→1.05）；`L_smooth` 改善 NPSS（消除两端不连续）；`L_c` 单独用时对指标无明显影响，但 `L_c + L_foot` 联合能缓解脚滑并改善全部指标（0.93→0.89）。

## 8. 训练细节

- 窗口 `T = t_ctx + t_max_trans + 2`：多出 2 帧，1 帧留给最大长度时的目标帧，1 帧用于后处理在目标帧算曲率（等价于提供目标帧速度）。
- **预处理**：把最后一个上下文帧移到原点（仅 XZ，Y 不变）并让根节点朝 X+ 轴，使所有过渡起点一致，降低学习复杂度。
- 每迭代**随机采样过渡长度**与受约束帧：约束候选数 = `q×(t_tgt − t_ctx)`，每个以概率 `p` 启用；`p=0.5, q=0.1`。
- mini-batch 32；Adam（`β₁=0.9, β₂=0.999, ε=1e-8`）+ **Noam 学习率调度**（warmup 8000 步）。
- **两阶段顺序训练**：先训 Context，训完**冻结参数**，再用其预测作为输入训 Detail。产出 Table 1 数值的迭代数：Context 181k、Detail 1430k。
- 最多 200 epoch，早停防过拟合。
- **不用 dropout**：实验发现加 dropout 反而使生成动画出现高频噪声般的不连续。推测 dropout 对运动去噪任务或有帮助（输入本就有瑕疵），但补间任务推理时上下文帧不带噪声，故无益。尝试过渐减 dropout 到 0 等策略，与 dropout=0 无显著差异。随机过渡长度采样已足够防过拟合。
- **随机姿态约束采样**对 Detail Transformer 有效（附录 C）：缓解过拟合、改善验证指标。

## 9. 后处理 ψ

Context 输出在进 Detail 前，非过渡帧会回填真值——这可能在过渡两端引入小间隙（即使有 `L_smooth`）。定义逐通道偏移 `δ* ∈ R^D` 加到整段过渡上，使**最后一个上下文帧与目标帧处的运动曲线曲率最小**：

```
ψ(y, δ) = y + δ  （t ∈ [t_ctx, t_tgt)）；  x  （其他）
δ* = argmin_δ [ κ(ψ(y,δ), t_ctx−1) + κ(ψ(y,δ), t_tgt) ] / 2
```

`κ(x,t)` 是 `x` 在帧 `t` 的曲率，实现中用相邻线段夹角近似。**只在 Context Transformer 推理时应用**。目标帧处算 `κ` 需要的额外一帧不需用户手输，可由已有关键帧插值/外推得到。

## 10. 实验

### 10.1 数据集与指标

**LAFAN1**（Ubisoft La Forge 开源动捕数据集）：
- 约 496,672 帧、30Hz（30 帧 = 1 秒），生产级光学动捕
- 5 名演员，含走/跑/攀爬/跳舞等多种动作
- BVH 格式（骨架层级 + 逐帧根位置与关节欧拉角）
- 刻意剔除随机短动作（上下文模糊、难预测），故特别适合补间任务

**Benchmark 设置**（沿用 Harvey 2020）：训练集 Subject 1-4，窗口 42、offset 1 → 404,195 clip；测试集 Subject 5，窗口 65、offset 40 → 2,232 clip。统一 `t_ctx=10`、训练最大过渡 30 帧，评估 5/15/30/45 帧。

**三个指标**：
- **L2P**：全局关节位置的平均 L2 距离（每关节每帧）
- **L2Q**：全局关节旋转（四元数）的平均 L2 距离
- **NPSS**（Gopalakrishnan 2019）：Normalized Power Spectrum Similarity，在频域评估角度差异，衡量整体运动真实性

计算前对运动数据归一化（提取旋转与位置表示的均值/标准差），使指标可跨数据集比较。

### 10.2 定量对比（Table 1，▲为相对本文的劣化幅度）

| 方法 | 参数量 | L2P@5 | L2P@15 | L2P@30 | L2P@45 | L2Q@30 | NPSS@30 |
|---|---|---|---|---|---|---|---|
| Interp.（线性+Slerp） | — | 0.37 | 1.25 | 2.32 | 3.45 | 0.98 | 0.2013 |
| FCN 基线 | — | 0.25 | 0.69 | 1.63 | 4.69 | 0.77 | 0.1524 |
| MC-Trans（local 模式） | — | 0.23 | 0.74 | 1.37 | — | 0.71 | 0.1430 |
| ERD-QV（RNN） | — | 0.23 | 0.65 | 1.28 | 2.24 | 0.69 | 0.1328 |
| Δ-Interp | 40.4M | 0.13 | 0.47 | 1.00 | 3.23 | 0.57 | 0.1217 |
| Ours（T_ctx only） | 10.4M | 0.17 | 0.49 | 1.07 | 2.00 | 0.60 | 0.1238 |
| Ours（T_ctx only, 后处理） | 10.4M | 0.13 | 0.46 | 1.04 | 1.94 | 0.59 | 0.1210 |
| **Ours（完整两阶段）** | 10.4+10.1M | **0.10** | **0.39** | **0.89** | **1.68** | **0.54** | **0.1124** |

- 正常长度平均提升 16%、超长（45 帧）提升 55%；参数量不到 Δ-Interp 一半。
- Table 1 倒数第二行是"Context 单阶段但用上除 `L_smooth` 外全部损失项"的对照——**证明单阶段即使配齐损失也达不到两阶段效果**。
- Δ-Interp 在 45 帧急剧劣化（3.23）：Slerp 只看两个关键帧，可能与上下文矛盾（如上下文暗示左转、Slerp 却右转），delta 修正不过来；且其学习位置编码用 one-hot 过全连接，超长泛化差。

### 10.3 定性观察

- FCN 在短过渡尚可，但超训练长度后生成动作明显丢失细节。
- 相比 ERD-QV，本文关节速度沿时间分布更好。ERD-QV 有根关节位置/旋转突变——因其自回归：远离目标时像做运动预测（输出与目标关系弱），接近目标时才转向目标，两阶段交界处突变。本文三个特性避免此问题：两阶段生成、整段一次输出、自注意力可跨任意距离聚合信息。
- Δ-Interp 的过渡初段接近真值，但当 Slerp 偏离真值后，它到达无法用 delta"修正"Slerp 的点，末段就退化成 Slerp 结果。

### 10.4 掩码消融（Table 4）

| 变体 | L2P@30 | L2P@45 | NPSS@45 |
|---|---|---|---|
| T_ctx 完整 | 1.04 | 1.94 | 0.3349 |
| 去 m_in | 1.04 | 1.95 | 0.3566 |
| 去 m_atten | 1.41 | 2.91 | 0.3846 |
| 都去 | 1.48 | 2.92 | 0.3928 |

去 `m_atten` 最伤——占位值会在每个 Transformer 层污染自注意力。去 `m_in` 影响较小（仅失去区分有效值与占位值的能力）。

### 10.5 四足泛化（Table 5）

在 Zhang 2018 四足数据集（147,541 帧，取约 1/3 即 33,376 帧作测试集）上训练，最大过渡 60 帧（2 秒）。训练集窗口 72、offset 1 → 111,680 clip；测试集窗口 135、offset 20 → 1,569 clip。

| 方法 | L2P@30 | L2P@60 | L2P@90 |
|---|---|---|---|
| Interp. | 0.70 | 1.26 | 1.96 |
| T_ctx | 0.45 | 0.74 | 1.50 |
| T_det | 0.35 | 0.63 | 1.27 |

验证方法不限于人形骨架。

### 10.6 速度（Table 7，i7-7700K + GTX 1080Ti，单位 ms）

| 方法 | bs=1,30帧 | bs=1,60帧 | bs=1,90帧 | bs=100,30帧 | bs=100,90帧 |
|---|---|---|---|---|---|
| ERD-QV | 33.1 | 57.2 | 85.2 | 36.7 | 91.6 |
| Δ-Interp | 32.5 | 33.9 | 36.0 | 352.5 | 478.7 |
| ours（T_ctx only） | 6.3 | 6.2 | 7.0 | 12.9 | 26.1 |
| **ours（完整）** | **12.1** | **12.7** | **11.9** | **19.2** | **32.7** |

约 2.7× 加速，且时间几乎不随过渡长度增长（帧并行处理），而两个对比方法推理时自回归。

### 10.7 Maya 工具与用户研究

集成到 Autodesk Maya。与 Harvey 2020（用户注入噪声生成变体）不同，本文通过**全约束/部分约束**给动画师精细控制：
- 三个按钮：为选定时间范围生成过渡（绿）、给当前帧选定关节加约束（青）、移除约束（红）
- **全约束模式**：可在上下文与目标之间指定任意姿态，生成的过渡满足该时空约束。虽也能通过对各子区间分别预测达到同样目标，但把约束喂进模型更好——单次推理输出整段，约束附近的过渡更连贯。
- **部分约束模式**：只约束部分关节（如只约束根关节位置与旋转），动画师专注需要特定姿态/接触的身体部位，其余由工具补全。

用户研究：3 位专业动画师，3 个任务（各做有工具/无工具两遍，随机顺序，记录耗时）：

| Clip | 帧数 | 无工具 | 有工具 | 提速 |
|---|---|---|---|---|
| A（跑步循环） | 30 | 3.25h | 1.75h | 86% |
| B（前走+右转） | 60 | 4.92h | 2.42h | 103% |
| C（跨过箱子） | 60 | 4.58h | 2.17h | 112% |

约 2 倍提速。动画师还反馈：姿态约束除提供精细控制外，还能作为**管理关键姿态的手段**（类似传统关键帧），从而放心重新生成任意过渡而不担心手工关键姿态被覆盖。

## 11. 局限（论文自述）

1. 输入上下文模糊或与训练分布差异大时会产生不理想结果（Figure 9 例：期望角色自转 360° 再过渡到目标，但最后上下文帧与目标帧差距小，角色可能顺惯性几帧后反向转——可通过加姿态约束解决）。
2. 即使有 Detail Transformer，脚滑仍可能残留，需用输出的接触信息 + IK 后处理。
3. 过渡超过训练最大长度 **1.5 倍**后质量快速下降。
4. 姿态约束作用于关节**局部**位置/旋转，但动画师实际更想约束**末端效应器**（手、脚）——这需要模型在生成时处理 IK，留作未来工作。

## 12. 配套代码库（motion_inbetweening）

### 12.1 目录结构

```
motion_inbetweening/
├── configs/                     # lafan1_context_model.json / lafan1_detail_model.json
├── packages/motion_inbetween/
│   ├── config.py                # 配置加载（路径解析、工作区创建）
│   ├── benchmark.py             # L2P/L2Q/NPSS + zeroV/Slerp 基线
│   ├── data/
│   │   ├── bvh.py               # BVH 解析
│   │   ├── loader.py            # BvhDataSet（滑窗切 clip、FK、足部接触）
│   │   ├── utils_np.py          # numpy FK / 旋转转换 / 接触提取
│   │   └── utils_torch.py       # torch 6D↔9D、FK、Slerp、坐标系对齐
│   ├── model/
│   │   ├── __init__.py          # ContextTransformer / DetailTransformer / Discriminator(未用)
│   │   ├── transformer.py       # MultiHeadedAttention / RelMultiHeadedAttention / PFFN / 正弦PE
│   │   └── rmi.py               # ERD-QV 基线（LSTM + TimeToArrival embedding）
│   ├── train/
│   │   ├── context_model.py     # Context 训练/推理 + 掩码/PE/占位值构造
│   │   ├── detail_model.py      # Detail 训练/推理（输入来自 Context）
│   │   ├── rmi.py               # RMI 基线训练
│   │   └── utils.py             # 损失、checkpoint、Noam、visdom、后处理
│   └── visualization/maya.py    # 输出 JSON → Maya 可视化
├── scripts/                     # train_/eval_ context/detail、run_baseline_benchmark 等
├── datasets/lafan1/             # 需自行下载 BVH
└── experiments/                 # 预训练权重放此
```

### 12.2 关键配置（configs/*.json）

- `train`: `context_len=10, min_trans=5, max_trans=30, batch_size=32, lr_warmup_iterations=8000, midway_targets_amount=0.1, midway_targets_p=0.5`；context 的 `lr=0.0625`、detail 的 `lr=0.025`
- `model`: `n_layer=6, n_head=8, d_model=512, d_head=64, d_pff_inner=512, d_encoder_in=136, d_out=135`（context）/`139`（detail）, `dropout=0.0, pre_lnorm=true, max_seq_len=65`
- `indices`: `r[0:132]` 旋转、`p[132:135]` 根位置、`c[135:139]` 接触
- `weights`: context `rw=1.0, pw=0.01, sw=0.005`；detail `rw=1.0, pw=0.02, cw=0.05, fw=0.05`
- 数据窗口：context 训练窗口 41、detail 42

### 12.3 论文概念 ↔ 代码位置

| 论文概念 | 代码位置 |
|---|---|
| 6D 旋转 | `utils_torch.matrix9D_to_6D_torch` / `matrix6D_to_9D_torch` |
| encoder/decoder MLP | `ContextTransformer.encoder/decoder`（两层 PReLU MLP） |
| pre-norm Transformer 层 | `PositionwiseFeedForward`, `RelMultiHeadedAttention` |
| Keyframe PE | `keyframe_pos_layer` + `get_keyframe_pos_indices` |
| Learned Relative PE | `rel_pos_layer` + `RelMultiHeadedAttention.skew()` |
| 注意力掩码 | `context_model.get_attention_mask` / `detail_model.get_attention_mask` |
| 输入掩码 | `context_model.get_data_mask` |
| 足部接触输出 | Detail `d_out=139`, `c_slice=135:139`, sigmoid |
| 后处理 ψ | `train/utils.anim_post_process` |
| `L_foot` 切断梯度 | `train/utils.cal_f_loss` 里 `c_out.detach()` |
| 随机过渡长度/约束帧 | `context_model.train` 的 `random.randint` + `get_midway_targets` |
| 起帧对齐预处理 | `utils_torch.to_start_centered_data` |
| RMI 风格指标 | `benchmark.get_rmi_style_batch_loss`（用 `to_mean_centered_data`） |

### 12.4 复现命令

```bash
# 基线评测（zeroV + 插值）
python scripts/run_baseline_benchmark.py lafan1_context_model
# 两阶段完整推理，trans=30
python scripts/eval_detail_model.py lafan1_detail_model lafan1_context_model -t 30
# 生成单 clip 过渡 JSON（-i 100，-p 开后处理）
python scripts/eval_detail_model.py lafan1_detail_model lafan1_context_model -t 30 -i 100
# 训练（需先启动 visdom）
python scripts/train_context_model.py lafan1_context_model
python scripts/train_detail_model.py lafan1_detail_model
```

预训练权重复现的指标：完整方法 trans 5/15/30/45 → gpos 0.1049/0.3943/0.8948/1.6777。

### 12.5 代码与论文的差异（阅读时注意）

1. **部分姿态约束未完整实现**：论文称支持 `m_in ∈ R^{T×D}` 的逐维部分约束，但发布代码 `d_mask=1`、`constrained_slices=[[null,null]]`，实际只有"整帧全约束/无约束"两态。
2. **注意力缩放用 `1/√d_model`（1/√512）而非论文式的 `1/√d_K`（1/√64）**：见 `RelMultiHeadedAttention.atten_scale`。
3. **`delta`（帧间差分）算了也统计了归一化参数，但从未喂给模型**：`detail_model.get_model_input` 返回 `(state, delta)`，后续只用 `state`。
4. **`Discriminator` 类与 `dw=0.0` 权重存在但未使用**：疑为早期 GAN 训练残留。
5. **dropout 余弦退火逻辑存在但配置为 0.0**：`update_dropout_p` 实际不生效，与论文结论一致。
6. **"encoder-decoder"措辞易误导**：decoder 是两层 MLP，非自回归解码器。
7. Context 训练窗口 41 vs 论文公式的 42：小不一致，不影响（后处理只在推理用，推理窗口是 `context+trans+2`）。

## 13. 一句话总结

先粗后细、两段式、纯 Transformer Encoder + 掩码注意力的补间框架：Context Transformer 负责全局走向、Detail Transformer 负责运动细节，靠 Keyframe PE + Learned Relative PE 实现超长过渡泛化，输出足底接触供 IK 后处理，最终在 LAFAN1 上以更少参数超越 RNN/CNN/Transformer 各流派 SOTA，并落地为 Maya 生产工具（动画师提速约 2 倍）。
