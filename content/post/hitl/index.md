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

# 让图停下来问人

> 今天终于把 LangGraph 的 HITL 串起来了。
>
> 前面一直觉得 `interrupt()`、`Command(resume)`、`update_state`、`checkpoint` 是几个分散的 API，直到把代码跑通后才发现，它们其实只是在解决三个问题：
>
> ```text
> 什么时候停？
> 停下来以后怎么继续？
> 人能不能修改下一步？
> ```
>
> 这篇不展开讲理论，直接从代码看。

---

# 一、先看最终流程

今天的代码最后是：

```text
START
  ↓
decide
  ↓
approve
  ↓
tools
  ↓
audit
  ↓
summarize
  ↓
END
```

其中：

```text
decide
→ 模型决定要调用什么工具

approve
→ 人审批

tools
→ 真正执行

audit
→ 记录“这是审批人改的”

summarize
→ 最后总结，而且没有工具权限
```

今天还有一个额外实验：

```text
checkpoint
   ↓
get_state_history()
   ↓
挑历史存档
   ↓
Replay / Fork
```

---

# 二、为什么 `decide` 和 `approve` 必须拆开？

先看：

```python
def decide(state):
    ai_msg = llm.invoke(state["messages"])
    return {"messages": [ai_msg]}
```

它只做一件事：

> **让模型产生 `tool_calls`，然后先写进 state。**

例如：

```text
AIMessage
└── tool_calls
    └── delete_file("/tmp/a.txt")
```

下一步才进入：

```python
def approve(state):
    answer = interrupt(...)
```

这里有一个非常容易踩坑的地方：

> `interrupt()` 恢复时，所在节点会重新执行。

所以不能：

```python
def agent(state):
    ai_msg = llm.invoke(...)
    answer = interrupt(...)
```

因为第一次执行到 `interrupt()` 时，还没 `return`，`ai_msg` 还没进入 state。

所以要拆成：

```text
decide
→ 先落 state

approve
→ 再 interrupt
```

这样人暂停时，state 里已经有完整的 `AIMessage.tool_calls`。

---

# 三、`interrupt()` 到底怎么恢复？

第一次：

```text
approve
 ↓
interrupt()
 ↓
暂停
```

人回答：

```python
Command(resume="approve")
```

恢复以后不是简单地从 `interrupt()` 下一行继续，而是：

```text
approve
 ↓
重新执行节点
 ↓
再次来到 interrupt()
 ↓
这次得到 "approve"
 ↓
继续
```

所以：

```text
[approve]
[approve]
```

出现两次是正常的。

这也意味着：

> **`interrupt()` 前面最好不要放不可安全重跑的副作用。**

---

# 四、三态审批：`approve / reject / edit`

现在人不只是：

```text
approve
reject
```

还可以：

```text
edit
```

例如模型原本准备：

```text
delete_file("/tmp/a.txt")
```

人说：

```text
edit
```

然后图外：

```python
st = graph.get_state(cfg)

last = st.values["messages"][-1]

fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tool_calls,
)

graph.update_state(
    cfg,
    {
        "messages": [fixed],
        "audit_note": "审批人把 a.txt 改成了 old.txt",
    },
)
```

这里最重要的是：

```python
id=last.id
```

因为 `add_messages` 会按照 message id 更新消息。

保留原 id：

```text
原消息 → 替换
```

不保留：

```text
原消息 + 新消息
```

所以：

> **改消息时，记得保留原 `id`。**

---

# 五、为什么 `edit` 没有自己的分支？

代码其实是：

```python
if answer == "reject":
    return {
        "messages": [
            AIMessage(content="已取消删除")
        ]
    }

return {}
```

没有：

```python
if answer == "edit":
```

因为真正的“修改参数”已经在：

```python
graph.update_state(...)
```

里完成了。

所以：

```text
reject
→ 拦住

approve
→ 放行

edit
→ 图外已经修改
→ 放行
```

`approve` 是**审批闸门**，不是编辑器。

---

# 六、为什么还要 `audit`？

人把：

```text
/tmp/a.txt
```

改成：

```text
/tmp/old.txt
```

这个动作发生在图外。

模型如果不知道这个变化，就可能觉得：

> “为什么我刚才突然删了 old.txt？”

所以：

```python
audit_note
```

专门记录：

```text
审批人主动把删除目标改成了 /tmp/old.txt
```

然后 `audit` 把它重新放进消息历史。

所以：

```text
audit
=
给模型补上“为什么参数发生变化”的上下文
```

这就是我这里说的：

> **治叙事。**

---

# 七、为什么 `summarize` 不绑定工具？

如果最后还是：

```python
llm.bind_tools(...)
```

模型有可能：

```text
工具执行
 ↓
模型
 ↓
又产生 tool_calls
 ↓
再执行
```

单靠提示词：

```text
不要重试
```

属于软约束。

所以最后换成：

```python
llm_summarize = ChatOpenAI(...)
```

不绑定工具。

于是：

```text
tools
 ↓
audit
 ↓
summarize
 ↓
END
```

最后这个模型根本没有工具可以调用。

> **Prompt 是说明书，不是锁。**

---

# 八、`thread_id` 和 `checkpoint_id` 终于分清了

这个是我这次口述里答错的地方。

不要再把两个 id 混在一起。

```text
thread_id
=
我是谁
=
哪个存档槽
```

```text
checkpoint_id
=
我在哪一帧
=
这个槽里的哪个 checkpoint
```

可以记成：

```text
thread_id      → 账户号
checkpoint_id  → 交易流水号
```

所以：

> **身份是 `thread_id`，时间是 `checkpoint_id`。**

---

# 九、`interrupt_before` 和 `interrupt()` 也不要混

两者最大的区别不是“谁能修改 state”。

真正应该分两个维度：

|                | `interrupt_before`     | `interrupt()`           |
| -------------- | ---------------------- | ----------------------- |
| 停在哪里           | 节点边界                   | 节点内部                    |
| `resume` 传值    | ❌                      | ✅                       |
| `update_state` | ✅                      | ✅                       |
| 恢复             | `invoke(None, config)` | `Command(resume=value)` |

所以：

```text
interrupt_before
→ 没有 resume 通道
≠
不能 update_state
```

这一点一定要记住。

---

# 十、`.values / .next / .config`

执行：

```python
st = graph.get_state(cfg)
```

以后，先看三个东西：

```python
st.values
st.next
st.config
```

我现在的记法：

```text
.values
→ 内容

.next
→ 下一步

.config
→ 这份状态的定位信息
```

尤其是：

```python
st.next
```

它是判断图有没有结束的好工具：

```text
next != ()
→ 还有工作

next == ()
→ 已结束
```

例如：

```text
interrupt_before["tools"]

next == ("tools",)
```

说明：

> agent 已经跑完，tools 还没跑。

而：

```text
interrupt() 写在 agent 内

next == ("agent",)
```

说明：

> agent 本身还没结束。

---

# 十一、checkpoint 为什么能变成“时间旅行”？

有了：

```python
checkpointer = MemorySaver()
```

每个线程会保存历史 checkpoint。

于是：

```python
graph.get_state_history(config)
```

就可以拿到存档列表。

然后：

```python
snap.config
```

可以作为这份存档的钥匙。

所以：

```python
graph.invoke(
    None,
    snap.config,
)
```

就是：

> **从这个历史 checkpoint 继续。**

这就是 Replay。

---

# 十二、怎么证明真的只重跑了 B/C？

不能只说：

> “我看日志。”

应该写断言：

```python
assert "→ B 执行" in txt_replay
assert "→ C 执行" in txt_replay
assert "→ A 执行" not in txt_replay
```

最关键的是：

```text
A 没出现。
```

因为：

```text
B 出现 + C 出现
```

只能证明它们跑了。

而：

```text
A 没出现
```

才能证明：

> **没有从头跑。**

这也是今天我最想留下来的一个工程习惯：

> **不仅验证“应该发生什么”，还要验证“不应该发生什么”。**

---

# 十三、Replay 和 Fork

Replay：

```python
graph.invoke(
    None,
    target.config,
)
```

意思：

> 原样读档。

Fork：

```python
new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    },
)

graph.invoke(
    None,
    new_cfg,
)
```

意思：

> 改档后，再发展一条新的未来。

所以：

```text
Replay
= 读档重玩

Fork
= 读档 + 改属性 + 重玩
```

而且原来的 checkpoint 还在。

---

# 十四、今天最终记住这 8 句话

```text
1. checkpointer = HITL / checkpoint 的存档基础

2. thread_id = 哪个 thread / 哪个存档槽

3. checkpoint_id = 这个 thread 的哪一帧

4. interrupt_before = 节点边界哨卡

5. interrupt() = 节点内部对话

6. resume = 图内传值
   update_state = 图外改档

7. replay = 原样从历史 checkpoint 继续

8. fork = 修改历史 state 后产生另一条未来
```

再加一句验证原则：

> **日志里“没有 A”，和“出现 B/C”一样重要。**

---

# 十五、完整源码

下面这份就是今天的核心实验代码。

为了学习方便，我把注释写得比较详细，但业务逻辑本身尽量保持简单。

```python
"""
W6-D4：LangGraph HITL
三态审批 + 改参 + 审计留痕 + 结构收口

学习目标：

1. interrupt()
2. Command(resume=...)
3. update_state()
4. tool_calls 改参
5. audit
6. 无工具 summarize
"""

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
    """
    演示工具。

    不真的删除文件，只打印执行结果。
    """

    print(f">>> 工具真身执行：删除 {path}")

    return f"文件已删除：{path}"


# ============================================================
# 2. State
# ============================================================

class S(TypedDict):

    # 对话历史
    #
    # add_messages 会负责消息的追加 / 更新。
    messages: Annotated[list, add_messages]

    # 普通字段：
    # 专门记录审批人的操作。
    audit_note: str


# ============================================================
# 3. 两个 LLM
# ============================================================

_common = {
    "model": os.getenv(
        "DEEPSEEK_MODEL",
        "deepseek-v4-flash",
    ),
    "api_key": os.getenv(
        "DEEPSEEK_API_KEY",
    ),
    "base_url": os.getenv(
        "DEEPSEEK_BASE_URL",
        "https://api.deepseek.com",
    ),
    "temperature": 0,
}


# 决策模型：
# 有工具。
llm_decide = ChatOpenAI(
    **_common
).bind_tools(
    [delete_file]
)


# 收口模型：
# 没有工具。
#
# 这样最后只能总结，
# 不能再次产生 tool_calls。
llm_summarize = ChatOpenAI(
    **_common
)


# ============================================================
# 4. decide
# ============================================================

def decide(state: S) -> dict:
    """
    决策官。

    只负责：
    让模型决定要不要调用工具。

    注意：
    这里不能先 interrupt，
    因为 AIMessage 需要先 return，
    才会真正进入 state。
    """

    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    print(
        "[decide] tool_calls =",
        len(ai_msg.tool_calls),
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 5. approve
# ============================================================

def approve(state: S) -> dict:
    """
    审批官。

    读取上一条 AIMessage，
    找到模型准备执行的 tool_call，
    然后问人。
    """

    last = state["messages"][-1]

    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return {}

    tc = last.tool_calls[0]

    print(
        "[approve] 待审批参数：",
        tc["args"],
    )

    # interrupt 第一次运行会暂停。
    #
    # resume 后这个节点会重新执行，
    # 再次来到这里时，
    # interrupt() 会直接返回 resume 值。
    answer = interrupt(
        {
            "question": (
                f"模型想删除 "
                f"{tc['args']['path']}，"
                f"批准吗？"
            ),
            "options": [
                "approve",
                "reject",
                "edit",
            ],
        }
    )

    # reject：
    # 返回一条没有 tool_calls 的消息。
    #
    # 后面路由看到没有 tool_calls，
    # 就会 END。
    if answer == "reject":

        return {
            "messages": [
                AIMessage(
                    content=(
                        "已取消删除，"
                        "本次未执行任何操作。"
                    )
                )
            ]
        }

    # approve / edit：
    #
    # 两者都直接放行。
    #
    # edit 真正修改参数的动作，
    # 已经在图外 update_state() 完成。
    return {}


# ============================================================
# 6. audit
# ============================================================

def audit(state: S) -> dict:
    """
    把人工修改重新注入消息历史。

    解决：
    “为什么实际参数和原来不一样？”
    """

    note = state.get(
        "audit_note",
        "",
    )

    if not note:
        return {}

    print(
        "[audit]",
        note,
    )

    return {
        "messages": [
            HumanMessage(
                content=(
                    f"【审批人操作记录】{note}。"
                    "这是审批人主动、有意做出的决定，"
                    "已获批准执行。"
                    "这不是错误，无需重试，"
                    "请直接总结实际结果。"
                )
            )
        ],

        # 用完清空
        "audit_note": "",
    }


# ============================================================
# 7. summarize
# ============================================================

def summarize(state: S) -> dict:
    """
    收口。

    这里故意使用没有绑定工具的 LLM。
    """

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    print(
        "[summarize] tool_calls = 0"
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 8. decide 后的路由
# ============================================================

def route_after_decide(state: S):

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "approve"

    return END


# ============================================================
# 9. approve 后的路由
# ============================================================

def route_after_approve(state: S):

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "tools"

    return END


# ============================================================
# 10. 构图
# ============================================================

builder = StateGraph(S)

builder.add_node(
    "decide",
    decide,
)

builder.add_node(
    "approve",
    approve,
)

builder.add_node(
    "tools",
    ToolNode([delete_file]),
)

builder.add_node(
    "audit",
    audit,
)

builder.add_node(
    "summarize",
    summarize,
)


# START → decide

builder.add_edge(
    START,
    "decide",
)


# decide → approve / END

builder.add_conditional_edges(
    "decide",
    route_after_decide,
    {
        "approve": "approve",
        END: END,
    },
)


# approve → tools / END

builder.add_conditional_edges(
    "approve",
    route_after_approve,
    {
        "tools": "tools",
        END: END,
    },
)


# tools → audit → summarize → END

builder.add_edge(
    "tools",
    "audit",
)

builder.add_edge(
    "audit",
    "summarize",
)

builder.add_edge(
    "summarize",
    END,
)


# ============================================================
# 11. 编译
# ============================================================

graph = builder.compile(
    checkpointer=MemorySaver()
)


# ============================================================
# 12. C 场景：edit
# ============================================================

cfg = {
    "configurable": {
        "thread_id": "w6-d4-edit",
    }
}


# ------------------------------------------------------------
# 第一次运行
#
# 图会走到：
#
# START
#   ↓
# decide
#   ↓
# approve
#   ↓
# interrupt
#
# 然后暂停。
# ------------------------------------------------------------

graph.invoke(
    {
        "messages": [
            (
                "user",
                "帮我删除 /tmp/a.txt",
            )
        ],
        "audit_note": "",
    },
    cfg,
)


# ============================================================
# 13. 查看暂停时的 state
# ============================================================

st = graph.get_state(cfg)

print("\n当前 next =", st.next)

print(
    "当前消息数量 =",
    len(st.values["messages"]),
)

last = st.values["messages"][-1]

print(
    "当前 tool_calls =",
    last.tool_calls,
)


# ============================================================
# 14. 修改 pending tool_call
# ============================================================

new_tool_calls = [
    {
        **tc,

        # 只修改 path，
        # 其它参数保留。
        "args": {
            **tc["args"],
            "path": "/tmp/old.txt",
        },
    }
    for tc in last.tool_calls
]


# ------------------------------------------------------------
# ★ 关键：
# 必须保留原来的 message id。
#
# 表示：
# “更新刚才那条消息”
#
# 而不是：
# “新增一条消息”
# ------------------------------------------------------------

fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tool_calls,
)


# ============================================================
# 15. 改档 + 写审计
# ============================================================

graph.update_state(
    cfg,
    {
        "messages": [fixed],

        "audit_note": (
            "审批人把删除目标从 "
            "/tmp/a.txt 改成了 "
            "/tmp/old.txt"
        ),
    },
)


# ============================================================
# 16. resume
# ============================================================
#
# edit 已经体现在 state 里。
#
# resume("edit") 只是告诉 approve：
# “人的答案是 edit。”
#
# 然后 approve 放行。
# ============================================================

result = graph.invoke(
    Command(
        resume="edit"
    ),
    cfg,
)


print(
    "\n最终结果：",
    result["messages"][-1].content,
)


# ============================================================
# 17. 最终验收
# ============================================================

final_state = graph.get_state(cfg)

print(
    "\n最终 next =",
    final_state.next,
)

print(
    "最后一条 tool_calls =",
    getattr(
        final_state.values["messages"][-1],
        "tool_calls",
        None,
    ),
)

# 理论上的最终状态应该：
#
# next == ()
#
# 表示图结束。
```

---

# 四十三、这一篇我以后真正需要回看的地方

不用重新读全文。

直接看这张：

```text
┌──────────────────────────────────────┐
│           D3 / D4 速查               │
├──────────────────────────────────────┤
│ thread_id       = 哪个会话 / 存档槽   │
│ checkpoint_id   = 哪一帧             │
│                                      │
│ interrupt_before = 节点前哨卡         │
│ interrupt()      = 节点内部问人       │
│                                      │
│ resume           = 图内传值            │
│ update_state     = 图外改档            │
│                                      │
│ values           = state 内容          │
│ next             = 下一步              │
│ config           = 定位这份快照的钥匙   │
│                                      │
│ Replay = 原样读档                     │
│ Fork   = 改档后产生新未来              │
└──────────────────────────────────────┘
```

以及最重要的两个判据：

```python
# 图有没有结束？
state.next == ()

# Replay 是不是从头跑了？
assert "→ A 执行" not in replay_log
```

这就够了。

> **以后忘了，就回来查；不用重新学一遍。**
