---
description: ""
title: "让 Agent 施工，让 Human 决定怎么走"
draft: false
date: "2026-09-30T09:05:24+08:00"
slug: "VibeCoding"
categories:
 - VibeCoding
tags:
 - null
image: ""
---

# 第一次 Vibe Coding：让 Agent 施工，让 Human 决定怎么走

这是我第一次真正用 Vibe Coding 的方式完成一个项目。

现在回头看，它的自动化程度并不高。

Agent 每完成一个阶段都会回来问我：

> Plan 可以执行吗？  
> 要不要运行测试？  
> 要不要继续修改？  
> 这个 Task 可以结束吗？  
> 是否进入下一个 Task？

后来我又尝试了自治程度更高的开发方式。

但我没有因此认为第一种方式已经落后。

恰恰相反，真正做过更自治的项目以后，我才重新理解它的价值：

> **Agent 负责提出路径和执行路径，Human 保留在关键分叉点修改路径的权力。**

它不是 Autonomous Runtime。

它更像一种：

> **Human-Controlled Interactive Development。**

---

# 一、最开始的问题不是怎么让 Agent 自治

第一次做完整项目以前，我使用 AI 写代码的方式很简单：

```text
想到一个功能
↓
告诉 AI
↓
AI 写代码
↓
发现问题
↓
继续聊天
↓
再修改
```

小功能没有什么问题。

但项目一旦变大，一个问题很快就会出现：

> 项目到底存在于哪里？

如果产品目标、架构、边界、任务和历史决策全部存在于聊天记录里，那么 AI 每一次工作都依赖当前 Conversation。

于是第一次真正做项目时，我开始把项目从聊天里搬出来。

逐渐形成：

```text
PRODUCT.md
SYSTEM.md
DECISIONS.md
TASKS.md
BUILD.md
CHANGELOG.md
```

它们分别回答不同的问题：

```text
PRODUCT
→ 我要解决什么问题？

SYSTEM
→ 系统准备怎么工作？

DECISIONS
→ 哪些关键选择已经确定？

TASKS
→ 接下来需要完成什么？

BUILD
→ Human 和 AI 怎么一起施工？

CHANGELOG
→ 项目实际上发生了什么？
```

从这里开始：

> **聊天不再是项目本身。**

真正的 Source of Truth 开始回到项目文件。

---

# 二、第一次形成了三个角色

当时的协作结构并不是一个 Agent 自己把整个项目做完。

而是三个角色：

```text
Product / Business Owner
        │
        │
        ├──────────────┐
        ▼              ▼
Design / Review     Codex Worker
Assistant
```

## Product / Business Owner

也就是我。

负责：

- 真实业务目标；
- 甲方规则；
- 架构关键取舍；
- Review；
- 验收；
- 是否进入下一 Task。

真正的最终控制权仍然在人。

---

## Design / Review Assistant

负责：

- 维护项目文档；
- 帮助拆 Task；
- Review Worker 的 Plan；
- Review Diff 和测试结果；
- 检查架构漂移；
- 帮助定位问题；
- 经我批准以后更新设计文档。

它不是主要施工者。

它更像我的设计和 Review 助手。

---

## Codex Worker

Worker 才是真正负责施工的角色。

它负责：

- 阅读指定文档；
- 理解当前 Task；
- 输出 Plan；
- 等待 Review；
- 实现当前 Task；
- 运行测试；
- 报告 Diff；
- 根据 Review 修复；
- Git Commit；
- 更新 CHANGELOG。

因此第一次 Vibe Coding 并不是：

```text
Human
↓
告诉 AI 一个需求
↓
AI 写代码
```

而已经变成：

```text
Human
↓
Project Docs
↓
Task
↓
Worker Plan
↓
Human Decision
↓
Implementation
↓
Evidence
↓
Human Decision
```

---

# 三、真正的核心：Task-by-Task

当时有一个非常重要的原则：

> **不要把 TASKS.md 全部一次性交给 Worker 自动完成。**

每一次只执行一个 Task。

例如：

```text
T01
↓
完成 / 验证
↓
T02
↓
完成 / 验证
↓
T03
↓
...
```

给 Worker 的任务类似：

```text
阅读：

PRODUCT.md
SYSTEM.md
DECISIONS.md
BUILD.md

执行：

TASKS.md 中的 Txx

要求：

先输出 Plan。
不要修改代码。
等待 Review 后再执行。
```

这意味着 Worker 即使知道后面还有 T02、T03、T04，也不能自己一路向后施工。

**项目推进权仍然属于 Human。**

---

# 四、Task 开始之前，先给我 Plan

这是我后来依然非常喜欢第一种方式的原因。

Worker 拿到 Task 以后，第一件事不是修改代码。

而是：

```text
Read Docs
↓
Read Relevant Code
↓
Understand Task
↓
Generate Plan
↓
STOP
```

Plan 至少需要告诉我：

```text
当前 Task 是什么？

你怎么理解这个需求？

准备修改哪些文件？

准备分几步实现？

有什么风险？

准备怎么测试？

是否会触碰
PRODUCT / SYSTEM / DECISIONS？
```

然后才来到：

```text
          Worker Plan
               ↓
          HUMAN GATE
        ┌──────┼──────┐
        ↓      ↓      ↓
      Accept  Change  Stop
        │
        ▼
    Implement
```

这个停顿非常重要。

---

# 五、Plan Gate 不是形式上的“审批”

以前我可能会觉得：

> AI 每一步都来问我，有点麻烦。

后来我才意识到，这恰恰是这种模式最重要的能力之一。

假设 Worker 原本准备：

```text
修改 A
+
修改 B
+
修改 C
+
新增一个抽象层
+
重构一部分旧逻辑
```

但我看到 Plan 后可能马上发现：

> 不对。

真正需要的可能只是：

```text
修改 A
+
一个最小测试
```

于是原本可能发生：

```text
错误方向
↓
写代码
↓
改多个文件
↓
测试
↓
Review
↓
发现方向错了
↓
返工
```

被提前截断成：

```text
错误 Plan
↓
Human 发现
↓
修改 Plan
↓
再执行
```

错误停在了最便宜的位置。

所以 Plan Gate 真正做的不是：

> “Human 给 AI 签字。”

而是：

> **在真正付出实现成本之前，让 Human 有机会重新选择路径。**

---

# 六、我不需要比 Agent 更会写代码

这也是我逐渐理解的一件事情。

Worker 可能比我更熟悉：

```text
怎么拆函数
怎么调用 API
怎么组织模块
怎么处理异常
怎么写测试
```

但这不代表它应该决定所有事情。

Human 更应该判断：

```text
这个功能真的有必要吗？

为什么要增加这个抽象？

有没有更简单的方法？

这是不是已经偏离当前 Task？

这个风险值得吗？

现在真的需要完整测试吗？

这个方向是不是和最初目标冲突？
```

于是 Human 和 Agent 的关系逐渐变成：

```text
Human
负责目标 / 边界 / 取舍
        ↓
Agent
提出实现 Plan
        ↓
Human
修改路径
        ↓
Agent
负责具体施工
```

我不需要亲自写出所有代码，才能拥有项目控制权。

---

# 七、Plan 通过以后，Worker 才开始施工

只有 Plan 被接受以后：

```text
Plan Approved
↓
Implement
↓
Test
↓
Diff Summary
↓
Review
```

Worker 只能修改当前 Task 所需要的内容。

如果施工过程中发现：

> 顺便还可以把另外一个模块重构一下。

不能直接做。

而是：

```text
发现额外问题
↓
记录
↓
报告
↓
Human 决定
├── 加入当前 Task
├── 建立新 Task
└── Ignore
```

这个规则后来一直影响我的工程思维：

> **不要因为“顺手能做”，就让复杂度自然生长。**

---

# 八、做完以后，再把选择权交回来

实现完成并不代表 Task 自动结束。

Worker 需要提供证据，例如：

```text
git diff --stat

关键 Diff 摘要

实际运行的测试

测试结果

新增 / 修改测试

已知限制

是否与设计文档一致
```

然后再次：

```text
Implementation
↓
Evidence
↓
HUMAN GATE
```

Human 再决定：

```text
Accept

Fix

继续测试

查看 Diff

让 Review Assistant 检查

自己检查代码

Skip 某些验证

直接进入下一步

Stop
```

这也是第一代另一个非常重要的特点：

> **Human 不只是决定做不做，还可以决定做到多深。**

---

# 九、每一步都问我，有时候就是优点

假设 Agent 说：

> 当前修改已经通过 targeted test，建议再运行 full regression。

我可以判断：

> 现在不用。

然后：

```text
SKIP
↓
Next Step
```

也可能另一个 Task 改动了核心 Runtime。

这时候我会说：

> 跑全量测试。

于是：

```text
RUN FULL TEST
↓
Review Result
```

甚至可能测试本身没有必要：

```text
Agent 建议 Test
        ↓
      Human
   ┌────┼────┐
   ↓    ↓    ↓
  RUN  SKIP CHANGE
```

因此第一代的运行路径并不是完全提前冻结的。

Human 可以不断根据刚刚获得的信息调整：

```text
执行什么？

不执行什么？

验证多深？

要不要继续？

要不要改变方向？
```

---

# 十、Human 可以实时给执行路径剪枝

这是后来使用更自治的开发方式以后，我重新认识到的优势。

一个高度自治的 Runtime 可能提前规定：

```text
Implement
↓
Test
↓
Review
↓
Regression
↓
Next Task
```

规则一旦满足，它就继续执行。

但 Human 有一种非常强的能力：

> **知道什么时候“不值得继续做”。**

例如：

```text
这个测试现在不用跑。

这个 Review 没必要。

这个方案不要继续研究。

先验证最小路径。

这个问题暂时接受。

这里不用继续优化。
```

于是 Human 可以直接剪掉执行树上的分支：

```text
                 Current State
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
        Test        Review      Continue
          │
          X
       Human Skip
```

节省的不只是时间。

还可能包括：

```text
Token
Context
Agent 调用
测试时间
返工
无意义探索
```

所以“每一步都问我”并不天然意味着设计落后。

它付出的是：

> Human Attention。

换来的是：

> **运行时路径选择权。**

---

# 十一、Human 不需要默认逐行看代码

保留 Human Gate 也不意味着：

> 每个 Task 我都必须逐行 Code Review。

否则 Agent 写代码的速度远远超过 Human 读代码的速度。

更合理的方式是从证据开始。

例如：

```text
Task: T07

Goal:
实现 Context Builder

Changed:
3 files

Targeted Tests:
17 PASS

Regression:
324 PASS

Architecture Drift:
NO

Business Rule Change:
NO

Known Limitation:
xxx
```

Human 首先看：

```text
Result
+
Evidence
```

如果已经足够：

```text
CONTINUE
```

如果不够：

```text
Result / Evidence
       ↓
Reviewer Analysis
       ↓
Diff
       ↓
Code
       ↓
Debug
```

逐层向下。

所以 Human Gate 的目标不是证明：

> “我亲眼检查过 AI 写的每一行代码。”

而是：

> **我拥有足够证据决定项目是否应该继续。**

---

# 十二、什么时候我会自己看代码？

当代码本身成为判断所需证据时。

例如：

```text
业务规则变化

架构边界变化

State Transition

恢复 / 幂等

权限

删除 / 覆盖

外部副作用

测试无法解释的异常

Worker 与 Reviewer 判断冲突

我自己对结果产生怀疑
```

这时候：

```text
Evidence 不够
↓
Human Drill Down
↓
Diff
↓
Code
```

所以：

> **看不看代码也是 Human 的运行时决策。**

而不是固定仪式。

---

# 十三、原来的 Gate 很严格，但今天我不会全部写死

第一次项目里的 BUILD 对工程流程要求非常严格：

```text
Plan Gate

Scope Gate

Review Gate

Git Gate

Test Gate

Documentation Gate

Log Gate

Regression Gate
```

这些 Gate 帮助我第一次真正把一个 AI 项目收口成工程。

所以它们非常有价值。

但如果今天重新使用第一种开发方式，我不会把所有 Gate 都理解成：

> 每次必须完整执行。

而更愿意理解成：

```text
Available Gates

Plan
Scope
Test
Review
Regression
Git
Documentation
Log
        │
        ▼
Human 根据当前状态
决定执行深度
```

当然，真正危险或者不可逆的动作仍然应该有硬边界。

但普通施工过程没有必要为了流程完整而执行没有价值的步骤。

---

# 十四、这套模式真正控制的不是 Task，而是路径

一开始我以为：

> Human 控制的是 Task 生命周期。

后来发现还不够准确。

Human 实际控制：

```text
这个 Task 做不做？
↓
这个 Plan 怎么走？
↓
实现范围多大？
↓
要不要测试？
↓
测试多深？
↓
要不要 Review？
↓
我要不要自己看代码？
↓
这个结果够不够？
↓
要不要进入下一 Task？
```

所以第一代真正的运行模型应该是：

```text
Agent
↓
提出下一动作
↓
Human 判断
├── RUN
├── SKIP
├── CHANGE
└── STOP
↓
Agent 执行
↓
产生新证据
↓
再次提出下一动作
↓
Human 再判断
```

因此我更愿意把它叫做：

> **Human-Controlled Interactive Development**

Agent 负责工作。

Human 控制路径。

---

# 十五、为什么这种模式特别适合探索性项目？

因为有些项目在开始的时候，下一步本来就是未知的。

例如：

```text
最初理解
↓
Task 1
↓
得到新信息
↓
修改原来的认识
↓
Task 2
↓
发现新问题
↓
重新调整
↓
Task 3
```

这时候如果提前要求：

> 把整个流程冻结，然后让 Runtime 自动执行。

可能反而是在强迫系统沿着一个还没有被证明正确的路径继续走。

而 Interactive Development 允许：

> **设计和施工交替演化。**

每完成一步，Human 都重新获得一次决策机会。

所以这里的停顿不是浪费。

> **停顿本身就是开发过程的一部分。**

---

# 十六、第一代真正的问题：Human 也是 Runtime

当然，这种模式有非常明显的成本。

整个系统实际上是：

```text
                  Human
                    │
              Select Task
                    │
                    ▼
                  Worker
                    │
                  Plan
                    │
                    ▼
                  Human
                    │
                 Approve
                    │
                    ▼
                  Worker
                    │
             Implement / Test
                    │
                    ▼
                  Human
                    │
            Accept / Change
                    │
                    ▼
                Next Task
```

也就是说：

> **Human 本身就是 Runtime。**

如果我离开：

```text
Worker 完成
↓
等待
↓
……
```

项目也会停下来。

所以它无法很好地无人值守运行。

Human Attention 也成为持续成本。

---

# 十七、什么时候这个缺点开始真正成为问题？

关键并不是项目“大不大”。

而是：

> **Human 每一次被叫回来时，还有没有提供新的判断？**

假设 Agent 连续十次问：

```text
Plan 是否执行？
```

我每次都：

> Yes。

又连续十次问：

```text
Targeted Test 是否运行？
```

我还是：

> Yes。

Review PASS 以后：

```text
是否进入下一 Task？
```

答案永远：

> Continue。

这时候 Human Gate 已经没有提供新的信息。

它开始退化成：

```text
Agent
↓
等待 Human 点 Yes
↓
Agent
↓
等待 Human 点 Yes
↓
Agent
↓
等待 Human 点 Continue
```

这才是真正值得自动化的信号。

---

# 十八、什么时候我今天仍然会选择第一代？

如果一个项目：

```text
路径还不确定

每个 Task 都可能改变后面的设计

我希望跟着施工一起理解项目

下一步高度依赖刚得到的结果

我经常会拒绝 Agent 的 Plan

我经常会跳过某些 Test / Review

Token / Runtime 成本需要实时控制

项目不需要长时间无人值守
```

我会主动选择：

> **Human-Controlled Interactive Development。**

因为这时候 Human Gate 不是瓶颈。

Human Gate 本身就在产生价值。

反过来，如果：

```text
设计已经稳定

Task 明确

Acceptance 明确

正常 Transition 高度重复

Human 基本只是在点击 Continue
```

那么这些 Gate 才开始获得下沉给 Runtime 的资格。

---

# 十九、这不是第一代淘汰第二代，也不是第二代淘汰第一代

后来我尝试了更自治的开发方式。

Human 可以提前定义：

```text
什么时候 Review

什么时候 Test

什么时候 Retry

什么时候 Stop

什么情况 Human Required
```

然后 Runtime 自己执行正常流程。

这同样非常有价值。

而且自治程度高也不代表 Human 失去控制。

因为：

> **协议仍然是 Human 写的。**

Human 仍然可以规定：

```text
Review 只在某些条件触发

Full Test 只在阶段结束执行

某些操作必须 Human 主动发起

某些异常必须立即停止

某些 Task 不允许自动推进
```

所以第一代和后来的自治 Runtime 不是：

```text
Human Control
VS
No Human Control
```

而更像：

```text
第一代

Human 在运行过程中
持续注入决策
```

与：

```text
第二 / 第三代

Human 把更多决策
提前写进 Protocol
↓
Runtime 根据协议自治
```

它们只是把 Human Decision 放在了不同的位置。

---

# 二十、一个判断：这个 Gate 应不应该自动化？

现在如果让我重新判断一个 Human Gate 是否应该保留，我会问：

> **如果 Agent 每次问我，我的答案都可能不同吗？**

如果答案是：

> 会。

那就保留。

因为 Human 仍然在提供信息。

如果：

> 它问我十次，我十次都会给出相同答案。

那就应该开始问：

> **为什么还要问？**

这个 Gate 已经获得了下沉给 Runtime 的资格。

这也是后来我开始尝试更高自治开发方式的原因。

---

# 二十一、第一代完整运行模型

最终可以把这种开发方式压缩成：

```text
             Project Docs
                  │
                  ▼
               Current Task
                  │
                  ▼
                Worker
                  │
              Propose Plan
                  │
                  ▼
                Human
        ┌─────────┼─────────┐
        ▼         ▼         ▼
       RUN      CHANGE      STOP
        │
        ▼
      Worker
        │
     Implement
        │
        ▼
   Result + Evidence
        │
        ▼
      Human
   ┌────┼────┬───────┐
   ▼    ▼    ▼       ▼
 TEST REVIEW SKIP   DRILL DOWN
   │    │             │
   └────┴──────┬──────┘
               ▼
             Human
        ┌──────┼──────┐
        ▼      ▼      ▼
      NEXT    FIX    STOP
```

其中：

```text
Agent
→ 提出路径
→ 执行路径
→ 提供证据

Human
→ 选择路径
→ 修改路径
→ 决定执行深度
→ 决定是否继续
```

---

# 二十二、给其他 AI：如何生成这种开发模式

如果以后我要重新采用第一代方式，可以直接把下面的协议交给 AI。

```text
我要使用一种：

HUMAN_CONTROLLED_INTERACTIVE_DEVELOPMENT

开发模式。

核心原则：

Agent 负责分析、提出下一步方案和执行；
Human 保留关键分叉点的路径选择权。

这不是 Autonomous Runtime。

不要自动连续执行整个 TASKS。
不要因为能够自动化就主动增加 Runtime、Queue、
Watcher、Graph 或其他自治机制。

一、项目资料

优先建立或读取：

PRODUCT.md
SYSTEM.md
DECISIONS.md
TASKS.md
BUILD.md
CHANGELOG.md

Project Files 是 Source of Truth。
Chat History 不是项目状态。

二、角色

至少区分：

1. Human Owner

负责：

- 产品目标
- 业务规则
- 架构关键取舍
- Scope
- 是否批准 Plan
- 验证深度
- 是否进入下一 Task
- 最终责任

2. Design / Review Assistant（可选）

负责：

- 帮助维护设计文档
- 帮助拆 Task
- Review Plan
- Review Diff / Test / Evidence
- 检查 Architecture Drift
- 帮助 Human 理解问题

它是 Human 的工具。
它不拥有项目推进权。

3. Worker

负责：

- 读取项目资料
- 读取当前 Task
- 分析相关代码
- 提出 Plan
- 获得批准后实现
- 运行被批准或协议要求的验证
- 输出 Evidence
- 根据 Human 决策修复
- 在批准后 Commit / 记录

三、Task 执行协议

每次只处理一个 Current Task。

开始时：

Read Docs
→ Read Relevant Code
→ Understand Task
→ Generate Plan
→ STOP

Plan 至少说明：

- Current Task
- Requirement Understanding
- Files To Change
- Implementation Steps
- Scope
- Risk
- Suggested Tests
- 是否触碰 PRODUCT / SYSTEM / DECISIONS

在 Human 明确批准前：

不得修改代码。

Human 可以：

RUN
CHANGE
SKIP
STOP

四、执行阶段

Plan 获批后，只在批准 Scope 内施工。

发现额外问题时：

记录并报告。

不要顺手扩展 Scope。
不要顺手重构整个项目。
不要自动增加新的架构复杂度。

需要改变：

PRODUCT
SYSTEM
DECISIONS
业务规则
关键依赖
重要边界

时，STOP 并返回 Human。

五、完成阶段

实现后返回最小但足够的 Evidence：

- Changed Files
- Diff Summary
- Acceptance Status
- Tests Already Run
- Suggested Additional Tests
- Known Limitations
- Architecture Drift
- Business Rule Change
- Suggested Next Action

然后等待 Human。

不要默认自动进入下一 Task。

六、验证深度由 Human 决定

Human 可以选择：

RUN TEST
SKIP TEST
TARGETED TEST
FULL REGRESSION
AGENT REVIEW
HUMAN DIFF REVIEW
HUMAN CODE REVIEW
FIX
NEXT
STOP

不要假设每个 Task 都需要相同验证深度。

如果建议执行 Test / Review，请同时说明：

为什么建议执行；
不执行的主要风险是什么。

让 Human 根据当前目标、成本和风险选择。

七、Review 原则

Human 不需要默认逐行阅读代码。

默认从：

Result
+
Evidence

开始。

证据不足时再逐层下钻：

Result / Evidence
→ Reviewer Analysis
→ Diff
→ Code
→ Debug

Human 的目标不是证明自己检查了每一行代码。

Human 的目标是获得足够证据决定项目是否应该继续。

八、项目推进权

Worker 完成当前工作后不得自动领取下一 Task。

必须等待 Human：

NEXT
FIX
CHANGE
STOP

只有 Human 拥有项目推进权。

九、复杂度原则

复杂度必须获得存在资格。

自治也必须获得存在资格。

如果某个 Human Gate 仍然经常产生不同决策，
保留它。

如果同类 Gate 反复出现，
而 Human 几乎总是给出相同答案，
记录这个信号。

它可能已经适合在下一阶段下沉给 Runtime。

十、最终目标

这套模式追求的不是最大自动化。

它追求：

Agent 承担施工能力，
Human 保留有价值的运行时判断。

Agent 每到重要分叉点，
把选择权还给 Human。
```

如果最终生成的开发流程变成：

```text
Human
↓
START
↓
Agent 自动完成全部 TASKS
↓
Human
```

说明已经偏离了这种模式。

正确形态应该始终保留：

```text
Agent
↓
Propose
↓
Human Decide
↓
Agent Execute
↓
Evidence
↓
Human Decide
↓
Next
```

---

# 二十三、后来为什么还要继续进化？

第一代已经能够很好地完成项目。

真正推动我继续改变它的，并不是：

> 它不能工作。

恰恰相反。

是因为它工作以后，我开始发现：

```text
有些 Plan
我总是批准。

有些 Test
我总是执行。

有些 Review PASS
我总是进入 Next Task。

有些 Transition
已经不再需要我真正思考。
```

如果 Human 每一次出现都在提供新的判断，那么 Human 应该留下。

但如果 Human 只是机械地重复：

> Yes。  
> Continue。  
> Run。  
> Next。

那么问题就变成了：

> **这些已经稳定下来的判断，为什么不能提前写进协议？**

于是我的第二次尝试开始改变一个东西：

不是让 Agent 写更多代码。

而是尝试把那些已经不再需要 Human 临场判断的控制权，逐渐交给 Runtime。

第一代解决的是：

> **怎么让 Agent 在 Human 控制下完成一个真实项目。**

而接下来的问题变成了：

> **哪些控制权应该继续属于 Human，哪些已经获得了下沉的资格？**

这成为下一阶段的起点。