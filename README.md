# LearnAnimation

个人「角色动画技术」学习与研究知识库：

- Motion Matching / Learned Motion Matching
- Motion In-betweening
- 骨骼驱动布料（Pan 2022 VirtualBones → Jin 2024 RopeChain/QNN → Meta 2026 HyperBones/PhySkin 谱系）

## 结构

| 路径 | 说明 |
|---|---|
| `wiki/` | **Obsidian 知识库（主要产物）**。三层架构：`RawSource/`（论文原文 PDF/提取文本/通读笔记）、`TheWiki/`（领域笔记与索引）、`TheSchema/`（规则层）。用 `/obsidian-wiki` 工作流维护 |
| `README.md` | 本文件 |

> `wiki/` 目录即 Obsidian Vault 根。`wiki/.obsidian/`、`wiki/.claudian/` 为机器本地配置，不入库（见 `wiki/.gitignore`）。

## 论文原文

`wiki/RawSource/articles/` 下每篇论文 = **PDF + pdftotext 全文提取(.txt) + 通读笔记(.md)**，命名 `<作者>-<年份>-<主题>`。

## 本地代码工程（不纳入本仓库）

以下代码目录**保留在本地工作区，各自有独立 git**，本仓库通过 `.gitignore` 排除、不追踪（也不做 submodule），仅在此记录地址；完整映射见
[wiki/TheWiki/CodeRepos.md](wiki/TheWiki/CodeRepos.md)。

| 本地目录 | 上游 / 仓库 | 关联论文/内容 |
|---|---|---|
| `motion_inbetweening/` | <https://github.com/victorqin/motion_inbetweening> | Motion In-betweening via Two-stage Transformers (2022) |
| `VirtualBones/` | <https://github.com/non-void/VirtualBones> | Pan 2022，Predicting Loose-Fitting Garment Deformations… |
| `physkin/` | <https://github.com/facebookresearch/physkin> | PhySkin 2026，Physics-based Bone-driven Neural Garment Simulation |
| `Dual-Quaternion-Skinning-Sample-Codes/` | <https://github.com/brainexcerpts/Dual-Quaternion-Skinning-Sample-Codes> | LBS / Dual Quaternion 蒙皮基础参考实现 |
| `SkirtQNN/` | 独立 git 仓库（本地，暂无 remote） | 个人裙摆 QNN 运行时工程（关联 Jin 2024 rope-chain/QNN 思路） |

`.claude/`（Claude Code 本地配置、技能与许可）同样不入库。
