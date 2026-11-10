---
type: schema-changelog
title: Schema 变更记录
---

# Schema 变更记录

<!-- 每次 Schema 更新后在此追加记录，格式：-->
<!-- - YYYY-MM-DD: [变更描述] (触发：[用户请求摘要]) -->

- YYYY-MM-DD: 初始化知识库 (触发：首次使用 obsidian-wiki Skill)
- 2026-08-27: 录入 Learned Motion Matching 论文（Holden 2020），新建 motion-matching 领域——1 篇 RawSource 全文 + 8 个知识页（2 concept / 3 entity / 1 comparison / 1 summary / 1 领域 index） (触发：用户请求拆解 LMM 论文并入 wiki)
- 2026-09-02: ingest-guide 新增「数学公式 Markdown 格式（必守）」书写规则（行内 `$...$`、块级 `$$...$$`，禁公式进代码块/ASCII/手写体符号）；并按该规则修正 TheWiki/deep-learning/RecurrentNetworks 页全部公式 (触发：用户反馈 RecurrentNetworks 公式未用正确 markdown 输出，要求修正并写入 schema)
- 2026-09-07: 公式规范从 ingest-guide 抽出独立成篇 `TheSchema/math-notation.md`：新增「公式 vs 结构示意图边界（代码块豁免）」规则 + 全书统一符号约定表（向量/矩阵/算子/学习相关/数据形状），ingest-guide 改为引用链接；并按新规范修正 TheWiki/deep-learning/NeuralNetworkBasics.md 全部公式（触发：用户要求统一数学公式书写格式并沉淀 Nielsen Ch.1 基础概念入 NeuralNetworkBasics）
