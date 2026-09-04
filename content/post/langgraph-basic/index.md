---
description: ""
title: "认识LangGraph"
draft: false
date: "2026-09-04T01:30:22+08:00"
slug: "LangGraph-basic"
categories:
 - LangGraph
tags:
 - Agent
image: ""
---


# W5-D1 · 从"写流程"到"声明流程"

> **关键词**：`StateGraph` / State / node / edge / `compile()` / Runnable 互通
> **前置**：W4-D1 LCEL 三件套、D3 记忆桥 `_to_lc`、D5 `@tool` + `bind_tools`、D6 闭环 `ask()`
> **涉及文件**：`W5/lg_basics.py`、`W5/lg_c1_w4_chain.py`、`memory/memory_manager.py`
> **收尾预告**：D2 条件边（`add_conditional_edges`）——让图自己决定"下一步走哪"

* * *

## 零、开场：今天不写新功能，换脑子

**先说清楚今天的目标**：W5-D1 不产出更聪明的 Agent，它产出的是**一种新写法**。

你 W4-D6 那个 `ask()` 已经能跑了——能记忆、能调工具、能多轮对话。今天要做的，是把同样的东西用**图**重新表达一遍。**跑通那一刻你会发现它甚至还不如你的 REPL 好用**（这点我不骗你，文末"魔鬼代言人"会说）。

但地基必须今天打。因为 D2 加分支、D4 加回边、D5 把 `_run_tools` 整个删掉换成一条边——**全都要站在这张图上**。

* * *

## 一、为什么要有 LangGraph——先回忆你的"撞墙感"

### 1.1 你 W4-D6 手写的那个循环

```fallback
while ai.tool_calls:          # 模型还要调工具？
    执行工具 → 回填 ToolMessage → 再问模型
```

它能跑。**但你已经隐隐感觉到三堵墙**：

| 墙 | 具体表现 |
|---|---|
| **状态散落** | `messages`、轮次、记忆全是你自己用变量管的，谁都能改、容易乱 |
| **没有"中途停"** | 想让 Agent"先问用户确认再继续"，你得自己写一堆 `if` |
| **分支靠手写** | "失败重试""走 A 还是走 B"全靠自己在循环里堆逻辑 |

### 1.2 三堵墙的根源是同一个

盯着那个 `while` 看三秒——**流程的走向，是被写死在代码的行序里的**。

```
  while ai.tool_calls:
      for call in ai.tool_calls:     ← 第 1 步在这行
          result = ...
          messages.append(...)       ← 第 2 步在这行
      ai = model.invoke(messages)    ← 第 3 步在这行
```

想在第 2 步和第 3 步之间插一个"人工确认"？你得改代码结构。想让失败时重试而不是继续？再堆一个 `if`。**流程每变一次，循环体就胖一圈。**

> **类比**：这就像**用一堆 if-else 拼出来的地铁线路图**——每加一站，就得改一遍所有相关的判断。线路一复杂，没人敢动。

### 1.3 LangGraph 的答案很朴素

**别再用 `while` 手推流程了，把流程画成一张图，让图自己走。**

你只声明三件事：

1. 有哪些步骤（node）
2. 谁连谁（edge）
3. 什么条件下走哪条（conditional edge，D2 学）

剩下的交给框架。

> 🎯 **一句话给面试官**：*"LCEL 是无环链——一条道走到黑；LangGraph 是有环有状态的图——能循环、能分支、能中断恢复，这正是 Agent『调工具 → 看结果 → 再决定』循环需要的。"*

### 1.4 LCEL 和 LangGraph 到底什么关系

很多人以为学了 LangGraph 就不用 LCEL 了——**错，它俩是互补的**：

```
┌──────────────────────────────────────────────────────┐
│  LCEL（W4）                                            │
│  prompt | model | parser                              │
│  · 无环：数据流单向，走完就结束                         │
│  · 无状态：每次 invoke 都是干净调用                     │
│  · 擅长：单个步骤内部的"加工流水线"                    │
├──────────────────────────────────────────────────────┤
│  LangGraph（W5）                                       │
│  StateGraph：node + edge + state                      │
│  · 有环：能绕回去重复执行                              │
│  · 有状态：State 在节点间传递并被更新                   │
│  · 擅长：多个步骤之间的"编排调度"                      │
└──────────────────────────────────────────────────────┘
           ↓ 不是替代，是分工 ↓
  步骤内部用 LCEL，步骤之间用 LangGraph
```

> **类比**：LCEL 是**每道工序里的机器**，LangGraph 是**车间里连接各台机器的传送带网络**。你不会用传送带去替代机器，也不会用机器去替代传送带。

* * *

## 二、贯穿全篇的类比：工厂流水线 + 工单

**今天所有概念都往这个类比上挂**，记牢它，后面每个新名词都有落点。

| LangGraph 概念 | 工厂类比 | 一句话职责 |
|---|---|---|
| **State** | 工单 | 在工位间传递的纸质单子，上面有几个格子（字段），每个工位只填自己负责那格 |
| **node** | 工序 / 工位 | 一道工序（比如"盖章"）——接过工单，填自己那格，交回去 |
| **edge** | 传送带 | 决定干完这道活传给谁 |
| **START / END** | 投料口 / 成品出口 | 固定边界，不是你自己起的工位名 |
| **`compile()`** | 组装流水线 | 把图纸变成真能跑的生产线 |
| **`invoke()`** | 放单 | 把一张初始工单放到投料口，一路跑到出口 |
| **`stream()`** | 站在旁边看 | 你站在传送带旁，看工单经过每道工序时被填了什么 |

```
  投料口              工位 A            工位 B            成品出口
  START    ──▶    [ think ]   ──▶   [ respond ]   ──▶    END
                       │                  │
                       ▼                  ▼
                  填"answer"格       在 answer 后面追加

  工单（State）流转：
  {question: "什么是 LangGraph?"}                    ← 投料时
  {question: "...", answer: "[已思考] ..."}           ← 过完 think
  {question: "...", answer: "[已思考] ... → 回复完成"} ← 过完 respond → 出库
```

**关键体会**：**工单是同一张，在每个工位被逐步填写**——不是每个工位重新造一张。这个区别，就是步骤 B 的 `turns` 字段要让你体会的东西。

* * *

## 三、四个概念逐个拆

### 3.1 完整最小图（照抄跑一遍，5 分钟）

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

# ── State：定义"工单上有哪些格子" ─────────────────────────
class State(TypedDict):
    question: str      # 入口时填好（invoke 传入）
    answer: str        # 由节点往这里填

# ── node：普通函数，输入整张工单，返回"我要更新的格子" ────
def think(state: State) -> dict:
    return {"answer": f"[已思考] {state['question']}"}   # ⚠️ 只返回 dict

def respond(state: State) -> dict:
    return {"answer": state["answer"] + " → 回复完成"}

# ── 搭图：加工序 → 连传送带 ───────────────────────────────
builder = StateGraph(State)
builder.add_node("think", think)          # 给工序起名字 + 绑函数
builder.add_node("respond", respond)
builder.add_edge(START, "think")          # 投料口 → 第一道工序
builder.add_edge("think", "respond")      # think 干完 → respond
builder.add_edge("respond", END)          # respond 干完 → 出口

# ── compile：图纸变生产线（图因此变成了一个 Runnable）─────
graph = builder.compile()

# ── invoke：放一张工单，跑到底 ─────────────────────────────
out = graph.invoke({"question": "什么是 LangGraph?"})
print(out)   # {'question': '...', 'answer': '[已思考] ... → 回复完成'}
```

### 3.2 State：工单长什么样

用 `TypedDict` 声明——**不是普通 dict，也不是 dataclass**：

| 选择 | 行不行 | 为什么 |
|---|---|---|
| `TypedDict` | ✅ 官方推荐 | 既有类型提示，又是纯 dict（序列化友好，checkpointer 要用） |
| 普通 `dict` | ⚠️ 能跑 | 没有字段提示，IDE 不提醒，字段多了容易拼错 |
| `dataclass` | ⚠️ 能用但别扭 | 是对象不是 dict，序列化要额外处理 |

**State 的本质就是一张"字段清单"**——声明这张工单上有哪些格子、每格装什么类型。

> 💡 有个细节先埋个伏笔：**每个字段的更新策略默认是"覆盖"**（后写的覆盖先写的）。如果你想让某个字段"累加"（比如 `turns` 每次 +1、消息列表每次 append），需要 `Annotated[list, operator.add]` 这种 **reducer** 声明。这是 D2/D3 的事，今天先知道有这么个开关。

### 3.3 node：工序（最容易误解的地方）

**node 就是一个普通的 Python 函数**，签名是 `(state) -> dict`。

```python
def think(state: State) -> dict:
    return {"answer": f"[已思考] {state['question']}"}
```

但它的**返回值语义**新手十有八九会搞错：

```
  ❌ 误解：return 的是"新的完整 state"
  ✅ 正解：return 的是"我要更新 state 的哪些格子"（partial update）
```

**对比一下就清楚了**：

```python
# ❌ 错误：返回整个 state（而且没带的字段会丢失语义）
def think_bad(state: State) -> State:
    return {"question": state["question"], "answer": "..."}   # 能跑但画蛇添足

# ✅ 正确：只返回要更新的字段
def think(state: State) -> dict:
    return {"answer": "..."}

# ✅ 也合法：什么都不更新
def noop(state: State) -> dict:
    return {}
```

> ⚠️ 如果你 `return` 了一个 State 里**不存在的字段名**，LangGraph 会直接报错（`InvalidUpdateError`）——工单上没这个格子，你往哪填？

> **类比**：工位接过工单，**只在自己负责的那格写字，然后把整张单子交回传送带**。它不会重新打印一张新工单，也不能在工单上画表格外的新格子。

### 3.4 edge + START/END：传送带与边界

```python
builder.add_edge(START, "think")
builder.add_edge("think", "respond")
builder.add_edge("respond", END)
```

- `add_edge(A, B)` = "A 干完，工单自动传到 B"
- **`START` 和 `END` 是从 `langgraph.graph` 导入的固定常量**，不是你自己起的节点名
- 每个非 END 节点**必须有一条出边**，否则图不知道往哪走

```
  START ──▶ think ──▶ respond ──▶ END
   ↑                              ↑
  投料口（固定）                  出口（固定）
```

> 💡 **改流程 = 改边，不动节点。** 想把 `think → respond` 换成 `think → verify → respond`？加一个节点、改两条边，`think` 和 `respond` 的函数一个字都不用动。**这就是"声明流程"的威力**——流程拓扑和业务逻辑解耦了。

### 3.5 `compile()`：图纸变生产线

```python
graph = builder.compile()
```

**为什么不 compile 就不能跑？** 因为 `builder`（`StateGraph`）只是**图纸**——你往上面画了节点和边，但它还不是可执行的东西。`compile()` 做的事：

1. **校验图结构**：有没有孤立节点、有没有走不通的边、START 到 END 是否连通
2. **生成执行器**：把拓扑翻译成可执行的调度逻辑
3. **包装成 Runnable**：让图获得 `invoke / stream / batch / ainvoke`

> ⚠️ **`draw_ascii()` 报"没有图"或报属性错误，99% 是忘了 `compile()`**——你在对一张图纸问"生产线长啥样"。

### 3.6 🎯 三个铁律（今天就要刻进脑子）

1. **节点必须 `return {...}`**——它返回的是"我要更新 state 的哪些格子"，不是整个新 state。图不读全局变量。
2. **必须 `compile()` 才能 `invoke()`**——图纸不能直接跑。
3. **`START` / `END` 从 `langgraph.graph` 导入**——它们是固定边界，不是你自己起的节点名。

* * *

## 四、亲眼看见流转：`stream()`

光看 `invoke()` 的最终结果不过瘾——**你想看工单经过每个工位时被填了什么**。加两行：

```python
# 在 lg_basics.py 末尾追加
print("\n--- 逐节点观察流转 ---")
for chunk in graph.stream({"question": "你好"}):
    for node_name, update in chunk.items():   # chunk = {节点名: 它更新了什么}
        print(f"[{node_name}] 填了 → {update}")
```

预期输出：

```fallback
--- 逐节点观察流转 ---
[think] 填了 → {'answer': '[已思考] 你好'}
[respond] 填了 → {'answer': '[已思考] 你好 → 回复完成'}
```

**注意 `stream()` 吐出来的 chunk 结构**：`{节点名: 该节点的返回值}`。每个节点跑完就吐一个 chunk——所以你看到的是**工单被逐步填写的过程**，不是最终结果。

> 💡 **这就是整张图最性感的时刻**：一个节点跑完 → 自动进下一个，**顺序由边决定，不是由你的代码顺序决定**。改边 = 改流程，不用动节点函数。

**再加一个：打印图结构**

```python
print(graph.get_graph().draw_ascii())
```

它会把拓扑画成 ASCII 框图（实际输出是方框排版，这里简化示意）：

```fallback
START → think → respond → END
```

> ⚠️ 新版 API 是 `graph.get_graph().draw_ascii()`，**不是** `graph.draw_ascii()`。版本对不上就报属性错误。

* * *

## 五、🎯 思维切换表（今天最重要的一节，多读两遍）

| 你过去的做法（W2 / W4） | 图思维（W5） | 为什么 |
|---|---|---|
| 状态是散在代码里的变量，谁都能改 | State 是唯一的"工单"，节点只更新自己那格 | 多人/多步骤协作时，全局变量=灾难；工单=可控 |
| 流程顺序靠代码行序 / `while` 控制 | 流程顺序靠**边**（edge） | 想插一步/改顺序，改边即可，节点不动 |
| "该调工具还是回答"自己 `if` 判断 | 条件边让**图自己决定**走哪条（D2 学） | 决策逻辑从代码里拿出来，变成图的一部分 |
| 中断/恢复自己写 | 图状态天然可保存恢复（D3 学） | 框架级能力，不用自己造 |

> **核心理念就一句**：**在 LangGraph 里，你不再"写流程"，而是"声明流程"**——声明有哪些节点、怎么连、什么条件下走哪条边，然后图自己跑。

### 5.1 这个切换有多反直觉

给你一个真实的翻车场景——刚从手写循环转过来的人，十有八九会写出这种代码：

```python
# ❌ 手写循环思维的残留
answer = ""                      # 全局变量存答案

def think(state: State) -> dict:
    global answer                # 改全局变量！
    answer = f"[已思考] {state['question']}"
    return {}                    # 什么都不返回

# 结果：state["answer"] 永远是空的
```

**为什么错**：节点的数据出口**只有返回值这一条**。你在函数里改全局变量、改外部对象，图一概不知——工单上那格还是空的。

```
  正确心智：
  ┌─────────────┐
  │    node     │
  │  读 state   │ ← 入参（只读，别改）
  │  算一算     │
  │  return {}  │ ← 唯一出口（要更新的字段）
  └─────────────┘
```

> ⚠️ 卡住的时候**回来读第 5 节这张表**。从"手写循环思维"切到"图思维"，最常犯的就是这一条。

* * *

## 六、图也是 Runnable——互通是怎么来的

`compile()` 返回的图对象，**本身就是个 Runnable**：它支持 `invoke / stream / batch / ainvoke`。

这意味着**双向互通**：

```
  方向一：LangChain → LangGraph
  ┌──────────────────────────────────────┐
  │  把 W4 的 LCEL 链塞进图里当节点       │
  │  chain = prompt | llm | parser        │
  │  node 函数里 chain.invoke(...)        │
  └──────────────────────────────────────┘

  方向二：LangGraph → LangChain
  ┌──────────────────────────────────────┐
  │  把一张图塞进 LCEL 链里当一环         │
  │  big_chain = prompt | graph | parser  │
  └──────────────────────────────────────┘
```

**为什么能互通？** 因为两边都实现了同一套 Runnable 协议——**`|` 只认 `Runnable`，不认你具体是链还是图**。

> **类比**：工厂的机器接口是**标准插座**（Runnable 协议），不管是"打磨机"（LCEL 链）还是"整条子流水线"（编译好的图），只要插头符合标准就能插上去。

**这就是步骤 C"互通"能成立的原因**，也是 W5 后期"记忆组件一行不改、只改编排"的物质基础。

* * *

## 七、实战：步骤 A → B → C

### 7.1 步骤 A：最小图跑通

你已经在上面的演示代码里做过一遍了。照任务表建 `W5/lg_basics.py` 收尾：

```python
print(graph.get_graph().draw_ascii())
```

**确认打印出 `START → think → respond → END` 再勾掉这一项。**

> ✅ 别小看这一步——**确认图结构和你脑子里画的一样**，是后面所有调试的基准。图复杂了以后，第一件事永远是打结构图。

### 7.2 步骤 B：加 `turns` 字段（思维切换的试金石，别跳过）

```python
class State(TypedDict):
    question: str
    answer: str
    turns: int          # 新增

def think(state: State) -> dict:
    return {
        "answer": f"[已思考] {state['question']}",
        "turns": state["turns"] + 1,     # 读旧值，写新值
    }
```

`invoke` 时传入 `{"question": "...", "turns": 0}`，观察输出里 `turns == 1`。

#### 这一小步到底在验什么

**验的是：state 是"同一张工单被传递"，不是每个节点重新造一张。**

```
  invoke 传入          {question: "..."} , turns: 0
       │
       ▼
  think 节点读到        state["turns"] == 0     ← 拿到的是上一手传下来的值
       │
       │  return {"turns": 0 + 1}
       ▼
  工单变成             turns: 1                ← 在同一张单子上改
       │
       ▼
  下一个节点读到        state["turns"] == 1     ← 能看见 think 的修改
```

如果 `turns` 输出是 `1` 而不是 `0`，说明你真的理解了：**节点之间通过工单传递信息，而不是通过全局变量或函数参数。**

> ⚠️ **一个必须知道的限制**：这里的 `+1` 是**你自己算的**（`state["turns"] + 1`），不是框架帮你累加的。默认情况下字段更新是**覆盖**——如果你有两个节点都写 `turns`，后写的会盖掉先写的。
>
> 想让框架自动累加，要用 reducer：
> ```python
> from typing import Annotated
> import operator
>
> class State(TypedDict):
>     turns: Annotated[int, operator.add]   # 每次 return 的值会被"加"上去
> ```
> 有了 reducer，节点里 `return {"turns": 1}` 就会自动累加，不用读旧值。**这是 D3 讲记忆/消息列表时的核心机制**，今天先认识一下这个开关。

### 7.3 步骤 C：把 W4 的链包成节点（完整代码）

**先说一个判断，避免你走弯路**：

> 🚫 **你的 `ask()` 不能整段塞进今天的图。**
>
> 它里面含工具循环（`_run_tools`），而 W5-D1 的图里**还没有工具执行节点**（那是 D4 的 `ToolNode` 的事）。
>
> 今天步骤 C 只做一件事：**验证"链能进图"**。所以进图的是 `prompt | model` 这条链，工具循环继续留在图外面（D5 才真正换掉它）。

#### 完整代码：`W5/lg_c1_w4_chain.py`

```python
# W5/lg_c1_w4_chain.py —— W5-D1 步骤 C：把 W4 的链包成 LangGraph 节点
# 一句话：验证 LangChain ↔ LangGraph 互通 —— 链能进图，图能跑链，记忆照常工作
import os, sys
from pathlib import Path
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))  # 回项目根，同 W4
from dotenv import load_dotenv
load_dotenv(Path(__file__).parent.parent / ".env", override=True)

from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from memory.memory_manager import MemoryManager      # W3 组件，原样复用

# ══════════ 1. W4 组件照搬（一行不改） ══════════
mm = MemoryManager(window=8, max_tokens=1500, keep=4)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是有记忆，能调工具的助手。历史对话和检索到用户事实、喜好和用户说记下的内容都在history里"),
    ("placeholder", "{history}"),   # 历史消息原地展开
    ("human", "{question}"),
])

# ⚠️ 关键：这里用【不绑工具】的 llm，不是 W4 那个 bind_tools 的 model！
llm = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)

# W4 的桥：dict 消息 → BaseMessage（原样搬）
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "assistant":                # ✅ 标准映射（见下方说明）
        return AIMessage(content=content)
    return HumanMessage(content=content)

# 组装成 W4 那条链：prompt → llm（它就是 Runnable，图节点直接 invoke）
chain = prompt | llm

# ══════════ 2. 定义图：一个节点，壳是图、心是 W4 的链 ══════════
class State(TypedDict):
    question: str    # invoke 时传入
    answer: str      # 由节点填

def run_w4_chain(state: State) -> dict:
    """节点 = 薄壳：取记忆 → 跑 W4 链 → 写回记忆。业务逻辑全在壳外，一行没改"""
    history = [_to_lc(m) for m in mm.get_context(query=state["question"])]
    ai = chain.invoke({"history": history, "question": state["question"]})
    mm.add({"role": "user", "content": state["question"]})          # 记忆照常写回
    mm.add({"role": "assistant", "content": ai.content})
    return {"answer": ai.content}      # 只把 answer 更新进 State

builder = StateGraph(State)
builder.add_node("w4_chain", run_w4_chain)
builder.add_edge(START, "w4_chain")
builder.add_edge("w4_chain", END)
graph = builder.compile()

# ══════════ 3. 跑 ══════════
if __name__ == "__main__":
    print("=== W5-D1 步骤 C：W4 链包成 LangGraph 节点 ===\n")
    print(graph.get_graph().draw_ascii())          # 应看到 START → w4_chain → END

    # 3 连问：①纯问答 ②写记忆 ③考记忆（证明图外组件 mm 照常工作）
    for q in ["你好，简单介绍下你自己",
              "记住：我最喜欢的模型是 DeepSeek",
              "我最喜欢的模型是什么？"]:
        out = graph.invoke({"question": q})        # 每次从 START 全量跑一遍
        print(f"Q: {q}\nA: {out['answer']}\n")
```

> ⚠️ **注意 `_to_lc` 我改了映射**：这里写的是 `if role == "assistant": return AIMessage(...)`，**不是**你 W4 代码里的 `if role == "user": return AIMessage(...)`。
>
> 后者是我在 **W4-D6 那篇里定位过的 bug**——它把你说的 user 消息映射成了 `AIMessage`，导致多轮对话历史里**角色全反**。标准映射是：
> ```
> system    → SystemMessage
> assistant → AIMessage
> user      → HumanMessage   （兜底分支）
> ```
> 同样的 bug 我在 W4-D6 的"坑 5"里详细推演过后果，这里直接用修好的版本。**你 W4 的 `lc_agent_demo.py` 也记得同步改掉。**

#### 预期输出形状

```fallback
START → w4_chain → END          ← draw_ascii 画出的图结构
                                  （实际输出是 ASCII 方框排版，这里简化示意）

Q: 你好，简单介绍下你自己
A: 你好！我是有记忆、能调工具的助手……

Q: 记住：我最喜欢的模型是 DeepSeek
A: 好的，我记住了！

Q: 我最喜欢的模型是什么？
A: 你最喜欢的模型是 DeepSeek。   ← 证明 mm 记忆跨 invoke 生效
```

> 💡 第 3 问答不答得出，取决于你 W3 `mm.get_context` 的检索能力（语义检索 / 窗口）——**这不是今天要验证的**。今天只验证：图能跑链、记忆组件在节点里被正常调用。

* * *

## 八、与 W4 代码逐行对照（面试能讲清"哪改了、哪没改"）

| 你的 W4 | 步骤 C 的改动 | 为什么 |
|---|---|---|
| `model = ChatOpenAI(...).bind_tools(tools, parallel_tool_calls=False)` | `llm = ChatOpenAI(...)` **去掉 `bind_tools`** | 今天图里没工具节点，绑了会触发空 `content` |
| `prompt.format_messages(...)` 再 `model.invoke(messages)` | `chain = prompt \| llm`，节点里 `chain.invoke({...})` | `ChatPromptTemplate` 本身是 Runnable，直接组链 |
| `_run_tools()` 工具循环 | **不进图**（留在 W4 文件里） | D5 用 `ToolNode` 替换它，今天别一把梭 |
| `ask()` 里取记忆 / 写回 | 搬进 `run_w4_chain` 节点函数 | 组件逻辑一行没改，只是"调用点"从 `ask()` 挪进节点 |
| `mm = MemoryManager(...)` | 原样 | W3/W4 红利：**只改编排，不改业务** |

**看清楚这张表的最后一列**——除了"去掉 `bind_tools`"这一处是**必要的适配**，其余全是**搬家**：把 W4 已经写好的东西换了个调用位置，代码本身一字未改。

> 🎯 **这就是 W5 的主线**：**组件不动，只改编排。** 你 W1~W4 造的轮子，在图里全部照常使用。

* * *

## 九、踩坑记

### 坑 1：`ImportError: langgraph` ⚠️

- **现象**：`from langgraph.graph import ...` 报模块不存在
- **解法**：`pip install langgraph`（纯 Python，Windows 直接装，不需要编译）
- 💡 它和 `langchain-core` 是独立的包，**装了 langchain 不等于装了 langgraph**

### 坑 2：`draw_ascii()` 报错或没图 ⚠️

- **现象**：属性错误，或打印出来是空的
- **原因**：99% 是忘了 `compile()`，或用了旧版 API
- **解法**：新版一定是 `graph.get_graph().draw_ascii()`（不是 `graph.draw_ascii()`）

### 坑 3：节点改了全局变量，但 state 没变 ⚠️

- **现象**：节点函数明明执行了，`state["answer"]` 还是空的
- **原因**：违反铁律①——**节点必须 `return {...}`**
- **解法**：删掉 `global`，把要写的值放进返回 dict
- 💡 这是从"手写循环思维"切到"图思维"最常犯的错，**卡住就回来读第 5 节那张表**

### 坑 4：用了绑工具的 model，`answer` 是空的 ⚠️

- **现象**：`out['answer']` 是空字符串，但没报错
- **原因**：**绑了工具的模型收到问题可能主动发起 `tool_calls`**，而今天的图里没有工具执行节点 → 没人回应 → `ai.content` 为空
- **解法**：步骤 C 用**不绑工具的干净 `llm`**
- 🎯 **记住这个因果关系**：**"绑工具"和"图里有 `ToolNode`"必须成对出现。** D4 你加 `ToolNode` 时，就会明白今天这个坑是在教你怎么配对

> **类比**：你给服务员发了一本登记簿（`bind_tools`），他填了单子要厨房做菜——但今天这个车间**根本没有厨房**。菜没人做，客人（你）只拿到一张空白单子。

### 坑 5：`from memory.memory_manager import MemoryManager` 报错 ⚠️

- **原因**：`sys.path` 是硬编码回根目录的，文件路径对不上就找不到包
- **前提**：文件必须放在与 `memory/`、`.env` 同一项目根下的 `W5/` 子目录
- **解法**：目录结构不同就改对 `os.path.dirname(...)` 的层数
- 💡 你 W4 用了 `parent.parent`，W5 同样——**两份文件保持同一个根目录约定**

### 坑 6：连续 `invoke` 三次，记忆为什么能跨次生效？🤔

这不是坑，但是个**必须想明白的边界**：

```
  第一次 invoke ──▶ mm.add(...) ──▶ 记忆写进 MemoryManager（图外部对象）
  第二次 invoke ──▶ mm.get_context() 读到第一次的  ✅
  第三次 invoke ──▶ mm.get_context() 读到前两次的  ✅
```

**因为 `mm` 是图外部的单例对象，State 里根本没放 history。**

> 🎯 **这正是 W5 的编排哲学**：**记忆是外部组件，图只负责编排调用。**
>
> 今天你看到的是"组件在节点里被手动调用"（取记忆 / 写回都写在节点函数里）。真正的**图状态持久化**是 D3 的 checkpointer——到时候 State 自己就能跨 invoke 保存恢复。**今天先感受这个边界**，别急着把记忆塞进 State。

### 坑 7：字段更新默认是"覆盖"，不是"合并" ⚠️

- **现象**：两个节点都写了 `answer`，后者把前者的内容整个盖掉了
- **原因**：LangGraph 默认更新策略是 last-write-wins
- **解法**：要累加就用 `Annotated[int, operator.add]` 这类 reducer（见 7.2）
- 💡 `respond` 节点里 `state["answer"] + " → 回复完成"` 是**手动拼接**——正因为默认会覆盖，所以才需要你自己先把旧值读出来


## 十一、速查卡片（复习直接看这）

```python
# ===== 四件套 =====
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    question: str                              # 入口字段（invoke 传入）
    answer: str                                # 节点填
    # turns: Annotated[int, operator.add]      # 要"累加"就用 reducer

def node(state: State) -> dict:                # 🎯 必须 return dict（partial update）
    return {"answer": ...}                     # 只返回要更新的字段

builder = StateGraph(State)
builder.add_node("name", node)                 # 加工序
builder.add_edge(START, "name")                # 投料口 → 工序
builder.add_edge("name", END)                  # 工序 → 出口
graph = builder.compile()                      # 🎯 必须 compile 才能 invoke

# ===== 三种跑法 =====
out = graph.invoke({"question": "..."})        # 跑到底，拿最终 State
for chunk in graph.stream({"question": "..."}):# 逐节点观察
    print(chunk)                               # 结构：{节点名: 该节点 return 的 dict}
print(graph.get_graph().draw_ascii())          # 🎯 新版 API，别漏 get_graph()

# ===== 互通：链进图 =====
chain = prompt | llm                           # W4 的 LCEL 链
def run_w4_chain(state: State) -> dict:
    ai = chain.invoke({"history": ..., "question": state["question"]})
    return {"answer": ai.content}              # 只把结果写回 State
builder.add_node("w4_chain", run_w4_chain)

# ===== 三个铁律 =====
# 1. 节点必须 return {...}，不碰全局变量
# 2. 必须 compile() 才能 invoke()
# 3. START / END 从 langgraph.graph 导入
```

* * *

## 十二、一句话总结

**今天你只学一个心智——从"写流程"切换到"声明流程"：声明节点、声明边、声明工单长什么样，剩下的让图自己跑。**

三个落点：

1. **State 是唯一工单**，节点只填自己那格（`return {...}`，不碰全局变量）
2. **流程顺序由边决定**，改边不改节点——这是"声明流程"的全部威力
3. **图编译后是 Runnable**，所以能和 W4 的 LCEL 链双向互通——**组件一行不改，只换调用点**

跑通 `think → respond` 那一刻，你就已经站在 D4"工具循环进图"的起跑线上了。


真正的分水岭在 D4——**当你亲手把 W4 那个 `while` 工具循环删掉、换成一条回边时**，那才是图真正发力的时刻：

```
  今天（D1）：  START → w4_chain → END              线性，无环
  D2：         START → think →┬→ respond → END      有分支
                             └→ retry ↗
  D4：         START → agent →┬→ tools → agent ↗    有环（回边）
                             └→ END

                                    ↑
                          这一刻，while 才真正消失
```

* * *
