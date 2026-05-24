# ai-engineering-notebook

这是一份个人学习笔记的公开版本。原始版本在 Notion 中维护，这里是清洁后可公开阅读的版本，每篇笔记附原文链接。

## 索引

按收录时间倒序排列。

| 标题 | 一句话 | 类型 | 标签 | 收录时间 |
| --- | --- | --- | --- | --- |
| [为 Agent 设计工具：从契约重写到评估驱动](./notes/为-Agent-设计工具.md) | 工具是确定性系统与非确定性 agent 之间的契约——必须按 prompt 工程的思维去设计，按 agent loop eval 的方式去验证 | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [LLM Tool Use 全景：从机制到生产工程](./notes/LLM-Tool-Use-全景.md) | 模型只会生成文本；tool use 是一套结构化协议，让 harness 替模型动手，并通过 agent loop 把结果喂回去 | 主题笔记 | AI / Agent | 2026-05-18 |
| [Agent SDK 概览：Claude Code 拆出来当库用](./notes/Agent-SDK-概览.md) | Agent SDK 把 Claude Code 的内核拆出来当库用，让你在 95 分起点上做业务定制 | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [Agent Skills：写成文件夹的方法论手册](./notes/Agent-Skills.md) | Skill = 一个含 SKILL.md 的文件夹，把指令+子文件+脚本打包给 Agent，通过渐进式披露按需加载 | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [LangGraph 入门：图编排 + Durable Execution](./notes/LangGraph-入门.md) | LangGraph 不是 agent 框架而是「编排运行时」——用图取代链来表达 agent 的循环/分支/并行 | 主题笔记 | AI / Agent | 2026-05-18 |
| [Deep Agents 的 context 管理](./notes/Deep-Agents-的-context-管理.md) | Long-running agent 在 context 满之前就会因 context rot 退化——三段压缩 + 双轨保存 + 主动加压的 targeted eval | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [LangChain Agent Middleware：把怕 LLM 犯浑的事抢回来交给代码](./notes/LangChain-Agent-Middleware.md) | Agent middleware = 6 个 hook 组成的可插拔机制，让确定性代码包住概率性的 LLM-工具循环 | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [Agent 委派者守则：能用 create_react_agent 就别搭图](./notes/Agent-委派者守则.md) | 委派者最大的杠杆不是学更多 LangGraph 原语，而是建立「能简单就不搞复杂」的拒绝清单 | 方法论笔记 | AI / Agent | 2026-05-18 |
| [Code Execution × MCP：让 Agent 写代码，不当调度员](./notes/Code-Execution-MCP.md) | 让 LLM 写代码调用 MCP，而不是逐个 tool call，单任务 token 可从 150k → 2k（-98.7%） | 主题笔记 | AI / Agent | 2026-05-18 |
| [Agent Harness 解剖：Model 是智能，Harness 才是你能动的杠杆](./notes/Agent-Harness-解剖.md) | Agent = Model + Harness。模型只贡献智能，凡是围绕智能让它变成有用工作的工程系统全都是 harness | 主题笔记 | AI / Agent / 文章 | 2026-05-18 |
| [Harness Engineering 实战：固定模型只动 harness，52.8→66.5](./notes/Harness-Engineering-实战.md) | 固定模型、只动 Prompt/Tools/Middleware 三个旋钮 + Trace 反馈闭环，把 Terminal Bench 2.0 推 +13.7 分 | 主题笔记 | AI / Agent / 文章 / 编程 | 2026-05-18 |

## 目录结构

- `notes/` 单篇笔记 Markdown 文件
- `notes/index.md` 详细索引（与本 README 表格内容一致，可单独浏览）
- `notes/assets/` 图片等附件

## 同步说明

本仓库由从 Notion 公开版数据库同步生成，请勿直接修改 `notes/` 中的文件。
如需修订，请在 Notion 中编辑后重新运行同步流程。

最后更新于 2026-05-24
