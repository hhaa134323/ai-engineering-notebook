# LangGraph 入门：图编排 + Durable Execution = Agent harness 的开源积木

- 类型: 主题笔记
- 标签: AI / Agent
- 收录: 2026-05-18
- 原文: https://docs.langchain.com/oss/python/langgraph/overview

> LangGraph 不是 agent 框架而是「编排运行时」——用图（state/node/edge）取代链来表达 agent 的循环/分支/并行，再用 checkpoint/interrupt/store 三件套把 durable execution 做成开源原语。

---

**本质一句话**：LangGraph 是 agent 的「编排运行时」——把流程从『链』升级为『图』，再用 checkpoint/interrupt/store 三件套把『durable execution』做成开源原语。

**最高 ROI 单点改动**：长时任务一律 `compile(checkpointer=SqliteSaver(...))` + `thread_id`——崩了续跑，纯 Python while 复刻代价 1000+ 行。

**最容易被忽视的纪律/陷阱**：Node 只能返回 partial update（diff），不返回完整 state——这是并发安全的根基，违反就 race condition。

## 为什么需要这篇文章

传统/现状怎么做：用 for/while + if-else + try/except 串 LLM 调用 + 工具调用。Anthropic 的 Claude Code、Cursor agent 内部也是这套思想——但是闭源、绑死自家 SDK。

为什么传统做法坏了：

- 流程一旦有循环+分支+并行，纯代码迅速失控
- 崩了从头重跑 6 小时，没人能接受
- 「请用户确认这步」这种 HITL 自己写要管 queue + state machine + 续跑

这篇文档的视角：把 agent 编排抽象成状态机图——node 是函数、edge 是路由、state 是共享白板，再叠加 checkpoint/interrupt/store 这 3 个原语。Deep Agents 就是用这套积木拼出来的，证明这个抽象层是对的。

## 4 层心智模型

### Layer 1：为什么是「图」而不是「链」

**核心 mindset**：Agent 工作流不是直线，是有循环、分叉、并行的网络——所以编排数据结构必须是图（graph），不是链（chain）。

| 维度 | Chain（链） | Graph（图） |
| --- | --- | --- |
| 类比 | 工厂流水线 | 地铁线路图 |
| 支持循环 | 否 | 是 |
| 支持分支 | 否（顶多 if-else） | 是（任意路由函数） |
| 支持 fan-out 并行 | 否 | 是 |

```javascript
真正『非图不可』的 3 种形态：

(1) 循环          (2) 分支              (3) Fan-out
[A]→[B]             ┌→[X]                 ┌→[A]
 ↑   │        [LLM]─┼→[Y]           [入]─┼→[B]→[汇]
 └───┘              └→[Z]                 └→[C]
```

**原文金句**："LangGraph is very low-level, and focused entirely on agent orchestration"——文档明确定位 LangGraph 是编排层，不是 framework 也不是 harness。

**易错点**：以为「有 if-else = 必须用图」。判别真正需要图的关键词是「要不要回头?」——回头 = 循环 = 必须图。带条件的 fallback 只是带分支的链，try/except 就够。

### Layer 2：State + reducer + Partial Update

**核心 mindset**：State 不是普通 dict——它带「合并规则（reducer）」。每个 node 返回 partial update（diff）而不是完整 state，由 reducer 决定怎么合并进白板。

| 写法 | 行为 | 用途 |
| --- | --- | --- |
| 默认（无 reducer） | 覆盖（后来居上） | 当前快照：strategy、params、result、iter_count |
| `Annotated[list, operator.add]` | 追加 | 累积历史：history、tries |
| `Annotated[list, add_messages]` | 智能合并 + 去重 | 聊天消息（chatbot 不丢历史的关键） |
| 自定义 reducer | 任意逻辑 | 取 max、求和、deduplicate |

```javascript
没 reducer（默认覆盖）              有 reducer (operator.add)
Node A: {hist: ['t1']}                  Node A: {hist: ['t1']}
    → 白板: {hist: ['t1']}                  → 白板: {hist: ['t1']}
Node B: {hist: ['t2']}                  Node B: {hist: ['t2']}
    → 白板: {hist: ['t2']} t1 丢失         → 白板: {hist: ['t1','t2']}
```

**原文金句**："LangGraph is inspired by Pregel and Apache Beam"——partial update 模型直接来自 Pregel 的 BSP（Bulk Synchronous Parallel）范式。

**易错点**：以为 node 应该返回「完整 state」。错——并行 node 各自返回完整 state，合并时不知道听谁的，冲突无解。类比 Git：每次 commit 必须是 diff（patch），不是仓库快照。

### Layer 3：Durable Execution——LangGraph 真正的护城河

**核心 mindset**：每次 node 执行完，state 自动存档到数据库（checkpoint）。崩了能续跑、能暂停审核、能 time-travel 回滚——这才是 LangGraph 区别于「自己写 while 循环」的根本卖点。

| 能力 | 纯 Python while | LangGraph + checkpointer |
| --- | --- | --- |
| 崩了续跑 | 从头重来 | 同 thread_id 续跑 |
| 暂停审核 (HITL) | 自己写 queue（数百行） | `interrupt_before=["node_x"]` |
| Time-travel | 没存就没了 | 回到任意 checkpoint 改 state 重跑 |
| 多轮对话上下文 | 自己写 session 持久化 | thread_id 自动管理 |

关键 4 行代码：

```python
from langgraph.checkpoint.sqlite import SqliteSaver
checkpointer = SqliteSaver.from_conn_string("backtest.db")
graph = graph.compile(checkpointer=checkpointer)
config = {"configurable": {"thread_id": "session_42"}}
graph.invoke({"strategy": ...}, config=config)
# 崩了重启后：
graph.invoke(None, config=config)   # None = 续跑信号
```

**原文金句**："Durable execution: Build agents that persist through failures and can run for extended periods, resuming from where they left off"——文档把这一项列为 first core benefit。

**易错点 2 连**：

1. 不传 thread_id，每次都是新会话，续跑失效
2. 生产用 MemorySaver（进程内），进程一挂所有 checkpoint 没了；生产必须 SqliteSaver 或 PostgresSaver

**判断框架**：「能不能复刻」 vs 「开箱即用」是两个问题。LangGraph 的价值不是「做到 durable execution」——纯 Python 也行——而是「开箱即用 + 实战验证」。自己造代价是 1000+ 行 + 一堆并发/序列化坑。

### Layer 4：HITL + Store → Deep Agents 怎么「长」出来

**核心 mindset**：LangGraph 提供 4 块原语（StateGraph / Checkpoint / Interrupt / Store），Deep Agents 把它们组装成 harness（Planning / Subagent / Filesystem / Detailed Prompt）。没有 LangGraph 就没有 Deep Agents。

| 维度 | Checkpoint（短期） | Store（长期） |
| --- | --- | --- |
| 存什么 | 整个 state 快照 | 任意 KV 事实 |
| 绑定 | `thread_id`（会话） | `user_id` / 任意 namespace |
| 寿命 | 一次会话 | 跨周/月/年 |
| 类比 | 工作记忆（开会笔记） | 长期记忆（脑子里的偏好） |

```javascript
   Deep Agents (harness 套件)
┌──────────────────────────────────────┐
│ Planning Tool   (TODO list)          │
│ Subagent        (独立 state 子 graph)│
│ Filesystem      (虚拟文件)            │
│ Detailed Prompt (Claude Code 风)      │
└──────────────────────────────────────┘
           全部基于
┌──────────────────────────────────────┐
│  LangGraph 4 大原语                   │
│   StateGraph  → 流程编排（可递归）   │
│   Checkpoint  → 短期记忆/续跑        │
│   Interrupt   → HITL/人工介入        │
│   Store       → 长期记忆 + Filesystem│
└──────────────────────────────────────┘
```

**易错点**：以为 subagent = 父 graph 调一个普通 python 函数。错——subagent = 父 graph 里 invoke 另一个独立 StateGraph，有独立 state schema。这才是 context isolation 的实现方式。

**最 mind-blowing 的 take**：StateGraph 的递归性——父 harness 是 graph，每个 subagent 也是 graph，Planning 是 graph 里的一个 node。「全是图」是 LangGraph 的统一性。

## 三个「如果只能记一句」

| 问题 | 一句话 |
| --- | --- |
| **本质是什么** | LangGraph = agent 编排运行时；用「图 + 状态 + checkpoint」三件套把闭源 harness 的能力开源原语化 |
| **最高 ROI 实践** | 长时任务一律 compile(checkpointer=SqliteSaver(...)) + thread_id；崩了续跑代价归零 |
| **思维方式改变** | 不再问「我自己能不能写」，改问「原语 vs 开箱即用代价多大」——大多数 agent 工具的真实价值在这一层 |
