spec: task
name: "第四篇：从 Runtime 到产品"
inherits: project
tags: [chapter, part4, product, coding-agent]
estimate: 2d
depends: [ch08-10-runtime]
---

## Intent

撰写第 11-14 章，解释通用 agent runtime 如何变成可用的 coding agent 产品。
重点是四个产品化设计：会话树、Compaction、三级配置覆盖、System Prompt 装配。

## Decisions

- 第 11 章配源码：`SessionEntry` 类型联合 + `SessionHeader` 结构
- 第 11 章包含 session tree 图（Mermaid），展示分支、压缩、跳转
- 第 12 章配源码：`CompactionEntry.details` 字段 + `extractFileOperations()`
- 第 14 章配源码：system-prompt.ts 的装配逻辑

## Boundaries

### Allowed Changes
- pi-book/chapters/ch11-session-tree.md
- pi-book/chapters/ch12-compaction.md
- pi-book/chapters/ch13-config-layers.md
- pi-book/chapters/ch14-system-prompt.md

### Forbidden
- 不要展示 interactive mode 的 UI 组件代码

## Completion Criteria

Scenario: 会话树设计讲清
  Test: review_ch11_session_tree
  Given 第 11 章 markdown 文件
  When 审阅会话树段落
  Then 解释 JSONL + parentId 的树形结构
  And 列出 9 种 entry 类型并解释各自用途
  And 包含 session tree 的 Mermaid 图
  And 包含 JSONL append-only 的取舍分析

Scenario: Compaction 文件追踪
  Test: review_ch12_file_tracking
  Given 第 12 章 markdown 文件
  When 审阅 compaction 段落
  Then 解释压缩时追踪 readFiles 和 modifiedFiles 的设计
  And 解释 extension 可以通过 fromHook 接管 compaction

Scenario: 三级配置有具体示例
  Test: review_ch13_config_example
  Given 第 13 章 markdown 文件
  When 审阅配置段落
  Then 包含全局、项目、目录三级配置的具体路径示例
  And 解释合并规则

Scenario: Prompt 装配流程
  Test: review_ch14_prompt_assembly
  Given 第 14 章 markdown 文件
  When 审阅 prompt 装配段落
  Then 列出装配来源的优先级顺序
  And 包含"动态拼接让 prompt 难以预测最终形态"的取舍分析
