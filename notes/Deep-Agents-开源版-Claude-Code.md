# Deep Agents：开源版 Claude Code——「装好车」的 Agent Harness，4 大支柱 + LangGraph 原生 + 模型无关

- 类型：主题笔记
- 标签：AI / Agent / 文章
- 收录：2026-05-18
- 原文：<https://github.com/langchain-ai/deepagents>

> **本质一句话**：deepagents = 开源 + 模型无关 + LangGraph 原生的 Claude Code。它不是新框架（那是 LangGraph 的事），而是把通用 agent 需要的「车上配件」（planning / filesystem / sub-agents / 教模型用工具的 prompt）默认装好。
>
> **最高 ROI 单点改动**：如果你正打算用 raw LangGraph + `create_agent` 自己拼 harness——停。读 deepagents 源码，**把它当作你的「Claude Code 通用化的逆向工程参考实现」**，而不是从零搭。
>
> **最容易被忽视的陷阱**：以为 filesystem 工具给了你「无限上下文」——错。`write_file` 卸载到「文件」再 `read_file` 读回，每一轮都消耗 token。它只是把**管理负担**挪给了模型本身，不是物理边界的突破。

---

## L1：Harness vs Framework——deepagents 到底是什么

> **核心 mindset**："Instead of wiring up prompts, tools, and context management **yourself**, you get a working agent **immediately**"——README 原句。deepagents 解决的不是「LLM 怎么调工具」（那是 LangGraph 的事），是「**怎么让 LLM 调工具调得像一个能干活的人**」。

| 维度 | Framework（LangGraph / SDK） | Harness（deepagents / Claude Code） |
| --- | --- | --- |
| 抽象层级 | 积木：图 / 节点 / 状态 / checkpointer | 整车：装好工具 + 装好 prompt + 装好策略 |
| 第一行代码 | 定义节点 / 画边 / 写 system prompt | `create_deep_agent()` 一行出 agent |
| 默认带啥 | 什么都不带（自由度 100%） | 4 套工具 + 教模型用工具的 prompt |
| 类比 | Lego 积木桶 | Lego 已搭好的车，你换轮胎 |

> **原文金句**："This project was primarily inspired by Claude Code, and initially was largely **an attempt to see what made Claude Code general purpose, and make it even more so.**"
>
> → 产品定义是**逆向工程 Claude Code**——把它好用的那套通用配置抽出来开源。

> **易错点**：把 deepagents 和 LangGraph 当竞品。
>
> **为什么错**：deepagents 跑在 LangGraph 之上，`create_deep_agent()` 返回的就是一个 **compiled LangGraph**。它不是 LangGraph 的替代品，是 LangGraph 的「opinionated 默认应用层」。

## L2：能力边界三圈——知道它不做什么比知道它做什么更值钱

> **核心 mindset**：能力边界 = **ReAct loop + 固定工具集 + LangGraph runtime 的并集 − 你必须自己补的洞**。

```text
╔══════════════════════════════════════════════╗
║   Zone 3: 边界外（明确不做）                 ║
║                                              ║
║   ┌──────────────────────────────────────┐   ║
║   │ Zone 2: 可扩展（你接什么）           │   ║
║   │                                      │   ║
║   │  ┌──────────────────────────────┐    │   ║
║   │  │ Zone 1: 核心 11 项（开箱即用）│   │   ║
║   │  │  Planning / Filesystem /     │    │   ║
║   │  │  Shell / Sub-agents /        │    │   ║
║   │  │  Context mgmt / Smart prompts│    │   ║
║   │  └──────────────────────────────┘    │   ║
║   │  Backends / MCP / Skills /           │   ║
║   │  HITL / Permissions / 自定义工具     │   ║
║   └──────────────────────────────────────┘   ║
║   多用户协作 / RAG / Deterministic 工作流    ║
║   实时低延迟 / peer-to-peer 协商 / 模型救场  ║
╚══════════════════════════════════════════════╝
```

### Zone 1：核心能力（11 项）

| 能力 | 具体工具 | 解决什么病 |
| --- | --- | --- |
| Planning | `write_todos` | 任务拆解（**是工具不是 prompt**，强迫显式写下来） |
| Filesystem | `ls` `read_file` `write_file` `edit_file` `glob` `grep` | 大上下文卸载到「文件」 |
| Shell | `execute`（仅 sandbox backend） | 跑命令 / 测试 / build |
| Sub-agents | `task` | 派活 + 上下文隔离（hierarchical fan-out） |
| Interpreters | QuickJS 跑 TS/JS | 程序化组合工具、转结构化数据 |
| Auto-summarization | 对话太长自动压缩 | 防 context window 溢出 |
| Long-term memory | LangGraph Memory Store | 跨 thread 持久记忆 |
| Filesystem permissions | 声明式规则 | 限制可读 / 可写路径 |
| HITL | LangGraph `interrupt` | 敏感操作要人审批 |
| Skills | 文件夹形态可复用工作流 | 装专业打法（不是装新工具） |
| **Smart default prompts** | opinionated system prompts | **教模型怎么用这些工具——核心 trick** |

> **关键数字**：模型无关性是真的——**8 个 provider 一行切换**（Google / OpenAI / Anthropic / OpenRouter / Fireworks / Baseten / Ollama / 等）。`model="openrouter:deepseek/..."` 就跑 DeepSeek。

### Zone 3：做不了 / 容易翻车的 8 件事

| 不擅长 | 本质原因 | 替代 |
| --- | --- | --- |
| Deterministic 工作流 | 底层 ReAct loop，模型自主决定下一步 | raw LangGraph 节点 + 边自己画状态机 |
| 真正多 agent 协商 | `task` 是 fan-out，不是 peer-to-peer | LangGraph supervisor / swarm 模式 |
| 模型救场 | 默认 prompt 假设模型有强 tool calling 能力 | 没救——换强模型 |
| RAG / 向量检索 | filesystem 工具是文件操作不是 embedding | 包成工具注入 |
| 多租户 / RBAC | 核心库单 agent / 单线程设计 | 外层加 auth + per-user thread |
| 实时低延迟（< 1s） | tool call 都要回 graph 一轮 + planning 额外轮次 | 更轻 ReAct 或预编译路径 |
| 真 shell（默认） | 默认 filesystem 是虚拟的 | 用 Modal / Daytona / Deno sandbox |
| 「无限上下文」 | 卸到 file 再读回，每轮消耗 token | 没救——LLM 物理边界 |

> **4 个最容易翻车的场景**：
>
> **① 短任务 overkill**：不需要 planning / sub-agents 的小任务，让模型花 token 在 `write_todos` 上是浪费。→ 用 LangChain `create_agent`。
>
> **② 弱模型 + 默认 prompt = 灾难**：开源 7B/13B 可能根本不理解教它的 prompt。
>
> **③ 当 deterministic workflow engine 用**：例如金融交易**执行**（先校验 → 再下单 → 必须签名）——ReAct loop 可能跳步。
>
> **④ 误以为 filesystem 是真文件系统**：默认 in-memory backend 重启就丢。要持久化必须显式选 LangGraph Store 或本地磁盘。

## L3：选型决策——为什么不选其他

> **核心 mindset**：deepagents 的护城河 = **4 条交集**（batteries + 模型无关 + LangGraph runtime + 单 deep agent + 按需 sub-agent 范式）。**全开源界同时满足这 4 条的，只有它。**

### 真正同类替代（圈 1，4 个）

| 替代 | 核心理念 | 劣势 |
| --- | --- | --- |
| **smolagents**（HF） | 「Code as Action」——LLM 写 Python 代码而不是 JSON tool call | 没有 batteries（planning / sub-agent / filesystem 全要自己加）；要沙盒（安全负担大）；非 LangGraph |
| **DIY LangGraph + create_agent** | 自己拼装 | **一个月后你写的就是 deepagents 90%**——这就是 README 那句 "don't wire it up yourself" 的现实意义 |
| **Claude Agent SDK** | Anthropic 把 Claude Code 拆库 | 锁 Anthropic；非 LangGraph；半开源 |
| **PydanticAI** | 类型安全 + 极简 | 没有 batteries；偏单 agent 简单场景 |

### 不同范式（圈 3，不是替代品）

> **这一圈经常被误以为是 deepagents 替代品，其实不是**。它们解决**不同的 multi-agent 拓扑**问题：
>
> - **CrewAI**：role-based pipeline（虚拟公司：CEO/Engineer/QA 协作）
> - **Microsoft Agent Framework**（前 AutoGen 已停更）：conversational debate（agents 互相聊天）
> - **OpenAI Agents SDK**（前 Swarm 已弃用）：handoff swarm（控制权移交）
> - **Google ADK**：Google 生态一站式

## L4：Multi-agent 不是一个东西——5 种拓扑

> **核心 mindset**：「我要 multi-agent」是**不够精确的需求**。先问：**哪种拓扑**？

| 拓扑 | 形态 | 框架代表 | deepagents 行不行 |
| --- | --- | --- | --- |
| Hierarchical fan-out | 主 agent → N 子 agent（派活 + 回报，子之间不通信） | **deepagents** | 这就是它干的 |
| Role-based pipeline | 预设角色 + 任务流 | CrewAI | 要去圈 3 |
| Conversational debate | Agents 互相对话 / 辩论 | Microsoft Agent Framework | 要去圈 3 |
| Handoff swarm | 移交控制权（接力棒） | OpenAI Agents SDK | 要去圈 3 |
| 任意自定义图 | 节点 = agent | **raw LangGraph** | 你自己画 |

> **2026 圈 3 地壳变动**：
>
> - **AutoGen** 已进入 **maintenance mode** → 用 Microsoft Agent Framework
> - **OpenAI Swarm** 已弃用 → 用 OpenAI Agents SDK

## 三个「如果只能记一句」

| 问题 | 一句话 |
| --- | --- |
| 本质是什么 | deepagents = 开源 + 模型无关 + LangGraph 原生的 Claude Code——逆向工程 Claude Code 的通用化打包 |
| 最高 ROI 实践 | 选型时先匹配 4 条交集（batteries / 模型无关 / LangGraph / hierarchical 范式）——任一条命中就选 deepagents，**否则继续往下问** |
| 思维方式改变 | 「要 multi-agent」不是需求——「**要哪种 multi-agent 拓扑**」才是。光选框架不分拓扑会买错车 |
