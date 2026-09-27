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

## 0. 为什么需要这套手册

最开始使用 Coding Agent 时，很容易形成这样的工作方式：

```text
我提出需求
    ↓
Agent 写代码
    ↓
Agent 自己测试
    ↓
我检查
    ↓
有问题让 Agent 修改
    ↓
我再次检查
    ↓
进入下一步
```

小项目这样完全够用。

但项目变复杂以后，我逐渐发现两个问题。

第一，Agent 的能力越强，自主性也越强。如果没有明确边界，它可能在完成 A 的同时顺手修改 B、重构 C、升级 D，最后代码可能能跑，却已经偏离原来的系统设计。

第二，我自己变成了整个系统的人工 Runtime：

```text
Agent做完
   ↓
问我下一步

Agent修改完
   ↓
问我能不能继续

测试完成
   ↓
等我Review

Review失败
   ↓
我重新告诉它怎么改
```

这让我开始意识到：

> Vibecoding 真正需要解决的，不只是“怎么让 AI 写代码”，而是怎么把目标、架构、边界、权限、状态和验收标准外化，让 AI 在一个可控系统里工作。

单 Agent 和 Multi-Agent 的文档体系，本质上都是为了解决这个问题。

---

# 一、先理解一个核心：文档不是给 AI 看的说明书，而是项目的外部记忆

如果所有信息都存在聊天记录里：

```text
需求
架构
为什么这样设计
哪些方案已经否决
当前做到哪里
什么可以修改
什么不能修改
```

换一个 Agent，这些信息就没了。

因此需要把项目上下文从：

```text
Human Memory
+
Chat History
+
AI Memory
```

逐渐迁移到：

```text
Repository
```

理想情况下，一个完全不了解项目的新 Worker 进入仓库以后，通过阅读项目文件就能回答：

```text
WHAT
我要做什么？

HOW
系统怎么设计？

WHY
为什么这样设计？

NOW
现在做到哪里？

PERMISSION
我有什么权限？

HISTORY
以前发生了什么？
```

这就是整个文档体系存在的原因。

---

# 二、单 Agent：最基础的 Worker Context Pack

对于一个中大型、需要持续开发的项目，我目前采用六份核心文件：

```text
docs/
├── PRODUCT.md
├── SYSTEM.md
├── DECISIONS.md
├── TASKS.md
├── BUILD.md
└── CHANGELOG.md
```

它们分别对应：

```text
WHAT        → PRODUCT
HOW         → SYSTEM
WHY         → DECISIONS
NOW         → TASKS
PERMISSION  → BUILD
HISTORY     → CHANGELOG
```

这六个问题比六个文件名本身重要。

---

# 三、PRODUCT.md：定义“什么叫做对”

## 它解决什么问题？

Agent 最大的问题之一不是不会写代码，而是不知道：

> 什么才算业务正确。

因此 PRODUCT 不是项目宣传，而是项目的**正确性来源**。

推荐结构：

```text
# PRODUCT

## 项目目标

## 用户是谁

## 输入

## 输出

## 核心业务流程

## Business Rules

## Edge Cases

## Non-Goals

## Definition of Done
```

其中最重要的是：

```text
输入
输出
正确性
边界
```

还应该明确：

> **谁拥有最终正确性的定义权。**

可能是：

```text
客户规则
产品经理
API协议
数据库Schema
测试用例
法律规范
用户确认
```

例如一个业务 Agent，如果业务规则属于客户，就必须明确：

```text
客户拥有业务规则定义权。

Agent不得因为自己的知识认为另一种分类“更合理”而修改规则。
```

否则 Worker 很容易把“实现需求”变成“重新设计需求”。

---

# 四、SYSTEM.md：定义系统边界

PRODUCT 回答：

> 做什么？

SYSTEM 回答：

> 已经决定怎么做？

推荐结构：

```text
# SYSTEM

## Architecture

## Components

## Responsibility

## Data Flow

## Control Flow

## State

## Storage

## Error Flow

## External Dependencies

## Interfaces

## Observability
```

我认为其中一个非常值得长期保留的写法是：

| 模块 | 负责 | 不负责 |
|---|---|---|
| Runtime | 调度 | 业务判断 |
| Context Builder | 提取证据 | 判断业务 |
| Judge | 语义判断 | 修改业务规则 |
| Storage | 保存数据 | 做决策 |

为什么“**不负责**”也要写？

因为 AI 很容易发生 Responsibility Drift：

```text
最开始：

Context Builder
= 提取文本

后来：

Context Builder
= 提取文本
+ 调LLM
+ 判断业务
+ 修改状态
+ 处理失败
```

每一步单独看似乎都有理由。

最后模块边界却消失了。

因此 SYSTEM 实际上也是：

> **Boundary Document。**

---

# 五、DECISIONS.md：告诉 Agent“这里不是没想到”

SYSTEM 只能告诉 Worker：

```text
我们选择 A。
```

但 Worker 可能认为：

```text
B更新、更高级。

我帮你升级成B。
```

所以需要记录重要的 Architecture Decision。

推荐使用 ADR：

```text
# ADR-001：决策名称

Status:
Accepted

Problem:
要解决什么问题？

Options:
A / B / C

Decision:
最终选择什么？

Why:
为什么？

Trade-offs:
为此牺牲什么？

Rejected:
为什么暂时不用其他方案？

Revisit When:
出现什么证据后允许重新讨论？
```

最后一个：

```text
Revisit When
```

非常重要。

它让一个决策从：

> 永远不允许改变。

变成：

> **现在不改变，直到现实证据满足重新评估条件。**

例如：

```text
Decision:
V1使用逻辑Context切片。

Rejected for V1:
Embedding / MMR / Rerank。

Why:
目前没有真实数据证明复杂方案值得增加工程成本。

Revisit When:
Context Token长期过高；
人工修正显示明显漏召回；
Trace证明Context噪音是主要错误来源。
```

于是未来 Agent 就知道：

> Embedding 不是 TODO，而是当前明确拒绝的方案。

---

# 六、TASKS.md：限制 Agent 当前只做什么

前面的文件描述整个项目。

TASKS 只回答：

> **现在做什么？**

一个好的 Task 至少应该有：

```text
# T08 Context Builder

## Goal

## Dependencies

## Input

## Output

## Scope

## Out of Scope

## Acceptance Criteria

## Tests

## Observability
```

其中最值得长期保留的是：

```text
Goal
Out of Scope
Acceptance Criteria
```

它们分别控制：

```text
Goal
→ 不要做偏

Out of Scope
→ 不要做多

Acceptance
→ 不要假装已经完成
```

Task 不应该只是：

```text
实现Context Builder。
```

因为这几乎把所有决策权都交给了 Agent。

---

# 七、BUILD.md：定义 Worker 的权限

即使前面四份文件都写好了，Worker 仍然可能：

```text
一次完成十个Task
顺手重构
升级依赖
修改架构
测试失败继续
完成后直接做下一步
```

因此 BUILD 定义的不是产品，而是：

> **Worker Operating Policy。**

最简单的控制循环：

```text
Read
 ↓
Plan
 ↓
STOP
 ↓
Human Review
 ↓
Implement
 ↓
Test
 ↓
Review
 ↓
Fix
 ↓
Commit
 ↓
Log
```

其中应该定义几个 Gate：

```text
Plan Gate
Scope Gate
Test Gate
Review Gate
Git Gate
Documentation Gate
```

以及非常重要的：

## Stop Conditions

例如：

```text
需求存在歧义
架构需要改变
业务规则需要改变
需要新增重要依赖
测试证明当前设计不可行
需要修改Task范围外模块
发现用户已有未提交修改
可能进行破坏性数据操作
```

出现这些情况：

```text
STOP
 ↓
报告问题
 ↓
等待Human决定
```

而不是让 Agent 自己“合理处理”。

---

# 八、CHANGELOG.md：保存项目历史

Git 告诉我们：

```text
代码改了什么。
```

CHANGELOG 应该回答：

```text
为什么改？
解决什么问题？
验证了吗？
Review结果是什么？
还有什么没解决？
```

推荐：

```text
## YYYY-MM-DD — T08 Context Builder

### Changed

### Why

### Tests

### Review

### Git Commit

### Known Limitations
```

因此：

```text
Git
= Code History

CHANGELOG
= Engineering History
```

两者不是重复关系。

---

# 九、除了六份 MD，还有两个重要部分

## tests / fixtures：正确性的实例化

文档可能写：

```text
普通微信提及不构成业务。
```

测试直接表达：

```text
input:
某文章仅出现“微信公众号”

expected:
business = []
```

因此：

> 文档告诉 Agent 什么叫正确，测试证明它有没有做到。

随着项目成熟，测试对 Worker 的约束甚至可能比 Prompt 更强。

---

## config：保存可调假设

例如：

```text
batch_size
timeout
retry
context_window
overlap
model
cache_ttl
```

这些既不是业务真理，也不是架构真理。

它们只是：

> **当前参数假设。**

因此最好配置化。

一个很好用的分类是：

```text
业务事实
→ PRODUCT / rules

架构事实
→ SYSTEM / DECISIONS

可调假设
→ config

正确性证据
→ tests
```

---

# 十、单 Agent 的完整工作方式

最终形成：

```text
                 Human
                   │
             PRODUCT
             SYSTEM
            DECISIONS
                   │
                   ↓
                 TASK
                   │
                   ↓
                 Worker
                   │
                  Plan
                   │
                   ↓
              Human Review
                   │
                   ↓
                Execute
                   │
                  Test
                   │
                   ↓
              Human Review
              ↙           ↘
           FAIL           PASS
            ↓               ↓
          Worker           Git
            ↑               ↓
            └──── Fix    CHANGELOG
                            ↓
                         Next Task
```

这里的问题也很明显：

> Human 同时承担了 Lead 和 Reviewer。

项目越来越大以后，人就会成为控制循环的瓶颈。

这自然引出了 Multi-Agent。

---

# 十一、Multi-Agent：不是增加三个程序员，而是拆开三种责任

我目前更认可的基本模型是：

```text
Plan → Execute → Verify → Replan
```

分别对应：

```text
Lead Agent
    ↓
Worker Agent
    ↓
Reviewer Agent
    ↓
Lead Agent
```

三个角色：

```text
Lead
= Control

Worker
= Execution

Reviewer
= Verification
```

这比“三个 Agent 一起写代码”重要得多。

---

# 十二、Lead Agent：只负责控制

Lead 负责：

```text
读取Project State
读取Review
检查Dependency
选择READY Task
生成Plan
分配Worker
处理FAIL
Replan
决定Next Action
```

核心问题永远是：

> **下一步应该发生什么？**

Lead 不应该：

```text
直接写业务代码
Review自己的实现
擅自修改PRODUCT
擅自修改Business Rule
绕过Reviewer宣布完成
```

Lead 是：

> **Control Plane。**

---

# 十三、Worker Agent：只负责执行

Worker 需要知道：

```text
我的Task是什么？
输入是什么？
输出是什么？
Contract是什么？
允许修改什么？
不能修改什么？
Acceptance是什么？
怎么证明完成？
```

然后：

```text
Execute
 ↓
Test
 ↓
提交Evidence
 ↓
IMPLEMENTED
 ↓
STOP
```

Worker 不决定：

```text
下一个Task是什么
架构要不要改变
业务规则要不要改变
Reviewer错没错
```

---

# 十四、Reviewer Agent：只负责验证

Reviewer 负责：

```text
读取Task
 ↓
读取Acceptance
 ↓
读取Worker Diff
 ↓
运行/检查Test
 ↓
检查Architecture Drift
 ↓
检查权限越界
 ↓
PASS / FAIL
```

输出：

```text
Verdict
Evidence
Findings
Severity
```

Reviewer 最重要的约束是：

> **只有验证权，没有生产代码修改权。**

Reviewer 不应该看到问题以后直接修改代码。

因为：

```text
Reviewer发现问题
       ↓
Lead判断问题性质
       ↓
Implementation Bug?
Architecture Problem?
Task Problem?
Business Rule Problem?
       ↓
重新决定下一步
```

否则 Reviewer 会慢慢变成第二个 Worker。

---

# 十五、Multi-Agent 为什么需要 Shared Workspace

如果 Lead、Worker、Reviewer 是三个不同应用：

```text
Lead不知道Worker刚刚干了什么。

Reviewer不知道当前Review哪个Commit。

Worker不知道Reviewer上次为什么FAIL。
```

因此不能依赖：

```text
Chat History
AI Memory
自然语言转述
```

必须有：

> **External Source of Truth。**

整体结构：

```text
                    Human
                      │
                 Policy Gate
                      │
                      ↓
                   Lead
                      │
                    Plan
                      ↓
              Shared Workspace
               ↙            ↘
          Worker            Reviewer
          Execute            Verify
               ↘            ↙
              Shared Workspace
                      ↓
                    Lead
                      ↓
                   Replan
```

---

# 十六、Multi-Agent 的 Shared Documents

原来的项目级文件仍然保留：

```text
docs/

PRODUCT.md
SYSTEM.md
DECISIONS.md
```

但需要增加：

```text
CONTRACTS.md
```

最终：

```text
WHAT        → PRODUCT
HOW         → SYSTEM
WHY         → DECISIONS
INTERFACE   → CONTRACTS
```

---

# 十七、CONTRACTS.md：Multi-Agent 最重要的新文件之一

单 Worker 时，很多接口信息可以依靠同一个 Agent 的上下文维持。

多个 Worker 同时开发时不行。

例如：

```text
Worker A:
Fetch返回str。

Worker B:
Context以为Fetch返回ArticleContent。

Worker C:
Storage认为ArticleContent还有status。
```

三个人可能都写得很好。

最后：

> 接不起来。

所以 CONTRACTS 定义模块之间的接口。

例如：

```text
## ArticleContent

Input:
article_id: str
url: str

Output:
content: str
source: str
fetch_status: enum

Errors:
FETCH_TIMEOUT
FETCH_FORBIDDEN
FETCH_NOT_FOUND
EMPTY_CONTENT
```

再例如：

```text
## ContextBuilder

Input:
ArticleContent
KeywordHits
Config

Output:
ContextSet
```

核心原则：

> **SYSTEM 定义模块；CONTRACTS 定义模块怎么说话。**

多 Agent 能不能真正并行，很大程度取决于 Contract 是否稳定。

---

# 十八、每个 Agent 还需要自己的 Role Manual

目录：

```text
agents/

LEAD.md
WORKER.md
REVIEWER.md
```

Shared Docs 定义：

> 世界是什么。

Role Docs 定义：

> **你在这个世界里能做什么。**

---

# 十九、LEAD.md 应该写什么

推荐：

```text
# ROLE: LEAD

## Mission

## Responsibilities

## Allowed Actions

## Forbidden Actions

## Inputs

## Outputs

## State Fields You Own

## Decision Rules

## Escalation Conditions

## Handoff Protocol
```

重点不是告诉 Lead：

> “你是一个优秀的资深软件架构师。”

这种 Prompt 没什么约束力。

真正重要的是：

```text
你能修改哪些状态？

什么情况下允许创建Task？

什么情况下必须等待Human？

Reviewer FAIL以后你做什么？

什么情况下不能Replan？
```

---

# 二十、WORKER.md 应该写什么

推荐：

```text
# ROLE: WORKER

## Mission

## Responsibilities

## Allowed Files

## Forbidden Files

## Required Inputs

## Required Outputs

## Test Requirements

## Git Requirements

## Stop Conditions

## Handoff Format
```

尤其要明确：

```text
Worker不能自己开始下一个Task。

Worker不能改变Task Scope。

Worker不能修改Architecture。

Worker不能修改Business Rule。

Worker完成后必须STOP。
```

---

# 二十一、REVIEWER.md 应该写什么

推荐：

```text
# ROLE: REVIEWER

## Mission

## Review Inputs

## Acceptance Procedure

## Required Evidence

## Architecture Drift Checks

## Permission Checks

## PASS Conditions

## FAIL Conditions

## Forbidden Actions

## Review Output Format
```

Reviewer 输出最好高度结构化：

```text
Task:
Target Commit:

Verdict:
PASS / FAIL

Acceptance:
[x]
[x]
[ ]

Findings:

Evidence:

Architecture Drift:
YES / NO

Business Rule Conflict:
YES / NO

Return To:
LEAD
```

这样 Lead 不需要重新理解 Reviewer 的一大段自由文本。

---

# 二十二、Multi-Agent 还需要 PROTOCOL.md

三个 Agent 有了自己的 Role 还不够。

还必须定义：

> **它们怎么交接。**

这就是 Protocol。

例如：

```text
BLOCKED
 ↓
READY
 ↓
ASSIGNED
 ↓
RUNNING
 ↓
IMPLEMENTED
 ↓
REVIEWING
 ↓
PASS / FAIL
```

FAIL：

```text
FAIL
 ↓
Lead
 ↓
Replan
 ↓
REVISION_READY
 ↓
Worker
```

PASS：

```text
PASS
 ↓
Lead
 ↓
DONE
 ↓
Unlock Dependencies
```

需要人工：

```text
NEEDS_HUMAN
 ↓
STOP ALL RELATED WORK
```

因此 PROTOCOL 本质上定义：

> **Multi-Agent State Machine。**

---

# 二十三、状态转换权必须明确

不能所有 Agent 都能修改所有状态。

例如：

| 当前状态 | 操作者 | 下一状态 |
|---|---|---|
| BLOCKED | Lead | READY |
| READY | Lead | ASSIGNED |
| ASSIGNED | Worker | RUNNING |
| RUNNING | Worker | IMPLEMENTED |
| IMPLEMENTED | Reviewer | REVIEWING |
| REVIEWING | Reviewer | PASS / FAIL |
| FAIL | Lead | REVISION_READY |
| PASS | Lead | DONE |

这里有一个很好用的原则：

> **谁负责一个阶段，谁只能更新属于自己的状态。**

这样 Agent 之间不会互相覆盖控制权。

---

# 二十四、State 和 Log 必须分开

最开始很容易想到：

```text
shared_log.md
```

三个 Agent 都往里面写。

但 Log 只能回答：

> 发生过什么？

它不适合作为：

> 现在到底是什么状态？

所以 Multi-Agent 至少需要：

```text
Current State
+
Event Log
```

例如：

```text
state/tasks/T08.yaml
```

保存：

```text
task: T08
revision: 2
status: NEEDS_FIX
worker: worker-a
review_status: FAIL
target_commit: abc123
last_review: reviews/T08-R1.md
next_action: FIX_REVIEW_FINDINGS
```

而：

```text
logs/events.jsonl
```

保存：

```text
13:00 Lead → READY
13:05 Worker → RUNNING
13:30 Worker → IMPLEMENTED
13:40 Reviewer → FAIL
13:43 Lead → REVISION_READY
```

所以：

```text
State
= 现在是什么

Event Log
= 怎么变成现在这样
```

---

# 二十五、Task 也应该从列表升级为 Task Package

单 Worker 可以：

```text
TASKS.md
```

Multi-Agent 更适合：

```text
tasks/

T001.md
T002.md
T003.md
```

每个 Task 成为独立合同：

```text
# T008 Context Builder

Status:
READY

Owner:
Worker-A

Depends On:
T006
T007

Goal:

Input:
See CONTRACTS#KeywordHits

Output:
See CONTRACTS#ContextSet

Scope:

Out of Scope:

Allowed Files:

Forbidden Files:

Acceptance Criteria:

Tests:

Observability:

Reviewer:
Reviewer-A
```

这样 Lead 分配任务时，不需要给 Worker 复制一大段聊天记录。

只需要告诉它：

> 执行 T008。

---

# 二十六、Review 也应该成为 Artifact

目录：

```text
reviews/

T008-R1.md
T008-R2.md
```

例如：

```text
# T008 Review R1

Target Commit:
abc123

Verdict:
FAIL

Acceptance:

[x] 单关键词
[x] 多关键词
[x] overlap
[ ] 远距离Context不能物理合并

Evidence:

test_context_far_distance FAILED

Finding:

两个不重叠窗口被错误合并。

Severity:
BLOCKING

Architecture Drift:
NO

Return To:
LEAD
```

然后 Lead 创建：

```text
T008 Revision 2
```

重新交给 Worker。

于是：

```text
Task
 ↓
Implementation
 ↓
Commit
 ↓
Review
 ↓
FAIL
 ↓
Replan
 ↓
Revision
 ↓
Commit
 ↓
Review
 ↓
PASS
```

形成完整审计链。

---

# 二十七、Git 在 Multi-Agent 中还有通信作用

单 Agent 中：

> Git 是版本控制。

Multi-Agent 中：

> Git 还是 Agent 之间的交付锚点。

Worker 不应该告诉 Reviewer：

> “你看看我当前目录。”

而应该告诉它：

```text
Base Commit:
91fa001

Target Commit:
a83fc21
```

Reviewer Review：

```text
91fa001..a83fc21
```

最后：

```text
Review:
T008-R2

Target:
a83fc21

Verdict:
PASS
```

于是项目获得明确关系：

```text
Task
 ↓
Implementation Commit
 ↓
Test Evidence
 ↓
Review
 ↓
PASS
 ↓
DONE
```

---

# 二十八、什么时候三个 Agent 可以真正并行？

不是：

> 有三个 Agent，所以开三个 Task。

而是看：

> **Dependency Graph。**

例如：

```text
        T01
         ↓
    ┌────┴────┐
    ↓         ↓
   T02       T03
Storage     Config
    ↓         ↓
    └────┬────┘
         ↓
        T04
       Runtime
```

T02 与 T03：

```text
没有相互依赖
+
Contract已冻结
```

可以：

```text
Worker A → T02
Worker B → T03
```

同时执行。

T04 必须：

```text
T02 PASS
AND
T03 PASS
```

以后才能 READY。

所以：

> **并行能力不是 Agent 数量决定的，而是任务依赖与接口稳定性决定的。**

---

# 二十九、完整的 Multi-Agent 目录

对于复杂长期项目，可以演化为：

```text
project/
│
├── docs/
│   ├── PRODUCT.md
│   ├── SYSTEM.md
│   ├── DECISIONS.md
│   └── CONTRACTS.md
│
├── agents/
│   ├── LEAD.md
│   ├── WORKER.md
│   └── REVIEWER.md
│
├── protocol/
│   └── PROTOCOL.md
│
├── tasks/
│   ├── T001.md
│   ├── T002.md
│   └── ...
│
├── reviews/
│   ├── T001-R1.md
│   └── ...
│
├── state/
│   ├── project.yaml
│   └── tasks/
│
├── logs/
│   └── events.jsonl
│
├── config/
│
├── tests/
│
└── CHANGELOG.md
```

但仍然不要死记目录。

它们实际对应：

```text
Knowledge
→ PRODUCT / SYSTEM / DECISIONS

Interface
→ CONTRACTS

Role
→ LEAD / WORKER / REVIEWER

Work Unit
→ TASK

Coordination
→ PROTOCOL

Current Truth
→ STATE

Verification
→ TEST / REVIEW

Immutable Evidence
→ GIT

History
→ EVENT LOG / CHANGELOG
```

---

# 三十、Human 应该在哪里？

Multi-Agent 的最终目的不是：

> 完全没人管。

而是让 Human 从**正常控制循环**里退出。

单 Agent：

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

Multi-Agent：

```text
             Lead
               ↓
             Plan
               ↓
            Worker
               ↓
            Verify
            ↙    ↘
         PASS    FAIL
          ↓       ↓
        DONE    Replan
          ↓       │
       Next ←─────┘
```

Human 只处理：

```text
Architecture Change

Business Rule Change

High Risk Operation

Requirement Ambiguity

Agent Conflict

Policy Exception
```

也就是：

```text
正常路径
→ Agent自治

异常路径
→ Human
```

这才是真正降低人工审批成本。

---

# 三十一、单 Agent 和 Multi-Agent 最终对照

| 问题 | 单 Agent | Multi-Agent |
|---|---|---|
| 项目目标 | PRODUCT | PRODUCT |
| 系统设计 | SYSTEM | SYSTEM |
| 架构原因 | DECISIONS | DECISIONS |
| 接口 | SYSTEM中即可 | CONTRACTS独立 |
| 当前任务 | TASKS | tasks/* |
| Worker权限 | BUILD | WORKER.md |
| 调度 | Human | LEAD.md |
| Review | Human | REVIEWER.md |
| 协作协议 | Human隐式控制 | PROTOCOL |
| 当前状态 | TASKS/人工判断 | state/* |
| 历史 | CHANGELOG | Event Log + CHANGELOG |
| 验证 | Tests + Human | Tests + Reviewer |
| 版本证据 | Git | Git |
| Replan | Human | Lead |
| 架构变化 | Human | Human |
| 业务规则变化 | Human | Human |

---

# 三十二、什么时候用单 Agent，什么时候 Multi-Agent？

不要因为 Multi-Agent 更高级就一定使用。

简单项目：

```text
Human + Worker
```

完全够用。

如果出现：

```text
每一步都必须人工批准

大量时间花在检查AI

任务数量增加

多个模块可以并行

Worker自己Review可信度不足

项目持续时间很长

不同Agent需要交接

Human开始成为Runtime
```

才说明：

> Multi-Agent 开始具有工程价值。

因此 Multi-Agent 不是技术炫技。

它解决的是：

> **协调成本和人工控制瓶颈。**

---

# 三十三、我目前给自己的 Vibecoding 开工检查

## 单 Agent

开工前问：

```text
WHAT
→ PRODUCT写了吗？

HOW
→ SYSTEM写了吗？

WHY
→ 关键Decision记录了吗？

NOW
→ Task明确吗？

PERMISSION
→ Worker知道不能做什么吗？

ACCEPTANCE
→ 怎么证明做对？

HISTORY
→ Git/CHANGELOG准备好吗？
```

满足以后：

```text
Plan
→ Review
→ Execute
→ Test
→ Review
→ Commit
→ Log
```

---

## Multi-Agent

除了上面，还要问：

```text
INTERFACE
→ Contract冻结了吗？

ROLE
→ 每个Agent权限明确吗？

PROTOCOL
→ 状态怎么流转？

STATE
→ 谁是当前事实来源？

HANDOFF
→ Agent之间交付什么？

EVIDENCE
→ Reviewer根据什么判断？

DEPENDENCY
→ 哪些Task真的可以并行？

ESCALATION
→ 什么情况下必须找Human？
```

然后才能：

```text
Lead Plan
    ↓
Worker Execute
    ↓
Reviewer Verify
    ↓
PASS ─────→ Next Task
    ↓
   FAIL
    ↓
Lead Replan
    ↓
Worker Execute
    ↓
...
```

---

# 三十四、最后：这不是最终模板，而是 V0.1

这篇文章最重要的地方不是：

> 以后所有项目必须创建这些 MD。

真正应该保留的是：

```text
WHAT
HOW
WHY
NOW
PERMISSION
INTERFACE
STATE
EVIDENCE
HISTORY
```

项目小：

> 合并文件。

项目大：

> 拆文件。

单 Worker：

> Human 承担 Lead + Reviewer。

Multi-Agent：

> 把 Plan / Execute / Verify / Replan 拆成不同角色。

未来如果发现：

```text
DECISIONS没人读取
→ 改。

CHANGELOG和Event Log重复
→ 合并。

Reviewer经常误判
→ 改Review Contract。

Lead频繁找Human
→ 分析Escalation规则。

三个Agent交接成本比人工还高
→ 回到更简单的模式。
```

所以这套系统本身也应该接受：

```text
使用
 ↓
观察
 ↓
发现摩擦
 ↓
分析原因
 ↓
修改协议
 ↓
再次验证
```

Vibecoding 的目标并不是找到一份“万能 Prompt”。

而是逐渐建立一套：

> **即使更换模型、更换 Agent、更换 IDE、更换项目，仍然能够把目标、边界、状态、权限和验证方法交给新的执行者，并让整个工程继续运行的外部系统。**

当这些东西存在以后，AI 才真正从“聊天窗口里帮我写代码的人”，变成了一个可以被调度、被约束、被验证的工程 Worker。