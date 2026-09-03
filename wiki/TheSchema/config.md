---
vault: wiki
domain: MotionMatching 角色动画技术研究（基础 Motion Matching 算法与 Learned Motion Matching 学习方法）
language: zh-CN
schema-version: "1.1"
created: 2026-08-27
openviking:
  enabled: false              # install.bat 在 local 模式会翻为 false（见 obsidian-wiki Skill Part E）
  workspace: "[OV 元数据路径：由 ov.conf 的 storage.workspace 决定，不在此配置]"
  resources:
    - uri: "viking://resources/TheWiki"
      local: "TheWiki"
      role: "wiki-native"                # Query L1.5 使用
    - uri: "viking://resources/RawSource"
      local: "RawSource"
      role: "raw"                        # Query L3 使用
  exclude:
    - "TheSchema/**"                     # 规则不参与语义检索
    - "TheWiki/**/_index.md"             # 与 .overview.md 重复
    - "TheWiki/_lint-report.md"
    - "TheWiki/**/Summary-*.md"          # 派生型，原文在 RawSource
    - ".obsidian/**"
    - ".trash/**"
---

# 知识库配置

## 基本信息

- **Vault 名称**：wiki
- **知识域**：MotionMatching 角色动画技术研究
- **首选语言**：中文
- **创建日期**：2026-08-27

## 规则总览

- **录入规则**：见 [[TheSchema/ingest-guide]]
- **查询规则**：见 [[TheSchema/query-guide]]
- **体检规则**：见 [[TheSchema/lint-guide]]
- **会话启动包注入规则**：见 [[TheSchema/session-init-guide]]

## 页面类型命名约定

| type | 命名格式 | 示例 | 是否进 OpenViking |
|------|---------|------|-----------------|
| index | `_index.md` | `TheWiki/ui/_index.md` | ❌ |
| entity | PascalCase | `UIManager.md` | ✅（若无 source 字段）|
| concept | PascalCase | `EventDrivenPattern.md` | ✅（若无 source 字段）|
| summary | `Summary-XXX` | `Summary-NetworkModule.md` | ❌（派生型）|
| comparison | `Compare-A-vs-B` | `Compare-ECS-vs-OOP.md` | ✅（若无 source 字段）|
| answer | `QA-SlugTitle` | `QA-HowToAddPanel.md` | ✅ |

## OpenViking 集成说明

本知识库使用 OpenViking 作为语义检索外脑（见向量索引分层策略）。

- **Ingest 时**：RawSource 全量入库；TheWiki 按"原生 vs 派生"分流
- **Query 时**：优先 Obsidian 原生（L0/L1/L2），语义兜底走 OpenViking（L1.5/L3）
- **Lint 时**：第 7 项检查向量索引与 Vault 的一致性
- **TheSchema 始终不进向量库**：规则变更不触发重建索引

> **local 模式（OV 关闭）**：若 `openviking.enabled: false`，ingest 不调 `add_resource`，query 走 Obsidian 原生检索（L0/L1/L2），lint 跳过第 7 项。详见 obsidian-wiki Skill「Part E」与「Vault 定位」节。
