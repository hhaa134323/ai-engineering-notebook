# Agent Eval Readiness Checklist：60-80% 精力先做 error analysis，再用 binary 拆维度、把 eval 跑成飞轮

- 类型：方法论笔记
- 标签：AI / Agent / 文章
- 收录：2026-05-18
- 原文：<https://langchain-blog.ghost.io/agent-evaluation-readiness-checklist/>

> **本质一句话**：Agent eval 的关键不是写更多 grader，而是**先做 error analysis（60-80% 精力）→ 把每个维度拆成 binary → 跑成 offline/online/ad-hoc 飞轮**。
>
> **最高 ROI 单点改动**：把「1 个 LLM-judge 打总分」改成「5-10 个 specialized binary grader」——失败时直接知道哪一维挂了，且每一项可重复、可审计、不会吵架。
>
> **最容易被忽视的纪律/陷阱**：State change 维度——agent 说「完成了」不等于真完成。会议没建、文件没写、数据库没改是常态。**做事的 agent 必须测最终真实状态，不能只测它说了什么。**

---

## 为什么需要这篇文章

传统软件测试有确定输入输出，写 unit test 就够。Agent 不同——同样 prompt 跑两次结果不一样，failure 模式像万花筒：可能是 prompt 不清、tool 设计烂、模型边界、数据 bug 或基础设施挂了，**根因不同，修法完全不同**。新手最常见的错是「拿 helpfulness/coherence 这种泛 metric 跑分」，结果分数 80 → 85 你也不知道是真改进还是噪音。

这篇是 LangChain 出的 **how-to checklist**，6 个 section、≈30 个 checkbox。它不是工具教程，是**思维框架**——告诉你做 eval 之前应该想清楚的事，避免你一上来就跳到 LLM-as-judge 然后自欺欺人。

## 4 层心智模型

### L1 · 心态前置：先 error analysis，后写 eval 代码

> **核心 mindset（原文）**："**60-80% of your eval effort should be in error analysis**, not building automated graders." 写 eval 代码之前先**手动读 20-50 个真实 trace**——你从这里学到的失败模式比任何自动化系统都多。

**反例 vs 正例对比表**：

| 维度 | 新手做法 | 文章推荐 |
| --- | --- | --- |
| 第一步 | 选个 metric 就开跑 | 手动读 **20-50 个真实 trace** |
| 失败定义 | "Summarize this document well" | "Extract 3 action items, each <20 words, include owner if mentioned" |
| 归因 | "agent 不行 → 换模型" | 分类到 5 类 taxonomy（见下） |
| 归责陷阱 | 基础设施 bug 当成推理失败 | Witan Labs 案例：**修一个 extraction bug，benchmark 从 50% → 73%** |

**Failure taxonomy（5 类，背下来）**：

1. **Prompt 问题**——指令不清
2. **Tool 设计问题**——接口让 agent 容易犯错
3. **Model 能力问题**——指令清但模型搞不定 edge case
4. **Tool 调用失败**——timeout / API 异常
5. **数据/基础设施 bug**——最容易被冤枝成「agent 不行」

> **最容易踩的坑**：把 capability eval 和 regression eval 混在一起。
>
> - **Capability eval** = "它能做这件事吗？" → 期望**低通过率**（如 30%），给你爬坡空间
> - **Regression eval** = "它还能做这件事吗？" → 期望 **≈95% 通过率**，防止退化
>
> **95% pass 的 capability eval 是失败设计**——说明任务比 agent 还简单，它在测一个已经解决的问题。

**真实修复动作（4 步法）**：

1. **Gather**——挑 50 个「输出有问题」的 case
2. **Open coding**——每个 case 写 1-2 行原始观察，**不预设分类**
3. **Categorize**——50 个看完事后归类（用上面 5 类 taxonomy）
4. **Iterate**——看到不再出现新类别为止

> **原文金句**："If two experts can't agree on pass/fail, the task needs refinement."

### L2 · 评估层级 + 数据集构造：测什么，用什么测

> **核心 mindset（原文）**："**Start at trace-level (full-turn). Most teams should start here.**" 不要一上来就做 single-step（脆弱）或 multi-turn（复杂）。

**三层评估粒度**：

```text
                 一次用户请求
                       │
     ┌─────────────────┼─────────────────┐
     ▼                                   ▼
单次 LLM call                       多轮对话
(Run / Step)                        (Thread)
     │                                   │
     ▼                                   │
┌──────────────────────────┐             │
│   一个完整 turn (Trace)   │ ◄─── 起点  │
│   ─────────────────────  │             │
│   user msg → tool calls  │             │
│   → final response       │             │
└──────────────────────────┘             │
     ▲                                   │
     └────────── N-1 测试技巧 ───────────┘
     (拿生产前 N-1 轮，让 agent 只生成最后一轮)
```

**三层 eval 对比**：

| 层级 | 测什么 | 何时用 | 易错点 |
| --- | --- | --- | --- |
| **Single-step (Run)** | "选对工具了吗？API 参数对吗？" | 调 prompt/tool 后想 debug 具体一步 | 架构没稳前别做，**改一次 tool 定义就全废** |
| **Full-turn (Trace)** | 3 维：final response + trajectory + state change | **大部分团队的起点** | 大家只测前两维，**忘了 state change** |
| **Multi-turn (Thread)** | "上下文保持住了吗？" | 真有多轮才做 | 全合成 conversation 会 compounding error，用 **N-1 trick** |

> **最容易被忽视的维度：State change**——原文金句："**The final response can say 'Done!' while the actual state is wrong.**"
>
> - Agent 说「会议已安排」 → 但日历事件根本不存在 / 时间错了 / 参会人漏了
> - Agent 说「代码写好了」 → 但代码跑不起来
> - Agent 说「数据库更新了」 → 但行数不对
>
> **判断 state change 的诀窍**：问「如果 agent 进程崩了，下次重启之后哪些**持久化的痕迹**还在？」——这些就是 state（文件、DB 行、git commit、外部通知、配置文件）。

**数据集构造的 6 条铁律**：

| 规则 | 一句话理解 | 数字/原话 |
| --- | --- | --- |
| 每个 task 必须 unambiguous + 有 reference solution | "如果 agent 不可能成功，是 task 坏了" | — |
| 正例 + 反例都要有 | 只测「该 search 时是否 search 了」 → agent 学会 search 一切 | "examples designed to **falsify** your assumptions" |
| 结构匹配 eval 层级 | Run-eval 要 reference tool call；Trace-eval 要 expected output + state | — |
| 量 < 质 | 20-50 手工 review > 几百合成 | "**20-50 hand-reviewed examples** outperform hundreds of synthetic" |
| 三种 sourcing 混用 | ① Dogfood ② 改造外部 benchmark（cherry-pick）③ 手写 behavior test | Terminal Bench / BFCL |
| 建 trace → dataset 飞轮 | 生产出错 trace → annotation → dataset → 下次防住 | LangSmith 原生 |

**Dataset 来源优先级**：B 真实历史请求 > A 手写 50 > D 外部 benchmark 改造 > C GPT-4 合成 500。**真实分布永远是金本位**，合成数据是冷启动备胎，不是 quality 来源。

### L3 · Grader 设计：用什么打分

> **核心 mindset（原文）**："**Decompose evaluation into specialized graders per dimension rather than one monolithic grader.**" 拆 5-10 个 specialized grader，每个只判一件事，每个返回 binary（pass/fail）。

**四种 grader 类型**：

| Grader 类型 | 最适合 | 速度/成本 | 注意事项 |
| --- | --- | --- | --- |
| **Code-based** | 有客观对错的（tool 调用、格式、metric 值、文件存在性） | 毫秒级 / 免费 | 容易「假阳性 fail」，格式有差异但语义对的会被判错 |
| **LLM-as-judge** | 主观质量、rubric 评分、开放式输出 | 秒/分钟级 / 贵 | **必须用 20-100 人工标注 calibrate** |
| **Human** | 校准、edge case、subjective criteria | 最慢最贵 | 用于「教会」LLM-as-judge |
| **Pairwise** | 版本对比（v1 vs v2 哪个更好） | 中等 | 不告诉你「对不对」，告诉你「哪个更好」 |

> **Default to code-based.** 客观的事用 LLM-judge 判 = 自找麻烦：贵、慢、判分不一致还可能掩盖真实回归。"Reserve LLM-as-judge for genuinely subjective assessments."

**Guardrail vs Evaluator（这是两个独立的轴，不要混）**：

```text
                  【实现方式轴】
                  code-based  |  LLM-as-judge  |  human
                  ────────────┼────────────────┼─────────
【何时跑的轴】
    inline       │   ✅       │  少见(慢)      │  ❌
   (guardrail)   │ 检查 eval  │  内容审核      │
   ─────────────┼─────────────┼────────────────┼─────────
    async        │  sharpe>1  │  报告质量打分  │  校准
   (evaluator)   │  字段齐全  │  judge 评分    │  edge case
```

|  | **Guardrails** | **Evaluators** |
| --- | --- | --- |
| 何时跑 | **inline**——用户看到输出**之前** | **async**——生成**之后**异步跑 |
| 速度 | 必须毫秒级 | 秒/分钟级可接受 |
| 目的 | **拦截危险/恶意输出** | **测量质量、检测回归** |
| 例 | PII 检测、格式校验、安全过滤 | LLM-judge 评分、轨迹分析 |
| 失败后果 | 阻止输出 | 记录到 dashboard，不阻止 |

**Binary > 1-5 scale**：1-5 scale 两大问题——①相邻分数没有客观差异（reviewer 必吵架）②统计显著性差（要 10 倍样本量）。

**正确做法**：拆多个 binary。例如「报告好不好（1-5）」拆成：① 包含 sharpe/drawdown/年化 三个数字？② 数字和原始指标一致？③ 说明了风险？④ 有 actionable next step？→ 4 个 binary = 0-4 评分，每项可重复、可审计。

**Witan Labs 5-grader 拆分法**：

```text
原方案：              拆分后：
┌─────────────┐       ┌──────────────────┐
│ 1 个总评 LLM │       │ ① 内容准确性     │ code-based
│ 给 0-10 分   │  →    │ ② 结构正确性     │ code-based
│             │       │ ③ 视觉格式       │ LLM-judge
└─────────────┘       │ ④ 公式场景       │ code-based
说不清哪里挂了        │ ⑤ 文本质量       │ LLM-judge
                      └──────────────────┘
                      → 失败时直接告诉你哪一维挂了
```

> **原文金句**："**Don't grade the path the agent took, grade what it produced.**" —— Anthropic。**但 efficiency（token/步数）和 security（API key 泄漏）例外**，必须看 path。

### L4 · 运行迭代 + 生产化：从写完 eval 到守住生产

> **核心 mindset（原文）**："**The teams that ship reliable agents aren't the ones with the most sophisticated eval infrastructure — they're the ones who started evaluating early and never stopped iterating.**" Eval 不是项目，是飞轮。

**三种评估时机**：

```text
         ┌────────────────────────────────────────────────┐
         │              Eval 的三个战场                    │
         ├────────────────────────────────────────────────┤
开发期   │  OFFLINE     ← 你 90% 的精力应该在这里         │
         │  curated dataset + 控制变量实验                │
上线前   │  CI gate     ← 改 prompt/code 自动跑回归       │
         │  code-based 快 grader 把关每次 PR              │
生产中   │  ONLINE      ← 持续监控真实流量                │
         │  在生产 trace 上跑 LLM-judge / heuristic       │
每周     │  AD-HOC      ← 人工探索性 review               │
         │  发现自动 eval 抓不到的新 pattern              │
         └────────────────────────────────────────────────┘
```

| 时机 | 本质 | 何时跑 | 用什么 grader |
| --- | --- | --- | --- |
| **Offline** | "已知的怪兽，自动复盘" | 改代码/prompt 后部署前 | code-based 为主 + 少量 LLM-judge |
| **Online** | "实时的怪兽，自动监控" | 生产里每条真实请求 | 轻量 LLM-judge + heuristic（不能太贵） |
| **Ad-hoc** | "未知的怪兽，人肉巡山" | 每周 1-2 次人工 | 你自己的眼睛 + trace |

> **最容易丢的环节是 ad-hoc**。自动 eval **只能抓你已经知道的失败模式**——新的、没见过的失败必须靠人工探索发现。**每周给 ad-hoc 留 1 小时**。

**非确定性应对**：

- **pass@k**——k 次尝试**至少 1 次**成功（适合可重试场景，如手动触发）
- **pass^k**——k 次尝试**全部**成功（适合不可重试，如定时自动化流水线）

**质量 + 效率双指标**：原文金句 "**An agent that's 95% accurate but 10x slower might not be an improvement.**" 必须追踪 observed steps / ideal steps、observed tool calls / ideal、observed latency / ideal、token cost。

**Capability → Regression 升级机制**：

```text
新写的 eval (pass rate 30%)   →   爬升优化   →   pass 95%+   →   毕业进 regression suite (要求 100%)
```

**生产化 7 件套**：

```text
[1] PR 触发 CI
     ▼
[2] Offline eval (code-based, 快, 便宜)
     ▼  fail → block PR + 进 annotation queue
     ▼  pass
[3] Preview deploy
     ▼
[4] Online eval (LLM-judge, preview 上跑真实 traffic 子集)
     ▼  fail → 回滚 + alert
     ▼  pass
[5] 上线
     ▼
[6] 生产持续 online eval + user feedback
     ▼
[7] 失败 trace → 回填 dataset → 下次 PR 必被它防住
     └────────── flywheel ↺ ──────────
```

> **最高 ROI 建议（原文）**："**Invest in tool interface design and testing, not just prompt optimization.**" Anthropic 做 SWE-bench agent 时**花在 tool 设计上的时间比 prompt 上多**——好的 tool 接口让一整类错误**结构上不可能发生**，那这类错误就不需要写 eval 守它。
>
> 例：要求 tool 参数必须传**绝对路径**（不是相对路径）→ 一整类「agent 找不到文件」的 bug 直接消失。

## 三个「如果只能记一句」

| 问题 | 一句话答案 |
| --- | --- |
| **本质是什么** | Agent eval 的核心是 error analysis（60-80% 精力）+ specialized binary grader 拆分 + offline/online/ad-hoc 飞轮——不是堆 metric。 |
| **最高 ROI 实践** | 把 1 个 monolithic LLM-judge 拆成 5-10 个 specialized binary grader（Witan Labs 法），失败时立刻知道哪一维挂。 |
| **思维方式改变** | 从「写更多 eval 拦截错误」 → 「重新设计 tool/prompt 让错误结构上不可能发生」——这是 eval 工作量↓的根本路径。 |
