---
type: schema-guide
title: 查询操作规则
updated: YYYY-MM-DD
---

# 查询规则

## 检索分层策略（L0 / L1 / L1.5 / L2 / L3）

- **L0 目录层**：`obsidian read` TheWiki/_index.md + 子目录 _index.md，定位子领域
- **L1 摘要层**：`obsidian search`，看 snippet + 每页 frontmatter.summary
- **L1.5 语义兜底**：`openviking find` target=`viking://resources/TheWiki`，对 TheWiki **原生**页做语义检索
- **L2 正文层**：`obsidian read` 完整页面（默认 1 页，对比类最多 2 页）
- **L3 原始层**：`openviking find` target=`viking://resources/RawSource`，兜底到未沉淀的原始资料

> 按"能答则止、按需下沉"原则推进。L1 → L1.5 → L2 → L3 依次激活，不跳层、不回头。
> **OV 关闭时（local 模式）**：L1.5 与 L3 不可用，仅走 L0/L1/L2 原生检索。

## 搜索优先级（同层级内排序）

1. `type: answer`（已验证的高质量回答）
2. `type: entity` / `type: concept`（权威知识页）
3. `type: summary`（原始资料摘要）
4. `type: comparison`（对比分析）

## 何时止步于 L0 + L1

- "Wiki 里有没有 XXX" → L1 命中列表
- "XXX 大致是什么" → 直接使用 summary
- 路线/目录导览类问题 → 只读 index

## 何时进入 L1.5

- L1 零命中或命中均为字面相似但语义模糊
- 用户使用自然语言描述而非专有名词（典型：近义词、用途描述）
- 典型受益场景：TheWiki 中无 RawSource 根的原生页面（如引擎类）

## 何时进入 L2

- 需要具体实现细节、代码片段、步骤指令
- 对比类问题（A vs B），各取 1 页
- L1/L1.5 摘要置信度高但信息量不足

## 何时进入 L3

- L1 + L1.5 均未命中
- 回答时必须附加"沉淀建议"（模板 F），提醒用户 Ingest

## 输出格式偏好

- 默认输出：Markdown 段落
- 对比类问题：优先使用表格
- 涉及代码：优先使用代码块+注释

## 回答质量标准

满足以下条件的回答建议回存 Wiki（回存时必须补写 summary 字段）：
- 回答时间 > 5 分钟思考
- 实际进入了 L2 正文精读
- L3 命中触发的回答（强烈建议回存，这是首次沉淀）
- 涉及多个 Wiki 页面的综合
- 用户表示"这个很有用"
