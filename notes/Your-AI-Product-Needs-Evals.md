# Your AI Product Needs Evals：Eval 是飞轮——同一条质检线白送 fine-tune 数据 + debug + 合成管线

- 类型：方法论笔记
- 标签：AI / Agent / 文章
- 收录：2026-05-18
- 原文：<https://hamel.dev/blog/posts/evals/index.html>

> **本质一句话**：成功的 AI 产品 = 迭代速度，迭代速度 = (评估质量 + debug + 改行为) 三活动循环；多数人只做 #3 所以撞墙。
>
> **最高 ROI 单点改动**：自建 data viewer，**删除一切看数据的摩擦**——但更深层是逼你定义「你的产品什么算好」。
>
> **最容易被忽视的纪律**：LLM-judge 和人类 agreement 高 ≠ judge 有用——类别不均衡时 agreement 是骗子，必须单看 recall。

---

## 为什么需要这篇文章

**传统做法**：觉得做 AI 产品 = 写好 prompt + 拼装好模型，靠「vibe check」看输出。出 bug 就改 prompt，每次改完手动测几条就上线。

**为什么传统做法坏了**：Hamel 的实战观察是——这种打法在 demo 阶段还行，但**一进入持续改进阶段就立刻坏掉**：改 prompt 修一个 bug，另一个冒出来（whack-a-mole）；想知道整体表现如何只能靠感觉（无 visibility）；prompt 越加越长想覆盖每个 edge case（prompt 膨胀）。**这不是技术问题，是缺基础设施**。Hamel 接手 Rechat 公司的房产 AI 助手 Lucy 时，团队就卡在这种 prompt-only 迭代的死循环里。

**Hamel 用什么视角重答**：AI 产品工程的本质是「迭代速度」，迭代速度由三件事决定——**评估质量 / debug / 改行为**。你只做「改行为」等于只蹬自行车的一只踏板。建好 eval 系统 = 蹬上另一只踏板 + **顺手白送你三个基础设施**（fine-tune 数据引擎 + debug 工具 + 数据合成管线）。所以这篇文章的真正主张不是「加点测试」，是**重新定义 AI 产品的工作模式**。

## 3 层心智模型

### Layer 1 · 三活动循环 + Lucy 高原症状

> **核心 mindset**："Many people focus exclusively on #3 [changing behavior], which prevents them from improving their LLM products beyond a demo."（原文）—— **只蹬一只踏板的人永远在 demo 状态打转。**

**三活动各自该长什么样**：

| 三活动 | 只做 #3 的团队 | 三活动都做的团队 |
| --- | --- | --- |
| ① Evaluate quality | 没单测，没系统化打分 | L1+L2 自动跑，每次改动都验证 |
| ② Debug issues | 看聊天记录瞎猜 | trace database + viewer 精确定位 |
| ③ Change behavior | ← 100% 精力都在这里 | 有理有据地改，验证再上线 |

**Lucy 高原三症状**：

| 症状 | 表现 | 根因（缺什么） |
| --- | --- | --- |
| Whack-a-mole | 改完一个 bug 另一个冒出来 | 缺 regression test（已修 bug 没有自动复检） |
| 无 visibility | 只能「感觉好像变好了」 | 缺 trace logging + 系统化打分 |
| Prompt 膨胀 | 每次 edge case 就往 prompt 加一段 | 缺 feature×scenario 拆分（没分解只能堆一处） |

> **易错点**：见过的最大误判——「我项目还小，eval 先不做」。**错。一旦你引入 LLM 调用，下一行代码就该是 trace logging**。否则你每次改 prompt 都是盲修，无法积累。

**真实案例 + 修复**：Hamel 接手 Rechat 的 Lucy 时，团队卡在 prompt-only 迭代，bug 反复出现。Hamel 引入 L1（pytest 风格单测）+ L2（自建 viewer + LLM-judge）后，**新 bug 修复后永久进 regression suite**，三症状全部消失。

### Layer 2 · 三层金字塔 + 自建 viewer + Feature×Scenario + LLM-judge 翻车点

> **核心 mindset**："You must remove all friction from the process of looking at data."（原文）—— **自建 viewer 不是基础设施浪费，是最高 ROI 的杠杆**：你看 100 条 trace 的速度决定迭代速度，**摩擦每多 1 秒，少看 90% 的数据**。

**三层金字塔（频率 + 成本）**：

| 层级 | 动作 | 跑的频率 | 成本 |
| --- | --- | --- | --- |
| L1 Unit Tests | pytest 风格 assertion（代码自动判断对错） | 每次代码改动 | 极低 |
| L2 Human + Model Eval | 看 trace + 人打分 + LLM-judge | 周期性（每周 / sprint） | 中 |
| L3 A/B Testing | 真实用户行为对比 | 重大产品变更后 | 高 |

> 成本 L3 > L2 > L1。先把 L1 做透再上 L2，否则你会用 LLM-judge 测一些本该 5 行代码 assert 的东西，浪费钱又掩盖真问题。

**Feature × Scenario 分解（L1 核心技法）**——拿「策略代码生成 agent」举例：

```text
Feature: write_strategy（写策略代码）
   │
   ├─ Scenario C-1: 描述清晰 "5 日均线上穿 10 日均线买入"
   │   └─ Assertion: exec(code) 不报语法错
   │
   ├─ Scenario C-2: 描述模糊 "做个均线策略"
   │   └─ Assertion: agent 回复包含 "?"（反问澄清，不瞎写）
   │
   └─ Scenario C-3: 物理不可能 "未来 3 天涨 10% 就买"
       └─ Assertion: agent 回复包含 "未来" / "无法"（拒绝 + 解释 lookahead bias）
```

**Assertion 是什么**：一行**自动判断对错**的代码。员工说「会议安排好了」，你去 calendar 看一眼真的安排了没——这个「看一眼」动作就是 assertion。一旦写下来，**它能跑 1 万次不抱怨**，这是单测能换取迭代速度的根本原因。

**核心纪律 3 条**：

- ① 用 LLM 合成测试用例——不用等真实流量，50 条合成够起步
- ② Pass rate 不必 100%——**"your pass rate is a product decision"**（原文），容忍度由产品决定
- ③ 持续从真实失败更新 tests——每发现一个新失败模式就加一条

**LLM-as-judge 经典翻车点（必背）**：

> **原文警告**："Using raw agreement is generally not recommended and can be misleading when classes are imbalanced. Instead, measure precision and recall separately."

混淆矩阵示例（100 条样本，95 好 / 5 坏，「懒鬼 judge 永远说好」）：

```text
                  真实是"好" (95)   真实是"坏" (5)
                ┌─────────────────┬─────────────────┐
judge 说"好" → │   95 ✅          │    5 ❌         │  ← 5 个真坏漏掉
(100)           │   (放对了)      │   (全漏抓)      │
                ├─────────────────┼─────────────────┤
judge 说"坏" → │   0              │    0            │
(0)             │                  │                 │
                └─────────────────┴─────────────────┘

        Agreement = 95%  (看着很美)
        Recall（坏 case 的）= 0/5 = 0%  (灾难)
```

> **陷阱本质**：5% 不均衡场景下，永远说「好」的 judge 都有 95% agreement——但它**让所有坏 case 通过**，等于把门永远开着。
>
> **修复动作**：永远**单独看 recall**（特别是少数类 = 坏 case 的 recall）。或用 F1 score 兼顾 precision/recall。
>
> **一句话肌肉记忆**：「类别不均衡场景下，agreement 是骗子，recall 才是真相。」

> **元洞察**：自建 viewer 的真正价值**不是工具本身**，而是**逼你定义「你的产品什么算好」**。通用 eval 框架的危害在于**让你把「什么是好」的决策权外包了**——而这件事根本不能外包。如果连「我的产品好不好」都需要别人告诉你，那你不是在做产品，是在做「通用 LLM 测试 demo」。

### Layer 3 · Eval 飞轮的三个白送超能力

> **核心 mindset**："The infrastructure required for evaluation is largely the same as that needed for debugging."（原文）—— **你以为在投资测试系统，其实顺手建了一个数据工厂 + 一个故障维修车间 + 一套训练材料库**。这是 "flywheel" 的本意。

**飞轮全景图（同一条质检线，三个出口）**：

```text
                   ┌───────────────────────┐
                   │   你的 Eval 质检线    │
输入 ─────────→    │  (assertion + judge)  │ ──→ 出口
                   └───────────────────────┘
                               │
    ┌──────────────────────────┼──────────────────────────┐
    ↓                          ↓                          ↓
① Fine-tune 数据           ② Debug 基础设施         ③ 数据合成管线
(改坏 case = 训练样本)     (trace DB + viewer)      (LLM 合成 → 过滤)
```

**三个超能力**：

| 超能力 | 怎么白送的 |
| --- | --- |
| ① Fine-tune 数据引擎 | 你在 viewer 改坏 case = 同时产出「输入→正确输出」训练样本。100 条 eval = 100 条潜在训练数据。 |
| ② Debug 基础设施 | 建 eval 时搭的 trace DB / viewer / log，debug 时直接复用一行不改。"eval infra ≈ debug infra" |
| ③ 数据合成管线 | LLM 凭空生成假数据 → 过同一条 assertion + judge 质检线 → 筛出高质量合成数据。**质检线一行不用改**，只换 input 来源 |

**为什么 "同一条质检线"（工厂类比）**：

```text
平时 eval 模式:                  数据合成模式:
[真实生产的椅子]                  [LLM 设计的椅子图纸]
        ↓                                 ↓
  质检员 A: 量尺寸                  质检员 A: 量尺寸
  (assertion 代码)                  (← 一行不用改 ←)
        ↓                                 ↓
  质检员 B: 坐一坐                  质检员 B: 坐一坐
  (LLM-judge)                       (← 一行不用改 ←)
        ↓                                 ↓
  [70 张合格 / 30 张丢弃]           [400 条合格 / 600 条丢弃]
```

**唯一变化**：input 来源（真用户 ↔ LLM 凭空生成）。其他全部复用。这就是「白送」的精确含义。

**Fine-tune vs RAG 分工**：

| 维度 | Fine-tune（微调） | RAG（检索增强生成） |
| --- | --- | --- |
| 大白话 | 改员工性格 / 习惯（一辈子记得） | 给员工查询手册（每次现查） |
| 适合场景 | 风格、语法、固定规则 | 上下文、事实、最新数据 |
| **陷阱** | 把会过期的东西 fine-tune（如实时股价） | 把不变的硬规则只靠 RAG（不够稳） |

> **最大陷阱**：实时数据**永远不该 fine-tune**——烙进模型的就是死的。今天 $180 烙进去，明天涨到 $200 模型还记得 $180。
>
> **口诀**：会过期的、会变的东西 → RAG；永远不变的风格 / 规则 → fine-tune。

**Hamel 七条 takeaway（收尾必背）**：

| # | 原文 | 白话 |
| --- | --- | --- |
| 1 | Remove all friction from looking at data | 看数据要顺手——自建 viewer |
| 2 | Keep it simple. Don't buy fancy LLM tools | 别迷信花哨工具——Shiny / Streamlit 小程序够用 |
| 3 | You are doing it wrong if you aren't looking at lots of data | 不看数据 = 错，没有讨论空间 |
| 4 | Don't rely on generic evaluation frameworks | 不要套通用 eval 框架，自己定义「好」 |
| 5 | Write lots of tests and frequently update them | 多写测试 + 持续更新 |
| 6 | LLMs can be used to unlock evals | LLM 是测试工具：合成 / judge / critique |
| 7 | You can never stop looking at data—no free lunch exists | 看数据永无止境，没有一劳永逸 |

> **最易忽视的是第 4 条**：很多人想找「测 AI 用什么开源框架最好」——这个问题本身就错了。每个产品的「好」定义不同，所以 eval 必须 domain-specific 自建。

## 三个「如果只能记一句」

| 维度 | 一句话 |
| --- | --- |
| **本质是什么** | 建一次 eval 系统 = 同一条质检线，三个出口（fine-tune 数据 / debug / 合成管线）白送 |
| **最高 ROI 实践** | 自建 viewer——但表面是工具，**实质是逼自己定义「产品什么算好」** |
| **思维方式改变** | 从「AI 产品 = 写好 prompt」 → 「AI 产品 = 建好 eval 基础设施」。eval 不是质检员，是飞轮 |
