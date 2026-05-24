# Better Harness：Eval = Harness 工程的训练数据，飞轮自动迭代

- 类型：主题笔记
- 标签：AI / Agent / 文章 / 编程
- 收录：2026-05-18
- 原文：<https://blog.langchain.com/better-harness-a-recipe-for-harness-hill-climbing-with-evals/>

> **本质一句话**：Better-Harness 把 agent harness 工程从「手作」升级到「工业体系」——用 eval 作为训练数据信号，配合 trace 诊断 + 单点改动 + holdout 防过拟合 + 用户纠错自动产 eval 的飞轮，让 harness 自己持续 hill-climbing。
>
> **最高 ROI 单点改动**：从今天起把每次用户纠错的 trace 都收集起来——这是飞轮的入口，**生产环境的纠错 trace 是最高质量的 eval 来源**。
>
> **最容易被忽视的纪律**：Agent 的 eval 测的是**过程**（agent 行为对不对），不是**产物**（产品指标）。把这两个搞混了 = 把整套体系做歪。

---

## 为什么需要这篇文章

读到这里之前你会想：「harness 工程嘛，就是调调 prompt、改改 tool description——手工活、靠经验。」这条路有天花板：**人工调优不可扩展、不可复利、新模型一来从零开始**。

问题在哪：靠人改 prompt 没办法把「哪个改动起了作用」沉淀下来。Trace 看完一遍就过去了，eval 跑过就丢，flywheel 完全没启动。这是 LLM 应用从 demo 到工业化最大的断层。

这篇用 ML 训练数据的视角重新回答：**Eval 是 harness 工程的 training data**——每条 eval 给「做对/做错」的 gradient 信号，驱动 harness 下一步该怎么改。配上 holdout、human review、自动从 trace 产 eval，整个 harness 升级过程**变成可工业化的复合系统**。

## 四层心智模型

### L1：Eval = Training Data for Agents

> "Evals encode the behavior we want our agent to exhibit in production. They're the **training data** for harness engineering."

**ML vs Harness Engineering 平行对照**：

| ML 概念 | Harness 工程对应 |
| --- | --- |
| Training data | Eval cases |
| Gradient（更新信号） | 「agent 做对了 / 没做对」的 eval 结果 |
| Overfitting | Harness 被调成只过 eval set，泛化差 |
| Train/test split | Optimization set / Holdout set |
| Data quality curation | Eval case 手工筛 + tagging |
| Model weights update | Harness（prompt / tools / middleware）更新 |

**Eval ≠ Trace（互补关系）**：

- Eval = 考试评分员给的**分数**（pass/fail，告诉你哪些 case 挂了）
- Trace = 你做卷子时的**草稿纸**（告诉你为什么挂了）
- 只有分数没草稿 → 不知怎改；只有草稿没分数 → 不知改哪个值得改

> **L1 最大易错点**：Agent 的 eval 测**过程**（行为正确性），不是**产物**（产品指标）。
> - 错的 eval：「产品指标必须达到 X」（这是产品验收）
> - 对的 eval：「agent 在代码里没用未来函数」/「agent 做了 Walk-Forward 切片」/「agent 在模糊需求时主动反问」
>
> **把这两个混了，做出来的「eval 体系」完全不能用。**

### L2：Eval 从哪来 + 怎么防作弊

> "Quality > quantity, a small set of well-tagged evals covering the behaviors you care about beats thousands of noisy but high-coverage evals."

**四种 Eval 来源**（按 ROI 排序）：

| 来源 | 特性 | ROI |
| --- | --- | --- |
| 1. Hand-curated | 高质量、量产难、适合冷启动 | 高（单条） |
| 2. Production traces | 「失败 trace → eval case」，自动飞轮 | **最高（总量）** |
| 3. External datasets | 需要手工 curated，不能直接用 | 中 |
| 4. Tag everything | 给每条打 behavioral 标签，跑子集省钱 | 杠杆器 |

**Agents are famous cheaters（著名作弊者）**：

> "Reward hacking — the agent overfits its structure to make the existing evals pass that it can see."

表面：Eval 全过；实际：holdout 全挂。

**两道防线**：

- **防线 ① Holdout set**（机器自动）：optimization set 上爬山、holdout 上验证泛化。如果反过来用 = 直接舞弊。
- **防线 ② Human review**：砍掉 overfit 的 prompt 补丁（「may not hurt 但 waste of tokens」）。

> **金融时序里 Holdout 的特殊坑**（不能直接用 ML 通用 random split）：
> 1. **Look-ahead bias**：random split 让「未来」数据进训练集 → 必须按时间切
> 2. **非平稳（regime shift）**：牛市训的策略熊市挂 → holdout 跨多个 regime
> 3. **少样本**：10 年 ≈ 2520 个日 K → 比 ML 数据集稀疏 N 倍
> 4. **Survivorship bias**：数据集只含活下来的股票 → 用 point-in-time 数据库
>
> **正确做法：Walk-Forward**（expanding / rolling window），不是 random split。

**Behavioral tag**（纵坐标） ≠ **流程阶段**（横坐标）：

- 错误 tag（流程阶段）：development / running / evaluating
- 正确 tag（行为类型）：look_ahead_bias_avoidance / hypothesis_verbalization / tool_chain_correctness / edge_case_handling / regime_robustness / confidence_calibration / explanation_quality

类比：流程阶段 = 「考试做到第几题」；behavioral tag = 「代数能力 / 几何能力」——后者**跨越所有题目**。

### L3：Optimization Loop + 四条跨模型通用 fix

> **Loop 内核**：Diagnose（从 trace 找失败模式）→ Experiment（一次只动一组改动）→ Validate（holdout + 人工 review）。

**四条跨模型通用 prompt-fix**（Claude Sonnet 4.6 + GLM-5 都生效）：

| 失败模式 | 加的 instruction（原文） | 修复效果 |
| --- | --- | --- |
| 卡在琐碎缺信息上不行动 | "Use reasonable defaults when the request clearly implies them." | stopped blocking on trivial missing wording |
| 反复问用户已给的信息 | "Do not ask for details the user already supplied." | followup 不再问冗余 schedule |
| 同义搜索死循环 | "Do not keep issuing near-duplicate searches once you have enough information." | 不再死循环 |
| 问错抽象层级 | "Ask domain-defining questions first, before implementation questions." | followup_vague 类 eval 通过率上升 |

> **最反直觉的发现**：这四条在 Sonnet 和 GLM-5 上**同时生效** — 说明这不是模型 quirks，是 LLM-as-agent 的**结构性问题**。可以直接拄进 baseline prompt，不用自己重新发现。

**两层 Loop 区分**（结合 Ralph Loop）：

| 维度 | Ralph Loop（任务内、秒级） | Optimization Loop（任务间、天/周级） |
| --- | --- | --- |
| 谁在跑 | runtime agent | meta-agent / 工程师 |
| 输入信号 | checklist 未完成项 | trace + eval 聚合分数 |
| 输出动作 | 继续干活 / 跑测试 | 改 prompt / 改 tool 描述 |
| 作用对象 | 当前这次任务 | harness 本身 |

类比：**Ralph loop = 考试时检查答案**；**Optimization loop = 考完试老师改教学大纲**。

### L4：Regression + Spring Cleaning + 终极飞轮

> "Once our agent handles a case correctly, we don't want to lose that gain. **The eval becomes a regression test**."

**Eval 的双重身份**：

- **Hill-climbing 视角**：信号驱动 harness 改动（向前推）
- **Regression test 视角**：保护已经做对的事不退化（向后守）

→ 选出一个 **smoke test eval set**（always green line），突然挂了立刻 rollback。

**Spring Cleaning（春季大扫除）**：

> "We don't think our eval suite should grow monotonically. Spring cleaning of evals is good!"

砍 eval 的三个时机：

1. 这条 eval 99%+ 通过几个月 → 删（无 gradient 信号）
2. 团队对 agent 期望行为变了 → 旧 eval 标准过时 → 删
3. 多条 eval 在测同一行为 → 留 1-2 条代表性的

心态：**eval set 像花园不像粮仓，修剪比堆积重要**。

**终极飞轮**：

```text
用户使用 agent
     ↓
所有 run 打到 trace 平台（LangSmith）
     ↓
     ├─ ① Human review：砍 overfit prompt、抓 waste-token 补丁
     ├─ ② Auto-cluster：constantly monitor & classify failure modes
     ├─ ③ Auto-generate evals from traces（用户纠错最值钱）
     └─ ④ Compare harness versions：side-by-side trace diff
     ↓
更多 eval → 更好 harness → 用户更想用 → 飞轮加速
```

> **最值钱的一句**："A trace where the agent made a mistake is an eval case. A trace where a user corrected the agent is even better."
>
> → **用户纠错 = 最高质量 eval**（自带 ground truth）。

**用户纠错信号强度金字塔**（设计产品时按这个抓信号）：

```text
信号强度 ↑
5. 用户改了 agent 代码（diff = ground truth）  ← 最强
4. 用户补充具体约束（"加上止损"）
3. 用户重新提问 + 加约束
2. 用户删除/重写整段输出
1. 用户点差评（只知不好，不知哪里不好）         ← 最弱
```

> **L4 易错点：把 eval 当 KPI**
> - 错："eval 通过率从 60% 涨到 90%！" → 对："**holdout** 通过率涨了才算"
> - 错："多写 eval = 更好" → 对："**能产生 gradient 信号**的 eval 更好"
> - 错："eval 一直留着" → 对："spring cleaning：失效的 eval 该删就删"

**Fitting models to harnesses（最后一个洞察）**：

> "There's a large amount of work that goes into fitting every model to its harness."

- 传统直觉：harness 是 model-agnostic
- 文章揭示：每个模型有自己的「风格偏好」（codex 推荐特定 Edit tool 格式）
- 真正成熟的做法是 **harness per model**

## 三个「如果只能记一句」

| 维度 | 一句话 |
| --- | --- |
| 本质是什么 | Eval 是 agent harness 工程的训练数据；trace 是草稿纸；两者驱动 harness 自动 hill-climbing |
| 最高 ROI 实践 | 从今天起把所有用户纠错的 trace 收集起来 — 飞轮入口、自带 ground truth 的最高质量 eval 来源 |
| 思维方式改变 | Harness 不再是手工调的「工艺品」，而是有 eval / holdout / regression test / spring cleaning 的「工业体系」，且 **eval 测的是过程不是产物** |
