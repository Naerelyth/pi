spec: task
name: "第八篇：产品化实证"
inherits: project
tags: [chapter, part8, mom, pods]
estimate: 1d
depends: [ch11-14-product]
---

## Intent

撰写第 28-29 章，展示 pi 内核在 Slack bot (mom) 和 GPU 编排 (pods) 两个产品中
的复用方式。用实际案例验证前面讲的架构设计。

## Decisions

- 第 28 章配源码：mom 的数据目录结构
- 第 28 章重点：channel 级数据隔离 + Docker sandbox 的安全设计
- 第 29 章篇幅较短，重点讲"为什么 agent 仓库还要管模型部署"

## Boundaries

### Allowed Changes
- pi-book/chapters/ch28-mom-slack.md
- pi-book/chapters/ch29-pods-gpu.md

### Forbidden
- 不要展示 Slack API 调用的细节代码
- 不要展示 vLLM 部署的运维命令

## Completion Criteria

Scenario: mom 数据隔离设计
  Test: review_ch28_data_isolation
  Given 第 28 章 markdown 文件
  When 审阅 mom 设计段落
  Then 包含 per-channel 数据目录结构示例
  And 解释 Docker sandbox 的安全边界
  And 包含"Slack 消息限制约束交互设计"的取舍分析

Scenario: pods 定位清晰
  Test: review_ch29_pods_rationale
  Given 第 29 章 markdown 文件
  When 审阅 pods 段落
  Then 解释"端到端可控"的设计理由
  And 字数在 2000-3000 之间（短篇）
