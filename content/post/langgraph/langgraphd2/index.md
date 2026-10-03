---
description: ""
title: "条件边"
draft: false
date: "2026-09-05T07:47:33+08:00"
slug: "LangGraphd2"
categories:
 - LangGraph
tags:
 - null
image: ""
---


# W5-D2 · 条件边：让图自己选路

> **关键词**：`add_conditional_edges` / 路由函数 / `path_map` / 入口条件路由 / MemoryState
> **前置**：W5-D1（StateGraph 四件套、节点 `return {...}`、`compile()`）、W3 `MemoryManager`
> **涉及文件**：`W5/lg_d2_router.py`、`W5/lg_d2_memorystate.py`
> **收尾预告**：D3 `ToolNode` + `tools_condition`——把 Function Calling 接进图，搭出图形态的 ReAct Agent

* * *

## 零、开场：D1 学会了画直线，D2 学画岔路

D1 你搭了一张图：`START → think → respond → END`。每一步去哪个节点，是**写死的**——`add_edge("think", "respond")` 意味着 think 干完**必去** respond，没有第二个选项。

D2 只加一个新东西：**条件边（Conditional Edge）**——让图的走向**根据 state 里的数据现场决定**，而不是写死在代码里。

```
  D1 的图（传送带）        D2 的图（分拣机器人）
  ───────────────         ───────────────────
  START → A → B → END     START →┬→ A → END
                                  ├→ B → END
                                  └→ C → END
                          看一眼 state，决定扔哪条道
```

> **类比**：D1 是高铁**固定轨道**——A 站必到 B 站；D2 是**到了枢纽站看票**——票上写着目的地（state 里存着信息），广播让你去几号站台（路由函数的返回值决定去哪个节点）。

**为什么这步非走不可**：真实 Agent 里，很多决策没法写死——

- 图怎么知道**该不该调用工具**？
- 该走**短期记忆还是长期记忆**？
- 消息是**该压缩还是照常存**？

这些答案要看 state 里的数据才知道。**D2 就是给图装上"看数据做决定"的能力。**

* * *

## 一、先回忆 Day1 的局限：边是写死的

### 1.1 固定轨道只能直来直去

```python
builder.add_edge("upper", "join")     # upper 跑完必去 join
```

这一行就是"upper 之后只能去 join"。**没有岔路、没有选择、没有"看情况"**。

而你想让图做的事，往往是"看情况"的。比如一个最简单的诉求：**按问题类型走不同的处理节点**——数学题走计算、搜索题走工具、闲聊直接答。用 Day1 的语法表达不了。

### 1.2 Day1 的答案是：加一堆 if 在节点里

如果只有普通边，你会怎么写？把所有分支逻辑塞进节点函数里：

```python
def hub(state):
    q = state["question"]
    if any(k in q for k in ["+", "搜索"]):
        return {"answer": handle_math_or_tool(state)}   # 在一个节点里全干了
    return {"answer": handle_chat(state)}
```

**能跑，但很糟**——所有业务挤在一个节点里，加一个分支就要改这个节点，节点越写越胖。

> ⚠️ 想想 D1 你刚建立的认知：**改流程 = 改边，不动节点。** 塞 if 的做法等于把"流程"又塞回了"节点"——开倒车。

### 1.3 🎯 条件边的答案：把"选择"本身变成图的一部分

```fallback
  塞 if：   决策逻辑在节点内部 → 看不见、改起来要动业务代码
  条件边：  决策逻辑在"边"上 → 图自己决定走哪条 → 拓扑和业务解耦
```

**决策从"代码里"拿出来，变成"图结构的一部分"**——这是 D2 全部内容的核心。

* * *

## 二、条件边三要素

条件边的 API 长这样：

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

> ⚠️ **路由函数是只读的**。它的职责是"看一眼 state，报个目的地"，不该在里面写数据或调外部服务——**改数据那是节点的活**。这个边界清楚，图才好调试。

* * *

## 三、最容易懵的两个点

### 3.1 路由函数为什么必须返回字符串，不能返回节点对象？

因为图是**声明式**的。

你 `add_node` 时只注册了"**名字 → 函数**"的映射，**编译之后** LangGraph 才知道节点对象在哪、怎么调度。路由函数跑在运行时，它只负责报"目的地叫什么名字"，具体怎么走到那个节点，由图引擎处理。

```
  你声明时：  add_node("math_node", math_node)     ← 注册"名字→函数"
  运行时：    router(state) → "math_node"          ← 只报名字
  图引擎：    "math_node" → 找到函数 → 执行         ← 引擎负责
```

> **类比**：你在站台上喊"我要去北京"（字符串），而不是拽着列车员就跑（对象）。**列车员知道去北京该上哪趟车——那是他的职责。**

如果 router 返回一个函数对象，LangGraph 会报 `TypeError`——它要的是名字，不是人。

### 3.2 `path_map` 什么时候可以省略？

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

> 💡 初学阶段建议**一律显式写 `path_map`**。省掉它确实短一行，但它把"返回值 ↔ 节点名"的对应关系藏起来了——图一复杂，debug 时得靠猜。等图结构熟了再省。

* * *

## 四、最小可运行：问题分类器（`W5/lg_d2_router.py`）

目标：**用户输入问题 → 按关键词分到 数学 / 工具 / 闲聊 三条路之一**。

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

跑起来：

```fallback
{'question': '1+1 等于几',    'answer': '[数学] 1+1 等于几'}
{'question': '搜索 LangGraph', 'answer': '[工具] 搜索 LangGraph'}
{'question': '今天天气不错',   'answer': '[闲聊] 今天天气不错'}
```

**同一个图、同一个 START，输入不同 → 走的路线不同。**

### 用 stream 看路由的"瞬间"

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

**每次只有一个节点出现**——另外两个压根没执行：

```
  ❌ 不是：三个节点都跑，最后挑一个结果
  ✅ 是：   只跑被选中的那一个
```

> 💡 这点很重要，尤其后面接 LLM 的时候：**没被选中的节点不会消耗 token**。路由不只是"选路"，它还是**省钱的开关**。

* * *

## 五、🎯 一个常见优化：入口条件路由，省掉分类节点

上面那张图有个"藏起来的冗余"——你注意看：`classify_router` 把 `math` / `tool` / `chat` 的**判断逻辑直接写在路由函数里**了，根本没有单独的"分类节点"。

对比另一种常见写法（**四节点版**）：

```
  START → classify → ┬→ math_node  → END
                     ├→ tool_node  → END
                     └→ chat_node  → END
      ↑ 先跑个 classify 节点，把 "math" 写进 state["qtype"]
          再由 router 读 qtype 决定去哪
```

而第一节代码是**三节点 + 入口路由**：

```
  START →┬→ math_node  → END
         ├→ tool_node  → END
         └→ chat_node  → END
      ↑ 分类逻辑直接写进 router，START 就分叉
```

| | 四节点版 | 三节点 + 入口路由版 |
|---|---|---|
| 节点数 | 4（多一个 classify） | 3 |
| State 需要多一个字段？ | 需要 `qtype` 存中间结果 | **不需要** |
| 分类结果留在最终 state？ | 是 | 否（用完即弃） |

**差别在哪**：`qtype` 只是路由的中间变量——**没人需要它在最终 state 里**。既然如此，为什么为它专门设一个节点、占一个字段？直接让 router 算完就扔，图更干净。

> 🎯 **取舍标准一句话：这个中间结果，还有别人要用吗？**
> - 后续节点需要知道"问题被判成什么类型"（比如写日志、做统计）→ 保留 `classify` 节点，把它写进 state
> - 没人要 → 直接让路由函数算完就扔，`START` 直接挂条件边

* * *

## 六、重头戏：把 W3 的 MemoryManager 画成一张图

### 6.1 为什么这个练习最值钱

D3 预告过：本周的目标之一是**用 LangGraph 重写你 W3 手写的 `MemoryManager`**——短期窗口 / 长期 Chroma / 滚动压缩，本来是 `if/elif/else` 的判断逻辑，现在要**用图表达**：

```fallback
  W3 的 MemoryManager（命令式）：          画成图（声明式）：
  def get_context(...):                   START
      if len > 20: 压缩                        │
      elif "记住":  存长期              decide_path(state)
      else:        短期                        │
                                  ┌───────────┼───────────┐
                                  │           │           │
                                  ▼           ▼           ▼
                               compress     long        short
```

**下面这个 `MemoryState` 雏形，就是那件事的草稿版。** 今天的节点只写 `decision` 占位，但**路由逻辑是真的**——它验证的是："判断走哪条路"这件事，用三条边 + 一个函数表达得清不清楚。

> 🎯 **更关键的体验是"用图重新组织自己写过的代码"**：你 W3 用命令式 if/else 表达的分流，换成声明式的边，**逻辑一模一样，表达方式全变了**。这个手感，就是 LangGraph 真正的价值。

### 6.2 完整代码（`W5/lg_d2_memorystate.py`）

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
    return {"decision": "compress", "compressed_summary": "[模拟] 老消息已摘要"}

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

### 6.3 三条路的触发逻辑

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
if len(state["messages"]) > 20:      # 先看长度
    return "compress"
recent = ...
if any("记住" in c for c in recent):  # 再看内容
    return "long"
return "short"                        # 兜底
```

**压缩的优先级高于长期记忆。为什么？**

> 💡 压缩是**兜底手段**——消息都堆到 20 条了，再不压缩上下文窗口就要爆了。这时候"用户刚说了句'记住 xxx'"已经不是首要矛盾。**先保命，再谈功能。**
>
> 这个顺序和你 W3 `MemoryManager` 的触发思路一致：**阈值判断永远优先于语义判断。**

### 6.4 三个节点各自只写自己那格

```python
def short_node(state):    return {"decision": "short"}
def long_node(state):     return {"decision": "long", "long_term_hit": "..."}
def compress_node(state): return {"decision": "compress", "compressed_summary": "..."}
```

**只有被选中的那条分支会写入自己的专属字段**：

```
  走 short   → 最终 state 里 long_term_hit、compressed_summary 保持 ""
  走 long    → long_term_hit 被填上，compressed_summary 还是 ""
  走 compress→ compressed_summary 被填上，long_term_hit 还是 ""
```

> ✅ 这正是 D1 讲的"**未返回的字段原样保留**"——三个节点共用一张工单，各写各的格子，互不干扰。**如果字段更新是"覆盖整个 state"，这条分支一跑，另外两个字段就丢了——reducer 语义（按字段合并）是这套设计的前提。**

### 6.5 给 MemoryState 的节点加上"真正的记忆逻辑"

雏形的节点只改 `decision` 占位。等你要让它真正干活时，只需要把占位换成你 W3 的真实调用：

```python
def long_node(state: MemoryState) -> dict:
    # 模拟 → 真实：调 W3 的长期记忆写入
    return {"decision": "long", "long_term_hit": "[模拟] 从向量库召回用户画像"}
    #                      ↓
    # return {"decision": "long", "long_term_hit": str(mm.long_term.recall(...))}
```

**图的结构一行不用改**——`decide_path` 还是那三条边，变的只是节点内部的"怎么做"。这就是把 W3 代码"搬进图"的完整路径：**先画骨架（雏形），再把真逻辑填进节点。**

* * *

## 七、踩坑记

### 坑 1：state 字段名三处不一致（`compress` / `compressed` 打架）🔴

TypedDict 的字段名出现在**三个地方，必须一模一样**：① schema 声明、② 节点返回值、③ `invoke` 传入的初始值。任何一处写错，都静默出事：

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

- schema 里根本没有 `compressed_summary` 这个字段，传进去的值要么被忽略、要么触发 `InvalidUpdateError`（取决于 LangGraph 版本）
- `compress_summary` 字段**从未被初始化**——虽然 `compress_node` 会写入它，但在那之前它是"不存在"的状态
- 最终 state 里可能同时出现两个长得几乎一样的 key，debug 时极难察觉

**修法（二选一，但必须统一）**：

```python
# 方案 A：统一成 compressed_summary（推荐）
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

> ⚠️ **为什么这类错误特别烦人**：它不会让程序崩溃（大部分时候），而是让你**拿到一个字段值对不上的 state**。盯着 `out.get("compressed_summary")` 返回空字符串，会以为是节点没执行，其实是字段名写错了。
>
> 💡 **防范习惯**：写完 TypedDict 后，**把字段名复制粘贴到节点返回值和测试数据里，别手打**。`compress` / `compressed`、`summary` / `summery` 这种差异，肉眼扫十遍也看不出来。

### 坑 2：`.get()` 的默认值类型要和字段一致 ⚠️

写 `m.get("content", ...)` 时，**默认值要和 content 的真实类型一致**：

```python
# ❌ 默认值给成空字典
recent = [m.get("content", {}) for m in state["messages"]]
if any("记住" in c for c in recent):
```

**为什么不报错**：`"记住" in {}` 在 Python 里是合法的——它在检查 dict 的 **key 里有没有"记住"**，返回 `False`。所以语义错了但程序照跑。

**什么时候会踩雷**：一旦 `content` 真的缺失，`recent` 里就混进了 dict 和 str 两种类型。这行虽然不崩，但**类型不一致的 list 是定时炸弹**——后续任何字符串操作（`c.split()`、`c.strip()`）都会在某一刻炸给你看。

```python
# ✅ 默认值用空字符串
recent = [m.get("content", "") for m in state["messages"]]
```

> 💡 判断口诀：**`.get(key, 默认值)` 里的默认值，必须和字段值的类型相同**。content 是 str，默认值就该是 `""`，不是 `{}`。

### 坑 3：路由函数返回了节点对象 → `TypeError` ⚠️

- **现象**：`TypeError`，提示无法处理
- **原因**：router 必须返回 `str`（或 `list[str]`），不能返回函数对象
- **解法**：返回**节点名字符串**（忘了加引号写 `return math_node` 就是这个坑）

### 坑 4：`path_map` 的 value 写错节点名 → `KeyError` ⚠️

- **现象**：路由时报 KeyError
- **原因**：`path_map` 的 value 必须是 `add_node` 时用的名字
- **解法**：对照 `add_node` 的名字逐个检查；省略 `path_map` 只在"返回值 = 节点名"时成立

### 坑 5：分支节点忘了连 END → 图不结束 ⚠️

- **现象**：图跑完分支就卡住，或报"没有出边"
- **原因**：每个分支节点都要自己连到 END
- **解法**：加完节点顺手连 END——用 `for name in [...]: b.add_edge(name, END)` 循环给所有分支统一连，是个值得养成的好习惯，**加分支时不会漏**

### 坑 6：路由函数里塞了副作用 ⚠️

- **现象**：调试输出乱，或 state 被意外修改
- **原因**：router 里 `print` 太多、或写了 `mm.add(...)`
- **解法**：router 保持**纯函数**——只读 state，只返回字符串。要写数据交给节点

### 坑 7：测试数据里 `messages` 混了 str 和 dict ⚠️

```python
# ❌ 混用
{"messages": [{"content": "记住我住北京"}, "x"*30]}
#                                          ↑ 这个元素没有 .get()

# ✅ 统一 dict
{"messages": [{"role": "user", "content": "..."}]}
```

**为什么危险**：`decide_path` 里 `m.get("content", "")` 假设每个元素都是 dict。混进一个 str，直接 `AttributeError: 'str' object has no attribute 'get'`。

> 💡 测试数据保持单一类型，路由函数才不会在类型假设上翻车。**脚本注释里写一句"messages 统一用 dict，别混 str"，下次自己就不会犯。**

### 坑 8：调试代码重复跑了两遍 ⚠️

调试到一半常会留下重复的验证循环——两份几乎一样的 `for ... tests.items()`，第二份是残留。不影响正确性，但**每个测试都多调一次 `invoke`**——节点是模拟的没感觉，等接了真 LLM，这就是**双倍的 token 消耗**。

> 💡 顺带：打印用 `out.get("decision")` 而不是 `out["decision"]`——字段缺失时前者返回 `None`，后者直接 `KeyError`。

* * *

## 八、速查卡片（复习直接看这）

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

# ===== MemoryState 雏形（记忆画成图）=====
# 路由 decide_path：消息>20 → compress；含"记住" → long；否则 → short
# 三个节点各写一格：short 只写 decision，long 加 long_term_hit，
#                  compress 加 compressed_summary —— 未返回的字段原样保留
```

* * *

## 九、一句话总结

**条件边 = 让图的走向由 state 里的数据现场决定，而不是写死在代码里。**

三个落点：

1. **router 是纯函数**：读 state、返回字符串、不改数据——改数据那是节点的活
2. **`path_map` 是翻译表**：返回值是代号就必写，返回值就是节点名才能省
3. **入口路由能省掉分类节点**——当中间结果没人需要时，`START` 直接分叉最干净

**而 MemoryState 雏形验证了一件事**：你 W3 用 `if/elif/else` 写的"短期 / 长期 / 压缩"分流，可以用三条边 + 一个函数清清楚楚地画出来——**逻辑一行没变，从代码变成了图**。把节点里的占位换成真实的 `mm.add` / `mm.recall`，你的 `MemoryManager` 就真的成了一张图。

```
  今天：  START → decide_path →┬→ short     （雏形：节点只写 decision）
                               ├→ long
                               └→ compress

  下一步：把 W3 的真实逻辑填进节点 → MemoryManager 成图
  D3：    把 Function Calling 接进图 → 图开始会"干活"
```

**下一篇：D3 `ToolNode` + `tools_condition`**——把 W2 的 Function Calling 接进图，搭出图形态的 ReAct Agent。到那天你会发现：**今天学的条件边，就是 D3 那张图的"刹车"**——模型要不要继续调工具，由一条条件边看着 `tool_calls` 决定。

* * *

### 🔴 魔鬼代言人（最后一段，别跳过）

跑完分类器，你可能会想：**"这不就是几个 if 吗？图个啥？"**

你的感觉对了一半。**单看这个分类器，`if/elif/else` 确实更直接**——为一个三路分流搭一张图，是杀鸡用牛刀。

但注意两件事：

**① 图的价值不在"写一次"，在"改的时候"。** 今天你要加第四个分支"遇到'算税'走税务节点"——`if` 版本要改函数体，图版本只在 `decide_path` 加一个 return、加一个节点。**分支越多，图的优势越大。**

**② 今天的图还没有环。** 从 D1 到 D2，图从直线变岔路，但仍然是"一路向前"。真正的质变在 D3——加一条回边，图会自己转圈了，那时候"用 if 写循环"和"用图写循环"的差别才真正显现。

**所以别急着下"图不过如此"的结论。** 今天的条件边，是 D3 那张会转圈的图的地基——**没有"看 state 做决定"的能力，那个圈永远转不起来。**
