---
type: raw-article
title: "SoftMax: Neural Simulation of Skeletal Physics"
created: 2026-09-04
updated: 2026-09-04
tags: [cloth-animation, skeletal-physics, secondary-motion, neural-simulation, gdc-2026, talk-notes, needs-verification]
summary: GDC 2026 ML Summit（LIGHTSPEED/光子，FateTrigger）Talk 档案：神经网络替代角色骨骼次级物理（发丝/布料/挂件），QAT+自研 C++ 推理+动态 LOD，单核 CPU 0.02–0.13ms/角色。当前仅官方元数据+摘要可信；视频本体与讲义未获取，待字幕核验后深化。
---

# SoftMax: Neural Simulation of Skeletal Physics

> **信息来源分层（务必先读）**：
> - **① 官方一手（已核对，可信）**：GDC 官方议程页 —— 会议信息表 + 「官方摘要」节；
> - **② KM 视频本体**（https://km.netease.com/v4/detail/video/21642）：未获取字幕/讲义，尚未做逐字通读；
> - **③ KM 页面内置 AI 工具生成的摘要**：**不可靠，已证伪，本文不将其作为事实引用**。详见「内容可信度评估」。

## 会议信息

| 项 | 内容 |
|---|---|
| 标题 | SoftMax: Neural Simulation of Skeletal Physics |
| 环节 | GDC 2026 · Machine Learning Summit |
| 演讲人 | Xin Qiao（Senior Software Engineer）· Xiaolin Zhang（Gameplay Engineer） |
| 单位 | **LIGHTSPEED STUDIOS（腾讯光子）** |
| 时间/地点 | 2026-03-11（周三）15:10–16:10，Room 2014, West Hall, Moscone Center, San Francisco |
| 案例游戏 | **FateTrigger**（光子动作游戏） |
| 官方议程页 | https://schedule.gdconf.com/session/softmax-neural-simulation-of-skeletal-physics/915222 |
| 内部转载 | KM 视频 [21642](https://km.netease.com/v4/detail/video/21642)，上传：杜敬辉（互娱培训发展组），2026-05-19 |

## 一句话（仅依据官方摘要）

用轻量神经网络取代角色骨骼上挂的实时次级物理（头发、布料、挂件），把大量角色的仿真成本压到单核 CPU 每角色 **0.02–0.13 ms**，在 FateTrigger 数十角色上量产落地；配套 **QAT 量化**、自研 **C++ 推理引擎**与**动态负载均衡 LOD**。

## 官方摘要（一手，原文）

> This session presents a production-proven Neural Simulation system for skeletal physics, as seen on dozens of characters in the action game FateTrigger. The presenters explain how they overcame the notorious performance bottleneck of real-time secondary physics simulations (hair, cloth, accessories) for a large cast of characters. Their deep learning solution delivers high-fidelity, stable physics for a fraction of the cost — just 0.02-0.13ms on a single CPU core — making complex simulations viable while maintaining smooth, high-framerate gameplay. The talk explores the entire production pipeline, from its meticulously designed network architecture with quantization-aware training, to its high-performance custom C++ inference engine and a dynamic, load-balanced LOD system. It demonstrates that neural physics is no longer a research concept but a powerful solution for achieving superior performance while scaling to a higher volume of dynamic character simulations.

### 中文摘要（翻译）

本 talk 展示一套**已量产验证**的骨骼物理神经模拟系统，落地于动作游戏 FateTrigger 的数十个角色。演讲者讲述如何攻克大量角色实时次级物理仿真（头发、布料、挂件）的经典性能瓶颈：其深度学习方案以单核 CPU **0.02–0.13ms** 的极低成本，交付高保真、稳定的物理表现，让复杂仿真在保持流畅高帧率的前提下变得可行。内容覆盖完整生产管线——带**量化感知训练（QAT）**的网络架构、自研高性能 **C++ 推理引擎**、**动态负载均衡 LOD 系统**。核心论点：神经物理已不是研究概念，而是能在扩展到更大规模动态角色仿真时取得更优性能的成熟工程方案。

### Takeaway（官方给的收获点）

> Attendees gain a practical blueprint for building their own high-performance neural physics system. This includes specific techniques for creating an automated data pipeline, designing a versatile network architecture for temporal dynamics, and deploying a highly optimized C++ inference engine for real-time performance in a production environment.

即：自动化数据管线的构建方法、面向时序动态的网络架构设计、生产环境下高度优化的 C++ 推理引擎部署。

### 目标听众

Graphics programmers、engine developers、AI programmers、technical artists；建议 3D 数学与引擎架构基础（Advanced 级）。

## 内容可信度评估（重要）

KM 视频页内置 **AI 工具生成的摘要**经核对**不可靠**，含编造细节，**不得作为事实引用**：

### 已证伪 / 与官方冲突

| AI 摘要声称 | 判定 |
|---|---|
| 单位「Sino Software」旗下未具名开放世界游戏 | **编造**。官方演讲单位 = LIGHTSPEED STUDIOS（腾讯光子），游戏 = FateTrigger |
| 术语「Damping Quantization」 | 疑把「动态量化」误译；正确术语应为 **Dynamic Quantization**（AI 摘要的英文回译不可信） |
| 目标 120 fps 等帧率表述 | 官方摘要未提具体帧率 |

### 无官方佐证（AI 摘要独有，暂存疑）

- 性能数字：64 角色实例、推理延迟 <5ms（95th）、省 70% 显存 —— 与官方「0.02–0.13ms/角色」口径不一致，存疑
- 「核心网络为带状态记忆的 **Transformer**」「隐式状态编码器-解码器」「分层时间积分」「物理引导机制」等架构描述 —— 仅出自 AI 摘要，官方摘要未证实

### 与官方摘要方向一致（低风险线索，仍待讲义证实）

以下点即便出自 AI 摘要，也与官方摘要口径相容，可作后续核实的「待证线索」：多角色 CPU 部署、量化（QAT 方向）、C++ 推理、动态 LOD、自动数据管线。

## 已知信息缺口 / 下一步

- [ ] **获取可信原文**：导出 KM 视频字幕/讲义（若页面提供「字幕/AI 字幕」导出），或找到演讲者公开 slides 后，再做逐字通读
- [ ] 网络架构具体形式、训练数据来源、QAT 位宽、LOD 策略、状态量设计 —— 待原文核实后补充
- [ ] **KM 内置 AI 摘要不再作为本页来源**；如需保留参考，单独存放并标注存疑

## 与本仓库谱系的关系（仅保守结论）

工业落地的「骨骼次级物理神经模拟」与本仓库学术支线（[[Pan2022-VirtualBones]]、[[Jin2024-RopeChainQNN]]、[[HyperBones2026]]、[[PhySkin2026]]）**目标一致**：用轻量网络替代逐顶点物理积分、按骨骼/关节驱动次级运动。差异在于本 talk 的落地约束是**单核 CPU 实时 + 量化 + 多角色 LOD**（学术论文少展开的"生产规模化"维度）。具体机制（是否 Transformer、是否类 [[QNN]]/[[HyperBones2026]] 时序）**待原文证实后再定位**。

## 关联

- 谱系概览：[[Compare-BoneDrivenCloth-Lineage]]
- 学术近邻：[[HyperBones2026]]（动态、GPU）· [[PhySkin2026]]（准静态、CPU µs 级）
- 前驱概念：[[VirtualBones]] · [[QNN]] · [[RopeChain]]
