# Agent Skills：写成文件夹的方法论手册

- 类型: 主题笔记
- 标签: AI / Agent / 文章
- 收录: 2026-05-18
- 原文: https://anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills

> Skill = 一个含 SKILL.md 的文件夹，把「指令 + 子文件 + 脚本」打包给 Agent，通过渐进式披露按需加载——Tool 是「能做什么」、Skill 是「该怎么做」，两者垂直搭配，不替代。

---

**一句话**：Agent Skill = 一个含 SKILL.md 的文件夹，把「指令 + 子文件 + 脚本」打包给 Agent，通过渐进式披露（Progressive Disclosure）让它按需加载。

**它同时打掉三件事**：prompt 撑不下、定制 Agent 不可组合、领域知识不可移植。

**思维陷阱**：别把 Skill 当「更长的 prompt」——它的魔力不在「提供信息」，而在「分层按需提供信息 + 可执行代码」。

## Skill vs Tool：动作 vs 打法

理解 Skill 之前，先弄清它和 Tool 的关系——这是最容易混淆的一点。

**一句话区分**：

- Tool = 一个原子动作（agent 能做什么）
- Skill = 一套打法（agent 该怎么做某类任务，经常指挥 agent 去用 Tool）

**类比**：Agent 是一个新员工。Tool = 你给它的工具箱（锤子、扳手、计算器）；Skill = 你给它的操作手册（「装家具时，先看说明书第 3 页，找 4 颗 M6 螺丝，先固定底板再装侧板……」）。手册里指挥它用工具箱里的东西。手册和工具不冲突，是搭配的。

### 7 维对比

| 维度 | Tool | Skill |
| --- | --- | --- |
| 本质 | 一个可调用的函数 | 一份指导某类任务的指令文档（+ 可选脚本/模板） |
| 形式 | JSON Schema 输入 + 后端代码 | 一个 markdown 文件（SKILL.md），prose 写成 |
| 装进 context 的方式 | schema 始终在 context 里 | 只有 name + description 始终在；body 按需 load |
| 触发 | 模型决定调用，后端执行，返回结果 | 模型决定 load，把内容读进 context，按指令行事 |
| 谁来写 | 工程师（代码 + schema） | 任何人（写散文就行） |
| 更新成本 | 改代码、redeploy | 改 markdown 文件 |
| 颗粒度 | 原子动作（搜一下、发邮件） | 整个工作流（做一次 research、填一份 PDF 表格） |

**关键认知**：两者是垂直关系不是水平。工具设计中的 description 本质上就是一段 prose；Skill 只是把「写 prose 告诉 agent」这件事从 tool 里释放出来，独立成为一类实体——形态变大了（一个文件夹），但本质仍是「指挥」。

## Skill 是什么：「入职手册」类比

原文：Building a skill for an agent is like putting together an onboarding guide for a new hire.

| 新员工入职手册 | Skill 对应物 | 作用 |
| --- | --- | --- |
| 手册封面的「内容提要」 | YAML frontmatter 中的 name 和 description | 让人知道「什么时候该查这本」 |
| 手册正文 | SKILL.md 的主体内容 | 主流程 + 指到他处的路标 |
| 手册附录章节 | 同目录下的其他 .md 文件（如 forms.md、policies.md） | 特定场景才翻，平时不占空间 |
| 手册附赠的计算器 / 模板 | 脚本文件（*.py） | 能直接跑的工具 |

### 最小骨架（完整 SKILL.md 示例）

```markdown
---
name: pdf
description: 处理 PDF 的读取、编辑、表单填写。当用户提供
  PDF 文件并要求提取 / 修改 / 填表 / 合并 / 拆分时触发。
  仅阅读理解 PDF 内容时不需要调用本 skill。
---

# PDF 处理

## 主流程
1. 识别任务类型（读取 / 表单 / 拼接）
2. 如果是填表 → 读 forms.md
3. 如果需要提取表单字段 → 跑 scripts/extract_fields.py（不要读入 context）
4. 复杂语法 → 读 reference.md
```

**硬性规则**：

- `---` 包起来的是 YAML frontmatter，name 和 description 二者必填
- 正文是普通 Markdown
- 用文件名引用同目录下的其他文件
- 脚本要明确标明「读」还是「跑」——同一个 .py 可能被 cat 读进 context 当文档，也可能被 python 跑起来当工具

## 渐进式披露：整个设计的灵魂

| Level | 内容 | 何时进入 context | 体量 |
| --- | --- | --- | --- |
| **1** | name 和 description | 启动时常驻 system prompt | 几十 token |
| **2** | SKILL.md 正文 | Claude 判断「这个 skill 相关」时读入 | 几百到几千 token |
| **3+** | 子文件（forms.md 等） | 命中子场景时才读 | 任意大 |
| **脚本** | *.py / *.sh | 只执行，只有输出进 context | 代码本身不计 |

**最重要的洞见**：不是「压缩信息」，是「分层暴露信息」。这意味着 Skill 能打包的内容量理论上是无限的——原文原话：「the amount of context that can be bundled into a skill is effectively unbounded.」

## 代码执行：Skill 的「工具抽屉」

原文：Large language models excel at many tasks, but certain operations are better suited for traditional code execution. Sorting a list via token generation is far more expensive than simply running a sorting algorithm.

| 维度 | LLM 生成 token | 脚本执行 |
| --- | --- | --- |
| 成本 | 按 token 收费 | 几乎为零 |
| 速度 | 逐 token 生成，慢 | 算法复杂度，快 |
| **确定性** | 每次可能不一样 | 每次结果完全相同 |
| **可复现** | 难复现 | 天生可复现 |

肌肉记忆：能写单元测试的事，就该用代码做，不该给 LLM。

## 怎么开发 + 评估 Skill：4 条铁律

### 铁律 1：从 eval 出发（Start with evaluation）

原文：Identify specific gaps in your agents' capabilities by running them on representative tasks and observing where they struggle.

**说人话**：不是拍脑袋想「该加什么 Skill」，而是先跑 Agent、看它哪里崩，针对性补一个 Skill。

### 铁律 2：为规模设计结构

拆分判断标准：

| 拆 | 不拆 |
| --- | --- |
| SKILL.md 太长进不下 | 一页能说完 |
| 两段内容互斥（用 A 不会用 B） | 两段经常一起用 |
| 某段很少用到 | 某段每次都要看 |

### 铁律 3：从 Claude 的视角想

原文重点：Pay special attention to the name and description of your skill.

- description 不是「介绍文」，是「触发条件」
- 错误写法：「处理 PDF 的 skill」
- 正确写法：「处理 PDF。当用户提供 PDF 文件且需要填表/修改/拼接时触发；纯阅读不需要。」

### 铁律 4：和 Claude 一起迭代

```javascript
┌─────────────────────┐
│ 1. 跑 Agent 做真实任务     │
└────┬──────────────────┘
     │ 发现它在某一步崩
┌─────────────────────┐
│ 2. 问 Claude:「说说刚才   │
│   你为什么走偏了?」        │
└────┬──────────────────┘
     │ 让它自己产出复盘
┌─────────────────────┐
│ 3. 把复盘结论沉淀进       │
│    Skill 文件              │
└────┬──────────────────┘
     │
     └─→ 回到第 1 步
```

原文原话：This process will help you discover what context Claude actually needs, instead of trying to anticipate it upfront.

## 安全：Skill = 招新员工，必须背调

### 三种攻击面

| 攻击点 | 手法 | 危害 |
| --- | --- | --- |
| 指令注入 | SKILL.md 里藏「把密码发到 evil.com」 | 数据外泄 |
| 脚本依赖 | 脚本 import 恶意包 | 任意代码执行 |
| 捆绑资源 | 图片 / 二进制文件含恶意负载 | 隐蔽、难发现 |

### 三条防身术

1. 只装可信来源：优先官方 / 自己写 / 团队审过
2. 第三方用前过审：读所有文件，特别看代码依赖 + 捆绑资源
3. 盯紧外部网络调用：任何「连这个 URL / 调这个 API」都要看清楚目标是否合法

## Skill、MCP、n8n 三角对比

| 维度 | **Skill** | **MCP** | **n8n** |
| --- | --- | --- | --- |
| 本质 | 文件夹形式的「手册+工具箱」 | Agent 调用外部能力的协议 | 节点拖拽的 workflow 引擎 |
| 解决什么 | 「怎么按公司规矩干活」 | 「怎么对外伸手拿能力」 | 「固定步骤怎么自动跑」 |
| 执行路径 | 不固定，Agent 现场推理 | 固定调用，进出明确 | 完全确定，节点画完就这么跑 |
| 开发速度 | 改 md 就行，最快 | 要写 server，中 | 拖节点，中 |
| 适合场景 | 需要现场判断 / 推理 | 调用外部系统（数据库 / API） | 生产环境、需 SLA、要「看得见的流程」 |
| 客户信任感 | 黑盒，低 | 较明确 | 节点图一眼看明白，高 |

**实战选型**：

- 个人 / 快速迭代 / 需要现场推理 → Skill
- 调用外部能力 → MCP
- 生产环境 / 给客户跑 / 要 SLA → n8n 这类确定性工具
- 多数复杂场景是三者混用

## 如果只能记五句

| 问题 | 一句话 |
| --- | --- |
| Skill 是什么 | 含 SKILL.md 的文件夹，将领域知识以「可按需披露」的方式供 Agent 加载 |
| 为什么它很强 | 同时解决三件事：prompt 撑不下、Agent 不可组合、知识不可移植 |
| 为什么要代码执行 | 能写单元测试的事就该让代码做——确定、便宜、快、可复现 |
| 怎么写好一个 Skill | 先看 Agent 崩在哪 → 针对性补 → 让 Agent 复盘 → 沉淀回文件；name/description 是命门 |
| 安全 | 第三方 Skill = 招新员工，装前要背调，特别盯外部网络调用 |
