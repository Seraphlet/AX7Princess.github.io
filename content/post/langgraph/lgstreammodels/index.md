---
description: ""
title: "观察一张图：四种流式看法（打字机与异步第一课）"
draft: false
date: "2026-09-06T09:03:37+08:00"
slug: "LGStreamModels"
categories:
 - LangGraph
tags:
 - 
image: ""
---


# W5-D4 · 观察一张图：四种流式看法

> **关键词**：`stream_mode` / updates / values / messages / `astream_events` / async / await
> **前置**：W5-D3（ReAct 图 agent ↔ tools 循环）；**本篇含 Python 异步零基础课**
> **涉及文件**：`W5/lg_d4_stream_modes.py`
> **收尾预告**：D5 State Schema 进阶（TypedDict → Pydantic）——为 MemoryManager 图和 W6 打底

* * *

## 零、开场：D3 造了机器，D4 学"看"机器

D3 你搭了一张会自己转圈的 ReAct 图：`agent ↔ tools` 循环，模型调工具、工具回填、再决策——**机器造好了**。

D4 学的是**怎么观察它运转**。同一张图，四种看法，粒度从粗到细：

| 看法 | 粒度 | 你看到的东西 | 类比 |
|---|---|---|---|
| `updates` | 节点级 | 谁干完了、改了哪格 | **车间广播**：某工位完工，"我改了这几个字段" |
| `values` | 节点级 | 每一步后的**完整 state** | **整张工单拍照**：每步拍一张全貌 |
| `messages` | **token 级** | 模型**逐字**吐字 | **打字机**：一个字一个字蹦 |
| `astream_events` | **事件级** | token + 工具开始/结束全记录 | **导演监视器**：每句词、每个动作都带时间戳 |

> 把一次"图执行"当一部电影：**updates 是分镜摘要**（谁出场做了什么），**values 是每一帧完整截图**，**messages 是角色逐字念台词**，**events 是带导演注释的完整记录**。

**选型口诀（今天最值钱的一句话）**：

> 🎯 **调结构用 updates，看全貌用 values，做产品用 messages，做监控用 events。**

前三种都是你熟悉的同步写法，第四种 `astream_events` 一上来就是 `async`——**如果你没学过异步，会直接卡死在这一行 `async for`**。所以这篇在讲第四种之前，先补一章异步零基础课（第四节），别跳过。

* * *

## 一、看法① `updates`：谁改了什么（车间广播）

**默认模式**。D1–D3 你跑 `graph.stream()` 看到的就是它：**每个节点执行完，吐一个 `{节点名: 该节点更新的字段}`**。

```python
for event in graph.stream(TEST_INPUT, stream_mode="updates"):
    print(event)
```

```fallback
{'agent': {'messages': [AIMessage(tool_calls=[add(15, 27)])]}}     # agent 决策要调 add
{'tools': {'messages': [ToolMessage(content='42')]}}               # tools 执行并回填
{'agent': {'messages': [AIMessage(tool_calls=[multiply(42, 2)])]}} # agent 看到 42，再调 multiply
{'tools': {'messages': [ToolMessage(content='84')]}}               # tools 再回填
{'agent': {'messages': [AIMessage(content='(15+27)×2 = 84')]}}     # agent 给最终答案
```

**关键点**：每个 event 只有**本节点的增量**——它不携带其他字段，所以轻、干净、一眼能定位"谁干了什么"。

**用途**：**调试图结构**。图跑歪了，看 updates 就知道是哪个节点改了不该改的字段、哪个节点压根没执行。

> **类比**：车间广播只在**完工时**响一声："3 号工位完成，改了 `result` 字段。" 你不听全程，只听广播就能知道流水线进行到哪、每步动过什么。

* * *

## 二、看法② `values`：每一步的完整快照（工单拍照）

updates 给你**增量**，values 给你**全量**——**每一步执行完之后，整个 state 长什么样**。

```python
for i, snap in enumerate(graph.stream(TEST_INPUT, stream_mode="values")):
    print(f"第{i}步 消息数={len(snap['messages'])}  最后一条={snap['messages'][-1].__class__.__name__}")
```

```fallback
第0步 消息数=1  最后一条=HumanMessage      # 初始：只有用户问题
第1步 消息数=2  最后一条=AIMessage         # agent 决策（带 tool_calls）
第2步 消息数=3  最后一条=ToolMessage       # tools 回填
第3步 消息数=4  最后一条=AIMessage         # agent 再决策
第4步 消息数=5  最后一条=ToolMessage       # tools 再回填
第5步 消息数=6  最后一条=AIMessage         # 最终答案
```

**注意第一行是"第 0 步"**——values 模式从**初始 state** 就开始吐，比 updates 多一个起点快照。

**用途**：**理解状态演变**。看着 `messages` 列表怎么从 1 条长到 6 条，`add_messages` reducer 有没有生效、每次加了什么，用 values 最直观。

### updates vs values：一张表说清

| | updates | values |
|---|---|---|
| 内容 | 本节点的**增量** | 执行后的**完整 state** |
| 每条多大 | 小（只含改动字段） | 大（全字段，消息多了会很长） |
| 有几条 | 等于"有更新的节点数" | 等于"节点数 + 1"（含初始） |
| 适合 | 调结构：谁改了什么 | 看演变：状态怎么一步步长 |

> **类比**：updates 是**车间广播**，values 是**整张工单拍照**。广播只说改动，照片连没动过的格子都拍下来了——信息全，但张张都大。values 模式打印多了会刷屏，实际用的时候只挑你想看的字段打（比如只打 `len(messages)`）。

* * *

## 三、看法③ `messages`：逐字念台词（打字机地基）⭐

### 3.1 它和前两个最大的不同：按 token 吐，不按节点吐

`updates` / `values` 都是**节点执行完才吐一次**。`messages` 模式在 **LLM 生成的过程中**就吐——模型每生成一个 token，就 yield 一次 `(AIMessageChunk, metadata)`。

```python
for chunk, metadata in graph.stream(TEST_INPUT, stream_mode="messages"):
    if chunk.content:                       # 过滤空 chunk
        print(chunk.content, end="", flush=True)   # flush=True：立即刷到屏幕
print()
```

`flush=True` 很重要——print 默认有缓冲，不加它，逐字效果会变成一次性蹦出来。

### 3.2 为什么 chunk 经常是"空的"

**模型在工具调用阶段输出的不是文字，是参数 JSON**。它"想"调 `add(15, 27)` 时，流出来的是 `{"a": 15, "b": 27}` 这种结构——`content` 是空的，文字在 `tool_calls` 字段里。

```fallback
  模型在干什么            流出的内容            chunk.content
  ─────────────────      ──────────────       ───────────
  决策要不要调工具         只有角色标记            空
  输出工具参数            {"a":15,"b":27}       空（参数不进 content）
  生成正式回答            "(15+27)×2 = 84"      有字 ✅
```

所以循环里必须 `if chunk.content:` 过滤——**不然你会打出一堆空行**。

> ⚠️ 顺带解答一个常见困惑：`stream_mode="messages"` 只吐 **LLM 生成的 token**。工具执行阶段（`ToolNode` 在跑代码）**没有流**——那是代码在算，不是模型在说话。

### 3.3 `metadata["langgraph_node"]`：这一句是谁说的

每个 chunk 还带一个 metadata 字典，里面最有用的是 `langgraph_node`——告诉你**这个 token 来自哪个节点**：

```python
for chunk, metadata in graph.stream(TEST_INPUT, stream_mode="messages"):
    if chunk.content:
        print(f"[{metadata['langgraph_node']}]", chunk.content, end="")
```

单 Agent 图里它永远是 `"agent"`，看着没意思。**但到了 W7 多 Agent 图（supervisor + 多个 worker），每句话是谁说的全靠它区分**——"当前是谁在说话"是个必须能回答的问题。

### 3.4 🎯 封装成打字机函数（W10 直接搬）

`messages` 模式是**做产品**用的——把 `print` 换成 `yield`，就是一个生成器；W10 用 FastAPI SSE 把它推给前端，浏览器就能逐字显示了：

```python
def stream_answer(graph, user_input: str):
    """打字机函数：逐字 yield 模型的回答。W10 接 FastAPI SSE 直接用"""
    for chunk, metadata in graph.stream(
        {"messages": [("user", user_input)]}, stream_mode="messages"
    ):
        if chunk.content:
            yield chunk.content
```

```python
# 用法（生成器逐字消费）：
for word in stream_answer(graph, "讲个笑话"):
    print(word, end="", flush=True)
```

> **类比**：updates 是车间广播、values 是拍照，`messages` 是**角色在台上逐字念台词**——只有它能还原"说话的过程"，所以打字机、直播、聊天界面全用它。

* * *

## 四、异步第一课（为 `astream_events` 铺路）

第四种看法 `astream_events` 是 `async` 的。**如果你没学过异步，`async for` 就是天书。** 这一节从零讲——不深入语言细节，只讲够用、且讲"为什么"。

### 4.1 先感受问题：同步代码的"干等"

你写的普通 Python 代码是**同步**的：一行执行完才执行下一行。遇到要**等待**的操作（网络请求、读文件、API 调用），整个程序就**卡住干等**：

```python
import time

print("准备调模型")
time.sleep(3)          # 假装等 API 3 秒 —— 这 3 秒里程序啥也干不了
print("拿到结果")
```

**`time.sleep(3)` 是"阻塞式等待"**：进程停摆 3 秒，期间连另一件不相关的事（比如给另一个用户回话）都做不了。对命令行脚本无所谓；但对"一个服务同时服务很多用户"的程序，这是灾难——**一个用户慢，所有人排队**。

> **类比**：你去银行柜台办业务，柜员全程只服务你，你填表 3 分钟他就在旁边干等 3 分钟——**后面排队的全被你堵住了**。这是"同步柜台"。

### 4.2 异步的答案：等的时候，先干别的

异步（async）解决的就是"等待时浪费 CPU"：**一个任务在等 IO 的时候，先把控制权交出来，让程序去跑别的任务；等的结果回来了，再回来继续。**

> **类比**：咖啡馆的做法——**下单后给你个号牌，你先找座等着，咖啡好了叫号**。咖啡师不会因为你那杯要等 3 分钟就停手，他同时在冲别人的咖啡。**一个咖啡师服务很多人，靠的不是分身，是"不等"**——每个订单等待时他都在做别的事。

**网络 IO（调 API、查数据库、等工具结果）是典型的"咖啡等待"**——大部分时间花在网络上，CPU 其实闲着。异步就是把这些"闲着等"的时间利用起来。

### 4.3 三件套：`async def` / `await` / `asyncio.run()`

Python 异步的最小语法就三个：

```python
import asyncio

async def make_coffee():            # ① async def：定义一个"协程"
    print("开始冲泡")
    await asyncio.sleep(1)          # ② await：挂起，把控制权交出去
    print("咖啡好了")
    return "☕"

asyncio.run(make_coffee())          # ③ asyncio.run()：唯一的启动入口
```

逐个拆：

**① `async def` 定义的不是普通函数，是"协程（coroutine）"**——一个**可以暂停、可以恢复**的函数。普通函数一旦开始就必须跑完；协程可以"跑到一半挂起，等会儿再回来"。

```python
async def hello():
    return "hi"

f = hello()          # ⚠️ 注意：调用不执行！返回的是一个"协程对象"
print(f)             # <coroutine object hello at 0x...>
# 协程对象要被执行，得放进事件循环（asyncio.run 干的事）
```

**② `await` 是暂停点**：`await asyncio.sleep(1)` 的意思是"我要等 1 秒，**这 1 秒里控制权交给事件循环，你去忙别的**；1 秒到了叫我回来"。

**③ `asyncio.run(coro)` 是唯一的入口**：它创建事件循环 → 把协程放进去跑 → 跑完收尾。**普通脚本里所有异步代码，最终都要靠它启动**（后面 `astream_events` 也一样）。

### 4.4 事件循环：那个"咖啡师"

协程能"暂停/恢复"、多个任务能"交错推进"，靠的是一个藏在后台的**事件循环（event loop）**：

```fallback
  事件循环 = 那个不停巡视的咖啡师/前台
    │
    ├─ 任务 A 说："我要等网络响应"  → 好，你去等着，我记下你
    ├─ 任务 B 说："我要算点东西"    → 好，你先算（CPU 不空转）
    ├─ 任务 A 的响应到了！           → 好，叫你回来继续
    └─ 所有任务都完成了             → 事件循环收工
```

`asyncio.run()` 就是**"开店打烊"**：开店（建事件循环）→ 让里面的任务自己交错跑 → 全部完工 → 打烊（关循环）。事件循环自己不写代码，但你要理解"await 的背后有个调度者在盯着"——这解释了为什么 await 时程序没死，而是在干别的。

### 4.5 `async for`：异步迭代

第四个语法点，也是 `astream_events` 直接用的：**`async for`（异步迭代）**。

普通 `for` 从迭代器里**同步取**元素——取不到就一直等，期间程序卡住。`async for` 从**异步迭代器**里取元素——**等下一个元素时，当前协程挂起，事件循环去干别的**：

```python
# 同步 for：每取一个元素，都要"排队等"
for item in normal_iterator:
    ...

# 异步 for：取元素的过程可以"等"，等待时不阻塞别人
async for item in async_iterator:
    ...
```

> **类比**：同步 `for` 像**在窗口排队领号**——前面没人发号你就干站；`async for` 像**坐下等叫号**——等着的时候你还能刷手机（事件循环去跑别的任务）。

### 4.6 一个能跑的对照实验

把 4.3 的例子扩展成"两件事同时进行"，感受异步的威力：

```python
import asyncio

async def make_coffee():
    print("[咖啡] 开始冲泡")
    await asyncio.sleep(2)              # 模拟：泡咖啡要等 2 秒
    print("[咖啡] 完成")
    return "咖啡好了"

async def main():
    task = asyncio.create_task(make_coffee())   # 把协程"派给"事件循环，不等它
    print("[主] 咖啡开始煮了，我先去干点别的")
    await asyncio.sleep(1)              # 主任务也等 1 秒（但它不阻塞咖啡）
    print("[主] 干完了，回来等咖啡")
    result = await task                 # 等咖啡任务结束，拿结果
    print("[主] 拿到：", result)

asyncio.run(main())
```

输出顺序：

```fallback
[咖啡] 开始冲泡
[主] 咖啡开始煮了，我先去干点别的
[主] 干完了，回来等咖啡
[咖啡] 完成
[主] 拿到： 咖啡好了
```

**看时间线**：咖啡任务在第 0 秒启动后 `await sleep(2)` 挂起 → 主任务趁机跑了 1 秒 → 第 1 秒主任务也挂起等咖啡 → 第 2 秒咖啡完成，唤醒等待者。**两件事交错推进，总耗时 2 秒而不是 2+1=3 秒**——这就是"等待时不浪费"。

> 💡 对比同步写法：两个 `time.sleep` 串行就是 3 秒。异步不是"跑得更快"，是**"等得更聪明"**——把等待的时间重叠了。

### 4.7 小结：异步四板斧（够用版）

```python
# 1. async def：定义一个"可暂停"的协程
# 2. await：在协程里挂起，等结果的同时让事件循环干别的
# 3. async for：异步迭代——等下一个元素时挂起不阻塞
# 4. asyncio.run(coro)：普通脚本启动异步的"唯一入口"
```

**你没学的部分还多（Task 并发、gather、Future…），但用 `astream_events` 只需要这四板斧**——它把复杂的调度全藏起来了，你要做的只是 `async def` 包一层 + `async for` 收事件 + `asyncio.run()` 启动。

* * *

## 五、看法④ `astream_events`：导演监视器（站在异步上）

### 5.1 为什么它没有同步版

看名字：`astream_events`——**开头的 `a` = async**。它是异步 API，**没有对应的同步 `stream_events`**。

为什么设计成异步？因为事件流是**持续不断产生事件**的管道——token 在来、工具在跑、节点在切换。用 `async for` 接收时，**等下一个事件期间协程挂起、不阻塞其他任务**——这对"一个服务同时跑多个 Agent 会话"是刚需（W10 的 FastAPI 就是异步服务器）。**它天生为"并发场景"设计，所以 API 也是异步的。**

### 5.2 能抓到什么：三类事件

updates / values / messages 都只给你"结果"。`astream_events` 给你**过程里的一切**——连工具什么时候开始、什么时候结束都抓得到：

```python
async def run_events():
    async for ev in graph.astream_events(TEST_INPUT, version="v2"):
        if ev["event"] == "on_chat_model_stream":       # 模型吐 token
            data = ev["data"]["chunk"].content
            if data:
                print(data, end="", flush=True)
        elif ev["event"] == "on_tool_start":            # 工具开始执行
            print(f"\n[tool开始] {ev['name']}")
        elif ev["event"] == "on_tool_end":              # 工具执行完
            print(f"[tool结束] {ev['name']} → {ev['data'].get('output')}")

asyncio.run(run_events())
```

你会看到事件**交错出现**：

```fallback
[token] 让我帮你算
[tool开始] add
[tool结束] add → 42
[token] 然后乘以2
[tool开始] multiply
[tool结束] multiply → 84
[token] 所以结果是 84
```

**同一个循环里，token 流和工具事件按真实时间顺序混在一起**——这就是"导演监视器"：全片过程带时间戳地重放。

### 5.3 两个必写参数 / 概念

- **`version="v2"` 必须显式写**：不写会 `DeprecationWarning`（v1 要被淘汰了，固定 v2）。
- **`ev` 是个大字典**，常见字段：`event`（事件名）、`name`（谁触发的，如工具名）、`data`（内容）、`metadata`（含 `run_id`）。
- **`metadata["run_id"]` 是这次运行的唯一编号**——把多个事件串成一条"trace"就靠它。W7 Harness 的 Trace 模块（记录每个节点耗时 / token）的原料就是它。

### 5.4 🎯 什么时候用它，什么时候别用

```
  messages 模式：只要"模型说的话" → 打字机 → 够用，别用 events
  astream_events：要"过程全记录" → 埋点/计时/监控/trace → 用它

  ⚠️ events 是杀鸡用的牛刀——它多一层事件解析，
     打字机场景用 messages 更简单直接。
```

> **选型收口**：**做产品用 messages（简单够用），做监控用 events（全量可观测）。**

* * *

## 六、完整可运行代码（`W5/lg_d4_stream_modes.py`）

自包含版（复制 D3 构图，避免 import 路径问题）。`api_key` 换成你的读取方式即可：

```python
"""W5-D4: 一张 ReAct 图，四种看法（updates / values / messages / events）"""
import os, asyncio
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# ── 1. 复用 D3 的 ReAct 图 ──
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

llm = ChatOpenAI(
    model="deepseek-v4-flash",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)
llm_with_tools = llm.bind_tools(tools)

def agent_node(state: AgentState) -> dict:
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

builder = StateGraph(AgentState)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode(tools))
builder.add_edge(START, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")
graph = builder.compile()

TEST_INPUT = {"messages": [("user", "帮我算 (15 + 27) 乘以 2")]}

# ── 看法①：updates（看结构）──
print("=== ① updates：谁改了什么 ===")
for event in graph.stream(TEST_INPUT, stream_mode="updates"):
    print("  ", event)

# ── 看法②：values（看全貌）──
print("\n=== ② values：每一步完整 state ===")
for i, snap in enumerate(graph.stream(TEST_INPUT, stream_mode="values")):
    print(f"  第{i}步 消息数={len(snap['messages'])}  "
          f"最后一条={snap['messages'][-1].__class__.__name__}")

# ── 看法③：messages（打字机）──
print("\n=== ③ messages：token 级打字机 ===")
for chunk, metadata in graph.stream(TEST_INPUT, stream_mode="messages"):
    if chunk.content:                 # 过滤空 chunk（工具调用阶段没文本）
        print(chunk.content, end="", flush=True)
print("\n")

# ── 看法④：astream_events（最细粒度，异步）──
print("=== ④ astream_events：token + 工具事件 ===")
async def run_events():
    async for ev in graph.astream_events(TEST_INPUT, version="v2"):
        if ev["event"] == "on_chat_model_stream":
            data = ev["data"]["chunk"].content
            if data:
                print(data, end="", flush=True)
        elif ev["event"] == "on_tool_start":
            print(f"\n  [tool开始] {ev['name']}")
        elif ev["event"] == "on_tool_end":
            print(f"  [tool结束] {ev['name']} → {ev['data'].get('output')}")

asyncio.run(run_events())             # 🎯 异步代码的唯一启动入口
print("\n=== 完成 ===")
```

**看懂最后一段的异步骨架**——它就是我们第四节学的四板斧的完整落地：

```python
async def run_events():                       # ① async def：协程
    async for ev in graph.astream_events(...):# ③ async for：异步迭代收事件
        ...
asyncio.run(run_events())                    # ④ asyncio.run：启动入口
# （② await 被藏在 async for 内部——等下一个事件时自动挂起）
```

* * *

## 七、选型总表 + 多模式组合

### 7.1 四看法对比表（背这张）

| stream_mode | 输出粒度 | 输出内容 | 什么时候用 |
|---|---|---|---|
| `"updates"`（默认） | 节点级 | `{节点: 更新dict}` | **调结构**：哪个节点改了什么 |
| `"values"` | 节点级 | 完整 state 快照 | **看全貌**：状态怎么一步步演变 |
| `"messages"` | **token 级** | `(AIMessageChunk, metadata)` | **做产品**：打字机、SSE、实时显示 |
| `astream_events(v2)` | 事件级（最细） | 全部事件（token/工具/节点） | **做监控**：可观测性、trace、埋点 |

### 7.2 还想两个都要？多模式组合

`stream_mode` 可以传一个**列表**，同时拿到两种粒度——每步 yield 的是 `(mode, data)` 元组：

```python
for mode, data in graph.stream(TEST_INPUT, stream_mode=["updates", "messages"]):
    if mode == "updates":
        print("[结构]", data)          # 节点增量
    else:
        # mode == "messages"：token 流（这里不打印，太乱）
        pass
```

> ⚠️ **组合模式必须解包元组**——`for mode, data in ...`，只写一个变量会直接 `TypeError`。

### 7.3 顺带认识两个"了解即可"的参数

```python
# stream_mode="custom"：节点里手动吐自定义数据（进度条/自定义事件）——🟡 了解
# subgraphs=True：嵌套图（W6 子图）时把子图事件也吐出来——W6 见
```

* * *

## 八、踩坑记

### 坑 1：`messages` 模式 content 全空 🔴

- **现象**：打字机打了半天，全是空行
- **原因**：模型在 tool_calls 阶段输出的是**参数 JSON**，不是文字——`content` 为空
- **解法**：`if chunk.content:` 过滤
- 💡 工具调用阶段本来就没有文本 token，这不是 bug 是**模式特点**

### 坑 2：`astream_events` 在普通函数里跑报错 ⚠️

- **现象**：`await` / `async for` 在普通函数里直接语法错误或 `RuntimeError`
- **原因**：它是异步 API，**只能在 `async` 函数里用**
- **解法**：`async def run_events():` 包一层 + `asyncio.run(run_events())`

### 坑 3：`astream_events` 报 `DeprecationWarning` ⚠️

- **现象**：控制台黄色警告
- **原因**：没指定 version，默认走要被淘汰的 v1
- **解法**：固定写 `version="v2"`

### 坑 4：`await` 只能在 `async` 函数里用 ⚠️

- **现象**：`SyntaxError: 'await' outside function` 或 `RuntimeError`
- **原因**：混淆了"协程"和"普通函数"的边界
- **解法**：记住分层——`async def`（能 await 的层）→ 普通 def（不能）；跨层要用 `asyncio.run()` 或包一层 async

### 坑 5：调 `async def` 函数但它"没执行" ⚠️

- **现象**：函数里的 print 一个都没打
- **原因**：**调用协程函数只创建协程对象，不执行**——必须交给事件循环（`asyncio.run` / `await` / `create_task`）
- **解法**：所有异步入口最终都要 `asyncio.run(...)` 启动

### 坑 6：以为 stream 会自动进浏览器 ⚠️

- **现象**：本地跑出打字机，浏览器却啥也没有
- **原因**：LangGraph 的 stream 是 **Python 侧**的输出——它把 token 吐给你了，但没送到前端
- **解法**：W10 用 FastAPI SSE 包一层 `stream_answer`，浏览器才能逐字收到

### 坑 7：多模式组合忘了解包元组 ⚠️

- **现象**：`TypeError: cannot unpack non-iterable ...`
- **原因**：`stream_mode=["updates", "messages"]` 每步 yield `(mode, data)`
- **解法**：`for mode, data in graph.stream(...)`

### 坑 8：`values` 模式刷屏 ⚠️

- **现象**：完整 state 太长，终端被消息列表淹没
- **解法**：只打印关心的字段（`len(messages)`、最后一条的类名），别整个 state 直接 print

* * *

## 九、速查卡片（复习直接看这）

```python
# ===== 四种看法 =====
for event in graph.stream(TEST_INPUT, stream_mode="updates"):     # ① 谁改了什么
    print(event)                                                   # {节点: 更新}

for i, snap in enumerate(graph.stream(TEST_INPUT, stream_mode="values")):
    print(f"第{i}步 消息数={len(snap['messages'])}")              # ② 完整 state

for chunk, metadata in graph.stream(TEST_INPUT, stream_mode="messages"):
    if chunk.content:                                              # ③ token 级
        print(chunk.content, end="", flush=True)                   #    打字机

async def run_events():                                            # ④ 事件级（异步）
    async for ev in graph.astream_events(TEST_INPUT, version="v2"):
        if ev["event"] == "on_chat_model_stream" and ev["data"]["chunk"].content:
            print(ev["data"]["chunk"].content, end="", flush=True)
        elif ev["event"] == "on_tool_start":
            print(f"\n[tool开始] {ev['name']}")
asyncio.run(run_events())

# ===== 选型口诀 =====
# 调结构用 updates，看全貌用 values，做产品用 messages，做监控用 events

# ===== 打字机封装（W10 接 SSE）=====
def stream_answer(graph, user_input: str):
    for chunk, metadata in graph.stream(
        {"messages": [("user", user_input)]}, stream_mode="messages"
    ):
        if chunk.content:
            yield chunk.content          # yield 出去，W10 用 FastAPI SSE 推送

# ===== 异步四板斧 =====
# async def  定义可暂停的协程（调用不执行！）
# await      挂起点：等的时候把控制权交给事件循环
# async for  异步迭代：等下一个元素时不阻塞
# asyncio.run(coro)  普通脚本唯一的启动入口
```

* * *

## 十、一句话总结

**D3 造了机器，D4 学会了"看"机器——同一张 ReAct 图，四种看法粒度从粗到细：`updates` 是节点级增量（调结构）、`values` 是完整快照（看演变）、`messages` 是 token 级流式（做打字机）、`astream_events` 是事件级全记录（做监控）。**

**而本篇真正多学的一块，是异步**：

```fallback
  同步 = 银行柜台：一个人办事，全队等他
  异步 = 咖啡馆叫号：等咖啡的时候，咖啡师在冲别人的

  async def（可暂停的函数）
  await（等的时候让出控制权）
  async for（异步迭代：等下一个事件不阻塞）
  asyncio.run()（启动入口）
```

`astream_events` 用到的就这四板斧——**你没学的不是"async 语法"，是"等待时不浪费"的心智模型**。一旦接受"await = 把控制权交出去，好了叫我"，异步代码读起来就和同步一样顺了。

> 回头看这条线的落点：`messages` 模式 + 今天的 `stream_answer` 生成器，是 **W10 打字机**的第一块地基；`metadata["langgraph_node"]` 是 **W7 多 Agent**"谁在说话"的答案；`run_id` 是 **W7 Trace** 模块的原料。**今天学的每一种"看法"，都在给后面某个零件备料。**

**下一篇：D5 State Schema 进阶**——从 TypedDict 升级到 Pydantic（带校验）、嵌套字段、`Annotated` reducer 的原理。为 Day6 的 MemoryManager 图（多字段复杂状态）打底，也为 W6 的 Checkpointer / HITL 做铺垫。

* * *

### 🔴 魔鬼代言人（跑完四种模式，回答自己三个问题）

① 为什么 `messages` 模式在"调 add 工具"那一段会有一段空白？

**提示**：模型那时候在输出什么？——它在输出 `{"a": 15, "b": 27}` 参数 JSON，不是人话。**没有文本 token，当然空白。**

② 从 `values` 的输出里，你能数出 messages 一共累加了几条？和 `updates` 的节点数对得上吗？

**对照**：values 从"第 0 步初始快照"开始吐，节点每执行一次 +1 条消息。数出来应该是 6 条消息（Human + 2×AI + 2×Tool + 最终 AI）…… 等等，**别急着信这个数**——去数你自己的输出。

③ 哪个数字是你"算"出来的、哪个是"拍脑袋"的？

**这就是今天想让你养成的习惯**：跑完程序，先看真实输出，再对照自己的预期。**输出和预期对不上时，先怀疑"拍脑袋"的那一半**——通常是你对图的理解有盲区，而不是框架出错。

* * *
