---
description: ""
title: "从有状态 Agent 到生产级工作流"
draft: false
date: "2026-09-13T13:47:06+08:00"
slug: "LangGarphsummarize2"
categories:
 - LangGraph
 - Runtime
tags:
 - Memory
image: ""
---

---

# LangGraph W6 周总结：从有状态 Agent 到生产级工作流

这一周主要学习 LangGraph 的**状态管理、人工介入、流程恢复以及任务拆分能力**。

如果说 W5 学习的是：

> 如何让 Agent 按照图结构运行

那么 W6 学习的是：

> 如何让 Agent 像一个真正的产品一样长期运行，并且在关键节点接受控制。

这一周几个核心关键词：

* Checkpoint：保存 Agent 每一步状态
* Persistence：让状态跨进程保存
* HITL（Human In The Loop）：让人参与 Agent 决策
* Time Travel：基于历史状态恢复执行
* Subgraph：拆分复杂 Agent
* Send：动态并行任务分发

---

# 一、W5 无状态 Agent vs W6 有状态 Agent

之前的 Agent：

```
用户输入
 ↓
Agent运行
 ↓
返回结果
```

一次运行结束，状态消失。

但是实际产品需要：

例如：

用户：

> 帮我记录一下，我喜欢喝咖啡。

下一次：

> 我喜欢喝什么？

Agent 需要知道之前的信息。

所以需要：

```
第一次运行

State
 ↓
Checkpoint保存


第二次运行

读取Checkpoint
 ↓
恢复State
 ↓
继续执行
```

LangGraph 通过 checkpoint 机制实现状态保存。

---

# 二、Checkpoint：Agent 的历史快照

Checkpoint 可以理解为：

> 每执行完一个节点，对当前 State 保存一次快照。

例如：

```
START

 ↓

agent节点

State:
{
 messages:[
  "用户问题"
 ]
}


保存 checkpoint


 ↓


tool节点

State:
{
 messages:[
  "用户问题",
  "tool结果"
 ]
}


再次保存 checkpoint

```

所以 checkpoint 不只是保存最终结果，而是保存：

* 当前 state
* 当前执行位置
* thread 信息

这也是后面：

* resume
* update_state
* time travel

的基础。

---

# 三、MemorySaver 和 SqliteSaver

## MemorySaver

之前学习：

```python
memory = MemorySaver()

graph = builder.compile(
    checkpointer=memory
)
```

特点：

* 保存到内存
* 程序关闭后消失

适合：

* 学习
* 测试

---

## SqliteSaver

生产环境需要落盘。

例如：

```python
from langgraph.checkpoint.sqlite import SqliteSaver


checkpointer = SqliteSaver.from_conn_string(
    "agent.db"
)


graph = builder.compile(
    checkpointer=checkpointer
)
```

区别：

|      | MemorySaver | SqliteSaver |
| ---- | ----------- | ----------- |
| 存储位置 | 内存          | 数据库         |
| 重启恢复 | ❌           | ✅           |
| 适合   | 测试          | 生产          |

---

# 四、thread_id：实现多用户隔离

LangGraph 使用 config 区分不同会话。

例如：

```python
config = {
 "configurable":{
     "thread_id":"user_001"
 }
}


graph.invoke(
    input,
    config=config
)
```

不同 thread：

```
user_001

checkpoint A
checkpoint B


user_002

checkpoint C
checkpoint D
```

它们互相不会影响。

所以：

> thread_id 本质是一个会话隔离标识。

---

# 五、HITL：让 Agent 停下来等待人确认

Agent 最大的问题：

能力越强，风险越高。

例如：

普通回答：

```
用户问题
 ↓
LLM回答
```

风险低。

但是：

```
用户请求删除文件

 ↓

Agent调用工具

 ↓

删除执行
```

如果 Agent 判断错误，会造成真实损失。

所以需要：

```
Agent决定

 ↓

暂停

 ↓

人确认

 ↓

继续执行
```

这就是 HITL。

---

# 六、interrupt() 和 interrupt_before 的区别

这是本周重点验收。

## interrupt_before

在 compile 时指定：

```python
graph = builder.compile(
    interrupt_before=["tools"]
)
```

作用：

> 在进入某个节点之前暂停。

流程：

```
agent

 ↓

暂停

 ↓

tools
```

它控制的是：

节点之间。

---

## interrupt()

写在节点内部：

```python
from langgraph.types import interrupt


def approval_node(state):

    answer = interrupt(
        "是否允许执行?"
    )

    return {
        "result":answer
    }
```

流程：

```
进入节点

 ↓

执行到 interrupt()

 ↓

暂停

 ↓

等待恢复
```

它控制的是：

节点内部。

核心区别：

|      | interrupt_before | interrupt() |
| ---- | ---------------- | ----------- |
| 位置   | 节点外部             | 节点内部        |
| 控制粒度 | 节点级              | 代码级         |
| 适合   | 固定审批点            | 动态判断        |

---

# 七、Command(resume)：恢复暂停的 Agent

暂停后：

需要通过：

```python
Command(
    resume=True
)
```

恢复。

例如：

```python
graph.invoke(
    Command(
        resume=True
    ),
    config=config
)
```

含义：

告诉 LangGraph：

> 人已经做出决定，继续执行。

---

# 八、审批三态设计

实际生产不是简单 yes/no。

通常有三种：

## 1. 放行

```
resume=True
```

继续执行。

---

## 2. 拒绝

```
resume=False
```

Agent 根据拒绝原因重新规划。

例如：

删除文件：

```
用户拒绝

↓

换成备份文件
```

---

## 3. 修改参数

人工直接修改 state：

```python
graph.update_state(
    config,
    {
      "path":"new_file.txt"
    }
)
```

然后继续执行。

这比简单批准更加接近真实系统。

---

# 九、Time Travel：回到过去重新执行

因为 LangGraph 保存 checkpoint。

所以可以：

查看历史：

```python
history = graph.get_state_history(
    config
)
```

找到旧 checkpoint：

```
checkpoint_1

checkpoint_2

checkpoint_3
```

然后：

从旧状态重新运行。

应用场景：

例如：

```
Agent:

发送邮件

↓

发现参数错误


回到发送前

↓

修改参数

↓

重新执行
```

这就是 Agent 的调试能力。

---

# 十、Subgraph：拆分复杂 Agent

复杂 Agent 不应该全部写在一个图。

例如：

```
主Agent

├── 搜索Agent
│
├── 分析Agent
│
└── 写作Agent
```

LangGraph 可以：

先编译子图：

```python
subgraph = child_builder.compile()
```

然后：

作为节点加入：

```python
builder.add_node(
    "research",
    subgraph
)
```

最终形成：

```
parent

 ↓

research

    ↓

 research.search

 research.analyze
```

子图解决：

> 大 Agent 拆分和复用问题。

---

# 十一、Send：动态任务并行

普通 edge：

```
A

↓

B

↓

C
```

但是很多任务天然适合并行。

例如：

搜索多个关键词：

```
关键词:

python
langgraph
agent


同时搜索

 ↓


汇总结果
```

使用 Send：

```python
from langgraph.types import Send


def dispatch(state):

    return [
        Send(
            "worker",
            {
              "keyword":k
            }
        )
        for k in state["keywords"]
    ]
```

产生：

```
dispatch

 ├── worker(python)

 ├── worker(langgraph)

 └── worker(agent)


        ↓


      join
```

这就是：

Map-Reduce 模式。

---

# 十二、本周完成能力总结

## 知识层

✅ 理解 checkpoint：

> 每个节点执行后的 state 快照。

✅ 理解 MemorySaver 和 SqliteSaver：

> 内存保存 vs 数据库存储。

✅ 理解 thread_id：

> 不同会话通过 thread_id 隔离。

✅ 理解 HITL：

> Agent 在关键节点暂停，让人参与决策。

✅ 理解：

```
interrupt()

vs

interrupt_before
```

区别：

> 节点内部暂停 vs 节点边界暂停。

✅ 理解 Send：

> 动态产生多个并行任务。

---

# 十三、W6 最大收获

之前：

```
Agent = 调用LLM完成任务
```

现在：

```
生产级Agent=

状态保存

+

人工控制

+

错误恢复

+

任务拆分

+

并行执行
```

真正可用的 Agent，不只是会回答问题。

而是：

> 在复杂环境中，可以暂停、恢复、修改、追踪，并且安全执行任务。

这也是 LangGraph 和普通 Chain 最大的区别。

---

W6 完成后，Agent 开发进入了一个新的阶段：

从「能运行」

进入：

「可控制、可恢复、可维护」。
