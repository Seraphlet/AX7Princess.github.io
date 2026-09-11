---
description: ""
title: " LangGraph_HITL"
draft: false
date: "2026-09-10T09:54:52+08:00"
slug: "LangGraph_HITL"
categories:
 - null
tags:
 - null
image: ""
---

# W6-D4 · 审批不只是放行：三态、改参、审计留痕与结构收口

> 审批不是简单的「是 / 否」按钮。
>
> 真实一点的审批应该有第三种权力：
>
> **「可以做，但参数得改一下。」**
>
> 而这第三种权力一旦加进 LangGraph，马上会碰到三个新问题：
>
> 1. 人改过的参数，怎么在工具执行前塞回 state？
> 2. 模型怎么知道这是人主动改的，而不是自己犯了错？
> 3. 工具执行完以后，怎么保证模型不会又自作主张重试一次？
>
> D4 做的就是把这三个坑一次填掉。
>
> 最终图只有一条主干：
>
> ```text
> 用户
>   ↓
> decide        模型决定调用什么工具
>   ↓
> approve      人审批：approve / reject / edit
>   ↓
> tools        真正执行
>   ↓
> audit        记录“人改过参数”
>   ↓
> summarize    只负责说话，不再拥有工具
>   ↓
> END
> ```
>
> 一句话：
>
> **D3 让图学会“停下来问人”，D4 则让人拥有“改单子”的能力。**

---

## 一、先把需求说清楚：D3 只有两态，不够用了

D3 的 HITL 很简单：

```text
模型：我要删 /tmp/a.txt

        ↓

人：

approve
reject
```

这叫二态审批。

但是现实里的审批经常不是这样。

比如：

```text
模型：我要删除 /tmp/a.txt

人：这个文件不对。
   改成 /tmp/old.txt，再删。
```

这里既不是 approve，也不是 reject。

而是：

```text
edit
```

所以 D4 的审批状态变成：

```text
approve   → 原参数执行

reject    → 什么都不执行

edit      → 修改待执行参数，再执行
```

到这里看起来只是多了一个按钮。

真正麻烦的是：

> **这个按钮到底改哪里？**

答案是：

> 改 state 里那条还没有执行的 `AIMessage.tool_calls`。

因为在 `ToolNode` 真正运行之前：

```python
AIMessage.tool_calls
```

还只是一张“待执行工单”。

工具甚至还没有碰到这个参数。

所以：

```text
tool_calls = 待执行工单
ToolNode   = 真正执行工单
```

中间正好留出了一个可以人工修改的窗口。

---

# 二、先看完整数据流：这次为什么突然多出来三个节点

D3 很像：

```text
agent → tools → agent
```

D4 把它拆成：

```text
START
  │
  ▼
decide
  │
  ├── 没有 tool_calls ─────────→ END
  │
  ▼
approve
  │
  ├── reject ──────────────────→ END
  │
  ▼
tools
  │
  ▼
audit
  │
  ▼
summarize
  │
  ▼
END
```

五个节点各自只做一件事：

| 节点          | 做什么                | 不负责什么    |
| ----------- | ------------------ | -------- |
| `decide`    | 让模型产生 `tool_calls` | 不负责问人    |
| `approve`   | `interrupt()` 问人   | 不真正执行工具  |
| `tools`     | 真正执行工具             | 不解释人工行为  |
| `audit`     | 把人工修改告诉模型          | 不调用工具    |
| `summarize` | 最终总结               | 根本没有工具权限 |

这里有一个很重要的设计原则：

> **一个节点解决一个问题。**

不要把：

```text
“模型决策”
+
“暂停审批”
+
“工具执行”
+
“最终总结”
```

全部塞到一个 agent 节点里。

D4 的很多坑，恰恰就是从这里开始的。

---

# 三、第一坑：`interrupt()` 不是 return，而是“把节点掐停”

这一点是 D4 最重要的基础。

很多人第一次写 HITL 会自然地认为：

```python
def agent_node(state):
    ai_msg = llm.invoke(...)
    answer = interrupt(...)
    return {"messages": [ai_msg]}
```

也就是：

```text
先算 ai_msg
↓
暂停
↓
以后继续
↓
return ai_msg
```

看起来很合理。

但实际上不是。

`interrupt()` 的行为更接近：

```text
抛出一个“暂停”信号
```

节点在这里就停了。

而这个节点此前算出来的：

```python
ai_msg
```

只是一个局部变量。

如果还没有 `return`：

> 它根本没有进入 checkpoint 里的 state。

所以你会看到一个非常诡异的现象。

---

## 报错现场

假设代码写成：

```python
def agent_node(state: S) -> dict:
    ai_msg = llm.invoke(state["messages"])

    decision = interrupt({
        "question": "批准吗？",
        "options": ["approve", "reject", "edit"],
    })

    return {"messages": [ai_msg]}
```

然后暂停之后你去：

```python
st = graph.get_state(cfg)
last = st.values["messages"][-1]
```

你可能发现：

```text
last = HumanMessage(...)
```

而不是：

```text
AIMessage(...)
```

于是：

```python
last.tool_calls
```

直接：

```text
AttributeError
```

为什么？

因为：

```text
llm.invoke()
     ↓
ai_msg 在内存里产生
     ↓
interrupt()
     ↓
节点被暂停
     ↓
还没 return
     ↓
ai_msg 根本没进入 state
```

所以：

> **`interrupt()` 之前算出来的东西，不代表已经写进 state。**

---

# 四、修法：把“决策”和“审批”拆成两个节点

正确结构：

```python
def decide(state: S) -> dict:
    ai_msg = llm_decide.invoke(state["messages"])
    return {"messages": [ai_msg]}


def approve(state: S) -> dict:
    last = state["messages"][-1]

    tc = last.tool_calls[0]

    answer = interrupt({
        "question": f"模型想删除 {tc['args']['path']}, 批准吗?",
        "options": ["approve", "reject", "edit"],
    })

    if answer == "reject":
        return {
            "messages": [
                AIMessage(content="已取消删除，本次未执行任何操作。")
            ]
        }

    return {}
```

现在时序变成：

```text
decide
  │
  │ llm.invoke()
  │
  │ return
  ▼
state
  │
  │ AIMessage(tool_calls=...)
  ▼
approve
  │
  │ interrupt()
  ▼
等待人
```

这个区别特别重要。

### `decide` 的职责

只负责：

> **产生一张工具工单。**

### `approve` 的职责

只负责：

> **审核这张工单。**

所以当程序停下来时：

```python
st.values["messages"][-1]
```

已经是：

```text
AIMessage
```

里面已经有：

```python
tool_calls
```

这时候你才真正拥有：

> **修改 pending tool call 的机会。**

---

# 五、第二坑：怎么修改模型准备执行的参数？

现在进入 D4 的核心。

假设 state 里已经有：

```python
AIMessage(
    tool_calls=[
        {
            "name": "delete_file",
            "args": {
                "path": "/tmp/a.txt"
            }
        }
    ]
)
```

工具还没执行。

人说：

```text
edit
```

我要改成：

```text
/tmp/old.txt
```

那么就从 state 里取出它：

```python
st = graph.get_state(cfg_c)

last = st.values["messages"][-1]
```

然后构造一个新的 tool call：

```python
new_tc = [
    {
        **tc,
        "args": {
            **tc["args"],
            "path": "/tmp/old.txt"
        }
    }
    for tc in last.tool_calls
]
```

注意这里我故意写成：

```python
"args": {
    **tc["args"],
    "path": "/tmp/old.txt"
}
```

而不是：

```python
"args": {
    "path": "/tmp/old.txt"
}
```

因为前一种写法是：

> 保留原参数，只覆盖 `path`。

假设以后工具变成：

```python
delete_file(
    path,
    recursive=False,
    force=False,
)
```

你只改：

```python
path
```

而不会把：

```text
recursive
force
```

一起弄丢。

---

# 六、为什么不能直接修改原来的 `AIMessage`？

因为 checkpoint 里面保存的是状态快照。

你可以把它想成：

> **已经归档的工单。**

不要直接拿红笔改档案原件。

正确姿势是：

```text
旧工单
  ↓
复制
  ↓
改副本
  ↓
用副本替换原工单
```

所以：

```python
fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tc,
)
```

然后：

```python
graph.update_state(
    cfg_c,
    {
        "messages": [fixed]
    }
)
```

这里又出现了一个非常容易踩坑的地方。

---

# 七、`id=last.id`：这是改参成功的生死线

你可能会写：

```python
fixed = AIMessage(
    content=last.content,
    tool_calls=new_tc,
)
```

看起来没问题。

但是这样会自动产生一个新的 message id。

而：

```python
messages: Annotated[list, add_messages]
```

的 reducer 会根据 message id 判断：

> 这是修改旧消息，还是新增一条消息？

所以：

```python
AIMessage(id=last.id, ...)
```

意味着：

```text
同一个 id
→ 替换原消息
```

而：

```python
AIMessage(...)
```

不带旧 id，则可能变成：

```text
旧 AIMessage
+
新 AIMessage
```

最终 state 里出现两张工单。

你以为自己“修改”了：

```text
/tmp/a.txt
```

实际上可能只是：

```text
旧工单：/tmp/a.txt

新工单：/tmp/old.txt
```

两张都还在。

所以 D4 这里记一条死规则：

> **要替换原消息，就必须保留原来的 `id`。**

---

# 八、第三坑：改完 state 之后，不能忘记 `resume`

`update_state()` 干的事情是：

> **改档案。**

它不是：

> **继续跑图。**

所以：

```python
graph.update_state(...)
```

完成以后，图还是停在那里。

然后才：

```python
graph.invoke(
    Command(resume="edit"),
    cfg_c,
)
```

整个流程是：

```text
interrupt()
   ↓
图暂停
   ↓
get_state()
   ↓
拿到 AIMessage(tool_calls)
   ↓
复制
   ↓
修改 path
   ↓
update_state()
   ↓
state 已经变成 old.txt
   ↓
Command(resume="edit")
   ↓
approve 从 interrupt 处继续
   ↓
tools
   ↓
delete_file("/tmp/old.txt")
```

顺序不要反。

必须是：

```text
update_state
    ↓
resume
```

而不是：

```text
resume
    ↓
update_state
```

否则图先恢复执行，原始参数就可能已经被继续往下传了。

---

# 九、`get_state()` 和 `invoke()`：名字只差一个，性质完全不一样

这一对 API 是 HITL 最容易搞混的地方之一。

### `get_state`

```python
st = graph.get_state(cfg)
```

意思：

> **我只想看看现在卡在哪里。**

它不会继续执行节点。

### `invoke`

```python
graph.invoke(...)
```

意思：

> **让图继续跑。**

所以：

```text
get_state = 偷看
invoke    = 开工
```

尤其在审批状态下：

不要为了“看看 state”写：

```python
graph.invoke(...)
```

因为你以为自己只是在读：

```text
“现在最后一条消息是什么？”
```

实际上可能是在：

```text
继续恢复执行
→ approve
→ tools
→ 真正删除
```

这类 bug 最危险的地方就在于：

> **代码不一定报错。**

它可能只是悄悄把工具执行了。

---

# 十、现在三态审批就完整了

我们把它整理成：

```text
                decide
                  │
            有 tool_calls?
                  │
              ┌───┴───┐
              │       │
             否       是
              │       │
              ▼       ▼
             END    approve
                       │
                 ┌─────┼─────┐
                 │     │     │
              approve reject edit
                 │     │     │
                 │     ▼     │
                 │    END     │
                 │            │
                 └─────┬──────┘
                       ▼
                     tools
```

这里最关键的一点：

### approve

不需要在 `edit` 时自己重新生成 AIMessage。

它只需要：

```python
return {}
```

为什么？

因为：

> **真正修改 tool call 的动作发生在图外的 `update_state()`。**

所以 `approve` 只负责：

```text
我收到人的决定了。

approve → 放行
edit    → 放行
reject  → 结束
```

这也是为什么代码会看起来有一点反直觉：

```python
if answer == "reject":
    return {
        "messages": [
            AIMessage(content="已取消删除...")
        ]
    }

return {}
```

因为：

```text
approve / edit
```

都意味着：

> **让 state 当前那张 AIMessage 继续流向 `tools`。**

---

# 十一、第三个问题来了：为什么模型会突然说“我误删了”？

这是 D4 最容易被忽略、但实际非常重要的一层。

假设发生：

```text
用户：
帮我删除 /tmp/a.txt

模型：
我要调用 delete_file("/tmp/a.txt")

人：
edit

人工把它改成：
delete_file("/tmp/old.txt")

工具：
文件已删除 /tmp/old.txt
```

到了最后模型看到的上下文可能只有：

```text
Human:
帮我删除 /tmp/a.txt

AI:
tool_calls = /tmp/old.txt

Tool:
文件已删除 /tmp/old.txt
```

模型一看：

```text
用户要 a.txt

我调的是 old.txt

old.txt 还真的删成功了
```

那么它最自然的解释是什么？

> “抱歉，我刚才把路径写错了。”

这其实不是模型故意撒谎。

它只是：

> **不知道中间发生过人工修改。**

因为：

```text
人工修改
```

发生在图外的：

```python
update_state()
```

而模型没有上帝视角。

模型只能看到：

```text
state 里有什么
```

看不到：

```text
state 之外发生了什么
```

所以：

> **人工干预如果没有进入 state，对模型来说就等于从未发生。**

---

# 十二、修法：加一个 `audit` 节点

所以我们给 state 多加一个字段：

```python
class S(TypedDict):
    messages: Annotated[list, add_messages]
    audit_note: str
```

其中：

```python
messages
```

负责：

> 模型和工具之间的正常消息流。

而：

```python
audit_note
```

负责：

> 人工审批发生了什么。

例如：

```python
audit_note = (
    "审批人把删除目标从 /tmp/a.txt "
    "改成了 /tmp/old.txt"
)
```

然后：

```python
def audit(state: S) -> dict:
    note = state.get("audit_note", "")

    if not note:
        return {}

    return {
        "messages": [
            HumanMessage(
                content=(
                    f"【审批人操作记录】{note}。"
                    "这是审批人主动、有意做出的决定，已获批准执行；"
                    "这不是错误，无需道歉、无需重试其他路径，"
                    "请直接据实向用户总结结果。"
                )
            )
        ],
        "audit_note": "",
    }
```

于是模型最后看到：

```text
用户：
删除 a.txt

AI：
准备删除 a.txt

工具：
删除 old.txt

审批人操作记录：
审批人主动把目标从 a.txt 改成 old.txt
```

这时候模型就不会再脑补：

```text
“我刚刚误删了。”
```

因为：

> **真正改变参数的人已经出现在上下文里。**

---

# 十三、为什么 `audit` 必须放在 `tools` 后面？

最终结构：

```text
tools
  ↓
audit
  ↓
summarize
```

而不是：

```text
AI(tool_calls)
  ↓
audit
  ↓
ToolMessage
```

原因很简单。

工具调用消息有自己的消息结构约束：

```text
AIMessage(tool_calls)
        ↓
ToolMessage
```

这两者需要正确对应。

所以不要把审计消息硬插在：

```text
AI(tool_calls)
```

和：

```text
ToolMessage
```

中间。

最安全的顺序：

```text
AI(tool_calls)
        ↓
ToolMessage
        ↓
HumanMessage(审批记录)
```

也就是：

```text
tools → audit
```

---

# 十四、但是 audit 还不够：模型可能继续重试

假设我们已经成功解决：

```text
“我误删了”
```

还有一个问题：

> 模型会不会又想调用一次 `delete_file`？

比如：

```text
工具：
文件已删除 /tmp/old.txt

模型：
等等，用户原来要删的是 /tmp/a.txt。

我要不要补救一下？
→ delete_file("/tmp/a.txt")
```

然后：

```text
再次进入 approve
再次等待人
再次调用模型
```

虽然：

> **安全边界没有被突破。**

因为第二次仍然会经过：

```text
approve
```

但是：

> **图开始空转。**

这就是 D4 的第二个病：

```text
模型自纠 → 再调工具 → 再审批 → 再调用
```

---

# 十五、真正的修法：结构上把“工具按钮”拆掉

这里不要靠 prompt：

```text
请不要再次调用工具。
```

这种方案不够硬。

因为你是在：

> 请求模型不要做某件它有能力做的事情。

D4 更好的办法是：

> **最后一个节点根本不给模型工具。**

所以我们创建两个 LLM：

```python
llm_decide = ChatOpenAI(
    **_common
).bind_tools([delete_file])


llm_summarize = ChatOpenAI(
    **_common
)
```

注意：

```text
llm_decide
```

有工具。

而：

```text
llm_summarize
```

没有工具。

这两个模型的职责完全不同。

---

# 十六、`decide` 可以“动手”，`summarize` 只能“说话”

`decide`：

```python
def decide(state: S) -> dict:
    ai_msg = llm_decide.invoke(state["messages"])
    return {"messages": [ai_msg]}
```

它可以产生：

```text
tool_calls
```

所以：

```text
decide
```

负责：

> **决定做什么。**

而：

```python
def summarize(state: S) -> dict:
    ai_msg = llm_summarize.invoke(state["messages"])
    return {"messages": [ai_msg]}
```

这个模型：

```text
没有 delete_file
没有任何工具
```

所以它只能产生：

```text
普通 AIMessage
```

不能产生：

```text
tool_calls
```

于是整个末端变成：

```text
tools
  ↓
audit
  ↓
summarize
  ↓
END
```

这里没有任何回头路。

---

# 十七、这就是“结构收口”

这两个办法一定要区分：

| 问题         | 修法               | 本质         |
| ---------- | ---------------- | ---------- |
| 模型不知道人改过参数 | `audit`          | 给模型补上下文    |
| 模型想再次调用工具  | `summarize` 不绑工具 | 从结构上消灭重试路径 |

所以：

```text
audit
```

治的是：

> **叙事。**

而：

```text
summarize
```

治的是：

> **重试。**

一个是：

```text
“模型知道发生了什么”
```

一个是：

```text
“模型根本没有能力再做什么”
```

这两个不要混成一回事。

---

# 十八、完整代码，现在再看其实没有那么复杂

下面重新把代码放一起。

```python
import os
from dotenv import load_dotenv

load_dotenv(override=True)

from typing import Annotated, TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import ToolNode
from langgraph.types import interrupt, Command

from langchain_core.tools import tool
from langchain_core.messages import AIMessage, HumanMessage
from langchain_openai import ChatOpenAI


# ============================================================
# 1. 工具
# ============================================================

@tool
def delete_file(path: str) -> str:
    """危险：删除本地文件（这里只演示，不真删）"""
    print(f">>> 工具真身执行：删除 {path}")
    return f"文件已删除：{path}"


# ============================================================
# 2. State
# ============================================================

class S(TypedDict):
    messages: Annotated[list, add_messages]

    # 普通字段：
    # 不走 messages reducer
    # 用来承载“审批人做了什么”
    audit_note: str


# ============================================================
# 3. 两个 LLM
# ============================================================

_common = {
    "model": "deepseek-v4-flash",
    "api_key": os.getenv("DEEPSEEK_API_KEY"),
    "base_url": "https://api.deepseek.com",
    "temperature": 0,
}

# 决策模型：允许调用工具
llm_decide = ChatOpenAI(**_common).bind_tools(
    [delete_file]
)

# 收口模型：不允许调用工具
llm_summarize = ChatOpenAI(**_common)


# ============================================================
# 4. decide：只负责“想”
# ============================================================

def decide(state: S) -> dict:
    ai_msg = llm_decide.invoke(state["messages"])

    print(
        f"[decide] tool_calls={len(ai_msg.tool_calls)}"
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 5. approve：只负责“问人”
# ============================================================

def approve(state: S) -> dict:

    last = state["messages"][-1]

    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return {}

    tc = last.tool_calls[0]

    print(
        f"[approve] 待审批参数：{tc['args']}"
    )

    answer = interrupt({
        "question":
            f"模型想删除 "
            f"{tc['args']['path']}，批准吗？",

        "options": [
            "approve",
            "reject",
            "edit",
        ],
    })

    # reject：
    # 返回一条没有 tool_calls 的消息
    # 后面的路由就会直接 END
    if answer == "reject":
        return {
            "messages": [
                AIMessage(
                    content=
                    "已取消删除，本次未执行任何操作。"
                )
            ]
        }

    # approve / edit：
    #
    # 都放行。
    #
    # approve：
    # 原参数直接执行。
    #
    # edit：
    # 外部 update_state 已经把参数改好了。
    return {}


# ============================================================
# 6. audit：记录人工修改
# ============================================================

def audit(state: S) -> dict:

    note = state.get(
        "audit_note",
        ""
    )

    if not note:
        return {}

    print(
        f"[audit] 注入留痕：{note}"
    )

    return {
        "messages": [
            HumanMessage(
                content=(
                    f"【审批人操作记录】{note}。"
                    "这是审批人主动、有意做出的决定，"
                    "已获批准执行；"
                    "这不是错误，无需道歉、"
                    "无需重试其他路径，"
                    "请直接据实向用户总结结果。"
                )
            )
        ],

        # 用完清空
        "audit_note": "",
    }


# ============================================================
# 7. summarize：只负责说话
# ============================================================

def summarize(state: S) -> dict:

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    print(
        "[summarize] 收口完成，"
        "tool_calls=0"
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 8. 路由
# ============================================================

def route_after_decide(state: S):

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "approve"

    return END


def route_after_approve(state: S):

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "tools"

    return END


# ============================================================
# 9. 构图
# ============================================================

builder = StateGraph(S)

builder.add_node("decide", decide)
builder.add_node("approve", approve)
builder.add_node(
    "tools",
    ToolNode([delete_file])
)
builder.add_node("audit", audit)
builder.add_node("summarize", summarize)


builder.add_edge(
    START,
    "decide"
)


builder.add_conditional_edges(
    "decide",
    route_after_decide,
    {
        "approve": "approve",
        END: END,
    }
)


builder.add_conditional_edges(
    "approve",
    route_after_approve,
    {
        "tools": "tools",
        END: END,
    }
)


builder.add_edge(
    "tools",
    "audit"
)


builder.add_edge(
    "audit",
    "summarize"
)


builder.add_edge(
    "summarize",
    END
)


graph = builder.compile(
    checkpointer=MemorySaver()
)
```

代码看起来还是一百多行。

但现在你应该能发现：

> 真正的业务逻辑其实只有五件事。

```text
decide    → 模型想干什么
approve   → 人让不让干
tools     → 真干
audit     → 人改了什么
summarize → 说完就结束
```

其余代码基本都是 LangGraph 的“接线”。

---

# 十九、场景 C：一次完整的“改参”

现在看你真正测试 D4 的部分。

先启动一轮：

```python
cfg_c = {
    "configurable": {
        "thread_id": "state-c"
    }
}

graph.invoke(
    {
        "messages": [
            ("user", "帮我删除 /tmp/a.txt")
        ]
    },
    cfg_c,
)
```

此时图会停在：

```text
approve
```

因为：

```python
interrupt(...)
```

已经暂停。

---

## 第一步：只读 state

```python
st = graph.get_state(cfg_c)
```

注意：

```python
get_state
```

只是看。

不会继续执行。

然后：

```python
last = st.values["messages"][-1]
```

现在应该拿到：

```text
AIMessage
```

而不是：

```text
HumanMessage
```

里面有：

```python
last.tool_calls
```

例如：

```python
[
    {
        "name": "delete_file",
        "args": {
            "path": "/tmp/a.txt"
        }
    }
]
```

这就是：

> **待执行工单。**

---

# 二十、第二步：复制并修改

```python
new_tc = [
    {
        **tc,
        "args": {
            "path": "/tmp/old.txt"
        },
    }
    for tc in last.tool_calls
]
```

然后：

```python
fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tc,
)
```

最关键的是：

```python
id=last.id
```

意思：

> 我要替换原来这张工单。

---

# 二十一、第三步：把“修改”和“审计”一起写进去

你的这一版代码有一个很好的细节：

```python
graph.update_state(
    cfg_c,
    {
        "messages": [fixed],

        "audit_note":
            "审批人把删除目标从 "
            "/tmp/a.txt 改成了 "
            "/tmp/old.txt",
    },
)
```

为什么两个东西要一次写？

因为：

```text
参数修改
```

和：

```text
为什么修改
```

本质上是一件审批事件的两个侧面。

如果分成两次更新：

```text
第一次：改参数
第二次：写审计
```

中间理论上就可能出现：

```text
参数已经变了
审计还没写
```

这样状态容易失去一致性。

所以：

> **改工单 + 写操作原因，一次 update_state 完成。**

---

# 二十二、第四步：恢复 `interrupt`

然后：

```python
out_c = graph.invoke(
    Command(resume="edit"),
    cfg_c,
)
```

这里：

```python
Command(resume="edit")
```

不是：

> 再次告诉模型“你要 edit”。

而是：

> **把 `edit` 这个人的答案交还给正在 interrupt 的节点。**

于是：

```python
answer = interrupt(...)
```

这一行恢复以后：

```python
answer == "edit"
```

随后：

```python
approve()
```

返回：

```python
{}
```

于是继续：

```text
tools
```

而这时候 state 里的 tool call 已经被你改过：

```text
/tmp/a.txt
        ↓
/tmp/old.txt
```

所以工具打印：

```text
>>> 工具真身执行：删除 /tmp/old.txt
```

这就证明：

> **真正执行的是人工修改后的参数。**

---

# 二十三、最后为什么一定要检查 `next`？

很多时候终端会打印：

```text
C 最终：文件已删除...
```

人一看：

> “跑完了。”

不一定。

LangGraph 里：

> **“我已经看到一条最终文本”**
>
> 不等于
>
> **“图已经真正结束”。**

所以必须：

```python
st_c = graph.get_state(cfg_c)

print(
    "next =",
    st_c.next
)
```

真正结束应该是：

```text
next = ()
```

这才叫：

> 没有下一步了。

再检查：

```python
print(
    "最后一条 tool_calls:",
    getattr(
        st_c.values["messages"][-1],
        "tool_calls",
        None,
    ),
)
```

应该得到：

```text
[]
```

也就是说：

```text
最后一条消息
= 普通 AI 回复
≠ 工具调用
```

两个条件一起成立：

```text
next == ()
tool_calls == []
```

才是真正意义上的：

> **收敛。**

---

# 二十四、D4 最终版到底解决了哪三个问题？

现在可以把整个 D4 压缩成一张表。

| 问题         | 原因                            | D4 修法                           |
| ---------- | ----------------------------- | ------------------------------- |
| 人改不了工具参数   | `tool_calls` 没进入 state，或者拿错状态 | `decide` 先 `return`，再 `approve` |
| 模型不知道谁改了参数 | `update_state` 是图外操作          | `audit_note` + `audit`          |
| 模型执行完还想再试  | 最后的模型仍然拥有工具                   | `summarize` 使用不绑工具的 LLM         |

这三个问题分别对应：

```text
① 改参
② 叙事
③ 重试
```

不要把它们混在一起。

---

# 二十五、D4 最值得记住的三句话

### 1. `interrupt()` 前算出来，不等于已经进 state

所以：

```text
decide → return
          ↓
       state
          ↓
      approve → interrupt
```

而不是：

```text
agent → llm.invoke → interrupt
```

---

### 2. `update_state()` 改的是“待执行工单”

工具还没执行时：

```python
AIMessage.tool_calls
```

就是：

> 下一步准备做什么。

所以改它，本质上就是：

> **改模型准备执行的动作。**

---

### 3. 防止重试不要只靠提示词

```python
llm_summarize = ChatOpenAI(...)
```

不绑工具。

这样：

```text
summarize
```

即使产生了：

> “我应该再删一次……”

它也没有办法产生：

```python
tool_calls
```

因为：

> **按钮已经被拆掉了。**

---

# 二十六、最后，把整个 D4 用一句话说完

D3：

```text
模型想调用危险工具
        ↓
停下来问人
        ↓
批准 / 拒绝
```

D4：

```text
模型想调用危险工具
        ↓
停下来问人
        ↓
approve
reject
edit
        ↓
edit 就修改 pending tool_calls
        ↓
记录“这是人改的”
        ↓
工具执行
        ↓
不给最后的模型工具权限
        ↓
只能总结
        ↓
END
```

所以 D4 真正升级的不是：

> “审批多了一个 edit 按钮。”

而是把 HITL 从：

> **人能不能放行**

升级成了：

> **人能够在执行前接管模型准备执行的动作，同时让模型知道这个改变来自人，并在执行后把工具能力物理收口。**

这才是一个真正可控的 HITL 工作流。

---

## 二十七、D4 的最终拓扑

最后再看一次：

```text
                         ┌──────────────┐
                         │    decide    │
                         │  LLM + tools │
                         └──────┬───────┘
                                │
                   有 tool_calls？
                         │
              ┌──────────┴──────────┐
              │                     │
             NO                    YES
              │                     │
              ▼                     ▼
             END                 approve
                                  │
                      ┌───────────┼───────────┐
                      │           │           │
                   approve      reject       edit
                      │           │           │
                      │           ▼           │
                      │          END          │
                      │                       │
                      └───────────┬───────────┘
                                  ▼
                                tools
                                  │
                                  ▼
                                audit
                                  │
                                  ▼
                             summarize
                           （无 tools）
                                  │
                                  ▼
                                 END
```

这张图就是 W6-D4。

前面的 D3 学的是：

```text
“怎么停下来问人”
```

D4 学的是：

```text
“人停下来以后，
到底能改什么，
怎么把修改写回去，
怎么告诉模型发生了什么，
以及怎么保证最后真的收口。”
```

下一步继续往 D5 走时，就可以把这个“手工作坊版”的 `audit_note` 和 `MemorySaver`，升级成真正的持久化审批记录、可观测性和多用户审批系统。
