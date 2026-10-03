---
description: ""
title: " LangGraph 总览"
draft: false
date: "2026-09-12T05:47:58+08:00"
slug: "LangGraphSummary"
categories:
 - LangGraph
tags:
 - Note
image: ""
---

# LangGraph 总览：从会用 API 到理解 Agent Runtime

> **这篇不是 LangGraph API 速查表。**
>
> 目标不是记住：
>
> ```python
> StateGraph(...)
> Send(...)
> interrupt(...)
> ```
>
> 而是建立这样一种能力：
>
> > **看到一个 Agent 需求，能够解释为什么这样建图、状态怎么流、控制流怎么走、为什么需要某个机制，以及不用它会发生什么。**
>
> LangGraph 真正值得学习的，不是 API 数量，而是 **Agent Runtime 的运行模型和系统设计方法**。

---

# 一、先回答：LangGraph 到底解决什么问题？

最容易形成的错误理解是：

> LangGraph = 一个比 LangChain 更复杂的框架。

更准确地说：

```text
LangChain
解决：
“模型、Prompt、Tool、Retriever 怎么组合？”

LangGraph
解决：
“一个有状态、会分支、会循环、会暂停、
会恢复、会并行的 Agent 系统怎么运行？”
```

所以：

```text
Chain
→ 描述一条处理流水线

Graph
→ 描述一个运行中的状态系统
```

这也是为什么 LangGraph 会出现：

```text
State
Node
Edge
Reducer
Checkpoint
HITL
Subgraph
Send
Supervisor
Harness
```

这些东西看起来分散，实际上都在回答同一个问题：

> **Agent 在运行过程中，状态和控制权到底如何流动？**

---

# 二、先建立 LangGraph Runtime 心智模型

以后学习任何 LangGraph API，都先把下面这张图放在脑子里：

```text
                    ┌───────────────┐
                    │     State     │
                    └───────┬───────┘
                            │
                            ↓
                       ┌─────────┐
                       │   Node  │
                       └────┬────┘
                            │
                   State Update
                            │
                            ↓
                       ┌─────────┐
                       │ Reducer │
                       └────┬────┘
                            │
                            ↓
                       Updated State
                            │
                            ↓
                       ┌─────────┐
                       │   Edge  │
                       └────┬────┘
                            │
                  ┌─────────┴─────────┐
                  ↓                   ↓
               Node A             Node B
```

这里有一个极其重要的区分：

```text
Node
→ 我做什么？

Edge
→ 下一步去哪？

State
→ 系统现在知道什么？

Reducer
→ 新状态怎么与旧状态合并？
```

这四个概念如果没有分清，后面学 `Send`、`Subgraph`、`HITL`、`Supervisor` 都会变成背 API。

---

# 三、State：不是“参数”，而是 Agent 的运行时状态

例如：

```python
class State:
    messages
    summary
    retrieved
    tool_result
    user_id
    approval
    next_step
```

不要把它理解成：

> “为了让函数传参数方便。”

更应该理解：

> **State = 整个 Agent 在某一时刻掌握的运行事实。**

因此：

```text
Node
↓
读取 State

Node
↓
产生 State Update

Runtime
↓
合并 State

下一个 Node
↓
继续读取
```

于是第一个深度问题就出现了：

### 为什么不能随便往 State 里塞字段？

因为每一个字段都会成为：

* 节点之间的共享契约
* Reducer 的输入
* Checkpoint 的内容
* 子图的接口
* 调试时的事实来源

所以 **State Schema 本质上是在设计 Agent 的共享内存模型。**

---

# 四、Node：不是“一个函数”，而是职责边界

最简单的 Node：

```python
def node(state):
    return {"result": "..."}
```

但真正重要的问题是：

> **为什么这个逻辑应该成为 Node？**

好的 Node 通常拥有清晰职责：

```text
route_node
→ 决定下一步

compress_node
→ 压缩上下文

memory_node
→ 读写记忆

agent_node
→ 让模型做决策

tool_node
→ 执行工具
```

所以不要问：

> “这里怎么写一个 Node？”

应该先问：

> **这里有没有一个独立的职责边界？**

---

# 五、Edge：控制流到底怎么走

普通 Edge：

```text
A → B
```

Conditional Edge：

```text
       ┌→ B
A ─────┤
       └→ C
```

这里真正值得理解的是：

> **谁决定下一步？**

可能是：

```text
Python 规则
↓
Router
```

也可能：

```text
LLM
↓
结构化输出
↓
Router
```

所以 Conditional Edge 本质不是一个 API。

它表达的是：

> **系统的决策点。**

---

# 六、State Flow 和 Control Flow 必须分开理解

这是理解 LangGraph 最重要的分界之一。

```text
State Flow
──────────
“数据怎么走？”

State
↓
Node
↓
State Update
↓
Reducer
↓
New State
```

而：

```text
Control Flow
────────────
“程序接下来做什么？”

Node
↓
Edge
↓
Next Node
```

于是很多概念可以重新定位：

```text
State / Reducer
→ 数据流

Edge / Conditional Edge / Send
→ 控制流

Checkpoint
→ 时间维度

HITL
→ 控制权

Subgraph
→ 执行边界

Harness
→ 系统边界
```

**这是以后理解复杂 LangGraph 最重要的一张地图。**

---

# 七、Reducer：为什么 State Update 不是简单赋值

例如：

```python
messages: Annotated[list, add_messages]
```

它表达的不是：

> `messages` 是一个 list。

而是：

> **messages 的更新方式由 reducer 决定。**

因此：

```text
旧 State
+
Node Update
↓
Reducer
↓
新 State
```

所以：

```python
return {"messages": new_messages}
```

并不一定意味着：

```text
messages = new_messages
```

可能是：

```text
messages = merge(old_messages, new_messages)
```

这也是为什么你之前的压缩场景容易出问题。

所以学习 Reducer 时最应该问：

> **这个字段的更新语义是什么？**

而不是只记：

```python
Annotated[..., ...]
```

---

# 八、ToolNode + ReAct：Agent 为什么其实是一张循环图

一个典型 Agent：

```text
START
 ↓
Agent
 ↓
需要 Tool？
 ├── No → END
 └── Yes
       ↓
     Tool
       ↓
     Agent
```

这时应该开始建立一个重要认识：

> **Agent 不是一个 LLM。**

Agent 是：

```text
LLM
+
State
+
Tool
+
Routing
+
Loop
```

所以“Agent 能力”实际上是一套运行机制。

---

# 九、Stream：不是打字机，而是观察 Runtime

你已经学过：

```text
updates
values
messages
events
```

不要只记：

> `messages` 可以实现打字机。

更应该理解：

> **Stream 是观察 Graph Execution 的接口。**

所以：

```text
updates
→ 哪个节点更新了什么？

values
→ 当前完整 State 是什么？

messages
→ LLM 消息怎么产生？

events
→ 更底层的运行过程是什么？
```

所以 Stream 和 W7 的 Trace 其实是一条连续的能力：

```text
Stream
↓
观察运行
↓
Trace
↓
工程化记录运行
↓
Eval
↓
衡量运行质量
```

---

# 十、Checkpoint：Agent 为什么开始“记得过去”

没有 Checkpointer：

```text
Run
↓
State
↓
END
↓
结束
```

有 Checkpointer：

```text
Run
↓
State
↓
Checkpoint
↓
继续
↓
Checkpoint
↓
继续
```

所以真正重要的不是：

> “SqliteSaver 是一个保存状态的东西。”

而是：

> **Checkpoint 让 Agent 的执行过程变成了可恢复的状态历史。**

于是你才能拥有：

```text
恢复
多会话
调试历史
HITL 暂停
时间旅行
重新执行
```

所以：

> **Checkpoint 是 Agent 从“一次性程序”走向“持续运行系统”的关键。**

---

# 十一、thread_id：Graph 和 Session 是两回事

同一张 Graph：

```text
             Graph
          /    |    \
         /     |     \
   thread A thread B thread C
```

Graph 是：

> 流程定义。

thread_id 是：

> 某个具体会话/运行实例的身份。

因此：

```text
Graph ≠ Session
```

这是工程化 Agent 必须掌握的概念。

---

# 十二、HITL：真正解决的是什么？

`interrupt()` 不应该记成：

> “一个暂停 API。”

真正的问题是：

> **Agent 的自主性和系统的安全性怎么平衡？**

例如：

```text
Agent
 ↓
“我要删文件”
 ↓
风险判断
 ↓
interrupt
 ↓
Human
 ├── approve
 ├── reject
 └── modify
 ↓
resume
```

因此 HITL 实际上解决：

```text
Agent autonomy
        ↕
Human control
```

随着 Agent 权限增加，人类控制的重要性也会上升。

---

# 十三、interrupt() 和 interrupt_before：区别不在名字

应该问：

> **暂停点发生在哪里？**

```text
interrupt()
→ 节点执行过程中主动暂停

interrupt_before
→ 节点边界之前暂停
```

所以以后碰到 HITL，不要只问：

> “哪个 API 好用？”

而应该先问：

> **我的审批点是一个动态执行逻辑，还是一个固定节点边界？**

---

# 十四、Time Travel：为什么历史状态不是日志，而是可执行资产

假设：

```text
A → B → C → D
```

拥有历史 Checkpoint 后，可以：

```text
A → B → C
        ├→ D
        └→ E
```

所以历史状态不再只是：

> “发生过什么。”

它变成：

> **“可以从这里重新运行什么。”**

这使得：

* 调试
* 恢复
* 实验
* 审批
* 分支执行

成为可能。

---

# 十五、Subgraph：不是“把图塞进去”

简单说：

> Subgraph = 把一套独立的 Graph 执行逻辑作为另一个 Graph 的组成部分。

例如：

```text
Parent Graph
│
├── Planning
│
├── Memory Subgraph
│     ├── Search
│     ├── Rank
│     └── Summarize
│
└── Agent
```

更深的理解是：

> **Subgraph 是模块化的执行边界。**

因此必须继续追问：

```text
父图 State 怎么进入子图？
子图结果怎么回父图？
config 怎么透传？
thread_id 怎么处理？
checkpoint 属于谁？
失败边界在哪里？
```

这才是 Subgraph 的深度。

---

# 十六、Send：真正解决的是动态 fan-out

你现在已经学过 Send，接下来最重要的是忘掉：

```python
Send("worker", payload)
```

这个语法。

先想任务：

```text
topics = [
    Python,
    Go,
    Rust,
    Java
]
```

需要：

```text
             ┌── Worker(Python)
             ├── Worker(Go)
Input ───────┼── Worker(Rust)
             └── Worker(Java)
```

这是：

> **动态 fan-out。**

也就是说：

> 运行时，根据当前数据决定到底创建多少个执行分支。

---

# 十七、为什么不能普通 Conditional Edge？

普通 Conditional Edge 更接近：

```text
A
├── B
└── C
```

它表达：

> 从几个已经定义好的路径里选一个。

而 Send：

```text
A
↓
运行时得到 N 个 item
↓
创建 N 个 worker execution
```

表达的是：

> **动态生成执行实例。**

因此：

```text
Conditional Edge
→ 选择路径

Send
→ 动态产生多个执行任务
```

不要把 Send 理解成：

> “更灵活的 Conditional Edge。”

它们解决的是不同的问题。

---

# 十八、为什么 worker 拿不到 `state["messages"]`？

这里最关键的是理解：

> **Graph State 和 Send Payload 不是一个概念。**

例如：

```python
[
    Send("worker", {"topic": "Python"}),
    Send("worker", {"topic": "Go"}),
]
```

你实际上是在说：

```text
Graph Runtime
↓
创建 worker execution
↓
这个 execution 收到指定 payload
```

所以：

```text
原始 Graph State
≠
某个 Send task 的输入
```

因此 worker 为什么拿不到：

```python
state["messages"]
```

不能简单回答：

> “因为 Send 不传。”

真正应该解释：

> **Worker 的输入边界由当前图的 State Schema、Send payload 以及状态合并机制共同决定；Send 是在创建动态执行任务，而不是简单把整个原始 State 做一次函数参数复制。**

这个答案才开始接近 Runtime 层。

---

# 十九、Send 之后，状态怎么回来？

这是 Send 最重要的问题。

假设：

```text
Worker A
→ {"results": ["A"]}

Worker B
→ {"results": ["B"]}

Worker C
→ {"results": ["C"]}
```

问题：

> **最后 State 到底是什么？**

不是 Send 决定。

而是：

```text
Worker Updates
↓
State Schema
↓
Reducer / Channel Semantics
↓
Merged State
```

所以：

```text
Send
负责 fan-out

Reducer
负责 fan-in 的数据合并
```

这两个概念必须放在一起理解。

---

# 二十、Join：不是一定存在一个 `join()` API

所谓 Join，本质是：

> **多个并行执行完成后，进入一个聚合点。**

例如：

```text
             ┌── Worker A ──┐
             ├── Worker B ──┤
Fan-out ─────┼── Worker C ──┼──→ Aggregate
             └── Worker D ──┘
```

所以 Join 实际上是：

> **一个同步/聚合点。**

因此看到 Send，你必须立刻问：

> **“这些分支最终在哪里汇合？”**

---

# 二十一、并行为什么不是“免费加速”

假设：

```text
100 个 worker
```

并发可能降低 wall-clock time。

但同时增加：

```text
模型调用数
Token 成本
并发资源
限流风险
错误数量
状态合并复杂度
```

所以：

> **能并行 ≠ 应该并行。**

真正的工程判断是：

```text
任务能否拆分？
任务之间是否独立？
并行收益是否大于协调成本？
失败是否可接受？
结果是否需要保持顺序？
```

---

# 二十二、并行为什么会出现顺序问题？

因为：

```text
A
B
C
D
```

并发执行后可能：

```text
C 完成
A 完成
D 完成
B 完成
```

所以如果业务需要顺序，就必须明确设计：

```text
结果是否带 index？
Reducer 是否保证顺序？
Join 如何排序？
```

这说明：

> **并发改变的不只是速度，也改变执行顺序语义。**

---

# 二十三、Worker 失败怎么办？

Parallel Execution 必须考虑失败策略：

```text
A ✅
B ✅
C ❌
D ✅
```

可能有：

```text
策略 1
→ 全部失败

策略 2
→ partial result

策略 3
→ 只 retry C

策略 4
→ fallback

策略 5
→ 超过失败阈值后整体终止
```

所以以后看到 Send：

> **一定顺手问 Failure Semantics。**

这才开始进入真正的 Runtime Engineering。

---

# 二十四、Map-Reduce：Send 背后的更高层抽象

可以把 Send 理解成：

```text
                Map
                 ↓
       ┌─────────┼─────────┐
       ↓         ↓         ↓
      W1        W2        W3
       ↓         ↓         ↓
       └─────────┼─────────┘
                 ↓
               Reduce
```

所以：

```text
Send
→ Map

Worker
→ Parallel Execution

Reducer / Aggregator
→ Reduce
```

这时候你就不再是在记“LangGraph 的一个 API”。

你在理解：

> **动态任务编排。**

---

# 二十五、Multi-Agent：什么时候应该拆 Agent？

多 Agent 不是：

> “任务复杂，所以多加几个 Agent。”

真正应该问：

```text
职责是否天然不同？
工具是否不同？
上下文是否不同？
是否可以并行？
是否需要独立循环？
是否存在明确的角色边界？
```

如果没有：

> 单 Agent 很可能更简单。

所以：

> **Multi-Agent 的第一能力不是“会搭 Supervisor”，而是“会判断什么时候不需要 Multi-Agent”。**

---

# 二十六、Supervisor：谁负责调度？

典型结构：

```text
                Supervisor
               /    |    \
              ↓     ↓     ↓
        Researcher Writer Critic
              \     |     /
               \    |    /
                Supervisor
                     ↓
                   FINISH
```

Supervisor 的本质：

> **控制器。**

它负责：

* 选 Worker
* 读取结果
* 决定下一步
* 判断是否结束

因此要继续问：

> Worker 怎么反馈？

> Supervisor 怎么知道已经完成？

> 什么情况下 FINISH？

> 如果 Worker 失败呢？

> 如果 Supervisor 无限循环怎么办？

---

# 二十七、Reflection：其实就是一个受控循环

```text
Generate
   ↓
Critic
   ↓
Good enough?
 ├── Yes → END
 └── No  → Revise
              ↓
           Generate
```

所以 Reflection 的真正本质是：

```text
Loop
+
Evaluation
+
Conditional Exit
```

因此一定要有：

```text
MAX_REVISE
```

否则：

```text
Generate
→ Critic
→ Revise
→ Critic
→ Revise
→ ...
```

变成成本黑洞。

---

# 二十八、Harness：Agent 为什么不是一个 LLM

一个真正的 Agent 系统：

```text
             LLM
              ↓
         Agent Core
              ↓
┌─────────────────────────────┐
│           Harness           │
│                             │
│ Tool Registry               │
│ Permission Gate             │
│ Session Store               │
│ Context Compaction          │
│ Trace                       │
└─────────────────────────────┘
```

LLM 是：

> **决策能力。**

Harness 是：

> **运行环境。**

所以：

> Agent 的工程能力，不等于 Prompt 能力。

---

# 二十九、Tool Registry：工具多了以后怎么办？

工具越来越多：

```text
tool1
tool2
tool3
...
tool50
```

如果全部直接给 Agent：

```text
工具发现困难
schema 混乱
权限混乱
冲突增加
```

所以 Tool Registry 管的是：

```text
注册
发现
Schema
冲突
权限声明
```

从：

> “调用工具”

升级成：

> **“管理工具生态”。**

---

# 三十、Permission Gate：为什么权限不应该写进 Tool？

例如：

```text
delete_file()
```

Tool 应该只关心：

> 如何删除。

不应该负责：

> 当前 Agent 有没有权限删除。

所以：

```text
Agent
↓
Tool Registry
↓
Permission Gate
↓
HITL
↓
Tool
```

这叫：

> **职责分离。**

---

# 三十一、Session Store：Session 和 Memory 不是同一个东西

可以简单这样理解：

```text
Memory
→ Agent 记得什么

Session
→ 当前用户/任务的运行上下文
```

所以：

```text
thread_id
SqliteSaver
```

在 W6 是持久化机制，

到了 Harness，则成为：

> **Session Store 的基础。**

---

# 三十二、Context Compaction：为什么 Agent 越聊越长反而越危险？

如果：

```text
messages = 1,000 条
```

全部塞进模型：

```text
Context ↑
Cost ↑
Latency ↑
Noise ↑
```

所以：

```text
Old Context
↓
Summary
+
Recent Messages
```

这和你之前做的 MemoryManager 直接连起来：

```text
W3 Memory
↓
W5 MemoryGraph
↓
W6 Persistence
↓
W7 Context Compaction
```

你的路线不是知识点堆积，而是在不断：

> **把旧能力提升为更高层的系统抽象。**

---

# 三十三、Trace：能运行 ≠ 能解释

如果一个 Agent 失败：

```text
User
 ↓
Supervisor
 ↓
Worker A
 ↓
Tool
 ↓
Worker B
 ↓
Critic
 ↓
Fail
```

你需要知道：

```text
哪个节点慢？
调用了几次 LLM？
哪个 Tool 出错？
为什么选 Worker A？
为什么进入循环？
哪里消耗最多 Token？
```

所以 Trace 是：

> **把 Agent 的运行过程变成可解释、可分析的记录。**

它最终会与：

```text
Eval
Observability
Debugging
Cost Analysis
```

连接起来。

---

# 三十四、把整个 LangGraph 压缩成五层

以后忘记某个 API 时，先找它属于哪一层。

```text
Layer 1：Flow
────────────────────
State
Node
Edge
Conditional
Loop

解决：
“Agent 怎么走？”


Layer 2：Execution
────────────────────
ToolNode
Reducer
Stream
Events

解决：
“Agent 怎么执行？”


Layer 3：Runtime
────────────────────
Checkpoint
thread_id
Persistence
HITL
Recovery

解决：
“Agent 怎么长期运行？”


Layer 4：Orchestration
────────────────────
Subgraph
Send
Map-Reduce
Multi-Agent
Supervisor
Reflection

解决：
“复杂任务怎么拆、并行、协作？”


Layer 5：Engineering
────────────────────
Tool Registry
Permission Gate
Session Store
Compaction
Trace

解决：
“怎么把 Agent 做成工程系统？”
```

---

# 三十五、以后每学一个知识点，固定回答这 10 个问题

这是你以后学习 LangGraph 最重要的模板。

## ① 它是什么？

先定义。

## ② 它解决什么问题？

没有它会怎样？

## ③ 为什么需要它？

为什么普通 Python / Chain 不够？

## ④ 它改变的是 State 还是 Control Flow？

这是核心分类。

## ⑤ Runtime 在这里做了什么？

它不是一个 API 调用就结束了。

## ⑥ 它和什么概念容易混淆？

主动找边界。

## ⑦ 为什么不用更简单的机制？

这是设计判断。

## ⑧ 正常情况怎么走？

画执行流。

## ⑨ 异常情况怎么走？

考虑：

```text
failure
retry
timeout
partial result
recovery
```

## ⑩ 并发 / 顺序 / 成本 / 权限怎么办？

从 Demo 进入工程系统。

---

# 三十六、真正的 LangGraph 学习等级

不要再用：

> “我学完 W6 了。”

判断自己有没有学会。

可以用这五级：

```text
Level 1
API
会写

↓


Level 2
Mechanism
知道为什么能跑

↓


Level 3
Design
知道为什么这样设计

↓


Level 4
System
能组合成完整 Agent

↓


Level 5
Engineering Judgment
知道什么时候该用、什么时候不该用，
失败时怎么办，成本和边界在哪里
```

你的目标应该是：

> **至少把核心知识做到 Level 4，重要知识做到 Level 5。**

---

# 三十七、真正会 LangGraph 的标准

以后有人问：

> “你懂 LangGraph 吗？”

真正能证明你懂，不是：

> “我会 StateGraph、Send、interrupt。”

而是你能解释：

```text
为什么需要 Graph？

State 怎么设计？

Node 如何拆职责？

Edge 怎么决定控制流？

Reducer 怎么定义状态更新语义？

Checkpoint 如何让执行可恢复？

HITL 如何改变控制权？

Subgraph 如何建立执行边界？

Send 为什么能动态 fan-out？

并行结果怎么 fan-in？

失败怎么处理？

什么时候单 Agent 足够？

什么时候需要 Multi-Agent？

Supervisor 为什么需要？

Harness 为什么存在？
```

**能把这些问题连成一条因果链，你才算真正理解 LangGraph。**

---

# 三十八、最终记住这一句话

> **LangGraph 不只是“把 Agent 画成图”。**
>
> 它是在给 Agent 建立一套：
>
> **状态模型 + 控制流模型 + 执行模型 + 持久化模型 + 人机协作模型 + 并行模型 + Agent 编排模型。**
>
> API 只是这些模型暴露出来的操作入口。

所以以后看到：

```python
Send(...)
```

不要先问：

> “这个怎么写？”

先问：

> **它正在解决哪个 Runtime 问题？**
>
> **它改变了什么执行语义？**
>
> **为什么普通 Edge 不够？**
>
> **任务的数据从哪里来、到哪里去？**
>
> **并行后怎么合并？**
>
> **失败怎么办？**
>
> **什么时候根本不应该使用它？**

当你开始习惯这么思考时，你学到的就不再只是 LangGraph。

而是在学习：

> **Agent Runtime / Workflow Orchestration / Agent System Design。**
