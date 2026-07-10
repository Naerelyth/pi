# 《深入解读 pi-mono：从 LLM 抽象到 Coding Agent 产品化》大纲

生成日期：2026-04-09

## 写作定位

这本书建议定位为：

- 面向有工程经验、想真正看懂 agent 系统设计的开发者
- 兼顾准备给 `pi` 生态做扩展或贡献源码的人

写法建议：

- 以“运行链路”为主轴，而不是按目录机械导览
- 以“产品形态”为案例，而不是把每个 package 当成孤立项目
- 重点讲清设计约束、抽象边界、数据流和事件流

---

## 全书主线

这本书不应按仓库目录逐包展开，而应围绕这样一条主线：

1. 一个用户请求如何进入系统
2. 请求如何被转成统一的模型调用
3. 模型响应如何进入 agent loop
4. 工具调用如何变成状态转移
5. 会话、压缩、重试、资源装配如何把 runtime 变成产品
6. 同一套内核如何落在终端、浏览器、Slack 和远端模型部署场景里

---

## 序章

## 1. 为什么值得解读 `pi-mono`

### 小节

- AI 编程代理正在从“产品形态”回到“基础设施层”
- `pi-mono` 和一般 AI SDK、CLI、聊天 UI 的区别
- 全书阅读方法：看边界、看链路、看约束，不陷入实现噪音

### 本章目标

先回答“它到底是什么”，而不是“目录里有什么”。

### 建议图示

- 仓库总览图

---

## 第一篇：单仓与分层

## 2. 单仓全景：七个包不是七个项目

### 小节

- `pi-ai` 负责什么
- `pi-agent-core` 负责什么
- `pi-coding-agent` 为什么是产品内核
- `pi-tui`、`pi-web-ui`、`mom`、`pods` 分别扮演什么角色

### 本章目标

建立全书最重要的一张分层图。

### 建议图示

- 分层架构图：`pi-ai -> pi-agent-core -> pi-coding-agent -> tui/web-ui/mom/pods`

## 3. 先搭阅读地图：哪些文件是骨架，哪些文件是细节

### 小节

- 最先读的 10 个文件
- 最晚读的 10 个文件
- 如何避免一开始陷入 TUI 和交互细节

### 本章目标

告诉读者怎样高效读码。

### 建议图示

- 推荐读码顺序图

---

## 第二篇：从模型调用开始

## 4. `pi-ai` 的核心抽象：Model、Provider、Api

### 小节

- 为什么把 provider 和 api 分开
- provider 注册机制如何工作
- `stream` / `complete` 为什么故意保持很薄

### 重点源码

- `packages/ai/src/stream.ts`

### 本章目标

说明 `pi-ai` 不是“某家厂商 SDK 的简单封装”，而是统一调用面。

## 5. 消息为什么不能原样传递

### 小节

- 不同 provider 的消息约束
- thinking block 的保留、降级与丢弃
- tool call ID 归一化
- 为什么要补 synthetic tool result

### 重点源码

- `packages/ai/src/providers/transform-messages.ts`

### 本章目标

解释跨模型 handoff 的真实复杂度。

### 建议图示

- 跨 provider 消息变换前后对照图

## 6. 统一事件流设计

### 小节

- 为什么选择流式事件而不是一次性响应
- `start`、`delta`、`done` 的价值
- 错误与中止如何进入统一消息模型

### 本章目标

让读者理解“事件流是整个系统的最底层脉搏”。

---

## 第三篇：Agent Runtime 的核心循环

## 7. `AgentLoop`：整个系统真正的发动机

### 小节

- turn 的定义
- `prompt` 与 `continue` 的区别
- 消息何时注入上下文
- 循环何时退出

### 重点源码

- `packages/agent/src/agent-loop.ts`

### 本章目标

把一轮 agent 执行拆成可解释的状态机。

## 8. 工具调用不是插件，而是状态转移

### 小节

- tool call 的提取
- 参数验证
- `beforeToolCall` / `afterToolCall` 的位置
- `parallel` 与 `sequential` 的差别

### 本章目标

说明工具系统如何嵌入循环，而不是外挂在外面。

### 建议图示

- 带工具执行的 turn 时序图

## 9. 为什么还需要 `Agent` 这一层

### 小节

- `AgentLoop` 与 `Agent` 的职责分离
- 状态封装
- 订阅器屏障
- steering queue 和 follow-up queue

### 重点源码

- `packages/agent/src/agent.ts`

### 本章目标

解释“循环”与“运行时对象”为什么必须分开。

---

## 第四篇：从 Runtime 到产品内核

## 10. `AgentSession`：`pi-coding-agent` 的真正中心

### 小节

- 为什么所有模式都围绕 `AgentSession`
- 会话持久化
- 事件转发
- 模型与工具管理
- 产品级职责为何集中在这里

### 重点源码

- `packages/coding-agent/src/core/agent-session.ts`

### 本章目标

读者必须把这一章看成全书核心。

## 11. Session 不只是历史记录

### 小节

- JSONL 会话结构
- branch 与 fork
- 树形导航
- 为什么“会话树”比“聊天记录列表”更适合 coding agent

### 本章目标

解释这个项目最有辨识度的数据模型之一。

### 建议图示

- session tree 图

## 12. Compaction 与 Retry：把不稳定模型包进稳定产品

### 小节

- context overflow 如何被检测
- 自动压缩何时触发
- 错误重试策略
- 为什么 retry 和 compaction 要放在产品层而不是模型层

### 本章目标

把“代理可用性”问题讲透。

### 建议图示

- overflow / retry 流程图

---

## 第五篇：Prompt、Skills、Resources

## 13. system prompt 是怎么被拼出来的

### 小节

- 默认 prompt
- `SYSTEM.md` 与 `APPEND_SYSTEM.md`
- 项目级 `AGENTS.md`
- 工具片段与规范片段如何进入最终 prompt

### 重点源码

- `packages/coding-agent/src/core/system-prompt.ts`

### 本章目标

解释 prompt 不是一个字符串，而是一套装配流程。

## 14. Skills 机制：把能力外包给文档

### 小节

- skill frontmatter
- 自动发现
- 命名与冲突
- 为什么 skill 是“可读指令包”而不是 MCP

### 重点源码

- `packages/coding-agent/src/core/skills.ts`

### 本章目标

讲清这个仓库最鲜明的哲学选择。

### 建议图示

- skill 发现与注入流程

## 15. Resource Loader：一切资源的统一入口

### 小节

- extensions、skills、prompts、themes 的加载顺序
- 全局与项目作用域
- 包管理与资源扩展
- 冲突检测

### 重点源码

- `packages/coding-agent/src/core/resource-loader.ts`

### 本章目标

说明产品是如何动态长出新能力的。

## 16. Model Registry：模型配置、认证与扩展性

### 小节

- built-in models
- `models.json`
- provider override
- OAuth 与 API key
- 动态 provider 注册

### 重点源码

- `packages/coding-agent/src/core/model-registry.ts`

### 本章目标

解释“模型选择”背后其实是配置系统和认证系统。

---

## 第六篇：工具系统与受约束的文件操作

## 17. 为什么 `read` 不是简单的 `cat`

### 小节

- 偏移与分页读取
- 截断策略
- 图片读取
- 为什么工具设计要替模型兜底

### 重点源码

- `packages/coding-agent/src/core/tools/read.ts`

### 本章目标

让读者看到“工具是带护栏的接口”。

## 18. 为什么 `edit` 不是直接写文件

### 小节

- 精确替换
- 多处编辑的一致性
- diff 生成
- 文件变更队列
- 为什么编辑工具要限制模型表达方式

### 重点源码

- `packages/coding-agent/src/core/tools/edit.ts`

### 本章目标

解释 coding agent 工具设计的核心原则。

## 19. bash 工具与外部世界的边界

### 小节

- 什么时候该用 bash，什么时候不该
- 结构化工具与非结构化工具的权衡
- 产品如何管理 bash 带来的风险与复杂性

### 本章目标

把“万能工具”与“专用工具”的关系讲清楚。

---

## 第七篇：两套 UI，同一颗内核

## 20. `pi-tui`：终端里做应用，而不是刷日志

### 小节

- 差分渲染
- overlay
- 聚焦与输入分发
- 硬件光标与 IME
- 为什么 TUI 实现会如此复杂

### 重点源码

- `packages/tui/src/tui.ts`

### 本章目标

说明终端 UI 的复杂度并不低于 Web UI。

### 建议图示

- render cycle 图

## 21. 编辑器组件为什么会这么大

### 小节

- 多行编辑
- 自动补全
- 粘贴处理
- 滚动与光标
- 它如何和上层 agent 交互

### 本章目标

解释为何交互体验的难点集中在 editor。

### 建议图示

- 输入事件流图

## 22. `pi-web-ui`：浏览器里的复用而不是重写

### 小节

- `ChatPanel` 和 `AgentInterface`
- storage 层
- artifacts
- sandbox runtime provider
- 浏览器环境下的限制与补偿

### 重点源码

- `packages/web-ui/src/ChatPanel.ts`

### 本章目标

说明 Web UI 只是另一层宿主，而不是另一套 agent。

### 建议图示

- Web UI 组件关系图

---

## 第八篇：产品化案例

## 23. `mom`：把 coding agent 变成 Slack 同事

### 小节

- 为什么它复用了 `AgentSession`
- Slack 上下文如何映射成 prompt
- memory 与 skills 如何频道化
- 线程回复与工具日志的产品决策

### 重点源码

- `packages/mom/src/agent.ts`

### 本章目标

展示宿主化的真实方式。

## 24. `pods`：为什么这个仓库还要管 GPU pod

### 小节

- 远程模型部署与 agent 开发的关系
- vLLM 管理
- OpenAI-compatible endpoint
- 本地 agent 与远端推理服务的连接方式

### 重点源码

- `packages/pods/src/commands/pods.ts`

### 本章目标

解释这个单仓为何同时覆盖“模型供给侧”和“代理消费侧”。

---

## 第九篇：设计哲学与取舍

## 25. 极简核心，能力外置

### 小节

- 为什么很多系统做成内建特性，这里却做成 extensions / skills / packages
- 这种选择带来的收益与代价

### 本章目标

提炼作者的方法论。

## 26. 反主流选择背后的判断

### 小节

- 为什么不内建 sub-agents
- 为什么不内建 MCP
- 为什么不内建 permission popup
- 为什么不内建 to-do 和 plan mode

### 本章目标

把 README 里的哲学变成架构判断。

## 27. 这套架构适合什么，不适合什么

### 小节

- 适合做什么产品
- 哪些团队会喜欢
- 哪里会踩坑
- 如果要二次开发，优先改哪里

### 本章目标

帮读者形成最终判断。

---

## 附录

1. 核心类型表：`Model`、`Context`、`AgentMessage`、`AgentEvent`、`ToolDefinition`
2. 全书代码索引：每章重点文件
3. 一次完整请求的时序图
4. 二次开发入口清单：加模型、加工具、加 skill、加 extension、加 UI

---

## 建议写作顺序

优先写：

1. 第 1 到 10 章，先把主干立住
2. 第 13 到 18 章，再把机制讲透
3. 第 20 到 24 章，用产品案例把抽象落地
4. 第 25 到 27 章，最后收束成设计哲学和判断标准

这样不容易写成“源码流水账”，也更适合逐章连载或边写边补图。

---

## 后续可继续补充的内容

- 每章摘要
- 每章目标读者
- 每章预计字数
- 每章必读源码文件清单
- 每章建议插图和时序图
- 可作为专栏连载的拆分方案
