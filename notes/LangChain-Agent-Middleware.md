# LangChain Agent Middleware：把怕 LLM 犯浑的事抢回来交给代码

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://blog.langchain.com/how-middleware-lets-you-customize-your-agent-harness/

> Agent middleware = 6 个 hook 组成的可插拔机制，让确定性代码包住概率性的 LLM-工具循环；心智模型同 web 框架 middleware / Spring AOP / 装饰器。

---

> **Middleware 的使命：把怕 LLM 犯浑的事情抢过来用代码做。**

## 一句话定义

Agent middleware = **6 个 hook 组成的可插拔机制**，让确定性代码包住概率性的 LLM-工具循环。

心智模型类比（挑一个你熟的就行）：

- Web 框架的 middleware（Express `app.use((req, res, next) => {...})`）
- Spring AOP（`@Before` / `@After` / `@Around`）
- Python 装饰器
- 只不过被拦截的「请求处理」换成了「agent 执行循环」

## Why：为什么 prompt + tool 还不够

- LLM 是**概率模型**，99% × 99% × ... 在长循环里会**复利衰减**
- prompt 表达力不足、模型可能漏 / 忘 / 绕弯
- 原文金句：**"You can't prompt your way to HIPAA compliance."**
- 翻译：合规要的是 100%，模型给的是 99%，差距必须靠**确定性代码**补上

> **结论：把硬规则从 LLM 嘴里抢回来，交给 middleware。**

## What：6 个 Hook 的位置

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
agent 启动
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
         │
         ▼
   ① before_agent     ← 一次性：连 DB、装资源
         │
         ▼
╔════════════════════════════════════════════╗
║  循环开始（每个 turn 跑下面这些）          ║
╠════════════════════════════════════════════╣
║    ② before_model    ← 喊 LLM 前            ║
║         │                                  ║
║    ③ wrap_model_call ← 包住 LLM 调用        ║
║         │   (LLM API 在内部跑)              ║
║    ④ after_model     ← LLM 答完、工具还没跑 ║
║         │                                  ║
║    ⑤ wrap_tool_call  ← 包住工具调用         ║
║         │   (工具在内部跑)                  ║
║         └─── 循环回 ② 直到 LLM 说 done      ║
╚════════════════════════════════════════════╝
         │
         ▼
   ⑥ after_agent      ← 一次性：存结果、关连接
```

### 按频率分两类

| 类型 | 什么时候跑 | 典型用途 |
| --- | --- | --- |
| **生命周期型**（一次性）<br>before_agent / after_agent | 任务开始/结束各一次 | 连数据库、加载资源 / 存日志、关连接 |
| **循环内型**（每 turn）<br>before_model / wrap_model_call / after_model / wrap_tool_call | 每个 LLM turn 跑一次 | 压缩上下文、查缓存、风控拦截 |

### `wrap_*` vs `before_/after_` 的关键差别

|  | before_/after_ | wrap_* |
| --- | --- | --- |
| **它在哪** | X 之外（前面/后面） | **包住 X**（X 在它内部跑） |
| **能不能跳过 X** | 不能 | 能，可控制跑几次/换参数/直接替换 |
| **典型用途** | 观察、记录、预处理 | **重试、缓存、降级、动态换模型** |
| **AOP 类比** | `@Before` / `@After` | `@Around` |

> **判断 hook 的口诀：问触发时机**
>
> - 任务开始/结束一次 → `before_/after_agent`
> - 每次 LLM 调用前/后 → `before_/after_model`
> - 包住某次 LLM 调用 → `wrap_model_call`
> - 包住某次工具调用 → `wrap_tool_call`

## How：5 类 Pattern × Hook 组合

| Pattern | 典型场景 | 关键 hook |
| --- | --- | --- |
| **业务合规** | PII 脱敏、内容审核、风控止损 | `before_model` / `after_model`（输入/输出双向）<br>或 `wrap_tool_call`（工具级拦截） |
| **动态控制** | 动态注入工具、中途换模型、改 system prompt | `wrap_model_call` |
| **上下文管理** | 压缩、offload、双轨保存 | `before_model`（看 token 数） / `wrap_tool_call`（offload 大结果） |
| **生产稳定性** | 重试、降级、人工确认 | `wrap_model_call`（重试 LLM） / `wrap_tool_call`（重试工具） / `after_model`（人工确认） |
| **资源管理** | 连数据库、启 sandbox、挂载 shell | `before_agent` / `after_agent` |

## 经典反模式：绝不要写「让 LLM 自己评估」的 middleware

> 任何形如**「让 LLM 每次评估 X / 检查 Y / 判断 Z」**的 middleware，都是错的——
> 这本质上还是 prompt engineering，**绕了一圈又回到 99% 概率性**。
>
> Middleware 的全部价值 = **用确定性代码替 LLM 做硬规则**。

反例 vs 正例（以风控止损为例）：

| 错误做法 | 正确做法 |
| --- | --- |
| `wrap_model_call`：让 LLM 每次回答前自己评估回撤风险 | `wrap_tool_call`：LLM 想调下单工具时，middleware 用 Python 代码算回撤，>20% 直接拒绝执行工具 |
| 风险计算依赖 LLM 输出（概率性） | 风险计算是确定性 Python（100%） |

**经验法则**：能在 `wrap_tool_call` 拦的，优先在那里拦——拦的是**确定性的工具调用**，不是**概率性的 LLM 意图**。

## Meta：Middleware 的未来分化

原文做了一个大胆的预判：模型变强后，**部分 middleware 会被吸收进模型本身，但另一部分永远会留**。

```
┌───────────────────────────────────────────────────┐
│  未来会被吸收进模型（原文点名）：                 │
│    • 上下文管理（summarization、output trimming） │
│    • 工具筛选（tool selection）                   │
└───────────────────────────────────────────────────┘

┌────────────────────────────────────────────────┐
│  永远留在 middleware：                          │
│    业务合规（你公司/国家的规矩）                │
│    生产稳定性（你的 SLA、外部 API 抽风）        │
│    资源管理（你的 DB、凭证、基础设施）          │
│    业务逻辑（你的客户、报告格式）               │
└────────────────────────────────────────────────┘
```

> **判断锚点**：**「模型变强 = 通用智能变强；middleware = 你和你公司的独特性。」**
> 两件事原则上**没有重叠面**——模型再强也学不会你公司的规矩、你的钱包、你的基础设施、你的客户。

## 终极 Meta：Middleware 是组织问题，不是技术问题

原文金句：

> Middleware lets different teams own different concerns, keeps business logic decoupled from core agent code, and makes it easy to reuse logic across an org.

这是**康威定律（Conway's Law）的体现**——软件结构反映组织沟通结构。

反面教材场景（没有 middleware，4 个团队挤一个 core loop 文件）：

- 算法团队改 prompt（第 47 行）
- 风控团队加风控（第 73 行）
- 运维团队加重试（第 89 行）
- 合规团队加 PII 脱敏（第 45 行 + 第 81 行）
- Git 冲突 4 处；算法不小心删了合规代码 → **上线泄露隐私**
- 想测试 prompt 改动？必须把整个 agent 跑起来（包括风控、重试、脱敏）
- 想回滚？改动纠缠在同一函数，**回不掉**

有 middleware 之后：每团队拥有自己的 `*Middleware.py` 文件，0 冲突合并、独立单测、独立发布节奏。

> **这一层最高 mindset：好抽象的标准 = 「它的边界刚好让不同的人能独立工作不互相打架」。**
> LangChain 切 6 个 hook 不是技术上只能切 6 刀——是这 6 刀**正好对应工程实践中最常见的关注点分工**。

## 4 句肌肉记忆（带走）

1. **「把怕 LLM 犯浑的事情抢过来用代码做。」**
2. **「判断 hook = 问触发时机。」**
3. **「模型变强干掉通用 middleware；干不掉关于你独特性的 middleware。」**
4. **「好抽象的标准 = 边界对得上人的分工。」**
