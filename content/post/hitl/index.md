---
description: ""
title: "让图停下来问人"
draft: false
date: "2026-09-09T11:06:52+08:00"
slug: "hitl"
categories:
 - LangGraph
tags:
 - HITL
image: ""
---

# W6-D3 · 让图停下来问人：HITL 的 interrupt 与 Command

## 零、为什么需要"停下来"

D1/D2 你给图装上了存档系统：它能跨 invoke 记事、能按 thread_id 分会话、能把存档落到磁盘。但有个场景存档解决不了：

```fallback
用户："帮我把 /data 下的旧文件删了"
无 HITL: agent 决定删 → 直接执行 → 😱 删错了没人负责
有 HITL: agent 决定删 → 【暂停问人】→ 人批准/拒绝 → 才继续 ✅
```

**Agent 越自主，就越需要有人能在关键时刻按下暂停键。** 这就是 HITL（Human-In-The-Loop）：让图"停下来问人 → 拿到人的决定 → 从停的地方继续"。

**HITL 的硬前提是 checkpointer**——这不是巧合，而是逻辑必然：

```fallback
interrupt() 一停 → 整个 state 和"停在哪"必须落盘（checkpoint）
                 → 人审批完 → Command(resume=...) 读同一个存档 → 从暂停处继续
```

没有 checkpointer，图停不下来（不知道停在哪），也续不上（resume 只能从头跑）。**所以 D1/D2 不是"HITL 的前置作业"，它就是 HITL 的一半。**

> 类比：D1/D2 你学会的是"录影带能存能放"；今天学的是**在录影带里插一个"待定帧"**——暂停时把画面存下来，人看完说"过"，再从这一帧接着放。

---

## 一、两种停法：哨卡 vs 对话窗口

LangGraph 有两种中断方式，先看清它们的形状再动手：

| | `interrupt_before` / `interrupt_after` | `interrupt()`（函数） |
|---|---|---|
| 位置 | **compile() 时配置**，在节点**边界**停 | 节点**内部**调用，可停在中间 |
| 灵活度 | 固定停（每次执行到这就停） | 条件停（只在你想停的时候停） |
| 能不能提问 | 停后只能看 state | 可以 `interrupt({问题})` 把问题抛出去 |
| 用户怎么回 | 只能"放行"继续 | `Command(resume=值)` 把值**传回节点内** |
| 停在哪 | `state.next` 指向**下一个**节点 | `state.next` 指向**当前**节点 |
| 一句话 | **哨卡**（固定位置拦人） | **对话**（边问边收答案） |

> 类比：哨卡像地铁安检——不管你是谁，走到这都得停，检完放行就完事；对话窗口像柜台办事——工作人员抬头问你一句"确定要办吗？"，你回答"办"或"不办"，甚至可以说"换个方式办"，他拿到你的回答才继续。

**现代 LangGraph 推荐优先用 `interrupt()`**——它能提问、能收任意答案、还能按条件决定停不停。但哨卡写法更直白，适合"这个节点每次都必须审批"的场景。两种都要会，因为它们的返回值语义不一样。

---

## 二、方式一：`interrupt_before` —— 编译期哨卡

### 2.1 代码

```python
"""W6-D3 方式一：interrupt_before —— 在 tools 节点前设哨卡"""
import os
from dotenv import load_dotenv
load_dotenv(override=True)   # 需要指定项目根时用 Path(__file__).parent.parent / ".env"

from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import ToolNode, tools_condition
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# ── 危险工具（示例版，不真删；print 是"执行标记"，验收要用）──
@tool
def delete_file(path: str) -> str:
    """【危险】删除本地文件"""
    print(f"【工具真的被执行了】路径: {path}")
    return f"文件已删除: {path}"

class S(TypedDict):
    messages: Annotated[list, add_messages]

llm = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
).bind_tools([delete_file])

def agent_node(state: S) -> dict:
    return {"messages": [llm.invoke(state["messages"])]}

b1 = StateGraph(S)
b1.add_node("agent", agent_node)
b1.add_node("tools", ToolNode([delete_file]))
b1.add_edge(START, "agent")
b1.add_conditional_edges("agent", tools_condition)   # 有 tool_calls → tools
b1.add_edge("tools", "agent")                        # 回边

# ⭐ 唯一的差异：编译时声明"进 tools 之前必须停"
g1 = b1.compile(checkpointer=MemorySaver(), interrupt_before=["tools"])

cfg1 = {"configurable": {"thread_id": "hitl-1"}}

# ① 跑到哨卡处自动停住
g1.invoke({"messages": [("user", "帮我删掉 /tmp/a.txt")]}, cfg1)

# ② 看暂停现场（注意：进度信息在 get_state 的快照上，不在 invoke 返回值上）
st = g1.get_state(cfg1)
print("① 暂停！下一步将执行:", st.next)                    # ('tools',)
print("   当前消息数:", len(st.values["messages"]))        # 2

# ③ 人批准 → 不带新输入，用同一个 config 继续
r1b = g1.invoke(None, cfg1)
print("   最终回复:", r1b["messages"][-1].content)
```

### 2.2 三段逻辑逐个讲

**① `interrupt_before=["tools"]`**：编译期声明"每次要进 tools 节点之前，先停下"。参数是**节点名列表**，可以写多个（比如 `["tools", "send_email"]`）。图照常跑到 agent 产出 tool_calls、`tools_condition` 判"该去 tools"的那一刻——**在迈步之前被拦下**。

**② 暂停时的 state 长什么样**：

```fallback
messages = [
  HumanMessage("帮我删掉 /tmp/a.txt"),
  AIMessage(tool_calls=[delete_file(path="/tmp/a.txt")]),
]
                                    ← 缺这个：还没有 ToolMessage
```

**消息数是 2，且没有 ToolMessage**——这是"工具还没执行"的直接证据。配合工具函数里那句 `print("【工具真的被执行了】")`：暂停阶段它**没有出现**，resume 之后才出现。**两条合起来，才是"工具是在审批之后才跑的"铁证。**

**③ 恢复为什么是 `invoke(None, cfg1)`**：哨卡式没有"问题"，所以也不需要答案——你只是告诉图"放行，继续跑"，输入传 `None` 即可。config 必须带**同一个 thread_id**，否则图不知道要续哪个档（换了 thread_id 就是开新档，暂停就丢了）。

> ⚠️ 一个容易写错的 API：`r1.next` 是**不存在的**。`invoke()` 返回的是 state 的**值字典**（就是 `{"messages": [...]}` 这种内容），没有 `.next`；"下一步去哪"属于**执行进度**，挂在 `get_state(config)` 返回的快照对象上。
>
> 类比：`invoke()` 给你的是**工单内容**（填了哪些格子）；`get_state()` 给你的是**工单 + 进度标签**（干到哪、下一步去哪个工位）。"下一步"写在进度标签上，不写在工单纸上。

---

## 三、方式二：`interrupt()` —— 节点内对话

哨卡只能"拦住放行"，如果你要**问一句并收一个答案**，就得用节点内的 `interrupt()`。

```python
"""W6-D3 方式二：interrupt() —— 节点内提问，Command(resume=...) 回话"""
from langgraph.types import interrupt, Command

def agent_ask(state: S) -> dict:
    last = state["messages"][-1].content
    # ⭐ 暂停在这一行，把问题抛给调用方；用户 resume 时传的值会成为它的返回值
    decision = interrupt({
        "question": f"模型想执行危险操作: {last}",
        "options": ["approve", "reject"],
    })
    if decision == "approve":
        return {"messages": [("assistant", "【已获批准】继续执行")]}
    return {"messages": [("assistant", "【已拒绝】取消删除操作")]}

b2 = StateGraph(S)
b2.add_node("agent", agent_ask)               # ← 换成会问人的 agent
b2.add_node("tools", ToolNode([delete_file]))
b2.add_edge(START, "agent")
b2.add_conditional_edges("agent", tools_condition)
b2.add_edge("tools", "agent")
g2 = b2.compile(checkpointer=MemorySaver())    # ← 这里不写 interrupt_before

cfg2 = {"configurable": {"thread_id": "hitl-2"}}

# ① 跑到 interrupt() 处停住（某些版本会抛 GraphInterrupt，某些版本正常返回）
try:
    g2.invoke({"messages": [("user", "删掉 /tmp/b.txt")]}, cfg2)
    print("② invoke 正常返回")
except Exception as e:
    print("② 收到暂停信号:", type(e).__name__)   # GraphInterrupt

# ② 用 .next 确认它真的停了（这是唯一可靠的判据）
st2 = g2.get_state(cfg2)
print("   停在:", st2.next)                    # ('agent',)

# ③ 人点"拒绝" → 把值送回 interrupt() 那一行
out2 = g2.invoke(Command(resume="reject"), cfg2)
print("   结果:", out2["messages"][-1].content)   # 【已拒绝】取消删除操作
```

### 3.1 三个必须想明白的点

**① `Command(resume="reject")` 里的值，是怎么"回到" `interrupt()` 那一行的？**

这是今天最反直觉的地方——**它不是"回到"那一行，而是把节点重新跑了一遍**。

流程是这样的：`interrupt()` 被调用时，框架把当前 state 和"停在第几个节点"存成 checkpoint，然后中断；你调用 `Command(resume="reject")` 时，框架读同一个 checkpoint，**从暂停的那个节点重新执行**，但当代码再次执行到 `interrupt(...)` 这一句时，框架不再中断，而是直接把 `"reject"` 当作它的返回值。

```fallback
第一次执行 agent_ask：
   ... → interrupt(...) → 【保存 checkpoint，中断】

resume 后重新执行 agent_ask：
   ... → interrupt(...) → 【直接返回 "reject"，不中断】→ 继续往下走
```

**所以：interrupt 之前的语句会被执行两次**（比如节点开头读 state、调 LLM 那些代码）。**不要把有副作用的操作（写库、发请求）写在 `interrupt()` 之前**——否则 resume 时会重复执行。这是 `interrupt()` 最需要记住的一条工程纪律。

> 💡 顺带解释为什么节点必须"从开头重跑"：因为 Python 没有"从函数中间恢复"的能力，框架只能用"重放 + 让 interrupt 直接返回"来模拟。理解了这一点，`Command(resume=...)` 就不再是魔法了。

**② `GraphInterrupt` 到底算不算报错？**

**不算错，是"暂停信号"。** 但这里有个版本陷阱：**有些版本会抛出 `GraphInterrupt` 异常，有些版本 invoke 正常返回、图其实已经暂停了**。你那份运行的输出就属于后者——打印了"正常返回"，但 `get_state(config).next` 是 `('agent',)`，说明确实停住了。

**结论：判断"图停没停"，永远看 `get_state(config).next` 是否非空，不要靠 try/except 猜。** 这也解释了为什么第一节那张 API 表要刻进脑子里。

**③ 停在哪，`.next` 就指向哪**：

- 哨卡式：停在 tools **之前** → `.next == ('tools',)`（下一步本来要去 tools）
- 对话式：停在 agent **内部** → `.next == ('agent',)`（当前节点还没跑完）

看到 `.next` 非空，就是图在等人。**这也是 D2 学的"state.next 是中断探针"真正派上用场的地方。**

### 3.2 这一版的一个已知缺陷（先说清，D4 改）

`agent_ask` 是**每次进 agent 都问人**——哪怕用户只是说"你好"，也会被拦下来问"是否批准"。因为它把 `interrupt()` 写在了节点最前面，无条件触发。

更合理的做法是：**只在模型真的要调危险工具时才 interrupt**（判断 `tool_calls` 里有没有 `delete_file`）。这是 W6-D4 的改造点，今天先跑通机制。

---

## 四、怎么证明"它真的停住了"

HITL 最容易自欺的地方：你以为它停了，其实工具已经跑完了。所以验证要用**两条独立证据**，而不是一条：

| 证据 | 怎么取 | 说明 |
|---|---|---|
| ① 暂停时没有 ToolMessage | `len(st.values["messages"]) == 2` | Human + AI(带 tool_calls)，**缺 ToolMessage** 说明工具没执行 |
| ② 工具的执行标记只在 resume 后出现 | 工具里的 `print("【工具真的被执行了】")` | 暂停阶段没这行输出，resume 后才出现 = 审批之后才跑 |
| ③ 停在哪 | `st.next` 非空 | 哨卡式 `('tools',)` / 对话式 `('agent',)` |

**两条合起来才是铁证**：消息数 2 只能说明"还没回填结果"，如果工具其实已经跑过（比如用了 `interrupt_after` 而不是 `interrupt_before`），消息数会是 3（多一条 ToolMessage）、print 也会提前出现。**所以单看一条会被骗。**

> 💡 把 `print` 塞进工具函数当"执行标记"，是验证 HITL 最省事的一招。它不依赖任何框架的调试工具，纯肉眼可见——**凡是"我以为它没跑"的怀疑，都可以靠一句 print 终结。**

---

## 五、暂停 → 审批 → 继续的完整时序

把两种方式的过程画成一张图，方便回看：

```fallback
【哨卡式 interrupt_before=["tools"]】

invoke(用户输入) → agent 产出 tool_calls → 条件边判"去 tools"
                                            │
                                      ⛔ 哨卡拦下（保存 checkpoint）
                                            │
                      你：get_state(cfg).next == ('tools',)  ← 确认停住
                      你：看 st.values["messages"] == 2 条     ← 确认工具没跑
                                            │
                      invoke(None, cfg) ────┘ 放行
                                            │
                              tools 执行 → 回 agent → 最终答案 → END


【对话式 interrupt()】

invoke(用户输入) → agent_ask 开始执行 → 遇到 interrupt({问题})
                                            │
                                      ⛔ 停住（保存 checkpoint，问题抛出）
                                            │
                      你：get_state(cfg).next == ('agent',)   ← 确认停住
                      你：把问题展示给人 → 人点"拒绝"
                                            │
                      invoke(Command(resume="reject"), cfg)
                                            │
                      重放 agent_ask → interrupt() 直接返回 "reject"
                                     → 走拒绝分支 → END
```

**两者共同的三要素**：暂停（checkpoint 记录位置与 state）、判据（`.next` 非空）、恢复（**同一个 thread_id** + `None` 或 `Command(resume=...)`）。

---

## 六、完整可跑代码（两版合一）

前面是分块讲，这一节把两种方式合成一个文件，可直接复制运行。跑完你会看到四段输出：哨卡停住 → 放行执行 → 对话停住 → 拒绝收尾。

```python
"""W6-D3: HITL —— interrupt_before 哨卡 vs interrupt() 对话（两版合一）"""
import os
from dotenv import load_dotenv
load_dotenv(override=True)   # 想钉死项目根：load_dotenv(Path(__file__).resolve().parent.parent / ".env", override=True)

from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import interrupt, Command
from langchain_core.tools import tool
from langchain_openai import ChatOpenAI

# ── 危险工具：print 是"执行标记"，用来验证工具到底什么时候跑的 ──
@tool
def delete_file(path: str) -> str:
    """【危险】删除本地文件"""
    print(f"【工具真的被执行了】路径: {path}")
    return f"文件已删除: {path}"

class S(TypedDict):
    messages: Annotated[list, add_messages]

llm = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
).bind_tools([delete_file])

# 普通 agent：只负责产出决策（可能带 tool_calls）
def agent_node(state: S) -> dict:
    return {"messages": [llm.invoke(state["messages"])]}

# 会问人的 agent：节点内 interrupt()（本版无条件触发，D4 再加条件）
def agent_ask(state: S) -> dict:
    last = state["messages"][-1].content
    decision = interrupt({                       # ← 停在这一行，问题抛给调用方
        "question": f"模型想执行危险操作: {last}",
        "options": ["approve", "reject"],
    })
    if decision == "approve":
        return {"messages": [("assistant", "【已获批准】继续执行")]}
    return {"messages": [("assistant", "【已拒绝】取消删除操作")]}

def build(agent_fn, **compile_kw):
    """同一份拓扑，换个 agent 节点 / 换个编译参数就能出两版图"""
    b = StateGraph(S)
    b.add_node("agent", agent_fn)
    b.add_node("tools", ToolNode([delete_file]))
    b.add_edge(START, "agent")
    b.add_conditional_edges("agent", tools_condition)
    b.add_edge("tools", "agent")
    return b.compile(checkpointer=MemorySaver(), **compile_kw)

if __name__ == "__main__":
    # ══════ 方式一：哨卡 —— 进 tools 之前必停 ══════
    g1 = build(agent_node, interrupt_before=["tools"])
    cfg1 = {"configurable": {"thread_id": "hitl-1"}}

    g1.invoke({"messages": [("user", "帮我删掉 /tmp/a.txt")]}, cfg1)
    st = g1.get_state(cfg1)                      # 进度信息在这里，不在 invoke 返回值上
    print("① 暂停！下一步将执行:", st.next)         # ('tools',)
    print("   当前消息数:", len(st.values["messages"]))      # 2 → 还没有 ToolMessage
    # 此刻不应出现「工具真的被执行了」——出现了就说明没拦住

    r1b = g1.invoke(None, cfg1)                  # 批准：同一个 config，输入传 None
    print("   最终回复:", r1b["messages"][-1].content)

    # ══════ 方式二：对话 —— 节点内提问、收回答案 ══════
    g2 = build(agent_ask)                        # 不传 interrupt_before
    cfg2 = {"configurable": {"thread_id": "hitl-2"}}

    try:
        g2.invoke({"messages": [("user", "删掉 /tmp/b.txt")]}, cfg2)
        print("② invoke 正常返回（本版本不抛异常，但图已暂停）")
    except Exception as e:
        print("② 收到暂停信号:", type(e).__name__)   # 有的版本抛 GraphInterrupt

    st2 = g2.get_state(cfg2)
    print("   停在:", st2.next)                  # ('agent',) = 停在 agent 内部

    out2 = g2.invoke(Command(resume="reject"), cfg2)   # 人点"拒绝"
    print("   结果:", out2["messages"][-1].content)    # 【已拒绝】取消删除操作
```

**跑之前心里有数的四段预期**：

```fallback
① 暂停！下一步将执行: ('tools',)
   当前消息数: 2                      ← 此时还没出现「工具真的被执行了」
   最终回复: 文件已删除: /tmp/a.txt     ← resume 之后才出现执行标记
② invoke 正常返回 / 或 GraphInterrupt
   停在: ('agent',)
   结果: 【已拒绝】取消删除操作
```

---

## 七、踩坑记

| # | 坑 | 解法 |
|---|---|---|
| 1 | 🔴 把 `GraphInterrupt` 当报错 | 它是暂停信号；**判断停没停只看 `get_state(config).next`**，别靠 try/except |
| 2 | 🔴 用 `invoke()` 的返回值取 `.next` | 返回值是 state 值字典（dict），没有 `.next`。进度在 `get_state()` 快照上 |
| 3 | 🔴 resume 时换了 thread_id | 暂停和继续必须同一把钥匙。换号 = 开新档 = 从头跑、消息重复 |
| 4 | 🔴 没配 checkpointer | HITL 的硬前提。没有它，图停不下来也续不上 |
| 5 | 🔴 把有副作用的代码写在 `interrupt()` 之前 | resume 会重放节点，**这些代码会执行两次**（写库/发请求要特别小心） |
| 6 | ⚠️ 工具没用 `@tool` 声明 | `bind_tools` / `ToolNode` 要的是标准工具对象，裸函数会出问题 |
| 7 | ⚠️ 分不清停"之前"还是"内部" | `.next == ('tools',)` = 拦在 tools 前；`.next == ('agent',)` = 停在 agent 内 |
| 8 | ⚠️ 无条件 `interrupt()` 导致每轮都问人 | 加判断：只在模型真要调危险工具时才 interrupt（D4 改造） |

---

## 八、速查卡片（复习直接看这）

**接线**：

```python
# 哨卡式
g = builder.compile(checkpointer=MemorySaver(), interrupt_before=["tools"])
g.invoke(payload, cfg)          # 跑到哨卡前自动停
g.invoke(None, cfg)             # 放行继续

# 对话式（节点内）
decision = interrupt({"question": "...", "options": [...]})   # 停住并提问
g.invoke(Command(resume="reject"), cfg)                       # 把答案传回去
```

**三问三答**：

| 想干嘛 | 用哪个 |
|---|---|
| 看 state 内容（messages 等） | `invoke()` 返回值 或 `get_state(cfg).values` |
| 看停在哪 / 下一步去哪 | `get_state(cfg).next` |
| 判断"图到底停没停" | `.next` 非空 = 暂停中，**别靠 try/except 猜** |

**一句话速记**：

- HITL = 暂停 → 人决策 → 从暂停处继续；**checkpointer 是它的地基**
- 哨卡固定拦（只能放行），对话能问能收答案（`Command(resume=...)`）
- `interrupt()` 靠"重放节点"实现，所以它前面的代码会跑两次
- 暂停与继续，**同一个 thread_id**

---

## 九、一句话总结 + 下一篇预告

**HITL 的本质是"checkpointer 暂停 + 人工干预 + resume 恢复"：哨卡式的 `interrupt_before` 在编译期声明、固定拦在节点边界，只能放行；对话式的 `interrupt()` 在节点内部调用，能把问题抛给用户，再用 `Command(resume=值)` 把答案传回节点内部——它靠"重放节点 + 让 `interrupt()` 直接返回值"实现，所以 `interrupt()` 之前的代码会执行两次。两种方式都硬性依赖 checkpointer：没有存档，图既不知道停在哪，也没法从暂停处续跑。** 而判断"它真的停了"不能看异常，要看 `get_state(config).next` 是否非空。

W6-D4 会把今天的机制改造成真正能用的审批流：**三态审批（放行 / 拒绝 / 改参数）**——拒绝之外还能让人改掉工具参数再执行；**`update_state` 人工修正**——你 D2 学过的"外部改档"在这里成为纠正 Agent 的手段；以及**时间旅行回放**——回到某个 checkpoint 换一条路重跑。顺带把今天那个"每轮都问人"的 `agent_ask` 改造成**只在模型真想调危险工具时才 interrupt**。

**魔鬼代言人**：如果跑通之后你冒出"这不就是加了个暂停按钮吗"的念头——**机制上确实如此**，难的是工程细节：resume 时到底要传什么（`None` 还是 `Command`）、节点重放会不会把副作用跑两遍、人在半路改了主意（`update_state` 改档）会不会和原 state 冲突、以及最要命的一点——**你的审批界面怎么知道该问什么**。今天代码里那句 `print` 就是你的"审批界面"；真实产品里，它是前端弹窗、是企业微信审批流、是一张工单。**框架负责"停得住、续得上"，"问什么、怎么问、谁来答"是你的产品问题——别指望框架替你决定。**

**自查三问**（能答上 = 真懂了）：① 哨卡式暂停时 `st.next` 是 `('tools',)`，对话式是 `('agent',)`——为什么指向的节点不同？② `Command(resume="reject")` 的值"回到" `interrupt()` 那一行，实际发生了什么？这带来什么工程纪律？③ 你的版本里 `interrupt()` 不抛异常、invoke 正常返回——你该用什么判断图到底停没停？