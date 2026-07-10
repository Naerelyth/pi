spec: task
name: "第二篇：统一调用面 — pi-ai 的设计"
inherits: project
tags: [chapter, part2, pi-ai]
estimate: 2d
depends: [ch02-03-layering]
---

## Intent

撰写第 4-7 章，解剖 pi-ai 层的四个核心设计决策：provider 插件注册、跨模型消息变换、
流式事件契约、OAuth 统一认证。每章聚焦一个设计问题及其取舍。

## Decisions

- 第 4 章配源码：`api-registry.ts` 全文（98 行，极简设计范例）
- 第 5 章配源码：`transform-messages.ts` 的 tool call ID 归一化逻辑
- 第 6 章配源码：`StreamFn` 类型定义及其契约注释（types.ts:23-26）
- 第 7 章配图：OAuth 流程统一抽象图（Mermaid）
- 第 5 章包含跨 provider 消息变换前后对照图

## Boundaries

### Allowed Changes
- pi-book/chapters/ch04-provider-registry.md
- pi-book/chapters/ch05-message-transform.md
- pi-book/chapters/ch06-event-stream.md
- pi-book/chapters/ch07-oauth.md

### Forbidden
- 不要展示 provider 的具体实现代码（anthropic.ts, openai-responses.ts 等）
- 只展示设计层面的接口和类型

## Completion Criteria

Scenario: api-registry 源码展示
  Test: review_ch04_source
  Given 第 4 章 markdown 文件
  When 检查源码引用
  Then 包含 `registerApiProvider` 函数的代码块
  And 包含对"为什么只有 stream 和 streamSimple 两个方法"的解释

Scenario: 消息变换取舍
  Test: review_ch05_tradeoff
  Given 第 5 章 markdown 文件
  When 查找取舍分析
  Then 包含"有损交接好过不能交接"的论述
  And 包含 tool call ID 归一化的具体例子（OpenAI 450+ 字符 vs Anthropic 64 字符限制）

Scenario: StreamFn 契约
  Test: review_ch06_contract
  Given 第 6 章 markdown 文件
  When 查找流式契约段落
  Then 包含"Must not throw"契约的引用和解释
  And 包含错误编码进事件流而非抛异常的设计理由

Scenario: OAuth 覆盖完整
  Test: review_ch07_oauth
  Given 第 7 章 markdown 文件
  When 检查 OAuth 覆盖
  Then 至少提及 3 种 OAuth 流程（Anthropic, GitHub Copilot, Google 系列）
  And 解释为什么认证放在 ai 层而非产品层
