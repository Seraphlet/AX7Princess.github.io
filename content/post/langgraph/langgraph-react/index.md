---
description: ""
title: "手搭 ReAct 图"
draft: false
date: "2026-09-05T09:38:22+08:00"
slug: "LangGraph-React"
categories:
 - LangGraph
tags:
 - React
image: ""
---


# W5-D3 · 手搭 ReAct 图：把 W2 的 fc_loop 画成一张带环的图

> **关键词**：`add_messages` reducer / `ToolNode` / `tools_condition` / 回边 / ReAct 拓扑
> **前置**：W2 `fc_loop`、W4-D5 `bind_tools`、W5-D1 四件套、D2 条件边
> **涉及文件**：`W5/lg_d3_react.py`
> **本日地位**：🔴 **W5 第一个里程碑**——跑通它，你就有了一个**图形态的 Agent 骨架**
> **收尾预告**：D4 Stream 模式深入（`updates` / `values` / `messages` 三种模式 + `astream_events` 的 token 级输出）

* * *

## 零、今天为什么是里程碑

**D1 你学会了画直线，D2 学会了画岔路，D3 要画一个圈。**

前两天的图都是**无环**的——`START → ... → END`，一路向前不回头。今天要加一条**回边**，让图跑成一个圈。

这一步的分量在于：**你 W2 手写的 `fc_loop`、W4 手搓的 `_run_tools`、W4-D6 的 `while ai.tool_calls`——到今天全部有了统一的图形态表达。** 从 W2 开始困扰你的那个循环，终于有了一个"能画出来"的样子。

> 💡 今天这课 **90% 是落地验证训练**——"跑通"不是终点，**"亲眼确认链路对"**才是。文末会问你那句"你怎么知道它做对了"，先记着这个问题往下看。

* * *

## 一、先给地图：今天要搭的图

```fallback
                    ┌─────────────────────────────┐
                    │                             │
                    ▼                             │
START → agent ──(有 tool_calls)──→ tools ────────┘
            │
            └──(无 tool_calls)──→ END
```

**一句话本质：这就是你 W2 手写 `fc_loop` 的画法。**

```gdscript3
  while True:              →  tools → agent 回边（圈）
  if tool_calls:           →  tools_condition 条件路由
  execute + append 结果    →  ToolNode 自动执行并回填
```

你 W4 用 `while` 手搓的东西，今天变成一张**会自己转圈的图**。

> **类比**：W2 你是一个**人工接线员**——手动接电话（调 LLM）、看要不要转工具、手动把结果塞回去再打一遍。今天这张图是**自动交换机**——agent 节点拨号、tools 节点自动接通并回填结果、没电话了就自动挂断（END）。
>
> **你从"干活的人"变成了"设计交换机的人"。**

* * *

## 二、三个新概念

### 2.1 `messages` 字段 + reducer（为什么消息不会互相覆盖）

#### 问题：默认行为是覆盖

D1 讲过，state 字段的默认更新策略是**覆盖**——节点 `return {"count": 1}`，旧值就被顶掉。

**但聊天消息不能覆盖。** 第二轮不能把第一轮清空，否则模型直接失忆：

```fallback
  第 1 轮  messages = [Human, AI]              记着"我叫小明"
  第 2 轮  return {"messages": [AI]}           覆盖！
           messages = [AI]                     ← 上一轮的对话没了
  第 3 轮  模型看到只有一条消息，根本不知道你叫什么
```

#### 解法：给字段声明一个 reducer（合并器）

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]   # 追加，不是覆盖
```

`Annotated[类型, 合并函数]` 的意思是：**这个字段的更新方式，用第二个参数指定的函数来合并，而不是直接覆盖。**

> **类比**：普通字段是**便签纸**（写新的就盖掉旧的）；`messages` 是**聊天记录本**（只能往下加页，不能撕掉前面的）。`Annotated[list, add_messages]` 就是在本子封面上贴了张条："这个字段只许追加"。

#### 🎯 `add_messages` 不只是 append

市面上常见另一种写法 `Annotated[list[AnyMessage], operator.add]`，简单场景下效果接近，但 `add_messages` 其实更聪明：

| 行为 | `operator.add` | `add_messages` |
|---|---|---|
| 追加新消息 | ✅ 列表拼接 | ✅ 追加 |
| **按 `id` 更新已有消息** | ❌ 会变成两条 | ✅ 同 id 覆盖更新 |
| **删除消息**（`RemoveMessage`） | ❌ 不支持 | ✅ 支持 |

```python
# add_messages 的"更新"语义：同 id 的消息会被替换而不是追加
# 这让"覆盖某条历史消息""删除某条消息"成为可能
from langchain_core.messages import RemoveMessage
return {"messages": [RemoveMessage(id=msg_id)]}
```

> 💡 现在用不到删除/更新，但要知道**这个设计是为 W6 的"上下文压缩"留的口子**——压缩的本质就是"删掉中间几条、换成一条摘要"。`add_messages` 从第一天就为此做好了准备。

**回到眼前**：没有 reducer，工具结果回填后下一轮就被清空，**Agent 根本转不起来**。这是整张图的地基。

### 2.2 `ToolNode`（自动执行工具 + 回填结果）

`ToolNode(tools)` 把你的工具列表包成一个**图节点**。它自动做三件事：

```fallback
  ① 读 LLM 最后一条消息里的 tool_calls
  ② 逐个调用对应的工具函数
  ③ 把结果包装成 ToolMessage，追加回 state 的 messages
```

```python
builder.add_node("tools", ToolNode(tools))
```

就这么一行，替换掉你 W4 手写的整个 `_run_tools` 函数。

#### 🎯 这里必须点破一件事

你 **W4 踩过的 `insufficient tool messages` 坑**——模型声明了 N 个 `tool_calls`，你少回填一条，下一轮 `invoke` 就 400 报错。

那个坑的本质是：**工具调用协议要求"发几个 call，就必须回填几条 ToolMessage"。**

`ToolNode` 从根上解决了这个问题——**它保证每个 `tool_call` 都有对应的 `ToolMessage`，一个不落**。你一行都不用写。

```fallback
  W4 的痛：                              W5 的解：
  手动 for 循环执行                   →   ToolNode 自动执行
  失败要 try/except 手动回填          →   ToolNode 自动回填（含失败）
  绑了工具却没有执行节点 → 空 content  →   两者成对出现才完整
  并行调用怕回填不齐 → 只能强制串行    →   并行也能逐条正确回填
```

> ⚠️ 还记得 W4 为了绕开并行调用的问题，你特意加了 `parallel_tool_calls=False` 吗？今天用 `ToolNode`，**模型一次并行发 3 个 `tool_calls` 也能全部正确回填**——那个妥协可以放下了。

### 2.3 `tools_condition`（官方替你写好的路由函数）

D2 你亲手写过 router（`decide_path` 读 state、返回字符串）。今天这个路由**官方帮你写好了**：

```python
tools_condition
# 等价于：
#   最后一条 AI 消息有 tool_calls → 返回 "tools"
#   没有                          → 返回 END
```

**你 D2 学的一切在这里原样复用**：

```python
builder.add_conditional_edges("agent", tools_condition)
```

挂在 `agent` 节点后面，返回 `"tools"` 就去工具节点，否则图结束。

#### 它的判据从哪来

往上追一层——模型返回的 `AIMessage` 里有个字段叫 **`finish_reason`**：

```fallback
finish_reason: 'tool_calls'   → 模型说"我要调工具，话没说完"
finish_reason: 'stop'         → 模型说"我说完了"
```

`tools_condition` 看的就是最后一条 AI 消息**带不带 `tool_calls`**（底层信号就是 `finish_reason`）。

> 💡 用 `stream` 跑一次，你会亲眼看到这个切换：第一条 agent 消息的 `finish_reason` 是 `'tool_calls'`，最后一条是 `'stop'`。**看到这个变化，你就知道路由在按预期工作。**

* * *

## 三、完整可运行代码（逐块拆解）

### 3.1 State + 工具

```python
from typing import Annotated, TypedDict
from langgraph.graph.message import add_messages
from langchain_core.tools import tool

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def add(a: int, b: int) -> int:
    """计算两个整数之和"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """计算两个整数之积"""
    return a * b

tools = [add, multiply]
```

**`@tool` 的价值**（W4-D5 讲过，这里再确认一次）：从**函数签名 + docstring 自动生成 JSON Schema**，不用你手写。

> ⚠️ **docstring 就是给模型看的说明书**。`"计算两个整数之和"` 这句写得清楚，模型才知道"遇到加法该调它"。写含糊了，模型要么不调、要么调错。

### 3.2 LLM 绑定

```python
llm = ChatOpenAI(
    model="deepseek-v4-flash",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)

llm_with_tools = llm.bind_tools(tools)   # 🎯 关键：绑了工具的 llm 才会返回 tool_calls
```

**两个细节**：

- `bind_tools` 返回的是**新对象**。必须用返回值 `llm_with_tools`，原 `llm` 是没绑工具的。
- `os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY")` 这个**双 fallback** 写法很实用——两个 key 哪个存在用哪个，换厂商不用改代码。

> 💡 这里**不需要像 W4 那样加 `parallel_tool_calls=False`**——因为有了 `ToolNode`，并行调用不再是问题。这个变化本身就是"框架解决了什么"的证据。

### 3.3 agent 节点（含防死循环收口）

```python
def agent_node(state: AgentState) -> dict:
    if len(state["messages"]) > 10:
        forced = {"role": "user", "content": "请基于以上工具结果直接给出最终答案，不要再调用工具。"}
        return {"messages": [llm_with_tools.invoke(state["messages"] + [forced])]}
    return {"messages": [llm_with_tools.invoke(state["messages"])]}
```

设计意图：消息超过 10 条就**强制收口**——追加一条用户指令，让模型基于已有工具结果直接作答。

**两个容易写错的地方**：

```python
# ⚠️ 1. forced 必须包成列表：list + dict 会 TypeError
state["messages"] + forced      # ❌ TypeError: can only concatenate list (not "dict") to list
state["messages"] + [forced]    # ✅

# ⚠️ 2. 收口指令是发给模型的，错字会改变模型行为
"请基于以上共计结果……不要在调用工具。"     # ❌ "共计"可能被引申为"再求和"
"请基于以上工具结果……不要再调用工具。"     # ✅
```

> 💡 **异常路径上的代码必须单独测。** 这段收口逻辑平时永远执行不到（消息 ≤10 条时走 `else`），只有模型真跑飞了才触发——所以它是全文件最该被验证、却最容易被忽略的一块。手动塞 12 条消息进去跑一次，确认它真的能用。

### 3.4 构图：ReAct 经典拓扑

```python
builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))      # 工具执行节点

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)   # 有 tool_calls → tools，否则 → END
builder.add_edge("tools", "agent")              # 🎯 回边！

graph = builder.compile()
```

**🔴 最后那条回边是整张图的命门。**

```fallback
  ✅ 有回边：  agent → tools → agent → END
              工具结果回到 agent，模型看到结果，生成最终答案

  ❌ 没回边：  agent → tools → END
              工具执行完就结束，没人把结果变成答案
              → 你拿到的最后一条消息是 ToolMessage(content='60')，
                而不是 "答案是 60"
```

> **类比**：回边就是**把电话转回给总机**。客户要查余额，客服（tools）查到了数字，如果不转回给客户（agent），客户只听到"系统提示：60"，而不是"您的余额是 60 元"。
>
> 💡 这条边在 D2 的世界里不存在——**D1/D2 的图都是无环的，回边是 D3 才出现的新物种**。它也是 W4-D1 那句"LCEL 无环、LangGraph 有环"第一次真正兑现的地方。

### 3.5 完整版（可直接抄）

```python
"""W5-D3: 手搭 ReAct 图 —— agent ↔ tools 循环"""
import os
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# ── 1. State：messages 用 reducer 自动累加 ──
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

# ── 2. 工具（@tool 从签名 + docstring 自动生成 schema，docstring 是给模型看的）──
@tool
def add(a: int, b: int) -> int:
    """计算两个整数之和"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """计算两个整数之积"""
    return a * b

tools = [add, multiply]

# ── 3. LLM（多厂商只换 base_url / model）──
llm = ChatOpenAI(
    model="deepseek-v4-flash",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)
llm_with_tools = llm.bind_tools(tools)   # 关键：绑了工具的 llm 才会返回 tool_calls

# ── 4. agent 节点（含防死循环：消息超 10 条强制收口）──
def agent_node(state: AgentState) -> dict:
    if len(state["messages"]) > 10:
        forced = {"role": "user", "content": "请基于以上工具结果直接给出最终答案，不要再调用工具。"}
        return {"messages": [llm_with_tools.invoke(state["messages"] + [forced])]}
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

# ── 5. 构图：ReAct 经典拓扑 ──
builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))       # 工具执行节点

builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)   # 有 tool_calls → tools，否则 END
builder.add_edge("tools", "agent")               # 🔴 回边！没有它工具跑完就结束，永远没有最终答案

graph = builder.compile()

# ── 6. 链式任务验收：需要"先 add 再 multiply"两次工具调用 ──
result = graph.invoke(
    {"messages": [("user", "帮我算一下 (12 + 8) 乘以 3")]},
    config={"recursion_limit": 25},              # 框架级保险丝
)
print("=== 完整链路（肉眼确认）===")
for m in result["messages"]:
    m.pretty_print()

# ── 7. stream 看 agent / tools 交替 ──
print("\n=== stream 看节点流转 ===")
for event in graph.stream({"messages": [("user", "3 加 5 等于几")]}, stream_mode="updates"):
    print(event)

# ── 8. prebuilt 对照验证 ──
from langgraph.prebuilt import create_react_agent

prebuilt_agent = create_react_agent(llm_with_tools, tools)
result2 = prebuilt_agent.invoke({"messages": [("user", "(12+8)*3 等于几")]})

print("=== prebuilt 版输出 ===")
for m in result2["messages"]:
    m.pretty_print()
```

### 3.6 跑 + 肉眼确认链路

```python
result = graph.invoke({"messages": [("user", "帮我算一下 (12 + 8) 乘以 3")]})
print("=== 完整链路（肉眼确认）===")
for m in result["messages"]:
    m.pretty_print()
```

> 💡 注意 `("user", "...")` 这种**元组简写**——LangGraph 会把它自动转成 `HumanMessage`。等价于 `HumanMessage(content="...")`，写起来更短。

**预期链路（必须肉眼看到这 6 步才算对）**：

```fallback
HumanMessage: 帮我算一下 (12 + 8) 乘以 3
AIMessage:    tool_calls=[add(12, 8)]      ← agent 第一次决策
ToolMessage:  20                            ← tools 执行 add 并回填
AIMessage:    tool_calls=[multiply(20, 3)]  ← agent 看到 20，再调 multiply
ToolMessage:  60                            ← tools 执行 multiply
AIMessage:    (12+8)×3 = 60                 ← agent 无 tool_calls → END
```

**中间那两步 AI 消息是重点**——模型**不是一次算完的**：

```fallback
  它先调 add → 看到结果 20 → 再调 multiply → 看到 60 → 才作答
              ↑                        ↑
         每一次"看到结果再决定"，都是一次 agent→tools→agent 的转圈
```

> 🎯 **这个"看工具结果再决定下一步"的循环，就是 ReAct 的灵魂**（ReAct = **Rea**son + **Act**）。而在你手搭的图里，它完整跑通了。

### 3.7 stream 看节点交替

```python
for event in graph.stream({"messages": [("user", "3 加 5 等于几")]}, stream_mode="updates"):
    print(event)
```

```fallback
{'agent': ...tool_calls=[add(3, 5)]...}   ← agent 决策要调工具
{'tools': ...ToolMessage(content='8')...} ← tools 执行并回填
{'agent': ...tool_calls=[]...}            ← agent 拿到 8，不再调 → 结束
```

**看第一条和第三条 agent 消息的区别**：

| | 第 1 次 agent | 第 3 次 agent |
|---|---|---|
| `finish_reason` | `'tool_calls'` | `'stop'` |
| `tool_calls` | `[add(3,5)]` | `[]` |
| 路由去向 | → tools | → END |

> ✅ **这就是 `tools_condition` 的判据在起作用**：最后一条 AI 消息带不带 `tool_calls`，带就回 tools，不带就 END。
>
> 你亲手写的 `agent_node` + 官方 `tools_condition` + `ToolNode`——**三者配合，图就自己转起来了。**

* * *

## 四、W2 `fc_loop` → 今天这张图（对照表）

| 你 W2 / W4 手写的 | LangGraph 里 | 变化 |
|---|---|---|
| `while True:` 循环 | `tools → agent` **回边** | 循环变成了图的一条边 |
| `if response.tool_calls:` | `tools_condition` 路由 | 判断变成官方路由函数 |
| `execute_tool()` + 手动 append 结果 | `ToolNode` 自动执行 + 回填 | W4 的 insufficient 坑被框架解决 |
| `MAX_ROUNDS` 防死循环 | ⚠️ **业务级收口要自己加**（另有框架级保险丝，见第五章） | 这是唯一没被替代的一块 |

**最后一行值得单独说**——框架替你做了三件事，唯独"防死循环"的业务级收口还得自己写。为什么？

> 💡 因为"什么时候算跑偏了"是**业务判断**，框架不知道你的 Agent 该跑几轮算合理。它能做的是"跑太多次就报错"（框架级），不能做的是"跑偏了就优雅收口"（业务级）。

* * *

## 五、防死循环：三层机制

### 第 1 层：框架级保险丝 `recursion_limit`

```python
graph.invoke(
    {"messages": [...]},
    config={"recursion_limit": 25}    # 默认值就是 25
)
```

**超过限制会抛 `GraphRecursionError`**——它是**硬保险丝**：

| | `recursion_limit` | `>10 条收口` |
|---|---|---|
| 定位 | 框架级，兜底 | 业务级，主动 |
| 超限行为 | **抛异常**，程序中断 | **提示模型作答**，优雅收口 |
| 默认值 | 25 次节点跳转 | 需自己写 |
| 优点 | 零成本，永远在线 | 用户体验好 |

> 🎯 **正确做法是两层一起用**：`recursion_limit` 兜住"真跑飞了"的极端情况，`>10 条收口` 处理"跑得有点久但还能救"的常见情况。**别以为有了其中一个就够了。**

### 第 2 层：业务级软收口

```python
if len(state["messages"]) > 10:
    forced = {"role": "user", "content": "请基于以上工具结果直接给出最终答案，不要再调用工具。"}
    return {"messages": [llm_with_tools.invoke(state["messages"] + [forced])]}
```

### 第 3 层：⚠️ `len(messages)` 是个"代理指标"，不精确

**用消息条数当轮次上限，是个近似**：

```fallback
  每转一圈，messages 增加几条？
    简单情况：+1（AI）+1（Tool）= +2 条
    并行调 2 个工具：+1（AI）+2（Tool）= +3 条
    并行调 3 个工具：+1（AI）+3（Tool）= +4 条

  所以 "> 10 条" 究竟等于几轮？
    → 3 轮 ~ 5 轮不等，取决于模型有没有并行调用
```

**想要精确控制轮次，加一个计数器字段**：

```python
import operator
from typing import Annotated

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]
    rounds: Annotated[int, operator.add]      # reducer 自动累加

def agent_node(state: AgentState) -> dict:
    if state.get("rounds", 0) >= 3:           # 🎯 精确的 3 轮上限
        forced = {"role": "user", "content": "请基于以上工具结果直接给出最终答案，不要再调用工具。"}
        return {"rounds": 1, "messages": [llm_with_tools.invoke(state["messages"] + [forced])]}
    return {"rounds": 1, "messages": [llm_with_tools.invoke(state["messages"])]}
```

> 💡 这就是 D2 埋的那个伏笔——**`Annotated[int, operator.add]` 让字段自动累加**，节点里 `return {"rounds": 1}` 就会被加进去，不用读旧值。
>
> 学习阶段用 `len(messages) > 10` 完全够用（能跑、能防住）；等你做 W6 的 HITL 和长会话时，再换成精确计数器。

* * *

## 六、验收：三个证据

回到开头那个问题——**"你怎么知道它做对了？"**

### 证据 1：链路完整（pretty_print 肉眼确认）

6 步一步不能少：

```fallback
Human: 帮我算一下 (12+8)×3
AI:    tool_calls=[add(12, 8)]      ← 决策1：先算括号
Tool:  20                           ← add 执行，回填
AI:    tool_calls=[multiply(20, 3)] ← 决策2：看到 20，再算乘
Tool:  60                           ← multiply 执行，回填
AI:    (12+8)×3 = 60                ← 无 tool_calls → 结束
```

**核验要点**：中间必须出现**两次 AI tool_calls**。如果只出现一次（模型一次并行调了 add 和 multiply），说明它没"看到 20 再决定"——那也是能跑的（见文末魔鬼代言人），但**不是今天要验的链式推理**。

### 证据 2：节点交替（stream）

```fallback
{'agent': ...tool_calls=[add(3, 5)]...}   ← agent 决策要调工具
{'tools': ...ToolMessage(content='8')...} ← tools 执行并回填
{'agent': ...tool_calls=[]...}            ← agent 拿到 8，不再调 → 结束
```

**核验要点**：最后一个 `agent` 的 `tool_calls` 必须是空的，`finish_reason` 必须是 `'stop'`。

### 证据 3：prebuilt 对照（行为等价）

```python
from langgraph.prebuilt import create_react_agent

prebuilt_agent = create_react_agent(llm_with_tools, tools)
result2 = prebuilt_agent.invoke({"messages": [("user", "(12+8)*3 等于几")]})

print("=== prebuilt 版输出 ===")
for m in result2["messages"]:
    m.pretty_print()
```

**对照标准**：prebuilt 版也应该走出 `add(12,8) → 20 → multiply(20,3) → 60 → 最终答案` 的同款链路。

两条链路行为一致 → **你手搭的拓扑 == 官方 `create_react_agent` 内部拓扑**。

> **类比**：官方给你一辆原厂车，你刚才用零件自己拼了一辆——现在并排开一圈，发现换挡逻辑、转向手感一模一样，**说明你拼的就是那辆原厂车**。

> 💡 **版本提示**：`create_react_agent` 是 LangGraph 的经典 prebuilt 入口。LangChain 1.0 之后另有 `langchain.agents.create_agent` 这个高层入口（内部同样基于 LangGraph）。做对照验证时用前者，链路最直观、文档最全；熟悉之后再换哪个都行。

* * *

## 七、真实 API 环境的两个"彩蛋"

跑真 API 时，`pretty_print` 里除了消息内容，还会带出 token 统计。留意这两行：

**① `reasoning_tokens: 8`**

DeepSeek 内部有**思考过程**。这是它能自己规划出"先算括号、再算乘法"这种多步策略的原因之一——它不是在猜，是真的推了一步。

**② `prompt_cache_hit_tokens: 384`**

**prompt 前缀缓存命中**——第二次调用时，前面那 384 个 token（系统提示 + 历史消息）**没重新计费**。

```fallback
  第 1 次调用： messages = [Human]                 → 全量计费
  第 2 次调用： messages = [Human, AI, Tool]       → 前 384 token 命中缓存，省钱
  第 3 次调用： messages = [Human, AI, Tool, AI...] → 命中更多
```

> 💡 **循环越长，缓存省得越多。** 这是 Agent 真实部署时的重要成本优化点——也解释了为什么"消息自动累加"（`add_messages`）不只是功能需求，**它还是省钱的前提**：消息能累积，前缀才能复用。

* * *

## 八、踩坑记

### 坑 1：state 的 messages 没加 reducer ⚠️

- **现象**：工具结果回填后，下一轮模型看不到；或消息越来越少而不是越来越多
- **原因**：字段默认覆盖
- **解法**：`messages: Annotated[list, add_messages]`

### 坑 2：`bind_tools` 忘了调 ⚠️

- **现象**：模型永远不返回 `tool_calls`，直接瞎答
- **解法**：`llm_with_tools = llm.bind_tools(tools)`，**并且用返回值**

### 坑 3：忘了 `add_edge("tools", "agent")` 回边 ⚠️

- **现象**：工具执行完直接结束，最后一条消息是 `ToolMessage(content='60')` 而不是答案
- **解法**：补上回边——**工具结果要回喂给 agent 才能变成答案**

### 坑 4：`tools_condition` 挂错节点 ⚠️

- **现象**：路由判断错位，图行为诡异
- **原因**：条件边必须挂在 **agent** 后面，不是 tools 后面
- **解法**：`add_conditional_edges("agent", tools_condition)`

### 坑 5：工具 docstring 没写 / 写得太差 ⚠️

- **现象**：模型不调工具，或调错工具
- **原因**：docstring 就是工具描述，是给模型看的
- **解法**：写清"干什么 + 什么时候用"

### 坑 6：收口指令忘了包成列表 ⚠️

```python
state["messages"] + forced      # ❌ TypeError: can only concatenate list (not "dict") to list
state["messages"] + [forced]    # ✅
```

- **现象**：`TypeError`，只在触发收口分支时出现
- **原因**：`state["messages"]` 是 list，`forced` 是 dict，Python 不允许 list + dict
- **解法**：包成列表再拼接
- ⚠️ **难查的原因**：这段只在消息超阈值时执行，平时永远走不到——**异常路径必须单独测**

### 坑 7：把防死循环全押在 `recursion_limit` 上 ⚠️

- **现象**：跑飞时抛 `GraphRecursionError`，用户看到报错而不是答案
- **原因**：`recursion_limit` 是硬保险丝，抛异常而非优雅收口
- **解法**：框架级 + 业务级两层一起用

### 坑 8：写进 prompt 的文字有错别字 ⚠️

- **现象**：塞了强制收口指令，模型还是继续调工具
- **原因**：**prompt 会被逐字执行**，错字会改变语义（"共计结果"可能被理解成"再求和"）
- **解法**：**写进 prompt 的文字，按"会被逐字执行"的标准检查**

### 坑 9：把 `len(messages)` 当成精确轮次 ⚠️

- **现象**：设了 `> 10` 却把控不准实际轮数
- **原因**：并行调用时每轮增加 2~4 条不等
- **解法**：需要精确控制就加 `rounds: Annotated[int, operator.add]` 计数器

* * *

## 九、速查卡片（复习直接看这）

```python
# ===== ReAct 最小图 =====
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]     # 🎯 追加，不是覆盖

@tool
def add(a: int, b: int) -> int:
    """计算两个整数之和"""                       # 🎯 docstring = 给模型的说明书
    return a + b

llm_with_tools = ChatOpenAI(...).bind_tools(tools)   # 🎯 用返回值

def agent_node(state: AgentState) -> dict:
    if len(state["messages"]) > 10:             # 业务级软收口
        forced = {"role": "user", "content": "请基于以上工具结果直接给出最终答案，不要再调用工具。"}
        return {"messages": [llm_with_tools.invoke(state["messages"] + [forced])]}  # ⚠️ [forced]
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

b = StateGraph(AgentState)
b.add_node("agent", agent_node)
b.add_node("tools", ToolNode(tools))
b.add_edge(START, "agent")
b.add_conditional_edges("agent", tools_condition)    # 🎯 挂 agent 后面
b.add_edge("tools", "agent")                         # 🔴 回边，缺了没有最终答案
graph = b.compile()

# 跑 + 确认链路
result = graph.invoke({"messages": [("user", "帮我算一下 (12 + 8) 乘以 3")]},
                      config={"recursion_limit": 25})    # 框架级保险丝
for m in result["messages"]:
    m.pretty_print()

# 对照验证
from langgraph.prebuilt import create_react_agent
prebuilt = create_react_agent(llm_with_tools, tools)
```

* * *

## 十、一句话总结

**ReAct Agent 在 LangGraph 里就是一张带环的图：`messages` 用 `add_messages` 让消息自动累加；`agent` 节点是绑了工具的 LLM，返回带 `tool_calls` 的 AI 消息；`tools_condition` 检查最后一条消息有没有 `tool_calls`，有就路由到 `ToolNode`，没有就结束；`ToolNode` 自动执行工具并回填 `ToolMessage`，再经回边回到 `agent` 重新决策——循环直到模型不再调工具。**

**对照你 W2 的 `fc_loop`**：

```fallback
  while True              →  tools → agent 回边
  if tool_calls           →  tools_condition
  execute + append        →  ToolNode（还顺手治好了 W4 的 insufficient 坑）
  MAX_ROUNDS              →  框架不包业务级收口，得自己加
```

**今天是转折日**——之前你"用代码写循环"，今天起你"**画循环**"。这张 `agent ↔ tools` 带环的图，就是 W5 当周产出②（ReAct 最小图），也是你后面所有 Agent 的**原型骨架**。

**下一篇：D4 Stream 模式深入**——`updates` / `values` / `messages` 三种 `stream_mode` 的差异与选型，以及 `astream_events` 的 token 级输出。今天你用 `stream` 看节点流转只是热身，D4 要把它用到**打字机效果**上。

* * *

### 🔴 魔鬼代言人（跑之前给你打预防针）

**① DeepSeek 可能一次并行返回多个 tool_calls**

比如同时调 `add` 和 `multiply`。**别慌**——`ToolNode` 会全部执行并逐条回填，这正是 W4 想要而没得到的行为。

但如果它真并行调了，链式任务就变一步了，**链路会短一点**：

```fallback
  预期（串行）：  AI(add) → Tool(20) → AI(multiply) → Tool(60) → AI(答案)   5 步
  实际（并行）：  AI(add, multiply) → Tool(20) → Tool(60) → AI(答案)        4 步
```

**这不算错**，只是模型选择了不同的执行策略。但要注意：**并行时 `multiply` 的参数从哪来？** 如果模型在没拿到 `add` 结果的情况下就填了 `multiply(20, 3)`，那它是**猜的**——这次猜对了，不代表永远对。

> ⚠️ 真正的链式依赖任务（第二步依赖第一步的结果），**并行调用是危险的**。W4 你用 `parallel_tool_calls=False` 强制串行，是有道理的。如果发现并行导致结果不对，可以在 `bind_tools` 里把它加回来。

**② 第一次跑会烧几次 API 调用**

真 API 实战的正常成本。但如果模型**反复调工具不收敛**（超过 10 条会强制收口兜底），别急着加钱——**先调 prompt**（把工具 docstring 写清楚、把任务描述写具体）。

**③ 警惕"跑通一次就以为会了"**

把题目换成 **"帮我算 7 的阶乘再除以 2"** 再跑一次——这是个**需要多轮决策**的任务：

```fallback
  7! = 7×6×5×4×3×2×1 = 5040，再 ÷ 2 = 2520
  模型要么连续调 6 次 multiply（考验回边），
  要么一次并行调多个（考验 ToolNode 的多回填）
```

**观察 ToolNode 怎么逐条回填**——那是 W4 踩过的 `insufficient tool messages` 坑**被框架自动解决的最佳演示**，值得亲眼确认一次。

* * *
