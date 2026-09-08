---
description: ""
title: "给图装上存档系统"
draft: false
date: "2026-09-08T13:29:58+08:00"
slug: "checkpointer"
categories:
 - LangGraph
tags:
 - Memory
image: ""
---


# W6-D1/D2 · 给图装上存档系统：Checkpointer、thread_id 与 SqliteSaver

## 零、先看清我们要补的三个缺口

W5 结束时你手里有一张能跑的图：会路由、会调工具、会压缩记忆。但它有个致命属性——**每次 `invoke` 都从空 state 开始，跑完 state 就丢**。

W5 里你其实已经撞过这个缺口三次，只是当时没有合适的工具补它：

| 现象 | 你当时怎么绕的 | 真正的缺口 |
|---|---|---|
| D1 的 `turns` 累加实验：第二次要把上次输出**手动喂回去**才继续涨 | 手动传 state | 图不记得上次跑过什么 |
| D6 的 `memory_store = {}` 能跨轮记住"美式咖啡" | 靠进程内全局变量**碰巧还活着** | 记忆住在图外面，进程一关就没 |
| D6 的 agent 能直接"记住/删除"东西，没人拦 | 没做，反正单agent | 没有让图**中途停下来问人**的位置 |

> 类比：W5 的图像一台**每次开机都重置的计算器**——算完结果抄走，内存清空。W6 是给它装上**存档系统 + 确认按钮 + 多开窗口**，让它从"玩具"变成"产品"。

W6 的三件事对着这三个缺口：**记住过去**（Checkpointer 持久化，D1–D2）、**敢问人**（HITL 审批，D3–D4）、**能拆能并行**（子图 + Send，D5 起）。今天开篇讲第一件：**给图装上存档系统。**

---

## 一、先搞清一件事：checkpoint ≠ 记忆库

这是学 Checkpointer 之前必须先钉死的区分，否则后面所有用法都会长歪。

### 1.1 记忆其实分三层

| 层 | 是什么 | 谁在管 | 例子 |
|---|---|---|---|
| ① 模型上下文窗口 | 这一轮模型"看得见"什么 | 你手动拼 | W3 的 buffer 取最近 N 条塞进 prompt |
| ② 会话状态存档 | 这个会话跑到哪、state 是什么 | **框架（checkpointer）** | 今天按 thread_id 存的 checkpoint |
| ③ 长期事实库 | 真正"学到"的知识 | 你的 LTM | W3 的 Chroma 落盘 |

W3 的短期 buffer 属于 **①**——它是你给模型喂话的"原料"；今天的 checkpointer 属于 **②**——它管的是"整个会话的状态怎么跨轮恢复"；W3 的 Chroma 属于 **③**——跨会话的长期知识。

**它们不是替代关系。今天的 checkpointer 并没有取代你的短期记忆，它取代的是你"手动跨轮攒消息"那一整套动作。**

### 1.2 checkpoint 和记忆库的正面对比

| | checkpoint（今天） | 记忆库（W3 的 Chroma） |
|---|---|---|
| 存什么 | **整张图的 state**：messages 全列表、counter、summary、任何字段 | 提炼出的**重要事实**（"用户喜欢美式咖啡"） |
| 粒度 | 会话级——一个对话的完整现场 | 跨会话级——长期知识，任何会话都能查 |
| 类比 | **游戏存档**（整个游戏现场：等级、背包、剧情） | **现实笔记本**（你记下来的重要事情） |
| 生命周期 | 跟着会话走（thread_id） | 独立持久存储 |

一句话区分：**checkpoint 回答"我们刚才聊到哪了"，记忆库回答"这个用户我们了解多少"。**

> ⚠️ 最常见的误解就是把 checkpoint 当长期记忆用——指望它跨会话检索用户信息。它做不到，也不该做。会话现场和长期知识是两码事。

### 1.3 那 W3 的 buffer 和 checkpoint 到底差在哪

你可能会问：W3 那个全局 buffer 不也能跨轮记住"小明"吗？**能，但它是你在应用层"模拟"出来的。** 三个字的差别——**隔离 + 抽象**：

| 维度 | W3 buffer（全局变量） | checkpointer + thread_id |
|---|---|---|
| 归属层 | 应用层，你的类里 | 图运行层，LangGraph 引擎里 |
| 存的内容 | 一份消息列表（你自己 `add` 进去的） | **整个图的 state 快照**（messages + 所有字段 + 跑到哪一步） |
| 会话隔离 | ❌ 全局一份，谁用都是它 | ✅ 每个 thread_id 一份，互不干扰 |
| 跨 invoke 记住 | 靠进程活着 + 你手动 add | 框架按 thread_id 自动续档 |
| 进程重启 | 没了 | MemorySaver 也没了；**换一行 SqliteSaver 就落盘** |
| 落盘成本 | 自己写 load/save、管文件路径 | 换一个 saver 实现，图代码不动 |

单进程单用户时，两者效果一样——W3 的 buffer 本质上是在"假装整个进程只有一个会话"。**一旦要多会话、要重启、要断点恢复，手搓版就撑不住了。**

> 类比：W3 的 buffer 是你手边一本**公用笔记本**——一个人记没问题，两个人同时用就串行，进程一关本子烧了。checkpointer 是**游戏存档槽**——每个玩家有自己的档，读自己的档就记得上次玩到哪；不选存档就接不上。MemorySaver 是"内存卡"，想长久保存就换"硬盘卡"（SqliteSaver），游戏机（图代码）不用改。

---

## 二、接入只差一行：`compile(checkpointer=...)`

先看那个最痛的场景：

```python
graph.invoke({"messages": [("user", "我叫小明，记住我")]})
graph.invoke({"messages": [("user", "我叫什么？")]})   # ❌ 不记得
```

每次 `invoke` 都从空 state 开始 → 跑完丢弃。图是一台"无状态函数"，像 `f(x) = y`：输入相同，它不记得上一次。

**Checkpointer 的答案**：在图执行过程中，每个节点跑完后**自动存一份 state 快照（checkpoint）**。下一次 invoke 不是从零开始，而是从上次的 checkpoint **续跑**：

```fallback
无 checkpoint:  invoke → 跑 → 结果 → 【state 丢掉】
有 checkpoint:  invoke → 每个节点后存 checkpoint → 下次从存档续跑
```

接线方式只有一行：

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()                                  # ① 注意是实例，不是类名
graph = builder.compile(checkpointer=checkpointer)             # ② 编译时传入
```

**图拓扑一行都不用改**——节点、边、reducer 全不变。checkpoint 是后台自动行为，不需要你手动"存"。

两个必须记住的细节：

- **你 W5 学的 `Annotated[list, add_messages]` reducer 依然生效**。checkpoint 保存的是"reducer 合并后"的 state——messages 该追加还是追加，存档里存的是追加完的结果。
- **保存粒度是"每个节点执行后"**。所以每个节点边界都留了一个存档点，将来能回放每一步（W6-D4 的时间旅行就靠这个）。

图拓扑在概念上多了一层看不见的"后台存档"：

```fallback
START → agent ⇄ tools（循环）→ END
         │
         └─ 后台：每个节点跑完 → 按 thread_id 存一份 checkpoint
```

---

## 三、`thread_id`：存档槽位编号

同一张图可能服务很多会话。图怎么知道这次续的是哪个对话？靠 `config` 里的 **`thread_id`**：

```python
config = {"configurable": {"thread_id": "demo-1"}}   # 会话身份证
graph.invoke({"messages": [...]}, config)            # 每次 invoke 都要带
```

> 类比：游戏机的**存档槽**。1 号槽存"小明会话"，2 号槽存"小红会话"。不带槽位号 = 每次开新档，永远失忆；每次带同一个号 = 接续存档。

**两个最容易踩的失忆原因**（表现一样，根因不同，要会分辨）：

1. **换了一个 thread_id**——等于主动开了个新档，当然是空的。这是设计行为，不是 bug。
2. **完全忘了传 config**——某些版本会直接报错要求 `thread_id`，某些版本当成新会话处理。**判断方法：跑一次看它是抛错还是失忆，别猜。**

---

## 四、四组对照实验：怎么证明它真的记住了

D1 的核心不是"接上一行代码"，而是**设计证据链**——光看"第二次答出小明"是不够的，那可能是模型从当前问题里瞎猜的。真正的铁证是 `get_state` 里的**消息总数**。

```python
"""W6-D1: 给 W5 的 ReAct 图接上 MemorySaver —— 让图记住过去"""
import os
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# ── 1. 原样复用 W5-D3 的 ReAct 图 ──
class AgentState(TypedDict):
    messages: Annotated[list, add_messages]

@tool
def add(a: int, b: int) -> int:
    """计算两个整数之和"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """计算两数之积"""
    return a * b

llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
).bind_tools([add, multiply])

def agent_node(state: AgentState) -> dict:
    return {"messages": [llm.invoke(state["messages"])]}

# 工厂函数：同一份拓扑，想编译几次编译几次（下面要对照组）
def build():
    b = StateGraph(AgentState)
    b.add_node("agent", agent_node)
    b.add_node("tools", ToolNode([add, multiply]))
    b.add_edge(START, "agent")
    b.add_conditional_edges("agent", tools_condition)
    b.add_edge("tools", "agent")
    return b

# ── 2. 关键差异：同一张图，编译成两个版本 ──
graph_cp   = build().compile(checkpointer=MemorySaver())   # 有存档
graph_nocp = build().compile()                             # 无存档（对照组）

# ── 3. 验证 1：有 checkpoint + 同一 thread_id → 跨 invoke 记忆 ──
config = {"configurable": {"thread_id": "demo-1"}}
r1 = graph_cp.invoke({"messages": [("user", "我叫小明，记住我")]}, config)
print("第一次:", r1["messages"][-1].content)

r2 = graph_cp.invoke({"messages": [("user", "我叫什么名字？")]}, config)
print("第二次:", r2["messages"][-1].content)      # 应该能答出"小明"

# ── 4. 验证 2：读档确认——消息真的在累加（这是铁证，不是看回答猜）──
state = graph_cp.get_state(config)
print("消息总数:", len(state.values["messages"]))   # 期望 ≥ 4 = 2 次提问 + 2 次回答

# ── 5. 验证 3：换一个 thread_id → 开新档 → 失忆 ──
config2 = {"configurable": {"thread_id": "demo-2"}}         # 换槽位
r3 = graph_cp.invoke({"messages": [("user", "我叫什么名字？")]}, config2)
print("换 thread:", r3["messages"][-1].content)    # 答不出小明

# ── 6. 验证 4：无 checkpoint 的图，永远从零开始 ──
graph_nocp.invoke({"messages": [("user", "我叫小明")]})
g2 = graph_nocp.invoke({"messages": [("user", "我叫什么？")]})
print("无cp版消息数:", len(g2["messages"]))        # 期望 = 2（只有这一轮的提问+回答）
```

**四组证据各自的职责**：

| 验证 | 证明什么 | 为什么不能只看它 |
|---|---|---|
| 第二次答出"小明" | 跨 invoke 记忆**可能**生效 | 模型也可能从问句里瞎猜 |
| **`get_state` 消息总数 ≥ 4** | ✅ **铁证**：上一次的消息真的留在 state 里被续上了 | — |
| 换 thread_id 答不出 | 隔离生效：存档是按槽位分的 | — |
| 无 cp 版消息数 = 2 | 没有存档机制 → 每次全新开局 | — |

**如果"答出小明"但消息总数只有 2，说明记忆是假的**——模型在编，state 里根本没有历史。这就是为什么验证 2 才是关键：**让代码自己说话，别靠回答猜。**

> 💡 顺带说个"数字洁癖"：`≥ 4` 是**推算**出来的（2 问 + 2 答）。如果模型中途调了工具，会变成 5、6、7……所以跑完以 `get_state` 的**实测值为准**。推算用来设计实验，实测用来下结论。

### 4.1 亲眼看到"每个节点后存一份"

"每节点存快照"这句话是概念，用 D4 学过的 stream 能把它变成眼见为实——**stream 每吐一次，就代表有一个节点刚跑完，也就意味着刚存了一份 checkpoint**：

```python
for event in graph_cp.stream({"messages": [("user", "再聊一句")]}, config):
    print("节点跑完:", list(event.keys()))
```

跑之前先读一次 `len(get_state(config).values["messages"])`，跑完再读一次——**数字的增长量，就是这一轮新增的消息数**。存档点数量 ≈ 这一轮经过的节点数。

这一步的价值在于把"后台自动存档"从黑盒变成可观测：**你能数出它存了几份，而不是相信文档说它存了。**

---

## 五、一张图，两个用户：会话隔离

D1 证明了"能续档"，D2 第一课是"分档"——同一张图，`thread_id` 就是柜子号，checkpoint 按它分桶存储：

```fallback
graph.invoke(..., {"configurable": {"thread_id": "alice-001"}})  → 读写 alice 的桶
graph.invoke(..., {"configurable": {"thread_id": "bob-001"}})    → 读写 bob 的桶
```

两份会话**共用同一套节点逻辑**，但 state 快照各存各的——Alice 说的"我喜欢猫"不会漏进 Bob 的上下文。这就是多用户 Agent 服务的**最小模型：每个用户一个 thread_id**。

```python
"""W6-D2：thread_id 多会话隔离 —— 一张图，两个用户，互不干扰"""
# （构图部分与 D1 完全相同：State / 工具 / agent_node / builder）

graph = builder.compile(checkpointer=MemorySaver())   # 编译时只传一次 checkpointer

alice = {"configurable": {"thread_id": "alice-001"}}
bob   = {"configurable": {"thread_id": "bob-001"}}

graph.invoke({"messages": [("user", "我叫爱丽丝，我喜欢猫")]}, alice)
graph.invoke({"messages": [("user", "我是鲍勃")]}, bob)

# 各自问自己的信息
r_a = graph.invoke({"messages": [("user", "我叫什么？我喜欢什么？")]}, alice)
r_b = graph.invoke({"messages": [("user", "我叫什么？我喜欢什么？")]}, bob)
print("Alice 记得:", r_a["messages"][-1].content)   # 应答出：爱丽丝 / 猫
print("Bob   记得:", r_b["messages"][-1].content)   # 应答出：鲍勃（不知道猫）
```

**判定标准（三种结果三种根因）**：

| 现象 | 判定 | 根因 |
|---|---|---|
| Alice 答出"爱丽丝、猫"，Bob 只答出"鲍勃"、**不知道猫** | ✅ 隔离成功 | — |
| Bob 也知道猫 | ❌ | 两个 invoke 用了**同一个 thread_id**（最常见） |
| 两个都失忆 | ❌ | invoke 时**忘了带 config** |

**Bob 不知道猫不是 bug，是隔离成功。** 这句话是这一节唯一要记住的。

---

## 六、从内存到磁盘：SqliteSaver 与重启恢复

`MemorySaver` 存在**内存**里——进程一关，存档全没。它是练习版。真正跨进程靠 **SqliteSaver**：存进 SQLite 文件，关掉程序再打开，对话还在。

### 6.1 save / load 两个脚本

```python
"""W6-D2：SqliteSaver 落盘 + 重启恢复（save 与 load 分两次运行）"""
# （构图部分与前面完全相同，到 builder 为止，先不要 compile）

from langgraph.checkpoint.sqlite import SqliteSaver

def save():
    """第一次运行：写入会话"""
    with SqliteSaver.from_conn_string("checkpoints.db") as saver:
        graph = builder.compile(checkpointer=saver)
        cfg = {"configurable": {"thread_id": "persist-1"}}
        graph.invoke({"messages": [("user", "我叫小明，住在北京")]}, cfg)
        print("已写入会话")

def load():
    """关掉进程后再运行：读回存档"""
    with SqliteSaver.from_conn_string("checkpoints.db") as saver:
        graph = builder.compile(checkpointer=saver)
        cfg = {"configurable": {"thread_id": "persist-1"}}
        r = graph.invoke({"messages": [("user", "我叫什么？住哪？")]}, cfg)
        print("重启后回复:", r["messages"][-1].content)   # 应答出 小明 / 北京

if __name__ == "__main__":
    save()      # 第一次跑这个
    # load()    # 然后注释掉 save、打开这行，重启进程再跑
```

**重启恢复成功 = W6 第一道硬门槛过了**：数据真的躺在磁盘上，不是靠进程"活着"装的。

### 6.2 三个版本的坑（提前打预防针）

1. **同步 / 异步别混用**。不同 LangGraph 版本里，`SqliteSaver.from_conn_string()` 返回的东西不一样——有的是同步上下文管理器（`with` + `graph.invoke`），有的是异步的（`async with` + `await graph.ainvoke`）。**混用会抛 TypeError**。判断方法：看你的版本报不报错、或让 IDE 提示你它是不是 `AsyncContextManager`。两种写法**不要出现在同一个文件里**。
2. **两个脚本的 `.db` 路径要一致**。相对路径是相对**运行时的 cwd**——在 `W6/` 里跑和在项目根跑，会生成两个不同的 db 文件，"写入成功却读不到"多半是这个原因。
3. **`ImportError` → `pip install aiosqlite`**。SqliteSaver 底层依赖它。

### 6.3 MemorySaver vs SqliteSaver

| | MemorySaver | SqliteSaver |
|---|---|---|
| 存哪 | 内存 | SQLite 文件 |
| 进程重启 | ❌ 全丢 | ✅ 可恢复 |
| 用途 | 练习 / 单进程验证 | 本地持久化 / 生产起步 |
| 切换成本 | — | **只换 saver，图代码不动** |

> 💡 **"今天不落盘"是选了一个内存后端，不是设计上限。** saver 是个接口——现在插内存卡，明天插硬盘卡（SqliteSaver），将来插网络存储（PostgresSaver），游戏机（图代码）始终不用改。

---

## 七、读档三件套：`get_state` / `get_state_history` / `update_state`

落盘跑通后，把这三个 API 一起验掉——它们是你"查看和干预存档"的手，也是 W6-D3 HITL 的底层零件。

```python
def inspect():
    with SqliteSaver.from_conn_string("checkpoints.db") as saver:
        graph = builder.compile(checkpointer=saver)
        cfg = {"configurable": {"thread_id": "persist-1"}}

        # ① get_state：读当前存档
        state = graph.get_state(cfg)
        print("消息总数:", len(state.values["messages"]))   # 应 > 2
        print("下一步节点:", state.next)                      # 正常跑完应为空

        # ② get_state_history：看存档时间线（新 → 旧）
        print("历史 checkpoint 时间线:")
        for i, snap in enumerate(graph.get_state_history(cfg)):
            cid = snap.config["configurable"]["checkpoint_id"]
            n = len(snap.values["messages"])
            print(f"  #{i}  checkpoint_id={cid[:8]}  消息数={n}")

        # ③ update_state：从图外面改档（人注入，不是节点返回）
        graph.update_state(cfg, {"messages": [("user", "顺便记住：我喜欢喝美式咖啡")]})

        # ④ 验证注入真的被 agent 感知
        r = graph.invoke({"messages": [("user", "我喜欢喝什么？")]}, cfg)
        print("注入后回复:", r["messages"][-1].content)   # 应答出 美式咖啡
```

**三个 API 各自的用场**：

| API | 回答什么问题 | 类比 |
|---|---|---|
| `get_state` | 当前这局会话的现场：到哪了、攒了多少消息 | 读当前档 |
| `get_state_history` | 这局会话的所有存档点（时间线，新→旧） | 看所有存档记录 |
| `update_state` | 从图外面直接改 state | 用修改器改档 |

**`state.next` 是什么**：图"下一步要执行的节点"。正常跑完是空的；**图被中断过（比如 HITL 停下来等人）它才非空**——所以它是判断"图是不是卡在半路"的探针。

### 7.1 `update_state` vs 节点返回 dict：外部改档 vs 内部改档

这是这一节最值得想清楚的一组对照：

```fallback
节点 return {"messages": [...]}   →  内部改档：流水线工人自己填工单
graph.update_state(cfg, {...})    →  外部改档：车间主任从外面改工单
```

两者最终都是"更新 state"，但**发起方不同**：一个是节点在执行中自己更新，一个是人在图外面干预。正因为有"外部改档"这个能力，W6-D3 的 HITL 才成立——**agent 跑到一半停下来（此时 `state.next` 非空），人用 `update_state` 塞进指令或修正，agent 再继续。**

---

## 八、踩坑记（对着自查）

| # | 坑 | 解法 |
|---|---|---|
| 1 | 🔴 以为 checkpoint 就是长期记忆库 | 它是会话现场快照；长期知识仍然要向量库（W3 的 Chroma） |
| 2 | 🔴 第二次 invoke 忘了带 config / thread_id 写错 | 表现都是"失忆"。带同一个 `thread_id` 才能续档 |
| 3 | 🔴 同步 / 异步 saver 混用 | `with` 配 `invoke`，`async with` 配 `ainvoke`，二选一不混写 |
| 4 | ⚠️ `MemorySaver` 传成类名 | 要实例：`compile(checkpointer=MemorySaver())` |
| 5 | ⚠️ 改完 builder 没重新 compile | 图是编译产物，改了必须重编 |
| 6 | ⚠️ 两个脚本 `.db` 路径不一致 | 相对路径看 cwd；save/load 用同一个路径 |
| 7 | ⚠️ f-string 里嵌套同型引号 | `f"...{d["k"]}..."` 只在 Python 3.12+ 合法（PEP 701）；通用写法是先取变量再拼 |
| 8 | ⚠️ 只靠"答对了"判断记忆生效 | 答对可能是模型编的。看 `get_state` 的消息总数才是铁证 |
| 9 | ⚠️ 以为换了 saver 要改图代码 | 不用。saver 是接口实现，图拓扑一行不动 |

---

## 九、速查卡片（复习直接看这）

**接线三件套**：

```python
from langgraph.checkpoint.memory import MemorySaver      # 或 .sqlite 的 SqliteSaver
graph = builder.compile(checkpointer=MemorySaver())      # 编译期接入
graph.invoke(payload, {"configurable": {"thread_id": "会话ID"}})   # 运行期带槽位
```

**存档相关 API**：

| 调用 | 用途 |
|---|---|
| `graph.get_state(cfg)` | 读当前存档；`state.values` 是 state、`state.next` 是待执行节点 |
| `graph.get_state_history(cfg)` | 存档时间线，每个节点边界一份，新→旧 |
| `graph.update_state(cfg, {...})` | 外部改档（HITL 底层零件） |

**一句话速记**：

- checkpoint 回答"聊到哪了"，记忆库回答"了解多少"
- thread_id = 存档槽位，**换号 = 开新档 = 失忆（这是设计，不是 bug）**
- 每个节点跑完存一份 → 能回放每一步（时间旅行的前提）
- 内部改档靠节点 `return`，外部改档靠 `update_state`

---

## 十、一句话总结 + 下一篇预告

**W5 的图是无状态函数——每次 invoke 从空 state 开始、跑完就丢；W6 用 Checkpointer 给它装上存档：编译期 `compile(checkpointer=...)` 一行接入，运行期靠 `thread_id` 区分会话，框架在每个节点执行后自动存 state 快照，下次带同一个 thread_id 就续档。MemorySaver 存内存（进程一关就没），SqliteSaver 落盘（重启可恢复），换 saver 不动图代码。** 而验证"它真的记住了"不能看回答——要看 `get_state` 里的消息总数：那是消息被真正续上的铁证。

下一站是 W6-D3 的 **HITL（让 Agent 停下来问人）**。它正好搭在今天这三个零件上：图中断时 `state.next` 非空 → 人介入 → 用 `update_state` 把指令改进去 → agent 接着跑。今天你学会了"读档和改档"，明天就是"什么时候该停下来改"。

**魔鬼代言人**：如果跑通重启恢复之后你冒出"这不就是把 messages 存进数据库吗，我自己用 pickle 也能存"的念头——**技术上没错**，你自己存确实能跨进程。但你要自己解决三件事：按会话分桶（thread_id）、每个步骤留快照（历史回放）、外部安全改档（update_state 的语义与并发）。框架把这三件事做成了**接口**，你只写业务。**判断一个抽象值不值，不是看它能不能被手搓替代，而是看它替你挡掉了多少种你还没想到的失败方式。**

**自查三问**（能答上 = 真懂了）：① 第二次 invoke 答出了"小明"，但 `get_state` 显示消息总数是 2——记忆是真的吗？为什么？② 换 thread_id 和忘带 config，表现一样吗？根因有什么不同？③ `update_state` 和节点 `return {...}` 都在改 state，区别在哪，各用在什么场景？
