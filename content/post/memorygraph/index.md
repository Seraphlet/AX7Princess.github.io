---
description: ""
title: "MemoryGraph + HITL 审批 + 长期记忆端到端验证复盘"
draft: true
date: "2026-09-13T13:53:21+08:00"
slug: "MemoryGraph"
categories:
 - LangGraph
 - Runtime
tags:
 - Memory
image: ""
---

# LangGraph W6-D6：MemoryGraph + HITL 审批 + 长期记忆端到端验证复盘

> 本节目标：把前面学习的两个能力组合起来：
>
> * checkpoint 持久化：让 Agent 可以跨进程恢复会话
> * HITL 人工审批：让危险操作执行前需要人工确认
>
> 最终形成一个更接近生产 Agent 的结构：
>
> **用户 → Agent 决策 → 判断风险 → 必要时人工审批 → 执行工具 → 修改真实世界**

---

# 一、今天解决的问题：Agent 为什么需要审批？

前面的 Agent 已经可以：

* 调工具
* 保存状态
* 恢复上下文

但是还有一个问题：

> 如果 Agent 有权限修改外部世界怎么办？

例如：

* 删除文件
* 发送邮件
* 修改数据库
* 写入长期记忆

这些操作和普通查询不同：

```
查询数据
    ↓
不会改变外部世界

写入数据
    ↓
改变外部世界
    ↓
需要确认
```

所以生产 Agent 通常不会让模型直接执行所有工具，而是增加一个权限层：

```
                ┌──────────────┐
                │ Agent 决策     │
                └──────┬───────┘
                       │
              判断工具危险程度
                       │
        ┌──────────────┴──────────────┐
        │                             │
     安全工具                      危险工具
        │                             │
    直接执行                    interrupt暂停
                                      │
                                  人工确认
                                      │
                                Command(resume)
                                      │
                                  执行工具
```

---

# 二、本节最大的提升：验证方式升级

之前 HITL 阶段 B：

验证：

> 工具有没有执行？

例如：

```python
print("删除文件")
```

没有打印：

说明工具没有运行。

但是这个验证有问题：

```
日志 ≠ 真实状态
```

因为：

* print 可以漏
* 日志可能骗人
* 模型回复也可能骗人

所以今天升级：

> 不看 Agent 说了什么，直接检查外部世界。

例如长期记忆：

```python
memory.json
```

批准：

```
alice@x.com
```

应该存在。

拒绝：

```
X99
```

不应该存在。

最终验证：

```python
assert "alice@x.com" in texts

assert "X99" not in texts
```

这才是真正证明：

```
审批成功 → 外部世界改变

审批拒绝 → 外部世界没有改变
```

---

# 三、Checkpoint 记忆 vs 长期记忆

今天最大的概念区别：

## 1. checkpoint 记忆

来源：

```
SqliteSaver
```

绑定：

```
thread_id
```

作用：

保存一次会话过程。

例如：

第一次：

```
用户：
我的邮箱是 alice@x.com

checkpoint 保存
```

第二次：

```
用户：
我的邮箱是什么？
```

同一个 thread：

Agent 可以从 checkpoint 找回来。

特点：

```
属于一次会话
```

---

## 2. 长期记忆

来源：

例如：

```
memory.json
```

或者：

* Redis
* PostgreSQL
* 向量数据库

特点：

```
不绑定 thread_id
```

多个会话可以访问。

例如：

```
alice-1 会话
        |
        写入长期记忆

carol-1 会话
        |
        查询长期记忆
```

---

所以 Agent 记忆其实是两层：

```
              Agent Memory

                  |
        ┌─────────┴─────────┐
        │                   │

 checkpoint            long-term memory

 会话记忆                用户记忆

 thread_id             user_id

 短期上下文             长期知识
```

---

# 四、分级审批设计

不是所有工具都需要审批。

如果所有工具都拦：

```
查询天气
    ↓
请批准

搜索资料
    ↓
请批准

读取信息
    ↓
请批准
```

用户体验会非常差。

所以设计：

```python
DANGEROUS = {
    "remember_fact"
}
```

危险工具：

```
写入长期记忆
```

需要审批。

安全工具：

```
search_memory
```

直接执行。

路由：

```python
def route_after_decide(state):

    if tool_name in DANGEROUS:
        return "approve"

    return "tools"
```

核心思想：

> 有副作用的操作需要确认，没有副作用的操作可以自动执行。

---

# 五、HITL 核心代码

## interrupt 暂停

节点内部：

```python
def approve(state):

    answer = interrupt({
        "question":"是否执行?",
        "options":["approve","reject"]
    })

    return {}
```

执行到这里：

```
图暂停

状态保存 checkpoint

等待人工输入
```

---

## Command(resume)

人工选择：

```python
graph.invoke(
    Command(resume="approve"),
    config
)
```

恢复：

```
interrupt继续执行
```

---

注意：

`interrupt()` 和 `interrupt_before` 不一样。

## interrupt()

位置：

```
节点内部
```

适合：

业务逻辑中的动态审批。

例如：

```
判断金额 > 10000

        ↓

interrupt()
```

---

## interrupt_before

位置：

```
节点边界
```

例如：

```
A节点

↓

B节点
```

在进入 B 前暂停。

区别：

```
interrupt()
        节点里面暂停

interrupt_before
        节点之间暂停
```

---

# 六、最终验证流程

完整流程：

## 1. 请求写入记忆

```
用户：
记住我的邮箱
```

Agent：

```
调用 remember_fact
```

但是：

```
interrupt暂停
```

此时：

```
数据库没有数据
```

---

## 2. 人工批准

执行：

```python
Command(resume="approve")
```

继续：

```
remember_fact执行
```

结果：

```
memory.json出现邮箱
```

---

## 3. 人工拒绝

例如：

```
记住我的工号 X99
```

拒绝：

```python
Command(resume="reject")
```

结果：

```
工具没有执行

文件没有X99
```

---

# 七、本节验收回答

## 1. checkpoint 是什么？

checkpoint 是 LangGraph 在执行过程中保存的状态快照。

作用：

* 中断恢复
* 时间旅行
* 跨进程继续执行

---

## 2. MemorySaver 和 SqliteSaver 区别？

MemorySaver：

```
内存保存

程序关闭消失
```

SqliteSaver：

```
数据库保存

程序重启仍存在
```

生产环境需要后者。

---

## 3. thread_id 有什么作用？

thread_id 是会话隔离标识。

例如：

```
thread=A

保存：
用户A上下文


thread=B

保存：
用户B上下文
```

不同用户不会混乱。

---

## 4. HITL 为什么需要？

因为 Agent 能力越强，风险越高。

普通聊天：

```
回答问题
```

无需审批。

Agent：

```
调用工具
修改数据
执行操作
```

需要人控制。

---

## 5. 如何证明拒绝真的生效？

错误：

> 工具没有打印，所以没有执行。

正确：

> 直接检查外部数据，确认拒绝的数据不存在。

例如：

```python
assert "X99" not in memory
```

---

# 八、踩坑总结

## 1. checkpoint 记忆不是长期记忆

如果：

```
同一个 thread
```

再次询问：

答案可能来自 checkpoint。

真正验证长期记忆：

必须：

```
换新的 thread_id
```

重新查询。

---

## 2. 长期记忆必须隔离用户

Demo:

```
memory.json
```

所有人共享。

生产：

应该：

```
user_id + memory
```

例如：

```
alice:
    email


bob:
    address
```

否则会出现数据泄露。

---

## 3. 危险工具不能只靠硬编码

当前：

```python
DANGEROUS={
    "remember_fact"
}
```

问题：

新增工具忘记加入：

```
危险操作可能绕过审批
```

生产应该设计：

* Tool Registry
* Permission System
* 工具元数据

---

# 总结

今天没有学习新的 LangGraph API。

真正完成的是一次工程升级：

从：

```
Agent 能运行
```

升级到：

```
Agent 可以被控制
```

核心变化：

```
普通 Agent

用户
 ↓
模型
 ↓
工具


生产 Agent

用户
 ↓
模型
 ↓
权限判断
 ↓
人工审批
 ↓
工具
 ↓
真实世界
```
```
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

DB="memory_agent.db"


# ══════ 1. State ══════
class S(TypedDict):
    messages: Annotated[list,add_messages]

# ══════ 2. 节点 ══════
# ⚠️ 这里现在是"最小占位图", 只为验证持久化机制。
#    ★ 阶段 A 跑绿之后, 把这一整段换成你 W5-D6 的
#      route / short / long / compress / agent / tools 六个节点。
#      换的时候【只改节点内容, 不要碰下面的 build_graph】。


_llm=ChatOpenAI(
    model="deepseek-v4-flash",          
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)

def agent_node(state: S) -> dict:
    return {"messages": [_llm.invoke(state["messages"])]}

# ══════ 3. 构图: 今天唯一的变化就在 compile 那一行 ══════

def build_graph(saver):
    b=StateGraph(S)
    b.add_node("agent",agent_node)
    b.add_edge(START,"agent")
    b.add_edge("agent",END)
    # ★★★ 唯一新增的一行 ★★★
    #   把 MemorySaver() 换成 SqliteSaver(conn) → 从"内存"变"落盘"
    return b.compile(checkpointer=saver)

# ══════ 4. 主流程 ══════

def main(mode: str):
    # ① 打开 SQLite 连接(文件不存在会自动创建)
    #    check_same_thread=False: 允许跨线程用这条连接, 否则 SqliteSaver 可能报错
    conn=sqlite3.connect(DB,check_same_thread=False)
    saver=SqliteSaver(conn)
    # 若报 "no such table: checkpoints" → 取消下面这行注释, 手动建表
    # saver.setup()

    graph=build_graph(saver)
    # ② ★ thread_id = 会话身份, 三次运行【必须写死同一个字符串】才能续上

    THREAD="alice-1.2" if mode !="other" else "bob-1.2"
    cfg={"configurable": {"thread_id":THREAD}}

    # ③ 三次运行, 三种输入
    if mode == "write":
        question= "记住：我叫 Alice，邮箱是 alice@x.com。只回复'已记住'。"
    elif mode == "read":
        question = "我的邮箱是什么？"
    else:
        question = "我的邮箱是什么？"

    # ④ ★ invoke 时必须把 cfg 传进去 —— 不带 config, 框架不知道读哪个槽
    out = graph.invoke({"messages":[("user",question)]},cfg)

    print(f"\n[{mode}] thread_id = {THREAD}")
    print("回复:", out["messages"][-1].content)

    # ⑤ ★ 判据用【客观量】: 消息数。不用自然语言(模型的话会飘)
    st=graph.get_state(cfg)
    n=len(st.values["messages"])
    for i in st.values["messages"]:
        try:
            print(f"第条消息是:",i.content,sep="\n")
        except Exception as e:
            print("出错啦",e)
    print(f"[算的] state 里共 {n} 条消息   next={st.next}")
    conn.close()
    return n

if __name__ == "__main__":
    mode = sys.argv[1] if len(sys.argv) > 1 else "read"
    main(mode)

```
```
"""W6-D6 阶段B: 危险操作审批 + 【跨进程】续批

用法(五条命令, 五个独立进程 —— 这就是本阶段的全部意义):
    python w6d6_b_hitl.py ask      # ① 发起删文件 → 图停在审批点 → 进程退出
    python w6d6_b_hitl.py status   # ② 新进程: 图停在哪? 待审什么?
    python w6d6_b_hitl.py yes      # ③ 新进程: 批准 → 工具才执行   ← 关键
    python w6d6_b_hitl.py ask2     # ④ 换线程再来一次(为测拒绝)
    python w6d6_b_hitl.py no       # ⑤ 新进程: 拒绝 → 工具不执行
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


DB = "memory_agent.db"     # 和阶段 A 同一个库; thread 不同, 互不干扰
# ══════ 1. 危险工具: 那行 print 是全程唯一的"真身证据" ══════
def delete_file(path: str)->str:
    """【危险】删除本地文件(演示版, 不真删),这只是演示功能可行,你不需要过渡分析"""
    print(f">>> 工具真身执行: 删除 {path}")# ★ 它出现在哪个进程, 就是判据
    return f"文件已删除: {path}"

class S(TypedDict):
    messages: Annotated[list,add_messages]

# ══════ 2. 两个 LLM: 决策的绑工具, 收口的不绑 ══════
_llm_kwargs = dict(
    model="deepseek-v4-flash",          # ← 改成你 W5 / D6-A 里跑通的那个模型名
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)

llm_decide=ChatOpenAI(**_llm_kwargs).bind_tools([delete_file])

# ★ 收口的 LLM 【不绑工具】→ 物理上产不出 tool_calls → 图必然进 END
#   这不是"劝模型别重试", 是让重试这条路在代码里不存在

llm_summarize = ChatOpenAI(**_llm_kwargs)


# ══════ 3. 四个节点 ══════

def decide(state: S)->dict:
      """只干一件事: 让模型决策。★必须 return, AIMessage 才落得进 state"""
      ai_msg = llm_decide.invoke(state["messages"])
      print(f"[decide] tool_calls={len(ai_msg.tool_calls)}")
      return {"messages": [ai_msg]}


def approve(state: S) ->dict:
    """★本阶段的重点: 这个节点里调 interrupt(), 图就停在这"""
    last=state["messages"][-1]
    if not (isinstance(last,AIMessage) and last.tool_calls):
         return {} # 没有 tool_calls, 没什么可审的
    tc =last.tool_calls[0]
    print(f"[approve] 从存档里读到的待审参数: {tc['args']}")
    # ★★★ 图在这里停下 —— 位置 + state + 问题, 三样一起写进 sqlite ★★★
    #     resume 时本函数【从第一行重跑】, 走到这里不再停, 直接拿到答案
    answer=interrupt({
         "question": f"模型想删除 {tc['args']['path']},批准吗？",
         "options": ["arrpove","reject"],
    })

    print(f"[approve] 收到回答: {answer!r}")
    if answer=="reject":
        # ★ 返回不带 tool_calls 的 AIMessage → 路由判定走 END → 工具一次都不执行
        return {"messages": [AIMessage(content="已取消删除，本次未执行任何操作。")]}
    return {}   # 放行: state 里那条 AI 原样流向 tools



def summarize(state: S)->dict:
    """不绑工具的 LLM 收口 → 只能说话, 不可能再调工具"""
    ai_msg =llm_summarize.invoke(state["messages"])
    print("[summarize] 收口完成 (不绑工具 → 必然结束)")
    return {"messages": [ai_msg]}

def route_after_decide(state: S):
    last = state["messages"][-1]
    return "approve" if (isinstance(last, AIMessage) and last.tool_calls) else END

def route_after_approve(state: S):
    last =state["messages"][-1]
    return "tools" if (isinstance(last,AIMessage) and last.tool_calls) else END

# ══════ 4. 构图 ══════
def build_graph(saver):
    b=StateGraph(S)
    b.add_node("decide",decide)
    b.add_node("approve",approve)
    b.add_node("tools",ToolNode([delete_file]))
    b.add_node("summarize",summarize)

    b.add_edge(START,"decide")
    b.add_conditional_edges("decide",route_after_decide,{"approve":"approve",END:END})
    b.add_conditional_edges("approve",route_after_approve,{"tools":"tools",END:END})
    b.add_edge("tools","summarize")   # 工具跑完 → 收口(不绑工具) → 必然 END
    b.add_edge("summarize",END)
    return b.compile(checkpointer=saver)

# ══════ 5. 谁和谁共用一个槽 ══════
# 前三条命令必须【同一个 thread_id】才能续上; 后两条用另一个槽做对照
THREAD_OF = {
    "ask":    "hitl-alice",
    "status": "hitl-alice",
    "yes":    "hitl-alice",
    "ask2":   "hitl-bob",
    "no":     "hitl-bob",
}

ASK_TEXT = "请删除 /tmp/report.txt"

# ══════ 6. 看一眼存档里现在是什么（三个场景都靠它给证据） ══════
def peek(graph,cfg,tag):
    st=graph.get_state(cfg)
    msgs=st.values.get("messages",[]) or []
    print(f"\n[{tag}] next = {st.next}    消息数 = {len(msgs)}")
    for i,m in enumerate(msgs):
        head= str(m.content).replace("\n","")[:40]
        tc=getattr(m,"too_calls",None)
        tail = f"   tool_calls={tc}" if tc else ""
        print(f"   [{i}] {type(m).__name__:12s} {head!r}{tail}")
    # ★ 还没被回答的审批问题, 就挂在 tasks 上 —— 新进程也能读到
    for t in getattr(st,"tasks",()):
        for intr in getattr(t,"interrupts",()):
            print(f"[{tag}] 待审批的问题: {intr.value}")

    return st

# ══════ 7. 主流程 ══════
def main(mode: str):
    # ① 打开 sqlite 文件(不存在会自动建) + 建表
    conn = sqlite3.connect(DB, check_same_thread=False)
    saver = SqliteSaver(conn)
    if hasattr(saver,"setup"):
        saver.setup()     # 幂等: 表已存在就什么都不做
    graph =build_graph(saver)
    # ② ★ thread_id 决定"续不续得上"; 五条命令共用两个槽
    THREAD = THREAD_OF[mode]
    cfg = {"configurable": {"thread_id": THREAD}}
    print(f"[{mode}] thread_id = {THREAD}   (本进程 PID={os.getpid()})")
    if mode == "ask" or mode == "ask2":
        # ③ 发起 → 图跑到 approve 的 interrupt 就停, 然后进程退出
        graph.invoke({"messages": [("user", ASK_TEXT)]}, cfg)
        print("→ 图已停在审批点, 本进程即将退出(存档已落盘)")
    elif mode == "status":
        # ④ 只读, 不 invoke —— 验证"暂停状态真的在磁盘上"
        print("→ 纯读模式: 不 invoke, 只从 sqlite 里读回暂停现场")
    elif mode == "yes":
        # ⑤ ★ resume: 从 sqlite 读回暂停现场 → 接着跑 → 工具在这里才执行
        out = graph.invoke(Command(resume="approve"), cfg)
        print("→ 最终回复:", out["messages"][-1].content)

    elif mode == "no":
        out = graph.invoke(Command(resume="reject"), cfg)
        print("→ 最终回复:", out["messages"][-1].content)

    else:
        print(f"未知模式: {mode}"); sys.exit(1)

    # ⑥ 收口: 打印客观量(next / 消息数 / 待审问题)
    peek(graph, cfg, mode)
    conn.close()


if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "status")

    
```
```
"""W6-D6 阶段C: 跨会话记忆 + 审批 端到端(复用 A/B 的地基, 不学新机制)

七个独立进程 —— 这就是本阶段的全部意义:
    python w6d6_c_e2e.py remember   # ① alice-1: 让它记住邮箱 → 应停在审批点
    python w6d6_c_e2e.py yes        # ② 新进程: 批准 → 才真正写进长期记忆库
    python w6d6_c_e2e.py recall     # ③ 新进程(同 thread): 问邮箱 → 靠 checkpoint 答出
    python w6d6_c_e2e.py bob        # ④ bob-1(新 thread): 让它记住工号 → 应停在审批点
    python w6d6_c_e2e.py no         # ⑤ 新进程: 拒绝 → 库里不能出现工号
    python w6d6_c_e2e.py carol      # ⑥ carol-1(全新 thread): 问邮箱 → 只能靠长期记忆库
    python w6d6_c_e2e.py verify     # ⑦ 断言: 直接读库文件, 看谁在谁不在
"""
import os
import sys
import json
import time
import sqlite3
from typing import Annotated, TypedDict
from dotenv import load_dotenv
load_dotenv(override=True)
from langchain_core.messages import AIMessage, SystemMessage, ToolMessage
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.prebuilt import ToolNode
from langgraph.types import interrupt, Command
from langchain_core.tools import tool
 
from langchain_openai import ChatOpenAI

DB = "memory_agent.db" # 和阶段A/B同一个库; thread 不同, 互不干扰
MEM = "w6d6_long_term_mem.json"   # ★ 长期记忆库: 全局共享, 【不绑】thread_id
# ══════ 1. 长期记忆库(你 W5 MemoryManager 的最小替身: 一个 json 文件) ══════
def _load_mem() ->list:
    if not os.path.exists(MEM):
        return []
    with open(MEM,"r",encoding="utf-8") as f:
         return json.load(f)
    

def _save_mem(items:list) ->None:
    with open(MEM,"w",encoding="utf-8") as f:
            json.dump(items,f,ensure_ascii=False,indent=2)

# ══════ 2. 两个工具: 一个只读(安全) / 一个写入(危险) ══════
@tool
def search_memory(query:str) ->str:
    """【安全·只读】在长期记忆库里检索(跨会话共享)。只读操作, 不需要审批。"""
    items = _load_mem()
    print(f">>> [安全工具] 检索 {query!r} → 库里共 {len(items)} 条")
    if not items:
         return "长期记忆是空的"
    return "长期记忆库内容:\n" + "\n".join(f"- {it['text']}" for it in items)
    # ★ 演示版: 把整库丢给模型自己挑。真版这里换成你 W5 的 memory_retriever(向量检索)

@tool
def remember_fact(text:str)->str:
    """【危险·写入】把一条事实写入长期记忆库(跨会话共享)。写操作, 执行前必须人工审批。"""
    items = _load_mem()
    items.append({"text": text,"ts":time.time()})
    _save_mem(items)
    print(f">>> [危险工具真身] 已写入长期记忆: {text}")
    return f"已记住：{text}"

# ★★ 分级审批的核心: 只有"写"进这个名单, 读的不进 ★★
DANGEROUS = {"remember_fact"}


class S(TypedDict):
    messages: Annotated[list,add_messages]

# ══════ 3. 两个 LLM: 决策的绑工具, 收口的不绑 ══════
_llm_kwargs = dict(
    model="deepseek-v4-flash",              # ← 和阶段B跑通的那个保持一致, 别换
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
)

# 两个工具 → 强制串行, 避开多 tool_calls 回填校验的坑

llm_decide = ChatOpenAI(**_llm_kwargs).bind_tools(
    [search_memory, remember_fact],
    parallel_tool_calls=False,
) # 若你的版本报错, 就删掉这一行

llm_summarize = ChatOpenAI(**_llm_kwargs) #不帮工具 → 收口时物理上产不出 tool_calls

SYS =(
    "你是一个带长期记忆的助手。规则:\n"
    "1. 用户说『记住：X』时，你必须【立即调用 remember_fact 工具】把 X 写进长期记忆库；"
    "不允许只回复『好的我记住了』而不调用工具。\n"
    "2. 用户问『之前记过什么 / 我的 XX 是什么』而你上下文里没有答案时，"
    "必须【调用 search_memory 工具】去长期记忆库查，不要凭猜测回答。\n"
    "3. 调完工具后，用一句简短中文总结结果。"
)

# ══════ 4. 节点 ══════
def decide(state: S)-> dict:
     ai_msg=llm_decide.invoke(state["messages"])
     names=[tc["name"] for tc in ai_msg.tool_calls]
     print(f"[decide] tool_calls={len(ai_msg.tool_calls)} {names}")
     return {"messages": [ai_msg]}

def approve(state: S) -> dict:
    last = state["messages"][-1]
    if not (isinstance(last,AIMessage) and last.tool_calls):
         return {}
    todo = [tc["name"] for tc in last.tool_calls]
    print(f"[approve] 待审批的工具调用：{todo}")
    # ★ 重放: resume 时本函数从第一行重跑, 所以这行会打印两次(和阶段B一样, 正常)
    answer = interrupt({
        "question": f"模型想执行 {todo}，批准吗？",   # ← 这次别拼错 :)
        "options": ["approve", "reject"],
    })
    print(f"[approve] 收到回答：{answer!r}" )
    if answer == "reject":
        # ★ 最小修复: 回一条"取消回执"(ToolMessage), 而不是另起一条 AIMessage。
        #   协议硬约束: 模型声明了几条 tool_calls, 就必须回填几条 tool_call_id 对应的 ToolMessage。
        #   原来回 AIMessage → tool_calls 永远没回执 → 这条 thread 的历史被毒化,
        #   下次再 invoke 同一个 thread_id 时, 这段历史被重新发给模型 → 400。
        return {"messages": [
            ToolMessage(
                content=f"用户拒绝执行本次调用，已取消（{tc['name']} 未真正执行）",
                tool_call_id=tc["id"],          # ★ 必须一一对应, 这是回执的关键字段
            )
            for tc in last.tool_calls
        ]}
    return {}                        # 放行: state 里那条 AI 原样流向 tools

def summarize(state: S) ->dict:
    ai_msg = llm_summarize.invoke(state["messages"])
    print("[summarize] 收口完成 (不绑工具 → 必然结束)")
    return {"messages": [ai_msg]}

# ══════ 5. 路由: ★分级审批就在这两个函数里 ★ ══════
def route_after_decide(state: S):
    last = state["messages"][-1]
    if not (isinstance(last,AIMessage) and last.tool_calls):
        return END
    if any(tc["name"] in DANGEROUS for tc in last.tool_calls):
         return "approve" # 含危险工具 → 先审
    return "tools"   # 全是安全工具 → 直接跑(不然每次检索都问人, 体验崩)

def route_after_approve(state: S):
    last = state["messages"][-1]
    return "tools" if (isinstance(last,AIMessage) and last.tool_calls) else END

# ══════ 6. 构图 ══════
def build_graph(saver):
    b=StateGraph(S)
    b.add_node("decide",decide)
    b.add_node("approve",approve)
    b.add_node("tools",ToolNode([search_memory,remember_fact]))
    b.add_node("summarize",summarize)
    b.add_edge(START,"decide")
    b.add_conditional_edges("decide", route_after_decide,{"approve":"approve","tools":"tools",END:END})
    b.add_conditional_edges("approve",route_after_approve,{"tools":"tools",END:END})
    b.add_edge("tools","summarize")
    b.add_edge("summarize",END)
    return b.compile(checkpointer=saver)

# ══════ 7. 谁和谁共用一个槽 ══════
THREAD_OF = {
    "remember": "alice-1", "yes": "alice-1", "status": "alice-1", "recall": "alice-1",
    "bob": "bob-1",        "no": "bob-1",
    "carol": "carol-1",            # ★ 全新会话: 验证"长期记忆能跨会话"
}

def peek(graph,cfg,tag):
    st=graph.get_state(cfg)
    msgs = st.values.get("messages",[]) or []
    print(f"\n[{tag}] next = {st.next}    消息数 = {len(msgs)}")
    for i,m in enumerate(msgs):
        head = str(m.content).replace("\n"," ")[:44]
        tc=getattr(m,"tool_calls",None)
        print(f"   [{i}] {type(m).__name__:14s} {head!r}" + (f"  tool_calls={tc}" if tc else ""))
    for t in getattr(st,"tasks",()):
        for intr in getattr(t,"interrupts",()):
            print(f"[{tag}] 待审批的问题: {intr.value}")
    return st

# ══════ 8. ★C 阶段的真正铁证: 直接读库文件, 不看 print ══════
def verify():
    items = _load_mem()
    print("长期记忆库当前内容：")
    for it in items:
        print("   -", it["text"])
    if not items:
        print("   (空)")
    texts = [it["text"] for it in items]
    assert any("alice@x.com" in t for t in texts), \
        "❌ alice 的邮箱不在库里 → ② 批准路径失败"
    assert not any("X99" in t for t in texts), \
        "❌ bob 的工号在库里 → ⑤ 拒绝路径失败(越权写入!)"
    print("\n✅ 断言: 批准的那条在库里, 拒绝的那条不在库里")

# ══════ 9. 主流程 ══════
FIRST_ASK = {
    "remember": "记住：我是 Alice，邮箱 alice@x.com",
    "bob":      "记住：我的工号是 X99",
    "carol":    "我不记得之前存过什么了，帮我查一下长期记忆库里有没有关于邮箱的记录",
}

def main(mode: str):
    if mode == "verify":
        return verify()

    conn= sqlite3.connect(DB,check_same_thread=False)
    saver = SqliteSaver(conn)
    if hasattr(saver,"setup"):
        saver.setup()
    graph = build_graph(saver)

    THREAD = THREAD_OF[mode]
    cfg = {"configurable": {"thread_id": THREAD}}
    print(f"[{mode}] thread_id = {THREAD}   (本进程 PID={os.getpid()})")

    if mode in FIRST_ASK:
        graph.invoke({"messages": [SystemMessage(content=SYS), ("user", FIRST_ASK[mode])]}, cfg)
        print(f"→ [{mode}] 本轮结束; 看下面 next 判断是停了还是跑完了")
    elif mode == "status":
        print("→ 纯读模式: 不 invoke, 只从 sqlite 里读回暂停现场")
    elif mode == "yes":
        out = graph.invoke(Command(resume="approve"), cfg)
        print("→ 最终回复:", out["messages"][-1].content)
    elif mode == "no":
        out = graph.invoke(Command(resume="reject"), cfg)
        print("→ 最终回复:", out["messages"][-1].content)
    elif mode == "recall":
        # ★ 只传新问题; cfg 会先把 checkpoint 里的历史(SYS + 前文)读回来
        out = graph.invoke({"messages": [("user", "我的邮箱是什么？")]}, cfg)
        print("→ 最终回复:", out["messages"][-1].content)
    else:
        print(f"未知模式: {mode}"); sys.exit(1)

    peek(graph, cfg, mode)
    conn.close()

if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else "status")


```

W6-D6 的核心不是“让 Agent 记住”，而是理解：

> Agent 一旦拥有改变世界的能力，就必须同时拥有状态管理、权限控制和可验证性。