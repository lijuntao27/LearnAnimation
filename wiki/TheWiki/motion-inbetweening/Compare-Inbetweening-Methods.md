---
type: comparison
title: 补间方法横向对比（ERD-QV / MC-Trans / Δ-Interp / 本文）
created: 2026-08-27
updated: 2026-08-27
tags: [motion-inbetweening, comparison, benchmark, lafan1]
summary: 横向对比补间方法 ERD-QV(RNN)/MC-Trans(Transformer+1Dconv)/Δ-Interp(Enc-Dec+delta)/本文(Enc-only 两阶段)；本文以约一半参数在正常长度提升 16%、超长 45 帧提升 55%，推理 12.1ms vs 33ms。含 LAFAN1 数据集与 L2P/L2Q/NPSS 指标说明。
---

# 补间方法横向对比

> [[MotionInbetweeningTwoStage|论文]] §4.2 的定量对比。底层概念：ERD-QV 是 [[RecurrentNetworks|RNN]]，Δ-Interp 是 [[Transformer|Encoder-Decoder Transformer]]，本文是 Encoder-only 两阶段。

## 数据集：LAFAN1

Ubisoft La Forge 开源动捕数据集：

- ~496,672 帧、30Hz（30 帧 = 1 秒），生产级光学动捕
- 5 名演员，含走/跑/攀爬/跳舞等多种动作
- BVH 格式（骨架层级 + 逐帧根位置与关节欧拉角）
- 刻意剔除随机短动作（上下文模糊、难预测），故特别适合补间任务
- GitHub：`ubisoft/ubisoft-laforge-animation-dataset`

**Benchmark 设置**（沿用 Harvey 2020）：训练 Subject 1-4（窗口 42、offset 1 → 404,195 clip），测试 Subject 5（窗口 65、offset 40 → 2,232 clip）。统一 `t_ctx=10`、训练最大过渡 30 帧，评估 5/15/30/45。

## 三个指标

| 指标 | 全称 | 衡量 |
|---|---|---|
| **L2P** | L2 Position | 全局关节位置的平均 L2 距离（每关节每帧） |
| **L2Q** | L2 Quaternion | 全局关节旋转（四元数）的平均 L2 距离 |
| **NPSS** | Normalized Power Spectrum Similarity（Gopalakrishnan 2019） | 频域归一化功率谱相似度，评估角度差异，衡量整体运动真实性 |

计算前对运动数据归一化（提取旋转与位置表示的均值/标准差），使指标可跨数据集比较。代码 `benchmark.get_rmi_style_batch_loss` 用 `to_mean_centered_data`。

## 四个方法

| 方法 | 架构 | 特点 |
|---|---|---|
| **ERD-QV**（Harvey 2020, Robust Motion In-betweening） | [[RecurrentNetworks\|RNN（增强 LSTM）]] | 逐帧自回归，每步输入当前状态/偏移/目标 + time-to-arrival embedding + 可选目标噪声 |
| **MC-Trans**（Duan 2021） | [[Transformer\|Transformer Encoder]] + 1D 卷积 | 非自回归，BERT encoder + 1D conv 做补全 |
| **Δ-Interp**（Oreshkin 2022, Deep Δ-Interpolator） | [[Transformer\|Transformer Encoder-Decoder]] | 在 Slerp 之上预测 delta 动作，`输出 = Slerp + delta` |
| **Ours** | [[Transformer\|Encoder-only]] 两阶段 + 掩码 + 自定义 PE | 一次性输出整段，Context 粗 + Detail 精 |

## 定量对比（Table 1）

所有模型训练最大过渡 30 帧。▲为相对本文的劣化幅度。

| 方法 | 参数量 | L2P@5 | L2P@15 | L2P@30 | L2P@45 | L2Q@30 | L2Q@45 | NPSS@30 | NPSS@45 |
|---|---|---|---|---|---|---|---|---|---|
| Interp.（线性+Slerp 基线） | — | 0.37 | 1.25 | 2.32 | 3.45 | 0.98 | 1.25 | 0.2013 | 0.4493 |
| FCN 基线 | — | 0.25 | 0.69 | 1.63 | 4.69 | 0.77 | 1.39 | 0.1524 | 0.8571 |
| MC-Trans（local） | — | 0.23 | 0.74 | 1.37 | — | 0.71 | — | 0.1430 | — |
| ERD-QV | — | 0.23 | 0.65 | 1.28 | 2.24 | 0.69 | 0.94 | 0.1328 | 0.3311 |
| Δ-Interp | 40.4M | 0.13 | 0.47 | 1.00 | 3.23 | 0.57 | 1.15 | 0.1217 | 0.4539 |
| Ours（T_ctx only） | 10.4M | 0.17 | 0.49 | 1.07 | 2.00 | 0.60 | 0.92 | 0.1238 | 0.3369 |
| Ours（T_ctx only, 后处理） | 10.4M | 0.13 | 0.46 | 1.04 | 1.94 | 0.59 | 0.91 | 0.1210 | 0.3349 |
| Ours（T_ctx only, 全损失项但无 L_smooth, 后处理） | 10.4M | 0.13 | 0.45 | 1.01 | 1.88 | 0.59 | 0.90 | 0.1209 | 0.3365 |
| **Ours（完整两阶段）** | **10.4+10.1M** | **0.10** | **0.39** | **0.89** | **1.68** | **0.54** | **0.87** | **0.1124** | **0.3217** |

### 关键结论

- **正常长度平均提升 16%、超长（45 帧）提升 55%**；参数量不到 Δ-Interp 一半
- Table 1 倒数第二行（"全损失项但无 L_smooth"）证明：**单阶段即使配齐损失也达不到两阶段效果**——两阶段设计本身有用
- Δ-Interp 在 45 帧急剧劣化（1.00→3.23）：Slerp 只看两个关键帧、可能与上下文矛盾，delta 修正不过来；且其位置编码用 one-hot 过全连接，超长泛化差
- ERD-QV 的"突变"问题（[[RecurrentNetworks|自回归局部生成]]所致）在长过渡上尤为明显

## 定性对比要点

- **FCN**：短过渡尚可，但超训练长度后生成动作明显丢失细节
- **ERD-QV**：有根关节位置/旋转突变——自回归下远离目标时像做运动预测（输出与目标关系弱），接近目标时才转向，两阶段交界处突变。本文三个特性避免此问题：两阶段、整段一次输出、自注意力跨任意距离聚合
- **Δ-Interp**：过渡初段接近真值，但当 Slerp 偏离真值后，到达无法用 delta"修正"Slerp 的点，末段退化成 Slerp 结果

## 速度对比（Table 7，i7-7700K + GTX 1080Ti，ms）

| 方法 | bs=1,30帧 | bs=1,60帧 | bs=1,90帧 | bs=100,30帧 | bs=100,90帧 |
|---|---|---|---|---|---|
| ERD-QV | 33.1 | 57.2 | 85.2 | 36.7 | 91.6 |
| Δ-Interp | 32.5 | 33.9 | 36.0 | 352.5 | 478.7 |
| ours（T_ctx only） | 6.3 | 6.2 | 7.0 | 12.9 | 26.1 |
| **ours（完整）** | **12.1** | **12.7** | **11.9** | **19.2** | **32.7** |

约 2.7× 加速，且时间几乎不随过渡长度增长（帧并行处理），而两个对比方法推理时自回归、随长度线性增长。

## 掩码消融（Table 4，Context Transformer）

| 变体 | L2P@30 | L2P@45 | NPSS@45 |
|---|---|---|---|
| T_ctx 完整 | 1.04 | 1.94 | 0.3349 |
| 去 m_in（输入掩码） | 1.04 | 1.95 | 0.3566 |
| 去 m_atten（注意力掩码） | 1.41 | 2.91 | 0.3846 |
| 都去 | 1.48 | 2.92 | 0.3928 |

**去 `m_atten` 最伤**——占位值会在每个 Transformer 层污染自注意力。去 `m_in` 影响较小。

## 四足泛化（Table 5，Zhang 2018 数据集，训练 60 帧过渡）

| 方法 | L2P@30 | L2P@60 | L2P@90 |
|---|---|---|---|
| Interp. | 0.70 | 1.26 | 1.96 |
| T_ctx | 0.45 | 0.74 | 1.50 |
| T_det | 0.35 | 0.63 | 1.27 |

验证方法不限于人形骨架。

## 一句话

本文以 **Encoder-only 两阶段 + 掩码 + MLP 生成位置编码**，在 LAFAN1 上以约一半参数（20.5M vs 40.4M）在正常长度提升 16%、超长 45 帧提升 55%，推理快 2.7× 且不随长度增长——核心优势来自全局并行生成（治 RNN 的局部与慢）与不限长度的位置编码（治 Δ-Interp 的超长泛化崩）。

## 相关页面

- [[MotionInbetweeningTwoStage]] —— 论文总览
- [[RecurrentNetworks]] —— ERD-QV 的 RNN 底层
- [[Transformer]] —— MC-Trans / Δ-Interp / 本文的 Transformer 底层与三种用法
- [[InbetweeningPositionalEncodings]] —— 本文泛化超长的关键
