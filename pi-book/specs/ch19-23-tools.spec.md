spec: task
name: "第六篇：工具设计 — 约束即保护"
inherits: project
tags: [chapter, part6, tools]
estimate: 2d
depends: [ch08-10-runtime]
---

## Intent

撰写第 19-23 章，解析 pi 的工具设计哲学：给 LLM 的工具接口越受约束，犯错越少。
覆盖设计原则、edit、read、bash、find/grep 五个维度。

## Decisions

- 第 19 章：总论，7 个核心工具 + 4 个辅助模块的职责分配
- 第 20 章配源码：editSchema（TypeBox 定义）+ withFileMutationQueue 串行化设计
- 第 22 章：bash 与 system prompt 中工具选择指引的关系
- 第 23 章：find/grep 从 bash 中拆出来的设计理由

## Boundaries

### Allowed Changes
- pi-book/chapters/ch19-tool-principles.md
- pi-book/chapters/ch20-edit-tool.md
- pi-book/chapters/ch21-read-tool.md
- pi-book/chapters/ch22-bash-tool.md
- pi-book/chapters/ch23-search-tools.md

### Forbidden
- 不要贴工具的完整 execute 函数实现
- 只展示 schema 定义和设计接口

## Completion Criteria

Scenario: 工具设计原则
  Test: review_ch19_principles
  Given 第 19 章 markdown 文件
  When 审阅设计原则段落
  Then 解释"结构化工具 vs 非结构化工具"的设计理由
  And 列出 7 个核心工具的职责

Scenario: edit 的 mutation queue
  Test: review_ch20_mutation_queue
  Given 第 20 章 markdown 文件
  When 审阅 edit 设计段落
  Then 解释精确替换（oldText → newText）的设计选择
  And 解释 file-mutation-queue 的并发串行化
  And 解释 EditOperations 接口的 pluggable I/O 设计

Scenario: bash 边界管理
  Test: review_ch22_bash_boundary
  Given 第 22 章 markdown 文件
  When 审阅 bash 工具段落
  Then 解释 system prompt 如何引导模型在 bash 和专用工具之间选择
  And 包含 BashOperations pluggable 接口的设计说明

Scenario: search 工具拆分理由
  Test: review_ch23_search_split
  Given 第 23 章 markdown 文件
  When 审阅 find/grep 段落
  Then 解释从 bash 中拆分的具体理由
  And 包含"多了工具增加选择负担 vs 降低搜索出错率"的取舍分析
