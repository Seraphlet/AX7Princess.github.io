---
description: ""
title: "从单 Worker 到 Multi-Agent：我的 AI Coding 工程协作手册"
draft: false
date: "2026-09-27T05:39:46+08:00"
slug: "Agents"
categories:
 - Agetns
tags:
 - Note
image: ""
---

# 从单 Worker 到 Multi-Agent：我的 AI Coding 工程协作手册

> V0.2：在 V0.1 的单 Worker / Multi-Agent 文档体系上，增加 Risk-based Review、人类注意力分配、Human Gate，以及最终系统全景图。

---

## 三十五、不是所有代码都值得 Human Review

最开始使用 Coding Agent 时，我很容易默认：

```text
AI写代码
   ↓
我看代码
   ↓
确认没问题
   ↓
继续
```

但随着项目越来越复杂，我发现这其实会产生新的瓶颈：

> AI 写代码的速度远高于人读代码的速度。

如果 AI 每写一个文件、一个函数，我都重新阅读实现，那么 AI 虽然提高了 Coding Speed，却没有真正提高整个系统的 Throughput。

真正应该问的不是：

> “这段代码是不是 AI 写的？”

而是：

> **“这段代码如果错了，会造成什么后果？我能不能通过低成本证据发现它错了？错了以后能不能恢复？”**

因此 Review 不应该平均分配，而应该根据风险分配。

---

# 三十六、Risk-based Review：按照风险决定 Review 深度

判断一个 Task 是否需要 Human Review，可以看三个变量：

```text
                 Failure Impact
                  失败后果
                     ↑
                     │
                     │
                     │
低验证成本 ←── Verifiability ──→ 高验证难度
                     │
                     │
                     ↓
                Reversibility
                  可逆性
```

三个问题：

```text
① Failure Impact
如果错了，损失多大？

② Verifiability
能不能通过测试/结果可靠发现错误？

③ Reversibility
出错以后能不能轻易恢复？
```

因此可以把任务粗略分为三个等级。

| Risk | 特征 | Review |
|---|---|---|
| LOW | 易验证、易恢复、低副作用 | Worker + Reviewer 自动完成 |
| MEDIUM | 业务核心、可能 Silent Failure | Reviewer 强验证，Human 看 Specification/关键决策 |
| HIGH | 高副作用、不可逆、安全相关 | Human Gate |

这里非常重要的一点是：

> **核心业务代码不一定需要人逐行看代码。**

Human 更应该确认：

```text
Business Rule
Acceptance Criteria
Edge Cases
Architecture Boundary
```

至于具体：

```text
for怎么写
函数怎么拆
用了什么临时变量
Parser内部怎么循环
```

只要可以被可靠验证，就可以交给 Worker + Reviewer。

因此 Human Review 可以逐渐从：

```text
Code Review
```

转向：

```text
Specification Review
+
Evidence Review
+
Risk Review
```

---

# 三十七、LOW：可以完全退出 Human Review 的任务

例如：

```text
普通CSV读取
JSONL写入
Formatter
Parser
字符串处理
Keyword Position
Context Window
Deduplicate
普通Metrics
普通日志
数据结构转换
```

如果能够建立：

```text
明确Input
+
明确Output
+
Acceptance
+
Unit Test
+
Integration Test
```

那么：

```text
Worker
  ↓
Execute
  ↓
Test
  ↓
Reviewer
  ↓
PASS
  ↓
DONE
```

Human 可以完全不进入。

甚至具体实现只有 70 分，只要：

```text
功能正确
接口正确
测试稳定
没有危险副作用
```

V1 都可以接受。

因为当前目标可能只是：

> **先让系统正确跑通。**

代码质量、性能、抽象优雅程度可以成为未来独立的 Optimization Task。

---

# 三十八、MEDIUM：不一定看代码，但必须确认正确性定义

有一些代码不会删除文件、不会破坏数据库，却仍然存在很大风险。

例如：

```text
Rule Retrieval
Result Validate
Context Builder边界
LLM Judge Prompt
Batch Terminal Condition
State Transition
Pause / Resume
权限判断
```

它们最大的风险叫：

> **Silent Failure。**

例如：

```text
程序正常运行
↓
没有Exception
↓
测试如果覆盖不足也可能PASS
↓
输出格式完全正常
↓
但业务结果是错的
```

因此这种任务 Human 不一定需要逐行 Review Implementation。

Human 更应该 Review：

```text
Specification
Business Rule
Acceptance
Edge Cases
Regression Fixtures
Architecture Decision
```

然后让 Reviewer 验证实现是否符合这些标准。

也就是：

> **Human Review Specification，Agent Review Implementation。**

---

# 三十九、HIGH：Human 必须保留 Gate

某些操作即使自动测试全部 PASS，也不能完全交给 Agent 自治。

例如：

```text
删除文件
覆盖原始数据
递归目录操作

Shell / subprocess
系统命令执行

DROP / DELETE
生产数据库修改

生产部署

权限修改

Secret / API Key

认证授权

外部系统写操作

Git reset --hard
force push

自动修改业务规则
```

这些操作的共同特点不是“代码复杂”。

而是：

> **错误会改变真实世界状态，而且恢复成本可能很高。**

因此应该：

```text
Worker Plan
    ↓
Human Gate
    ↓
Execute
    ↓
Reviewer
    ↓
Human / Policy Gate
```

---

# 四十、Read 也不是天然安全

不能简单认为：

```text
Read = 安全
Write = 有风险
Delete = 危险
```

普通：

```text
read CSV
read JSON
read config
```

确实风险很低。

但：

```text
读取.env
读取SSH Key
读取Cookie
读取用户隐私
读取生产数据库
读取公司机密
↓
发送给外部LLM/API
```

虽然没有删除任何东西，却可能造成严重问题。

因此还需要一个基础安全模型：

```text
Confidentiality
机密性

Integrity
完整性

Availability
可用性
```

也就是常见的 CIA 三要素。

对于 Coding Agent，我可以把它简单理解成：

```text
它有没有看到不应该看的？

它有没有修改不应该修改的？

它有没有让本来可用的东西不可用了？
```

---

# 四十一、Human Attention 本身也是稀缺资源

以前容易认为：

> Review 越多越安全。

但实际上：

```text
大量低风险代码
        ↓
Human全部Review
        ↓
注意力消耗
        ↓
真正高风险代码出现
        ↓
Human已经疲劳
```

因此更合理的是：

```text
1000行普通Parser
→ Tests + Reviewer

20行生产数据库删除逻辑
→ Human重点Review

业务规则
→ Human确认Specification

Architecture
→ Human确认Decision

普通Implementation
→ AI自治
```

因此：

> **Human Attention 应该按照风险，而不是按照代码量分配。**

---

# 四十二、把 Risk Level 写进 Task

未来每个 Task 都可以增加：

```text
## Risk Level

LOW / MEDIUM / HIGH

## Side Effects

Read:
...

Write:
...

Delete:
...

External:
...

Secrets:
...

Production:
...

## Human Gate

Required:
YES / NO

Reason:
...
```

例如：

```text
Task:
T08 Context Builder

Risk:
MEDIUM

Side Effects:
Read article context
Write result JSONL

Delete:
None

External:
None

Human Gate:
Specification Review Only
```

另一个：

```text
Task:
T20 Cleanup Cache

Risk:
HIGH

Side Effects:
Delete cache directory

Human Gate:
REQUIRED
```

---

# 四十三、Lead 和 Reviewer 都要参与风险判断

不能只让 Lead 判断：

```text
Lead：
“我觉得这是LOW。”

↓
直接执行
```

应该：

```text
Lead
↓
Initial Risk Classification
↓
Worker
↓
Reviewer
↓
Risk Verification
```

Reviewer 再检查：

```text
Does this change:

[ ] Delete data?
[ ] Overwrite files?
[ ] Execute shell?
[ ] Modify database?
[ ] Access secrets?
[ ] Transmit sensitive data?
[ ] Change permissions?
[ ] Deploy production?
[ ] Change business rules?
[ ] Change architecture?
[ ] Introduce external side effects?
```

如果发现 Lead 低估风险：

```text
LOW
↓
Reviewer发现危险操作
↓
RISK_ESCALATION
↓
Lead
↓
Human
```

Reviewer 不能自行放行。

---

# 四十四、Risk-based Multi-Agent Workflow

加入 Risk Gate 后，原来的：

```text
Lead
 ↓
Worker
 ↓
Reviewer
 ↓
Lead
```

变成：

```text
                         Lead
                           │
                     Plan / Replan
                           │
                    Risk Classification
                           │
          ┌────────────────┼────────────────┐
          │                │                │
         LOW             MEDIUM            HIGH
          │                │                │
          ↓                ↓                ↓
       Worker           Worker        Human Plan Gate
          │                │                │
          ↓                ↓                ↓
        Test             Test            Worker
          │                │                │
          ↓                ↓                ↓
      Reviewer          Reviewer          Test
          │                │                │
          ↓                ↓                ↓
        PASS        Strong Verification  Reviewer
          │                │                │
          │           ┌────┴────┐           ↓
          │         PASS       FAIL      Human Gate
          │           │          │           │
          └───────────┼──────────┴───────────┘
                      │
                      ↓
                    Lead
                      │
              ┌───────┴───────┐
              │               │
            DONE            REPLAN
              │               │
              ↓               └────→ Worker
          Next Task
```

这时候 Human 不再是普通审批节点。

Human 变成：

> **Risk Owner。**

---

# 四十五、最终目标：Human 从正常控制环退出

最开始：

```text
Worker
 ↓
Human
 ↓
Worker
 ↓
Human
 ↓
Worker
```

Multi-Agent 以后：

```text
              NORMAL PATH

                 Lead
                  ↓
                 Plan
                  ↓
                Worker
                  ↓
                 Test
                  ↓
               Reviewer
                ↙    ↘
             PASS    FAIL
              ↓       ↓
            DONE    Replan
              ↓       │
          Next Task ←─┘
```

Human 只存在于异常路径：

```text
             EXCEPTION PATH

Architecture Change ──┐
Business Rule Change ─┤
High Risk Operation ──┤
Security Issue ───────┤
Requirement Ambiguity ├──→ HUMAN
Agent Conflict ───────┤
Production Change ────┤
Irreversible Action ──┘
```

最终目标不是：

> Human 不参与项目。

而是：

> **Human 不再参与每一次正常状态转换。**

---

# 四十六、Multi-Agent Coding 系统全景图

下面这张图是整套体系最值得长期保存的一张。

它可以叫：

> **Multi-Agent Coding Architecture & Control Flow**

也可以简单叫：

> **AI Coding 多 Agent 系统全景图**

```text
┌──────────────────────────────────────────────────────────────┐
│                         HUMAN OWNER                          │
│                                                              │
│  Product Goal │ Business Rules │ Architecture │ Risk Policy │
│                                                              │
│  Human只处理：                                                │
│  · Architecture Change                                       │
│  · Business Rule Change                                      │
│  · Security / Secret                                         │
│  · Production                                                │
│  · Irreversible Action                                       │
│  · Requirement Ambiguity                                     │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               │ Policy / Decision
                               ↓
┌──────────────────────────────────────────────────────────────┐
│                    SHARED PROJECT KNOWLEDGE                  │
│                                                              │
│  PRODUCT.md     WHAT      做什么 / 什么叫正确                 │
│  SYSTEM.md      HOW       系统怎么设计                        │
│  DECISIONS.md   WHY       为什么这样设计                      │
│  CONTRACTS.md   INTERFACE 模块之间怎么连接                    │
│                                                              │
│  config/        可调假设                                      │
│  tests/         正确性证据                                    │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ↓
                    ┌───────────────────┐
                    │    LEAD AGENT     │
                    │                   │
                    │   Plan / Replan   │
                    │   Dependency      │
                    │   Risk Classify   │
                    │   Dispatch        │
                    └─────────┬─────────┘
                              │
                       Create / Assign
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                         TASK LAYER                           │
│                                                              │
│ tasks/Txxx.md                                                │
│                                                              │
│ Goal                                                         │
│ Input / Output                                               │
│ Contract                                                     │
│ Scope / Out of Scope                                         │
│ Acceptance                                                   │
│ Tests                                                        │
│ Allowed / Forbidden Files                                    │
│ Risk Level                                                   │
│ Side Effects                                                 │
│ Human Gate                                                   │
└──────────────────────────────┬───────────────────────────────┘
                               │
                               ↓
                       ┌──────────────┐
                       │ WORKER AGENT │
                       │              │
                       │   Execute    │
                       │   Test       │
                       │   Commit     │
                       └──────┬───────┘
                              │
                         Evidence
                              │
                              ↓
                       ┌──────────────┐
                       │REVIEWER AGENT│
                       │              │
                       │ Acceptance   │
                       │ Tests        │
                       │ Diff         │
                       │ Risk         │
                       │ Drift        │
                       └──────┬───────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
                  PASS                 FAIL
                    │                   │
                    ↓                   ↓
                  Lead                Lead
                    │                   │
                  DONE               Replan
                    │                   │
                    │             Revision Task
                    │                   │
                    │                   ↓
                    │                Worker
                    │                   │
                    │               Reviewer
                    │                   │
                    └─────────┬─────────┘
                              │
                              ↓
                         NEXT TASK
```

但上面还只是**控制流**。

三个独立 Agent 真正能够协作，还需要下面这个共享状态层：

```text
┌──────────────────────────────────────────────────────────────┐
│                    SHARED SOURCE OF TRUTH                    │
│                                                              │
│  state/                                                      │
│  ├── project.yaml       当前项目状态                          │
│  └── tasks/*.yaml       当前Task状态                          │
│                                                              │
│  logs/                                                       │
│  └── events.jsonl       所有状态变化历史                      │
│                                                              │
│  reviews/                                                    │
│  └── Txxx-Rx.md         Reviewer证据                          │
│                                                              │
│  Git                                                         │
│  └── Commit             Worker交付的不可变代码锚点            │
│                                                              │
│  CHANGELOG.md                                                │
│  └── 工程决策与修改历史                                      │
└──────────────────────────────────────────────────────────────┘
```

因此完整关系实际上是：

```text
                        HUMAN
                          │
                       Policy
                          │
                          ↓
                    PROJECT DOCS
                          │
                          ↓
                        LEAD
                          │
                    Plan / Replan
                          │
                          ↓
                         TASK
                          │
                   Risk Classification
                          │
             ┌────────────┼────────────┐
             │            │            │
            LOW         MEDIUM        HIGH
             │            │            │
             │            │       HUMAN GATE
             │            │            │
             └────────────┼────────────┘
                          ↓
                       WORKER
                          │
                 Execute + Test
                          │
                          ↓
                     GIT COMMIT
                          │
                          ↓
                      REVIEWER
                          │
            ┌─────────────┼─────────────┐
            │             │             │
          PASS           FAIL      RISK ESCALATION
            │             │             │
            ↓             ↓             ↓
           LEAD          LEAD          HUMAN
            │             │
           DONE         REPLAN
            │             │
            │             ↓
            │         REVISION TASK
            │             │
            │             ↓
            │           WORKER
            │             │
            │         REVIEWER
            │             │
            └───────┬─────┘
                    ↓
                 NEXT TASK


整个过程中所有角色共同读写：

              ┌────────────────────┐
              │ SOURCE OF TRUTH    │
              │                    │
              │ Current State      │
              │ Event Log          │
              │ Task               │
              │ Review Evidence    │
              │ Git Commit         │
              │ Tests              │
              │ Changelog          │
              └────────────────────┘
```

---

# 四十七、最后把整个体系压缩成一棵树

如果未来我忘了所有文件名，只需要回来查看这棵树：

```text
AI-Native Software Engineering
│
├── 1. Knowledge：项目是什么？
│   │
│   ├── WHAT
│   │   └── PRODUCT.md
│   │
│   ├── HOW
│   │   └── SYSTEM.md
│   │
│   └── WHY
│       └── DECISIONS.md
│
├── 2. Contract：大家怎么连接？
│   │
│   └── INTERFACE
│       └── CONTRACTS.md
│
├── 3. Role：谁负责什么？
│   │
│   ├── CONTROL
│   │   └── LEAD.md
│   │
│   ├── EXECUTION
│   │   └── WORKER.md
│   │
│   └── VERIFICATION
│       └── REVIEWER.md
│
├── 4. Work：现在做什么？
│   │
│   └── TASK
│       ├── Goal
│       ├── Scope
│       ├── Out of Scope
│       ├── Input / Output
│       ├── Acceptance
│       ├── Test
│       └── Risk
│
├── 5. Coordination：怎么协作？
│   │
│   └── PROTOCOL.md
│       │
│       ├── Plan
│       ├── Execute
│       ├── Verify
│       └── Replan
│
├── 6. State：现在在哪里？
│   │
│   ├── Current State
│   │   └── state/*
│   │
│   └── Event History
│       └── events.jsonl
│
├── 7. Evidence：凭什么说它是对的？
│   │
│   ├── tests/
│   ├── fixtures/
│   ├── reviews/
│   └── Git Commit
│
├── 8. Risk：哪些事情AI不能自己决定？
│   │
│   ├── LOW
│   │   └── Agent Autonomous
│   │
│   ├── MEDIUM
│   │   └── Strong Agent Review
│   │
│   └── HIGH
│       └── Human Gate
│
├── 9. History：为什么变成现在这样？
│   │
│   ├── CHANGELOG.md
│   ├── Git History
│   └── Review History
│
└── 10. Human：人最终负责什么？
    │
    ├── Product Goal
    ├── Business Rules
    ├── Architecture Boundary
    ├── Risk Policy
    ├── Security
    ├── Irreversible Operations
    └── Exception Decision
```

---

# 四十八、现在我对 Vibecoding 的理解

最开始我以为：

> Vibecoding = 我描述需求，AI 帮我写代码。

后来变成：

> Vibecoding = 我设计，AI 实现，我 Review。

再继续发展：

> Vibecoding = 我定义目标、边界、正确性和风险，让多个 Agent 在明确的权限、状态、协议和验证机制下完成工程循环。

所以最终：

```text
Human
负责：
Goal
Boundary
Policy
Risk
Exception

Lead
负责：
Plan
Control
Replan

Worker
负责：
Execute

Reviewer
负责：
Verify

System
负责：
State
Protocol
Evidence
History
```

其中一个非常重要的变化是：

> **我不需要知道每一行代码是怎么写的，我需要知道为什么可以相信它。**

对于低风险、可验证、可逆的实现：

```text
Tests + Reviewer + Git
```

就是证据。

对于业务核心：

```text
Specification + Acceptance + Regression
```

是我需要关注的东西。

对于高风险操作：

```text
Human Gate
```

仍然保留。

因此最终追求的并不是：

> AI 写了多少代码。

而是：

> **有多少工程决策可以安全地从人的日常控制循环中移出去，同时仍然保持可验证、可追踪、可恢复、可控制。**

这可能才是我目前理解的 AI-Native Software Engineering 的核心。