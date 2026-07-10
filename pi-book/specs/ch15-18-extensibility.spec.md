spec: task
name: "第五篇：能力外置"
inherits: project
tags: [chapter, part5, extension, skill]
estimate: 1.5d
depends: [ch11-14-product]
---

## Intent

撰写第 15-18 章，阐述 pi 最鲜明的设计哲学：极端的能力外置。
Extension 系统、Skill 机制、Resource Loader、Model Registry 四个维度。

## Decisions

- 第 15 章配源码：ExtensionApi 接口（展示 extension 能触达的系统表面积）
- 第 16 章配源码：skill frontmatter 示例 + skills.ts 发现逻辑
- 第 16 章包含 skill 发现与注入流程图（Mermaid）
- 重点：第 16 章必须讲清"为什么 skill 是 markdown 而不是 MCP"

## Boundaries

### Allowed Changes
- pi-book/chapters/ch15-extensions.md
- pi-book/chapters/ch16-skills.md
- pi-book/chapters/ch17-resource-loader.md
- pi-book/chapters/ch18-model-registry.md

### Forbidden
- 不要展示具体的 extension 实现代码
- 不要列举所有 ExtensionApi 方法，只展示设计表面积

## Completion Criteria

Scenario: Extension 边界清晰
  Test: review_ch15_extension_boundary
  Given 第 15 章 markdown 文件
  When 审阅 extension 系统段落
  Then 区分"extension 能做什么"和"extension 不能做什么"
  And 解释 loader → runner → wrapper 三层架构

Scenario: Skill 哲学
  Test: review_ch16_skill_philosophy
  Given 第 16 章 markdown 文件
  When 审阅 skill 设计段落
  Then 包含"skill 是 markdown 而非 MCP"的设计对比
  And 列出 skill 的取舍：零依赖、人类可审计、版本控制友好 vs 无运行时能力

Scenario: Model Registry 动态性
  Test: review_ch18_dynamic_registry
  Given 第 18 章 markdown 文件
  When 审阅 model registry 段落
  Then 解释 built-in models vs models.json 自定义
  And 解释 extension 注册新 provider 的机制
