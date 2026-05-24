# Agent SDK 概览：Claude Code 拆出来当库用

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://code.claude.com/docs/zh-CN/agent-sdk/overview

> Agent SDK 把 Claude Code 的内核（agent loop + 10+ 内置工具 + context 管理）拆出来当库用，让你在 95 分起点上做业务定制——别从底层往上堆，从最高抽象层开始问「够不够用」。

---

**一句话**：Agent SDK = Claude Code 的内核（loop + 内置工具 + context）拆出来当库用，让你在 95 分起点上做业务定制。

**工程价值**：它替你写好了那套 harness loop，还附赠 10+ 个被 Anthropic 内部 eval 过的生产级工具。

**一个思维陷阱**：做 agent 选型时别默认「底层越定制」——底层 = 更多重复劳动，不是更大自由。

## 4 层心智模型

### 1. 本质：loop 谁来写

```python
# 方案 A：Client SDK——你自己写 loop
response = client.messages.create(...)
while response.stop_reason == "tool_use":
    result = your_tool_executor(response.tool_use)
    response = client.messages.create(tool_result=result, **params)

# 方案 B：Agent SDK——SDK 帮你写好了整个 loop
async for message in query(prompt="Fix the bug in auth.py"):
    print(message)
```

这就是 Agent SDK 的全部本质。两件事：

1. 它替你写了 `while stop_reason == "tool_use"` 循环
2. 它自带 10+ 个内置工具，你不用自己实现

### 2. 内置工具：95 分起点

| 工具 | 能力 |
| --- | --- |
| **Read** | 读工作目录任意文件 |
| **Write** | 创建新文件 |
| **Edit** | 对现有文件精确编辑 |
| **Bash** | 跑终端命令、脚本、git 操作 |
| **Glob** | 按模式找文件 |
| **Grep** | 用正则搜文件内容 |
| **WebSearch / WebFetch** | 搜网 / 取网页内容 |
| **Monitor** | 监听后台脚本输出 |
| **AskUserQuestion** | 向用户提多选项澄清问题 |

**最底层的洞见**：Read / Glob / Bash ... 每一个都是 Anthropic 用 Claude 自己反复迭代 + held-out eval 过的成品。你不仅省了 7 层设计——你省的是有 eval 数据支撑的、生产级品质的 7 层设计。

### 3. 6 类扩展点

| 扩展点 | 它解决什么问题 | 关键 API |
| --- | --- | --- |
| **Hooks** | 生命周期点插回调（审计/校验/通知） | `PreToolUse`, `PostToolUse`, `Stop`, `SessionStart` |
| **Permissions** | 限制 agent 能用哪些工具，需不需要批准 | `allowed_tools=[...]`, `permission_mode` |
| **MCP** | 接入你自己的系统 / 外部服务 | `mcp_servers={...}` |
| **Sessions** | 多轮持续：记住之前读了什么、聊了什么 | `session_id` · `resume=session_id` |
| **Subagents** | 派专家处理子任务（context 隔离 + 错误隔离 + 可并行） | `agents={...}` · allowed_tools 含 `"Agent"` |
| **.claude/ 配置** | 把方法论/命令/记忆沉淀成文件，不是写死在代码里 | `.claude/skills/*/SKILL.md`, `.claude/commands/*.md`, `CLAUDE.md` |

### 4. Subagents 是树状，不是网状

```javascript
    主
   ╱ │ ╲
  子 子 子      主 ↔ 子
 ╱ │
孙 孙          子 ↔ 孙（子 agent 也开放 Agent 工具）
               子1 ↔ 子2：永远不行——必须走主中转
```

为什么是树：Agent 工具本质上是「父（发起 query）调用一个子接什么任务」——这是父子关系，不是对等关系。

**Subagent 三重收益**：

- Context 隔离：子任务的中间型污染不了主的 context，主只收到最终摘要
- 错误隔离：子 agent 自己在自己 context 里重试、纠错，主看不到中间过程。一个子挂了不会拖垮主
- 可并行：主能一次派出多个子（需要在 prompt 里明说「请并行调用」）

### 5. 四档放置决策

| 选项（高抽象→低抽象） | 适用场景 | 什么时候选它 |
| --- | --- | --- |
| **Claude Code CLI** | 交互式 / 一次性任务 | 个人本地、周末脚本、无上线需求 |
| **Agent SDK** | 生产自动化 / CI-CD / 你自己的服务器 | 定时任务、要控制运行环境、需要集成进现有技术栈 |
| **Managed Agents** | Anthropic 托管 / 多租户 / 长跨异步 | B 端多客户、任务跨小时、不想管沙箱 |
| **Anthropic Client SDK** | 最底层原始 API | 几乎不选——只有上面三档都不够用时才选 |

**常见错误**：Client SDK 名字里有 SDK 两个字，让人以为它跟 Agent SDK 是同档的东西——它低一档。选它意味着你要亲手写整个 harness loop、亲手处理 tool_use block、亲手做错误重试。不是「更定制」，是「更累且大概率比 Anthropic 写得差」。

## 工程心法

**抽层心法**：从最高抽象层开始问「够不够用」——不够再降一层。不要默认从底层往上堆。在 agent 工程里，底层 ≠ 更自由；底层 = 更多重复劳动。真正的定制在每一层都开放的扩展点里，不在「获取底层 API」里。

## 三个「如果只能记一句」

| 问题 | 答案 |
| --- | --- |
| Agent SDK 本质 | Agent SDK = Claude Code 的内核（loop + 内置工具 + context 管理）拆出来当库用，让你在 95 分起点上做业务定制 |
| 工程价值 | 给你一个 95 分的框架直接用——loop 写好、工具齐备、还过了 Anthropic 内部 eval。你不是省代码，你是省了一个亲手写大概达不到的质量下限 |
| 思维方式改变 | 做 agent 工程决策时，从最高抽象层开始问「够不够用」——不够才往下降。别默认从底层往上堆 |
