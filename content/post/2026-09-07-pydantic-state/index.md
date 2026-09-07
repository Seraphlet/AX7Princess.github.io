---
description: ""
title: "State 升级：从便利贴到安检门"
draft: false
date: "2026-09-07T12:02:29+08:00"
slug: "pydantic-state"
categories:
 - LangGraph
 - Pydantic
tags:
 - null
image: ""
---


# W5-D5 · State 升级：从便利贴到安检门

> **关键词**：三层脉络 / TypedDict vs Pydantic / `Annotated` / reducer / `MessagesState`
> **前置**：W5-D1（State 四件套）、D2（条件边）、D3（ReAct 图）、D4（stream）；**搭配**：《Pydantic 校验完全指南》（库用法）
> **涉及文件**：`W5/lg_d5_pydantic_state.py`
> **收尾预告**：D6 MemoryManager 项目——summary / messages / 计数器多字段协同的复杂 state

* * *

## 零、开场：站在 W5 中间，先回一次头

学 D5 之前，值得花一章回答一个"回头看"的问题——**为什么你一路从手写循环走到了 LangGraph？** 因为你正站在 W5 中间，回头把 W2→W5 的脉络打通，才是"从会跑代码到能讲架构"的关键一跃。这一章先把脉络理清，D5 的正文就好理解了。

**核心事实**：无论你手写还是用框架，**心里永远都是同一张图**。

```fallback
用户输入
  ↓
LLM 决策 ──有 tool_calls──→ 执行工具 ──→ 结果回填 ──┐
  │                                                  │
  └──────────── 无 tool_calls → 输出最终答案 ←───────┘
```

这就是 **ReAct 循环**：决策 → 看要不要工具 → 执行 → 回填 → 再决策，直到模型不再要工具。

W2 你用 `while True` 手写它；W4 你用 LangChain 组件 + 手写循环；W5 你把它画成了一张图。**同一件事，三种表达。**

* * *

## 一、同一个 ReAct 循环，三层表达

### 1.1 第 0 层：裸 API——只有"一轮对话"，没有循环

```python
resp = client.chat.completions.create(model=..., messages=messages)
answer = resp.choices[0].message.content   # 一次问答，结束
```

模型只会"说话"，不会"干活"。你想让它用工具？**得自己造循环。**

### 1.2 第 1 层：手写 fc_loop（W2 的你）

```python
while True:                                    # ← 循环 = 图里"再决策"的部分
    resp = client.chat.completions.create(
        model=..., messages=messages, tools=tools)
    msg = resp.choices[0].message
    messages.append(msg)

    if not msg.tool_calls:                     # ← 路由：if tool_calls
        break                                  #    没有 → 结束
    for tc in msg.tool_calls:                  # ← 执行工具
        result = run_tool(tc.function.name, json.loads(tc.function.arguments))
        messages.append({                      # ← 手动回填
            "role": "tool", "tool_call_id": tc.id, "content": str(result)})
```

**这一层解决的核心问题**：模型说"我要调 add(12,8)"，得有人真正执行它、把结果喂回去。代价是——路由判断、结果回填、防死循环（`MAX_ROUNDS`）**全是你的 `if` 和 `while`**。

### 1.3 第 2 层：LangChain 组件（W4 的 lc_agent_demo）

```python
while True:                                    # ← ⚠️ 循环还得自己写！
    try:
        resp = llm_with_tools.invoke(messages) # ← 统一了厂商/schema/解析
    except Exception:
        resp = llm_without_tools.invoke(messages)  # ← 降级自愈
    messages.append(resp)
    if not resp.tool_calls:
        break
    for tc in resp.tool_calls:
        out = run_tool(tc["name"], tc["args"])
        messages.append(ToolMessage(content=str(out), tool_call_id=tc["id"]))
```

**这一层解决的问题**：`bind_tools` 自动生成 schema、`@tool` 从签名 + docstring 生成描述、`AIMessage`/`ToolMessage` 标准类型、厂商差异被抹平——**原子操作的样板代码没了**。但注意那个 `while True` 还在——**LangChain 不管控制流。循环、状态、跳转，还是你手写。**

### 1.4 第 3 层：LangGraph（W5-D3 的你）

```python
builder.add_node("agent", agent_node)                    # LLM 决策
builder.add_node("tools", ToolNode(tools))               # 执行 + 回填（自动）
builder.add_conditional_edges("agent", tools_condition)  # ← if tool_calls 变成路由
builder.add_edge("tools", "agent")                       # ← while True 变成回边
graph = builder.compile()
graph.invoke(...)                                        # 框架替你跑循环
```

**这一层解决的问题**：把控制流本身——循环、判断、状态累加——从"你写的代码"变成"你描述的结构（图）"。于是你获得了手写时代要自己造的一切：逐跳可见（D4 stream）、状态自动累加（D5 的 reducer，本篇主角）、之后还有 checkpoint 断点续跑 / 人审 / 子图 / 多 Agent。

### 1.5 三层对照表（背这张）

| 你 W2 手写的 | LangGraph 里的 |
|---|---|
| `while True:` 循环 | `tools → agent` 回边 |
| `if response.tool_calls:` 判断 | `tools_condition` 条件路由 |
| `execute_tool()` + 手动 append | `ToolNode`（自动执行 + 回填） |
| 手写 messages 列表 | `Annotated[list, add_messages]` reducer |
| `MAX_ROUNDS` 防死循环 | ⚠️ 框架不包业务级收口，节点里自己加 |

* * *

## 二、每次升级到底在解决什么

> **每一层都在把"你手写的代码"变成"框架替你跑的东西"——从命令式（教计算机每一步怎么做）走向声明式（描述结构是什么）。**

```fallback
  手写 fc_loop  解决：模型要调工具，谁来执行、谁来回填 → 你写 while + if
  LangChain     解决：调 LLM 的样板（厂商 / schema / 消息解析）→ 但循环还得自己写
  LangGraph     解决：控制流不可见、难扩展
                → 循环变回边、判断变路由、状态变 reducer、工具变 ToolNode
```

**代价**：每多一层，你少写一类代码，但多学一层抽象。为什么从手写走到 LangGraph，答案就是上面三行。

> 💡 多说一句：手写让你懂原理，框架让你懂抽象——**两条腿缺一不可**。你 W1–W3 亲手踩过每个坑，才看得懂 W4/W5 的每一层封装在替你省什么。

* * *

## 三、单独说"报错自愈"这条线

这条线**框架至今没替你完全解决**——它是工程韧性，得自己写，但写法变了：

- **W4 手写时代**：`try/except` 包整段循环——工具抛异常、API 拒绝，全都要自己兜底。你当时的三层容错：① 工具异常 → 错误信息当结果回填；② API 拒绝 → 降级不带工具；③ 兜底文案。
- **W5 图时代**：`try/except` 缩进到**节点函数内部**——`agent_node` 里包 LLM 调用，工具执行包在 `ToolNode` 内部。图结构不变，容错下沉到节点。

**核心原则没变**：**工具是增强项不是可用性依赖——工具挂了，对话也得继续。** 这条你 W4 就懂了，现在只是换了个地方写。

* * *

## 四、D5 正题之一：给 state 装上"运行时安检"

### 4.1 一张便利贴 vs 一道安检门

脉络讲完，进入 D5 正题。D1–D4 你的 state 一直是 **TypedDict**——它有个你没注意的软肋：**只在静态检查时有类型提示，运行时形同虚设**。

```python
from typing import TypedDict
from pydantic import BaseModel, Field

# —— TypedDict：只在写代码时给你提示，运行时完全不检查 ——
class TD(TypedDict):
    count: int

td: TD = {"count": "不是整数"}   # 运行时零报错！字符串被当作 int 用下去

# —— Pydantic：运行时真校验 ——
class PD(BaseModel):
    count: int = Field(ge=0, description="必须是非负整数")

PD(count="不是整数")   # ❌ ValidationError：类型错误被拦下
PD(count=-5)           # ❌ ValidationError：ge=0 拦住负数
PD(count=5).model_dump()   # ✅ {'count': 5}
```

| | TypedDict | Pydantic BaseModel |
|---|---|---|
| 类型提示 | ✅ | ✅ |
| **运行时校验** | ❌ 形同虚设 | ✅ 自动校验 |
| 值域约束（ge 等） | ❌ | ✅ `Field` |
| 序列化成 dict | 手动 | ✅ `model_dump()` |

> **类比**：TypedDict 像**便利贴**——写着"这里应该是整数"，但没人检查，你塞个字符串它也收。Pydantic 像**安检门**——不是整数？拦下报错；是负数但规定不能为负？也拦下。**早失败，早发现 bug。**

> 💡 想深挖 Pydantic 的字段、校验器、ConfigDict 怎么用，直接翻那篇《Pydantic 校验完全指南》——本篇只讲它在 LangGraph 里当 state 的角色。

### 4.2 为什么 D5 才换？因为 state 开始复杂了

D1–D4 你的 state 基本只有 `messages` 一个字段。但 D6 的 MemoryManager 图要有 **summary + messages + 计数器多个字段协同**——字段一多，"运行时没人检查"的便利贴就危险了：一个字段拼错类型，可能跑几轮才炸，而且炸得莫名其妙。**Pydantic 把错误提前到构造那一刻。**

* * *

## 五、D5 正题之二：Annotated + reducer——字段到底怎么合并

### 5.1 你 D3 就会写了，今天搞懂"凭什么"

你 D3 写过 `messages: Annotated[list, add_messages]`——当时"会用"，今天要搞懂**它凭什么能追加而不是覆盖**。

```python
from typing import Annotated
from pydantic import BaseModel

class RichState(BaseModel):
    messages: Annotated[list, add_messages]   # 字段类型 T + 元数据（reducer）
    #         └───────┬───────┘  └─────┬─────┘
    #           字段类型 list      第二个参数：reducer 函数
```

`Annotated` 是 Python 3.9+ 标准库：给类型 `T` **附加元数据**。LangGraph 读第二个参数——reducer 函数——来决定"多个节点更新同一字段时怎么合并"。

### 5.2 🎯 LangGraph 的 state 更新规则（背下来）

```fallback
  节点返回的 dict 里，每个 key 触发对应字段的一次更新：

  ① 有 Annotated 的字段（带 reducer）→ 走 reducer 合并
     messages: Annotated[list, add_messages]
     → 新消息被"追加"进现有列表

  ② 没有 Annotated 的字段 → 整体覆盖（last write wins）
     counter: int
     → 谁最后写，就整个盖掉前面的
```

**用一个最小实验验证**——这是全文最重要的 10 行：

```python
from typing import Annotated
from pydantic import BaseModel, Field
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class DemoState(BaseModel):
    messages: Annotated[list, add_messages] = Field(default_factory=list)  # 有 reducer：追加
    counter: int = 0                                       # 无 reducer：覆盖

def n1(state: DemoState) -> dict:
    return {"messages": [{"role": "user", "content": "第一条"}], "counter": 1}

def n2(state: DemoState) -> dict:
    return {"messages": [{"role": "assistant", "content": "第二条"}], "counter": 2}

b = StateGraph(DemoState)
b.add_node("n1", n1); b.add_node("n2", n2)
b.add_edge(START, "n1"); b.add_edge("n1", "n2"); b.add_edge("n2", END)
g = b.compile()

final = g.invoke({})
print(len(final["messages"]))   # 2  →  reducer：n1、n2 的两条都留下了（追加）
print(final["counter"])         # 2  →  无 reducer：n2 覆盖了 n1 的 1
```

> **类比**：`messages` 是**聊天记录本**（只许往下加页）；`counter` 是**便利贴**（新写的盖掉旧的）。同一个节点返回 dict，两个字段走了完全不同的合并路径。

### 5.3 自定义 reducer：累加 vs 覆盖的终极对照

reducer 不必是框架给的。规则只有一个：**接收 `(当前值, 新值)`，返回合并结果**。

```python
# 计数器：每个节点返回 {"visits": 1}，就会在现值上 +1
def increment(current: int, new: int) -> int:
    return current + new

# 日志：字符串列表拼接
def add_log(current: list[str], new: list[str]) -> list[str]:
    return current + new
```

**用一张小图同时验证三种字段行为**——这是 D5 收尾的"终极对照"，一个文件跑完，三行输出把概念钉死：

```python
from typing import Annotated
from pydantic import BaseModel, Field
from langgraph.graph import StateGraph, START, END

class CounterState(BaseModel):
    visits: Annotated[int, increment] = 0                     # 有 reducer → 累加
    log: Annotated[list[str], add_log] = Field(default_factory=list)   # 有 reducer → 拼接
    plain: int = 0                                            # 无 reducer → 覆盖（对照组）

def step1(state: CounterState) -> dict:
    return {"visits": 1, "log": ["step1 执行了"], "plain": 1}

def step2(state: CounterState) -> dict:
    return {"visits": 1, "log": ["step2 执行了"], "plain": 2}

b = StateGraph(CounterState)
b.add_node("step1", step1)
b.add_node("step2", step2)
b.add_edge(START, "step1"); b.add_edge("step1", "step2"); b.add_edge("step2", END)
g = b.compile()

final = g.invoke({})
print("visits:", final["visits"])   # 2    ← step1 +1，step2 再 +1（increment 累加）
print("log:", final["log"])          # ['step1 执行了', 'step2 执行了']（add_log 拼接）
print("plain:", final["plain"])      # 2    ← 无 reducer，step2 覆盖了 step1 的 1
```

**看这三个输出，一整章的知识浓缩成一行对照**：

```fallback
  visits: 2     increment（自定义 reducer）→ 每个节点写 1 都累加 → 计步器
  log: 两条      add_log（自定义 reducer）  → 每次都追加 → 聊天记录本
  plain: 2      普通字段（无 reducer）      → 后写覆盖 → 便利贴
```

> **类比**：`increment` 是**计步器**（走一步 +1，累计）；普通字段是**便利贴**（新写的盖旧）。同一个字段，一个累加、一个覆盖——**想要哪种行为，取决于你有没有给字段加 `Annotated`**。

**为什么你需要自定义 reducer**：见第八章那个真实排查——普通字段没法"每次节点路过都计数"，计数器必须用 reducer。

### 5.4 MessagesState：官方把你 D3 那行封装了

你 D3 手写的 `messages: Annotated[list[AnyMessage], add_messages]`，官方封装成了一个内置 state：

```python
from langgraph.graph import MessagesState

class MyState(MessagesState):    # 继承扩展：messages 字段免费送
    summary: str = ""            # 再自己加业务字段
    memory_path: str = ""
```

**关键**：知道它底层就是你手写的那行——这叫"会用 + 懂原理"。之后看到 `class X(MessagesState)` 时你清楚它内部发生了什么。

> ⚠️ 顺带一个 pydantic + LangGraph 的坑：**Pydantic state 的可变默认值必须用 `Field(default_factory=list)`**（`messages: Annotated[list, add_messages] = []` 会直接报错，Pydantic v2 禁止可变默认值）。上面示例都按这个写了。

* * *

## 六、完整可运行代码（`W5/lg_d5_pydantic_state.py`）

把 Pydantic state、普通字段 vs reducer、ReAct 循环合成一张图：

```python
"""W5-D5: Pydantic state + reducer 机制——为 D6 MemoryManager 打底"""
import os
from typing import Annotated
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

# ── 1. Pydantic state：messages 追加 / summary 覆盖 / 计数 ──
class RichState(BaseModel):
    messages: Annotated[list, add_messages] = Field(default_factory=list)
    summary: str = Field(default="", description="对话摘要")     # 普通字段：后写覆盖
    tool_use_count: int = 0                                     # 普通字段：见第八章为何是 0

# ── 2. 工具 + LLM ──
@tool
def get_weather(city: str) -> str:
    """查询城市天气(模拟)"""
    return f"{city}:晴,25°C"

llm = ChatOpenAI(
    model="deepseek-v4-flash",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)

# ── 3. 节点：Pydantic state 支持属性访问 state.messages ──
def agent_node(state: RichState) -> dict:
    return {"messages": [llm.bind_tools([get_weather]).invoke(state.messages)]}

def tag_node(state: RichState) -> dict:
    """收尾节点：普通字段每次覆盖式写入"""
    return {"summary": "本轮对话结束", "tool_use_count": state.tool_use_count + 1}

# ── 4. 构图：agent 分叉只能走条件边，END 写进 path_map ──
builder = StateGraph(RichState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode([get_weather]))
builder.add_node("tag", tag_node)

builder.add_edge(START, "agent")
builder.add_edge("tools", "agent")       # 回边：工具结果喂回 agent
builder.add_edge("tag", END)

# agent 之后是"二选一"：有 tool_calls → tools；没有 → 收尾 tag
builder.add_conditional_edges("agent", tools_condition, {
    "tools": "tools",   # 有 tool_calls → 工具节点
    END: "tag",         # 无 tool_calls → 收尾节点（再让 tag 连 END）
})
graph = builder.compile()

# ── 5. 跑通验证 ──
result = graph.invoke({"messages": [("user", "北京天气怎么样")]})
print("=== 完整链路 ===")
for m in result["messages"]:
    m.pretty_print()
print("=== state 最终状态 ===")
print("summary:", result.get("summary"))
print("tool_use_count:", result.get("tool_use_count"))
```

**为什么 agent 不能"既挂普通 END 边、又挂条件边"**——`agent` 节点后面是一个**二选一分叉**：有 `tool_calls` 去 tools、没有就结束。这个分叉**只能由条件边决定**。如果你再加一条 `add_edge("agent", END)`，等于同时给了 agent 两条路（一条无条件去 END，一条条件去 tools），构图时就冲突了。

> 🎯 **规则**：有分叉的节点，只用 `add_conditional_edges`；想让它"没工具时走 A 节点再收尾"，就把 A 写进 path_map 的 `END` 位上（`{END: "tag"}`），再给 A 连一条普通边到 END。

* * *

## 七、Pydantic state 避坑自查

| 坑 | 表现 | 解决 |
|---|---|---|
| `messages: list = []` 可变默认 | Pydantic v2 直接报错 | `Field(default_factory=list)` |
| agent 既挂普通 END 边又挂条件边 | 构图冲突 / 行为异常 | 分叉节点只用 `add_conditional_edges`，END 写进 path_map |
| state 里放消息但类型写死 `list[dict]` | 序列化 / 校验报错 | 用 `list[Any]` 或直接继承 `MessagesState` |
| invoke 传入 dict 与 schema 不匹配 | `ValidationError` | 必填字段都传；有默认值的可省略 |
| 嵌套 + reducer 混用 | 内层 dict 被整体覆盖 | 先扁平化（LangGraph 官方建议扁平优先） |

> 💡 最后一个"扁平化"值得展开：LangGraph 官方建议 **state 保持扁平**——别在 state 里套深 dict。reducer 只作用在顶层字段上，嵌套 dict 更新时会整体覆盖，很容易出"我改了内层一个 key，结果整层被冲掉"的问题。**需要复杂结构 → 用多个平铺字段 + Pydantic 组合。**

* * *

## 八、一次真实排查：`tool_use_count` 为什么是 0

### 8.1 现象

跑完上面那张图，`tool_use_count` 打印出来是 **0**——明明 `tag_node` 里有 `+1`？

### 8.2 第一反应（错误的）

"是不是普通字段被覆盖了？"——**先别急着下结论。** 覆盖的意思是"多个节点竞争写同一字段，后写的赢"。但你的图里只有 `tag_node` 写 `tool_use_count`，agent / tools 只写 messages——**根本没有"竞争"，哪来的覆盖？**

### 8.3 让代码自己说话：加一行 print

```python
def tag_node(state: RichState) -> dict:
    print(f"[tag] 执行了！进入时 tool_use_count = {state.tool_use_count}")
    ...
```

跑出来如果是 `[tag] 执行了…`，说明 tag 跑了、也返回了 1——那问题在别处；如果**根本没打印**，说明 **tag 压根没被执行**。

真相往往是后者——**tag 是"孤岛"**：条件边的 `END` 位直接指向了 `END`（`{END: END}`），无工具调用时图直接就结束了，`tag_node` 从未被触发，`tool_use_count` 从头到尾停在初始值 0。修正：让条件边的 `END` 位指向 `"tag"`，再 `add_edge("tag", END)`（即第六章的正确写法）。重跑 → tag 执行、返回 1、最终 state 是 1。

### 8.4 这个排查教会你什么

| 环节 | 关键动作 |
|---|---|
| 现象 | 值是 0 |
| 别急着归因 | "覆盖"要有"多个节点竞争"为前提——先确认事实 |
| 验证手段 | **加 print，让代码自己说话**——看节点到底跑没跑 |
| 定位 | 不是覆盖问题，是 tag 从没执行（图里是孤岛） |
| 根治 | 把 END 位改成 `{END: "tag"}` + 补 `add_edge("tag", END)` |

> 🎯 **调试的第一原则：不猜，让代码自己说话。** 先确认"它到底跑没跑"，再谈"为什么结果不对"——多数"值不对"其实是"那段代码根本没执行"。

### 8.5 顺着这个坑，把"计数器"想明白

回到概念：**`tag_node` 只在图结束时跑一次**，所以普通字段 `tool_use_count = state.tool_use_count + 1` 够用。

但如果你想要的是 **"每次 agent / tools 跑过都计数"**（数一数这轮对话调了几次工具）呢？普通字段做不到——agent / tools 的返回 dict 里没有这个键，不触发更新。**这才是 reducer 的用武之地**：

```python
def increment(current: int, new: int) -> int:
    return current + new

class RichState(BaseModel):
    messages: Annotated[list, add_messages] = Field(default_factory=list)
    tool_use_count: Annotated[int, increment] = 0   # ← reducer：谁写都累加

# 然后让每个节点都返回 {"tool_use_count": 1}：
#   agent 决策 1 次 → +1
#   每次工具执行 → +1  → tool_use_count 变成"真实调用次数"
```

```fallback
  普通 int 字段：后写覆盖      —— 适合"只写一次的最终值"（summary）
  Annotated[int, increment]：每写一次累加 —— 适合"路过就计数"（调用次数、步数）
```

* * *

## 九、速查卡片（复习直接看这）

```python
# ===== state 两种写法 =====
class TD(TypedDict):          # 运行时零校验（只给静态提示）
    count: int

class PD(BaseModel):          # 运行时真校验 + Field 约束 + model_dump
    count: int = Field(ge=0)

# ===== 字段合并规则 =====
# 有 Annotated（带 reducer）→ 合并（追加 / 累加）
# 无 Annotated → 后写覆盖（last write wins）

# ===== 三种 reducer =====
messages: Annotated[list, add_messages]        # 官方：消息追加
visits:   Annotated[int, increment]            # 自定义：累加
def increment(current: int, new: int) -> int:
    return current + new

# ===== 官方预置 =====
from langgraph.graph import MessagesState
class MyState(MessagesState):      # 底层 = messages: Annotated[list, add_messages]
    summary: str = ""

# ===== Pydantic state 三规则 =====
# 1. 可变默认必须 Field(default_factory=list)
# 2. 分叉节点只用 add_conditional_edges，收尾走 {END: "节点名"} 再连 END
# 3. state 保持扁平；嵌套 + reducer 会整体覆盖

# ===== 调试心法 =====
# 值不对 → 先确认代码跑没跑（加 print），再谈为什么不对
# 计数器要"路过就 +1" → 用 reducer；只写一次 → 普通字段
```

* * *

## 十、一句话总结

**这一篇你做了两件事：往前看——同一个 ReAct 循环，从手写 `while True` 到 LangChain 组件再到 LangGraph 图，每一层都在把"你手写的控制流"变成"你描述的结构"（循环变回边、判断变路由、状态变 reducer）；往后走——把 state 从"便利贴"（TypedDict，运行时零校验）升级成"安检门"（Pydantic，入口即校验），并真正搞懂了 `Annotated` 的第二个参数凭什么决定"追加还是覆盖"。**

三层脉络一句话版：

```fallback
  手写 fc_loop   解决"谁来执行工具"      —— while + if + 手动回填
  LangChain      解决"调 LLM 的样板"     —— 但循环还得自己写
  LangGraph      解决"控制流不可见"      —— 循环 → 回边，state → reducer
```

D5 升级一句话版：

```fallback
  TypedDict：便利贴 —— 写着"这是整数"，没人检查
  Pydantic：安检门 —— 不是整数就拦下，早失败早发现
  普通字段：后写覆盖（last write wins）
  Annotated + reducer：合并（追加 / 累加）—— 计数器用它
```

**下一篇：D6 MemoryManager 项目**——summary、messages、计数器多字段协同的复杂 state 正式登场。没有今天的"Pydantic 安检 + reducer 合并"底子，明天你会被多字段状态管理绕晕。

* * *

### 🔴 魔鬼代言人（跑完自问三个问题）

① **DemoState 实验为什么 `messages` 长度是 2、`counter` 是 2？**

两个节点都写了两个字段，但 `messages` 走 reducer 追加（两条都留下），`counter` 走覆盖（n2 盖掉 n1）。**同一个返回 dict，两个字段两条合并路径——这就是 `Annotated` 的意义。**

② **`tag_node` 只在最后跑一次，`tool_use_count = state.tool_use_count + 1` 为什么没问题？**

因为"覆盖"只在**多个节点竞争**时有意义。你的图里只有 tag 写它、且只写一次——没有竞争，普通字段足够。**要"路过就计数"才需要 reducer。**

③ **如果我把 `tool_use_count` 改成 `Annotated[int, increment]`，行为会有什么不同？**

它会变成"每次任何节点返回 `{"tool_use_count": 1}` 都累加"。数工具调用次数时，这就是你要的——**普通字段给"最终值"，reducer 给"累计值"**，想清楚你的字段属于哪种，再决定要不要加 `Annotated`。

* * *
