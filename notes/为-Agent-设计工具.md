# 为 Agent 设计工具：从契约重写到评估驱动

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://anthropic.com/engineering/writing-tools-for-agents

> 工具是确定性系统与非确定性 agent 之间的契约——必须按 prompt 工程的思维去设计，按 agent loop eval 的方式去验证。

---

**一句话**：工具是确定性系统与非确定性 agent 之间的契约。不能再用「写 API」的直觉写它——要用「写 prompt」的思维设计，用「跑 eval」的方式验证。

**最高 ROI 单点改动**：把 description 按「给实习生入职文档」的标准重写一遍。

**最容易被忽视的工程纪律**：先 eval、再上线；不要为了「看起来整洁」就 rename。

## 为什么需要这篇文章

传统软件里，API 的调用方是另一段确定性代码：你写 `client.get(url)`，调完进入下一步，整条流程都是可预测的。

Agent 时代，调用方变了：调用方是 LLM。它可能不调你的工具、可能调错、可能传错参数、可能调一半就因为 context 满了卡死。这是一个全新的、非确定性的契约边界。

这篇文章就是回答一个问题：在这种新契约下，工具应该怎么设计？

## 7 层心智模型

### 1. 契约：确定性 ↔ 非确定性

**核心 mindset**：Tools are a new kind of software which reflects a contract between deterministic systems and non-deterministic agents.

这一层决定了后面所有层的存在意义——因为调用方是非确定性的，你必须主动为 agent 设计，而不是把现有 API 包一层就当 tool。

### 2. Agent affordances：context 是稀缺资源

Context window 是 agent 的「工作记忆」。一个坏工具不仅自己不好用，还会污染整个 context。

两个推论：

- 单 tool 内：只返回必要信息，不要 dump 所有字段（通讯录 5000 人不要全返回，要 `search_contacts(name)`）
- 多 tool 间：按工作流合并，不要 API 一对一映射。文章举例 `schedule_event` 合并了 `list_users` + `list_events` + `create_event` 三步

### 3. 返回有意义的上下文

| 动作 | 反例 | 正例 |
| --- | --- | --- |
| 用语义 ID | `run_id: "a3f2-9b1c-..."` | `run_id: "momentum_v2_20241015_1430"` |
| 留 high-signal 字段 | concise 里塞 mime_type、config_hash | concise 里留 sharpe_ratio、status——决策字段必须默认可见 |
| 提供切换开关 | 只有一种返回形态 | `response_format: "concise" | "detailed"`（Anthropic 实测 Slack 工具 206 → 72 tokens，省 2/3） |

**易错点**：直觉会把 sharpe_ratio 这种「重要数据」放到 detailed 里。错。决策字段必须在 concise 里——否则 agent 必须对所有候选 run 都 fetch detail，变成暴力扫描，context 爆炸。

### 4. 错误响应是教程，不是诊断报告

传统错误：`{ "error_code": 422, "message": "Invalid date" }`——给开发者看。

Agent 友好错误：分字段、给格式、给例子、给候选值、给换工具提示。

关键洞见：每一条附加信息激活 agent 的不同自救路径：

| 错误响应里的信号 | 触发的 agent 行为 |
| --- | --- |
| `expected_format` · `example` | 改参数重试（格式错） |
| `did_you_mean: [...]` | 改参数重试（语义错 / 拼写错） |
| `hint`: 指向另一个工具 | 换工具策略（先去查可用值再回来） |

### 5. 命名空间：service > resource，但慎重 rename

当 agent 挂载几十个工具，tool selection 变成搜索问题。

好实践：

- 一级用 service 前缀：`asana_*`, `jira_*`, `slack_*`
- 二级用 resource：`asana_projects_search`, `asana_users_search`
- 用下划线显式分隔——`strategy_list` 优于 `strategylist`
- 工具名要含动词：`market_get_kline` 不是 `market_kline`

**陷阱**：读完文章后不要立刻回头给已有工具大改名。Rename 的代价：所有 prompt / eval / 文档 / 集成全部失效，eval baseline 重建。只在真出现冲突或工具数量超过 30-50 时才动。新加领域加前缀即可，老工具留着——必要时用 deprecation alias。

### 6. 描述工程：最高 ROI 的单点改动

**今天最重要的 mindset 转变**：Description 不是文档，是 prompt。Description 是每次模型调用都会被塞进 context 的那段指令。写它的时候要问的不是「我描述清楚了我的工具吗」，而是「agent 读完会做对决策吗」。

四块写满，agent 就会用对：

```javascript
Use when:    描述何时该用这个工具
Do NOT use:  描述何时不该用、应改用什么工具
Returns:     描述返回值结构，包括下一步该调什么
Example:     给一个完整调用示例
```

参数命名也要消歧义：

- 反例：`user` → 名字？邮箱？ID？对象？
- 正例：`user_id`（文章原话："instead of a parameter named user, try a parameter named user_id"）
- 反例：`filter: object`（无子 schema）→ agent 会幻觉一个不存在的结构
- 正例：`filter` 给子 schema + enum + 字段描述

**真实案例**：Claude 在 web search 时总自动给 query 加 "2025"——污染结果。修复方法不是改模型、不是改代码，是 description 加一句："Do not append years to the query unless explicitly asked." 一行文字修复整个工具行为。

### 7. 评估驱动：闭环工程

所有上面的原则都是假设——必须用 eval 来验证。

```javascript
prototype → 手动试 → 写 eval tasks → 跑 agent loop
    ↑                                       ↓
    └──── 改 description / schema / 响应 ←── 读 transcript 找 rough edges
```

三个反直觉细节：

1. 用真实工作流任务，不是 toy task。toy 只测「能不能跑」，真实任务才暴露「工具组合在一起的真实问题」
2. Agent 不会告诉你它困惑哪里。必须读原始 transcript，看它实际调了什么、漏调了什么
3. 让 Claude 优化它自己的工具。把 transcripts + tool 代码丢进 Claude Code，让它分析并改 description——Anthropic 实测：Claude 优化出来的工具在 held-out 测试集上仍然显著优于专家手写。文章原话："Most of the advice in this post came from repeatedly optimizing our internal tool implementations with Claude Code."

**一句话**：工具工程是经验科学，不是教条。没有 eval 的 best practice 都是猜的；有 eval 的蠢办法也比没 eval 的理论完美更靠谱。

## 三个「如果只能记一句」

| 问题 | 答案 |
| --- | --- |
| Agent 工具的本质 | 工具是确定性系统与非确定性 agent 之间的契约——它在 agent loop 的每一步里被 agent 重新决定要不要用、怎么用 |
| 最高 ROI 实践 | 重写 description（按 Use when / Do NOT use / Returns / Example 四块模板，给实习生入职的标准） |
| 思维方式改变 | Description 是 prompt，不是文档——写 tool 时随时反问「agent 读完会做对决策吗」，而不是「我描述清楚了吗」 |
