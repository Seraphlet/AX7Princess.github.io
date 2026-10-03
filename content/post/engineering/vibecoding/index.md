---
description: ""
title: "# 从 Human 到 Runtime：我对 AI Coding 开发方式的一次重新理解"
draft: false
date: "2026-09-30T09:05:24+08:00"
slug: "VibeCoding"
categories:
 - VibeCoding
tags:
 - 
image: ""
---

# 从 Human 到 Runtime：我对 AI Coding 开发方式的一次重新理解

最近做项目时，我越来越明显地感觉到一个问题：

**AI 能写代码，并不等于 AI 已经接管了开发。**

以前我会把任务交给 AI：

```text
写代码
→ 我检查
→ 跑测试
→ 我确认
→ 继续下一步
→ 再检查
→ 再确认
```

表面上 AI 在开发，实际上真正维持整个开发流程运行的人还是我。

什么时候开始下一项任务？

测试失败以后怎么办？

要不要重新测试？

什么时候全量回归？

AI 做完以后应该交给谁？

这些事情仍然需要我不断做决定。

后来我才意识到：

> **我虽然把 Implementation 交给了 AI，却没有把 Runtime 交出去。**

这让我开始重新思考 AI Coding。

---

# 一、第一阶段：Human 本身就是 Runtime

最开始的开发模式其实非常简单：

```text
                Human
                  │
          ┌───────┴───────┐
          ▼               ▼
       Worker          Test / Review
          │               │
          └───────┬───────┘
                  ▼
                Human
                  │
              下一任务
```

AI 负责执行，但所有状态转换实际上都由人完成。

比如：

```text
Worker 写完
↓
我判断该测试了

测试失败
↓
我判断让 Worker 修改

测试通过
↓
我判断可以进入下一任务
```

所以真正的 Runtime 是：

```text
Human
```

AI 更像 Runtime 调用的一个工具。

这种模式对于小任务没有什么问题。

但项目变大以后，人会逐渐成为瓶颈。

我之前做项目时就遇到了一个明显的问题：

**验收越来越频繁。**

改一点：

```text
测试
```

再改一点：

```text
再测试
```

完成一个小模块：

```text
全量测试
```

然后继续下一步。

最终大量时间和资源消耗在重复验收上。

更重要的是，我始终被绑在开发循环里面。

AI 并没有真正接管项目。

---

# 二、第二阶段：把控制权交给 Orchestrator Agent

既然问题不是“没人写代码”，而是“没人负责运行整个开发流程”，那么一个自然的想法就是：

> 增加一个主控 Agent。

于是开发结构开始变成：

```text
                 Orchestrator
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
       Builder                 Reviewer
        写代码                  审查 / 测试
          │                       │
          └───────────┬───────────┘
                      ▼
                 Orchestrator
```

职责开始真正分离。

### Builder

只有实现权。

```text
读取任务
→ 修改代码
→ 最小必要自检
→ 提交结果
```

它不能宣布：

> “这个任务已经最终通过。”

因为写代码的人不应该同时拥有最终验收权。

---

### Reviewer

拥有验收权。

```text
读取 Acceptance Criteria
→ 查看 diff
→ 选择对应测试范围
→ PASS / FAIL / BLOCKED
```

Reviewer 原则上不负责顺手修改业务代码。

否则：

```text
写
+
审
```

又重新混成了一个角色。

---

### Orchestrator

不负责具体实现。

它负责：

```text
现在做到哪里？
↓
下一步应该交给谁？
↓
失败应该返工还是升级？
↓
什么时候进入下一阶段？
```

这时候 Human 第一次可以从大量日常控制动作里退出。

---

# 三、但 Multi-Agent 又产生了一个新问题：他们怎么知道彼此做了什么？

如果三个 Agent 都是独立会话：

```text
Orchestrator
Builder
Reviewer
```

那么很快就会遇到问题：

> Builder 怎么告诉 Reviewer 自己改了什么？

最简单的方法当然是聊天。

但聊天并不是一个可靠的 Runtime。

于是我开始把开发过程里的“对话”与“状态”分开。

核心变化是：

> **Agent 不应该依靠聊天记录完成交接，而应该依靠共享 State。**

于是出现：

```text
.runtime/
├── state.json
├── current_task.json
├── builder_result.json
├── review_result.json
└── events.jsonl
```

这时候两个以前很抽象的概念突然变得非常具体。

---

# 四、State 解决“现在在哪”，Trace 解决“怎么到这里”

`state.json` 保存的是当前工作现场：

```text
当前 Phase
当前 Task
当前状态
当前应该由谁执行
最后一个通过的任务
是否存在 Block
```

它回答：

> **现在是什么状态？**

而 `events.jsonl` 保存：

```text
Orchestrator 派发了什么
Builder 修改了什么
Reviewer 为什么 FAIL
发生过几次 Retry
什么时候进入下一阶段
```

它回答：

> **为什么变成了现在这个状态？**

所以：

```text
State ≠ Trace
```

可以把它理解成：

```text
state.json
= 当前位置

events.jsonl
= 行车记录仪
```

Agent 不需要知道所有历史聊天。

只要读取 State，就能恢复当前工作。

而出现问题以后，再通过 Trace 回看发生过什么。

---

# 五、Prompt 里的“权限”并不是真正的权限

接下来又出现了另一个问题。

我可以在 Builder 的操作文档里写：

> 你不能修改 SYSTEM.md。

也可以告诉 Reviewer：

> 你只能审查，不能修改业务代码。

但 Agent 并不会因为我写了一句话，就真的失去修改文件的能力。

这让我第一次真正区分：

```text
Role
```

和：

```text
Permission
```

Prompt 定义的是角色。

但它不是权限系统。

所以权限需要逐渐从文字约定向系统约束演进：

```text
Prompt Role
    ↓
actor
    ↓
expected_actor
    ↓
runtime_version
    ↓
writable_scope
    ↓
Git
    ↓
Watchdog
    ↓
必要时 OS ACL / Runtime CLI
```

例如每一次 Result 都带：

```text
actor = builder
task = T-R03
based_on_runtime_version = 18
```

主控接收时检查：

```text
你真的是当前应该执行的人吗？

你依据的是不是最新 State？

你操作的是不是当前 Task？
```

于是：

```text
身份
+
版本
+
作用域
```

共同形成一层轻量的 Runtime Guard。

如果以后实际运行证明文字协议仍然经常被违反，再把这些约束下沉成真正的程序权限。

而不是一开始就造一个复杂权限系统。

---

# 六、Human 也不应该重新成为另一个 Orchestrator

做到这里还有一个很容易发生的问题。

即使有了主控 Agent，人还是可能不断插手：

```text
这个先删掉
这个重新写
先别测试
换一个实现
这个 Task 跳过
```

这样虽然表面上有 Orchestrator，但真正的 Orchestrator 仍然是 Human。

所以我重新定义了 Human 的位置。

在 Runtime 启动之前：

```text
Human
↓
定义 PRODUCT
↓
定义 SYSTEM
↓
定义 DECISIONS
↓
定义 TASKS
↓
定义 BUILD / Runtime Contract
```

一旦开始运行：

```text
START
══════════════════════

Agent Runtime 自治

Builder
Reviewer
Orchestrator
Watchdog

══════════════════════
```

Human 不再管理中间施工。

只控制 Runtime 生命周期。

---

# 七、Human Control 最后被压缩成两个状态

这让我最终把人的运行时控制压缩到了非常简单的程度：

```text
RUN
```

或者：

```text
PAUSE_AFTER_CURRENT
```

不是让 Human 发：

```text
删除 xxx
重写 xxx
修改 xxx
```

Human 只告诉 Runtime：

> 继续运行。

或者：

> 当前工作正常收口以后，不要开始新的工作。

于是出现：

```text
TASKS
= 路线图

STATE
= 当前位置

HUMAN_RUNTIME_STATE
= 方向盘

EVENTS
= 行车记录仪
```

而且 Orchestrator 每一次准备下发新任务之前，都必须重新读取 Human Runtime State。

```text
完成当前工作
↓
更新 State
↓
准备 Dispatch
↓
读取 Human State
↓
RUN?
├── YES → Dispatch
└── NO  → PAUSED
```

这比试图中途打断 Agent 简单很多。

如果真的需要强制停止，直接 Kill。

正常暂停应该发生在稳定的状态转换边界。

---

# 八、测试也应该属于 Runtime，而不是属于 Human

另一个重要变化是：

以前测试是：

```text
AI 做完
↓
Human：
“跑一下测试。”
```

现在测试策略应该在 Runtime 开始前就确定。

例如：

```text
普通代码修改
→ Minimal Check

Task Review
→ Targeted Test

Phase Complete
→ Integration Test

关键共享组件变化
→ Impact Regression

V0 Complete
→ Full Regression

Release Candidate
→ Full Regression + Golden Dataset
```

于是：

> **Task 验收不等于全量验收。**

测试范围由修改影响范围和当前 Gate 决定。

这样既避免每一步全量测试，也避免 Agent 为了“保险”无限重复测试。

---

# 九、重复失败不是继续 Retry 的理由

如果：

```text
FAIL
→ 改
→ 测
→ FAIL
→ 改
→ 测
→ FAIL
```

继续循环通常已经没有意义。

重复失败意味着：

> **当前对问题的理解可能错了。**

所以 Runtime 应该升级问题：

```text
Repeated Failure
        ↓
Root Cause Analysis
        ↓
┌────────┬────────┬──────────┬────────────┐
▼        ▼        ▼          ▼
代码问题  测试问题  环境问题    架构问题
```

前三种仍然由 Runtime 自己处理。

只有最后一种：

```text
STRUCTURAL_BLOCK
```

才应该真正暂停交给 Human。

而且不能只说：

> “架构可能有问题。”

必须带 Evidence：

```text
发生了什么
排除了什么
哪条 SYSTEM / DECISION 发生冲突
为什么局部修改解决不了
可能有哪些结构性选择
```

于是 Human 看到的是一个已经调查过的问题。

而不是 Agent 遇到困难以后把问题重新扔回来。

---

# 十、做到这里，我突然发现：Orchestrator Agent 本身也可能只是一个过渡阶段

这是这次设计里我觉得最有意思的地方。

现在 Orchestrator 会做：

```text
BUILD_DONE
→ REVIEW

REVIEW_PASS
→ NEXT_TASK

REVIEW_FAIL
→ RETRY

REPEATED_FAIL
→ DIAGNOSE

PHASE_COMPLETE
→ INTEGRATION_TEST
```

但仔细看会发现：

**这些决策很多根本不需要智能。**

它们实际上已经接近：

```text
状态机
```

既然如此，为什么还需要一个 LLM Agent 每次重新思考：

> “接下来应该让 Reviewer 工作。”

完全可以变成：

```text
if state == BUILD_DONE:
    next = REVIEWER
```

于是整个系统又开始发生下一次变化。

---

# 十一、第三阶段：Orchestrator 从 Agent 变成 State Machine

未来真正的 Runtime 可能是：

```text
                   Runtime
                      │
                      ▼
               State Machine
                      │
            根据 State 选择 Node
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
    Builder        Reviewer       Diagnose
       │              │              │
       └──────────────┴──────────────┘
                      │
                State Update
                      │
                      ▼
                   Runtime
```

这时候主控不需要一直运行。

它甚至不再是一个持续存在的 Agent。

流程可以变成：

```text
Builder Node
↓
执行
↓
写 State
↓
结束
↓
Runtime 被唤醒
↓
读取 State
↓
决定下一 Node
↓
Reviewer Node
↓
执行
↓
结束
```

每个 Agent 都只是一个短生命周期 Worker。

真正持续存在的是：

```text
State
+
Runtime
```

这时候控制权又向下移动了一层。

---

# 十二、这开始越来越像 LangGraph

做到这里，我突然发现以前学习 LangGraph 时的一些概念开始全部对应起来。

今天文件 Runtime 里的：

```text
expected_actor = builder
```

未来其实就是：

```text
state.next = builder
```

今天的：

```text
builder_result.json
```

未来就是：

```text
Builder Node
→ State Update
```

今天：

```text
Reviewer PASS
→ Orchestrator
→ 下一 Task
```

未来：

```text
Reviewer
↓
state.review_status = PASS
↓
Conditional Edge
↓
Next Task
```

今天：

```text
PAUSE_AFTER_CURRENT
```

未来：

```text
Human State
↓
Conditional Edge
├── PAUSE → Checkpoint / END
└── RUN   → Next Node
```

以前我学习：

```text
State
Node
Runtime
Conditional Edge
Checkpoint
Interrupt
```

这些东西时，很容易把它们理解成 LangGraph 提供的一组 API。

但现在重新看，它们其实都是某种真实工程问题的答案。

---

# 十三、TASKS 未来也可能从文档变成真正的任务队列

现在：

```text
TASKS.md
```

主要还是给 Orchestrator 阅读。

但未来任务本身也可以结构化：

```text
T01 READY

T02 WAITING(T01)

T03 WAITING(T01)

T04 WAITING(T02, T03)
```

于是 Runtime 可以自己判断：

```text
哪些 Task 已经 READY？
↓
Dispatch
↓
完成
↓
更新 Dependency
↓
新的 Task READY
```

如果：

```text
T02
```

和：

```text
T03
```

之间没有依赖：

```text
        T01
       /   \
     T02   T03
       \   /
        T04
```

那么自然就会出现：

```text
fan-out
↓
Builder A → T02
Builder B → T03
↓
join
↓
T04
```

这时候以前学过的：

```text
Send
Reducer
Join
Parallel Worker
```

也开始拥有真实意义。

不是因为：

> “LangGraph 有这些功能，所以我要用。”

而是因为：

> **我的 Runtime 真的出现了动态任务、并行执行和状态合并的问题。**

---

# 十四、最终的进化线

回头看，这套开发方式其实形成了一条非常自然的进化路线。

## V0：Human 是 Runtime

```text
Human
↓
Worker
↓
Human Review
↓
Next Task
```

AI 负责执行。

Human 负责控制。

---

## V1：Orchestrator Agent 是 Runtime

```text
           Orchestrator
          /            \
     Builder          Reviewer
          \            /
             State
```

Human 从正常控制环退出。

Agent 开始通过共享 State 协作。

---

## V2：State Machine 是 Runtime

```text
State
↓
Deterministic Runtime
↓
Builder / Reviewer / Diagnose Nodes
↓
State Update
```

确定性的调度不再消耗 LLM 推理。

Agent 只负责真正需要智能的节点。

---

## V3：Graph Runtime

当系统真正出现：

```text
任务依赖
动态任务
并行
Fan-out
Join
Checkpoint
Interrupt
Retry
Recovery
```

再演化成：

```text
Graph Runtime
```

此时：

```text
Builder
Reviewer
Diagnose
Planner
```

只是 Graph 上不同能力的 Node。

---

# 十五、我真正想要的不是“多 Agent”，而是控制权逐步下沉

这次最大的变化其实不是：

> 我从一个 Agent 变成了四个 Agent。

真正发生变化的是：

```text
Human
  ↓
Orchestrator Agent
  ↓
State Machine
  ↓
Graph Runtime
```

控制权不断从昂贵、不稳定、需要持续注意力的上层，向更确定的下层移动。

可以确定的：

```text
交给程序。
```

存在语义不确定性的：

```text
交给 Agent。
```

涉及架构、产品、责任和风险边界的：

```text
留给 Human。
```

所以最终形成的不是：

> **Everything is Agent。**

反而是：

> **只让 Agent 留在真正需要 Agent 的地方。**

---

# 十六、这也改变了我对 AI Coding 的理解

以前我会觉得：

> AI Coding 的核心是怎么让 AI 写出更好的代码。

现在我更倾向于认为：

> **真正的问题是如何设计一个系统，让 AI 能够在明确边界内持续工作，而 Human 不需要成为它的实时调度器。**

于是人的工作逐渐从：

```text
写代码
盯代码
告诉 AI 下一步
检查每个中间状态
```

变成：

```text
定义问题
↓
定义 PRODUCT
↓
设计 SYSTEM
↓
做关键 DECISIONS
↓
定义 Runtime Contract
↓
让 Runtime 构建
↓
拿完整产品真实使用
↓
观察 Trace / Evidence
↓
发现真正的问题
↓
进入下一轮设计
```

我不再希望每完成一步，AI 都回来问我：

> “这样对吗？”

只要没有出现系统性、结构性错误：

> **先按照已经确定的设计把它做出来。**

等真正的成品运行起来以后，我再从系统行为判断哪里应该修改。

因为很多问题只有系统完整运行以后才真正存在。

---

# 十七、最后：不要一开始就造最终 Runtime

虽然已经能够看到：

```text
State Machine
Graph Runtime
Parallel Worker
Checkpoint
Interrupt
```

但我现在并不准备立刻实现它们。

当前真正需要的是：

```text
多个 Agent Session
+
Role Instructions
+
Shared State
+
Result Files
+
Git
+
Trace
+
Human Runtime State
```

先让它真正跑一个项目。

如果未来发现：

> Orchestrator 绝大多数时候只是在机械执行状态转换。

再把它下沉成 State Machine。

如果未来真的出现：

> 并行、动态任务、Join、恢复。

再引入 Graph Runtime。

这也是我现在越来越认可的一条工程原则：

> **不要因为某个机制先进就加入它，而是等问题出现以后，让机制获得存在的资格。**

所以这条进化路线不是我要提前全部实现的架构图。

它更像一个方向：

```text
Human Runtime
      ↓
Agent Runtime
      ↓
State Machine Runtime
      ↓
Graph Runtime
```

每往下一层，都意味着一部分已经被证明足够确定的控制权，从 Human 或 LLM 手里移交给程序。

而 Human 最终留下的，应该是那些最难下沉的事情：

```text
定义问题
价值判断
架构选择
边界设计
风险承担
最终责任
```

这可能才是我目前对 AI Coding 最重要的一次理解：

> **AI 的价值不只是替我写更多代码，而是让我逐渐退出那些已经可以被系统接管的控制环，把注意力留给仍然需要人做判断的地方。**