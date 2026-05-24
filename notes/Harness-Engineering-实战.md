# Harness Engineering 实战：固定模型只动 harness，52.8→66.5

- 类型: 主题笔记
- 标签: AI / Agent / 文章 / 编程
- 收录: 2026-05-18
- 原文: https://blog.langchain.com/improving-deep-agents-with-harness-engineering/

> LangChain 用同一个 gpt-5.2-codex、只动 Prompt/Tools/Middleware 三个旋钮 + Trace 反馈闭环，把 Terminal Bench 2.0 从 52.8% 推到 66.5%（+13.7 分、Top 30→Top 5）——这是 Anatomy of Harness 的实证现场。

---

> **本质一句话**：Harness Engineering = 固定模型、用 **Trace 做反馈信号** 的 **迭代优化系统**。每轮只动 Prompt / Tools / Middleware 三个旋钮，按 boosting 思路专攻上轮的失败模式。
>
> **最高 ROI 单点改动**：给 agent 加 `PreCompletionChecklistMiddleware`——退出前强制跑一遍 verification。这一个 hook 就能堵掉「模型写完代码自我感觉良好就交卷」的最大失败模式。
>
> **最容易被忽视的陷阱**：算力 ≠ 效果。全程 xhigh reasoning 反而 53.9%（被 timeout 打死），三明治 xhigh-high-xhigh 才 66.5%。贵算力要押在 planning + verification 两端，中间执行用便宜档。

## 为什么需要这篇文章

上篇 *Anatomy of an Agent Harness* 讲了「harness 才是杠杆」的理论。但是空喊「harness 重要」没用——**做这件事到底怎么动手？哪些改动有效？哪些是石膏？哪些是承重墙？** 这篇就是作者自己跑出来的**实证现场**。

传统做法的问题是：很多人改 agent 就上来动 system prompt + 加 tool + 调 model + 改 memory + 试 subagent——**一次动太多变量**，跑完一轮分数变了也不知道是哪个起作用。

这篇文章用 **boosting 视角**重新组织 harness 优化：固定模型、收窄到三旋钮、用 trace 做反馈、每轮专攻上轮的失败模式。结果是 **gpt-5.2-codex 在 Terminal Bench 2.0 从 52.8% 推到 66.5%、+13.7 分**。

## 四层心智模型

### L1 · 实验骨架：三旋钮 + Trace 闭环

> **核心 mindset**：固定模型、收窄优化空间、用 trace 当反馈信号、按 boosting 跑迭代——"This works similarly to boosting which focuses on mistakes from previous runs."

```
┌─────────────────────────┐
│  Run 89 Terminal Bench  │
│  tasks → 写 LangSmith   │
└────────────┬────────────┘
             ▼
┌─────────────────────────┐
│  并行派 N 个分析 agent  │
│  看失败 trace →         │
│  主 agent 综合错误模式  │
└────────────┬────────────┘
             ▼
┌─────────────────────────────┐
│ 改 Prompt / Tools /         │
│ Middleware（minimal change）│
│ 人可介入 Step 3 防过拟合    │
└────────────┬────────────────┘
             ▼
        重新跑 Run
```

**三个旋钮（故意只动这三个）**：

| 旋钮 | 是什么 | 本篇例子 |
| --- | --- | --- |
| System Prompt | 开机指令 | "你的代码会被自动测试打分" |
| Tools | 可调工具集 | bash / 文件读写 / 测试运行器 |
| Middleware（hooks） | 在模型动作前后插一脚的代码 | LocalContext / LoopDetection / TimeBudget / PreCompletionChecklist |

**关键数字**：

| 状态 | 得分 | 说明 |
| --- | --- | --- |
| 默认 prompt + 标准 tools | **52.8%** | 刚出 Top 30 |
| 优化过 harness | **66.5%** | 进 Top 5 |
| 提升 | **+13.7 points** | 模型没动 |

**为什么是 boosting？** 每轮只动一两个旋钮、专攻上轮的失败模式，跟 GBDT 每棵新树拟合上一棵的残差是同一种思路。**少动、专攻、可归因**。

### L2 · 四个 Middleware 解剖

> **核心 mindset**：Middleware = harness 的**反射弧**。模型不会主动做的事（自测、自查、反思、注意时间），就用 hook 在它每一步前后塞一脚，把缺失的纪律变成系统性的强制注入。

```
        Agent 主循环
           │
   ┌───────┼───────┐
   ▼       ▼       ▼
 [启动]  [每步]  [退出]
   │       │       │
   ▼       ▼       ▼
┌──────────────────────────────────────────────┐
│ ① LocalContextMiddleware（启动）             │
│    map cwd + 探测 Python/工具 → 注入 prompt  │
├──────────────────────────────────────────────┤
│ ② LoopDetectionMiddleware（每步）            │
│    同文件 edit N 次 → 注 "换思路" 提示       │
├──────────────────────────────────────────────┤
│ ③ Time Budget Injection（周期）              │
│    定时塞 "剩 X 分钟" 警告                   │
├──────────────────────────────────────────────┤
│ ④ PreCompletionChecklistMiddleware（退出）   │
│    拦截退出 → 强制再跑 verification          │
│    = Ralph Loop 的 verification 版           │
└──────────────────────────────────────────────┘
```

**四个 middleware 对应的失败模式**：

| Middleware | Agent 直觉错误 | 注入的反射 |
| --- | --- | --- |
| LocalContext | "先 ls 看看" 找半天 | 启动时把 cwd + 工具直接塞 prompt |
| LoopDetection | 同一文件改 10+ 次（doom loop） | 注入 "consider reconsidering your approach" |
| TimeBudget | 过度规划耗光时间 | 定期注入 "剩 X 分钟" |
| PreCompletionChecklist | 代码看上去 OK 就交卷 | 退出前强制跑测试 + 对照 spec |

**原文金句（必记三句）**：

> "Models are biased towards their first plausible solution."
>
> "The purpose of the harness engineer: prepare and deliver context so agents can autonomously complete work."
>
> "Forcing models to conform to testing standards is a powerful strategy to avoid slop buildup over time."

> **陷阱：Middleware 不是越多越好**。每加一个都污染 prompt。作者原话："These guardrails will almost surely dissolve over time." Middleware 是「应付当代模型缺陷的临时石膏」——别把石膏修成大教堂、保留拆除接口。

### L3 · Reasoning Sandwich（反直觉数字层）

> **核心 mindset**：**Reasoning compute 不是越多越好**。在有时间预算的场景里，全程高算力会因 timeout 被打死。把贵算力**精准押在 planning + verification 两端**，中间执行用便宜档。

```
任务时间轴 ──────────────────────────────────►

  ┌────────┐    ┌───────────────────┐    ┌────────┐
  │  xhigh │ →  │        high       │ →  │  xhigh │
  │ (规划) │    │      (执行)        │    │ (验证) │
  └────────┘    └───────────────────┘    └────────┘
   想清楚            干活时省着用            提交前
   要做什么          别 timeout              校验一次
```

**反直觉数据**：

| 策略 | 得分 | 结论 |
| --- | --- | --- |
| 全程 xhigh（最贵） | **53.9%** | **被 timeout 打死**，钱花了跑不完 |
| 全程 high | **63.6%** | 中规中矩、时间没溢出 |
| xhigh-high-xhigh 三明治 | **66.5%** | 贵算力精准押 planning + verification |
| Default + 不优化 harness | 52.8% | 基线 |

**类比（考试做题分配）**：

- 笨学生：每道题都精雕细琢，卷子做不完
- 聪明学生：开卷先看一遍全卷（xhigh 规划）→ 中等题先走起来（high 执行）→ 最后留 30 分钟回头检查（xhigh 验证）

> **易错点**：Adaptive Reasoning（自适应推理）没你想得那么万能。同一个 harness 跑 Claude Opus 4.6 只 59.6%，比 Codex（手动三明治 66.5%）低 6.9 分。**手动三明治 > 模型自适应**。

### L4 · 五条 Takeaways + 跨模型迁移 + 未来方向

**五条原话**：

| # | Takeaway |
| --- | --- |
| ① | **Context Engineering on Behalf of Agents** |
| ② | **Help agents self-verify** |
| ③ | **Tracing as a feedback signal** |
| ④ | **Detect and fix bad patterns in short term** |
| ⑤ | **Tailor Harnesses to Models**（换模型重跑整个 Improvement Loop） |

**未来研究方向**：

```
1. Multi-model systems     →  Codex + Gemini + Claude 协作
2. Memory primitives       →  agent 自我持续学习、改 harness
3. RLM（Reward Language    →  专门的「打分模型」从海量 trace 自动挖训练信号
       Models）
```

**Harness 随模型变强会怎么变**：

```
会消失的 harness 组件：
- LoopDetection（模型自己有元认知了）
- Time Budget warning（模型会算时间了）
- PreCompletion checklist（自动 verify）

永远不会消失的 harness 资产：
① 工具与环境接口（model 不会自带你的数据源）
② Filesystem / 持久记忆
③ 权限/安全沙箱（人的需求）
④ 业务知识注入（model 不会自带你的业务逻辑）
⑤ 多 agent 协作框架
⑥ Trace / 可观测性
```

## 三个「如果只能记一句」

| 维度 | 一句话 |
| --- | --- |
| **本质是什么** | Harness Engineering = 固定模型 + 收窄三旋钮 + Trace 反馈 + boosting 迭代。52.8 → 66.5、+13.7 分就是这个方法的 receipt |
| **最高 ROI 实践** | 加 PreCompletionChecklist hook + 写好 LocalContext 注入——这两个直接堵掉「不验证」和「瞎探索」两大失败模式 |
| **思维改变** | **算力分配 > 算力总量**。同理：**精力分配 > 精力总量**——规划和 review 上强模型，执行用便宜档 |
