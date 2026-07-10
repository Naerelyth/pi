spec: task
name: "序章：不是又一个 LLM 包装器"
inherits: project
tags: [chapter, prologue]
estimate: 0.5d
---

## Intent

撰写第 1 章，建立全书的阅读框架。让读者理解 pi 是一个 agent **运行时**而非调用库，
明确全书的阅读方法：看决策、看边界、看取舍。

## Decisions

- 包含 pi 与通用 AI SDK（LangChain/Vercel AI SDK）的本质区别
- 包含"洋葱架构图"（Mermaid）：从 pi-ai 到 pi-agent-core 到 pi-coding-agent
- 不超过 4000 字

## Boundaries

### Allowed Changes
- pi-book/chapters/ch01-prologue.md

### Forbidden
- 不要在序章深入任何技术细节
- 不要贴源码

## Completion Criteria

Scenario: 章节结构完整
  Test: review_ch01_structure
  Given 第 1 章 markdown 文件
  When 审阅章节结构
  Then 包含"设计问题"开头
  And 包含"洋葱架构图"（Mermaid 代码块）
  And 包含 pi 与其他 SDK 的对比段落
  And 字数在 2000-4000 之间

Scenario: 不含源码
  Test: review_ch01_no_code
  Given 第 1 章 markdown 文件
  When 检查代码块
  Then 不包含任何 TypeScript/JavaScript 代码块
