# Code Execution × MCP：让 Agent 写代码，不当调度员

- 类型: 主题笔记
- 标签: AI / Agent
- 收录: 2026-05-18
- 原文: https://anthropic.com/engineering/code-execution-with-mcp

> MCP 解决了连接碎片化，但没解决 agent 的上下文经济学；让 LLM 写代码（而不是逐个 tool call）调用 MCP，单任务 token 可从 150k → 2k（-98.7%），并解锁 Skills 进化路径。

---

> **本质一句话**：MCP 把「连接」解决了，没解决「上下文经济」；让 LLM 写代码调 MCP，而不是逐个 tool call。
>
> **最高 ROI 单点改动**：把「中间结果必经 model」这条物理约束绕过去——用变量代替 token 流。
>
> **最易忽视的陷阱**：以为这是 prompt 调优能解决的问题。错。这是架构层换骨——MCP client 实现选择直接决定 agent 的 token 经济学。

## 为什么需要这篇文章

传统/现状：MCP client 把所有连接的 tool 定义全部前置注入 context，agent 直接 tool call，每次 tool 的返回结果穿过 model context 再被 model 作为下一个 tool 的参数重新输出。这是 MCP 自 2024-11 发布以来事实上的默认实现。

为什么这种做法坏了：当 agent 接 5+ 个 MCP server、上百个 tool 时，用户消息到达之前就已经烧掉几十万 token；中间任何一个 tool 的大返回值（如 2 小时会议转录约 50k tokens）必须过两遍 context——一次进 model，一次以输出 token 形式作为下一个 tool 的参数。结果：贵、慢、易出错、context 超限。

文章新视角：LLM 是写代码的高手——别让它当调度员逐个喊 tool，让它写一段代码交给执行环境跑。代码里中间结果是变量引用，不是 token 重生。原文实测 gdrive → salesforce 工作流从 150k → 2k tokens，节省 98.7%。Cloudflare 称之为 Code Mode，本质同一件事。

## L1 · 两个 token 黑洞（问题陈述）

> **核心 mindset**：MCP 解决了协议碎片化，没解决 agent 的上下文经济学。Tool 越多 → context 越胀 → agent 越慢越贵越笨。

| 黑洞 | 发生时机 | 表现 | 原文证据 |
| --- | --- | --- | --- |
| **定义膨胀** | **用户消息到达之前** | 所有 tool schema 前置注入 context | "agents will need to process hundreds of thousands of tokens before reading a request" |
| **中间结果穿透** | **tool call 之间** | tool A 返回值必须过 model context，再以输出 token 形式作为 tool B 参数重生 | 2 小时销售会议约 **50,000 tokens 走两遍** |

### 中间结果穿透的剧本

```
TOOL CALL: gdrive.getDocument(id=abc123)
        → 返回 50k tokens 转录
                                  ↓ 第一次穿透：进 model context
        → model 必须亲自把 50k 转录逐 token 输出作为下一个 tool 的参数
                                  ↓ 第二次穿透：作为输出 token 重生
TOOL CALL: salesforce.updateRecord(data={Notes: "...50k tokens..."})
```

### 关键数字

- 单任务对比：**150,000 tokens → 2,000 tokens**，**↓ 98.7%**
- 2 小时会议转录穿透成本：约 **50,000 额外 tokens**
- 文章原话："agents routinely build with **hundreds or thousands of tools across dozens of MCP servers**"

> **直觉陷阱**：以为 MCP 是协议层、上下文管理是 prompt engineering 层，两者无关。错。MCP client 的实现选择（直接 tool call vs code execution）直接决定 agent 的 token 经济学——这是架构决策，prompt 调参救不了。

## L2 · Mindset 切换：Tool Call → Code Execution

> **原文金句**："LLMs are adept at writing code and developers should take advantage of this strength to build agents that interact with MCP servers more efficiently."
>
> 白话：别让 LLM 当调度员逐个喊 tool，让它写一段代码扔给执行环境去跑。

### 范式对照表

| 维度 | Direct Tool Call | Code Execution with MCP |
| --- | --- | --- |
| Agent 角色 | 调度员（轮流喊 tool） | 程序员（写一段编排代码） |
| Tool 定义在哪 | **全部**塞进 context（前置） | 放在**文件系统**，按需 import |
| 中间结果 | **必经** model context | **留在**执行环境（变量），model 看不见 |
| 控制流（if/loop） | 每次决策都 round-trip | code 里直接写 if/while/try |
| Token 账单复杂度 | O(tools × turns × data) | O(用到的 tools + console.log) |
| 实测 | 150,000 tokens | **2,000 tokens（↓ 98.7%）** |

### 工程实现：MCP servers 摊成文件树

```
servers/
├── google-drive/
│   ├── getDocument.ts       ← 每个 tool = 一个文件（薄壳）
│   ├── getSheet.ts
│   └── index.ts
├── salesforce/
│   ├── updateRecord.ts
│   └── index.ts
```

每个文件就是包了一层的薄壳：

```typescript
// ./servers/google-drive/getDocument.ts
export async function getDocument(input: GetDocumentInput): Promise<GetDocumentResponse> {
  return callMCPTool<GetDocumentResponse>('google_drive__get_document', input);
}
```

于是 gdrive → salesforce 整个工作流对 model 而言只是 7 行代码：

```typescript
const transcript = (await gdrive.getDocument({ documentId: 'abc123' })).content;
await salesforce.updateRecord({
  objectType: 'SalesMeeting',
  recordId: '00Q5f000001abcXYZ',
  data: { Notes: transcript }   // ← 变量引用，不是 token 重生
});
```

> **陷阱 1**：以为是「换种 prompt 写法」。错：架构层换骨——必须有 MCP client 暴露 tools 为 code API + 沙箱执行环境。
>
> **陷阱 2**：以为「反正都是 LLM 输出，写代码 vs 写 tool call JSON 没差」。错：写代码时数据流靠变量（O(1) 引用）；写 tool call 时数据流靠 token（O(n) 重生）。信息论级别的差别。
>
> **陷阱 3**：以为是「agent 自己 coding」。更准确：agent 把「该做的事」翻译成代码，代码替它跑。LLM 不在循环里反复唤醒——只在写代码和读最终结果这两个端点出现。

## L3 · 四大收益机制

> **核心 mindset**：四个收益机制独立成立、可组合。识别信号 → 对应机制，是新范式的设计反射。

```
              ╔══════════════════════════════╗
              ║  Code Execution with MCP     ║
              ╚══════════════════════════════╝
                           │
    ┌──────────┬───────────┴───────────┬──────────┐
    ▼          ▼                       ▼          ▼
① 渐进披露  ② 结果过滤             ③ 控制流   ④ 隐私
杀定义膨胀  杀穿透                 省 RT+TTFT  PII/合规
```

| 收益 | 识别信号 |
| --- | --- |
| **渐进披露** | tool 数 > 20 或 server > 5 |
| **结果过滤** | tool 返回 > 5k tokens |
| **控制流** | 有 if/loop/retry/轮询 |
| **隐私保留** | 含 PII/账户/密钥 |

### 渐进披露（Progressive Disclosure）

两种实现：

- 文件系统按需读：`ls ./servers/salesforce/` → `cat ./servers/salesforce/updateRecord.ts`
- `search_tools` 工具（带 `detail_level: name_only | name_description | full`）

### Context-Efficient Tool Results（经典场景：10,000 行表格）

```typescript
// 旧范式：10,000 行全部进 model context
// 新范式：
const all = await gdrive.getSheet({ sheetId: 'abc' });
const pending = all.filter(r => r.Status === 'pending');
console.log(`Found ${pending.length} pending orders`);
console.log(pending.slice(0, 5));   // model 只看 5 行，9,995 行未到 model 眼前
```

> **新范式的语法基本功**：`console.log` 永远用 `length + slice(0, N)` 给「总数+样本」，绝不 dump 整数组。这是 LLM 验证逻辑对错的最经济信息组合。

### 强力且经济的控制流

```typescript
let found = false;
while (!found) {
  const messages = await slack.getChannelHistory({ channel: 'C123456' });
  found = messages.some(m => m.text.includes('deployment complete'));
  if (!found) await new Promise(r => setTimeout(r, 5000));
}
```

现实意义（盘中 8 小时、每 30 秒轮询一次行情 = 960 次轮询）：

- 旧范式：每次都把全部历史消息/K 线重新塞回 context（穿透的轮询版变体）
- 估算：可能上百万 tokens / 单日
- 新范式：while + setTimeout 跑在沙箱里，model 全程下班
- 额外收益：TTFT 大幅降低——条件判断不必等 model 推理 round-trip

### Privacy-Preserving Operations

```typescript
for (const row of sheet.rows) {
  await salesforce.updateRecord({
    data: { Email: row.email, Phone: row.phone, Name: row.name }
  });
}
console.log(`Updated ${sheet.rows.length} leads`);
// 真实邮箱/电话/姓名永远没进 model context，只 log 一行汇总
```

进阶：自动 tokenization——MCP client 把 PII 替换成带 ID 的 placeholder。

> **关键陷阱**：tokenization 必须带编号——`[EMAIL_1]` `[EMAIL_2]` 而不是 `[EMAIL]` `[EMAIL]`。后者无法反向映射，多个真值映同一占位符 = 一对多冲突，agent 不知道哪个对应哪个真账户。编号是核心，不是装饰。

## L4 · 状态持久化 与 Skills 闭环

> **原文金句**："Once an agent develops working code for a task, it can save that implementation for future use."
>
> 白话：Filesystem + 持久化 = agent 自己沉淀 Skill 的物理基础。

### 三层递进

```
┌────────────────────────────────────────────────────────┐
│ Level 1: 临时变量       const x = await tool();        │
│            ↓ 跑完就没了                                │
├────────────────────────────────────────────────────────┤
│ Level 2: 文件持久化     fs.writeFile('./workspace/...')│
│            ↓ 下次启动还能读                            │
├────────────────────────────────────────────────────────┤
│ Level 3: 沉淀为 Skill   ./skills/save-sheet-as-csv.ts  │
│           + SKILL.md → 未来 agent 直接 import 复用     │
└────────────────────────────────────────────────────────┘
```

### 三条进化路径对比

| 路径 | 改的是什么 | 谁主导 | 代价 |
| --- | --- | --- | --- |
| Fine-tuning | model weights | 训练团队 | 贵、慢、不可逆 |
| RAG | 检索语料 | 工程师 | 中等 |
| **Skills evolution** | **filesystem** | **agent + 用户** | **零边际成本** |

> **核心洞见**：Model 本身没变，变的是 model 周围的环境。`./skills/` 文件夹越长越胖 = agent 能力进化。这是 environment-side learning，不是 model-side learning——LLM 应用范式里的第三条进化路径。

### Tools / Skills 三者关系

|  | Tools | Skills |
| --- | --- | --- |
| 形式 | 原子动作 | 方法论/打法 |
| 存在哪 | ./servers/ | ./skills/ + SKILL.md |
| 怎么用 | 调用 | 阅读 + 执行 |
| 新东西从哪来 | 开发者写 | 人/agent 沉淀 |

> **陷阱：Tools vs Skills 不能混**。
>
> - **Tool**（./servers/）：怎么做一个**原子动作**（API 薄壳）
> - **Skill**（./skills/）：什么时候组合哪些动作、按什么打法（编排逻辑 + 领域经验）
>
> 混了 → tool description 塞业务逻辑（膨胀） + skill 重复实现工具调用（不可维护）。

## L5 · 代价与边界（何时不该用）

> **核心 mindset**：Code execution 不是免费午餐——拿「context 经济性」换「基础设施复杂度」。账要算清楚再换。

### 三笔代价账

1. **沙箱工程成本**：进程隔离 + 资源限制 + 监控/日志。选型：Cloudflare Workers / Deno Deploy / Docker+gVisor / WASM runtime。每个都有运维成本。
2. **安全责任面扩大**：agent 写错代码影响范围 = 沙箱权限 + 它能 import 的所有 server。必须有超时、网络白名单、文件系统隔离、memory limit。
3. **监控/可观测性**：一段代码里的 5 个 tool 调用需要单独埋点才能 trace。

### 决策表

| 场景 | 该用? | 原因 |
| --- | --- | --- |
| Tool < 10 且工作流 1-2 步 | **否** | 收益 < 沙箱成本 |
| Tool > 30 或 server > 5 | **强烈推荐** | 渐进披露边际收益巨大 |
| 中间结果常 > 5k tokens | **是** | 穿透痛已显性 |
| 多步骤循环/重试逻辑 | **是** | 控制流收益明显 |
| 处理 PII / 合规敏感数据 | **是** | tokenization 是合规底线 |
| 原型期 / Demo / 单次任务 | **否** | 沙箱搭建时间 > 项目本身 |
| 没有可信赖的 sandbox 方案 | **否** | 安全风险 > 收益 |

> **陷阱 1**：被 98.7% 冲昏头无脑铺开。真相：那是极端场景值，日常 5-10 个 tool 节省 30-50%，扣掉沙箱成本可能得不偿失。
>
> **陷阱 2**：「Cloudflare/Anthropic 都搞，我也搞」。真相：他们有 infra 团队。单兵作战时先用直接 tool call 跑通业务，痛点显性后再切。
>
> **陷阱 3**：当通用解药。真相：只治 context 经济性 + 数据流隔离。如果痛点是「agent 推理质量差/选错 tool/prompt 不稳」，code execution 一个都治不了。

## 三个「如果只能记一句」

| 维度 | 一句话 |
| --- | --- |
| **本质是什么** | 把 tool 调用从「LLM 逐个喊」改成「LLM 写一段代码、执行环境跑」——中间结果靠变量引用，而非 token 重生 |
| **最高 ROI 实践** | 把任何中间结果 > 5k tokens 或含轮询循环的工作流，重写成 code execution，跑在沙箱里 |
| **思维方式改变** | Agent 的能力进化不靠 fine-tune 或 RAG，而是靠 `./skills/` 越来越胖——environment-side evolution，零边际成本 |
