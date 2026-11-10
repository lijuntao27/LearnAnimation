---
type: schema-guide
title: 录入操作规则
updated: 2026-09-02
---

# 录入规则

## 值得录入的内容

- 代码模块分析（结构、职责、关键接口）
- 功能实现总结（设计决策、踩坑记录）
- AI 会话中产生的高价值洞察
- 外部参考资料的核心摘要

## 不录入的内容

- 临时调试代码
- 未经验证的猜测
- 可以直接查代码的细节（注释即文档）

## 页面粒度建议

- 一个模块/类 → 一个 entity 页面
- 一个架构模式 → 一个 concept 页面
- 一次会话总结 → 一个 summary 页面（+ RawSource 原文）

## Summary 字段要求（frontmatter 必填）

- 长度 40~80 字，点明"这页讲什么 + 关键实体"，不是标题复述
- 每次正文核心变更时，一并刷新 summary 与 updated
- 子目录 _index.md 表格的 Summary 列必须与页面 frontmatter 一致

## 派生型 vs 原生型（决定是否进 OpenViking）

- **派生型**：文件名以 `Summary-` 开头，或 frontmatter 含 `source:` 指向 RawSource → 不进向量库
- **原生型**：无 `source:` 字段的 entity/concept/answer/comparison → 进 OpenViking `viking://resources/TheWiki`
- RawSource/**：始终全量入 `viking://resources/RawSource`

## 内容质量标准

- 摘要应能让 3 个月后的自己快速理解核心内容
- 每个 entity 页面应包含：用途、关键接口、使用示例、注意事项
- 每个 concept 页面应包含：定义、适用场景、优缺点、在本项目中的体现

## 数学公式与符号约定（必守）

数学公式渲染格式、公式与「结构示意图」的边界、以及全书统一符号约定，**一律以 [[TheSchema/math-notation]] 为准**（2026-09-07 自本文件抽出独立成篇）。速记：

- 行内公式 `$...$`、块级公式 `$$...$$`（MathJax / Obsidian 原生 LaTeX）
- 禁：公式进代码块/反引号、ASCII/Unicode 手拼公式、下标上标手写体
- 代码块仍可用于**结构示意图/数据流图**（与公式的边界判定见 math-notation §2）
- 向量、矩阵、梯度、代价、更新式等符号按 math-notation §3 统一

参考：`TheWiki/deep-learning/RecurrentNetworks.md`、`TheWiki/deep-learning/NeuralNetworkBasics.md` 是按此规范修正的示例页。
