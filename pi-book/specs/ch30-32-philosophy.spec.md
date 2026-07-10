spec: task
name: "第九篇：设计哲学"
inherits: project
tags: [chapter, part9, philosophy]
estimate: 1.5d
depends: [ch15-18-extensibility, ch19-23-tools]
---

## Intent

撰写第 30-32 章，将前面所有具体设计决策升华为统一的设计哲学。
回答：这些决策背后有什么共通的思想？这套架构的适用边界在哪？

## Decisions

- 第 30 章：极简核心 + 能力外置的方法论，对比 Claude Code/Cursor/Windsurf
- 第 31 章：逐个解释"反主流选择"（不内建 sub-agents、MCP、permission popup、plan mode）
- 第 32 章：适用边界 — 适合什么团队、不适合什么产品、二次开发建议

## Boundaries

### Allowed Changes
- pi-book/chapters/ch30-minimal-core.md
- pi-book/chapters/ch31-contrarian-choices.md
- pi-book/chapters/ch32-boundaries.md

### Forbidden
- 不要在哲学章节贴源码
- 不要对竞品做负面评价，只做设计对比

## Completion Criteria

Scenario: 反主流选择有理有据
  Test: review_ch31_contrarian
  Given 第 31 章 markdown 文件
  When 审阅反主流选择段落
  Then 至少覆盖 4 个"不内建"的决策
  And 每个决策解释"不做 X"是因为"用更底层的机制 Y 组合出来"
  And 不包含对竞品的负面评价

Scenario: 适用边界具体
  Test: review_ch32_boundaries
  Given 第 32 章 markdown 文件
  When 审阅适用边界段落
  Then 包含"适合"和"不适合"两个方向的具体描述
  And 包含二次开发的优先修改建议
