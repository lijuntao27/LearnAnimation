---
type: concept
title: 特征向量 x 与姿态向量 y
created: 2026-08-27
updated: 2026-08-27
tags: [motion-matching, feature-engineering, pose-representation]
summary: Motion Matching 的两套数据表示：x 描述上下文与用户轨迹（R27），y 完整描述一帧姿态；按标准差归一化加用户权重，构成 Matching/Animation Database。
---

# 特征向量 x 与姿态向量 y

> 论文 §3。两套向量是 [[MotionMatching]] 数据库的基石，也是 [[LearnedMotionMatching]] 各网络的输入/输出契约。

## 特征向量 x（locomotion 场景，`∈ R²⁷`）

$$x = \{t_t,\ t_d,\ f_t,\ \dot{f}_t,\ \dot{h}_t\}$$

| 分量 | 维度 | 含义 |
|---|---|---|
| `t_t` | R⁶ | 未来 20/40/60 帧的 2D 轨迹位置（地面投影，local 到角色） |
| `t_d` | R⁶ | 未来 20/40/60 帧的轨迹朝向 |
| `f_t` | R⁶ | 双脚关节位置（local 到角色） |
| `ḟ_t` | R⁶ | 双脚关节速度 |
| `ḣ_t` | R³ | 髋部速度 |

> 不同行为选不同特征即可驱动不同控制器（见下表场景变体）。这是 MM 灵活性的来源之一。

## 姿态向量 y

$$y = \{y_t,\ y_r,\ \dot{y}_t,\ \dot{y}_r,\ \dot{r}_t,\ \dot{r}_r,\ o^*\}$$

| 分量 | 含义 |
|---|---|
| `y_t` / `y_r` | 关节局部位移 / 旋转 |
| `ẏ_t` / `ẏ_r` | 关节局部平移 / 旋转速度 |
| ṙ_t / ṙ_r | 角色根的平移 / 旋转速度（local 到角色前方） |
| `o*` | 其他任务输出（足部接触、其他角色位置、未来关节位置等） |

表示约定：旋转速度用 scaled angle-axis；关节旋转用四元数；输入/输出 NN 时转 2-axis rotation matrix（Zhang 2018 表示）。算法对姿态表示不强假设，character space 编码（Holden 2017）也通用。

## 归一化与权重

- 每特征按**数据集标准差**缩放（量级差异大，必须归一）
- 再乘**用户权重**调整搜索重要度（默认 1 即够）
- 可对 X、x̂ 做 PCA 降维（非必须，保留足够维数则距离不变）

## 两个数据库

- `X = [x_0,...,x_{n-1}]` —— **Matching Database**
- `Y = [y_0,...,y_{n-1}]` —— **Animation Database**
- 每帧同时算 x、y 拼成大矩阵

## 各场景的特征变体（论文 Table 2）

| 场景 | 关节 | 帧数 | 特征数 | 特征差异 |
|---|---|---|---|---|
| Locomotion | 28 | 89,480 | 27 | 上述基础 + 步态（走/跑/蹲） |
| Terrain | 28 | 170,534 | 35 | + 未来 0/15/30/45 帧脚趾下方地形高度（相对髋高） |
| Chair | 28 | 22,398 | 19 | 基础 + 相对椅子位置/朝向 |
| Interaction | 28 | 38,976 | 32 | + 对方角色位置/朝向/速度 |
| Dog | 58 | 124,418 | 39 | 四足：四爪位置/速度 + 轨迹 + 步态 |
| Bear | 75 | 694,272 | 55 | carpet unrolling 增强到 ~70 万帧 |

## 运行时构造 x̂

- 姿态类特征：从当前帧 `i` 在 X 中的对应行取
- 用户控制类特征（如未来轨迹位置/方向）：由 spring-damper 系统根据摇杆状态生成期望速度与方向（Kermse 2004）

## 在 LMM 中的角色

- `x` 是 [[Projector]] 与 [[Stepper]] 的输入/输出空间（Projector 还输出隐变量 `z`）
- `y` 是 [[Decompressor]] 的输出
- `z ∈ R^32` 是 Compressor 发现的「x 中缺失信息」的隐变量，与 x 一起喂给 Decompressor

## 注意事项

- 地形场景：运行时 Decompressor 须输出未来脚趾位置以采样地形高度；训练时需从动捕数据找地形高度（Appendix D，最近邻回归 xy→z 拟合「stepped」地形）
- Interaction：玩家 Decompressor 必须输出同步轨迹，否则 NPC 匹配时拿不到该数据
