# Agent Harness 解剖：Model 是智能，Harness 才是你能动的杠杆

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://langchain-blog.ghost.io/the-anatomy-of-an-agent-harness/

> Agent = Model + Harness。模型只贡献「智能」，凡是围绕智能让它变成有用工作的工程系统（prompt / tools / fs / sandbox / orchestration / hooks）全都是 harness，也是你唯一能动的杠杆。

---

> **本质一句话**：Agent = Model + Harness。模型只贡献「智能」本身，凡是围绕智能让它变成有用工作的所有工程代码/配置/执行逻辑全是 harness——也是你（不参与模型训练的人）唯一能动的杠杆。
>
> **最高 ROI 单点改动**：把 filesystem 当作 agent 的「持久大脑」用——把所有跨 session、跨 ctx window 的状态（plan、log、中间结果）都落到 fs，再配一条 Ralph Loop 强制续命，长程任务从「半路死」变「能跑完」。
>
> **最容易被忽视的陷阱**：扩 context 窗口（200K→1M→10M）解决不了 context rot——它是质量曲线问题（fill 越满模型越笨），不是装得下装不下的容量问题。

## 为什么需要这篇文章

不看这篇文章的人会怎么想：Agent 就是「一个聪明的大脑」。要让它更厉害，主要靠等 model 升级、或者堆更多 tool 给它。

这种想法坏了以后：(1) 你不是 model 训练方，model 升级不归你管。(2) 堆 tool 反而加剧 context rot——tools 越多，启动时 description 全注入 context 越占位置。(3) 把 agent 当「大脑」会让你忽视一件事——**今天能让一个 model 从「玩具」变成「能干活」的，几乎全部是 harness 工程，不是 model 本身**。

这篇文章用什么视角重新回答问题：把 agent 干脆切成两边——「model = 智能」和「harness = 让智能变得有用的一切系统」。然后用 "Behavior → Harness Design" 的倒推法，从「我想要的 agent 行为」反推「需要 harness 哪些组件」。

## 五层心智模型

### L1 · 核心切法：Agent = Model + Harness

> 原话："**If you're not the model, you're the harness.**"
>
> 凡是不是模型本身的代码/配置/执行逻辑，全部是 harness。

**对比表**：

| 维度 | Model（模型本体） | Harness（系统外壳） |
| --- | --- | --- |
| 本质 | 一个函数：text/image/audio/video → text | 围绕这个函数的所有工程代码 |
| 能力边界 | 只能在 context window 内推理；无持久记忆；不能执行代码 | 给 model 补上 state / 工具 / 反馈循环 / 约束 |
| 举例 | Claude Opus / GPT-5 的权重；ReAct 里「决定调哪个 tool」的推理本身 | system prompt、tool definitions、filesystem、sandbox、subagent orchestration、middleware/hooks |
| 你能改吗 | 不能 | 全部能 |

**ReAct loop 切分图**（model/harness 边界最易混淆处）：

```
┌─── ReAct 循环（整个循环是 harness）──────────────────┐
│   ① while(未完成):                ← while：harness   │
│   ②    把消息历史塞进 model       ← 拼 prompt：harness│
│   ③    model 推理 + 选 tool       ← 这一步：model    │
│   ④    harness 执行 tool 拿结果   ← 工具执行：harness │
│   ⑤    把结果 append 回消息历史   ← 状态维护：harness │
└──────────────────────────────────────────────────────┘
```

> **直觉陷阱**：把 "orchestration" 误认为 model 的事。实际上「选哪个 tool」是 model，「把选择变成实际执行 + 维护现场 + 重新拼下一轮 prompt」全是 harness。记忆口诀：**harness 是舞台+道具+提词器，model 是演员**。

### L2 · 执行三件套：Filesystem → Bash/Code → Sandbox

> 原话："**The filesystem is arguably the most foundational harness primitive.**"
>
> **filesystem 是 harness 最底层的地基**——后面所有 memory、长程执行、Ralph Loop、multi-agent 协作都依赖它。

**三件套依赖关系图**：

```
┌──────────────────────────────┐
│   ③ Sandbox                  │  ← 安全隔离 + 规模化
│   （隔离执行容器+默认工具池）│
│  ┌──────────────────────┐    │
│  │  ② Bash + Code Exec  │    │  ← 给 model 一台「通用计算机」
│  │  （通用问题求解工具）│    │
│  │  ┌────────────────┐  │    │
│  │  │ ① Filesystem   │  │    │  ← 一切的地基
│  │  │ （持久化记忆） │  │    │
│  │  └────────────────┘  │    │
│  └──────────────────────┘    │
└──────────────────────────────┘
     ↑ 越往里越底层、越被依赖
```

**三件套对比表**：

| 组件 | 解决的「模型缺陷」 | 核心能力 |
| --- | --- | --- |
| ① Filesystem | 只能在 ctx window 内推理；跨 session 失忆；信息装不下 | 读写数据/代码/文档；offload 中间结果；跨 session 持久化；多 agent 共享 ledger；+ git 加版本控制 |
| ② Bash + Code Exec | 不可能为每个动作预造一个 tool | 给 model 一台「电脑」，让它**自己写工具** |
| ③ Sandbox | 本地跑生成代码危险；单机不 scale；环境配置混乱 | (a) 安全隔离 + allow-list 命令 + 网络隔离；(b) **预装默认工具池**（语言 runtime、git、pytest、browser） |

> **反直觉点**：sandbox 不只是「安全容器」，它**还是默认工具的提供者**。**预装了什么 ≈ agent 默认能干什么**。一个没装 pandas 的 sandbox，agent 写 Python 分析数据就立刻卡住，得先 pip install 一圈。

### L3 · Context Engineering：让 agent 在长任务里不「老年痴呆」

> 原话："**Harnesses today are largely delivery mechanisms for good context engineering.**"
>
> context 是稀缺资源，harness 的核心活儿就是动态管理它——「动态进 + 动态出」。

**两类需求 vs 四类武器**：

| 需求 | harness 武器 | 怎么干 |
| --- | --- | --- |
| 知识不够新/不够多 | Memory file (AGENTS.md) | 跨 session 知识写进 fs，启动时注入 ctx |
| 同上 | Web Search / MCP | 查实时信息、新版本库文档 |
| Context Rot（填越满模型越笨） | Compaction（被动救火） | ctx 快满时智能总结 + offload |
| 同上 | Tool call offloading（主动预防） | 大 tool output 只保留头尾 N 行 in-ctx，全文 dump 到 fs |
| 同上 | Skills + Progressive disclosure | 启动时只加载 front-matter，用到时才完整展开 |

> **直觉错觉**：「把 ctx 从 200K 扩到 1M 就解决 context rot 了」——错。原文："models become worse at reasoning **as their context window fills up**"。这是质量曲线问题，不是容量问题——填到 50% 时模型就已经在变笨。扩 ctx 只是延后问题。真正解法是「动态进出」。

> **易混淆陷阱**：Tools vs MCP vs Skills 三者的真正区别 = **加载策略**，不是「另一种 tool」。
>
> - Tool/MCP：启动时 description 全注入 ctx → 解决「能调动作」
> - Skill：启动时只加载 front-matter，用到才展开 → 解决「tools 太多导致 ctx rot」
>
> **Skill 不是新组件，是 tool 的装载策略**。

### L4 · 长程自主执行：让前层原语在跨 ctx 下复利

> 原话："**Autonomous software creation is the holy grail for coding agents.**"
>
> 核心毛病三个：① early stopping（半路放弃） ② 拆不动大问题 ③ 跨 ctx window 失忆。
>
> **关键认知**：长程执行不是「一个新组件」，是 L2-L3 所有原语开始**复利**的结果。

**前层原语在 L4 升级映射**：

| L2-L3 原语 | L4 升级版 | 解决的新问题 |
| --- | --- | --- |
| filesystem | filesystem + git | 版本控制 + 新 agent 用 git log 快速 onboard |
| tool offloading | plan.md / progress.md | offload「整个项目状态」跨 ctx 持久化 |
| ReAct loop | **Ralph Loop**（跨 ctx 强制续命） | 防 early stopping |
| memory | planning（动态记录做了/还要做） | 拆不动大问题 |
| sandbox 执行 | self-verification hook | 错了不自知 → 自验证 + error 反喂 |

**Ralph Loop 流程图**（这一层最值得单独学的 trick）：

```
普通 agent loop（无 Ralph）         Ralph Loop（带钩子）
──────────────────────────         ──────────────────────────
prompt → model → 工作 →             prompt → model → 工作 →
model says "done!" → 结束           model says "done!" →
                                           │
但其实并没做完                              ▼ hook 拦截
                                   检查完成条件（plan 全打勾？）
                                   ┌──────┴──────┐
                                   ▼             ▼
                               真做完了      没做完
                               → 放行结束    → 开新 ctx
                                             → 重塞原 prompt
                                             → model 从 fs 读
                                               plan.md + 最新代码
                                             → 接着干
```

> **ReAct vs Ralph 易混淆**：
>
> - **ReAct Loop**：单个 ctx window **内部**的转动，model 自己决定下一步 action
> - **Ralph Loop**：跨多个 ctx window，**harness 强制**（拦截 model 的 exit attempt）
>
> 记忆口诀：**ReAct 是会话内的转动，Ralph 是会话外的强制重启**。

> **反方向的坑**：Ralph 防「该停时停了」，但**不防「不该停时不停」**——model 死循环 / 跑飞了。需要配套：① completion hook（plan 全打勾就停）② budget hook（max iterations / max tokens / max wall-clock time）。

### L5 · 耦合训练 + Harness 的未来

> 原话："**Today's agent products like Claude Code and Codex are post-trained with models and harnesses in the loop.**"
>
> 模型与 harness 是**一起训练**的，所以你看到的「某 model 在某产品里很猛」不是 model 单方面强，是**那个组合被反复训过**。

**耦合训练的两个反直觉效应**：

| 效应 | 含义 | 对你的影响 |
| --- | --- | --- |
| ① 正反馈循环 | 有用的 harness 原语 → 加进 harness → 训下一代 model 时一起训 → model 在这个 harness 里更猛 | 「原厂组合」会持续走在最前 |
| ② 过拟合副作用 | 换 tool 逻辑（如不同的 apply_patch 方言）→ model 性能立刻下降 | 不要轻易换 model 在 IDE 里的 tool 方言 |

> **最反直觉的一点**：原文 "**this doesn't mean that the best harness for your task is the one a model was post-trained with.**" —— 原厂 harness ≠ 对你任务最优。
>
> **Terminal Bench 2.0 案例**："Opus 4.6 in Claude Code **scores far below** Opus 4.6 in other harnesses." LangChain 自家改 harness："improved our coding agent **Top 30 to Top 5** on Terminal Bench 2.0 by only changing the harness."
>
> 带利益相关折扣读：LangChain 卖 deepagents 这个 harness 库，「换 harness 起飞」的论断对他们商业有利；benchmark 单一（编程任务）。但**方向上几乎肯定对**：harness 在同 model 上能造成显著差距。

**Harness 的未来**：

> "**The model contains the intelligence and the harness is the system that makes that intelligence useful.**"

LangChain 在 deepagents 上探索的三个方向：

- Orchestrating **hundreds of agents** in parallel on a shared codebase
- Agents that **analyze their own traces** to identify and fix harness-level failure modes
- Harnesses that **dynamically assemble** tools and context **just-in-time**

## 三个「如果只能记一句」

| 问 | 答 |
| --- | --- |
| **本质是什么** | Agent = Model + Harness。model 提供智能、不可改；harness 提供让智能变成有用工作的所有工程系统，**是你唯一能动的杠杆** |
| **最高 ROI 实践** | 把 filesystem 当 agent 的「持久大脑」：plan / state / 中间结果都落 fs；再配 Ralph Loop 强制续命 + plan 完成性 hook 防偏懒——长程任务从「半路死」变「能跑完」 |
| **思维方式改变** | 读前：把 agent 当「聪明系统」；读后：把 agent 切成「演员（model）+ 舞台道具提词器（harness）」，演员能力上限固定，**真正能改装的是装备** |
