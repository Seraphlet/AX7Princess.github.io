---
description: ""
title: "LangChain × LangGraph 记忆恢复：从模型调用到 Agent Runtime"
draft: false
date: "2026-10-18T12:14:12+08:00"
slug: "LangChain-LangGraph-review"
categories:
 - Review
 - Agent
tags:
 - 
image: ""
---

# LangChain × LangGraph 记忆恢复：从模型调用到 Agent Runtime

> **定位**：个人学习复盘笔记  
> **范围**：W4 LangChain 基础 + W5–W7 LangGraph 与 Agent 工程  
> **目标**：隔一段时间没写 Agent 后，重新恢复代码调用能力、数据流和运行机制。
>
> 本文不是 API 大全，也不是重新学一遍框架。重点是：**看到需求，知道用什么；开始写代码，知道放在哪里；调用结束，知道返回什么、怎么处理。**

---

## 一、先恢复整个系统：LangChain 和 LangGraph 各自负责什么

我以前从 W1 到 W3 手写过模型调用、工具循环和记忆管理。W4 学 LangChain，W5 开始学 LangGraph。

当时学到的东西并没有消失，只是分散在不同的笔记里。

先把它们重新放回同一个系统：

```text
用户输入
   │
   ▼
┌─────────────────────────────┐
│ LangChain                   │
│                             │
│ Prompt → Model → Parser     │
│            │                │
│            └── Tool Calls   │
└─────────────────────────────┘
   │
   ▼
┌─────────────────────────────┐
│ LangGraph                   │
│                             │
│ State → Node → Edge         │
│           │                 │
│           └── State Update  │
│                             │
│ Runtime 负责：              │
│ · 状态合并                  │
│ · 流程调度                  │
│ · 循环与分支                │
│ · 持久化与恢复              │
│ · 人工中断                  │
│ · 并行执行                  │
└─────────────────────────────┘
   │
   ▼
最终结果 / 流式事件 / 状态快照
```

可以先用两句话恢复：

**LangChain 负责把模型、提示词、工具和数据处理组件连接起来。**

**LangGraph 负责组织这些组件在一个有状态、可循环、可恢复的流程中运行。**

但要注意：LangGraph 的 Node 不一定调用 LLM。它也可以是普通 Python 函数、数据库查询、规则判断或工具执行。

LangGraph 是执行框架，不是模型本身。

---

## 二、最容易忘记的地方：输入和输出到底是什么

这是我重新接入真实 LLM 时遇到的第一个问题。

我知道 `invoke()`，但忘记了返回值到底是字典、字符串，还是对象。

### 2.1 先记住一个原则

**`invoke()` 是统一的调用接口，不代表统一的返回类型。**

返回什么，取决于调用的是哪个 Runnable。

| 调用 | 典型输入 | 典型输出 | 怎么取值 |
|---|---|---|---|
| `llm.invoke(...)` | 字符串或消息列表 | `AIMessage` | `result.content` |
| `prompt.invoke(...)` | 变量字典 | `ChatPromptValue` | `.to_messages()` |
| `parser.invoke(...)` | 消息或文本 | 解析后的数据 | 看解析器类型 |
| `chain.invoke(...)` | 链入口需要的数据 | 最后一个组件的输出 | 看最后一环 |
| `graph.invoke(...)` | Graph 输入 State | 最终 State，通常是字典 | `result["messages"]` |
| `retriever.invoke(...)` | 查询字符串 | `list[Document]` | `doc.page_content` |

我以前习惯用字典取值：

```python
result["content"]
```

但 LangChain 经常返回的是对象：

```python
result.content
```

这两种写法不能随意替换。

### 2.2 Python 的三种取值方式

**第一种：字典取值**

```python
data = {"name": "Agent", "count": 3}

print(data["name"])
```

因为 `data` 是 `dict`。

**第二种：对象属性**

```python
response = llm.invoke("你好")

print(response.content)
print(response.tool_calls)
```

因为 `response` 通常是 `AIMessage` 对象。

**第三种：序列解包**

```python
item = ("hello", {"node": "agent"})

chunk, metadata = item
```

因为右边是两个元素组成的元组。

所以遇到陌生返回值，先不要猜。

```python
print(type(result))
print(result)
```

如果是字典：

```python
print(result.keys())
```

如果是消息对象：

```python
print(result.content)
```

如果是列表：

```python
print(type(result[0]))
```

**先确定类型，再决定怎么取值。**

---

## 三、LangChain：如何创建模型、调用模型、处理输出

### 3.1 创建模型

以 OpenAI 兼容接口为例：

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

llm = ChatOpenAI(
    model=os.environ["LLM_MODEL"],
    api_key=os.environ["LLM_API_KEY"],
    base_url=os.environ["LLM_BASE_URL"],
    temperature=0,
)
```

这里最重要的是：

```python
llm = ChatOpenAI(...)
```

创建的是一个模型客户端对象。

它不是立即调用模型。

真正请求模型是在：

```python
response = llm.invoke(...)
```

发生的。

一般情况下，模型对象可以在图外创建，然后让需要调用模型的 Node 使用它。

### 3.2 最简单的模型调用

```python
response = llm.invoke("你好")

print(type(response))
print(response.content)
```

数据流：

```text
"你好"                str
  │
  ▼
llm.invoke(...)
  │
  ▼
AIMessage             对象
  │
  ├── content
  ├── tool_calls
  ├── response_metadata
  └── usage_metadata
```

`content` 是消息内容，但不应该永远假设它是字符串。不同模型和内容模式下，也可能包含内容块。

对于普通文本对话，通常可以直接读取：

```python
text = response.content
```

### 3.3 传入消息列表

```python
messages = [
    ("system", "你是一个简洁的助手"),
    ("human", "什么是 LangGraph？"),
]

response = llm.invoke(messages)
```

这里传的是：

```text
list
 ├── tuple(system, 内容)
 └── tuple(human, 内容)
```

LangChain 会把这些支持的消息表示转换为相应的消息对象。

也可以显式创建：

```python
from langchain_core.messages import (
    SystemMessage,
    HumanMessage,
)

messages = [
    SystemMessage(content="你是一个简洁的助手"),
    HumanMessage(content="什么是 LangGraph？"),
]
```

两种方式都可以使用。

### 3.4 记住四种常见消息

| 消息类型 | 代表什么 |
|---|---|
| `SystemMessage` | 系统指令 |
| `HumanMessage` | 用户消息 |
| `AIMessage` | 模型回复 |
| `ToolMessage` | 工具执行结果 |

一个带工具调用的对话历史可能长这样：

```text
SystemMessage
HumanMessage
AIMessage(tool_calls=[...])
ToolMessage(...)
AIMessage(content="最终回答")
```

特别注意：

`AIMessage.tool_calls` 代表模型请求调用工具，**不代表工具已经执行**。

---

## 四、Prompt：为什么有时传字典，有时传消息列表

这是第二个容易遗忘的地方。

### 4.1 Prompt 模板需要变量映射

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一名{role}"),
    ("human", "{question}"),
])
```

模板里有两个变量：

```text
role
question
```

因此调用时通常传：

```python
prompt_value = prompt.invoke({
    "role": "Python 工程师",
    "question": "什么是依赖注入？",
})
```

返回的是 `ChatPromptValue`。

可以转成消息列表：

```python
messages = prompt_value.to_messages()
```

完整过程：

```text
dict
{"role": "...", "question": "..."}
       │
       ▼
prompt.invoke(...)
       │
       ▼
ChatPromptValue
       │
       ▼
.to_messages()
       │
       ▼
list[BaseMessage]
```

### 4.2 LCEL 自动连接

```python
chain = prompt | llm

response = chain.invoke({
    "role": "Python 工程师",
    "question": "什么是依赖注入？",
})

print(response.content)
```

为什么 `chain.invoke()` 传字典？

因为链的第一环是 `prompt`。

为什么最终返回 `AIMessage`？

因为链的最后一环是 `llm`。

所以：

**链的输入看第一环，链的输出看最后一环。**

### 4.3 加上输出解析器

```python
from langchain_core.output_parsers import StrOutputParser

chain = prompt | llm | StrOutputParser()

result = chain.invoke({
    "role": "Python 工程师",
    "question": "什么是依赖注入？",
})

print(result)
```

现在：

```python
type(result) is str
```

在普通文本输出场景下成立。

因为最后一环变成了 `StrOutputParser`。

```text
dict
 ↓
Prompt
 ↓
ChatPromptValue
 ↓
LLM
 ↓
AIMessage
 ↓
StrOutputParser
 ↓
str
```

这就是 LCEL 的本质：**上一个组件的输出，成为下一个组件的输入。**

如果中间两个组件的数据契约不匹配，链就会失败。

---

## 五、Runnable：恢复四种组合方式

W4 学过 `RunnableParallel`、`RunnablePassthrough`、`RunnableLambda` 和 `RunnableBranch`。

它们不需要重新背一遍完整文档，先恢复各自的职责。

| 组件 | 作用 | 典型结果 |
|---|---|---|
| `RunnableLambda` | 把普通函数接入链 | 函数返回什么就得到什么 |
| `RunnableParallel` | 同一输入送往多个分支 | 字典 |
| `RunnablePassthrough` | 保留输入或追加字段 | 原始输入或扩展后的字典 |
| `RunnableBranch` | 根据条件选择一条链 | 被选中分支的结果 |

### 5.1 普通函数接入 Runnable

```python
from langchain_core.runnables import RunnableLambda

def normalize(text: str) -> str:
    return text.strip().lower()

normalizer = RunnableLambda(normalize)

result = normalizer.invoke("  HELLO  ")
```

返回：

```python
"hello"
```

### 5.2 并行处理

```python
from langchain_core.runnables import RunnableParallel

parallel = RunnableParallel(
    original=RunnableLambda(lambda x: x),
    length=RunnableLambda(lambda x: len(x)),
)

result = parallel.invoke("hello")
```

得到：

```python
{
    "original": "hello",
    "length": 5,
}
```

这里为什么是字典？

因为 `RunnableParallel` 以命名字段组织各分支的输出。

**并行结果是字典，不代表所有 Runnable 的输出都是字典。**

---

## 六、Tool Calling：模型提出调用，程序负责执行

W4 的工具调用是后面 ReAct Graph 的基础。

先恢复执行过程：

```text
用户问题
   ↓
LLM（已绑定工具）
   ↓
AIMessage
   │
   ├── 无 tool_calls → 直接回答
   │
   └── 有 tool_calls
           ↓
       Python 执行工具
           ↓
       ToolMessage
           ↓
       再次调用 LLM
```

### 6.1 定义工具

```python
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """计算两个整数的和。"""
    return a + b
```

`@tool` 会把函数包装成 LangChain 工具，并提供工具名称、参数 Schema 等信息。

### 6.2 绑定工具

```python
tools = [add]

llm_with_tools = llm.bind_tools(tools)
```

注意：

```python
bind_tools()
```

只是让模型知道有哪些工具可以请求调用。

它不会自动执行工具。

### 6.3 解析工具调用

```python
response = llm_with_tools.invoke(
    "计算 3 + 5"
)

print(response.tool_calls)
```

一次工具调用可能包含：

```python
{
    "name": "add",
    "args": {"a": 3, "b": 5},
    "id": "call_123",
    "type": "tool_call",
}
```

这才是适合使用字典取值的地方：

```python
for call in response.tool_calls:
    name = call["name"]
    args = call["args"]
    call_id = call["id"]
```

区分两层：

```text
response                 AIMessage 对象
   │
   └── .tool_calls       list[dict]
           │
           └── call      dict
```

### 6.4 执行工具并回填结果

```python
from langchain_core.messages import ToolMessage

messages = [
    ("human", "计算 3 + 5")
]

response = llm_with_tools.invoke(messages)

messages.append(response)

tools_by_name = {
    tool_obj.name: tool_obj
    for tool_obj in tools
}

for call in response.tool_calls:
    tool_obj = tools_by_name[call["name"]]

    result = tool_obj.invoke(call["args"])

    messages.append(
        ToolMessage(
            content=str(result),
            tool_call_id=call["id"],
        )
    )

final = llm_with_tools.invoke(messages)
```

这段代码演示的是一次工具执行与回填；真实 Agent 如果还可能继续调用工具，就需要循环和终止条件。

关键数据流：

```text
AIMessage.tool_calls
       │
       ▼
call["args"]          dict
       │
       ▼
tool.invoke(...)
       │
       ▼
工具执行结果
       │
       ▼
ToolMessage
       │
       ▼
messages.append(...)
       │
       ▼
再次调用 LLM
```

还要注意失败边界：模型可能生成错误参数，工具可能抛异常，也可能请求不存在的工具。生产代码需要处理这些情况，并限制循环次数。

---

## 七、结构化输出：让模型结果成为程序数据

普通模型调用通常返回 `AIMessage`。

但有时程序真正需要的是明确的字段。

例如：

```python
from typing import Literal
from pydantic import BaseModel

class Route(BaseModel):
    destination: Literal["search", "answer"]
    reason: str
```

使用：

```python
router_llm = llm.with_structured_output(Route)

result = router_llm.invoke(
    "用户想查询公司内部文档"
)
```

通常得到经过验证的 `Route` 实例：

```python
print(result.destination)
print(result.reason)
```

注意这里不是：

```python
result["destination"]
```

因为使用 Pydantic Schema 时，默认返回的是模型实例。

如果改为传 JSON Schema 等其他形式，返回类型可能不同。

**结构化输出不等于永远返回字典。返回类型由所选 Schema 和配置决定。**

这也是为什么写代码时，不能只记住 `invoke()`，还必须记住前面创建了什么对象。

---

## 八、LangGraph：LLM 应该放在哪里？

现在开始恢复 W5。

这是我重新写代码时最容易犹豫的地方：

> 我已经写了 Node，LLM 是放在 Node 里面，还是在 `graph.invoke()` 的时候传进去？

先分清三个层次。

```text
创建阶段
    │
    ├── 创建 llm
    ├── 定义 node
    ├── 创建 StateGraph
    ├── 注册 node / edge
    └── compile()
             │
             ▼
执行阶段
    graph.invoke(input)
             │
             ▼
       Runtime 调度
             │
             ▼
        agent_node
             │
             ▼
        llm.invoke()
             │
             ▼
       返回 State Update
             │
             ▼
        Runtime 合并
             │
             ▼
        最终 State
```

### 8.1 最小 Graph

```python
from typing import Annotated, TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    messages: Annotated[list, add_messages]


def agent_node(state: AgentState):
    response = llm.invoke(state["messages"])

    return {
        "messages": [response]
    }


builder = StateGraph(AgentState)

builder.add_node("agent", agent_node)

builder.add_edge(START, "agent")
builder.add_edge("agent", END)

graph = builder.compile()
```

运行：

```python
result = graph.invoke({
    "messages": [
        ("human", "你好")
    ]
})

print(result["messages"][-1].content)
```

### 8.2 三个调用的职责

**`llm.invoke()`**

调用真实模型，通常返回 `AIMessage`。

**`agent_node(state)`**

接收 State，执行业务逻辑，返回 State Update。

**`graph.invoke()`**

启动整个 Graph，让 Runtime 按拓扑执行节点、合并状态，最终返回结果。

所以 LLM 不是在 `graph.invoke()` 中临时接入的。

通常是：

```python
llm = ChatOpenAI(...)
```

先创建模型。

然后在 Node 中：

```python
response = llm.invoke(...)
```

最后：

```python
graph.invoke(...)
```

启动整个流程。

### 8.3 为什么 Node 不能直接返回 AIMessage？

错误示意：

```python
def agent_node(state):
    return llm.invoke(state["messages"])
```

因为 LangGraph 的 StateGraph 节点通常需要返回状态更新映射。

正确方式：

```python
def agent_node(state):
    response = llm.invoke(state["messages"])
    return {"messages": [response]}
```

这里：

```text
AIMessage
   ↓
{"messages": [AIMessage]}
   ↓
State Update
   ↓
Reducer
   ↓
Updated State
```

Node 返回的字典不是 HTTP JSON，也不是模型原始响应。

它是对 Graph State 的更新请求。

---

## 九、State、Node、Edge、Reducer：恢复 Runtime 的基本模型

这是 W5 的核心。

```text
State
  │
  ▼
Node
  │
  ▼
State Update
  │
  ▼
Reducer
  │
  ▼
Updated State
  │
  ▼
Edge
  │
  ▼
Next Node
```

### 9.1 State：系统当前掌握的数据

```python
class State(TypedDict):
    question: str
    answer: str
```

State 是多个 Node 之间共享的数据契约。

Node 读取 State：

```python
def answer_node(state: State):
    question = state["question"]

    return {
        "answer": f"收到：{question}"
    }
```

Node 不需要返回整个 State，只需要返回需要更新的字段。

### 9.2 Edge：决定执行路径

普通边：

```python
builder.add_edge("node_a", "node_b")
```

条件边：

```python
def route(state):
    if state["question"].startswith("搜索"):
        return "search"

    return "answer"


builder.add_conditional_edges(
    "router",
    route,
    {
        "search": "search_node",
        "answer": "answer_node",
    },
)
```

`route()` 返回的是路由标签，不是模型回答。

它决定下一步去哪。

### 9.3 Reducer：决定 State 怎样合并

没有特殊 Reducer 时，同一字段的更新通常是覆盖。

例如：

```python
return {"answer": "新的回答"}
```

对于消息列表，往往希望追加，而不是覆盖。

因此：

```python
messages: Annotated[list, add_messages]
```

`add_messages` 会根据消息 ID 合并消息，通常追加新消息；相同 ID 的消息可以被替换。

所以：

```python
return {"messages": [response]}
```

不会简单地把所有历史消息清空。

**State Flow 和 Control Flow 必须分开：**

```text
State + Reducer
    → 数据怎么变化

Edge + Conditional Edge
    → 下一步执行哪里
```

这是理解 LangGraph 后续机制的基础。

---

## 十、ReAct Graph：把手写工具循环交给图

W4 里我们手动写：

```python
while response.tool_calls:
    ...
```

W5 将这个循环改成 Graph。

```text
START
  │
  ▼
agent
  │
  ├── 有 tool_calls ──→ tools
  │                       │
  │                       ▼
  └──────────────────── agent
  │
  └── 无 tool_calls ──→ END
```

最小实现：

```python
from langgraph.prebuilt import ToolNode, tools_condition

builder = StateGraph(AgentState)

def agent_node(state: AgentState):
    response = llm_with_tools.invoke(
        state["messages"]
    )
    return {"messages": [response]}

builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))

builder.add_edge(START, "agent")

builder.add_conditional_edges(
    "agent",
    tools_condition,
)

builder.add_edge("tools", "agent")

graph = builder.compile()
```

这里要恢复几个关系：

- `llm_with_tools`：模型知道可以调用哪些工具。
- `ToolNode`：负责执行模型请求的工具。
- `tools_condition`：判断继续执行工具还是结束。
- `add_edge("tools", "agent")`：工具执行完，再让模型判断下一步。

这样 Graph 负责循环控制。

但这不意味着模型变得可靠了。仍然需要考虑错误工具名、失败重试、权限、循环上限和成本。

---

## 十一、Checkpointer：让 Graph 保存执行状态

W6 开始接触持久化。

先区分两个概念：

```text
messages
    → 对话消息

State
    → 整个 Graph 的运行数据

Checkpoint
    → 某个执行时刻保存的状态快照
```

### 11.1 使用内存 Checkpointer

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

graph = builder.compile(
    checkpointer=checkpointer
)
```

运行时传入：

```python
config = {
    "configurable": {
        "thread_id": "session-001"
    }
}

result = graph.invoke(
    {
        "messages": [
            ("human", "我叫小明")
        ]
    },
    config=config,
)
```

下一次调用相同的 `thread_id`：

```python
result = graph.invoke(
    {
        "messages": [
            ("human", "我叫什么？")
        ]
    },
    config=config,
)
```

Graph 可以接续同一个线程的状态。

注意：

`InMemorySaver` 只保存在内存中。进程结束后不会保留。

需要跨进程重启保存时，应使用持久化 Checkpointer，例如 SQLite 或数据库后端。

### 11.2 查看当前 State

```python
snapshot = graph.get_state(config)

print(snapshot.values)
print(snapshot.next)
print(snapshot.config)
```

三个重要字段：

```text
.values
    当前保存的 State

.next
    等待执行的节点

.config
    定位当前快照的配置
```

还要记住：

```text
thread_id
    哪一个执行线程

checkpoint_id
    这个线程中的哪一个快照
```

**Checkpointer 保存的是执行状态，不等于长期语义记忆。**

长期记忆可能需要额外的 Store、数据库或检索系统。

---

## 十二、HITL：让执行停下来，交给人决定

W6 学到的人工介入，不只是加一个 `input()`。

它解决的是：

> 当 Agent 即将执行危险操作时，如何安全地暂停、检查、修改和恢复？

### 12.1 最小执行流程

```text
START
  ↓
decide
  ↓
approve
  ↓
tools
  ↓
summarize
  ↓
END
```

其中：

```text
decide
    让模型产生 tool_calls

approve
    人工确认

tools
    真正执行工具

summarize
    总结结果
```

### 12.2 `interrupt()` 与恢复

```python
from langgraph.types import interrupt, Command

def approve_node(state):
    decision = interrupt({
        "question": "是否允许执行？"
    })

    return {
        "approved": decision == "approve"
    }
```

第一次执行到 `interrupt()` 时暂停。

恢复时：

```python
graph.invoke(
    Command(resume="approve"),
    config=config,
)
```

这里最容易忘记的是：

**恢复时，包含 `interrupt()` 的节点会重新执行。**

因此，不应把不可安全重复的副作用随意放在 `interrupt()` 前面。

### 12.3 `interrupt_before` 与 `interrupt()`

| | `interrupt_before` | `interrupt()` |
|---|---|---|
| 暂停位置 | 节点执行前 | 节点内部 |
| 恢复时传入值 | 不使用 `Command(resume=...)` 作为节点内返回值 | 使用 `Command(resume=value)` |
| 能否修改 State | 可以 | 可以 |
| 典型用途 | 节点执行前检查 | 节点内部交互式审批 |

这两种暂停机制的选择，取决于希望把审批放在节点边界还是节点内部。

---

## 十三、时间旅行：回放与分叉

有了 Checkpointer，才能讨论执行历史。

```python
history = list(
    graph.get_state_history(config)
)
```

每个历史快照包含当时的 State 和定位信息。

核心不是 API，而是两个动作：

**Replay：从历史快照重新执行。**

**Fork：从历史快照修改状态，再走另一条执行路径。**

```text
原始执行：

A → B → C → D
        │
        └── 历史快照
               │
               ├── Replay
               │      ↓
               │      C → D
               │
               └── Fork
                      ↓
                   修改 State
                      ↓
                   C' → D'
```

这里有一个重要的工程边界：

回放 Graph 不等于撤销外部世界的副作用。

如果历史节点执行过真实文件删除、支付或网络写入，重新执行可能再次触发副作用。

因此，时间旅行和幂等设计必须一起考虑。

---

## 十四、Subgraph 与 Send：复杂流程怎么拆分和并行

W6 后期开始接触复杂编排。

### 14.1 Subgraph：把一张图当作组件

```text
Parent Graph
    │
    ▼
  entry
    │
    ▼
 Subgraph
 ┌──────────────┐
 │ node_a       │
 │    ↓         │
 │ node_b       │
 └──────────────┘
    │
    ▼
  finish
```

Subgraph 解决的是：

**一段复杂流程如何封装成可复用的执行单元。**

它并不天然意味着多 Agent，也不天然意味着并行。

如果父图与子图的 State Schema 不一致，需要明确输入输出映射。

### 14.2 Send：根据数据动态派发任务

```python
from langgraph.types import Send

def dispatch(state):
    return [
        Send("worker", {"topic": topic})
        for topic in state["topics"]
    ]
```

如果：

```python
state["topics"] = [
    "RAG",
    "Agent",
    "MCP",
]
```

就会动态产生三份任务。

```text
             ┌→ worker(RAG) ────┐
             │                  │
dispatch ────┼→ worker(Agent) ──┼→ join
             │                  │
             └→ worker(MCP) ────┘
```

这里要记住：

`Send` 传给 Worker 的是专门构造的输入。

不能假设 Worker 一定能读取父图中的全部 State 字段。

### 14.3 并行结果怎样合并？

例如：

```python
from operator import add
from typing import Annotated

class State(TypedDict):
    results: Annotated[list[str], add]
```

Worker 返回：

```python
return {"results": ["RAG 完成"]}
```

多个 Worker 的结果通过 Reducer 合并。

这里有两个关键问题：

1. 多个 Worker 更新同一字段时，Reducer 是否支持正确合并？
2. 并行完成顺序是否会影响结果？

一般不应依赖并行任务的完成顺序来判断业务正确性。

如果业务需要稳定排序，可以在结果中携带序号，汇总后显式排序。

### 14.4 Send 与普通条件边

```text
Conditional Edge
    → 根据条件选择下一条路径

Send
    → 根据运行时数据动态生成任务
```

如果只是固定的 A/B 路由，用条件边通常更简单。

如果任务数量由输入列表决定，Send 更合适。

---

## 十五、W7：从能运行的 Agent 到可控的 Agent

W7 的 Harness、Supervisor、Router 等内容，其实是在前面所有知识的基础上增加工程边界。

先恢复一个认识：

**LLM 可以提出决策，但系统必须决定哪些决策允许执行。**

例如：

```text
用户输入
   ↓
Agent / LLM
   ↓
Tool Calls
   ↓
Tool Registry
   ↓
Permission Gate
   ↓
Tool Execution
   ↓
Trace / Audit
   ↓
结果回填
```

这里各部分职责不同。

| 模块 | 解决什么问题 |
|---|---|
| Tool Registry | 有哪些工具、怎样找到对应实现 |
| Permission Gate | 哪些操作允许执行 |
| Session Store | 会话与执行状态怎样保存 |
| Context Compaction | 上下文过长时怎样压缩 |
| Trace | 执行过程中发生了什么 |

这些模块不是 LangGraph 自动提供的完整业务方案，而是可以围绕 Agent Runtime 建立的应用层能力。

尤其要记住：

```text
Prompt
    → 告诉模型应该怎么做

Permission Gate
    → 程序实际允许它做什么
```

提示词不是权限系统。

同样：

```text
Trace
    → 记录执行证据

Checkpoint
    → 保存运行状态
```

两者目的不同，不能互相替代。

---

## 十六、流式输出：`stream` 与 `astream` 返回什么

现在回到最近写 FastAPI 时遇到的问题。

### 16.1 模型直接流式输出

```python
for chunk in llm.stream("你好"):
    print(chunk.content, end="")
```

`chunk` 通常是 `AIMessageChunk`。

如果异步：

```python
async for chunk in llm.astream("你好"):
    print(chunk.content, end="")
```

### 16.2 Graph 流式输出

```python
for event in graph.stream(
    {"messages": [("human", "你好")]},
    stream_mode="updates",
):
    print(event)
```

`updates` 模式通常返回节点状态更新事件。

例如：

```python
{
    "agent": {
        "messages": [AIMessage(...)]
    }
}
```

如果使用：

```python
stream_mode="values"
```

通常返回每个流式步骤的完整 State 值。

### 16.3 `stream_mode="messages"`

这正是我最近容易忘记的地方：

```python
async for chunk, metadata in graph.astream(
    {"messages": [("human", "你好")]},
    stream_mode="messages",
):
    print(chunk.content)
```

这里：

```text
graph.astream(...)
       │
       ▼
每次产生一个事件
       │
       ▼
(message_chunk, metadata)
       │
       ├── chunk
       │      AIMessageChunk 等消息块
       │
       └── metadata
              dict
```

所以可以写：

```python
chunk, metadata = event
```

这叫序列解包。

不是字典解包。

`metadata` 可以包含当前节点等执行信息，具体字段要根据实际事件检查。

不同的 `stream_mode`，输出结构不同。

因此遇到陌生流式代码时，第一件事不是看循环怎么写，而是看：

```python
stream_mode=...
```

---

## 十七、把所有知识串起来：一个 Agent 到底怎样运行

现在把 W4–W7 的内容放回一条主调用链。

```text
一、初始化
──────────────────────────
创建 LLM
创建 Tools
绑定 Tools
定义 State
定义 Nodes
注册 Edges
配置 Checkpointer
compile()

二、用户请求
──────────────────────────
graph.invoke(input, config)
        │
        ▼
Runtime 加载 State
        │
        ▼
根据 Edge 执行 Node
        │
        ▼
Node 内部调用 LLM
        │
        ▼
得到 AIMessage
        │
        ├── content
        │
        └── tool_calls
               │
               ▼
          Permission Gate
               │
               ▼
           ToolNode
               │
               ▼
          ToolMessage
               │
               ▼
          再次调用 LLM
        │
        ▼
Node 返回 State Update
        │
        ▼
Reducer 合并 State
        │
        ▼
Checkpointer 保存快照
        │
        ▼
下一节点 / 暂停 / END
        │
        ▼
最终 State
```

这张图里有四条不同的关系：

**调用关系**

```text
graph.invoke()
    → Node
        → llm.invoke()
```

**数据关系**

```text
输入 State
    → AIMessage
    → State Update
    → Updated State
```

**控制关系**

```text
Edge
    → Conditional Edge
    → Send / END
```

**持久化关系**

```text
State
    → Checkpoint
    → Resume / Replay / Fork
```

以后看到复杂 Agent 代码，先判断某一行属于哪条关系。

---

## 十八、忘记时直接查：常用调用恢复表

| 想做什么 | 代码入口 | 需要记住的输出 |
|---|---|---|
| 创建模型 | `ChatOpenAI(...)` | 模型对象 |
| 调用模型 | `llm.invoke(...)` | `AIMessage` |
| 异步调用 | `await llm.ainvoke(...)` | `AIMessage` |
| 模型流式输出 | `llm.stream(...)` | 消息块迭代器 |
| 构建 Prompt | `ChatPromptTemplate.from_messages(...)` | Prompt 模板 |
| 填充 Prompt | `prompt.invoke({...})` | `ChatPromptValue` |
| 组合链 | `prompt \| llm \| parser` | Runnable 链 |
| 获取纯文本 | `StrOutputParser()` | 通常为 `str` |
| 绑定工具 | `llm.bind_tools(tools)` | 绑定工具后的模型对象 |
| 获取工具调用 | `response.tool_calls` | `list[dict]` |
| 结构化输出 | `llm.with_structured_output(Schema)` | 配置后的模型对象 |
| 创建 Graph | `StateGraph(State)` | Builder |
| 注册节点 | `builder.add_node(...)` | 修改 Builder |
| 注册边 | `builder.add_edge(...)` | 修改 Builder |
| 编译 Graph | `builder.compile()` | 可执行 Graph |
| 执行 Graph | `graph.invoke({...})` | 最终 State |
| 查看状态 | `graph.get_state(config)` | StateSnapshot |
| 查看历史 | `graph.get_state_history(config)` | 快照迭代器 |
| 人工暂停 | `interrupt(...)` | 暂停后等待恢复值 |
| 恢复执行 | `Command(resume=...)` | 作为恢复输入 |
| 动态派发 | `Send(node, input)` | Send 对象 |
| 流式执行 | `graph.astream(...)` | 取决于 stream_mode |

---

## 十九、真正开始写代码时，应该怎样恢复思路

以后如果重新接入一个真实 LLM，不要试图一次回忆所有知识。

先恢复最短主链：

```text
创建模型
    ↓
构造消息
    ↓
llm.invoke()
    ↓
AIMessage
    ↓
读取 content
```

如果需要 Prompt：

```text
dict
    ↓
prompt
    ↓
llm
    ↓
AIMessage
```

如果需要 Graph：

```text
State
    ↓
Node
    ↓
llm.invoke()
    ↓
State Update
    ↓
Graph Runtime
```

如果需要工具：

```text
LLM
    ↓
tool_calls
    ↓
ToolNode
    ↓
ToolMessage
    ↓
LLM
```

如果需要会话恢复：

```text
Graph
    ↓
Checkpointer
    ↓
thread_id
    ↓
StateSnapshot
```

如果需要人工确认：

```text
Node
    ↓
interrupt
    ↓
人类决策
    ↓
Command(resume)
    ↓
继续执行
```

如果需要并行：

```text
Router
    ↓
Send × N
    ↓
Workers
    ↓
Reducer
    ↓
汇总节点
```

这几条链，基本可以恢复 W4–W7 的核心知识。

---

## 二十、最后记住的不是 API，而是六个问题

以后遇到不熟悉的 LangChain / LangGraph 代码，先问：

1. **这个对象是什么类型？** 是模型、Prompt、Runnable、消息，还是 Graph？
2. **谁调用它？** 是我直接调用，还是 Runtime 自动调用？
3. **它接收什么？** 是字符串、消息列表、变量字典，还是 State？
4. **它返回什么？** 是 `AIMessage`、字符串、字典、State Update，还是事件？
5. **结果交给谁？** 是解析器、工具执行器、Reducer，还是下一个 Node？
6. **失败在哪里处理？** 是输入校验、模型调用、工具边界，还是 Runtime 控制层？

尤其是第四个问题。

我今天已经知道 `invoke()`，却忘记它返回什么；知道 Prompt，却忘记输入应该传字典还是消息列表；知道 Node，却忘记 LLM 应该在哪一层调用。

这些问题的根源不是概念没学过，而是**没有及时恢复代码之间的数据契约**。

所以以后读源码，不只看函数名。

要重新建立：

**输入 → 调用 → 输出 → 下一步。**

这就是这次 LangChain × LangGraph 记忆恢复最重要的主线。


