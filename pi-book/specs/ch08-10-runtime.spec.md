spec: task
name: "第三篇：Agent Runtime — 循环引擎的设计"
inherits: project
tags: [chapter, part3, runtime, core]
estimate: 2d
depends: [ch04-07-pi-ai]
---

## Intent

撰写第 8-10 章，这是全书技术核心。解剖 agent-core 层的三层设计：无状态循环引擎、
工具执行管道、有状态 Agent 壳。重点讲清为什么 AgentLoop 和 Agent 必须分开。

## Decisions

- 第 8 章配源码：`agent-loop.ts` 的 `runLoop()` 函数（核心 ~70 行）
- 第 9 章配源码：`prepare → execute → finalize` 三阶段链条
- 第 10 章配源码：`Agent` 类的 `processEvents()` 方法
- 第 8 章包含双层循环状态机图（Mermaid）
- 第 10 章包含 Agent 作为循环+状态+队列的组合体图

## Boundaries

### Allowed Changes
- pi-book/chapters/ch08-agent-loop.md
- pi-book/chapters/ch09-tool-execution.md
- pi-book/chapters/ch10-agent-class.md

### Forbidden
- 不要展示 coding-agent 层的工具实现（那是第六篇）
- 不要展示 TUI 层如何消费事件（那是第七篇）

## Completion Criteria

Scenario: 双层循环讲清
  Test: review_ch08_dual_loop
  Given 第 8 章 markdown 文件
  When 审阅循环设计段落
  Then 清晰区分内层循环（tool call + steering）和外层循环（follow-up）
  And 包含 `runLoop()` 的简化代码展示
  And 包含双层循环状态机的 Mermaid 图

Scenario: 无状态取舍
  Test: review_ch08_stateless_tradeoff
  Given 第 8 章 markdown 文件
  When 查找取舍分析
  Then 解释无状态循环"不能自己管理会话、不能自己重试"的限制
  And 解释这带来的组合灵活性

Scenario: 三阶段工具执行
  Test: review_ch09_three_phases
  Given 第 9 章 markdown 文件
  When 审阅工具执行段落
  Then 清晰区分 prepare、execute、finalize 三阶段
  And 解释 beforeToolCall/afterToolCall 钩子的插入位置
  And 对比 parallel 和 sequential 模式的实现差异

Scenario: Agent vs AgentLoop 分离
  Test: review_ch10_separation
  Given 第 10 章 markdown 文件
  When 审阅 Agent 类设计
  Then 解释 Agent 拥有什么（transcript、listeners、queues、abort controller）
  And 解释 Agent 不管什么（持久化、UI、认证）
  And 包含 CustomAgentMessages 声明合并的设计解释
