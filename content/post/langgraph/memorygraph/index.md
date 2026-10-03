---
description: ""
title: "MemoryGraph + HITL 审批 + 长期记忆端到端验证复盘"
draft: false
date: "2026-09-13T13:53:21+08:00"
slug: "MemoryGraph"
categories:
 - LangGraph
 - Runtime
tags:
 - Memory
image: ""
---

# LangGraph W6-D6：Checkpoint、HITL 与长期记忆的一次端到端实验

> 本节目标：把前面学习过的几个能力真正串起来：
>
> * checkpoint 持久化：让 Agent 可以跨进程恢复会话
> * HITL 人工审批：让危险操作执行前需要人工确认
> * 长期记忆：让信息真正脱离某一次会话保存
> * 可验证性：不只看 Agent 说了什么，而是直接检查外部世界的真实状态

这一节对我来说，没有特别多新的 LangGraph API。

真正变化的是：

> **开始把前面学过的几个机制放到同一个完整系统里思考。**

以前更关注的是：

```text
这个 API 怎么用？
这个节点怎么写？
这个图怎么跑？
```

这一节开始需要进一步问：

```text
状态保存在哪里？
这个动作为什么允许执行？
人工审批到底停在哪里？
重新启动以后怎么继续？
最后到底有没有真的改变外部世界？
```

所以最终形成的 Agent 不再只是：

```text
用户
 ↓
模型
 ↓
工具
```

而逐渐变成：

```text
用户
 ↓
Agent 决策
 ↓
判断风险
 ↓
必要时人工审批
 ↓
执行工具
 ↓
改变外部世界
 ↓
验证真实结果
```

---

# 一、为什么 Agent 有了工具以后，需要“审批”？

前面的 Agent 已经可以：

* 调用工具
* 保存状态
* 恢复上下文
* 跨进程继续执行

但是工具能力越强，另一个问题就越明显：

> **如果 Agent 可以直接修改外部世界怎么办？**

例如：

```text
删除文件
发送邮件
修改数据库
写入长期记忆
```

这些操作和普通查询有一个很重要的区别：

```text
查询
 ↓
读取外部世界
 ↓
通常没有副作用
```

而：

```text
写入 / 删除 / 修改
 ↓
改变外部世界
 ↓
产生副作用
```

所以真正需要控制的不是：

> Agent 能不能调用工具。

而是：

> **Agent 提出的工具调用，什么时候可以真正执行。**

因此可以增加一层权限控制：

```text
                 Agent 决策
                     │
               判断工具风险
                     │
          ┌──────────┴──────────┐
          │                     │
       安全工具              危险工具
          │                     │
       直接执行             interrupt
                                │
                            人工确认
                                │
                         Command(resume)
                                │
                             执行工具
```

这里的 HITL 并不是单纯“弹一个确认框”。

它实际上是在控制：

```text
Control Flow
```

也就是：

> **这条执行路径到底有没有资格继续往下走。**

---

# 二、这一天最大的提升：验证方式发生了变化

之前学习 HITL 时，我很自然会用：

```python
print("删除文件")
```

作为工具执行证据。

如果看到：

```text
删除文件
```

就说明工具执行了。

如果没看到：

```text
删除文件
```

就认为工具没有执行。

但这个验证方式其实比较弱。

因为：

```text
日志 ≠ 真实状态
```

日志可能：

* 漏掉
* 打错
* 被误导
* 只能说明某段代码走到了，而不能完全证明外部状态正确

所以这一节开始把验证标准升级成：

> **不要只看 Agent 说了什么，也不要只看日志，直接检查外部世界。**

例如长期记忆：

批准：

```text
alice@x.com
```

应该存在。

拒绝：

```text
X99
```

不应该存在。

于是最终的验证：

```python
assert any("alice@x.com" in t for t in texts)
assert not any("X99" in t for t in texts)
```

这里验证的已经不是：

```text
工具有没有打印？
```

而是：

```text
批准
 ↓
外部数据真的发生变化

拒绝
 ↓
外部数据真的没有发生变化
```

这也是我这一天对测试理解比较明显的一次变化：

```text
过程证据
    ↓
日志、print、next、消息数

结果证据
    ↓
直接读取外部数据
```

如果任务本身有一个明确的外部结果，直接检查结果通常更有说服力。

---

# 三、Checkpoint 记忆和长期记忆不是一回事

这是今天最容易混淆的地方之一。

之前已经学过 checkpoint，因此很容易产生一个直觉：

> checkpoint 能保存以前的信息，那它是不是就是长期记忆？

不是。

---

## 1. checkpoint 保存的是一次执行过程的状态

这一节使用：

```text
SqliteSaver
```

保存 checkpoint。

并且通过：

```text
thread_id
```

区分不同会话。

例如：

```text
thread = alice-1
```

这个槽位里保存：

```text
消息
状态
执行位置
暂停点
……
```

因此 checkpoint 更像：

> **这个 Agent 的某一次会话，现在执行到哪里，以及之前发生过什么。**

例如：

```text
第一次

用户：
我的邮箱是 alice@x.com

        ↓

checkpoint 保存
```

下一次：

```text
用户：
我的邮箱是什么？

        ↓

使用同一个 thread_id
        ↓
恢复原来的会话状态
```

---

# 四、长期记忆又是什么？

这次实验另外使用：

```text
w6d6_long_term_mem.json
```

作为一个最小化的长期记忆库。

它和 checkpoint 最大的区别是：

> **它不属于某一个 thread。**

例如：

```text
alice-1
   │
   └──────┐
          ↓
      长期记忆库
          ↑
          │
   ┌──────┘
   │
carol-1
```

Alice 的会话写进去的信息，Carol 的新会话也可以查询。

所以可以先简单理解成：

```text
                 Agent Memory
                      │
          ┌───────────┴───────────┐
          │                       │
      checkpoint              long-term memory
          │                       │
       会话状态                  长期信息
          │                       │
      thread_id                 user_id
          │                       │
       当前上下文                跨会话访问
```

因此：

```text
checkpoint
```

解决的是：

> **这一次 Agent 执行过程如何保存和恢复？**

而：

```text
long-term memory
```

解决的是：

> **这个用户的信息如何脱离某一次会话长期保存？**

---

# 五、为什么必须专门设计一个“新 thread”来验证长期记忆？

假设：

```text
Alice
thread_id = alice-1
```

已经告诉 Agent：

```text
我的邮箱是 alice@x.com
```

然后还是：

```text
thread_id = alice-1
```

再问：

```text
我的邮箱是什么？
```

如果答对了：

```text
alice@x.com
```

我们其实不能确定到底是谁提供了这个答案。

可能是：

```text
checkpoint
 ↓
恢复了之前的对话
 ↓
模型知道邮箱
```

也可能是：

```text
long-term memory
 ↓
查询出了邮箱
```

所以真正的测试应该是：

```text
Alice
 ↓
alice-1
 ↓
保存邮箱
```

然后：

```text
Carol
 ↓
carol-1
 ↓
一个全新的 checkpoint
 ↓
询问邮箱
```

如果 Carol 仍然能找到：

```text
alice@x.com
```

才能证明：

```text
信息来自 long-term memory
```

而不是 Alice 的 checkpoint。

所以这个实验实际上是在主动把两个概念隔离开：

```text
同 thread
 ↓
验证 checkpoint
```

```text
新 thread
 ↓
验证 long-term memory
```

这是这次实验设计里比较重要的一点。

---

# 六、为什么不能所有工具都需要审批？

如果所有工具一律审批：

```text
搜索资料
 ↓
请批准

查询天气
 ↓
请批准

读取长期记忆
 ↓
请批准
```

Agent 虽然安全，但是基本没法正常使用。

所以更合理的方式是：

```text
工具
 ↓
判断是否产生副作用
 ↓
┌───────────────┴───────────────┐
安全                           危险
 ↓                               ↓
直接执行                       人工审批
```

这次实验把：

```text
remember_fact
```

定义为危险工具。

而：

```text
search_memory
```

是只读工具。

所以：

```python
DANGEROUS = {
    "remember_fact"
}
```

真正的思想不是“把某个工具名字记进一个集合”。

而是：

> **不同工具应该有不同的权限等级。**

现在使用硬编码只是为了完成实验。

未来真正工程化以后，可以继续发展成：

```text
Tool Registry
      ↓
Tool Metadata
      ↓
Permission System
      ↓
判断是否需要人工审批
```

---

# 七、HITL 到底停在哪里？

这一节重新使用了：

```python
interrupt()
```

它最大的特点是：

> **暂停发生在节点内部。**

例如：

```text
approve 节点
    ↓
读取模型提出的工具调用
    ↓
interrupt()
    ↓
图暂停
    ↓
保存 checkpoint
```

代码核心：

```python
answer = interrupt({
    "question": f"模型想执行 {todo}，批准吗？",
    "options": ["approve", "reject"],
})
```

执行到这里以后，不会继续往下执行。

之后等待：

```python
Command(resume="approve")
```

或者：

```python
Command(resume="reject")
```

再恢复。

---

# 八、interrupt() 和 interrupt_before 的区别

这一部分之前已经学过，这次放到实际审批场景里理解会更加直观。

## interrupt()

位置：

```text
节点内部
```

例如：

```text
approve 节点
    ↓
判断金额
    ↓
金额 > 10000
    ↓
interrupt()
```

适合动态业务逻辑。

---

## interrupt_before

位置：

```text
节点边界
```

例如：

```text
A
 ↓
B
```

可以在进入 B 之前暂停。

所以可以先记空间关系：

```text
interrupt()
    = 节点里面停

interrupt_before
    = 节点前面停
```

---

# 九、阶段 A：先验证 SqliteSaver 的跨进程持久化

这一阶段故意把事情做得非常简单。

不加审批，不加长期记忆。

只验证：

> **换一个进程以后，同一个 thread 能不能恢复之前的状态？**

实验：

```text
write
 ↓
保存 Alice 的邮箱

read
 ↓
新进程
同 thread
询问邮箱

other
 ↓
新 thread
询问邮箱
```

这里真正重要的代码只有几个部分。

首先：

```python
conn = sqlite3.connect(
    DB,
    check_same_thread=False
)

saver = SqliteSaver(conn)
```

然后编译：

```python
graph = build_graph(saver)
```

真正把 checkpoint 接进图：

```python
return b.compile(checkpointer=saver)
```

之后：

```python
cfg = {
    "configurable": {
        "thread_id": THREAD
    }
}
```

最后：

```python
graph.invoke(
    {"messages": [("user", question)]},
    cfg
)
```

所以这里需要把三个东西串起来：

```text
SqliteSaver
    ↓
保存 checkpoint


thread_id
    ↓
告诉系统保存/读取哪个会话


config
    ↓
invoke 时把这个身份传进去
```

并不是：

> “SQLite 自动让所有会话共享记忆。”

而是：

> **SQLite 保存 checkpoint，而 thread_id 决定你正在访问哪个 checkpoint 槽位。**

---

# 十、阶段 B：在危险工具前加入 HITL

阶段 B 开始真正把 HITL 接进控制流。

整体：

```text
START
  ↓
decide
  ↓
有没有 tool_calls？
  ↓
approve
  ↓
人工确认
  ↓
tools
  ↓
summarize
  ↓
END
```

关键点是：

```text
模型不是直接执行工具
```

而是：

```text
模型
 ↓
提出 tool_call
 ↓
审批节点
 ↓
决定是否放行
 ↓
ToolNode
```

所以真正控制危险动作的，不是模型本身。

而是：

```text
图的控制流
```

---

# 十一、拒绝时为什么不能随便返回一条 AIMessage？

这一节里遇到一个非常值得记录的坑。

最直觉的写法可能是：

```python
if answer == "reject":
    return {
        "messages": [
            AIMessage(content="已取消")
        ]
    }
```

但是模型已经产生了：

```text
AIMessage
  └── tool_calls
```

这意味着后面需要有与这些 `tool_calls` 对应的工具消息回执。

也就是：

```text
AIMessage
  └── tool_call_id = abc

        ↓

ToolMessage
  └── tool_call_id = abc
```

因此拒绝时最终返回：

```python
ToolMessage(
    content="用户拒绝执行本次调用",
    tool_call_id=tc["id"]
)
```

这个坑让我更清楚地意识到：

> **tool call 不是模型的一句普通文本，而是一套消息协议。**

模型声明：

```text
我要调用工具 X
```

那么后续消息历史中就需要有与之对应的回执。

---

# 十二、阶段 C：长期记忆 + HITL + 跨进程

阶段 C 才是真正的端到端实验。

同时存在：

```text
SQLite
    ↓
checkpoint / 会话状态

JSON
    ↓
long-term memory / 长期信息
```

同时存在两个工具：

```text
search_memory
    ↓
只读
    ↓
安全
    ↓
直接执行


remember_fact
    ↓
写入
    ↓
有副作用
    ↓
需要审批
```

因此完整的流程变成：

```text
用户
 ↓
decide
 ↓
判断 tool_calls
 ↓
是否包含危险工具？
 ├── 否 → tools
 │
 └── 是 → approve
             ↓
          interrupt
             ↓
        人工 approve/reject
             ↓
          tools / END
             ↓
         summarize
             ↓
            END
```

---

# 十三、这次实验最终怎么证明“系统真的正确”？

实验设计了两条路径。

## 路径一：批准

```text
Alice
 ↓
记住：我是 Alice，邮箱 alice@x.com
 ↓
remember_fact
 ↓
interrupt
 ↓
approve
 ↓
真正写入长期记忆
```

最终检查：

```text
alice@x.com
```

存在。

---

## 路径二：拒绝

```text
Bob
 ↓
记住：我的工号是 X99
 ↓
remember_fact
 ↓
interrupt
 ↓
reject
 ↓
工具不能真正执行
```

最终检查：

```text
X99
```

不存在。

因此：

```python
assert any("alice@x.com" in t for t in texts)

assert not any("X99" in t for t in texts)
```

这两条断言实际上证明了两件不同的事：

```text
批准路径确实产生了副作用
```

以及：

```text
拒绝路径没有产生副作用
```

---

# 十四、这一节让我对 Agent 的理解发生了什么变化？

如果只总结 API：

```text
SqliteSaver
interrupt
Command
ToolNode
ToolMessage
```

那么以后很容易又变成“看过，但不会真正串起来”。

现在我更倾向于从四层理解：

```text
第一层：State
    ↓
当前 Agent 有什么状态？
状态保存在哪里？


第二层：Control Flow
    ↓
下一步允许进入哪个节点？


第三层：Permission
    ↓
这个动作有没有资格真正执行？


第四层：Verification
    ↓
外部世界最后到底有没有发生变化？
```

所以一个成熟一点的 Agent，不应该只问：

> “模型能不能调用工具？”

而应该继续问：

```text
谁决定调用？
↓
谁允许执行？
↓
执行前能不能暂停？
↓
暂停后能不能恢复？
↓
执行后怎么证明真的发生了？
```

---

# 十五、W6-D6 的几个核心结论

## 1. checkpoint ≠ long-term memory

checkpoint：

```text
保存一次执行过程
```

long-term memory：

```text
保存脱离某次会话的长期信息
```

---

## 2. thread_id 是会话隔离的重要标识

```text
thread A
    ↓
A 的 checkpoint

thread B
    ↓
B 的 checkpoint
```

换 thread 才能真正测试：

```text
“这个信息是不是来自长期记忆”
```

---

## 3. HITL 本质上是在控制执行路径

不是：

```text
工具 + 一个确认弹窗
```

而是：

```text
模型提出动作
 ↓
路由判断
 ↓
危险操作进入审批
 ↓
批准才能继续
```

---

## 4. tool call 是消息协议的一部分

模型产生：

```text
tool_calls
```

以后，历史消息必须满足对应的回填关系。

所以拒绝时不能随便伪造一条 AIMessage 结束。

---

## 5. 验证不能只看日志

更强的验证方式：

```text
查看真实外部状态
```

例如：

```python
assert ...
```

直接检查长期记忆库。

---

# 总结：从“Agent 能运行”到“Agent 可以被控制”

W6-D6 表面上是在组合：

```text
Checkpoint
+
HITL
+
Long-term Memory
```

但我觉得真正重要的是：

```text
Agent 能运行
        ↓
Agent 能保存状态
        ↓
Agent 能跨进程恢复
        ↓
Agent 能调用工具
        ↓
Agent 可以被人工控制
        ↓
Agent 的副作用能够被验证
```

所以今天真正学到的不是：

> “怎么让 Agent 记住东西。”

而是：

> **当 Agent 开始拥有改变外部世界的能力以后，状态管理、权限控制和可验证性必须一起考虑。**

最终可以把今天的 Agent 理解成：

```text
用户
 ↓
模型决策
 ↓
判断动作
 ↓
权限控制
 ↓
人工审批（必要时）
 ↓
执行工具
 ↓
修改外部世界
 ↓
验证真实结果
```

这也是我目前对 W6-D6 最核心的理解：

> **Agent 一旦拥有改变世界的能力，就必须同时拥有状态管理、权限控制和可验证性。**

---

# 附：W6-D6 完整实验代码

下面保留本节三阶段完整代码。

这里不是为了让文章变成长篇代码教程，而是为了方便以后复习时，**直接从代码入口开始画执行流程，再回到前面的概念解释**。

---

## 阶段 A：SqliteSaver 跨进程持久化

运行方式：

```bash
python w6d6_a_persist.py write
python w6d6_a_persist.py read
python w6d6_a_persist.py other
```

完整代码：

```python
"""W6-D6 阶段A: 把图接上 SqliteSaver —— 跨【进程】持久化
用法(三次运行, 三个独立进程):
    python w6d6_a_persist.py write     # ① 让 agent 记住邮箱
    python w6d6_a_persist.py read      # ② 新进程问邮箱  ← 同 thread
    python w6d6_a_persist.py other     # ③ 反例: 换个 thread 问  ← 应该不知道
"""

import os
import sys
import sqlite3
from typing import Annotated, TypedDict

from dotenv import load_dotenv
load_dotenv(override=True)

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.sqlite import SqliteSaver
from langchain_openai import ChatOpenAI


DB = "memory_agent.db"


# ══════ 1. State ══════
class S(TypedDict):
    messages: Annotated[list, add_messages]


# ══════ 2. 节点 ══════
# 这里只做最小占位图，只为了验证持久化机制。

_llm = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)


def agent_node(state: S) -> dict:
    return {
        "messages": [
            _llm.invoke(state["messages"])
        ]
    }


# ══════ 3. 构图 ══════

def build_graph(saver):
    b = StateGraph(S)

    b.add_node("agent", agent_node)

    b.add_edge(START, "agent")
    b.add_edge("agent", END)

    return b.compile(
        checkpointer=saver
    )


# ══════ 4. 主流程 ══════

def main(mode: str):

    # ① 打开 SQLite
    conn = sqlite3.connect(
        DB,
        check_same_thread=False
    )

    saver = SqliteSaver(conn)

    if hasattr(saver, "setup"):
        saver.setup()

    graph = build_graph(saver)

    # ② thread_id 决定会话槽位
    THREAD = (
        "alice-1.2"
        if mode != "other"
        else "bob-1.2"
    )

    cfg = {
        "configurable": {
            "thread_id": THREAD
        }
    }

    # ③ 三种输入
    if mode == "write":

        question = (
            "记住：我叫 Alice，"
            "邮箱是 alice@x.com。"
            "只回复'已记住'。"
        )

    elif mode == "read":

        question = "我的邮箱是什么？"

    else:

        question = "我的邮箱是什么？"

    # ④ invoke 时必须传 config
    out = graph.invoke(
        {
            "messages": [
                ("user", question)
            ]
        },
        cfg
    )

    print(
        f"\n[{mode}] "
        f"thread_id = {THREAD}"
    )

    print(
        "回复:",
        out["messages"][-1].content
    )

    # ⑤ 使用客观量检查状态
    st = graph.get_state(cfg)

    n = len(
        st.values["messages"]
    )

    for i in st.values["messages"]:
        try:
            print(
                "第条消息是:",
                i.content,
                sep="\n"
            )
        except Exception as e:
            print(
                "出错啦",
                e
            )

    print(
        f"[算的] state 里共 {n} 条消息 "
        f"next={st.next}"
    )

    conn.close()

    return n


if __name__ == "__main__":
    mode = (
        sys.argv[1]
        if len(sys.argv) > 1
        else "read"
    )

    main(mode)
```

---

# 阶段 B：危险工具审批 + 跨进程恢复

运行方式：

```bash
python w6d6_b_hitl.py ask
python w6d6_b_hitl.py status
python w6d6_b_hitl.py yes

python w6d6_b_hitl.py ask2
python w6d6_b_hitl.py no
```

完整代码：

```python
"""W6-D6 阶段B: 危险操作审批 + 【跨进程】续批

用法(五条命令, 五个独立进程):
    python w6d6_b_hitl.py ask
    python w6d6_b_hitl.py status
    python w6d6_b_hitl.py yes
    python w6d6_b_hitl.py ask2
    python w6d6_b_hitl.py no
"""

import os
import sys
import sqlite3
from typing import Annotated, TypedDict

from dotenv import load_dotenv
load_dotenv(override=True)

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.prebuilt import ToolNode
from langgraph.types import interrupt, Command
from langchain_core.tools import tool
from langchain_core.messages import AIMessage
from langchain_openai import ChatOpenAI


DB = "memory_agent.db"


# ══════ 1. 危险工具 ══════

def delete_file(path: str) -> str:
    """
    危险操作。
    演示版不真正删除文件，只打印工具执行证据。
    """

    print(
        f">>> 工具真身执行: 删除 {path}"
    )

    return f"文件已删除: {path}"


class S(TypedDict):
    messages: Annotated[list, add_messages]


# ══════ 2. 两个 LLM ══════

_llm_kwargs = dict(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)

llm_decide = (
    ChatOpenAI(**_llm_kwargs)
    .bind_tools([delete_file])
)

llm_summarize = ChatOpenAI(
    **_llm_kwargs
)


# ══════ 3. 四个节点 ══════

def decide(state: S) -> dict:

    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    print(
        f"[decide] "
        f"tool_calls={len(ai_msg.tool_calls)}"
    )

    return {
        "messages": [ai_msg]
    }


def approve(state: S) -> dict:

    last = state["messages"][-1]

    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return {}

    tc = last.tool_calls[0]

    print(
        "[approve] "
        "从存档里读到的待审参数:",
        tc["args"]
    )

    answer = interrupt({
        "question": (
            f"模型想删除 "
            f"{tc['args']['path']}，批准吗？"
        ),
        "options": [
            "approve",
            "reject"
        ],
    })

    print(
        "[approve] 收到回答:",
        repr(answer)
    )

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

    return {}


def summarize(state: S) -> dict:

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    print(
        "[summarize] 收口完成"
    )

    return {
        "messages": [ai_msg]
    }


# ══════ 4. 路由 ══════

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


# ══════ 5. 构图 ══════

def build_graph(saver):

    b = StateGraph(S)

    b.add_node(
        "decide",
        decide
    )

    b.add_node(
        "approve",
        approve
    )

    b.add_node(
        "tools",
        ToolNode([delete_file])
    )

    b.add_node(
        "summarize",
        summarize
    )

    b.add_edge(
        START,
        "decide"
    )

    b.add_conditional_edges(
        "decide",
        route_after_decide,
        {
            "approve": "approve",
            END: END
        }
    )

    b.add_conditional_edges(
        "approve",
        route_after_approve,
        {
            "tools": "tools",
            END: END
        }
    )

    b.add_edge(
        "tools",
        "summarize"
    )

    b.add_edge(
        "summarize",
        END
    )

    return b.compile(
        checkpointer=saver
    )


# ══════ 6. thread 对应关系 ══════

THREAD_OF = {

    "ask":
        "hitl-alice",

    "status":
        "hitl-alice",

    "yes":
        "hitl-alice",

    "ask2":
        "hitl-bob",

    "no":
        "hitl-bob",
}


ASK_TEXT = (
    "请删除 /tmp/report.txt"
)


# ══════ 7. 查看 checkpoint ══════

def peek(graph, cfg, tag):

    st = graph.get_state(cfg)

    msgs = (
        st.values.get(
            "messages",
            []
        )
        or []
    )

    print(
        f"\n[{tag}] "
        f"next={st.next} "
        f"消息数={len(msgs)}"
    )

    for i, m in enumerate(msgs):

        head = (
            str(m.content)
            .replace("\n", "")
            [:40]
        )

        tc = getattr(
            m,
            "tool_calls",
            None
        )

        print(
            f"   [{i}] "
            f"{type(m).__name__:12s} "
            f"{head!r}"
        )

    return st


# ══════ 8. 主流程 ══════

def main(mode: str):

    conn = sqlite3.connect(
        DB,
        check_same_thread=False
    )

    saver = SqliteSaver(conn)

    if hasattr(saver, "setup"):
        saver.setup()

    graph = build_graph(saver)

    THREAD = THREAD_OF[mode]

    cfg = {
        "configurable": {
            "thread_id": THREAD
        }
    }

    print(
        f"[{mode}] "
        f"thread_id = {THREAD} "
        f"(PID={os.getpid()})"
    )

    if mode in ("ask", "ask2"):

        graph.invoke(
            {
                "messages": [
                    ("user", ASK_TEXT)
                ]
            },
            cfg
        )

        print(
            "→ 图已停在审批点，"
            "本进程即将退出"
        )

    elif mode == "status":

        print(
            "→ 纯读模式："
            "只从 sqlite 读取暂停现场"
        )

    elif mode == "yes":

        out = graph.invoke(
            Command(
                resume="approve"
            ),
            cfg
        )

        print(
            "→ 最终回复:",
            out["messages"][-1].content
        )

    elif mode == "no":

        out = graph.invoke(
            Command(
                resume="reject"
            ),
            cfg
        )

        print(
            "→ 最终回复:",
            out["messages"][-1].content
        )

    else:

        print(
            f"未知模式: {mode}"
        )

        sys.exit(1)

    peek(
        graph,
        cfg,
        mode
    )

    conn.close()


if __name__ == "__main__":

    main(
        sys.argv[1]
        if len(sys.argv) > 1
        else "status"
    )
```

---

# 阶段 C：长期记忆 + HITL + 跨会话验证

运行方式：

```bash
python w6d6_c_e2e.py remember
python w6d6_c_e2e.py yes
python w6d6_c_e2e.py recall

python w6d6_c_e2e.py bob
python w6d6_c_e2e.py no

python w6d6_c_e2e.py carol
python w6d6_c_e2e.py verify
```

完整代码：

```python
"""W6-D6 阶段C: 跨会话记忆 + 审批 端到端

七个独立进程:
    remember
    yes
    recall
    bob
    no
    carol
    verify
"""

import os
import sys
import json
import time
import sqlite3

from typing import (
    Annotated,
    TypedDict
)

from dotenv import load_dotenv
load_dotenv(override=True)

from langchain_core.messages import (
    AIMessage,
    SystemMessage,
    ToolMessage
)

from langgraph.graph import (
    StateGraph,
    START,
    END
)

from langgraph.graph.message import (
    add_messages
)

from langgraph.checkpoint.sqlite import (
    SqliteSaver
)

from langgraph.prebuilt import (
    ToolNode
)

from langgraph.types import (
    interrupt,
    Command
)

from langchain_core.tools import tool
from langchain_openai import ChatOpenAI


DB = "memory_agent.db"

MEM = (
    "w6d6_long_term_mem.json"
)


# ══════ 1. 长期记忆库 ══════

def _load_mem() -> list:

    if not os.path.exists(MEM):
        return []

    with open(
        MEM,
        "r",
        encoding="utf-8"
    ) as f:

        return json.load(f)


def _save_mem(items: list) -> None:

    with open(
        MEM,
        "w",
        encoding="utf-8"
    ) as f:

        json.dump(
            items,
            f,
            ensure_ascii=False,
            indent=2
        )


# ══════ 2. 两个工具 ══════

@tool
def search_memory(query: str) -> str:

    """
    安全·只读。
    在长期记忆库里检索。
    """

    items = _load_mem()

    print(
        f">>> [安全工具] "
        f"检索 {query!r}"
        f"→ 库里共 {len(items)} 条"
    )

    if not items:
        return "长期记忆是空的"

    return (
        "长期记忆库内容:\n"
        + "\n".join(
            f"- {it['text']}"
            for it in items
        )
    )


@tool
def remember_fact(text: str) -> str:

    """
    危险·写入。
    写入长期记忆库。
    执行前需要人工审批。
    """

    items = _load_mem()

    items.append({
        "text": text,
        "ts": time.time()
    })

    _save_mem(items)

    print(
        ">>> [危险工具真身] "
        f"已写入长期记忆: {text}"
    )

    return f"已记住：{text}"


# ══════ 3. 危险工具名单 ══════

DANGEROUS = {
    "remember_fact"
}


class S(TypedDict):
    messages: Annotated[
        list,
        add_messages
    ]


# ══════ 4. 两个 LLM ══════

_llm_kwargs = dict(
    model="deepseek-v4-flash",
    api_key=os.getenv(
        "DEEPSEEK_API_KEY"
    ),
    base_url=(
        "https://api.deepseek.com"
    ),
    temperature=0,
)


llm_decide = (
    ChatOpenAI(**_llm_kwargs)
    .bind_tools(
        [
            search_memory,
            remember_fact
        ],
        parallel_tool_calls=False
    )
)


llm_summarize = ChatOpenAI(
    **_llm_kwargs
)


SYS = (
    "你是一个带长期记忆的助手。规则:\n"
    "1. 用户说『记住：X』时，你必须"
    "立即调用 remember_fact 工具把 X 写进长期记忆库；\n"
    "2. 用户询问之前记录的信息时，"
    "如果上下文里没有答案，必须调用 search_memory；\n"
    "3. 工具完成后，用一句简短中文总结结果。"
)


# ══════ 5. 节点 ══════

def decide(state: S) -> dict:

    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    names = [
        tc["name"]
        for tc in ai_msg.tool_calls
    ]

    print(
        "[decide] "
        f"tool_calls={len(ai_msg.tool_calls)} "
        f"{names}"
    )

    return {
        "messages": [ai_msg]
    }


def approve(state: S) -> dict:

    last = state["messages"][-1]

    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return {}

    todo = [
        tc["name"]
        for tc in last.tool_calls
    ]

    print(
        "[approve] "
        f"待审批的工具调用：{todo}"
    )

    answer = interrupt({
        "question": (
            f"模型想执行 {todo}，批准吗？"
        ),
        "options": [
            "approve",
            "reject"
        ],
    })

    print(
        "[approve] 收到回答:",
        repr(answer)
    )

    if answer == "reject":

        return {
            "messages": [
                ToolMessage(
                    content=(
                        "用户拒绝执行本次调用，"
                        "已取消"
                    ),
                    tool_call_id=tc["id"]
                )
                for tc in last.tool_calls
            ]
        }

    return {}


def summarize(state: S) -> dict:

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    print(
        "[summarize] 收口完成"
    )

    return {
        "messages": [ai_msg]
    }


# ══════ 6. 路由 ══════

def route_after_decide(state: S):

    last = state["messages"][-1]

    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return END

    if any(
        tc["name"] in DANGEROUS
        for tc in last.tool_calls
    ):
        return "approve"

    return "tools"


def route_after_approve(state: S):

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "tools"

    return END


# ══════ 7. 构图 ══════

def build_graph(saver):

    b = StateGraph(S)

    b.add_node(
        "decide",
        decide
    )

    b.add_node(
        "approve",
        approve
    )

    b.add_node(
        "tools",
        ToolNode(
            [
                search_memory,
                remember_fact
            ]
        )
    )

    b.add_node(
        "summarize",
        summarize
    )

    b.add_edge(
        START,
        "decide"
    )

    b.add_conditional_edges(
        "decide",
        route_after_decide,
        {
            "approve": "approve",
            "tools": "tools",
            END: END
        }
    )

    b.add_conditional_edges(
        "approve",
        route_after_approve,
        {
            "tools": "tools",
            END: END
        }
    )

    b.add_edge(
        "tools",
        "summarize"
    )

    b.add_edge(
        "summarize",
        END
    )

    return b.compile(
        checkpointer=saver
    )


# ══════ 8. thread 对应关系 ══════

THREAD_OF = {

    "remember":
        "alice-1",

    "yes":
        "alice-1",

    "status":
        "alice-1",

    "recall":
        "alice-1",

    "bob":
        "bob-1",

    "no":
        "bob-1",

    "carol":
        "carol-1",
}


# ══════ 9. 查看 checkpoint ══════

def peek(graph, cfg, tag):

    st = graph.get_state(cfg)

    msgs = (
        st.values.get(
            "messages",
            []
        )
        or []
    )

    print(
        f"\n[{tag}] "
        f"next={st.next} "
        f"消息数={len(msgs)}"
    )

    for i, m in enumerate(msgs):

        head = (
            str(m.content)
            .replace("\n", " ")
            [:44]
        )

        tc = getattr(
            m,
            "tool_calls",
            None
        )

        print(
            f"   [{i}] "
            f"{type(m).__name__:14s} "
            f"{head!r}"
            + (
                f"  tool_calls={tc}"
                if tc
                else ""
            )
        )

    return st


# ══════ 10. 最终验证 ══════

def verify():

    items = _load_mem()

    print(
        "长期记忆库当前内容："
    )

    for it in items:
        print(
            "   -",
            it["text"]
        )

    if not items:
        print("   (空)")

    texts = [
        it["text"]
        for it in items
    ]

    assert any(
        "alice@x.com" in t
        for t in texts
    ), (
        "❌ alice 的邮箱不在库里"
    )

    assert not any(
        "X99" in t
        for t in texts
    ), (
        "❌ bob 的工号在库里"
    )

    print(
        "\n✅ 断言通过："
        "批准的那条在库里，"
        "拒绝的那条不在库里"
    )


# ══════ 11. 测试输入 ══════

FIRST_ASK = {

    "remember":
        "记住：我是 Alice，邮箱 alice@x.com",

    "bob":
        "记住：我的工号是 X99",

    "carol":
        "我不记得之前存过什么了，"
        "帮我查一下长期记忆库里有没有关于邮箱的记录",
}


# ══════ 12. 主流程 ══════

def main(mode: str):

    if mode == "verify":
        return verify()

    conn = sqlite3.connect(
        DB,
        check_same_thread=False
    )

    saver = SqliteSaver(conn)

    if hasattr(saver, "setup"):
        saver.setup()

    graph = build_graph(saver)

    THREAD = THREAD_OF[mode]

    cfg = {
        "configurable": {
            "thread_id": THREAD
        }
    }

    print(
        f"[{mode}] "
        f"thread_id = {THREAD} "
        f"(PID={os.getpid()})"
    )

    if mode in FIRST_ASK:

        graph.invoke(
            {
                "messages": [
                    SystemMessage(
                        content=SYS
                    ),
                    (
                        "user",
                        FIRST_ASK[mode]
                    )
                ]
            },
            cfg
        )

        print(
            f"→ [{mode}] 本轮结束"
        )

    elif mode == "status":

        print(
            "→ 纯读模式："
            "只从 sqlite 读取暂停现场"
        )

    elif mode == "yes":

        out = graph.invoke(
            Command(
                resume="approve"
            ),
            cfg
        )

        print(
            "→ 最终回复:",
            out["messages"][-1].content
        )

    elif mode == "no":

        out = graph.invoke(
            Command(
                resume="reject"
            ),
            cfg
        )

        print(
            "→ 最终回复:",
            out["messages"][-1].content
        )

    elif mode == "recall":

        out = graph.invoke(
            {
                "messages": [
                    (
                        "user",
                        "我的邮箱是什么？"
                    )
                ]
            },
            cfg
        )

        print(
            "→ 最终回复:",
            out["messages"][-1].content
        )

    else:

        print(
            f"未知模式: {mode}"
        )

        sys.exit(1)

    peek(
        graph,
        cfg,
        mode
    )

    conn.close()


if __name__ == "__main__":

    main(
        sys.argv[1]
        if len(sys.argv) > 1
        else "status"
    )
```

---

# 最后留给自己的复习入口

以后重新看 W6-D6，我不准备从 API 名字开始背，而是先问自己：

```text
1. Checkpoint 保存的到底是什么？
2. thread_id 为什么必须一致？
3. 为什么换 thread 才能证明长期记忆？
4. interrupt 到底停在哪里？
5. resume 之后是怎么继续的？
6. 为什么危险工具需要审批？
7. 为什么拒绝 tool_call 时需要 ToolMessage？
8. 为什么最后要直接检查长期记忆文件？
```

如果这几个问题能够重新顺着代码画出：

```text
START
 ↓
decide
 ↓
route
 ↓
approve
 ↓
interrupt
 ↓
resume
 ↓
tools
 ↓
summarize
 ↓
END
```

那么这一节才算真正重新掌握。

> **W6-D6 的核心不是“让 Agent 记住”，而是理解：Agent 一旦拥有改变世界的能力，就必须同时拥有状态管理、权限控制和可验证性。**
