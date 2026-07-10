# pi 的设计艺术：构建生产级 Coding Agent 的架构决策

> 在线阅读：**[zhanghandong.github.io/pi-book](https://zhanghandong.github.io/pi-book/)**

本书解剖 [pi-mono](https://github.com/nicedoc/pi-mono) 项目的关键设计决策。不是源码导读，而是回答"为什么这样设计"——每个决策放弃了什么、得到了什么。

## 目录

| 篇 | 章节 | 核心问题 |
|----|------|---------|
| **序章** | 1-3 | pi 是什么、分层架构、如何读源码 |
| **第二篇** | 4-7 | 如何用一个接口统一 20+ 家 LLM 厂商 |
| **第三篇** | 8-10 | agent 循环引擎应该知道多少 |
| **第四篇** | 11-14 | 通用 runtime 如何变成可用产品 |
| **第五篇** | 15-18 | 哪些能力内建，哪些外置 |
| **第六篇** | 19-23 | 给 LLM 什么样的工具接口 |
| **第七篇** | 24-27 | 同一内核如何跑在终端/浏览器/RPC |
| **第八篇** | 28-29 | Slack bot 和 GPU 编排的复用实证 |
| **第九篇** | 30-32 | 设计哲学与适用边界 |

## 适合谁

- 有工程经验、想理解 agent 系统设计的开发者
- 准备给 pi 生态做扩展或贡献源码的人
- 想在自己的项目中借鉴 agent 架构经验的技术负责人

## 本地构建

```bash
# 安装 mdbook
cargo install mdbook

# 构建
mdbook build

# 本地预览（自动刷新）
mdbook serve --open
```

## 版本基线

本书核心分析基于 **pi-mono v0.66.0**（2026 年 4 月）。

## License

CC BY-NC-SA 4.0
