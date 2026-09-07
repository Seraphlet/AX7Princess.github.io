---
description: ""
title: "记忆管理画成图"
draft: false
date: "2026-09-07T15:34:25+08:00"
slug: "memory-graph"
categories:
 - LangGraph
tags:
 - Memory
image: ""
---


# W5-D6 · 记忆管理成图：把 W3 的 MemoryManager 翻译成一张状态图

## 零、先立骨架

D1–D5 你攒了一堆零件：图怎么建（D1）、图怎么自己选路（D2）、工具循环怎么进图（D3）、图怎么被观察（D4）、state 怎么上安检（D5）。今天把它们装成 W5 的毕业设计——**把 W3 手写的 `MemoryManager` 类翻译成一张 LangGraph 图**。

先说清楚这篇文章的定位：这是一份**"先跑通"的项目代码**。记忆库用进程内 dict 顶着（W3 你用的 Chroma 落盘暂时没搬进来），召回匹配也是玩具级的子串——这些实现细节后面都会换，不必当最终架构。今天真正值钱的，是**同一套记忆逻辑，从"类的 if/else"变成"图的节点与边"之后，你要怎么想、怎么写、怎么排雷**。

目标形态一句话：**原来 `MemoryManager` 里的"什么时候写长期、什么时候压缩、什么时候就普通聊"这些 if 判断，全部外置成一个 route 节点 + 三条条件边；三个记忆组件变成三个独立节点；对话循环变成 ReAct agent 子图。**

先给总图，后面每一块都对着它讲：

```fallback
                START
                  │
                  ▼
            route_node            ← 原来 MemoryManager 里的 if/else 决策
                  │
        ┌─────────┼──────────┐
        ▼         ▼          ▼
   long_node  compress_node  short_node   ← 三条记忆路径 = 原来三个组件
        │         │          │
        │     summary 写入    │
        │     recent 窗口更新  │
        └────┬────┴────┬─────┘
             ▼         ▼
          agent_node（ReAct）   ← 原来 ChatSession 的对话循环
             │
       tools_condition
        ╱          ╲
  有 tool_calls    无 → END
      ▼
 tools_node（save / recall 长期记忆）
      │
      └──→ 回 agent（循环）
```

> 类比：W3 的 `MemoryManager` 是**手动接线板**——所有线路在类内部用 if 接好，想加一条新线路得拆开面板重新焊；今天的图是**模块化配电箱**——每个记忆策略是一个独立开关，想加新策略就加一个开关、拉一条线，主线路（agent）完全不用动。

---

## 一、先看原版：W3 你写的 MemoryManager 长什么样

图不是凭空来的，它是你 W3 代码的"翻译"。原版长这样（`memory/memory_manager.py`）：

```python
class MemoryManager:
    def __init__(self, window=8, max_tokens=2000, keep=4, llm=None):
        self.stm = ShortTermMemory(window=window, maxtokens=max_tokens)  # 短期
        self.ltm = LongTermMemory()        # 长期（Chroma 向量库）
        self.keep, self.max_tokens, self.llm = keep, max_tokens, llm

    def add(self, msg: dict, persist: bool = False):
        self.stm.add(msg)                                   # 永远进短期
        if persist and msg.get("role") == "user":           # 只有明确要记的才落长期
            self.ltm.add_fact(msg["content"], fid=_fid(msg["content"]))

    def get_context(self, query: str = None) -> list[dict]:
        base = self.stm.context()                           # 取滑动窗口
        if not query:
            return base
        facts = self.ltm.recall(query, k=3)                 # 有 query 才召回长期
        if not facts:
            return base
        facts_msg = {"role": "system", "content": "已知用户长期事实：" + "|".join(facts)}
        # 拼成：系统提示 + 长期事实 + 窗口消息
        return sys_msgs + [facts_msg] + others

    def maybe_compress(self, llm=None):                     # 超阈值自动压缩
        msgs = _auto_compress(self.stm.buffer, llm, max_tokens=self.max_tokens, keep=self.keep)
        if msgs is not self.stm.buffer:                     # 真的压了才写回 buffer
            self.stm.buffer = msgs
        return self.stm.buffer
```

三个方法背后是三条决策线，全埋在类内部：

| W3 方法 | 内部在做什么决策 | 靠什么触发 |
|---|---|---|
| `add` | 这条消息要不要落长期？ | 调用方手动传 `persist=True` |
| `get_context` | 这次对话要不要召回长期事实？ | 有没有传 `query` |
| `maybe_compress` | 窗口超没超阈值、要不要压缩？ | 压缩函数内部的阈值检查 |

**问题就在这里**：决策逻辑是"散在类方法里的 if"。想加一种新记忆策略（比如"语义缓存"），你得改 `MemoryManager` 的内部逻辑；想看清楚"一条消息进来到底走了哪条路"，没有任何可视化——只能靠 print。

今天的图，就是把这些**隐式的 if，变成显式的节点和边**。

---

## 二、翻译方案：类 → 图，四组一一对应

对照表先立起来，写代码时心里一直默念这张：

| W3 类实现 | W5 图实现 | 变化 |
|---|---|---|
| `add` / `get_context` / `maybe_compress` 方法体 | `short_node` / `long_node` / `compress_node` 节点 | 方法调用 → 图节点 |
| 方法里的 if 分支 | `route_node` + 条件边 | 决策逻辑 → 独立路由 |
| `self.stm.buffer`（短期窗口） | `state["messages"]` + `state["recent"]` | 实例属性 → state 字段 |
| `self.ltm`（Chroma） | `memory_store` dict + 两个工具 | 库 → 工具（实现可换回 Chroma） |
| ChatSession 的对话循环 | `agent_node` + `tools_node` 回边 | 手写循环 → ReAct 子图 |

**翻译后最大的收益**（也是整张图的设计思想）：

- **记忆选择被"外置"成路由决策**。想换路由策略（规则路由 / 多 LLM 投票路由），只改 `route_node` 一个函数。
- **每条路径是独立节点**。加新记忆策略 = 加节点 + 加边，其他节点一行不动。
- **agent 是通用的**。它不关心记忆怎么来的，只看到 `messages` + `summary` + `retrieved`——记忆的细节被隔离在图的上半部分。这就是**关注点分离**。

---

## 三、State 设计：五个字段，谁是"档案"谁是"窗口"

图要跑，先定义工单长什么样。D5 你刚学过 Pydantic state + reducer，今天直接实战：

```python
from typing import Annotated, Any
from pydantic import BaseModel, Field
from langgraph.graph.message import add_messages

class MemoryState(BaseModel):
    messages: Annotated[list[Any], add_messages] = Field(default_factory=list)  # 完整历史
    recent: list[Any] = Field(default_factory=list)    # 压缩产物：当前窗口（无 reducer）
    summary: str = Field(default="", description="压缩后保留的历史摘要")
    retrieved: list[str] = Field(default_factory=list, description="长期记忆召回内容")
    memory_path: str = Field(default="", description="本次走的路径: short/long/compress")
```

逐个看，五个字段三种角色：

### 3.1 `messages`：完整档案（只能追加）

挂了 `add_messages` reducer——D3 学过的"聊天记录本"，只许往下加页。**它存的是从图出生到现在的全部消息，是"完整档案"。**

### 3.2 `recent`：当前窗口（可以整体覆盖）——今天最关键的字段

普通字段，没有 reducer → **last write wins，后写的整体覆盖旧值**。

为什么必须单开一个 `recent`？直接看坑：

> ⚠️ **压缩陷阱（D6 最坑的点，提前打预防针）**：你第一反应肯定是"压缩 = 把 messages 裁成最近 4 条"，于是 `compress_node` 里写 `return {"messages": recent}`——**不生效**。因为 `messages` 带 reducer，框架把你返回的 4 条理解成"追加到旧历史后面"，历史一条没少，压缩等于没压。想"换血"（覆盖），必须用无 reducer 的字段。`recent` 就是为这个存在的。

一句话分工：**`messages` 当完整档案只许追加，`recent` 当压缩后的当前窗口可以整体换血**。agent 读窗口、state 存档案——压缩 = 给窗口换血，不销毁档案。

### 3.3 `summary` / `memory_path`：普通字段，一次写入

- `summary`：压缩节点把"旧消息的浓缩"写进来，agent 组装上下文时读它。
- `memory_path`：route 节点的输出——本次走了 short / long / compress 哪条路。它是给边当"路标"用的（见第五节构图）。

### 3.4 `retrieved`：预留位（本版先空着）

清单最初的设计是 `long_node` 直接调检索、把结果填进 `retrieved`，agent 读 state 就拿得到长期记忆。但本版改走了另一条更"Agent 味"的路线：**存什么、取什么由 agent 自己调 `recall_long_term` 工具决定**（见 5.4），所以 `retrieved` 字段这一版不会被填充——它是为"图内直连检索"那条路线留的口子。字段留着不碍事，后面想换路线直接用它。

---

## 四、长期记忆：dict 顶班 + 两个工具

W3 的长期记忆是 Chroma 向量库。今天为了先跑通，用一个**进程内共享的 dict** 顶班，对外只暴露两个工具：

```python
memory_store = {}    # {关键词: 内容} —— 玩具级记忆库，正式版换回 W3 的 Chroma

@tool
def save_long_term(key: str, content: str) -> str:
    """把重要信息存入长期记忆。当用户说'记住...'或提到个人偏好时调用。"""
    memory_store[key] = content
    return f"已记住: {key} = {content}"

@tool
def recall_long_term(query: str) -> str:
    """从长期记忆检索信息。当用户询问之前提过的事实/偏好时调用。
    query 请传 1~3 个关键词（如"咖啡、预算"），不要传完整问句。"""
    kws = [w for w in query.replace(",", " ").replace("，", " ").split() if w]
    hits = []
    for k, v in memory_store.items():
        if any(kw in k or kw in v for kw in kws) or query in k or query in v:
            hits.append(f"{k}: {v}")
    return "\n".join(hits) if hits else "(长期记忆中没有相关内容)"
```

**两个设计点要讲清楚：**

**① 为什么"存/取"要做成工具，而不是节点里直接写 dict？** 因为"要不要存、存什么、拿什么来查"是**模型在对话里才能判断的**——它看到"记住我喜欢喝美式咖啡"才知道该调 `save_long_term`。做成工具，就是让模型自己决定调用时机，这正是 D3 你搭的 ReAct 循环最擅长的活。

**② 匹配为什么写得这么"笨"？** 玩具级 dict 没有向量，只能子串匹配。中文没有空格分词，所以让模型传**短关键词**（description 里写死了），再对每个关键词做子串包含。模型偶尔还是会传整句——没关系，那正是第七节要讲的一个坑，有兜底。**正式版直接换回你 W3 的 Chroma 语义检索，这个笨匹配就退休了。**

工具声明完，绑模型：

```python
llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)
llm_with_tools = llm.bind_tools([save_long_term, recall_long_term])
```

注意这里准备**两个** llm：裸 `llm` 给 route 和 compress 做纯文本判断；`llm_with_tools` 只给 agent 用——agent 才需要"会调工具"的能力，路由和压缩让它瞎调工具反而添乱。

---

## 五、四个节点逐块拆

### 5.1 `route_node`：把 if/else 搬进一个独立节点

W3 的决策散在三个方法里，今天统一收口到一个节点。它读 state、输出一个 `memory_path`：

```python
def route_node(state: MemoryState) -> dict:
    # 规则优先：消息超阈值直接压缩（确定性，不赌模型心情；也是 W3 maybe_compress 的阈值语义）
    if len(state.messages) > 10:
        return {"memory_path": "compress"}

    # 其余交给 LLM 语义判断：要长期记忆，还是普通对话？
    prompt = f"""判断这段对话最需要哪种记忆处理，只输出一个词：
- long: 用户明确要求记住信息(说'记住...')，或询问之前提过的事实/偏好
- short: 普通对话，无需特殊处理
对话历史:
{fmt(state.messages[-4:])}"""
    decision = llm.invoke(prompt).content.strip().lower()
    if "long" in decision:
        return {"memory_path": "long"}
    return {"memory_path": "short"}
```

**这里有个刻意的设计取舍，值得停下来讲**：为什么 compress 用规则（`len > 10`）、long/short 用 LLM？

- **压缩有客观阈值**——超过 10 条就该压，这是确定性的，用规则既稳又省钱（少一次 LLM 调用）。而且这恰好还原了 W3 的行为：`maybe_compress` 本来就是靠阈值触发压缩的。
- **long / short 是语义判断**——"记住…"和"询问以前的事"没有硬性规则可写，只能交给 LLM。

> 💡 如果你的目标是"让 LLM 全权决定走哪条路"（连 compress 也让它判），把上面的规则行删掉、prompt 里加回 compress 分支即可。但测试某条路径时**别依赖 LLM 路由碰运气**——先用规则锁死路径验证节点逻辑，验证完再放开。**先测确定性，再测智能性**，这个顺序第七节还会再出现。

`fmt` 是给 LLM 看历史的小工具，把消息对象摊成纯文本（有的消息没 `content` 就退回 `str`）：

```python
def fmt(msgs) -> str:
    return "\n".join(
        m.content if hasattr(m, "content") and m.content else str(m)
        for m in msgs
    )
```

### 5.2 三条记忆路径：short / long / compress

**short_node —— 最朴素，但想清楚它为什么存在：**

```python
def short_node(state: MemoryState) -> dict:
    """普通对话：消息已在 messages 自然累加，recent 同步为当前窗口"""
    return {"recent": state.messages[-6:], "memory_path": "short"}
```

普通对话不需要任何处理，但它干了件重要的事：**把 `recent` 窗口同步成最近 6 条**。为什么？因为 agent 读的是 `state.recent or state.messages`（见 5.4）——如果走 short 却不更新 `recent`，`recent` 会一直是空，agent 每次都退回读全量 `messages`。窗口机制就形同虚设。**每条路径都要让 agent 拿到的"窗口视图"是新鲜的。**

**long_node —— 只打标，不干活：**

```python
def long_node(state: MemoryState) -> dict:
    """长期记忆路径：真正的'存什么/取什么'由 agent 调工具完成，这里只标记路径"""
    return {"memory_path": "long"}
```

它只写 `memory_path`，什么都不做。**这不是偷懒，是职责划分**：route 决定"这次对话需要启用长期记忆处理"（走 long 这条路），但**具体存哪条、查什么，是 agent 看到消息内容后才能决定的**——所以落库/召回全部下放给 agent 的工具调用。记住：路由只负责"要不要走这条路"，不负责"路上干什么"。

**compress_node —— 压缩的正解：**

```python
def compress_node(state: MemoryState) -> dict:
    """压缩：旧消息 → LLM 摘要；裁剪结果写进 recent（普通字段），不碰 messages"""
    if len(state.messages) <= 4:
        return {"recent": state.messages, "summary": state.summary, "memory_path": "compress"}
    old, recent = state.messages[:-4], state.messages[-4:]
    sp = (f"把以下对话压缩成一句摘要，保留关键事实(人名/偏好/结论):\n{fmt(old)}\n"
          f"(已有摘要:{state.summary or '无'})")
    new_summary = llm.invoke(sp).content.strip()
    return {"recent": recent, "summary": new_summary, "memory_path": "compress"}
```

对照第三节的坑看这段，逻辑就通了：

1. 消息 ≤ 4 条：没得压，原样把窗口给 `recent`。
2. 超过 4 条：`old` = 前面的旧消息，`recent` = 最后 4 条。
3. 旧消息交 LLM 压成一句摘要（**带上已有的 `summary` 一起压**——这是增量压缩，不是每次都从零开始，摘要会越滚越全）。
4. **关键：裁剪结果写进 `recent`（普通字段，整体覆盖），绝不返回 `{"messages": ...}`**。这样 messages 档案完好，agent 的窗口换成了新鲜 4 条 + 一句摘要。

### 5.3 一个隐藏小坑：为什么要保留 `summary` 原值返回

compress_node 的"消息不够压"分支里，`summary` 写的是 `state.summary`（原值）。这不是废话——state 更新是**局部更新**：节点返回什么，什么才被写进 state。如果这个分支不写 `summary`，且之前某轮已经压出过摘要，state 里的 `summary` 并不会丢（没节点动它就不变）。**但显式写回来是"声明式"的好习惯**：让每个节点的返回 dict 完整表达"我这条路径结束时这几个字段应该是什么"，读代码的人不用去猜 state 的来龙去脉。

### 5.4 `agent_node`：把窗口 + 摘要 + 记忆拼给模型

三条路径最终都汇进 agent。它把 state 里准备好的东西组装成上下文，交给绑了工具的模型：

```python
def agent_node(state: MemoryState) -> dict:
    # 读窗口：压缩过就用 recent（裁剪后的 4 条），没压缩过退回全量 messages
    window = state.recent if state.recent else state.messages

    ctx = []
    if state.summary:                      # 压缩摘要 → system
        ctx.append(("system", f"历史摘要:{state.summary}"))
    if state.retrieved:                    # 长期记忆直连召回结果 → system（本版预留）
        ctx.append(("system", f"长期记忆:\n{'，'.join(state.retrieved)}"))
    ctx.extend(window)

    # 兜底闸：工具已经调过 3 次还没收敛 → 强制作答，别再烧 API
    tool_turns = [m for m in state.messages if getattr(m, "type", "") == "tool"]
    if len(tool_turns) >= 3:
        ctx.append(("user", "请基于已有工具结果直接给出最终答案；若没有查到，就如实说没查到，不要再调用工具。"))

    return {"messages": [llm_with_tools.invoke(ctx)]}
```

**四行组装逻辑，每行一个知识点：**

- `window = state.recent if state.recent else state.messages`——**agent 永远只读窗口，不读档案**。压缩后读 recent（4 条 + 摘要），没压缩读全量。这就是第三节"reducer 不让裁 messages"的完整闭环：不裁档案，只换窗口。
- summary / retrieved 以 `("system", ...)` 元组形式**插在窗口前面**——让模型先看到"历史摘要""长期记忆"这些背景，再看窗口里的实时对话。元组 `("role", content)` 是 LangChain 的轻量消息写法，和 BaseMessage 混排没问题。
- **兜底闸写在 ctx 组装之后**：数 `state.messages` 里已经有多少条 `type == "tool"` 的 ToolMessage，超过 3 条就把"直接作答"指令塞进去。这条是第七节"工具空转"坑的防御——**工具调用必须有轮数上限，"查不到"必须成为可接受的答案**。
- 返回值 `{"messages": [ai]}`——走 reducer 追加，完整历史继续涨。agent 只负责"回答 + 可能发起工具调用"，工具执行是下一个节点的事。

### 5.5 构图：边把一切串起来

```python
builder = StateGraph(MemoryState)
builder.add_node("route", route_node)
builder.add_node("short", short_node)
builder.add_node("long", long_node)
builder.add_node("compress", compress_node)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode([save_long_term, recall_long_term]))

builder.add_edge(START, "route")
builder.add_conditional_edges(
    "route",
    lambda s: s.memory_path,                          # 路由函数：读 memory_path 当路标
    {"short": "short", "long": "long", "compress": "compress"},
)
for n in ["short", "long", "compress"]:
    builder.add_edge(n, "agent")                      # 三路径都汇入 agent
builder.add_conditional_edges("agent", tools_condition)   # 有 tool_calls → tools，否则 END
builder.add_edge("tools", "agent")                    # 工具执行完回 agent

memory_graph = builder.compile()
```

**两处"图会自己走"的精髓：**

- **route 的条件边**：路由函数 `lambda s: s.memory_path` 读 state 里的 `memory_path` 字段——short_node 之类刚写进去的值，立刻被当作下一步的路线。这就是 D2 学的"边看数据选路"，今天的路由函数甚至不用自己算，**节点已经把答案写进 state 了，边只需要读**。
- **agent ↔ tools 回边**：agent 发起 tool_calls → `tools_condition` 判"有"→ 去 tools → ToolNode 自动执行并回填 ToolMessage → 回 agent 再看结果 → 直到模型不再要工具 → END。这就是 D3 那张 ReAct 图原样嵌了进来。**整张图 = 上半截记忆决策（route → 三路径）+ 下半截对话循环（agent ↔ tools），中间用 agent 焊死。**

---

## 六、完整可跑代码（含四组测试）

前面是逐块拆解，这一节把全部拼成一份能整体复制的文件。结构：state → 工具与 LLM → 节点 → 构图 → 测试。跑 `python memory_graph.py` 即可。

```python
"""W5-D6 项目日：MemoryManager(类) → MemoryManagerGraph(图)
目标：先跑通。记忆库用进程内 dict（玩具级），正式版换 W3 的 Chroma。"""
import os
from typing import Annotated, Any
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

# ══════════ 1. State：档案(messages) 与 窗口(recent) 分开 ══════════
class MemoryState(BaseModel):
    messages: Annotated[list[Any], add_messages] = Field(default_factory=list)  # 完整历史，只追加
    recent: list[Any] = Field(default_factory=list)      # 当前窗口（无 reducer → 可整体覆盖）
    summary: str = Field(default="", description="压缩后保留的历史摘要")
    retrieved: list[str] = Field(default_factory=list, description="长期记忆召回内容(预留)")
    memory_path: str = Field(default="", description="本次走的路径: short/long/compress")

# ══════════ 2. 长期记忆：dict 顶班 + 两个工具 ══════════
memory_store = {}    # {关键词: 内容}；正式版换回 W3 的 Chroma 向量检索

@tool
def save_long_term(key: str, content: str) -> str:
    """把重要信息存入长期记忆。当用户说'记住...'或提到个人偏好时调用。"""
    memory_store[key] = content
    return f"已记住: {key} = {content}"

@tool
def recall_long_term(query: str) -> str:
    """从长期记忆检索信息。当用户询问之前提过的事实/偏好时调用。
    query 请传 1~3 个关键词（如'咖啡、预算'），不要传完整问句。"""
    kws = [w for w in query.replace(",", " ").replace("，", " ").split() if w]
    hits = [f"{k}: {v}" for k, v in memory_store.items()
            if any(kw in k or kw in v for kw in kws) or query in k or query in v]
    return "\n".join(hits) if hits else "(长期记忆中没有相关内容)"

llm = ChatOpenAI(
    model="deepseek-chat",
    base_url="https://api.deepseek.com",
    api_key=os.getenv("DEEPSEEK_API_KEY") or os.getenv("OPENAI_API_KEY"),
)
llm_with_tools = llm.bind_tools([save_long_term, recall_long_term])   # 只有 agent 用它

# ══════════ 3. 工具函数 ══════════
def fmt(msgs) -> str:
    """消息对象摊成纯文本，给 LLM 看"""
    return "\n".join(
        m.content if hasattr(m, "content") and m.content else str(m)
        for m in msgs
    )

# ══════════ 4. route：规则管压缩，LLM 管 long/short ══════════
def route_node(state: MemoryState) -> dict:
    if len(state.messages) > 10:                 # 规则优先：超阈值必压缩（确定性）
        return {"memory_path": "compress"}
    prompt = f"""判断这段对话最需要哪种记忆处理，只输出一个词：
- long: 用户明确要求记住信息(说'记住...')，或询问之前提过的事实/偏好
- short: 普通对话，无需特殊处理
对话历史:
{fmt(state.messages[-4:])}"""
    decision = llm.invoke(prompt).content.strip().lower()
    if "long" in decision:
        return {"memory_path": "long"}
    return {"memory_path": "short"}

# ══════════ 5. 三条记忆路径 ══════════
def short_node(state: MemoryState) -> dict:
    """普通对话：recent 同步为当前窗口"""
    return {"recent": state.messages[-6:], "memory_path": "short"}

def long_node(state: MemoryState) -> dict:
    """长期路径：只打标；真正的存/取由 agent 调工具完成"""
    return {"memory_path": "long"}

def compress_node(state: MemoryState) -> dict:
    """压缩：旧消息 → LLM 摘要；裁剪写 recent，绝不返回 messages（reducer 会追加）"""
    if len(state.messages) <= 4:
        return {"recent": state.messages, "summary": state.summary, "memory_path": "compress"}
    old, recent = state.messages[:-4], state.messages[-4:]
    sp = (f"把以下对话压缩成一句摘要，保留关键事实(人名/偏好/结论):\n{fmt(old)}\n"
          f"(已有摘要:{state.summary or '无'})")
    new_summary = llm.invoke(sp).content.strip()
    return {"recent": recent, "summary": new_summary, "memory_path": "compress"}

# ══════════ 6. agent：窗口 + 摘要 + 工具轮数闸 ══════════
def agent_node(state: MemoryState) -> dict:
    window = state.recent if state.recent else state.messages     # 只读窗口
    ctx = []
    if state.summary:
        ctx.append(("system", f"历史摘要:{state.summary}"))
    if state.retrieved:
        ctx.append(("system", f"长期记忆:\n{'，'.join(state.retrieved)}"))
    ctx.extend(window)
    tool_turns = [m for m in state.messages if getattr(m, "type", "") == "tool"]
    if len(tool_turns) >= 3:                          # 工具调过 3 次还不收敛 → 强制作答
        ctx.append(("user", "请基于已有工具结果直接给出最终答案；若没有查到，就如实说没查到，不要再调用工具。"))
    return {"messages": [llm_with_tools.invoke(ctx)]}

# ══════════ 7. 构图 ══════════
builder = StateGraph(MemoryState)
builder.add_node("route", route_node)
builder.add_node("short", short_node)
builder.add_node("long", long_node)
builder.add_node("compress", compress_node)
builder.add_node("agent", agent_node)
builder.add_node("tools", ToolNode([save_long_term, recall_long_term]))

builder.add_edge(START, "route")
builder.add_conditional_edges("route", lambda s: s.memory_path,
                              {"short": "short", "long": "long", "compress": "compress"})
for n in ["short", "long", "compress"]:
    builder.add_edge(n, "agent")
builder.add_conditional_edges("agent", tools_condition)
builder.add_edge("tools", "agent")

memory_graph = builder.compile()
print("✅ 图编译成功")

# ══════════ 8. 四组测试 ══════════
if __name__ == "__main__":
    # 测试 A：compress —— 12 条超阈值，规则锁死压缩路径
    r1 = memory_graph.invoke({"messages": [("user", f"第{i}句闲聊内容{i}") for i in range(12)]})
    print("\n[A compress] 路径:", r1["memory_path"])
    print("  压缩后 recent 消息数:", len(r1["recent"]), "| 摘要:", (r1.get("summary") or "")[:40])

    # 测试 B：long 记住（落库）
    r2 = memory_graph.invoke({"messages": [("user", "记住我喜欢喝美式咖啡")]})
    print("\n[B 记住] 路径:", r2["memory_path"], "| 记忆库:", memory_store)

    # 测试 C：long 召回（跨轮验证——同进程内 dict 共享）
    r3 = memory_graph.invoke({"messages": [("user", "我喜欢喝什么咖啡？")]})
    print("[C 召回] 路径:", r3["memory_path"])
    for m in r3["messages"][-2:]:
        m.pretty_print()

    # 测试 D：short 普通对话
    r4 = memory_graph.invoke({"messages": [("user", "你好，简单介绍下你自己")]})
    print("\n[D short] 路径:", r4["memory_path"], "| 回复:", r4["messages"][-1].content[:30])
```

**每个测试验证什么，跑之前心里要有数：**

| 测试 | 输入 | 期望路径 | 验证点 |
|---|---|---|---|
| A | 塞 12 条闲聊（超 10 条阈值） | compress | 规则触发压缩：recent 裁到 4 条、summary 非空 |
| B | "记住我喜欢喝美式咖啡" | long | agent 自主调 `save_long_term`，记忆落库 |
| C | "我喜欢喝什么咖啡？" | long | **跨轮召回**：agent 调 `recall_long_term` 命中 B 存的偏好并答出 |
| D | "你好，简单介绍下你自己" | short | 普通路径：窗口透传、正常回答 |

其中 **C 是 D6 最硬的验收**——B、C 连着跑、C 能答出"美式咖啡"，才证明长期记忆真的跨轮工作了，而不只是路由打了个标。

---

## 七、三场真实故障，三个沉淀下来的习惯

代码不是一次写对的。跑这张图时，几乎每个难点都会以"报错"或"卡住"的形式咬你一口。下面三场按从最坑到次坑排列，每场都是一次"现象 → 归因 → 让代码说话 → 沉淀习惯"的完整排查。

### 7.1 故障一：`KeyError: 'summary'` —— 键不存在，不是值为空

**现象**：测试 A 跑完，访问 `r1["summary"]` 抛 `KeyError`。

**第一反应（错）**：是不是压缩没生成摘要？summary 是空字符串？

**真相**：报错信息说得很清楚——`r1` 这个最终 state 字典里**根本没有 `summary` 这个键**，不是值为空，是键不存在。为什么？看谁在写 `summary`：**只有 `compress_node` 写它**。而这次运行 route 判成了别的路径，`compress_node` 根本没执行，`summary` 从头到尾没有任何节点写过。

**为什么"默认值 `""` 没兜住"**：state 里声明了 `summary: str = ""`，理论上字段应该存在。但 LangGraph 对"从未被任何节点更新过的字段"的最终输出处理，和你预期的并不总是一致（具体行为和版本有关）。**结论只有一个：不要赌键一定存在。**

**沉淀的习惯①：访问 state 字段用 `.get()` 兜底，排查缺失键先打印 `keys()`。**

```python
print("r1 keys:", r1.keys())                      # 让代码自己说话：到底有哪些键
print("路径:", r1["memory_path"])                 # 大概率是 short → 证实 compress 没走
print("摘要:", r1.get("summary", "")[:40])        # 键不存在返回 ""，不报错
```

`.get()` 是防御式取值的标配，打印 `keys()` 是排查 state 缺失键的第一动作——这两个习惯从今天起刻进肌肉记忆。

### 7.2 故障二：压缩"裁不动" —— 想替换 messages，reducer 不让你替换

**现象**：`compress_node` 返回 `{"messages": recent}`，想"把历史裁成最近 4 条"，结果历史还在涨，压缩等于没压。

**根因**：第三节那个坑的实战版——`messages` 挂了 `add_messages` reducer，LangGraph 的规则是：**带 reducer 的字段，节点返回的新值会追加到旧值后面，而不是替换**。你想"换成这 4 条"，框架理解成"把这 4 条追加到旧历史后面"。

**解法**：加一个无 reducer 的普通字段 `recent`。普通字段是整体覆盖（last write wins），compress_node 把裁剪结果写进 `recent`，agent 读 `state.recent or state.messages`。这就是 D5 学的"覆盖 vs 追加"在真实项目里的第一次实战：**messages 当"完整档案"只能追加，recent 当"当前窗口"可以整体换血**。

**沉淀的习惯②：动"带 reducer 的字段"之前，先问自己——我要的是追加还是替换？** 要追加（消息历史、日志）直接写；要替换（窗口、快照、结果）单开一个无 reducer 的字段，别跟 reducer 对着干。

### 7.3 故障三：图"卡住" —— 长期记忆查不到，模型反复空转

**现象**：测试 C（召回"我喜欢喝什么咖啡"）跑很久不结束，像死锁。`invoke` 是同步的，没跑完就不会打印下一行。

**真相**：不是死锁，是模型在空转。看 B 测试的落库结果——key 是模型自己编的（比如"咖啡偏好"），value 是"用户喜欢喝美式咖啡"。到 C 测试，模型要召回，它调 `recall_long_term` 时 query 是**现场编的**，很可能是完整问句。而旧的匹配逻辑是整句 `query in value`：

```
"我喜欢喝什么咖啡" in "用户喜欢喝美式咖啡"  → False
```

查不到 → 模型不甘心 → 换个 query 再查 → 还是查不到 → 再换……**每转一圈都是一次真 API 请求**，几秒一圈，多转几圈就是几十秒。这就是你看到的"卡住"。**"查不到"对模型来说也是一种结果，但它不认——它会反复试。**

**三个层次的修法**（本版代码里全用了）：

1. **引导模型传关键词**：`recall_long_term` 的 docstring 写明"query 请传 1~3 个关键词"，模型照着描述调工具，传"咖啡"就命中了。
2. **匹配放宽**：关键词拆分 + 双向子串包含，query 里带点杂质也能命中。
3. **加轮数闸**：`agent_node` 里数 ToolMessage 条数，调过 3 次还收敛不了就强制作答，明说"没查到就说没查到"。**工具调用必须有上限，"查不到"必须成为可接受的最终答案**——这是生产级 Agent 的标准动作，今天这版只是把标准提前用上了。

**沉淀的习惯③：凡是"工具可能查不到结果"的场景，把"查不到"当成一等公民来设计**——给它一个明确的返回文案、给模型一条"查不到就直说"的退路、给整个循环一道轮数闸。

---

## 八、踩坑记（汇总，对着自查）

| # | 坑 | 一句话解法 |
|---|---|---|
| 1 | 🔴 压缩想 `return {"messages": recent}` 裁历史 | 带 reducer 的字段只会追加不会替换——裁剪写**无 reducer 的 `recent`**，agent 读窗口 |
| 2 | 🔴 `KeyError` 访问 state 字段 | 节点没执行 = 键不存在 ≠ 值为空。用 `.get()` 兜底，先 `print(r.keys())` 定位 |
| 3 | 🔴 工具查不到 → 模型反复重查"卡住" | 查不到也是结果。docstring 引导关键词 + 拆词匹配 + 工具轮数闸（≥3 次强制作答） |
| 4 | ⚠️ 测试 compress 依赖 LLM 路由碰运气 | 别赌模型心情——先用规则锁路径验证节点逻辑，再放开给 LLM（先确定性，后智能性） |
| 5 | ⚠️ 收口消息用裸 dict 混进消息列表 | 收口指令要用与上下文一致的类型：`("user", "...")` 元组或 `HumanMessage`，别用裸 dict |
| 6 | ⚠️ 想一把梭写出完美版 | 分步走：先 route 三路通 → 再加 long 记忆 → 再加 compress → 最后接 agent 工具 |
| 7 | ⚠️ 全部记忆逻辑塞 agent 一个节点 | 路由负责"走哪条路"，节点负责"路上干什么"，agent 只负责"回答+调工具"——职责别揉在一起 |
| 8 | ⚠️ 忘了 route 的兜底 if/else | LLM 输出可能飘成别的词——`if "long" in decision` + 最后 fallback 到 short |

---

## 九、速查卡片（复习直接看这）

**五字段语义**：

| 字段 | reducer? | 角色 | 谁写 | 谁读 |
|---|---|---|---|---|
| `messages` | ✅ 追加 | 完整档案 | agent / ToolNode | route 数条数、compress 裁切 |
| `recent` | ❌ 覆盖 | 当前窗口 | short / compress | agent |
| `summary` | ❌ 覆盖 | 历史摘要 | compress | agent |
| `retrieved` | ❌ 覆盖 | 长期记忆直连结果（预留） | 本版无人写 | agent |
| `memory_path` | ❌ 覆盖 | 路径路标 | route / 三路径 | 条件边 |

**边清单**：

```fallback
START → route
route ──(memory_path)──→ short | long | compress   条件边，读 state 字段当路标
short / long / compress → agent                     三路径汇合
agent ──(tools_condition)──→ tools | END           有 tool_calls 就调，否则结束
tools → agent                                      回边（ReAct 循环）
```

**一行一句口诀**：

- 压缩 = 给窗口换血（`recent`），不销毁档案（`messages`）
- 路由 = 只决定走哪条路；路上的活交给节点和 agent 工具
- 测试带 LLM 的图：**先锁死路径验证逻辑，再放开让模型决策**
- 工具调用：**有上限，"查不到"是合法答案**

---

## 十、一句话总结 + 下一篇预告

**W3 的 `MemoryManager` 用 if/else 在类内部调度记忆，今天把它翻译成了一张图：决策外置成 route 节点 + 条件边，三条记忆路径是三个独立节点，对话循环是嵌在底部的 ReAct 子图——agent 完全不感知记忆细节，只看到 messages + summary + retrieved，加新记忆策略只需加一个节点、拉一条边。** 而今天真正的收获，是三个坑沉淀出的三种习惯：`.get()` 兜底与 `keys()` 排查、别跟 reducer 对着干（要替换就单开普通字段）、给工具调用装上轮数闸。

图形态的记忆管理还有一个 W3 类实现给不了的福利，清单里那句提示其实是全文的钩子：**给这张图挂一个 checkpointer，跨会话持久化就一行搞定**——`graph.compile(checkpointer=...)`。W3 你得自己把 buffer 落盘，W6 学了 Checkpointer 之后，图的中间状态（包括压缩到一半的 summary）都能自动保存、断点续跑。这就是"记忆管理成图"这条路的下一站。

**魔鬼代言人**：如果跑通之后你冒出"这不就是把我 W3 的类换个写法吗，图也没多神奇"的念头——**你的感觉一半是对的**。今天的图，三条路径都是线性的、汇进 agent 就结束了，确实没比类强到哪去。图的发力点在两个你还没碰到的地方：一是**可观测性**（D4 的 stream 能让你亲眼看到每条消息走了哪条路，类做不到）；二是 **checkpointer 之后的状态持久化与中断恢复**（W6 的主题，类想做到得自己写全套）。今天先把"翻译"做对，这两个优势等 W6 回来验收。

**自查三问**（能答上 = 真懂了）：① 为什么压缩不能直接返回 `{"messages": recent}`？② `recent` 和 `messages` 各是什么角色，agent 读哪个？③ 长期记忆"查不到"时，图靠哪三层机制不空转？
