---
description: ""
title: " LangGraph_HITL"
draft: false
date: "2026-09-10T09:54:52+08:00"
slug: "LangGraph_HITL"
categories:
 - LangGraph
tags:
 - HITL
image: ""
---

# 让图停下来问人

> 这篇是我这阶段最难的一篇。
>
> 难点不是 `interrupt()` API 本身，而是几个东西会同时发生：
>
> ```text
> 模型产生 tool_call
> ↓
> 图停下来
> ↓
> 人审批
> ↓
> 人可能修改参数
> ↓
> 工具执行
> ↓
> 模型还要知道“为什么参数变了”
> ↓
> 最后还得防止模型再次调用工具
> ```
>
> 所以这篇我不追求把代码写得多短。
>
> 我希望以后再忘记 HITL 的时候，可以直接顺着代码一块一块看回来。

---

# 一、先看今天最终要做成什么

目标非常简单：

```text
用户
 ↓
decide
 ↓
模型产生 tool_call
 ↓
approve
 ↓
人审批
 ↓
┌───────────────┐
│ approve       │
│ reject        │
│ edit          │
└───────────────┘
 ↓
tools
 ↓
audit
 ↓
summarize
 ↓
END
```

五个节点各干一件事：

| 节点          | 职责            |
| ----------- | ------------- |
| `decide`    | 模型决定要调用什么工具   |
| `approve`   | 暂停，问人         |
| `tools`     | 真正执行工具        |
| `audit`     | 记录“这是人改的”     |
| `summarize` | 最后总结，而且没有工具权限 |

先记住这个骨架。

后面所有代码都只是把这张图实现出来。

---

# 二、第一块：工具

```python
@tool
def delete_file(path: str) -> str:
    print(f">>> 工具真身执行：删除 {path}")
    return f"文件已删除：{path}"
```

这其实没什么特殊的。

为了学习，我故意不真的删除文件。

真正重要的是这行：

```python
print(f">>> 工具真身执行：删除 {path}")
```

因为我需要知道：

> **工具到底什么时候真的执行了。**

以后测试 HITL 时，我不能只看最后的结果。

我还需要观察：

```text
“工具执行”这件事，
到底发生在审批前还是审批后？
```

所以这个 `print` 其实就是我们后面的一个判据。

---

# 三、第二块：State

```python
class S(TypedDict):
    messages: Annotated[list, add_messages]
    audit_note: str
```

这里有两个字段。

## `messages`

整个对话和工具执行历史：

```text
用户消息
模型消息
tool_call
ToolMessage
审计记录
最终总结
```

所以它可以理解成：

> **主 transcript。**

---

## `audit_note`

这是专门为了 HITL 加的。

原因后面会看到：

> 人可能在图外修改参数。

但是：

```text
图外的人做了什么
```

不会自动出现在：

```text
messages
```

所以我们需要一个地方暂时保存：

```text
“审批人把 a.txt 改成了 old.txt”
```

这个地方就是：

```python
audit_note
```

---

# 四、第三块：为什么要两个 LLM？

这里是整个代码第一个特别值得注意的地方。

```python
llm_decide = ChatOpenAI(...).bind_tools(
    [delete_file]
)

llm_summarize = ChatOpenAI(...)
```

两个模型看起来差不多。

但权限完全不同。

## `llm_decide`

```python
.bind_tools([delete_file])
```

代表：

> **它拥有工具箱。**

它可以产生：

```text
tool_calls
```

所以它负责：

> “我想做什么？”

---

## `llm_summarize`

它没有：

```python
.bind_tools(...)
```

所以它没有能力产生工具调用。

它只负责：

> “事情最后怎么样了？”

这个设计是后面“结构收口”的关键。

先不用急着记，后面看到 `summarize` 就会明白。

---

# 五、第四块：`decide` —— 先让模型产生工单

代码：

```python
def decide(state: S) -> dict:
    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    print(
        f"[decide] tool_calls="
        f"{len(ai_msg.tool_calls)}"
    )

    return {
        "messages": [ai_msg]
    }
```

这里看起来非常普通。

但有一个非常关键的动作：

```python
return {
    "messages": [ai_msg]
}
```

这一步意味着：

> **模型刚才产生的 AIMessage 真正进入了 state。**

例如模型决定：

```text
delete_file("/tmp/a.txt")
```

state 里就会有：

```text
AIMessage
└── tool_calls
    └── delete_file
        └── path = /tmp/a.txt
```

这时候工具还没有执行。

所以现在可以把它理解成：

> **模型已经填好了一张待执行工单。**

---

# 六、第五块：我第一次真正卡住的地方——为什么不能把 `interrupt()` 直接写在这里？

最自然的代码其实是：

```python
def agent(state):
    ai_msg = llm.invoke(...)

    answer = interrupt(
        "确定吗？"
    )

    return {
        "messages": [ai_msg]
    }
```

乍一看没问题。

我当时的脑子也是这么想的：

```text
LLM
 ↓
拿到 ai_msg
 ↓
暂停
 ↓
人回答
 ↓
继续 return
```

但这是错误的。

---

# 七、`interrupt()` 不是普通的 Python 暂停

第一次执行：

```python
answer = interrupt(...)
```

时，LangGraph 会让当前节点暂停。

所以真实过程是：

```text
agent
 ↓
llm.invoke()
 ↓
ai_msg 只存在局部变量里
 ↓
interrupt()
 ↓
节点停止
```

注意：

```python
return {
    "messages": [ai_msg]
}
```

还没执行。

所以：

> **`ai_msg` 虽然已经计算出来，但还没有进入 state。**

这就是我第一次写 HITL 时的根本问题。

---

# 八、所以必须拆成两个节点

正确结构：

```text
decide
 ↓
return AIMessage
 ↓
state 已经有 tool_call
 ↓
approve
 ↓
interrupt()
```

所以：

```python
def decide(state: S) -> dict:
    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    return {
        "messages": [ai_msg]
    }
```

然后：

```python
def approve(state: S) -> dict:
    last = state["messages"][-1]

    tc = last.tool_calls[0]

    answer = interrupt({
        ...
    })
```

这样就非常清楚了：

### `decide`

负责：

> **产生工单并落 state。**

### `approve`

负责：

> **拿 state 里的工单问人。**

所以拆节点不是为了代码好看。

而是因为：

> **我必须先让 tool_call 成为 state，后面的人才能看到并修改它。**

---

# 九、`approve` 到底在看什么？

代码：

```python
last = state["messages"][-1]

if not (
    isinstance(last, AIMessage)
    and last.tool_calls
):
    return {}

tc = last.tool_calls[0]
```

这里不要把它想复杂。

就是：

```text
state
 ↓
拿最后一条消息
 ↓
确认它是 AIMessage
 ↓
确认里面有 tool_calls
 ↓
拿出 tool_call
```

如果：

```text
tc["args"]
```

是：

```python
{
    "path": "/tmp/a.txt"
}
```

那么审批问题就是：

```text
模型想删除 /tmp/a.txt，批准吗？
```

---

# 十、真正让图停下来的是 `interrupt()`

```python
answer = interrupt({
    "question": "...",
    "options": [
        "approve",
        "reject",
        "edit",
    ],
})
```

第一次执行时：

```text
interrupt()
 ↓
暂停
```

这时候：

```python
graph.get_state(cfg)
```

就可以去看看 checkpoint。

而恢复时：

```python
graph.invoke(
    Command(resume="approve"),
    cfg,
)
```

`"approve"` 会成为：

```python
answer
```

的值。

所以：

```python
answer = interrupt(...)
```

你可以把它理解成：

> **“我现在向图外问一个问题，以后这里会拿到一个答案。”**

---

# 十一、最容易记错的地方：恢复以后节点会重跑

这个必须单独记住。

第一次：

```text
approve()
 ↓
print("[approve]")
 ↓
interrupt()
 ↓
停
```

恢复：

```text
approve()
 ↓
print("[approve]")
 ↓
interrupt()
 ↓
得到 "approve"
 ↓
继续
```

所以你可能看到：

```text
[approve]
[approve]
```

出现两次。

这不是 Bug。

因为恢复时，节点会重新执行，然后在原来的 `interrupt()` 位置拿到已经提供的答案。

所以：

> **`interrupt()` 前面的代码必须能够安全重跑。**

---

# 十二、所以 interrupt 前最怕什么？

比如：

```python
def approve(state):

    charge_card()

    answer = interrupt(
        "确认付款吗？"
    )
```

第一次：

```text
扣款
 ↓
interrupt
 ↓
停
```

恢复：

```text
重新进入 approve
 ↓
再次扣款
 ↓
interrupt
 ↓
继续
```

可能就扣两次。

所以 interrupt 前适合：

```text
读取 state
纯计算
构建问题
打印日志
```

不适合随便放：

```text
扣款
发邮件
写数据库
创建订单
调用外部副作用 API
```

我自己的记忆：

> **interrupt 前可以想，可以读；不要随便动现实世界。**

---

# 十三、三态审批：为什么要有 `edit`？

如果只有：

```text
approve
reject
```

人只能：

```text
同意
拒绝
```

但现实里经常是：

> “可以，但参数错了。”

所以：

```text
approve
reject
edit
```

变成：

| 选择        | 含义      |
| --------- | ------- |
| `approve` | 原参数直接执行 |
| `reject`  | 不执行     |
| `edit`    | 修改参数后执行 |

这里的“edit”才是 D4 真正的核心。

---

# 十四、人修改的到底是什么？

假设 state 里已经有：

```python
last.tool_calls
```

内容：

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

工具还没有跑。

所以我们现在改的不是：

```text
“已经删除的文件”
```

而是：

> **下一步准备执行的工单。**

于是：

```python
new_tool_calls = [
    {
        **tc,
        "args": {
            **tc["args"],
            "path": "/tmp/old.txt",
        },
    }
    for tc in last.tool_calls
]
```

就是把：

```text
/tmp/a.txt
```

换成：

```text
/tmp/old.txt
```

---

# 十五、为什么不是直接修改原来的消息？

这里又有一个非常容易忽略的细节：

```python
fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tool_calls,
)
```

最重要的是：

```python
id=last.id
```

因为：

```python
messages: Annotated[list, add_messages]
```

不是简单的：

```python
messages.append(...)
```

它会根据 message id 处理更新。

所以：

```text
相同 id
→ 更新原消息
```

而：

```text
没有旧 id
→ 更像新增一条消息
```

我应该把它记成：

> **我要改工单，不是再新建一张工单。**

所以：

```python
id=last.id
```

不能随手删。

---

# 十六、`edit` 为什么没有 `if answer == "edit"`？

代码：

```python
if answer == "reject":
    return {
        "messages": [
            AIMessage(
                content="已取消删除..."
            )
        ]
    }

return {}
```

你会发现：

```text
approve
edit
```

都走：

```python
return {}
```

这是故意的。

因为：

```text
edit
```

真正发生的事情是：

```text
图外
 ↓
get_state()
 ↓
修改 tool_calls
 ↓
update_state()
 ↓
resume("edit")
```

所以 `approve` 节点根本不负责编辑。

它只是一个：

> **审批闸门。**

可以这样记：

```text
reject
→ 拦住

approve
→ 放行

edit
→ 参数已经在图外改好了
→ 放行
```

---

# 十七、`update_state()`：真正的“改档”

这行：

```python
graph.update_state(
    cfg,
    {
        "messages": [fixed],
        "audit_note": "...",
    },
)
```

不要把它理解成：

> “修改某个正在运行的 Python 变量。”

更准确的理解是：

> **直接修改 checkpoint 对应的 state。**

也就是：

```text
暂停
 ↓
拿到 checkpoint
 ↓
修改 checkpoint state
 ↓
继续
```

所以它更像：

> **图外的编辑器。**

而：

```python
Command(resume="edit")
```

只是告诉：

```python
answer = interrupt(...)
```

> “人的答案是 edit。”

这两个动作不是一个东西。

---

# 十八、这里顺便记住 `resume` 和 `update_state` 的区别

这是我这次口述里最容易混淆的地方。

### `resume`

```text
图内传值
```

比如：

```python
Command(resume="approve")
```

最后成为：

```python
answer = "approve"
```

---

### `update_state`

```text
图外改档
```

比如：

```python
graph.update_state(
    cfg,
    {"path": "/tmp/old.txt"}
)
```

它直接修改 state。

所以：

> **`resume` 是“把答案传回去”；`update_state` 是“直接改存档”。**

不要把这两个概念混起来。

---

# 十九、为什么还需要 `audit`？

现在人已经把：

```text
/tmp/a.txt
```

改成：

```text
/tmp/old.txt
```

但模型并不知道：

> **这是人改的。**

如果最后只看到：

```text
用户：
删除 a.txt

AI：
我要调用 a.txt

工具：
old.txt 删除成功
```

模型很可能疑惑：

> “为什么最后变成 old.txt 了？”

甚至可能觉得：

> “是不是我自己犯错了？”

所以加入：

```python
audit_note
```

记录：

```text
审批人把删除目标从 a.txt 改成 old.txt
```

然后 `audit` 把这件事情重新加入 transcript。

这就是：

> **给模型补一段缺失的因果链。**

---

# 二十、为什么我叫它“治叙事”？

因为 Agent 最容易出现的一种问题是：

```text
上下文前后不一致
```

而模型不知道真正原因时，很容易自己解释。

例如：

```text
用户要 A
模型原本准备 A
工具最后执行 B
```

如果没人告诉模型：

> “是审批人把 A 改成 B。”

它就可能开始自己编故事。

所以 `audit` 的作用不是让模型更聪明。

而是：

> **防止模型因为不知道中间发生过什么，而自己补一个错误的故事。**

---

# 二十一、最后一个坑：为什么 `summarize` 不绑工具？

现在事情做完了。

如果最后又让原来的：

```python
llm_decide
```

出来总结，那么它依然拥有：

```text
delete_file
```

它就可能：

```text
工具成功
 ↓
模型看到结果
 ↓
模型又产生 tool_call
 ↓
再次执行
```

我们当然可以在 prompt 里说：

```text
“请不要再调用工具。”
```

但这只是：

> **软约束。**

Prompt 是说明书，不是锁。

所以：

```python
llm_summarize = ChatOpenAI(
    **_common
)
```

故意不绑定工具。

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

最后这个模型：

> **根本没有工具可以调用。**

这就是：

> **结构收口。**

---

# 二十二、所以 `audit` 和 `summarize` 各自解决一个问题

这两个不要混：

```text
audit
→ 模型为什么看到不同参数？
→ 治叙事

summarize 无工具
→ 模型为什么不能再执行？
→ 治重试
```

一个解决：

> **理解。**

一个解决：

> **能力。**

---

# 二十三、补一个容易混的东西：`thread_id` 和 `checkpoint_id`

以后看 HITL 和 Time Travel，会经常看到两个 id。

千万别混。

```text
thread_id
=
哪一条 thread
=
哪个会话 / 存档槽
```

而：

```text
checkpoint_id
=
这一条 thread 的哪一帧
=
哪个历史 checkpoint
```

可以记成：

```text
thread_id
→ 账户号

checkpoint_id
→ 交易流水号
```

所以：

> **身份是 `thread_id`，时间是 `checkpoint_id`。**

---

# 二十四、再补一个容易混的东西：`interrupt_before`

`interrupt_before` 和 `interrupt()` 也不要混成“两个暂停 API”。

更准确地说：

|                         | `interrupt_before`     | `interrupt()`           |
| ----------------------- | ---------------------- | ----------------------- |
| 停在哪里                    | 节点边界                   | 节点内部                    |
| `Command(resume=value)` | ❌                      | ✅                       |
| `update_state()`        | ✅                      | ✅                       |
| 恢复方式                    | `invoke(None, config)` | `Command(resume=value)` |

所以：

```text
interrupt_before
没有 resume
```

不等于：

```text
不能 update_state
```

两者是两个维度。

---

# 二十五、最后一个非常重要的判据：`state.next`

很多时候我会说：

> “图好像停下来了。”

以后不要猜。

直接：

```python
state = graph.get_state(config)

print(state.next)
```

判断：

```text
next == ()
→ 图结束

next != ()
→ 还有下一步
```

比如：

```text
interrupt_before["tools"]
```

停下时：

```text
next == ("tools",)
```

表示：

> tools 还没跑。

而：

```text
interrupt()
```

如果写在：

```text
agent
```

里面，那么暂停时：

```text
next == ("agent",)
```

表示：

> agent 本身还没有完成。

所以 `.next` 是非常好用的“状态判据”。

---

# 二十六、现在把整条链重新看一遍

```text
用户
 ↓
decide
 ↓
AIMessage(tool_calls)
 ↓
approve
 ↓
interrupt()
 ↓
人
 │
 ├── reject
 │     ↓
 │    END
 │
 ├── approve
 │     ↓
 │    tools
 │
 └── edit
       ↓
    update_state
       ↓
    resume("edit")
       ↓
      tools
       ↓
      audit
       ↓
   summarize
       ↓
      END
```

这张图其实就是今天全部内容。

---

# 二十七、我最后真正记住的是这 7 句话

```text
1. decide 先把 AIMessage 写进 state，再 interrupt。

2. interrupt() 恢复时，所在节点会重新执行。

3. 人修改的是 pending tool_calls，而不是工具执行结果。

4. 修改已有消息时，要保留原 message id。

5. audit 解决“模型不知道人为什么改参数”。

6. summarize 不绑工具，解决“模型执行后继续重试”。

7. thread_id 是身份，checkpoint_id 是时间。
```

再补一句：

> **`resume` 是图内传值，`update_state` 是图外改档。**

---

# 二十八、完整源码

下面是这篇真正对应的完整源码。

代码故意保持“学习版”风格：

> 注释多一点没关系，重要的是以后重新打开时，我能顺着代码看懂。

```python
"""
LangGraph HITL 学习版
====================

目标：

1. decide：模型产生 tool_call
2. approve：interrupt() 问人
3. 三态：
   - approve
   - reject
   - edit
4. edit：修改 pending tool_calls
5. audit：记录人工修改
6. summarize：最后不绑定工具，结构性收口

注意：
delete_file() 不会真的删除文件，
只是用于观察 ToolNode 什么时候执行。
"""

import os

from dotenv import load_dotenv

load_dotenv(override=True)


# ============================================================
# LangGraph
# ============================================================

from typing import Annotated, TypedDict

from langgraph.graph import (
    StateGraph,
    START,
    END,
)

from langgraph.graph.message import add_messages

from langgraph.checkpoint.memory import MemorySaver

from langgraph.prebuilt import ToolNode

from langgraph.types import (
    interrupt,
    Command,
)


# ============================================================
# LangChain
# ============================================================

from langchain_core.tools import tool

from langchain_core.messages import (
    AIMessage,
    HumanMessage,
)

from langchain_openai import ChatOpenAI


# ============================================================
# 1. 工具
# ============================================================

@tool
def delete_file(path: str) -> str:
    """
    危险操作的学习版。

    不真的删除文件，
    只打印执行信息。
    """

    print(
        f">>> 工具真身执行：删除 {path}"
    )

    return (
        f"文件已删除：{path}"
    )


# ============================================================
# 2. State
# ============================================================

class S(TypedDict):

    # 主消息历史
    #
    # add_messages：
    # 新消息追加；
    # 如果 message id 相同，
    # 可以更新原来的消息。
    messages: Annotated[
        list,
        add_messages,
    ]

    # 人工审批留下的操作记录
    audit_note: str


# ============================================================
# 3. LLM
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


# ------------------------------------------------------------
# 决策模型
# ------------------------------------------------------------
#
# 有工具。
#
# 它可以：
#
#     tool_calls
#
# 所以它负责：
#
#     “我想做什么？”
#

llm_decide = ChatOpenAI(
    **_common
).bind_tools(
    [delete_file]
)


# ------------------------------------------------------------
# 收口模型
# ------------------------------------------------------------
#
# 没有绑定工具。
#
# 它只能总结，
# 不能再产生 delete_file。
#

llm_summarize = ChatOpenAI(
    **_common
)


# ============================================================
# 4. decide
# ============================================================

def decide(state: S) -> dict:
    """
    决策节点。

    关键原则：

    先让 AIMessage return，
    再进入 approve。

    不要在这里 interrupt。
    """

    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    print(
        "[decide] tool_calls =",
        len(ai_msg.tool_calls),
    )

    return {
        "messages": [
            ai_msg
        ]
    }


# ============================================================
# 5. approve
# ============================================================

def approve(state: S) -> dict:
    """
    审批节点。

    做三件事：

    1. 从 state 读取模型的 tool_call
    2. interrupt() 问人
    3. reject 就结束；
       approve/edit 就放行
    """

    # --------------------------------------------------------
    # 读取最后一条消息
    # --------------------------------------------------------

    last = (
        state["messages"][-1]
    )

    # 理论上这里一定应该是：
    #
    # AIMessage(tool_calls=[...])
    #

    if not (
        isinstance(
            last,
            AIMessage,
        )
        and last.tool_calls
    ):
        return {}

    # --------------------------------------------------------
    # 取第一条 tool_call
    # --------------------------------------------------------

    tc = last.tool_calls[0]

    print(
        "[approve] 待审批参数：",
        tc["args"],
    )

    # --------------------------------------------------------
    # interrupt
    # --------------------------------------------------------
    #
    # 第一次：
    #
    #     interrupt()
    #     ↓
    #     图停住
    #
    # resume：
    #
    #     节点重新执行
    #     ↓
    #     再次来到 interrupt()
    #     ↓
    #     返回 resume 值
    #

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

    # --------------------------------------------------------
    # reject
    # --------------------------------------------------------
    #
    # 返回一条没有 tool_calls 的消息。
    #
    # route_after_approve()
    # 就会判断：
    #
    # 没 tool_calls
    # → END
    #

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

    # --------------------------------------------------------
    # approve / edit
    # --------------------------------------------------------
    #
    # 两者都走这里。
    #
    # 因为：
    #
    # approve：
    #     原参数直接执行
    #
    # edit：
    #     参数已经在图外 update_state()
    #     修改完成
    #
    # 所以这里都只需要：
    #
    #     放行
    #

    return {}


# ============================================================
# 6. audit
# ============================================================

def audit(state: S) -> dict:
    """
    审计节点。

    解决：

    “为什么模型准备的参数
     和最后真正执行的参数不一样？”

    因为可能有人在图外修改了 state。
    """

    note = state.get(
        "audit_note",
        "",
    )

    # 没有人工修改记录
    if not note:
        return {}

    print(
        "[audit]",
        note,
    )

    # --------------------------------------------------------
    # 把人工动作重新放进 transcript
    # --------------------------------------------------------

    audit_message = HumanMessage(
        content=(
            f"【审批人操作记录】{note}。"
            "这是审批人主动、有意做出的决定，"
            "已获批准执行；"
            "这不是错误，无需重试，"
            "请直接据实总结结果。"
        )
    )

    return {
        "messages": [
            audit_message
        ],

        # 用完清空
        "audit_note": "",
    }


# ============================================================
# 7. summarize
# ============================================================

def summarize(state: S) -> dict:
    """
    最终收口。

    注意：

    llm_summarize 没有绑定工具。

    所以从结构上保证：
    这里不能再次产生 tool_calls。
    """

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    print(
        "[summarize] tool_calls = 0"
    )

    return {
        "messages": [
            ai_msg
        ]
    }


# ============================================================
# 8. decide 后路由
# ============================================================

def route_after_decide(
    state: S,
):

    last = (
        state["messages"][-1]
    )

    # 有工具调用
    if (
        isinstance(
            last,
            AIMessage,
        )
        and last.tool_calls
    ):
        return "approve"

    # 普通回答
    return END


# ============================================================
# 9. approve 后路由
# ============================================================

def route_after_approve(
    state: S,
):

    last = (
        state["messages"][-1]
    )

    # approve / edit：
    #
    # 仍然是带 tool_calls 的 AIMessage
    #
    # → tools
    #

    if (
        isinstance(
            last,
            AIMessage,
        )
        and last.tool_calls
    ):
        return "tools"

    # reject：
    #
    # 没有 tool_calls
    #
    # → END
    #

    return END


# ============================================================
# 10. 构建图
# ============================================================

builder = StateGraph(S)


# ------------------------------------------------------------
# 节点
# ------------------------------------------------------------

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
    ToolNode(
        [delete_file]
    ),
)

builder.add_node(
    "audit",
    audit,
)

builder.add_node(
    "summarize",
    summarize,
)


# ------------------------------------------------------------
# START → decide
# ------------------------------------------------------------

builder.add_edge(
    START,
    "decide",
)


# ------------------------------------------------------------
# decide → approve / END
# ------------------------------------------------------------

builder.add_conditional_edges(
    "decide",
    route_after_decide,
    {
        "approve": "approve",
        END: END,
    },
)


# ------------------------------------------------------------
# approve → tools / END
# ------------------------------------------------------------

builder.add_conditional_edges(
    "approve",
    route_after_approve,
    {
        "tools": "tools",
        END: END,
    },
)


# ------------------------------------------------------------
# tools → audit
# ------------------------------------------------------------

builder.add_edge(
    "tools",
    "audit",
)


# ------------------------------------------------------------
# audit → summarize
# ------------------------------------------------------------

builder.add_edge(
    "audit",
    "summarize",
)


# ------------------------------------------------------------
# summarize → END
# ------------------------------------------------------------

builder.add_edge(
    "summarize",
    END,
)


# ============================================================
# 11. 编译
# ============================================================
#
# checkpointer 非常重要。
#
# interrupt 能停下来，
# 就必须有人帮我们保存：
#
#     state
#     当前进度
#
# MemorySaver 适合学习。
#

graph = builder.compile(
    checkpointer=MemorySaver()
)


# ============================================================
# 12. thread_id
# ============================================================
#
# thread_id：
#
#     哪一条历史 / 哪个会话
#

cfg = {
    "configurable": {
        "thread_id": "w6-d4-hitl",
    }
}


# ============================================================
# 13. 第一次调用
# ============================================================
#
# 运行：
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
#

print("\n" + "=" * 60)
print("第一次运行")
print("=" * 60)

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
# 14. 查看暂停时 state
# ============================================================

st = graph.get_state(
    cfg
)

# ------------------------------------------------------------
# next
#
# 现在应该还有下一步。
# 因为图停在 approve。
# ------------------------------------------------------------

print(
    "\n当前 next =",
    st.next,
)


# ------------------------------------------------------------
# values
#
# 查看 state 内容。
# ------------------------------------------------------------

print(
    "当前消息数 =",
    len(
        st.values[
            "messages"
        ]
    ),
)


# ------------------------------------------------------------
# 最后一条应该是：
#
# AIMessage(tool_calls=[...])
# ------------------------------------------------------------

last = (
    st.values[
        "messages"
    ][-1]
)

print(
    "当前 tool_calls =",
    last.tool_calls,
)


# ============================================================
# 15. 修改 tool_calls
# ============================================================

new_tool_calls = [
    {
        # 保留原来的 tool_call 字段
        **tc,

        # 只修改 args 中的 path
        "args": {
            **tc["args"],

            "path":
                "/tmp/old.txt",
        },
    }

    for tc in last.tool_calls
]


# ============================================================
# 16. 构造新的 AIMessage
# ============================================================
#
# ★ 最重要的是：
#
#     id=last.id
#
# 表示：
#
#     “我要更新刚才那条消息。”
#
# 而不是新增一条消息。
#

fixed = AIMessage(
    id=last.id,

    content=last.content,

    tool_calls=new_tool_calls,
)


# ============================================================
# 17. update_state
# ============================================================
#
# 这里一次做两件事：
#
#     ① 改 tool_calls
#     ② 写 audit_note
#
# 也就是：
#
#     改档 + 记录原因
#

graph.update_state(
    cfg,
    {
        "messages": [
            fixed
        ],

        "audit_note": (
            "审批人把删除目标从 "
            "/tmp/a.txt 改成了 "
            "/tmp/old.txt"
        ),
    },
)


# ============================================================
# 18. resume
# ============================================================
#
# 这里不是重新发一条用户消息。
#
# 而是：
#
#     恢复之前 interrupt()
#     并让 answer 得到：
#
#         "edit"
#
# ------------------------------------------------------------

print("\n" + "=" * 60)
print("恢复执行")
print("=" * 60)

result = graph.invoke(
    Command(
        resume="edit"
    ),
    cfg,
)


# ============================================================
# 19. 查看最终结果
# ============================================================

print(
    "\n最终结果：",
    result[
        "messages"
    ][-1].content,
)


# ============================================================
# 20. 最终验收
# ============================================================

final_state = graph.get_state(
    cfg
)

# ------------------------------------------------------------
# next == ()
#
# 说明：
#     图已经结束。
# ------------------------------------------------------------

print(
    "\n最终 next =",
    final_state.next,
)


# ------------------------------------------------------------
# 最后一条消息：
#
# 应该是普通 AIMessage，
# 不应该再有 tool_calls。
# ------------------------------------------------------------

print(
    "最后一条 tool_calls =",
    getattr(
        final_state.values[
            "messages"
        ][-1],
        "tool_calls",
        None,
    ),
)
```

---

# 二十九、最后只留这一张图

```text
                    用户
                      │
                      ▼
                  ┌────────┐
                  │ decide │
                  │ 有工具  │
                  └────┬───┘
                       │
                 tool_calls
                       │
                       ▼
                  ┌────────┐
                  │ approve│
                  │  问人   │
                  └────┬───┘
                       │
             ┌─────────┼─────────┐
             │         │         │
          approve    reject     edit
             │         │         │
             │         ▼         │
             │        END        │
             │                   │
             │             update_state
             │                   │
             └──────────┬────────┘
                        ▼
                     tools
                        │
                        ▼
                      audit
                        │
                        ▼
                   summarize
                   （无工具）
                        │
                        ▼
                       END
```

然后记住四句话：

> **先 `decide`，再 `interrupt`。**

> **人改的是 pending `tool_calls`。**

> **`audit` 告诉模型为什么改，`summarize` 从结构上阻止重试。**

> **`thread_id` 是身份，`checkpoint_id` 是时间；`resume` 是图内传值，`update_state` 是图外改档。**

这篇对我来说真正难的地方，不是把 API 全记住。

而是终于把这些 API 放回了正确的位置。
