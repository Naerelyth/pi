spec: project
name: "pi-book"
tags: [book, documentation, design]
---

## Intent

撰写一本关于 pi-mono 项目设计艺术的技术书籍。全书以设计决策为骨架，源码为佐证，
面向有工程经验、想真正理解 agent 系统设计的开发者。

## Constraints

- 每章以一个设计问题开头，回答"为什么这样做"而非"代码怎么写"
- 每个关键设计决策必须标注取舍（得到什么、放弃什么）
- 源码引用精准到文件和行号，只在需要解释设计时出场
- 中文写作，技术术语保留英文
- 每章 3000-6000 字，不超过 8000 字
- 代码片段只展示设计相关的核心部分，不贴大段实现

## Decisions

- 书名: 《pi 的设计艺术：构建生产级 Coding Agent 的架构决策》
- 大纲: 参照 `outline.md` (v2 修订版)
- 章节文件: `chapters/ch{NN}-{slug}.md` 格式
- 图示: 使用 Mermaid 语法，放在章节内
- 源码引用格式: `packages/ai/src/api-registry.ts:66-78`

## Boundaries

### Allowed Changes
- pi-book/**

### Forbidden
- 不要修改 pi-mono 源码
- 不要创建与书无关的文件
- 不要在章节中贴超过 40 行的连续代码块
