# LLM Tool Use 全景：从机制到生产工程

- 类型: 主题笔记
- 标签: AI / Agent
- 收录: 2026-05-18
- 原文: https://github.com/anthropics/courses/tree/master/tool_use

> 模型只会生成文本；tool use 是一套结构化协议，让 harness 替模型动手，并通过 agent loop 把结果喂回去——可靠性来自结构化通道、好的工具描述、精准的 tool_choice 和错误透传。

---

## 一、核心心智模型

> LLM 不会「使用」工具。它只是写一张「工具调用单」，harness 替它去执行。

整个 tool use 就是把「模型只能生成文本」这条物理约束，用一套结构化协议变成「能办事的 agent」。

**餐厅类比**：模型是顾客，只会在订单纸上写字；服务员（harness）拿着订单去厨房（真实函数），把菜端回来。模型永远不走进厨房。

## 二、六层结构

### Layer 1｜模型只能生成文本

物理事实，不是设计选择。模型不能直接发邮件、查数据库、执行代码——只能吐字符。一切「工具使用」都是建立在这条约束之上的协议。

### Layer 2｜输出是分通道的结构化 block

一次 API 回复 = content 列表，每个元素有自己的 type：

```python
response.content = [
    { "type": "text",     "text": "好的，我帮你查一下。" },
    { "type": "tool_use", "name": "send_email", "input": {...} }
]
```

**两个常见陷阱**：

- 模型在 text block 里写一段「看着像 JSON」的字符串 ≠ 工具调用。Harness 只认 type 字段。
- text 和 tool_use 可以同时出现，不是二选一（「边说话边写订单」）。

### Layer 3｜Agent Loop + tool_result

```javascript
模型 → tool_use(id=abc) → harness 执行 → tool_result(tool_use_id=abc) → 模型
                                                                              ↓
                                                              下一轮继续，或只剩 text 即终止
```

**核心对称性**：模型 → harness 必须用结构化 tool_use；harness → 模型也必须用结构化 tool_result，靠 tool_use_id 绑回请求（取餐号牌制度）。

**终止条件**：当模型这一轮的输出只有 text、没有 tool_use，loop 结束。

### Layer 4｜工具菜单（name / description / input_schema）

每次调 API 传 tools 参数告诉模型有哪些工具：

```python
{
  "name": "send_email",
  "description": "Send an email. Use when... Do NOT use for... For X, use Y instead.",
  "input_schema": { ... JSON schema ... }
}
```

| 字段 | 给谁看 | 作用 |
| --- | --- | --- |
| name | 模型 & harness | 暗号——模型吐出这个名字，harness 调对应函数 |
| description | 主要给模型 | 决定模型选不选这个工具（命脉） |
| input_schema | 模型 & API | 参数形状 + 自动校验 |

**最重要的一条**：description 写的不是「工具是什么」，而是「何时该用 / 何时不该用 / 与相似工具的区别」。坏 description → 模型选择不稳定 → 80% 对 20% 错的不可 debug 系统。

### Layer 5｜tool_choice 四档严格度

```javascript
最自由 →→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→→ 最严格
   auto          any              tool             none
    │             │                │                │
模型自己决定   必须用工具       必须用指定的       禁止用任何
要不要用     模型选哪个         那个工具           工具
```

| 档 | 典型场景 |
| --- | --- |
| auto | 通用 chatbot |
| any | 路由 / 分类（必须做选择） |
| tool | 强制结构化输出（把 schema 校验交给 API） |
| none | 收尾轮 / 总结轮，避免再调工具 |

**关键洞察**：模型能力 ≠ 输出可靠性。即使模型 95% 选对，剩下 5% 也会让下游崩。需要确定性时，从一开始就用 tool，不要「先 auto 看看不行再换」。这是契约式思维，不是反应式思维。

### Layer 6｜并行调用 + 错误回传

**并行**：一轮 response 里可以有多个 tool_use block。Harness 应该并发执行（asyncio.gather / Promise.all），把所有 tool_result 装在同一条 user message 里返回，靠 tool_use_id 绑回。

**错误**：tool_result 有 is_error: True 标记。失败时把 traceback 原样塞进去，不要在 harness 里吞掉异常。模型看见错误后通常能自己恢复（改参数重试 / 换工具 / 认拖告诉用户）。

最常见的新手错误：harness 默默 try/except 吞掉异常。请永远把错误透传给模型——它能看见才能恢复。生产 agent 90% 的稳定性提升都来自这一块。

## 三、生产工程的三个旋钮

机制只搭一次，日常 90% 的工作在调这三个：

1. **description 怎么写**——决定模型选不选对工具
2. **tool_choice 选哪档**——决定输出的确定性
3. **错误怎么回传**——决定 agent 的鲁棒性

## 五、使用场景参考

- 中间探索轮（跑处理、拿指标）用 tool_choice = "auto"
- 收尾轮强制 tool_choice = {"type":"tool", "name":"submit_final_report"}，保证下游拿到固定 schema 的 JSON
- 所有工具的失败都透传给模型，不要吞掉
- 用「好 description」明确区分相似工具的边界
