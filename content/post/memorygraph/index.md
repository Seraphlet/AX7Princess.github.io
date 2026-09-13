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

W6-D6 的核心不是“让 Agent 记住”，而是理解：

> Agent 一旦拥有改变世界的能力，就必须同时拥有状态管理、权限控制和可验证性。
