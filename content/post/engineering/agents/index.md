---
description: ""
title: "第二次 Vibe Coding：把 Human 从施工循环里移出去"
draft: false
date: "2026-09-27T05:39:46+08:00"
slug: "VibeCodingOne"
categories:
 - VibeCoding
tags:
 - Note
image: ""
---

# 第二次 Vibe Coding：把 Human 从施工循环里移出去

第一次 Vibe Coding 时，我已经能够把一个完整项目拆成：

```text
PRODUCT
SYSTEM
DECISIONS
TASKS
BUILD
```

然后：

```text
Human
↓
启动 Task
↓
Agent 完成
↓
Human
↓
下一 Task
```

这种方式很好理解，也很好控制。

但它有一个非常明显的特点：

> **项目能不能继续运行，取决于 Human 有没有回来。**

后来开始做企业知识库项目时，我想尝试另一种开发方式：

> 如果产品、系统、关键决策和任务已经基本确定，能不能让 Agent 自己完成正常施工循环？

于是第二次 Vibe Coding 的核心问题从：

> 怎么让 Agent 完成一个 Task？

变成了：

> **怎么让一个项目在 Human 不持续调度的情况下继续施工？**

---

# 一、Human 先退出施工，而不是退出项目

我并不是想让 AI 自己决定整个产品。

知识库项目开始施工之前，Human 仍然负责：

```text
问题定义
↓
PRODUCT
↓
SYSTEM
↓
关键 DECISIONS
↓
TASKS
↓
BUILD
```

尤其是：

```text
产品到底解决什么问题？
LLM 应该做什么？
LLM 不应该做什么？
知识如何检索？
什么必须保留证据？
什么判断必须留给 Human？
哪些设计已经冻结？
```

这些东西仍然需要先确定。

所以 Human 并没有退出。

Human 只是从：

```text
施工过程的每一次状态转换
```

里退出。

整个结构开始变成：

```text
              Human
                │
        Product / System
        Decisions / Tasks
                │
                ▼
         Frozen Design
                │
                ▼
        Agent Runtime
                │
          Autonomous Build
                │
                ▼
             Product
                │
                ▼
              Human
```

Human 控制：

> **设计和边界。**

Runtime 控制：

> **已经确定范围内的施工。**

---

# 二、为什么一个 Agent 不够？

如果仍然把所有事情交给同一个 Agent：

```text
读取 Task
↓
写代码
↓
测试
↓
判断自己是否正确
↓
宣布 PASS
↓
进入下一 Task
```

会出现一个很明显的问题：

> **写代码的人同时拥有最终验收权。**

它可能：

```text
误解 Task
↓
按照自己的误解实现
↓
再按照同一个误解验证
↓
认为自己完成了
```

所以我开始把不同责任拆开。

最基础的是：

```text
Control
Implementation
Verification
```

在当时的项目里，它们可以表现为：

```text
Orchestrator / Lead
Worker / Builder
Reviewer
```

名称不是最重要的。

真正重要的是职责分离：

```text
Control
→ 现在应该做什么？

Worker
→ 把当前任务实现出来。

Reviewer
→ 它真的符合 Acceptance 吗？
```

于是：

```text
        Control
           │
           ▼
         Worker
           │
           ▼
        Reviewer
        ↙      ↘
      FAIL     PASS
       │         │
       ▼         ▼
     Rework    Next Task
```

这时候 Human 第一次有可能退出正常施工循环。

---

# 三、多个 Agent 不能靠聊天记住项目

角色拆开以后，一个新的问题马上出现：

> Worker 做完以后，Reviewer 怎么知道发生了什么？

如果靠 Human：

```text
Worker
↓
告诉 Human
↓
Human 复制给 Reviewer
↓
Reviewer
```

那 Human 只是从：

> Task 调度器

变成了：

> Agent 消息搬运工。

这没有解决问题。

所以 Agent 之间必须共享同一个外部世界。

这个外部世界就是：

```text
Shared Project
+
Markdown Contract
+
Shared State
```

---

# 四、Shared Project：真正发生了什么

代码、测试、配置、文档都存在同一个项目目录。

例如：

```text
src/
tests/

PRODUCT.md
SYSTEM.md
DECISIONS.md
TASKS.md
BUILD.md
```

Worker 修改：

```text
src/retrieval.py
```

Reviewer 不需要 Worker 把整个文件重新复制给它。

Reviewer 自己读取真实项目即可。

所以：

> **Project Files 是事实。**

Agent 的描述不是事实本身。

---

# 五、Markdown Contract：大家应该怎么协作

只有共享项目还不够。

Agent 还需要知道：

```text
我是谁？
我负责什么？
我不能做什么？
现在应该读取什么？
完成以后交给谁？
什么时候必须停下来？
什么时候需要 Human？
```

所以协作规则也开始从聊天中移出去。

例如：

```text
.agents/
├── COMMON.md
├── RUNTIME.md
└── roles/
    ├── CONTROL.md
    ├── WORKER.md
    └── REVIEWER.md
```

这些文件不是项目业务事实。

它们定义的是：

> **Agent 如何围绕这个项目协作。**

于是：

```text
Project
= 做什么

Contract
= 怎么协作
```

开始被分开。

---

# 六、Shared State：现在运行到哪里

即使所有 Agent 都知道规则，还需要知道：

> 当前项目现在在哪里？

所以出现 Shared State。

例如：

```json
{
  "current_task": "T007",
  "current_role": "WORKER",
  "status": "RUNNING",
  "review_attempt": 1,
  "human_required": false,
  "pause_requested": false
}
```

它回答：

```text
当前 Task 是什么？
现在轮到谁？
刚刚发生了什么？
Review 到第几轮？
是否需要 Human？
```

这样新的 Agent 被启动以后，不需要依赖旧 Session 的聊天历史。

它可以：

```text
读取 Contract
↓
读取 State
↓
读取 Current Task
↓
读取真实 Project
↓
恢复当前工作
```

这时候 Conversation 第一次真正从 Runtime State 里被剥离出去。

---

# 七、Project、Contract、State 开始各自负责一件事

整个结构逐渐稳定成：

```text
Shared Project
│
│ 保存真实工作结果
│
├── Code
├── Tests
└── Docs


Markdown Contract
│
│ 保存协作规则
│
├── Common Rules
├── Runtime Rules
└── Role Rules


Shared State
│
│ 保存当前运行现场
│
├── Current Task
├── Current Role
├── Status
├── Retry
└── Human Required
```

于是三个问题被分开：

```text
发生了什么？
→ Project

应该怎么协作？
→ Contract

现在运行到哪里？
→ State
```

这比把所有东西都塞进聊天历史稳定得多。

---

# 八、Handoff 只需要告诉下一角色去哪里看

Agent 之间仍然需要交接。

但既然大家共享 Project，就没有必要复制完整工作内容。

例如 Worker 完成：

```text
Task: T007
Status: WORK_DONE

Changed:
- src/retrieval.py
- tests/test_retrieval.py

Verification:
- targeted tests PASS

Attention:
- semantic fallback branch
```

Reviewer 看到以后：

```text
知道改了哪里
↓
自己读真实文件
↓
自己看 Diff
↓
自己运行必要验证
```

所以 Handoff 更像：

> **导航卡。**

而不是：

> 工作结果的副本。

可以简单概括为：

> **Project 保存事实，Handoff 保存指针。**

---

# 九、正常状态转换开始从 Human 手里移出去

第一代：

```text
Worker 完成
↓
Human
↓
Reviewer

Reviewer PASS
↓
Human
↓
Next Task
```

第二代希望变成：

```text
Worker 完成
↓
State = WORK_DONE
↓
Reviewer

Reviewer PASS
↓
State = REVIEW_PASS
↓
Next Task
```

也就是说：

> **正常、重复、已经有明确规则的 Transition，不再需要 Human 每次重新判断。**

Human 不应该一直回答：

```text
可以 Review 了。
可以返工了。
可以进入下一 Task 了。
```

这些如果已经是项目协议的一部分，就应该由 Runtime 自己完成。

---

# 十、Human 的位置开始发生变化

于是 Human 从：

```text
Task Scheduler
```

逐渐变成：

```text
Product Owner
Architecture Owner
Boundary Owner
Exception Handler
```

正常施工：

```text
Task
↓
Worker
↓
Reviewer
↓
Next Task
```

Human 不进入。

只有出现：

```text
需求不明确
设计冲突
架构问题
无法收敛
需要改变冻结决策
高风险动作
```

才重新把控制权交回来。

所以目标不是：

> Human 不参与。

而是：

> **Human 不参与已经能够由规则处理的正常状态转换。**

---

# 十一、Human Control 变成 Runtime Control

Human 仍然需要拥有最终控制权。

但控制方式不应该是不断进入内部施工。

更合理的是：

```text
START
PAUSE
RESUME
STOP
OVERRIDE
```

例如：

> 当前工作完成以后暂停。

Runtime 记录：

```text
pause_requested = true
```

当前工作完成以后：

```text
检查 Human Control
↓
PAUSE
↓
不领取下一 Task
```

Human 控制的是：

> **Runtime 生命周期。**

而不是：

> 每一次内部 Action。

---

# 十二、Review 也不能无限循环

自治以后，一个新的风险是：

```text
Worker
↓
Reviewer FAIL
↓
Worker
↓
Reviewer FAIL
↓
Worker
↓
Reviewer FAIL
↓
...
```

如果 Human 不再每轮介入，就必须给自治设置边界。

例如：

```text
review_attempt = 3
max_review_attempts = 3
```

达到上限：

```text
HUMAN_REQUIRED
↓
STOP
```

因为连续失败本身已经产生了新信息。

问题可能已经不是：

> 再改一次代码。

而可能是：

```text
Task 定义错误
Acceptance 冲突
架构理解不同
Reviewer 标准不合理
当前方案无法实现
```

所以：

> **自治必须有边界。**

---

# 十三、测试也应该进入施工协议

第一代里 Human 很容易不断说：

> 跑一下测试。

但如果施工阶段已经自治，测试什么时候运行也应该提前确定。

例如：

```text
普通修改
→ Targeted Test

共享模块变化
→ Impact Regression

阶段完成
→ Integration Test

最终完成
→ Full Regression
```

这样 Worker 和 Reviewer 不需要每次询问 Human：

> 现在要不要跑测试？

测试成为 Runtime Contract 的一部分。

---

# 十四、第二代真正改变的是控制权

如果把第一代和第二代放在一起：

```text
第一代

Human
↓
Task
↓
Agent
↓
Human
↓
Next Task
```

第二代：

```text
Human
↓
Frozen Design
↓
Runtime
├── Control
├── Worker
└── Reviewer
↓
Product
↓
Human
```

真正变化的不是：

> 一个 Agent 变成了三个 Agent。

而是：

> **Human 把一部分正常控制权交给了 Runtime。**

这也是我第一次真正开始理解：

```text
Implementation
```

和：

```text
Runtime Control
```

并不是同一件事。

---

# 十五、自动化并不是免费的

第二代最大的好处很明显：

> Human 不需要一直守着项目。

但是实际运行以后，我也开始感受到它的另一面。

第一代路径通常非常清楚：

```text
T01
↓
Human
↓
T02
↓
Human
↓
T03
```

我几乎始终知道系统正在做什么。

第二代变成：

```text
Control
↓
读取 State
↓
Worker
↓
读取 Project / Contract
↓
实现
↓
测试
↓
Reviewer
↓
再次读取 Project / Contract
↓
验证
↓
可能返工
↓
Control
↓
Next Task
```

Human 操作减少了。

但内部步骤增加了。

每个 Agent 都需要重新建立上下文。

每次 Review、Replan、Retry 都需要额外推理。

于是一个非常现实的问题出现：

> **自治减少了 Human Attention，却增加了 Runtime Cost。**

最直接的表现就是 Token 消耗明显增加。

---

# 十六、路径也开始变得不可见

第一代里：

```text
我启动 T04
```

所以我天然知道：

> 现在正在做 T04。

第二代里 Human 可能只看到：

```text
START
```

然后 Runtime 内部：

```text
Task
↓
Worker
↓
Reviewer
↓
Retry
↓
Reviewer
↓
Next Task
↓
Worker
...
```

如果不主动查看 State 和 Trace，我并不知道它具体运行到了哪里。

所以自治带来的交换关系开始变得明显：

```text
Human 操作减少
        ↓
Automation 增加

但同时：

Path Visibility 下降
Runtime Cost 上升
Token Predictability 下降
Debug Difficulty 上升
```

这并不说明自治错误。

只是说明：

> **自动化本身也有成本。**

---

# 十七、第一代因此没有失效

做到这里以后，我反而重新理解了第一代。

如果：

```text
任务明确
路径明确
项目不长
Human 希望理解每一步
Token 成本敏感
```

那么：

```text
Human-Controlled TaskList
```

可能反而更加简单。

如果：

```text
项目很长
Task 很多
正常 Transition 高度重复
Human 调度已经成为主要成本
```

那么才值得把更多控制权交给 Runtime。

所以：

```text
第一代
≠ 落后

第二代
≠ 更高级
```

它们交换的是不同的成本。

第一代支付：

```text
Human Attention
```

换取：

```text
Visibility
Predictability
Control
```

第二代支付更多：

```text
Token
Runtime Complexity
Observability Cost
```

换取：

```text
Autonomy
Human Attention
Long-running Capability
```

我开始意识到：

> **自治也必须获得存在资格。**

---

# 十八、但第二代还暴露出了一个更具体的问题

到这里，逻辑上的 Runtime 已经基本成立。

State 可以告诉系统：

```text
current_role = WORKER
```

Worker 完成以后可以写：

```text
next_role = REVIEWER
```

控制逻辑也可以知道：

```text
WORK_DONE
→ REVIEWER
```

但是实际运行时，我发现这里隐藏着一个一直没有被区分的问题：

> **知道下一步是谁，不等于能够把下一步的人叫起来。**

如果 Worker 和 Reviewer 是两个彼此独立的 AI Session：

```text
Worker Session

Reviewer Session
```

那么 Worker 即使知道：

```text
next = REVIEWER
```

Reviewer 也不会因此自动开始运行。

逻辑世界里：

```text
Worker
↓
Reviewer
```

物理世界里却仍然可能是：

```text
Worker 完成
↓
等待
↓
Human 打开 Reviewer Session
↓
Reviewer 才真正开始
```

---

# 十九、如果强行回到 Control，也会出现新的成本

一种很自然的想法是：

```text
Control
↓
Worker
↓
Control
↓
Reviewer
↓
Control
↓
Next
```

逻辑上非常漂亮。

但如果：

```text
Control
Worker
Reviewer
```

本身就是三个独立 Session，那么 Human 实际操作会变成：

```text
打开 Control
↓
打开 Worker
↓
打开 Control
↓
打开 Reviewer
↓
打开 Control
```

为了让逻辑上的控制中心始终存在，反而增加了物理 Session 切换。

这时候我才发现：

> **我一直把“决定下一步”和“真正启动下一步”当成了同一件事。**

其实它们是两个问题。

---

# 二十、第二次实践停在这里

第二代解决了第一代暴露的问题：

```text
Human 每个 Task 都要回来
```

于是我建立了：

```text
Shared Project
+
Markdown Contract
+
Shared State
+
Control / Worker / Reviewer
+
Bounded Review
+
Human Runtime Control
```

Human 开始退出正常施工循环。

但是实际运行又暴露出了新的边界：

```text
Logical Transition
≠
Physical Wake-up
```

State 可以知道：

> 下一步是 Reviewer。

Runtime Contract 可以规定：

> Worker 完成后进入 Review。

但如果 AI 软件本身没有跨 Session 启动能力：

> **谁真正把 Reviewer 叫起来？**

继续解决这个问题当然可以加入：

```text
CLI
API
Queue
Polling
Process Manager
Desktop Automation
```

但这样又会产生新的复杂度。

而如果当前 AI 软件本身支持：

```text
Host
+
SubAgent
```

问题又完全不同。

于是我开始意识到：

> 也许问题不是继续寻找一种“最强的 Multi-Agent Runtime”。

真正应该拆开的，是：

```text
协作协议
↓
控制逻辑
↓
物理执行方式
```

同一套协作协议，在不同 AI 环境下，也许本来就应该采用不同的执行方式。

这成为第三次重新设计 Vibe Coding 方法的起点。