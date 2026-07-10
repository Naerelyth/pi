spec: task
name: "附录"
inherits: project
tags: [chapter, appendix]
estimate: 0.5d
depends: [ch08-10-runtime, ch19-23-tools]
---

## Intent

撰写附录 A-D，提供全书的速查参考。核心类型表、设计模式索引、完整请求时序图、
二次开发入口清单。

## Decisions

- 附录 A：核心类型速查表（7 个关键类型）
- 附录 B：设计模式索引表（模式 → 文件 → 章节）
- 附录 C：一次完整请求的时序图（Mermaid sequence diagram）
- 附录 D：二次开发入口清单（"我想做 X → 从 Y 开始"表格）

## Boundaries

### Allowed Changes
- pi-book/chapters/appendix.md

### Forbidden
- 不要在附录中添加正文章节已覆盖的长篇论述

## Completion Criteria

Scenario: 时序图完整
  Test: review_appendix_sequence
  Given 附录 markdown 文件
  When 检查时序图
  Then 包含 Mermaid sequence diagram
  And 覆盖从"用户输入"到"session 持久化"的完整链路

Scenario: 二次开发入口实用
  Test: review_appendix_entry_points
  Given 附录 markdown 文件
  When 检查入口清单
  Then 至少覆盖 7 个常见二次开发场景
  And 每个场景指向具体的函数或文件路径
