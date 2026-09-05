---
description: ""
title: "条件边"
draft: false
date: "2026-09-05T07:47:33+08:00"
slug: "LangGraphd2"
categories:
 - LangGraph
tags:
 - 
image: ""
---


# 条件边：让图自己选路

> **关键词**：`add_conditional_edges` / 路由函数 / `path_map` / 入口条件路由 / MemoryState
> **前置**：W5-D1（StateGraph 四件套、节点 `return {...}`、`compile()`）、W3 `MemoryManager`
> **涉及文件**：`W5/lg_d1_fill.py`、`W5/lg_d2_router.py`、`W5/lg_d2_memorystate.py`
> **收尾预告**：D3 `ToolNode` + `tools_condition`——把 W2 的 Function Calling 接进图，搭出图形态的 ReAct Agent

* * *

## 零、今天两件事

**① 补 D1 差量**（约 20 分钟）+ **② 学 D2 条件边**（约 60 分钟）。

先别急着冲 D2。D1 有几项是清单要求但我没做的——它们看着简单，实际是在练 Day2 要用到的手感。文末"魔鬼代言人"会解释为什么这 20 分钟不能省。

在开跑之前，先花两分钟**看一眼全局地图**——你上周纠结过的方向问题，新计划表已经给了答案。

* * *

## 一、先看清全局：新路线表把 LangGraph 拆成了三周

### 1.1 你知识库里那份新增的计划表

`AI_Agent学习路线_LangGraph深入版.md`（v3，生成于 2026-09-04）——**13 周学习 + 3 周项目参考区**。

```
学习主表（13 周）
┌────────────────────────────────────────────────────────┐
│ W1   LLM 基础（Prompt + API + FC 最小）            🔴   │
│ W2   FC 进阶 + 多厂商 API + 端到端联调             🔴   │
│ W3   记忆与上下文管理（短/长/本地/压缩）           🔴   │
│ W4   LangChain 基础（为 LangGraph 打底）            🔴   │
│ W5   LangGraph 基础      ▓▓▓ 深入重点① ▓▓▓        🔴   │
│ W6   LangGraph 进阶      ▓▓▓ 深入重点② ▓▓▓        🔴   │
│ W7   LangGraph 多 Agent  ▓▓▓ 深入重点③ ▓▓▓        🔴   │
│ W8   RAG 全链路                                    🔴   │
│ W9   RAG 进阶 + Eval                               🔴   │
│ W10  FastAPI + Docker + SSE                        🔴   │
│ W11  前端最小对接                              🔴 / 🟡 │
│ W12  Agent Eval + Skills / MCP / A2A               🔴   │
│ W13  Computer-Use 🟡 + 13 周综合复盘               🟡   │
└────────────────────────────────────────────────────────┘
                          ↓
项目参考区（3 周，不计入学习周）
┌────────────────────────────────────────────────────────┐
│ P1   L9 Multi-Agent Writer                        🟡   │
│ P2   L6 Nano Agent（harness 视角）⭐ 强推              │
│ P3   L8 Reusable Skill Pack                       🟡   │
└────────────────────────────────────────────────────────┘
```

### 1.2 这份表回答了你上周的那个纠结

你之前问过"**先学 LangGraph 还是先补 RAG**"。

新表给的答案是：**骨架先行、知识后补**——

- LangGraph 从 1 周扩成 **W5/W6/W7 三周**，作为全表核心轴
- RAG 整体后移到 **W8/W9**
- 项目从学习表里**挪出去**，单列 P1–P3 参考区，不占学习周

**理由其实很朴素**：RAG 是"给 Agent 装知识"，LangGraph 是"让 Agent 会思考"。你得先有个会思考的骨架，才知道往哪儿装知识。**先造脑子，再灌知识。**

> 💡 这份表还有个改动值得注意：**深度分级 🔴 必深入 / 🟡 了解型**。W5–W7 全是 🔴，意味着这三周**没有"随便看看就行"的内容**——每天的练习都得真跑通。

### 1.3 你现在站在哪儿

```
  W5（09-07 ~ 09-13）LangGraph 基础 ▓▓▓
  ├── Day1  最小图（think→respond / turns / W4 链包节点）  ✅ 已封板
  ├── Day2  条件边 ← 你在这里
  ├── Day3  ToolNode + tools_condition（ReAct 最小图）
  └── ...
```

**W5 这周要交付两样东西**：

1. 用 LangGraph 重写你 W3 的 `MemoryManager`，把它**画成图**（短期 / 长期 / 压缩节点 + 条件路由）
2. 一个 ReAct 风格 Agent 最小图（`reason → tool → observe → answer`）

**今天的 Day2 练习，正好是第 1 项的预演**——上午跑的 `MemoryState` 雏形，就是那个"MemoryManager 成图"的草稿版。**别把它当成练习代码，它是本周交付物的第一版。**

* * *

## 二、D1 差量补齐（约 20 分钟）

### 2.1 差量表：哪缺哪不缺

| # | Day1 清单要求 | 我已完成 | 缺口？ |
|---|---|---|---|
| 1 | StateGraph 三要素 | ✅ think/respond 图 | 无 |
| 2 | 节点返回 dict（partial update） | ✅ `turns` 那节演示过"未返回字段保留" | 无 |
| 3 | `invoke` / `stream` 差异 | ✅ 用过 stream 看逐节点 | 无 |
| 4 | **State schema 三选一** | ⚠️ 只用了 TypedDict | **缺 Dataclass / Pydantic** |
| 5 | **加法器图**（输入 a,b → result） | ❌ 做的是字符串处理 | **缺** |
| 6 | **三节点字符串拼接流水线** | ⚠️ 做过两节点 | **缺三节点体验** |
| 7 | 五步构建法顺序 | ✅ 用的是新版 `add_edge(START/END)` | 等价即可 |
| 8 | **笔记三问归档** | ❌ 没写进笔记 | **缺** |

**一句话总结差异**：第 5、6 项（加法器 + 三节点拼接）不只是"再跑通两个例子"——它们是为了让你**亲眼看见 state 在多个节点间传递时，每个节点只改自己那格**。

`turns` 那节其实已经验证了这点，所以这两项对我来说 ≈ 20 分钟补完。

### 2.2 补差①：加法器

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class AddState(TypedDict):
    a: int
    b: int
    result: int

def add_node(state: AddState) -> dict:
    return {"result": state["a"] + state["b"]}

b = StateGraph(AddState)
b.add_node("add", add_node)
b.add_edge(START, "add")
b.add_edge("add", END)
g = b.compile()

print("加法器 invoke:", g.invoke({"a": 3, "b": 5}))   # {'a': 3, 'b': 5, 'result': 8}
for step in g.stream({"a": 3, "b": 5}, stream_mode="updates"):
    print("  stream:", step)                          # {'add': {'result': 8}}
```

**看输出里那两个细节**：

- `invoke` 结果是 `{'a': 3, 'b': 5, 'result': 8}`——**`a`、`b` 原样保留**，`result` 是新增的。节点只写了 `result` 一格，别的格子没被动过。
- `stream` 输出 `{'add': {'result': 8}}`——**结构是 `{节点名: 该节点改的字段}`**。

> ⚠️ 这个 `{节点名: 更新字段}` 的格式，Day2 看路由时会**反复出现**。今天记住它，明天省一半力气。

### 2.3 补差②：三节点拼接流水线（这项别跳）

```python
class ConcatState(TypedDict):
    parts: list[str]
    joined: str

def upper_node(state: ConcatState) -> dict:
    return {"parts": [p.upper() for p in state["parts"]]}

def join_node(state: ConcatState) -> dict:
    return {"joined": " | ".join(state["parts"])}

def length_node(state: ConcatState) -> dict:
    return {"joined": state["joined"] + f" (len={len(state['joined'])})"}

b2 = StateGraph(ConcatState)
for n, fn in [("upper", upper_node), ("join", join_node), ("length", length_node)]:
    b2.add_node(n, fn)

b2.add_edge(START, "upper")
b2.add_edge("upper", "join")
b2.add_edge("join", "length")
b2.add_edge("length", END)
g2 = b2.compile()

print("\n拼接 invoke:", g2.invoke({"parts": ["hello", "world"]}))
for step in g2.stream({"parts": ["hello", "world"]}):
    print("  stream:", step)
```

#### 为什么三节点比两节点多教一课

用 `stream` 看，你会看到三个节点**各自动了哪一格**：

```fallback
  stream: {'upper':  {'parts': ['HELLO', 'WORLD']}}          ← 只动 parts
  stream: {'join':   {'joined': 'HELLO | WORLD'}}            ← 只动 joined
  stream: {'length': {'joined': 'HELLO | WORLD (len=13)'}}   ← 又动 joined
```

**三件事在这一刻同时发生了**：

```
  ① upper 改的是 parts，joined 没被动过（因为还没人写它）
  ② join 读的是 upper 改过的 parts —— 数据在节点间流动了
  ③ length 读的是 join 写过的 joined —— 同一个格子被连续改了两次
```

第 ③ 点最关键：**`joined` 被 `join` 写了一次，又被 `length` 改了一次**。而 `length` 之所以能改，是因为它 `return` 时**手动把旧值拼了进去**（`state["joined"] + ...`）。

> ⚠️ 这就是 D1 讲过的**字段更新默认是"覆盖"**——如果 `length` 直接 `return {"joined": f"(len={...})"}`，前面的 `"HELLO | WORLD"` 就没了。**正因默认会覆盖，所以才需要你自己先把旧值读出来。**
>
> 想让框架自动累加，得用 reducer（`Annotated[list, operator.add]`）——那是 W6 讲记忆和消息列表时的核心机制。

**两节点图体验不到第 ③ 点**——`think → respond` 里两个节点改的是同一格没错，但你很容易误以为"框架帮我合并了"。**三节点才看得清：每次都是覆盖，拼接是你自己干的。**

### 2.4 补差③：State schema 三选一（不用跑，记住这句）

| 写法 | 特点 | 什么时候用 |
|---|---|---|
| **TypedDict** | 最简，就是个带类型提示的 dict | **Day1–Day2 用这个**，90% 场景够用 |
| **Dataclass** | 可以带方法（字段 + 行为放一起） | State 需要自带逻辑时 |
| **Pydantic** | 带**校验**（类型错了直接报错） | W6 接复杂状态、需要严格约束时再切 |

> 💡 一句话记忆：**TypedDict 最简、Dataclass 可带方法、Pydantic 带校验。**
>
> 现在全用 TypedDict，别提前上 Pydantic——校验是好东西，但在你还没搞清楚"state 里该放什么"的阶段，它会用一堆报错打断你的节奏。

### 2.5 Day1 封板自查表

| 你的代码 | 应该看到的输出 | 说明 |
|---|---|---|
| 加法器 `invoke` | `{'a': 3, 'b': 5, 'result': 8}` | a、b 原样保留，result 是新增 |
| 加法器 `stream` | `{'add': {'result': 8}}` | **结构是 `{节点名: 该节点改的字段}`** |
| 拼接 `stream` | 三条：upper 只动 parts → join 只动 joined → length 再动 joined | 每节点只写自己那格 |

**三行都对 → Day1 正式封板。**

* * *

## 三、D2 正课：条件边

### 3.1 Day1 的局限：边是写死的

`add_edge("upper", "join")` 意味着"**upper 跑完必去 join**"——没有第二个选项。

现实问题来了：

- 图怎么知道**该不该调用工具**？
- 该走**短期记忆还是长期记忆**？
- 消息是**该压缩还是照常存**？

这些答案**不能写死**，得看着 state 里的数据**现场决定**。

> **类比**：Day1 是高铁**固定轨道**，A 站必到 B 站；Day2 是**到了枢纽站看票**——票上写着目的地（state 里存着信息），广播让你去几号站台（路由函数的返回值决定去哪个节点）。

### 3.2 三要素

```python
builder.add_conditional_edges(
    "source_node",   # ① 从哪个节点出来（岔路口）
    route_decision,  # ② 路由函数：读 state，返回字符串
    {"math": "math_node", "chat": "chat_node"},  # ③ path_map：返回值 → 真实节点名
)
```

| 要素 | 是什么 | 类比 |
|---|---|---|
| **source** | 岔路口设在谁后面 | 哪个站台要分叉 |
| **router** | 普通 Python 函数 `(state) -> str`，**读** state 但不改 | 检票员，看一眼票 |
| **path_map** | 字典，把 router 返回值"翻译"成节点名 | 站台指示牌：B2 → 2 号站台 |

> ⚠️ **路由函数是只读的**。它的职责是"看一眼 state，报个目的地"，不该在里面 `mm.add(...)` 或改任何东西——**改数据那是节点的活**。这个边界清楚了，图才好调试。

### 3.3 最容易懵的两个点

#### Q1：路由函数为什么必须返回字符串，不能返回节点对象？

因为图是**声明式**的。

你 `add_node` 时只注册了"**名字 → 函数**"的映射，**编译之后** LangGraph 才知道节点对象在哪、怎么调度。路由函数跑在运行时，它只负责报"目的地叫什么名字"，具体怎么走到那个节点，由图引擎处理。

```
  你声明时：  add_node("math_node", math_node)     ← 注册"名字→函数"
  运行时：    router(state) → "math_node"          ← 只报名字
  图引擎：    "math_node" → 找到函数 → 执行         ← 引擎负责
```

> **类比**：你在站台上喊"我要去北京"（字符串），而不是拽着列车员就跑（对象）。**列车员知道去北京该上哪趟车——那是他的职责。**

如果 router 返回一个函数对象，LangGraph 会报 `TypeError`——它要的是名字，不是人。

#### Q2：`path_map` 什么时候可以省略？

**当 router 的返回值恰好就是节点名时**，LangGraph 会直接用返回值当节点名去查，可以省略 `path_map`。

判断标准一句话：

> **返回值 ≈ 节点名 → 省略；返回值是"类别代号" ≠ 节点名 → 必须写。**

```python
# 情况 A：返回代号，必须写 path_map
def router(state): return "math"           # "math" 不是节点名
add_conditional_edges("classify", router,
                      {"math": "math_node", "chat": "chat_node"})   # ✅ 必须

# 情况 B：返回值就是节点名，可省
def router(state): return "math_node"      # 正好是 add_node 用的名字
add_conditional_edges(START, router)       # ✅ 省略
```

> 💡 我的建议：**初学阶段一律显式写 `path_map`**。省掉它确实短一行，但它把"返回值 ↔ 节点名"的对应关系藏起来了——图一复杂，你 debug 时得靠猜。等你对图结构熟了再省。

### 3.4 最小可运行：问题分类器（`W5/lg_d2_router.py`）

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class QState(TypedDict):
    question: str
    answer: str

def classify_router(state: QState) -> str:
    """路由函数：只读 state，返回字符串（此处省略 path_map，因为返回值 = 节点名）"""
    q = state["question"].lower()
    if any(k in q for k in ["+", "-", "*", "/", "等于", "几"]):
        return "math_node"
    if any(k in q for k in ["搜索", "什么是"]):
        return "tool_node"
    return "chat_node"

def math_node(state: QState) -> dict:
    return {"answer": "[数学] " + state["question"]}

def tool_node(state: QState) -> dict:
    return {"answer": "[工具] " + state["question"]}

def chat_node(state: QState) -> dict:
    return {"answer": "[闲聊] " + state["question"]}

b = StateGraph(QState)
for name, fn in [("math_node", math_node), ("tool_node", tool_node), ("chat_node", chat_node)]:
    b.add_node(name, fn)

# START 直接挂条件边（入口路由）：省掉"先到分类节点再路由"那步
b.add_conditional_edges(START, classify_router)
for name in ["math_node", "tool_node", "chat_node"]:
    b.add_edge(name, END)

g = b.compile()

for q in ["1+1 等于几", "搜索 LangGraph", "今天天气不错"]:
    print(g.invoke({"question": q}))
```

### 3.5 🎯 你做的这处优化，比清单更漂亮

清单里的标准写法是**四节点**：

```
START → classify → ┬→ math_node  → END
                   ├→ tool_node  → END
                   └→ chat_node  → END
         ↑ 先跑个分类节点，把 "math" 写进 state["qtype"]
             再由 router 读 qtype 决定去哪
```

你的写法是**三节点 + 入口路由**：

```
START →┬→ math_node  → END
       ├→ tool_node  → END
       └→ chat_node  → END
    ↑ 分类逻辑直接写进 router，START 就分叉
```

**差别在哪**：

| | 清单版 | 你的版本 |
|---|---|---|
| 节点数 | 4（多一个 classify） | 3 |
| State 字段 | 需要 `qtype` 存中间结果 | **不需要 `qtype`** |
| 分类结果是否留在 state 里 | 是 | 否（用完即弃） |

> 💡 这不是偷工减料，是**正确的取舍**：`qtype` 只是路由的中间变量，没人需要它在最终 state 里。既然如此，**为什么要为它专门设一个节点、占一个字段？** 直接让 router 算完就扔，图更干净。
>
> 反过来说：**如果后续节点需要知道"这个问题被判成什么类型"（比如写日志、做统计），那就得保留 `classify` 节点把它写进 state。** 判断标准很朴素——**这个中间结果，还有别人要用吗？**

```fallback
跑起来：
{'question': '1+1 等于几',    'answer': '[数学] 1+1 等于几'}
{'question': '搜索 LangGraph', 'answer': '[工具] 搜索 LangGraph'}
{'question': '今天天气不错',   'answer': '[闲聊] 今天天气不错'}
```

**同一个图、同一个 START，输入不同 → 走的路线不同。**

### 3.6 用 stream 看路由的"瞬间"

`invoke` 只给你结果，看不出走了哪条路。换成 `stream`：

```python
for q in ["1+1 等于几", "搜索 LangGraph", "今天天气不错"]:
    for step in g.stream({"question": q}, stream_mode="updates"):
        print(step)
```

```fallback
{'math_node': {'answer': '[数学] 1+1 等于几'}}
{'tool_node': {'answer': '[工具] 搜索 LangGraph'}}
{'chat_node': {'answer': '[闲聊] 今天天气不错'}}
```

**每次只有一个节点出现**——另外两个压根没执行。

```
  ❌ 不是：三个节点都跑，最后挑一个结果
  ✅ 是：   只跑被选中的那一个
```

> ⚠️ 这点很重要，尤其后面接 LLM 的时候：**没被选中的节点不会消耗 token**。路由不只是"选路"，它还是**省钱的开关**。

* * *

## 四、重头戏：MemoryState 雏形

### 4.1 为什么这个练习最值钱

清单步骤 5 要求：把 W3 的 `MemoryManager` 思路（短期 / 长期 / 压缩）画成图。

**它不只是练条件边语法**——它是**W5 当周产出①的预演**：

```
  W5 当周产出①：用 LangGraph 重写 MemoryManager，画成图
                       ↑
  今天这张 MemoryState 雏形 = 它的草稿版
                       ↑
  区别在于：今天三个节点只改 decision 字段（占位），
           本周内要把真正的 mm.add / mm.recall / compress() 填进去
```

> 🎯 更关键的是这个体验：**用图重新组织你自己写过的代码。** 你 W3 写的 `MemoryManager` 是命令式的（一堆 if/else 判断走哪条路），今天要把它变成声明式的（三条边 + 一个 router）。**代码逻辑一模一样，表达方式全变了**——这个"换一种写法"的手感，才是 LangGraph 真正的价值。

### 4.2 完整代码（`W5/lg_d2_memorystate.py`）

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class MemoryState(TypedDict):
    messages: list[dict]
    compressed_summary: str
    long_term_hit: str
    decision: str

def decide_path(state: MemoryState):
    """路由函数：读 state，返回节点名（省略 path_map 因为返回值 = 节点名）"""
    if len(state["messages"]) > 20:
        return "compress"                                  # ① 消息太多 → 压缩
    recent = [m.get("content", "") for m in state["messages"]]
    if any("记住" in c for c in recent):
        return "long"                                      # ② 最近有"记住" → 存长期
    return "short"                                         # ③ 默认 → 短期窗口

def short_node(state: MemoryState) -> dict:
    return {"decision": "short"}                           # 只写自己那格，别的字段保留

def long_node(state: MemoryState) -> dict:
    return {"decision": "long", "long_term_hit": "[模拟] 从向量库召回用户画像"}

def compress_node(state: MemoryState) -> dict:
    return {"decision": "compress", "compress_summary": "[模拟] 老消息已摘要"}

b = StateGraph(MemoryState)
for name, fn in [("short", short_node), ("long", long_node), ("compress", compress_node)]:
    b.add_node(name, fn)

b.add_conditional_edges(START, decide_path)
for name in ["short", "long", "compress"]:
    b.add_edge(name, END)

g = b.compile()

# 三条测试路（messages 统一用 dict，别混 str）
tests = {
    "走短期": {"messages": [{"role": "user", "content": "今天天气不错"}],
               "compressed_summary": "", "long_term_hit": "", "decision": ""},
    "走长期": {"messages": [{"role": "user", "content": "记住我住北京"},
                            {"role": "assistant", "content": "好的"}],
               "compressed_summary": "", "long_term_hit": "", "decision": ""},
    "走压缩": {"messages": [{"role": "user", "content": f"普通消息{i}"} for i in range(21)],
               "compressed_summary": "", "long_term_hit": "", "decision": ""},
}

for label, data in tests.items():
    print(f"\n=== {label} ===")
    for s in g.stream(data, stream_mode="updates"):   # 看实际触发了哪个节点
        print("  ", s)
    out = g.invoke(data)
    extra = out.get("long_term_hit") or out.get("compressed_summary") or ""
    print(f"  decision={out.get('decision')}  {extra}")
```

### 4.3 三条路的触发逻辑

```
                    START
                      │
                      ▼
              decide_path(state)
                      │
        ┌─────────────┼─────────────┐
        │             │             │
  len(messages)>20  含"记住"      其他
        │             │             │
        ▼             ▼             ▼
   compress          long          short
   decision=         decision=     decision=
   "compress"        "long"        "short"
   +摘要             +召回结果     （只写 decision）
        │             │             │
        └─────────────┼─────────────┘
                      ▼
                     END
```

**注意判断顺序**——这是个容易被忽略的设计细节：

```python
if len(state["messages"]) > 20:   # 先看长度
    return "compress"
recent = ...
if any("记住" in c for c in recent):   # 再看内容
    return "long"
return "short"                     # 兜底
```

**压缩的优先级高于长期记忆。** 为什么？

> 💡 因为压缩是**兜底手段**——消息都堆到 20 条了，再不压缩就要爆上下文窗口。这时候"用户刚说了句'记住 xxx'"已经不是首要矛盾了。**先保命，再谈功能。**
>
> 这个顺序和你 W3 `MemoryManager` 里 `maybe_compress()` 的触发思路是一致的：**阈值判断永远优先于语义判断。**

### 4.4 三个节点各自的写法差异

```python
def short_node(state):    return {"decision": "short"}
def long_node(state):     return {"decision": "long", "long_term_hit": "..."}
def compress_node(state): return {"decision": "compress", "compress_summary": "..."}
```

**只有被选中的那条分支会写入自己的专属字段**：

```
  走 short   → 最终 state 里 long_term_hit、compressed_summary 保持 ""
  走 long    → long_term_hit 被填上，compressed_summary 还是 ""
  走 compress→ compressed_summary 被填上，long_term_hit 还是 ""
```

> ✅ 这正是 D1 讲的"**未返回的字段原样保留**"——三个节点共用一张工单，各写各的格子，互不干扰。三节点拼接那节练的就是这个。

* * *

## 五、踩坑记

### 坑 1：🔴 `compress_summary` 和 `compressed_summary` 打架（你代码里的 bug）

你 `lg_d2_memorystate.py` 里，**TypedDict 声明的字段名和测试数据里传的字段名不一致**：

```python
class MemoryState(TypedDict):
    messages: list[dict]
    compress_summary: str        # ← 声明：compress_summary（无 ed）
    long_term_hit: str
    decision: str

tests = {
    "走短期": {"messages": [...],
               "compressed_summary": "",    # ← 传入：compressed_summary（有 ed）
               ...}
}
```

**后果**：

- State schema 里根本没 `compressed_summary` 这个字段，你传进去的值要么被忽略、要么触发 `InvalidUpdateError`（取决于 LangGraph 版本）
- `compress_summary` 字段**从未被初始化**——虽然 `compress_node` 会写入它，但在那之前它是"不存在"的状态
- 最终 state 里可能同时出现两个长得几乎一样的 key，debug 时极难察觉

**修法（二选一，但必须统一）**：

```python
# 方案 A：统一成 compressed_summary（与 Day2 清单一致，推荐）
class MemoryState(TypedDict):
    messages: list[dict]
    compressed_summary: str
    long_term_hit: str
    decision: str

def compress_node(state: MemoryState) -> dict:
    return {"decision": "compress", "compressed_summary": "[模拟] 老消息已摘要"}

# 方案 B：统一成 compress_summary
#   → 那 tests 里也要改成 "compress_summary": ""
```

> ⚠️ **为什么这类 bug 特别烦人**：它不会让程序崩溃（大部分时候），而是让你**拿到一个字段值对不上的 state**。你盯着 `out.get("compressed_summary")` 返回空字符串，会以为是节点没执行，其实是字段名写错了。
>
> 💡 **防范习惯**：写完 TypedDict 后，**把字段名复制粘贴到测试数据和节点返回值里**，别手打。`compress` / `compressed`、`summary` / `summery` 这种差异，肉眼扫十遍也看不出来。

### 坑 2：`m.get("content", {})` 的默认值类型写错了

你代码里：

```python
recent = [m.get("content", {}) for m in state["messages"]]   # ← 默认值是 {}
if any("记住" in c for c in recent):
```

清单里是 `m.get("content", "")`——**默认值应该是空字符串，不是空字典**。

**为什么不报错**：`"记住" in {}` 在 Python 里是合法的——它在检查 dict 的 **key 里有没有"记住"**，返回 `False`。所以语义错了但程序照跑。

**什么时候会踩雷**：如果哪天 `content` 真的缺失，`recent` 里就混进了 dict 和 str 两种类型。这行虽然不崩，但**类型不一致的 list 是定时炸弹**——后续任何字符串操作（`c.split()`、`c.strip()`）都会在某一刻炸给你看。

```python
recent = [m.get("content", "") for m in state["messages"]]   # ✅ 默认值用空字符串
```

### 坑 3：路由函数返回了节点对象 → `TypeError`

- **现象**：`TypeError`，提示无法处理
- **原因**：router 必须返回 `str`（或 `list[str]`），不能返回函数对象
- **解法**：返回**节点名字符串**
- 💡 如果你的 router 写成 `return math_node`（忘了引号），就是这个坑

### 坑 4：`path_map` 的 value 写错节点名 → `KeyError`

- **现象**：路由时报 KeyError
- **原因**：`path_map` 的 value 必须是 `add_node` 时用的名字
- **解法**：对照 `add_node` 的名字逐个检查
- 💡 省略 `path_map` 只在"返回值 = 节点名"时成立，否则必须显式写

### 坑 5：分支节点忘了连 END → 图不结束

- **现象**：图跑完分支就卡住，或报"没有出边"
- **原因**：每个分支节点都要自己连到 END（`add_edge("branch", END)`）
- **解法**：加完节点顺手把 END 连上，别漏
- 💡 你代码里用 `for name in [...]: b.add_edge(name, END)` 循环连，这个写法很好——**加分支时不会漏**

### 坑 6：路由函数里塞了副作用

- **现象**：调试输出乱，或 state 被意外修改
- **原因**：router 里 `print` 太多、或写了 `mm.add(...)`
- **解法**：router 保持**纯函数**——只读 state，只返回字符串。要写数据交给节点

### 坑 7：测试数据里 `messages` 混了 str 和 dict

你代码里的注释已经点明了：`# 三条测试路（messages 统一用 dict，别混 str）`。

```python
# ❌ 混用
{"messages": [{"content": "记住我住北京"}, "x"*30]}
#                                          ↑ 这个元素没有 .get()

# ✅ 统一 dict
{"messages": [{"role": "user", "content": "..."}]}
```

**为什么危险**：`decide_path` 里 `m.get("content", "")` 假设每个元素都是 dict。混进一个 str，直接 `AttributeError: 'str' object has no attribute 'get'`。

> 💡 清单步骤 5 的示例里恰好就混了（`"x"*30`），你改成统一 dict 是**对的**——这个改动说明你注意到了类型一致性。

### 坑 8：调试代码重复跑了两遍

你文件末尾有两个 `for label, data in tests.items()` 循环，第二个是第一个的残留：

```python
for label, data in tests.items():      # 第一次：stream 看路径 + invoke 看结果
    ...
for label, data in tests.items():      # 第二次：又跑一遍 invoke
    out = g.invoke(data)
    print(f"\n数据结构：{label} → out = {out}\n")
    extra = out.get("long_term_hit") or out.get("compressed_summary") or ""
    print(f"{label} → decision={out['decision']}  {extra}")
```

不影响正确性，但**每个测试都多调一次 `invoke`**——现在节点是模拟的没感觉，等接了真 LLM，这就是**双倍的 token 消耗**。

> 💡 顺带一提：第二个循环里用了 `out['decision']`（硬下标），第一个循环用的是 `out.get('decision')`。**统一用 `.get()`** 更稳——字段缺失时返回 `None` 而不是抛 KeyError。

* * *

## 六、速查卡片

```python
# ===== 条件边三要素 =====
builder.add_conditional_edges(
    "source_node",        # ① 从哪个节点出来
    route_fn,             # ② 路由函数 (state) -> str，只读
    {"math": "math_node"} # ③ path_map（返回值=节点名时可省）
)

# ===== 入口条件路由：START 直接分叉（省掉分类节点）=====
b.add_conditional_edges(START, classify_router)
# router 返回 "math_node" 这种节点名 → 可省 path_map

# ===== 每个分支都要连 END =====
for name in ["math_node", "tool_node", "chat_node"]:
    b.add_edge(name, END)

# ===== 看路由走了哪条路 =====
for step in g.stream(data, stream_mode="updates"):
    print(step)     # {'math_node': {...}}   ← 只有被选中的节点出现

# ===== stream_mode 对比 =====
# "updates"（默认）: {节点名: 该节点更新的字段}      ← 看"谁改了什么"
# "values"        : 每个节点后的完整 state          ← 看"工单长什么样"

# ===== 路由函数铁律 =====
# 1. 返回 str（或 list[str]），不是函数对象
# 2. 只读 state，不改 state
# 3. 返回值 = 节点名 → 可省 path_map；是代号 → 必须写

# ===== State schema 三选一 =====
# TypedDict  最简（Day1-Day2 用）
# Dataclass  可带方法
# Pydantic   带校验（W6 接复杂状态再切）
```

* * *

## 七、D1 与 D2 的对照

| 维度 | Day1 | Day2 |
|---|---|---|
| 核心概念 | StateGraph 三要素 + 五步构建 | 条件边 + 路由函数 |
| 图形态 | 直线流水线 | **分叉路由** |
| 实战产出 | 加法器 + 字符串拼接 | 问题分类器 + MemoryState 雏形 |
| 核心抽象 | 节点 = 函数，state = 数据 | **router = 函数，path_map = 字典** |
| 验证方式 | `stream` 看节点更新 | `stream` 看路由选择 |

**两天的关键差异就一句话**：**Day1 的图是写死的，Day2 的图会自己选路。**

> **类比收尾**：Day1 的图是**传送带**，Day2 的图是**分拣机器人**——看一眼包裹（state），扔到对应滑道。这个"看一眼再决定"的能力，就是 W5 要做的 MemoryManager 图和 ReAct 最小图的**发动机**。

* * *

## 八、一句话总结

**D1 补的两个差量（加法器、三节点拼接）练的是"每节点只改自己那格"，D2 的条件边练的是"看 state 决定走哪条路"——两者拼起来，图才从"传送带"变成"分拣机器人"。**

三个落点：

1. **router 是纯函数**：读 state、返回字符串、不改数据——改数据那是节点的活
2. **`path_map` 是翻译表**：返回值是代号就必写，返回值就是节点名才能省
3. **入口路由能省掉分类节点**——当中间结果没人需要时，`START` 直接分叉最干净

**MemoryState 雏形是本周交付物的草稿**：今天三个节点只改 `decision` 占位，本周内要把真正的 `mm.add` / `mm.recall` / `compress()` 填进去——**那时候你 W3 写的 `MemoryManager` 就真的成了一张图**。

**下一篇：D3 `ToolNode` + `tools_condition`**——把 W2 的 Function Calling 接进图，搭出图形态的 ReAct Agent（`reason → tool → observe → answer`）。到那天，你 W4-D6 手写的那个 `_run_tools` 循环，就该被一条回边取代了。

* * *

### 🔴 魔鬼代言人（最后一段，别跳过）

如果你觉得**"加法器太简单，直接冲条件边不行吗"**——

**不行。这不是内容难度问题，是手感问题。**

Day2 的 MemoryState 雏形要求你**自己画图**——自己决定"哪些是节点、哪些是边、state 里放什么字段"。而画图的手感，恰恰来自那两个看起来很蠢的练习：

- **加法器**教你：`a`、`b` 原样保留，`result` 是新增的 → **节点只写自己那格**
- **三节点拼接**教你：`joined` 被写两次，第二次得手动拼旧值 → **字段更新是覆盖**

这两个认知，在 MemoryState 里直接变成：

```
  short_node    return {"decision": "short"}   ← 只写一格（加法器教的）
  compress_node return {"decision":..., "compress_summary":...}
                                    ↑
                        为什么别的字段没丢？因为默认保留（拼接教的）
```

**跳过它们，你在写 MemoryState 时就会犯嘀咕**："我 `return {"decision": "short"}`，`long_term_hit` 还在吗？"——然后得回头翻 D1。

**20 分钟换 Day2 不卡壳，这笔账划算。**

再补一句更扎心的：**今天这张 MemoryState 图，三个节点还都是假的**（只改 `decision`，`long_term_hit` 写的是 `"[模拟] 从向量库召回"`）。它现在**没有任何实际功能**。

但它验证了一件更重要的事：**你 W3 那套"短期 / 长期 / 压缩"的判断逻辑，可以用三条边 + 一个函数表达清楚。** 逻辑一行没变，从 `if/elif/else` 变成了图的拓扑——**这个转换能力，才是 LangGraph 真正的门槛。**

* * *
