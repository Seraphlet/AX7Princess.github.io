---
description: ""
title: "Vibe Coding：协作协议不变，执行方式可以不同"
draft: false
date: "2026-10-03T09:06:35+08:00"
slug: "VibeCodingThree"
categories:
 - 
tags:
 - null
image: ""
---

# 第三次重新理解 Vibe Coding：协作协议不变，执行方式可以不同

经过前面的实践，我现在更倾向于把 Vibe Coding 中的多 Agent 协作拆成几个彼此独立的问题：

```text
Shared Project
     │
     │ 真实工作结果
     ▼
Markdown Contract
     │
     │ 协作规则
     ▼
Shared State
     │
     │ 当前运行事实
     ▼
Execution Mode
     │
     │ Agent 如何真正被启动
     ▼
Agent
```

真正稳定下来的不是某一种固定的 Agent 拓扑，而是：

```text
Shared Project
+
Markdown Contract
+
Shared State
```

在这个基础上，再根据当前项目决定需要哪些角色，根据当前 AI 软件的能力选择 Execution Mode。

---

# 一、Shared Project：真实工作结果

所有 Agent 共同操作同一个项目目录。

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

代码、测试、配置、文档都以真实项目文件为准。

所以：

> **Project Files 是工作结果的 Source of Truth。**

Agent 之间不需要复制完整代码，也不需要把已经存在于项目里的内容重新描述一遍。

---

# 二、Markdown Contract：Agent 之间的协作制度

协作规则保存在项目中，例如：

```text
project/
│
├── PRODUCT.md
├── SYSTEM.md
├── DECISIONS.md
├── TASKS.md
├── BUILD.md
│
├── .agents/
│   ├── COMMON.md
│   ├── RUNTIME.md
│   └── roles/
│       └── <根据项目动态生成>
│
└── .runtime/
    ├── state.json
    └── handoff.md
```

其中：

```text
.agents/
= 应该怎么协作

.runtime/
= 现在协作到哪里

项目文件
= 实际做出了什么
```

`COMMON.md` 保存所有角色共同遵守的规则。

`RUNTIME.md` 保存当前项目的运行协议。

`roles/` 保存当前项目真正需要的角色职责。

---

# 三、角色由项目产生，而不是提前写死

一个项目可能只需要：

```text
Worker
Reviewer
```

另一个项目可能需要：

```text
Researcher
Writer
FactChecker
```

也可能只有一个主要执行 Agent。

所以：

> **Role Set 不是架构常量。**

生成协作契约之前，应该先：

```text
读取项目
↓
理解 PRODUCT / SYSTEM / DECISIONS
↓
理解任务和代码结构
↓
寻找真正不同的职责
↓
判断是否值得拆成独立角色
↓
生成 Role Contract
```

每增加一个 Agent，都应该能够回答：

> 为什么这个职责不能由现有角色、确定性程序或者 Human 更简单地完成？

Agent 本身也是复杂度。

所以：

> **每一个 Agent 都必须获得存在资格。**

---

# 四、每次启动都重新读取通信协议

不管采用哪一种 Execution Mode，执行 Agent 每次：

```text
START
RESUME
CONTINUE
```

都必须重新建立当前上下文。

标准启动过程：

```text
Wake Up
↓
COMMON.md
↓
RUNTIME.md
↓
Own Role Contract
↓
state.json
↓
Role Guard
↓
handoff.md
↓
Current Task
↓
必要的项目文件
↓
Work
```

即使还是同一个 Session，也不应该把聊天历史当成可靠运行状态。

因为真正跨 Agent 共享的是：

```text
Project
+
Contract
+
State
```

而不是 Conversation。

---

# 五、Role Guard：先确认现在是不是轮到自己

Agent 被唤醒以后，不能直接工作。

首先检查：

```text
my_role == expected_role ?
```

如果 State 表示：

```text
expected_role = REVIEWER
```

但当前打开的是：

```text
WORKER
```

那么 Worker 必须：

```text
REFUSE
↓
不执行 Task
↓
不修改项目
↓
报告 Role Mismatch
↓
STOP
```

例如：

```text
Expected Role: REVIEWER
Current Role: WORKER
Task: T007
Action: REFUSED
Project Modified: NO
```

这样即使 Human 打开了错误的 Session，错误也不会继续向项目内部传播。

---

# 六、Handoff 是导航卡，不是第二份项目

Agent 完成工作以后，只留下下一角色真正需要知道的信息。

例如：

```text
# Handoff

Task: T007
From: WORKER
Status: WORK_DONE

Changed:
- src/runtime.py
- tests/test_runtime.py

Verification:
- pytest tests/test_runtime.py
- PASS

Attention:
- pause_requested 后不能领取下一 Task
```

Reviewer 读取 Handoff 后，再自己查看：

```text
真实代码
Git Diff
测试
相关文档
```

而不是相信 Worker 对自己工作的完整描述。

所以：

> **Project 保存事实，Handoff 负责导航。**

或者更简单：

> **Result is Pointer, not Copy.**

---

# 七、State 只保存运行现场

`state.json` 不应该逐渐变成第二个数据库。

例如：

```json
{
  "execution_mode": "CHAINED_MULTI_SESSION",

  "current_task": "T007",
  "current_role": "WORKER",
  "expected_role": "REVIEWER",

  "status": "RUNNING",
  "last_event": "WORK_DONE",

  "review_attempt": 1,
  "max_review_attempts": 3,

  "handoff": ".runtime/handoff.md",

  "human_required": false,
  "pause_requested": false
}
```

它只需要回答：

```text
现在是什么 Task？
运行到哪里？
刚刚发生了什么？
下一步轮到谁？
Review 到第几轮？
是否暂停？
是否需要 Human？
```

代码、测试结果、完整 Review、业务事实仍然留在它们真正应该存在的位置。

---

# 八、记录存在，但记录本身也是资源

系统需要能够回答：

```text
哪一步失败？
为什么失败？
失败过几次？
最后为什么停止？
```

但没有必要保存所有 Agent 的完整过程。

例如 Review History：

```text
#1 FAIL — PAUSE 未持久化
#2 FAIL — PAUSE 后仍领取 Next Task
#3 PASS
```

已经足够用于定位问题。

所以 Trace 的原则是：

> **记录足以还原异常的信息，而不是记录一切。**

如果一条记录不会帮助：

```text
定位
恢复
审计
判断下一步
```

那么它很可能没有保存价值。

---

# 九、Review 必须是有界循环

Worker 和 Reviewer 之间不能无限修改。

例如：

```text
Worker
↓
Reviewer
↓ FAIL #1
Worker
↓
Reviewer
↓ FAIL #2
Worker
↓
Reviewer
↓ FAIL #3
```

State 中保存：

```text
review_attempt
max_review_attempts
```

例如：

```text
review_attempt = 3
max_review_attempts = 3
```

达到上限：

```text
HUMAN_REQUIRED
↓
留下最后失败原因
↓
STOP
```

因为连续多轮仍然无法收敛，本身已经说明问题可能发生了变化。

可能不再只是：

> 代码还需要修改。

而可能涉及：

```text
Task 定义
Acceptance
架构边界
Reviewer 标准
实现能力
```

这时候继续自动循环只会继续消耗资源。

所以：

> **自动修复必须有界，无法收敛时控制权重新上浮给 Human。**

---

# 十、Watcher：调用链之外的一根保险丝

Watcher 不参与项目执行流程。

它不属于：

```text
Worker → Reviewer → Next
```

也不属于：

```text
Control → Worker → Control → Reviewer
```

它只在旁边观察 Runtime State：

```text
        Main Workflow
             │
             ▼
           State
             │
             │ observe
             ▼
          Watcher
             │
          abnormal
             │
             ▼
           Human
```

Watcher 的职责只有一个：

> **发现预先定义的异常状态，通知 Human。**

例如：

```text
review_attempt >= max_review_attempts
```

或者其他已经明确需要观察的 Runtime 异常。

如果判断完全确定，Watcher 可以只是一个持续运行的 Python 脚本。

它不推进任务，也不进入 Agent 调用链。

默认：

```text
Observe
↓
Detect
↓
Notify Human
```

就够了。

是否启动 Watcher，由 Human 决定。

所以它更像一根独立的保险丝：

> **主流程负责运行，Watcher 只负责在异常出现时把 Human 叫回来。**

---

# 十一、两种 Execution Mode

同一套：

```text
Shared Project
Markdown Contract
Shared State
Role Guard
Minimal Handoff
Bounded Review
```

可以运行在不同的执行环境中。

目前我保留两种模式：

```text
CHAINED_MULTI_SESSION

CONTROLLED_SUBAGENT
```

它们不是版本升级关系。

而是：

> **同一套协作协议在不同物理运行环境下的两种执行方式。**

---

# 十二、Execution Mode A：CHAINED_MULTI_SESSION

当不同 Agent 存在于彼此独立的 Session 时，最自然的执行方式是链式：

```text
A → B → C → D
```

例如：

```text
Worker Session
Reviewer Session
Other Role Session
```

Worker 完成以后：

```text
Worker
↓
修改 Shared Project
↓
写 Minimal Handoff
↓
更新 State
↓
告诉 Human 下一角色
↓
STOP
```

例如：

```text
Task T007 completed.
Next Role: REVIEWER.
```

Human 不需要复制 Worker 的输出。

也不需要向 Reviewer 重新解释发生了什么。

只需要：

```text
打开 Reviewer Session
↓
“继续下一步”
```

Reviewer 自己重新读取：

```text
COMMON
RUNTIME
REVIEWER Role Contract
State
Handoff
Current Task
真实项目文件
```

然后继续工作。

---

# 十三、为什么 Multi Session 适合链式？

如果强制所有角色都返回一个 Control Session：

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

Human 实际需要：

```text
点 Control
↓
点 Worker
↓
点 Control
↓
点 Reviewer
↓
点 Control
```

反而增加了物理操作。

而很多 Transition 本身已经确定：

```text
WORK_DONE
→ REVIEWER
```

所以在独立 Session 环境下，链式交接更加自然。

Human 只承担：

> **Physical Wake-up。**

Agent 完成以后告诉 Human 下一跳是谁。

Human 打开那个 Session。

仅此而已。

---

# 十四、Multi Session 仍然存在物理成本

这种模式没有彻底消灭 Human 的机械操作。

以前可能是：

```text
允许 Task
允许改文件
允许测试
允许命令
允许继续
……
```

现在压缩成：

```text
Agent 完成
↓
告诉我下一跳
↓
我打开下一 Session
↓
“继续”
```

Human 仍然承担 Session Wake-up。

但已经不再承担：

```text
复制结果
解释上下文
搬运代码
重新描述 Task
判断普通 Transition
```

这个成本目前可以接受。

因为如果只是为了消灭一次点击，就加入：

```text
CLI
API
桌面自动化
Queue
Polling
Process Manager
```

那么新增复杂度可能远远大于得到的收益。

---

# 十五、Execution Mode B：CONTROLLED_SUBAGENT

如果当前 AI 软件原生支持：

```text
Host
+
SubAgent
```

那么更自然的执行方式是中心回环：

```text
            Human
              │
              ▼
              A
        Control Agent
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
       B      C      D
       │      │      │
       └──► A ◄── A ◄┘
```

运行顺序：

```text
A → B → A → C → A → D → A
```

A 是整个系统的控制中心。

---

# 十六、Control Agent 是 Human 的流程入口

在这种模式下，Human 不需要分别控制 B、C、D。

Human 只需要和 A 交互。

例如：

> 这一步做完以后暂停。

A 把它转换成：

```text
pause_requested = true
```

当前 Worker 完成：

```text
Worker
↓
Return to A
```

A 发现：

```text
pause_requested = true
```

于是：

```text
PAUSED
↓
不再启动下一角色
```

Human 也可以说：

> 跳过 Review。

A 负责：

```text
记录 Human Override
↓
改变正常 Transition
↓
选择下一合法节点
↓
继续
```

Human 不需要自己打开 `state.json` 修改字段。

所以：

> **Human 控制 Control Agent，Control Agent 控制 Runtime。**

---

# 十七、工作 Agent 完成后统一返回 Control

在 `CONTROLLED_SUBAGENT` 中，普通工作 Agent 不负责启动其他工作 Agent。

它只负责：

```text
接收任务
↓
执行自己的职责
↓
修改 Shared Project
↓
写 Minimal Handoff
↓
更新 State
↓
Return to Control
```

Control Agent：

```text
读取 State
↓
读取 Handoff
↓
检查 Runtime Contract
↓
检查 Human Override
↓
决定下一角色
↓
启动下一 SubAgent
```

所以：

> **所有工作 Agent 的默认下一节点都是 Control Agent。**

只有 Control Agent 拥有正常全局调度权。

---

# 十八、两种 Execution Mode 不是谁取代谁

它们应该这样理解：

```text
                   Same Contract
                        │
                   Shared State
                        │
                  Shared Project
                        │
             ┌──────────┴──────────┐
             ▼                     ▼

   CHAINED_MULTI_SESSION   CONTROLLED_SUBAGENT
             │                     │
        独立 Session          Host + SubAgent
             │                     │
        链式流转更自然          中心回环更自然
             │                     │
      Human 负责 Wake-up        A 负责 Wake-up
```

Multi Session：

```text
A → B → C → D
```

减少 Human 在 Session 之间反复切换更加重要。

SubAgent：

```text
A → B → A → C → A → D
```

返回 Control Agent 几乎没有额外物理成本。

因此可以获得统一调度、Human Override 和异常处理能力。

所以：

> **架构不是版本淘汰关系，而是能力适配关系。**

---

# 十九、Execution Mode 与 Watcher 是两个独立维度

Execution Mode 回答：

> Agent 怎么流转？

Watcher 回答：

> 要不要有人在旁边盯 Runtime？

所以可以是：

```text
CHAINED_MULTI_SESSION
+
Watcher ON
```

也可以：

```text
CHAINED_MULTI_SESSION
+
Watcher OFF
```

同样可以：

```text
CONTROLLED_SUBAGENT
+
Watcher ON
```

或者：

```text
CONTROLLED_SUBAGENT
+
Watcher OFF
```

Watcher 不改变主流程拓扑。

它只是旁路观察 State。

---

# 二十、以后由我选择 Execution Mode

以后让 AI 为一个具体项目生成协作文件时，我会明确指定：

```text
Execution Mode:
CHAINED_MULTI_SESSION
```

或者：

```text
Execution Mode:
CONTROLLED_SUBAGENT
```

AI 不替我选择。

它先读取：

```text
PRODUCT
SYSTEM
DECISIONS
TASKS
BUILD
当前代码
当前测试
```

再根据项目本身确定需要哪些 Role。

然后按照我指定的 Execution Mode 生成：

```text
COMMON.md
RUNTIME.md
Role Contracts
state.json
handoff.md
```

如果我需要 Watcher，再额外声明：

```text
Watcher: ON
```

否则默认不需要把 Watcher 加进项目执行链。

---

# 二十一、给未来 AI 的直接任务说明

以后可以把这篇文章和当前项目一起交给 AI：

```text
阅读本文和当前项目资料。

为当前项目生成一套基于：

Markdown Contract
+
Shared Project
+
Shared State

的 Multi-Agent Collaboration Contract。

不要设计通用 Multi-Agent Framework。

先读取当前项目，理解：

- PRODUCT
- SYSTEM
- DECISIONS
- TASKS
- BUILD
- 当前代码和测试结构

然后判断当前项目真正需要哪些执行角色。

不要预设 Worker、Reviewer、Orchestrator 等固定角色。

每增加一个 Agent，都要能够回答：

“为什么这个职责不能由现有角色、确定性程序或 Human 更简单地完成？”

我会明确指定 Execution Mode：

A. CHAINED_MULTI_SESSION

或：

B. CONTROLLED_SUBAGENT

不要替我更换 Execution Mode。
```

---

## CHAINED_MULTI_SESSION

如果选择：

```text
Execution Mode:
CHAINED_MULTI_SESSION
```

契约应该按照：

```text
Role A
↓
Role B
↓
Role C
↓
...
```

运行。

每个角色完成以后：

```text
修改真实项目
↓
Minimal Handoff
↓
更新 State
↓
告诉 Human 下一角色
↓
STOP
```

Human 打开下一角色对应的 Session：

> 继续下一步。

Agent 自己重新读取 Contract、State、Handoff 和 Project。

---

## CONTROLLED_SUBAGENT

如果选择：

```text
Execution Mode:
CONTROLLED_SUBAGENT
```

契约应该按照：

```text
Control
↓
Role B
↓
Control
↓
Role C
↓
Control
↓
...
```

运行。

工作 Agent：

```text
完成职责
↓
修改真实项目
↓
Minimal Handoff
↓
更新 State
↓
Return to Control
```

Human 主要与 Control Agent 交互：

```text
START
CONTINUE
PAUSE
RESUME
SKIP
RETRY
OVERRIDE
STOP
```

Control Agent 负责将 Human 意图转换成 Runtime Transition。

---

# 二十二、整个系统的硬约束

无论使用哪一种 Execution Mode：

```text
1. 每次执行 Agent 被唤醒，都重新读取通信协议和 State。

2. Agent 开始工作前必须执行 Role Guard。

3. Role 不匹配时：
   REFUSE
   不修改项目
   报告异常
   STOP / Return Control。

4. Project Files 是工作结果 Source of Truth。

5. Handoff 只保存最小交接信息。

6. State 只保存 Runtime 事实。

7. Review / Retry 必须存在次数上限。

8. 达到上限后：
   HUMAN_REQUIRED
   STOP。

9. Trace 只保存定位和恢复真正需要的信息。

10. Human 始终拥有最终 Override 权。

11. Role 根据当前项目动态产生。

12. 不因为 Multi-Agent 而强行增加 Agent。

13. Watcher 不进入项目执行调用链。

14. 不主动加入当前没有真实需求的 Queue、Polling、
    Heartbeat、跨软件通信等机制。

15. 复杂度必须获得存在资格。
```

---

# 二十三、最终结构

现在整个系统可以压缩成：

```text
                         Human
                           │
                  选择 Execution Mode
                           │
                           ▼
              Markdown Collaboration Contract
                           │
                    Shared State
                           │
                    Shared Project
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ▼                           ▼

 CHAINED_MULTI_SESSION          CONTROLLED_SUBAGENT

      Agent B                         Control A
         │                               │
         ▼                               ▼
      Agent C                          Agent B
         │                               │
         ▼                               ▼
      Agent D                          Control A
         │                               │
         ▼                               ▼
        ...                            Agent C
                                         │
                                         ▼
                                      Control A


                    Shared State
                         │
                    Optional Watcher
                         │
                      abnormal
                         ▼
                       Human
```

这里有三个不同的问题：

```text
Role
→ 谁负责工作？

Execution Mode
→ 工作角色怎么流转？

Watcher
→ Runtime 出现异常时，谁把 Human 叫回来？
```

不要因为它们都和“Agent 系统”有关，就把它们混成同一个调用链。

---

# 二十四、我现在真正保留下来的东西

现在真正稳定下来的，是：

```text
Shared Project
+
Markdown Contract
+
Shared State
+
Minimal Handoff
+
Role Guard
+
Bounded Review
```

根据项目变化的是：

```text
Execution Roles
```

根据运行环境变化的是：

```text
Execution Mode
```

根据我是否需要额外保险变化的是：

```text
Watcher ON / OFF
```

如果使用多个独立 Session：

> **CHAINED_MULTI_SESSION**

如果当前软件能够原生调度 SubAgent：

> **CONTROLLED_SUBAGENT**

如果希望 Runtime 异常时主动把我叫回来：

> **Watcher ON**

否则：

> **Watcher OFF**

我真正想保存下来的不是某个固定的 Agent 调用图，而是一套可以迁移的方法：

> **先根据项目决定谁负责工作，再根据运行环境决定这些角色如何流转；所有执行角色围绕同一个项目、同一套契约和同一个 State 协作；如果需要额外保险，就在调用链之外放一个只负责发现异常并通知 Human 的 Watcher。**

角色可以变。

AI 软件可以变。

执行方式可以变。

但协作契约不需要因此全部重写。

这才是我现在想要的 Vibe Coding Runtime。