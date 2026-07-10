spec: task
name: "第一篇：分层的纪律"
inherits: project
tags: [chapter, part1, architecture]
estimate: 1d
---

## Intent

撰写第 2-3 章，解释 pi 如何将一个庞大的 agent 系统切成互相不知道对方的层。
重点阐述 lockstep 版本策略、单向依赖纪律，以及如何高效阅读仓库。

## Decisions

- 第 2 章：七个包的分层关系，配 `package.json` workspace 配置展示依赖单向性
- 第 3 章：阅读地图，先读 vs 后读的文件列表
- 包含分层架构图（Mermaid），每层标注"知道什么 / 不知道什么"

## Boundaries

### Allowed Changes
- pi-book/chapters/ch02-packages.md
- pi-book/chapters/ch03-reading-map.md

### Forbidden
- 不要深入任何单个包的实现细节

## Completion Criteria

Scenario: 分层图准确
  Test: review_ch02_layer_diagram
  Given 第 2 章 markdown 文件
  When 审阅 Mermaid 分层图
  Then 包含全部 7 个包名
  And 依赖箭头只向下（tui/ai 在底层，coding-agent 在中层，mom/pods/web-ui 在顶层）

Scenario: lockstep 取舍分析
  Test: review_ch02_tradeoff
  Given 第 2 章 markdown 文件
  When 查找取舍分析段落
  Then 包含 lockstep 版本的"得到"和"放弃"

Scenario: 阅读地图实用
  Test: review_ch03_reading_map
  Given 第 3 章 markdown 文件
  When 审阅文件列表
  Then 包含"先读的 10 个文件"列表，每项有文件路径和理由
  And 包含"最后读的 10 个文件"列表
