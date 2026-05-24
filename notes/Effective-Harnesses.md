# Effective Harnesses: 跨会话「换班工程师」范式

- 类型：主题笔记
- 标签：AI / Agent / 文章 / 编程
- 收录：2026-05-18
- 原文：<https://anthropic.com/engineering/effective-harnesses-for-long-running-agents>

> **本质一句话**：长任务 agent ≠ 一个能跑好几天的 agent，而是**一队每次换班都失忆的工程师**——harness 的工作不是让单个 agent 更强，而是设计交接物（artifacts），让下一班 5 分钟内接上。
>
> **最高 ROI 单点改动**：在项目 repo 加 `.claude/INITIALIZER.md` + `.claude/CODING.md` 双 prompt 文件，强制双角色开场仪式。
>
> **最容易被忽视的纪律**：feature_list 必须用 JSON 而非 Markdown——**格式刚性 = 行为约束**，这是 agentic engineering 的第一性原理之一。

---

## 为什么需要这篇文章

**传统做法**：相信 LLM 的 `compaction`（上下文压缩）+ 把 prompt 写好 → 让 agent 跑一个 loop 就能干长任务。Anthropic 实验证明：即使 Opus 4.5 + Claude Agent SDK + 高质量 prompt，跑复杂 web app clone 都 fall short。

问题根因：(1) agent 试图**一口气干完（one-shotting）**，半路 context 用完留下半成品；(2) **误判完工（premature victory）**，后来的 agent 看到「代码挺多」就宣布完工。失忆只是诱因，真正的根因是 **agent 没有外部权威清单告诉它「完整」长什么样**——它在用相对感觉判断完工。

Anthropic 用一套**双 prompt + 三件套交接物**的范式重新回答：把「跨会话连续性」从 LLM 内存问题转化为**文件系统问题**——人类工程师团队用 issue tracker + 工作日志 + 版本控制做交接，agent 也用同样的东西做交接。

## 4 层心智模型

### L1：核心问题——「换班工程师」心智

> **核心 mindset（原文原话）**："agents must work in discrete sessions, and each new session begins with no memory of what came before."
>
> 长任务 agent ≠ 一个能跑好几天的 agent，而是**一队每次换班都失忆的工程师**。

**对照表：单上下文 vs 多上下文**

| 维度 | 单上下文（前 4 篇关心） | 多上下文（本篇关心） |
| --- | --- | --- |
| 类比 | 一个工程师专注一天 | 一队工程师每 8 小时换班、上一班全部失忆 |
| 核心瓶颈 | context bloat / 工具选择 / eval 飞轮 | 失忆 + 一口气想做完 + 误判已完成 |
| 解决路径 | compaction / MCP / 旋钮调优 | **交接物**（progress file + git + feature_list.json） |
| 状态存储层 | LLM 内存（compaction 后还在内存） | **文件系统**（磁盘 = 唯一持久层） |

**两大失败模式**：

- **模式 A：one-shotting** — agent 尝试一口气干完，半路 context 用完，留下「功能写一半且无文档」的烂摊子
- **模式 B：premature victory** — 项目中后期触发，agent 看到「已经有不少进展」就宣布完工。**根因不是失忆，是没有绝对锚点**——agent 用「代码越多越像完工」的相对感判断

> **易错点**：以为 compaction 解决了跨会话问题。错——compaction 是**会话内**续命，下次新会话开场**根本看不到上次 compaction 的产物**。
>
> **类比**：工程师下班前把脑子里的笔记浓缩贴在自己桌上 → 但第二天来的不是同一个工程师，新人不会去翻别人的桌子。

### L2：双角色分工 + 格式刚性原理

> **核心 mindset（原文脚注）**："We refer to these as separate agents in this context only because they have **different initial user prompts**. The system prompt, set of tools, and overall agent harness was otherwise identical."
>
> **「双 agent」是个错觉——它们是同一个 agent，只是开场 prompt 不同**。被分开的不是 agent，是职责的两个阶段。

**Initializer vs Coding 分工边界**

| 维度 | Initializer Agent | Coding Agent |
| --- | --- | --- |
| 跑几次 | 1 次（项目开篇） | N 次（每个新会话） |
| input prompt | "set up the initial environment" | "make incremental progress, leave structured updates" |
| 产出物 | `init.sh` / `feature_list.json` / `claude-progress.txt` / 首次 git commit | git commit / progress.txt 追加 / passes 字段翻转 |
| 绝对禁止 | 写任何 feature 代码 | **删除或修改 feature_list 的条目** |
| 类比 | 项目总监第一天画蓝图 | 每天执行工程师，按图施工 |

**关键设计：feature_list 必须用 JSON 不是 Markdown**

> **原文原话**："we landed on using JSON for this, as the model is **less likely to inappropriately change or overwrite JSON files** compared to Markdown files."

| 维度 | Markdown 清单 | JSON 清单 |
| --- | --- | --- |
| 格式刚性 | 软（缩进/符号/标题随意） | 硬（多一个逗号都报错） |
| agent 本能 | 「这是文档，我可以重写」 | 「这是结构化数据，只该改字段值」 |
| 失败模式 | 心血来潮重写整个清单、删功能 | 只能动 `passes: false → true` |
| git diff 可读性 | 一片重排，看不清动了什么 | 一眼看到 `"passes": false → true` |

> **第一性原理 P1：格式刚性 = 行为约束**
>
> 想约束 agent 的行为，最有效的不是写 "不要做 X"，而是给它一个**结构化到改不动的容器**。Prompt 是建议，约束是规则——写「不要做 X」只能让 agent 70% 听话，让它**物理上做不到 X** 才能 100% 听话。
>
> **推广例子**：JSON schema required 字段 / 沙箱目录 + chroot / 预填 Markdown 骨架 / 20 字符上限输入框。

**交接物三件套各自防什么病**

```text
┌───────────────────────────────────────────────────┐
│ 1. feature_list.json                              │
│    防病：误判完工、一口气干完                     │
│    机制：绝对锚点（200+ 条强制清单）              │
│    特征：write-once，agent 只能翻 passes 字段     │
├───────────────────────────────────────────────────┤
│ 2. claude-progress.txt                            │
│    防病：失忆、重复犯坑                           │
│    机制：换班日志（自然语言、追加式）             │
│    特征：append-only，最高价值是「坑的快照」      │
├───────────────────────────────────────────────────┤
│ 3. git history（commits + diff）                  │
│    防病：坏改动不可回滚、查不到「上次改了啥」     │
│    机制：版本控制（语义化提交信息）               │
│    特征：原子化、可 revert、不可被 agent 修改历史 │
└───────────────────────────────────────────────────┘
         ↓ 三者互补，缺一个就漏一类问题
```

**管理学映射表**

| 人类组织治理工具 | Agent 工程对应物 |
| --- | --- |
| SOP / 入职手册 | 系统 prompt + INITIALIZER.md |
| issue tracker (Jira) | feature_list.json |
| 工作日志 / 站会纪要 | claude-progress.txt |
| git / SVN | git（直接复用） |
| shift handoff | 会话间 progress + git log |
| 项目经理 | Initializer Agent |
| 工程师 | Coding Agent |

### L3：Testing 仪式——「用户视角」是被忽视的纪律

> **核心 mindset（原文原话）**："Absent explicit prompting, Claude tended to make code changes, and even do testing with unit tests or curl commands against a development server, but **would fail to recognize that the feature didn't work end-to-end**."
>
> **agent 默认的「完工」判定结构上就是错的**——它做单元测试、做 curl、看 dev server 返回 200，就觉得 OK。但用户的「OK」是端到端体验。

**Anthropic 的解：把「用户的眼睛」借给 agent**

```text
┌──────────────────────────────────────────────┐
│  Puppeteer MCP（让 agent 「看见」页面）      │
│                                              │
│  agent → puppeteer.navigate(url)             │
│       → puppeteer.screenshot()               │
│       → 截图返回（vision 模型解读）          │
│       → "我看到按钮但点击没反应"             │
└──────────────────────────────────────────────┘
```

**文章诚实承认的盲点**：

> "Claude can't see browser-native alert modals through the Puppeteer MCP, and features relying on these modals tended to be buggier as a result."

> **易错点**：以为有 vision 工具就解决了所有问题。**vision 也漏 bug**——性能/动画/原生弹窗/微观文字 bug 都看不见。最终防线仍是真人 UAT。
>
> **多工具补救路径**：(1) `page.on('dialog')` 监听原生弹窗 / (2) 替换原生 modal 为自建 DOM modal / (3) console 日志监听 / (4) 升级到 Playwright。

> **第一性原理 P9：用户视角 ≠ Agent 视角**
>
> agent 默认的「完工」判定结构上就是错的。**任何 passes 翻转必须有用户视角证据**（截图 / 图表 PNG / vision 解读）。
>
> **第一性原理 P10：多工具 + 盲区登记 > 完美单工具**
>
> 接受工具有盲点，把盲点显式登记到 progress.txt 的「已知 vision 盲区」区段，定期真人抽检。

### L4：5 篇 Agentic Engineering 交响曲中的位置

> 文章作者亲口承认这套方法可以泛化到 financial modeling / scientific research 等领域，不局限于 web app。

**5 篇 Agentic Engineering mindset 总览**

| 文章 | 一句话 mindset | 最高 ROI 实践 |
| --- | --- | --- |
| 篇一 Code Exec MCP | 工具结果是 token 黑洞 | 高频工具用 MCP 包一层（206→72 tokens） |
| 篇二 Anatomy | harness 是 5 层 stack | 不能 trace 就别调 |
| 篇三 Deep Agents | 结构改 > 模型升 | 加 PlanningMiddleware 一招 +13.7（52.8→66.5） |
| 篇四 Better Harness | eval 是 training data | hand-curate 5 条 eval 当起点 |
| **篇五 Long-running** | **agent 是换班工程师** | **repo 加 .claude/{INIT,CODING}.md 双 prompt** |

**mindset 升级**：

> **读篇五前**：loop 是实现稳定连续运作的关键（停留在篇二 Ralph loop 的内存思维）
>
> **读完篇五后**：借助文档记录是实现跨会话的核心（文件系统思维 — loop 只在单会话内 work，跨会话必须靠落地文件）

## 三个「如果只能记一句」

| 问题 | 一句答 |
| --- | --- |
| **本质是什么** | 跨会话 agent = 换班工程师团队，harness 工作是设计交接物（feature_list + progress + git），把状态从内存搬到文件系统 |
| **最高 ROI 实践** | 在 repo 加 `.claude/INITIALIZER.md` / `.claude/CODING.md` 双 prompt，强制开场仪式（pwd→读 progress→读 git log→读 features→跑 init.sh+smoke test） |
| **思维方式改变** | 从「loop 思维」（篇二 Ralph）切换到「文件系统思维」——loop 解决会话内连续，文件解决会话间连续。两个独立维度，必须叠加 |
