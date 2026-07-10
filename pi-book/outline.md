# 《pi 的设计艺术：构建生产级 Coding Agent 的架构决策》

修订日期：2026-04-10
基于 Codex v1 大纲修订，Claude 对照源码深度分析后重构

## 定位

不是源码导读，而是**设计决策的解剖**。

每一章回答一个设计问题，源码只在"为什么这样做"时出场。读者读完应该能回答：
- pi 做了哪些关键设计决策？
- 每个决策放弃了什么、得到了什么？
- 如果我要做类似系统，哪些可以直接借鉴，哪些需要根据场景变通？

面向读者：有工程经验、想真正理解 agent 系统设计的开发者。

---

## 全书主线

不按目录展开，而是沿着一条设计张力线：

> **如何用尽可能薄的内核，撑起尽可能丰富的产品？**

pi 的回答是一套"洋葱架构"：统一 LLM 抽象 → 无状态循环引擎 → 有状态运行时 → 产品壳。每一层只知道下一层的接口，不知道上层的存在。这条线贯穿全书。

---

## 序章：这本书在讲什么

### 1. 不是又一个 LLM 包装器

- 为什么 agent 系统的难点不在调 API，而在"调完之后怎么办"
- pi 和通用 AI SDK（LangChain/Vercel AI SDK）的本质区别：它是一个**运行时**，不是一个**调用库**
- 全书阅读方法：看决策、看边界、看取舍，不看实现细节

**建议图示**：一张"洋葱图"，从 pi-ai 到 pi-agent-core 到 pi-coding-agent，每层标注它的职责和它**不知道**什么

---

## 第一篇：分层的纪律

> 核心问题：如何把一个庞大的 agent 系统切成互相不知道对方的层？

### 2. 七个包不是七个项目

- 设计决策：lockstep 版本 — 所有包始终同版本号
- 依赖关系只能向下：`tui ← ai ← agent-core ← coding-agent ← mom/pods/web-ui`
- 每一层对上层完全无知的纪律
- **取舍**：lockstep 版本意味着每次发布都要升全部包，换来的是永远不会有版本兼容问题

**配源码**：`package.json` 的 workspace 配置，展示依赖关系的单向性

**建议图示**：分层架构图，每层标注"知道什么 / 不知道什么"

### 3. 怎样高效阅读这个仓库

- 先读的 10 个文件 vs 最后读的 10 个文件
- 为什么不要从 TUI 开始读
- "骨架文件"和"细节文件"的区分标准

---

## 第二篇：统一调用面 — pi-ai 的设计

> 核心问题：20+ 家 LLM 厂商，消息格式各异，如何用一个接口统一？

### 4. Provider 不是 Adapter

- 设计决策：provider 和 api 的分离 — 同一个 provider 可能暴露多种 api
- `ApiProvider` 接口为什么只有两个方法：`stream` 和 `streamSimple`
- 插件式注册：`registerApiProvider()` 和 `getApiProvider()`
- **取舍**：provider 运行时注册（而非编译时），牺牲了静态分析能力，换来了无限扩展性

**配源码**：`api-registry.ts` 全文（只有 98 行，是极简设计的范例）

### 5. 消息变换：跨模型交接的隐藏复杂度

- 设计问题：用户在 Claude 上聊了 50 轮，现在要切到 GPT — 历史消息怎么办？
- thinking block 的保留、降级与丢弃策略
- tool call ID 的跨 provider 归一化（OpenAI 450+ 字符 ID vs Anthropic 64 字符限制）
- `transformMessages()` 的设计：一个函数解决三个问题
- **取舍**：消息变换是有损的，但"有损交接"好过"不能交接"

**配源码**：`transform-messages.ts` 的 tool call ID 归一化逻辑

**建议图示**：跨 provider 消息变换前后对照图

### 6. 事件流：整个系统的脉搏

- 设计决策：所有交互都是流式事件，没有"一次性返回"
- `start → delta → done` 的三段式设计
- 错误不是异常 — 错误被编码进事件流（`stopReason: "error"`）
- **StreamFn 的契约**：`types.ts:23-26` 的注释是整个系统最重要的一段话 — "Must not throw"
- **取舍**：流式设计让每个消费者都要写状态机，但它让会话录制、回放、分布式追踪成为可能

**配源码**：`StreamFn` 类型定义及其契约注释

### 7. OAuth：统一认证的隐藏复杂度

- 为什么一个 LLM 抽象层还要管认证？
- 5 种 OAuth 流程（Anthropic、GitHub Copilot、Google Antigravity、Gemini CLI、OpenAI Codex）的统一
- PKCE 流程的共享实现
- **取舍**：把认证放在 ai 层而非产品层，因为 token 刷新必须在每次 LLM 调用前发生

**建议图示**：OAuth 流程统一抽象图

---

## 第三篇：Agent Runtime — 循环引擎的设计

> 核心问题：一个 agent 循环应该知道多少？

### 8. `agentLoop`：发动机只管转

- 设计决策：循环引擎是**无状态**的纯函数
- 双层循环的精妙设计：
  - 内层：处理 tool call 和 steering 消息
  - 外层：处理 follow-up 消息（agent 本来要停下来，但有新任务到达）
- 消息变换管道：`AgentMessage[] → transformContext → convertToLlm → Message[] → LLM`
- 为什么转换只发生在 LLM 调用边界（文件头注释：第 1-4 行）
- **取舍**：无状态循环意味着它不能自己管理会话、不能自己重试，但它可以被任何上层随意组合

**配源码**：`agent-loop.ts` 的 `runLoop()` 函数（核心 ~70 行）

**建议图示**：双层循环状态机图

### 9. 工具执行不是插件调用

- 设计决策：tool call 是循环的一部分，不是外挂
- `prepare → execute → finalize` 三阶段设计
- `beforeToolCall` / `afterToolCall` 钩子的位置选择
- parallel vs sequential 的实现差异：都先 prepare，parallel 并发 execute，sequential 串行
- TypeBox schema 做参数验证 — 模型犯错时工具层兜底
- **取舍**：钩子增加了每次工具调用的开销，但它让安全审计、速率限制、权限控制成为可能

**配源码**：`prepareToolCall()` → `executePreparedToolCall()` → `finalizeExecutedToolCall()` 链条

### 10. `Agent`：循环之上的有状态壳

- 为什么 `AgentLoop` 和 `Agent` 必须分开？
- `Agent` 拥有什么：transcript、listener set、steering/follow-up 两个队列、abort controller
- `MutableAgentState` 的设计：外部看 readonly，内部可写；数组赋值自动 copy
- `PendingMessageQueue` 的 `all` vs `one-at-a-time` 模式
- `CustomAgentMessages` 的声明合并：让应用层可以定义自己的消息类型，同时保持类型安全
- **取舍**：Agent 刻意不管持久化、不管 UI、不管认证 — 它只是"一个会管理自己状态的循环"

**配源码**：`Agent` 类的 `processEvents()` 方法 — 展示状态归约的完整逻辑

**建议图示**：Agent 作为循环+状态+队列的组合体

---

## 第四篇：从 Runtime 到产品 — pi-coding-agent 的设计

> 核心问题：一个通用 agent runtime 如何变成一个可用的 coding agent 产品？

### 11. 会话树：比"聊天记录"更好的数据模型

- 设计问题：coding agent 的会话不是线性对话，用户会回溯、分支、重试
- JSONL + parentId 的树形结构
- 6 种 entry 类型的设计：`message | compaction | branch_summary | model_change | thinking_level_change | custom`
- `custom` entry 的用途：extension 持久化状态而不污染 LLM context
- Branch 和 fork 的区别
- **取舍**：JSONL 没有随机访问能力（必须流式读取全部 entries），但它 append-only、容错性极强、人类可读

**配源码**：`SessionEntry` 类型联合 + `SessionHeader` 结构

**建议图示**：session tree 图，展示分支、压缩、跳转

### 12. Compaction：把无限对话装进有限窗口

- 设计问题：用户和 agent 聊了 200 轮，context window 装不下了
- 两种压缩：对话摘要压缩 + 分支摘要
- 文件操作追踪：压缩时不只记摘要，还记录 `readFiles` 和 `modifiedFiles`
- 为什么 compaction 放在产品层而不是 runtime 层
- Extension 可以接管 compaction（`fromHook` 标记）
- **取舍**：自动压缩是有损的，但"丢失细节"好过"context overflow 导致请求失败"

**配源码**：`CompactionEntry` 的 `details` 字段设计 + `extractFileOperations()` 的文件追踪逻辑

### 13. 三级配置覆盖

- 设计决策：全局 `~/.pi/agent/` → 项目 `.pi/` → 目录树上溯 `AGENTS.md`
- Settings Manager 如何合并三级配置
- `AGENTS.md` / `CLAUDE.md` 的自动发现和拼接
- **取舍**：三级覆盖增加了心智负担，但让"全局默认 + 项目定制 + 目录级规则"成为可能

### 14. System Prompt 是一套装配流程

- 不是一个字符串，而是多个来源动态拼接
- 默认 prompt → `SYSTEM.md` → `AGENTS.md` → 工具片段 → skill 片段
- 拼接顺序的设计意图：后来的可以覆盖先来的
- **取舍**：动态拼接让 prompt 难以预测最终形态，但让系统能适应任何项目和任何用户偏好

**配源码**：system-prompt.ts 的装配逻辑

---

## 第五篇：能力外置 — Extension、Skill、Tool 的设计哲学

> 核心问题：哪些能力应该内建，哪些应该外置？pi 选择了极端的外置。

### 15. Extension 系统：让产品长出新器官

- Extension 能做什么：订阅事件、注册工具、注册命令、注册快捷键、操作 UI
- `loader → runner → wrapper` 三层架构
- Extension 的 API 面：它能看到 `SessionManager`、`ModelRegistry`、`EventBus`、`KeybindingsManager`
- Extension 不能做什么：不能修改 agent loop 的核心行为
- **取舍**：开放但有边界 — extension 可以加能力，不能改规则

**配源码**：`ExtensionApi` 接口（展示 extension 能触达的系统表面积）

### 16. Skill 机制：用文档替代代码

- 设计决策：skill 不是 MCP endpoint，不是插件，而是**带 frontmatter 的 markdown 文件**
- 为什么这个选择如此反直觉但又如此有效
- Skill 的发现、命名冲突、注入流程
- **取舍**：skill 没有运行时能力（不能调 API），但它零依赖、人类可审计、版本控制友好

**配源码**：skill frontmatter 示例 + skills.ts 的发现逻辑

**建议图示**：skill 发现与注入流程

### 17. Resource Loader：一切外部资源的统一入口

- extensions、skills、prompts、themes 的统一加载顺序
- 全局与项目作用域的合并规则
- 为什么要有一个统一入口而不是各自加载

### 18. Model Registry：模型不只是一个 ID

- built-in models vs `models.json` 自定义
- provider override 机制
- 动态 provider 注册（extension 可以注册新 provider）
- **取舍**：动态注册让系统可以支持任何 LLM，但也意味着运行时才知道模型是否可用

---

## 第六篇：工具设计 — 约束即保护

> 核心问题：给 LLM 什么样的工具接口，才能让它犯最少的错？

### 19. 设计原则：工具是带护栏的接口

- 为什么不给 LLM 一个万能 bash 就够了
- 结构化工具（read/write/edit/find/grep/ls）vs 非结构化工具（bash）的设计理由
- 7 个核心工具 + 4 个辅助模块的职责分配
- TypeBox schema 做参数验证：模型犯错时工具层兜底

### 20. `edit` 的设计：为什么不能直接写文件

- 精确替换（oldText → newText）而非行号编辑
- `file-mutation-queue`：多个并发编辑的串行化
- `edit-diff`：行尾归一化、BOM 处理、diff 生成
- `EditOperations` 接口：pluggable I/O（本地 fs / SSH / 远程）
- **取舍**：精确替换限制了模型的表达方式，但几乎消除了"写错文件"的可能

**配源码**：`editSchema`（TypeBox 定义）+ `withFileMutationQueue` 的串行化设计

### 21. `read` 的设计：为什么不是简单的 `cat`

- 偏移与分页读取、截断策略、图片读取
- 为什么返回带行号的文本而非原始内容
- 工具如何替模型兜底

### 22. `bash` 与外部世界的边界

- 什么时候该用 bash，什么时候不该（system prompt 里的指引）
- 产品如何管理 bash 带来的风险
- `BashOperations` 接口：和 edit 一样的 pluggable 设计

### 23. `find` 和 `grep`：结构化搜索替代万能 bash

- 为什么要把 `find` 和 `grep` 从 bash 中拆出来做成独立工具
- 让模型用结构化参数搜索，而非自己拼 shell 命令
- **取舍**：多了两个工具增加了模型的选择负担，但大幅降低了搜索出错率

---

## 第七篇：UI 层 — 同一颗内核的不同宿主

> 核心问题：一个 agent 内核如何同时跑在终端、浏览器、Slack 里？

### 24. pi-tui：在终端里做应用

- 设计决策：不用 Ink.js，自建 TUI 框架
- `Component` 接口的极简设计：`render(width): string[]`
- 差分渲染 + synchronized output（CSI 2026）
- Overlay 系统、Focus 管理、IME 支持
- 35+ 自定义组件的组合模式
- **取舍**：自建框架意味着更多维护成本，但换来了完全的控制力

**配源码**：`Component` 接口定义（3 个方法，是极简设计的另一个范例）

### 25. 编辑器组件：交互复杂度的集中地

- 多行编辑、自动补全、粘贴处理、滚动与光标
- `@` 文件引用、`!` bash 执行、Tab 路径补全
- 为什么编辑器组件的代码量超过很多独立项目

### 26. RPC 模式：pi 作为后端服务

- 为什么一个 CLI 工具需要 RPC 模式？
- JSON-L 协议设计
- 使用场景：IDE 插件、Web 前端、自动化管道
- **取舍**：增加了一个运行模式，但让 pi 可以被任何前端驱动

### 27. pi-web-ui：浏览器里的复用

- Lit Web Components + Tailwind 的技术选型
- 如何复用 pi-ai 的 provider 和 pi-tui 的组件模式
- 浏览器环境的限制与补偿

---

## 第八篇：产品化实证

> 核心问题：同一套内核在不同产品形态里是怎么被复用的？

### 28. `mom`：Slack 里的 Coding Agent

- 如何复用 `AgentSession` 把 agent 变成 Slack 同事
- channel 级数据隔离：每个频道独立的 memory、context、skills
- Docker sandbox 的安全设计
- 线程回复与工具日志的产品决策
- **取舍**：Slack 的消息限制和线程模型约束了交互设计，但也强制了更好的信息组织

**配源码**：mom 的数据目录结构

### 29. `pods`：为什么这个仓库还要管 GPU

- 远程模型部署与 agent 开发的关系
- vLLM + OpenAI-compatible endpoint 的选择
- **取舍**：把部署工具放在 agent 仓库里看起来职责混乱，但它确保了"端到端可控"

---

## 第九篇：设计哲学

> 核心问题：这些决策背后有什么统一的思想？

### 30. 极简核心，能力外置

- 内核只做三件事：调模型、跑循环、管状态
- 所有可选能力（skill、extension、prompt template、theme）都是外部资源
- 为什么"能力外置"比"功能内建"更适合 agent 系统
- 与 Claude Code、Cursor、Windsurf 等产品的设计对比

### 31. 反主流选择背后的判断

- 为什么不内建 sub-agents（用 tool call 模拟就够了）
- 为什么不内建 MCP（skill 更轻量且人类可审计）
- 为什么不内建 permission popup（beforeToolCall 钩子更灵活）
- 为什么不内建 plan mode（transformContext 可以实现任何上下文策略）
- **每个"不做"背后都是一个"用更底层的机制组合出来"**

### 32. 这套架构的适用边界

- 适合什么产品：需要深度定制的 agent 系统
- 不适合什么产品：需要开箱即用的简单 chatbot
- 哪些团队会喜欢：重视工程纪律、有能力写 extension 的团队
- 如果要二次开发，优先改哪里

---

## 附录

### A. 核心类型速查表

| 类型 | 所在包 | 用途 |
|------|--------|------|
| `Model<TApi>` | pi-ai | 模型定义 |
| `Context` | pi-ai | LLM 调用上下文 |
| `AssistantMessageEvent` | pi-ai | 流式事件 |
| `AgentMessage` | pi-agent-core | 扩展消息类型 |
| `AgentTool<TParams>` | pi-agent-core | 工具定义 |
| `AgentEvent` | pi-agent-core | 循环生命周期事件 |
| `SessionEntry` | pi-coding-agent | 会话持久化条目 |

### B. 关键设计模式索引

| 模式 | 出现位置 | 章节 |
|------|---------|------|
| 插件注册 | api-registry.ts | 第 4 章 |
| 有损变换 | transform-messages.ts | 第 5 章 |
| 流式契约 | StreamFn type | 第 6 章 |
| 双层循环 | agent-loop.ts runLoop() | 第 8 章 |
| 三阶段工具执行 | agent-loop.ts prepare/execute/finalize | 第 9 章 |
| 声明合并扩展 | CustomAgentMessages | 第 10 章 |
| Append-only 树 | session-manager.ts | 第 11 章 |
| Pluggable I/O | EditOperations / BashOperations | 第 20 章 |
| 极简组件接口 | Component { render() } | 第 24 章 |

### C. 一次完整请求的时序图

用户输入 → Editor → Agent.prompt() → agentLoop() → transformContext → convertToLlm → streamSimple → Provider.stream → 事件流 → processEvents → UI 渲染 → tool call → beforeToolCall → execute → afterToolCall → 下一轮 → agent_end → session 持久化

### D. 二次开发入口清单

| 我想... | 起点 |
|---------|------|
| 加一个新 LLM provider | `registerApiProvider()` + models.json |
| 加一个新工具 | Extension API → `registerTool()` |
| 加一个新 slash command | Extension API → `registerSlashCommand()` |
| 改 system prompt | `SYSTEM.md` 或 `AGENTS.md` |
| 改 compaction 策略 | Extension hook: `onBeforeCompaction` |
| 加一个新 UI 模式 | 参考 `modes/rpc/` 实现 |
| 支持新的消息类型 | `CustomAgentMessages` 声明合并 |

---

## 与 v1 大纲的主要差异

| 维度 | v1（Codex） | v2（修订） |
|------|------------|-----------|
| 主线 | 运行链路 | **设计决策链** |
| 章节组织 | 按模块/代码结构 | 按设计问题 |
| 每章开头 | 小节列表 | **设计问题 + 决策 + 取舍** |
| 源码角色 | 重点源码（必读） | 佐证源码（选读） |
| OAuth | 未提及 | 独立一节（第 7 章） |
| Extension 系统 | 一笔带过 | 独立一章（第 15 章） |
| RPC 模式 | 未提及 | 独立一节（第 26 章） |
| find/grep 工具 | 未提及 | 独立一节（第 23 章） |
| 配置系统 | 未提及 | 独立一节（第 13 章） |
| file-mutation-queue | 未提及 | 融入 edit 章节（第 20 章） |
| 设计模式索引 | 核心类型表 | **设计模式 + 类型 + 二次开发入口** |

## 建议写作顺序

1. **第 8-10 章**（Agent Runtime）— 这是全书技术核心，先写最难的
2. **第 4-7 章**（pi-ai）— Runtime 的基石，第二优先
3. **第 11-14 章**（产品化）— 把抽象落地
4. **第 15-18 章**（能力外置）— pi 最鲜明的哲学
5. **第 19-23 章**（工具设计）— 具体且可借鉴的设计经验
6. **第 30-32 章**（设计哲学）— 最后升华
7. **第 1-3 章**（序章和导航）— 全书写完后回头写
8. **第 24-29 章**（UI 和产品化实证）— 穿插完成
