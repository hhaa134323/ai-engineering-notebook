# Deep Agents 的 context 管理：可逆优先 + 双轨保存 + signal amplification eval

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://blog.langchain.com/context-management-for-deepagents/

> Long-running agent 在 context 满之前就会因 context rot 退化——Deep Agents 用「先可逆后不可逆」的三段压缩 + 双轨保存 + 主动加压的 targeted eval，把 harness 做成可工程化、可验证的产物。

---

**本质一句话**：Deep Agents 把 long-running agent 的 context 压缩做成了「先可逆后不可逆」的三段式 + 双轨保存 + 主动调频 eval 的工程配方。

**最高 ROI 单点改动**：把「write/edit 入参换文件指针」这个零损失压缩抄进 agent harness——context 里 50%+ 的占用可能是冗余文件内容。

**最容易被忽视的陷阱**：long-running agent 翻车 ≠ crash，是 goal drift（agent 还在跑但已偏离原 intent）——log 看着在干活，结果一片狼藉。

## 为什么需要这篇文章

传统对 long-running agent 上下文的直觉是「等 window 满了再处理」或「加大 context window 就好」。事实是：

- LLM 的 context window 是硬约束（撑爆即截断），但真正的杀手是软约束——context rot：window 没满之前模型对早期 token 的利用质量已经在衰退。1M window ≠ 1M 有效记忆，装到 30-40% 就开始「看糊」。
- 传统聊天 agent 用 sliding window / session summary / RAG 三招处理 history，但这套假设 human-in-the-loop——用户每轮天然带新 context。Long-running agent 是 agent-in-the-loop，没人给它续命，必须自己管自己的记忆。
- 这篇文章用一个反直觉的视角回答问题：压缩不是一个动作，是一套有顺序、有阈值、有可逆性差异、必须配对 eval 的策略——而且这套策略的某些原则（尤其 signal amplification eval）根本不是 agent 专属。

## 4 层心智模型

### L1 — Why：context rot 是 long-running agent 的生死线

**核心 mindset**："Context window 不是装东西的盒子，是质量随容量下降的雷达屏。装得越满，看得越糊。"

**原文金句**："As the addressable task length of AI agents continues to grow, effective context management becomes critical to prevent context rot and to manage LLMs' finite memory constraints."

约束两面观：

| 约束 | 性质 | 表现 | 能否靠加大 window 解？ |
| --- | --- | --- | --- |
| 硬约束（容量） | 物理上限 | 装满 → 截断 / 报错 | 能（治标） |
| 软约束（context rot） | **质量随容量衰退** | 早期 token attention 被稀释、key 信息被淹没 | **不能**——1M window 在 30-40% 时就开始看糊 |

**陷阱**：直觉认为「DeepSeek V4 有1M context 就不用管 context engineering」——这正是 context rot 的盲区。1M window 不是 1M 有效记忆，是「前 200K 还能看清、之后越来越糊」的滑梯。

真实失败样貌（这是原文最值钱的洞见之一）：long-running agent 翻车不是 crash，是「agent 还在跑但已经偏离原 intent」——具体表现：

- 重复工作（忘了已经改过哪个文件）
- 假完成（突然宣布任务完成，其实只做了 30%）
- 工具 hallucination（编一个不存在的 tool）
- goal drift（从「重构 schema」飘到「优化某个无关函数」）

原文专门把它叫做 **"the most insidious failure mode"**——最阴险的失败模式。

### L2 — How：三段式压缩 + 阈值

**核心 mindset**：压缩不是一个动作，是一套有顺序的策略——先扔最便宜的，再扔次便宜的，最后才动昂贵的。

**隐含原则（最值得记住）**：先做可逆的、再做不可逆的。

三种动作对比：

| 动作 | 触发条件 | 具体阈值 | 动作 | 信息损失 | 可逆性 |
| --- | --- | --- | --- | --- | --- |
| A. Offload 工具结果 | 单次结果过大 | > 20K tokens | 存 filesystem，context 留 path + 前 10 行预览 | 微损失 | 可逆 |
| B. Offload write/edit 入参 | context 总量过大 | > 85% window | 把老的 write/edit 入参换成文件指针 | **零损失** | 完全可逆 |
| C. Summarization | B 已做完仍不够 | 同上更严格阈值 | 整段对话生成结构化摘要 | 真损失 | 不可逆（双轨保存救场） |

信息损失序 vs 执行序——这两件事不一样：

```javascript
执行顺序：A → B → C       (按触发频率/成本)
信息损失：B < A < C       (write/edit 入参是零损失「免费午餐」)
```

**关键洞察（最精妙的一手）**：B 之所以是「零损失免费午餐」，因为 write/edit 的工具调用入参就是完整文件内容，而这份内容已经在 disk 上了——context 里那一大坠是 100% 冗余信息。未来设计任何 agent harness，先扫一遍系统：哪里的数据被存了两份？那就是零成本优化点。

**陷阱**：以为压缩越激进越好。阈值太低（如5K）会让 agent 反复 re-read 文件，额外 round-trip 反而更耗 token；阈值太高（如50K）让 rot 在压缩前已经发生。20K ≈ 200K window 的 10%，是「in-context 利用成本 vs offload 再读回的 round-trip 成本」的平衡点。

### L3 — 设计哲学：双轨保存为什么不能省任何一边

**核心 mindset**：摘要不是「压缩」，是「working memory 和 long-term memory 的分离」。Agent 双轨设计直接抄人脑——你记得「上周开过会」（working），但要回忆「谁说了什么」得翻笔记（long-term）。

**原文金句**：summary includes "session intent, artifacts created, and next steps"——三个字段不是随便挑的。

```javascript
    ┌────────────────────────────────────┐
    │     原始完整对话历史 (N turns)        │
    └────────────────────────────────────┘
            │                       │
            ▼                       ▼
┌────────────────┐     ┌────────────────────┐
│ 轨道 1: in-context│     │ 轨道 2: filesystem    │
│  结构化摘要        │     │  完整原始消息持久化     │
│                  │     │                      │
│  - session intent│     │  (canonical record)  │
│  - artifacts     │     │                      │
│  - next steps    │     │                      │
└────────────────┘     └────────────────────┘
         │                          │
         ▼                          ▼
  "agent 知道在哪儿"           "agent 能查到细节"
   (working memory)          (long-term memory)
```

两轨缺一会发生什么：

| 缺失 | 失败模式 |
| --- | --- |
| 只留轨道 1（无 filesystem） | 摘要 lossy 丢的细节回不来——摘要里写「实现了 schema」但你问「某常量用 252 还是 365 天」，agent 不知道、也回不去查。数据精度敏感项目里丢一个常量就是 bug。 |
| 只留轨道 2（无 in-context 摘要） | 「信息没丢但不知道要找什么」——filesystem 像图书馆但 agent 没索引；RAG/search 需要 query，没有 working memory 连 query 都生成不出来。比「丢失」更阴险。 |

**最值钱的迁移点**：给人看的摘要 ≠ 给 agent 看的摘要。

- 给人的摘要：narrative（"我们讨论了 X，然后..."）
- 给 agent 的摘要：resume token（intent / artifacts / next steps = agent 可恢复执行的最小完备集）

设计任何 agent 的 state schema / checkpoint / resume prompt，都套这三项模板。

### L4 — Eval：signal amplification（最反直觉、最通用）

**核心 mindset**：低频但关键的失败模式，靠等是等不到的，必须主动制造高频信号才能测得动。

**原文操作**："triggering summarization at 10-20% of the available context window may lead to suboptimal overall performance, [but] it produces significantly more summarization events. This allows for different configurations ... to be compared."

信号放大是「控制变量、人为加压、提高事件密度」的通用方法论，**不限于 agent**：

| 维度 | LangChain 例（agent harness） |
| --- | --- |
| 想测的低频失败 | summarization 出错（goal drift / 信息丢失） |
| 默认下多久触发 | 85% window 满才触发——一个 benchmark run 只 1-2 次 |
| 「加压」动作 | 阈值改成 10-20% |
| 代价 | 整体性能变差 |
| 收益 | 每个 config 都能稳定看到 30+ 次事件，能统计对比 |

两个 targeted eval 与双轨设计严格对偶：

| Eval | 测什么失败模式 | 对应 L3 轨道 |
| --- | --- | --- |
| Objective preservation（强制 summarize 后看是否继续走原目标） | **goal drift** | 验证轨道 1（intent 字段真的留住了原意图） |
| Needle-in-haystack（早期埋事实 → 强制 summarize → 看能否找回） | **lossy summarization 丢细节** | 验证轨道 2（filesystem 完整 + agent 真会主动 read_file 找回） |

**最值钱的一点**：一个设计、一个验证，永远成对出现——这是工程美感的标志。L3 双轨架构 + L4 双 eval 严格对偶。看到任何 agent harness 设计都用这把尺子量一下：它有没有给自己的关键设计配上对偶的 eval？没配就是没说透。

**陷阱**：把 signal amplification 和 A/B 测试 sample size 计算混淆。A/B 是「加样本量」，amplification 是「加事件密度」——同一时间窗内人为提高目标事件的发生频率。两件事。

## 三个「如果只能记一句」

| 问题 | 一句话答案 |
| --- | --- |
| **本质是什么** | Long-running agent 的 context 必须主动管理——用「先可逆后不可逆」的三段压缩 + 双轨保存 + signal amplification eval，把 harness 做成可工程化、可验证的产物。 |
| **最高 ROI 实践** | 给你的 agent harness 加上「write/edit 入参 offload」——零信息损失的免费午餐，能直接砍掉 context 里 50%+ 的冗余文件内容。 |
| **思维方式改变** | 「压缩」不是一个动作而是一套有顺序的策略；「摘要」不是给人看的 narrative 而是给 agent 用的 resume token；「eval」不是被动看 benchmark 而是主动加压制造信号。 |
