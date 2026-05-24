# Agent 委派者守则：能用 create_react_agent 就别搭图——「简单优先」判定法

- 类型: 方法论笔记
- 标签: AI / Agent
- 收录: 2026-05-18
- 原文: https://academy.langchain.com/courses/intro-to-langgraph

> 作为「不写代码、只委派 AI 写」的人，最大的杠杆不是学更多 LangGraph 原语，而是建立一个「能简单就不搞复杂」的拒绝清单——逼 AI 在选 prebuilt 还是自搭 StateGraph 之前先回答 4 个判定题，把 80% 的需求挡在图编排之外。

---

## 为什么需要这篇笔记

**传统/现状怎么做**：学一个新框架（LangGraph / Mastra / Pydantic AI）后，下意识地「凡是 agent 需求都用它写」，因为新工具有快感、显得专业。

**为什么这样坏了**：

- AI agent（让它写代码的那个 AI）也有同样的偏好——它会自动选「能展示框架能力」的方案，而不是最简方案
- 结果：一个本来 30 行 Python 能搞定的 RAG 问答，被写成 StateGraph + Checkpoint + Subagent + Store 五件套，多 500 行代码、5 倍 debug 难度、3 个新故障点
- 作为「委派者」（不亲自写代码的人），你没有「写到一半发现太复杂」的反馈回路——AI 写完交给你时已经成型，沉没成本让你倾向接受

**这篇笔记用什么视角重新回答问题**：把「选框架 vs 选简单方案」做成一个前置判定流程——在让 AI 动手前强制走一遍，把 80% 的「过度设计」挡在第一道门外。

## 3 层心智模型

### Layer 1：AI 的「过度设计偏好」是默认开启的

> **核心 mindset**：AI 写代码时的默认行为不是「最简」，而是「展示能力 + 防御未来需求」。你不喊停，它就把 LangGraph 全家桶都端上桌。

| AI 的隐性目标 | 会导致什么 | 你看不见的代价 |
| --- | --- | --- |
| 「展示我懂 LangGraph」 | 简单需求也用 StateGraph | 多 5-10 倍代码 |
| 「防御未来需求」 | 提前加 Checkpoint / Store / Interrupt | 多 3 个故障点 + 部署复杂度 |
| 「显得专业」 | 抽象层套抽象层（subagent / 自定义 reducer） | 你 review 不动 = 失控 |
| 「迎合『agent』这个词」 | 任何「带 LLM 的脚本」都套 agent 框架 | 本可以是 cron + 函数 |

类比：让实习生写 CRUD，他会主动给你上 DDD + CQRS + Event Sourcing——不是他坏，是他在用「显得专业」对冲「被嫌弃水平不够」的恐惧。AI agent 同理。

> **易错点**：以为「我说『请用最简方案』就够」。不够——AI 对「简」的判定和你不同。必须给结构化判定题（见 Layer 2），让它有可对照的标准。

### Layer 2：4 道判定题——任意 No 才考虑搭图

> **核心 mindset**：「要不要用 LangGraph」不是品味问题，是有 4 个客观判定条件。4 个全 Yes → 一定别用；任意 No → 才进入「考虑用」阶段。

| # | 判定题 | Yes（不用图） | No（可能要图） |
| --- | --- | --- | --- |
| 1 | 流程是不是真直线（≤5步、无回头、无循环）？ | 纯 Python / LCEL | 有循环 → 可能要图 |
| 2 | 是不是单次 LLM + ≤3 个 tool 的 ReAct 模式？ | `create_react_agent` 一行 | 多 LLM 协作 / 复杂工具编排 → 可能要图 |
| 3 | 是不是完全无状态（一次性、无 follow-up、无续跑）？ | 普通函数 | 要记忆 / 要续跑 → 可能要图 |
| 4 | 团队是否有人能 debug 异步状态机？ | 有 → 可以考虑图 | 没有 → 先用同步链跑通 |

```
判定流程图：

         需求来了
            │
            ▼
      ┌──────────────┐
      │ 4 题全 Yes?  │
      └──────┬───────┘
         Yes │  No
      ┌─────┘   └──────┐
      ▼                 ▼
  别用 LangGraph    进入「升级阶梯」
  用降级阶梯              （Layer 3）
```

**关键句**：LangGraph 原文金句 *"LangGraph is very low-level, and focused entirely on agent orchestration"*——**编排层**这个定位反过来说就是：没有真正的「编排需求」就别用。直线流程 = 没有编排需求。

> **易错点**：把「有 if-else」当成「需要图」。错——分支 ≠ 编排。判别真正需要图的关键词是「要不要回头？」（循环）+「要不要并行 fan-out？」+「要不要 HITL 暂停？」。3 个里 ≥1 个才算真编排需求。

### Layer 3：3 阶降级阶梯——每升一级，代价指数级增长

> **核心 mindset**：从「纯函数」到「自搭 StateGraph」中间有 3 个台阶。每上一级，代码量翻 2 倍、debug 难度翻 3 倍、新故障点 +2。永远从最低一级开始，被现实逼着才升级。

| 级别 | 方案 | 适用场景 | 代码量基准 | 新增故障点 |
| --- | --- | --- | --- | --- |
| L0 | **纯 Python 函数** | 无 LLM 或单次 LLM 调用、无状态 | 30-80 行 | 0 |
| L1 | **LCEL / 链**（LangChain 表达式） | 直线 pipeline：retrieve → format → LLM → parse | 50-150 行 | +1（链运行时） |
| L2 | **`create_react_agent`** 预制 agent | LLM + tools 循环到结束（90% 的 tool-using agent） | 20-50 行（短！） | +1（agent runtime） |
| L3 | **自搭 StateGraph** | 需循环/并行/HITL/subagent/time-travel 至少一项 | 200-1000+ 行 | +5（state/reducer/checkpoint/interrupt/部署） |

```
    代码量
     ▲
1000 ┤                            ╱╱ L3 自搭 StateGraph
     │                          ╱╱
 300 ┤                       ╱╱
     │                    ╱╱
 100 ┤              ▄▄▄▄▄  L1 LCEL
     │      ▄▄▄▄▄▄        ▄▄▄ L2 create_react_agent（甚至比 L1 短）
  30 ┤▄▄▄▄  L0 纯函数
     └──────────────────────────────▶ 复杂度
```

**重点**：L2 比 L1 可能更短——因为 `create_react_agent` 把 tool 循环打包了。所以判定题 Q2 = No（多 LLM 协作）时，优先升 L3 而不是 L2；但 Q2 = Yes 时直接 L2 不要走 L1。

> **易错点 2 连**：
>
> 1. 「想未来扩展，先按 L3 搭」——反模式。未来真要扩展时再升级，重写成本远小于一直背着 L3 的重量
> 2. 「L2 不灵活，跳过直接 L3」——`create_react_agent` 接受 `state_modifier` / 自定义 tool 选择策略 / 自定义终止条件，比想象中灵活。先用它扳 3 个月再说

## 三个「如果只能记一句」

| 问题 | 一句话 |
| --- | --- |
| **本质是什么** | 委派者的最大杠杆不是学更多框架，而是建立「拒绝过度设计」的前置判定流程 |
| **最高 ROI 实践** | 每次让 AI 写 agent 前，强制 prompt 第一句：「先回答能不能用 create_react_agent 或纯 Python 完成，不能再列具体升级理由」 |
| **思维方式改变** | 从「我想用什么框架」改为「这个需求能不能不用框架」——抵抗 AI 和工程师共有的过度设计本能 |

## 委派 prompt 第一句模板（拷走即用）

```
在动手写代码前，先回答：
这个需求能不能用 create_react_agent 或纯 Python 完成？
如果不能，列出必须用 StateGraph 的具体原因
（参考升级触发条件：循环 / 并行 fan-out / HITL / subagent 隔离 / time-travel）。
```
