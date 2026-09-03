---
type: schema-guide
title: 会话启动包注入规则
updated: YYYY-MM-DD
---

# 会话启动包注入规则

## 触发条件（按需，非自动）

本注入机制**默认不生效**。触发需满足：**会话首条消息**（消息计数 == 1）且包含 `[kebab-case-slug]` 标记。

```
示例触发消息：
  "[render-pipeline-refactor] 继续讨论渲染管线方案"  → 直接按 slug 加载全文
  "[input-system-redesign] 继续输入系统改造"        → 直接按 slug 加载全文
```

**slug 识别规则**：方括号内全为小写字母与 `-`（kebab-case），长度 ≥ 3。通过 `obsidian search` 文本搜索 Vault 中匹配的文档。无法匹配时静默忽略。

**不触发时**：Agent 完全静默，不做任何检测或输出。

**仅执行一次**：本会话内触发并执行后，后续消息即使含上述标记也不重复执行。

## 与 OpenViking Memory 的职责边界

| 系统 | 覆盖命名空间 | 触发时机 | 控制方 |
|------|-------------|---------|--------|
| **OpenViking Memory 自动注入** | `viking://user/memories/`、`viking://agent/memories/`、`viking://session/` | 每轮用户消息 | 插件（自动） |
| **本规则（启动包注入）** | `viking://resources/TheWiki/`（= Vault 内 `TheWiki/`） | **仅会话首条消息** | 用户（通过 frontmatter） |

**简单口诀**：
- "**谁是谁**"（用户偏好、过往事件）→ Memory 插件管，本规则不动。
- "**什么是什么**"（项目知识、任务进度）→ 本规则管。

两者并行运行，互不干扰。

---

## Frontmatter 新增字段

在现有 `type / title / created / updated / source / tags / summary` 基础上，增加三个字段（**仅在需要参与启动包注入时填写**）：

| 字段 | 取值 | 含义 | 适用 type |
|------|------|------|----------|
| `pin` | `always` \| `task-active` \| `none`（默认） | 注入策略 | 任意 |
| `status` | `in-progress` \| `paused` \| `done` \| `abandoned` | 任务状态 | 仅 `handoff` |
| `session-init-priority` | 整数（默认 0，越大越先） | 同层多文档命中时排序 | 任意 |

字段语义：
- `pin: always` → 每会话首条必注入（第一层）
- `pin: task-active` + `status: in-progress` + type `handoff` → 任务接续候选（第二层）
- `pin: none` 或不填 → 不参与硬注入，但仍可被第三层 memsearch 召回

---

## 注入决策树（顺序执行）

```
[新会话第一条用户消息]
    ↓
┌──────────────────────────────────────────────────┐
│ Step 0.5: Slug 定向加载                            │
│   检测消息是否含 [kebab-case-slug] 标记            │
│   → obsidian vault="<name>" search query="<slug>" limit=1 │
│   → 命中文档：读全文，结束                        │
│   → 未命中：静默忽略，模型进入 Phase 0            │
└──────────────────────────────────────────────────┘
```

> 注：原 [wiki] 触发的 Step 1~3（pin 注入、handoff 接续、memsearch 召回）已移除。
> 模型自主检索（OV auto-recall + 模型根据 prompt 自行判断）已承担语义召回职责，
> session-init-loader 仅负责 slug 精准定向加载。

## 配额规则

1. **每次会话启动最多注入 1 篇全文**。Slug 定向加载即占用此配额。
2. 如需读多篇，用户应在会话进行中显式说"再读 [[XXX]]" / "把 [[YYY]] 也加载"。

---

## handoff 文档最小 schema

满足启动包接续条件的 handoff 文档**必须**包含以下三项：

```yaml
---
type: handoff
status: in-progress | paused | done | abandoned
slug: <kebab-case-unique-key>
# 以下为推荐但非强制
title: ...
pin: task-active
keywords: [...]
related-files: [...]
created: YYYY-MM-DD
updated: YYYY-MM-DD
session-init-priority: 0
---
```

文档正文必须包含 `## Next Entry Point` 段落，说明下次接续的入口指引。

> 其余字段（关键决策记录、进度、相关讨论等）均为可选，按需添加。

---

## 文件位置约定

任务 handoff 文档遵循"一任务一目录"，按任务所属领域归入 TheWiki 顶级目录，**不另起 `Tasks/` 目录**：

```
TheWiki/
├── <domain-a>/
│   └── <slug>/
│       ├── journal.md              ← handoff 主文档
│       └── attachments/             ← 截图、辅助资料（可选）
├── <domain-b>/
│   └── <slug>/
│       └── journal.md
└── ...其他领域
```

---

## Lint 校验项（供 lint-guide 引用）

启动包相关的体检项：

| 检查项 | 严重级别 | 说明 |
|-------|---------|------|
| `pin: always` 文档总数超过 5 篇 | warning | 永久注入过多会膨胀启动开销 |
| `type: handoff` 缺 `slug` 字段 | error | 阻塞召回与排序 |
| `type: handoff` 缺 `status` 字段 | error | 阻塞 Step 2 筛选 |
| `type: handoff` 文档正文无 `## Next Entry Point` | error | 接续无入口 |
| `status: done` 但 `pin: task-active` 未清 | warning | 已完成任务仍占第二层名额 |
| `pin` 取值非约定值 | error | 字段污染 |

---

## 与现有 schema 的关系

- 本规则是 `query-guide.md` 的**前置自动化**：在用户主动 query 之前就装载常用上下文。
- 本规则**不修改** `ingest-guide.md`：handoff 文档的 ingest 仍走标准录入流程。
- 本规则的 frontmatter 字段（`pin` / `status` / `session-init-priority`）属于**可选扩展**，不强制要求所有现有文档迁移。
