---
description: ""
title: "多 Agent 概念 + Supervisor 架构设计"
draft: false
date: "2026-09-14T23:58:08+08:00"
slug: "Supervisor"
categories:
 - Langraph
tags:
 - Supervisor
image: ""
---

# W7-Day1｜多 Agent 概念 + Supervisor 架构设计

> 今天是理论日，不写代码。
>
> 今天真正要理解的不是“Supervisor 的 API 怎么写”，而是：
>
> **为什么需要多 Agent、Agent 应该怎么拆、Supervisor 到底在调度什么、Agent 之间怎么通信，以及它和我之前学过的 Tool Routing 到底是什么关系。**

---

# 一、今天先建立一个总认识：多 Agent 到底是什么？

之前接触 Agent 的时候，我已经知道一种基本结构：

```text
用户
 ↓
Agent
 ↓
判断需要什么
 ↓
Tool
 ↓
返回结果
 ↓
Agent 再决定下一步
```

比如：

```text
用户：帮我查一下这个公司的最新信息

Agent
 ↓
判断需要搜索
 ↓
Search Tool
 ↓
返回搜索结果
 ↓
Agent 再决定下一步
```

今天学习多 Agent 后，我发现一个非常有意思的事情：

> **Supervisor 调度 Worker，和 Agent 调度 Tool，在控制结构上其实非常像。**

它们都可以抽象成：

```text
决策者
   ↓
选择一个执行单元
   ↓
执行
   ↓
返回结果
   ↓
再次决策
```

所以：

```text
Tool Routing
    ↓
选择一个 Tool

Supervisor Routing
    ↓
选择一个 Worker
```

这两个结构看起来非常接近。

但真正的区别不在“是不是路由”，而在于：

> **被路由的那个东西，到底有多强的自治能力。**

这个区别是今天理解多 Agent 的一个重要切入口。

---

# 二、先把 Tool、Agent、Worker、Supervisor 放到同一张图里

可以先用一个层级去理解：

```text
函数
 ↓
Tool
 ↓
Agent
 ↓
Worker
 ↓
Supervisor
 ↓
整个 Agent System
```

这不是 LangGraph 官方规定的分类，而是我今天用来建立架构直觉的一种方式。

越往上，通常意味着：

```text
职责更完整
状态更多
决策更多
生命周期更长
```

---

# 三、Tool 和 Worker 为什么看起来很像？

先看 Tool。

一个普通 Tool 大概是：

```text
输入
 ↓
执行固定能力
 ↓
输出
```

比如：

```text
search(query)
```

它通常不会自己思考：

```text
“我要不要调用另一个 Tool？”
“我要不要重新定义任务？”
“我要不要把结果交给 reviewer？”
```

它的主要职责是：

> **提供一个可被调用的能力。**

---

而 Worker 不一样。

例如：

```text
Supervisor
   ↓
Researcher
```

Researcher 接收到任务后，它可能自己：

```text
理解任务
 ↓
决定怎么查
 ↓
调用搜索工具
 ↓
分析搜索结果
 ↓
判断是否需要继续查
 ↓
整理结果
 ↓
返回研究结论
```

所以 Worker 更接近：

> **一个有自己任务目标、上下文和执行逻辑的 Agent。**

也就是说：

```text
Supervisor
   ↓
Worker
   ↓
Agent
   ↓
Tools
```

从 Supervisor 的角度看：

> Worker 是一个更高级的“能力模块”。

从 Worker 自己的角度看：

> Worker 内部又可能是一个完整 Agent。

---

# 四、所以我之前想到的“自己的 buffer”到底意味着什么？

这个问题很有意思。

如果一个 Tool 有：

```text
自己的 buffer
自己的状态
自己的 invoke
自己的内部处理流程
```

它会越来越像一个独立模块。

但这里要特别注意：

> **有 State，不等于就是 Agent。**

例如：

```text
Search Tool
  └── 保存最近 100 次查询
```

虽然它有状态，但它仍然可以只是：

```text
输入
 ↓
执行
 ↓
输出
```

真正让一个组件越来越接近 Agent 的，不只是“有状态”，而是：

```text
状态
+
目标
+
决策
+
行动
+
控制循环
```

所以可以这样区分：

| 类型         | 状态  | 自己决策 | 自己目标  | 控制循环 |
| ---------- | --- | ---- | ----- | ---- |
| 普通函数       | 通常无 | 无    | 无     | 无    |
| Tool       | 可有  | 很少   | 通常没有  | 通常无  |
| Agent      | 有   | 有    | 有     | 通常有  |
| Worker     | 有   | 有    | 有明确职责 | 有    |
| Supervisor | 有   | 有    | 是调度目标 | 有    |

这里真正应该抓住的是：

> **不是名字决定它是什么，而是它拥有多少控制权。**

---

# 五、这个发现让我重新理解“Supervisor”

之前容易把 Supervisor 理解成：

> 一个特殊的 Agent。

现在看，它更准确的理解是：

> **一个负责“Agent Routing”的中央调度者。**

也就是：

```text
Tool Routing：

Agent
  ↓
选择 Tool
  ↓
Tool 执行
  ↓
返回结果
  ↓
Agent 再决策
```

而：

```text
Supervisor Routing：

Supervisor
  ↓
选择 Worker
  ↓
Worker 执行
  ↓
返回结果
  ↓
Supervisor 再决策
```

于是：

```text
Tool Routing
    =
在 Tool 层做路由

Supervisor Routing
    =
在 Agent / Worker 层做路由
```

这就是两者非常像的根本原因。

---

# 六、那么为什么还要搞多 Agent？

这里回到多 Agent 本身。

多 Agent 首先不是：

> “让系统更聪明。”

而是：

> **把复杂任务拆成多个具有不同职责、工具和验收标准的执行单元。**

所以：

```text
多 Agent ≠ 更强的大脑

多 Agent = 分工系统
```

例如一个复杂写作任务：

```text
题目
 ↓
Planner
 ↓
Researcher
 ↓
Writer
 ↓
Reviewer
 ↓
Reviser
 ↓
最终文章
```

这里每一个角色关注的是不同事情。

---

# 七、单 Agent 为什么会遇到瓶颈？

## 1. 上下文争抢

一个 Agent 同时负责：

```text
搜索
写作
审查
修改
工具调用
记忆
```

所有信息进入一个上下文。

于是：

```text
工具越来越多
+
Prompt 越来越长
+
历史越来越长
=
上下文越来越复杂
```

---

## 2. 职责耦合

比如：

```text
“你负责查资料 + 写文章 + 自己审核 + 自己修改”
```

Prompt 会越来越长。

修改某一部分要求，还可能影响另外的行为。

所以真正的问题不是“Agent 不够聪明”，而是：

> **职责开始互相耦合。**

---

## 3. 不容易表达真正不同的角色

例如：

```text
Writer：
目标是把文章写好。

Reviewer：
目标是主动挑错。
```

这两个目标并不完全一致。

如果让一个 Agent：

```text
先写
然后自己批评自己
再自己决定通过
```

很容易变成：

> 自己写 → 自己检查 → 自己觉得没问题 → 自己通过。

所以把角色拆开，本质上是在建立：

> **独立视角。**

---

# 八、但多 Agent 也不是白赚的

拆分以后，新的成本也来了。

## Token 成本

多个 Agent 需要分别理解上下文。

---

## 延迟成本

如果：

```text
Planner
 ↓
Researcher
 ↓
Writer
 ↓
Reviewer
 ↓
Reviser
```

是串行的，那么：

```text
总耗时 ≈ 各阶段耗时之和
```

所以：

> **多 Agent 并不等于更快。**

---

## 失败传播

例如：

```text
Researcher 查错资料
       ↓
Writer 基于错误资料写
       ↓
Reviewer 审查错误内容
       ↓
最终结果全部建立在错误基础上
```

所以多 Agent 以后，还必须考虑：

> 错误在哪里发生？怎么发现？失败之后去哪？

---

## 编排复杂度

以前只有：

```text
Agent
```

现在变成：

```text
Supervisor
 ├── Planner
 ├── Researcher
 ├── Writer
 ├── Reviewer
 └── Reviser
```

于是多了很多系统问题：

```text
谁决定下一步？
谁拥有工具？
谁拥有写权限？
失败之后去哪？
什么时候停止？
怎么避免死循环？
```

所以：

> **多 Agent 的收益必须大于编排成本。**

---

# 九、什么时候应该拆 Agent？

今天一个非常重要的判断原则：

> **看专业边界，不要只看步骤数量。**

例如：

### “解释一段代码”

虽然可以拆成：

```text
阅读
 ↓
分析
 ↓
总结
```

但这些仍属于同一个专业：

> 代码理解。

所以没有必要为了“步骤多”就拆。

---

### “写一篇技术调研”

如果过程是：

```text
查资料
 ↓
写作
 ↓
独立审查
```

这里已经出现三个不同专业角色：

```text
Research
Writing
Review
```

而且它们还可以拥有不同工具和不同验收标准。

因此拆分就有价值。

---

# 十、Supervisor 的核心其实只有一句话

> **Supervisor 决定下一步由谁执行。**

例如：

```text
                   Supervisor
                       │
             决定下一步交给谁
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
    Researcher       Writer        Reviewer
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                  回 Supervisor
```

这里 Supervisor 自己并不负责：

```text
搜索资料
写文章
审文章
```

它负责：

> **调度。**

因此可以继续用之前的类比：

### Supervisor = 医院分诊台

```text
病人
 ↓
分诊台
 ↓
决定去哪个科室
 ↓
专科医生处理
 ↓
结果返回
 ↓
继续判断是否需要其他科室
```

分诊台不是医生。

Supervisor 也不是具体 Worker。

---

# 十一、Supervisor 有四个必须回答的问题

## ① 谁决策？

只有 Supervisor 决定下一步。

Worker 负责：

```text
执行
返回结果
```

而不是：

```text
直接控制整个流程
```

如果 Worker 开始自己决定：

```text
Researcher → Writer
Writer → Reviewer
Reviewer → Reviser
```

那么系统就开始往另一种架构移动。

---

# 十二、② 专业边界是什么？

每个 Worker 应该有：

```text
一个核心职责
+
明确目标
+
角色约束
+
自己的工具集合
```

例如：

| Worker     | 核心职责 |
| ---------- | ---- |
| Planner    | 拆任务  |
| Researcher | 查资料  |
| Writer     | 写作   |
| Reviewer   | 找问题  |
| Reviser    | 修改   |

这里真正重要的是：

> **角色不仅靠 Prompt 区分，也靠工具和权限区分。**

例如：

```text
Researcher
  ├── Search
  ├── Retrieval
  └── 文件读取

Writer
  └── 没有搜索工具
```

这样角色边界就不只是：

> “Prompt 里告诉你是 researcher。”

而是：

> **系统能力本身也限制了你是什么角色。**

---

# 十三、③ Agent 之间怎么通信？

可以先分成三种。

## 1. 共享 State

多个节点围绕同一个 State 工作：

```text
State
 ├── messages
 ├── 当前任务
 ├── 中间结果
 └── 路由信息
```

Worker：

```text
读取 State
 ↓
处理
 ↓
返回 State 更新
```

---

## 2. 带名字的消息

例如：

```text
Researcher：
“这是研究结果”

Reviewer：
“这是审查意见”
```

这样系统能知道：

> **这条信息是谁产生的。**

这个信息对调试特别重要。

---

## 3. Send 动态分发

这个是 W6 学过的：

```text
一个任务
 ↓
动态拆成 N 份
 ↓
并行执行
 ↓
汇合
```

这里最重要的是：

> **Send 和 Supervisor 虽然都会“动态派任务”，但解决的是不同问题。**

---

# 十四、Send 和 Supervisor：为什么这么容易混？

可以这样记：

### Send

> **一次性派活。**

例如：

```text
5 个数据源
 ↓
Send
 ↓
A B C D E
 ↓
Join
```

任务开始的时候就知道：

> 这 5 个都要做。

---

### Supervisor

> **每一步重新决定。**

例如：

```text
Research
 ↓
Writer
 ↓
Reviewer
 ↓
不通过？
 ↓
Reviser
 ↓
Reviewer
 ↓
通过？
 ↓
FINISH
```

所以：

```text
Send：
动态扇出

Supervisor：
动态路由
```

最简单的理解：

> **Send 是“派完就汇合”；Supervisor 是“干完以后再决定下一步”。**

---

# 十五、所以“多 Agent”和“并行”不是一回事

这是今天需要彻底区分的一点。

```text
多 Agent
=
分工

并行
=
执行方式
```

因此：

```text
多 Agent 可以串行

多 Agent 也可以结合 Send 做并行
```

例如：

```text
Supervisor
 ↓
Researcher
 ↓
Writer
 ↓
Reviewer
```

这是多 Agent，但完全可以串行。

而：

```text
Researcher
 ├── Source A
 ├── Source B
 ├── Source C
 └── Source D
```

这是可以并行的任务。

所以：

> **不要因为看到多个 Agent，就自动想到“并行”。**

---

# 十六、Control Flow 和 Data Flow

这是今天继续往深处理解 LangGraph 时非常重要的一层。

## Control Flow：控制流

回答：

> **下一步去哪？**

例如：

```text
Supervisor
 ↓
Researcher
 ↓
Supervisor
 ↓
Writer
```

这是：

```text
谁执行？
什么时候执行？
下一步去哪？
```

---

## Data Flow：数据流

回答：

> **信息怎么流？**

例如：

```text
Researcher
 ↓
研究结果
 ↓
State
 ↓
Writer
 ↓
初稿
 ↓
State
 ↓
Reviewer
```

所以：

```text
Control Flow = 路怎么走

Data Flow = 东西怎么传
```

以后看复杂 LangGraph 时，不能只看：

> “节点怎么连。”

还要同时看：

> **控制流怎么走？数据流怎么走？**

---

# 十七、为什么“有 State”还不等于“有 Agent”？

这和刚才的 Tool / Worker 区别可以连起来理解。

一个 Tool 完全可以有自己的状态：

```text
Tool
 └── Buffer
```

但它仍然可能只是：

```text
输入
 ↓
固定执行
 ↓
输出
```

Agent 的区别在于：

```text
State
+
目标
+
决策
+
行动
+
控制循环
```

所以可以记：

> **State 是基础，但自治能力才是 Agent 与普通工具进一步拉开差异的地方。**

---

# 十八、Supervisor、Worker、Tool 的层级关系

现在可以把前面的内容组合起来：

```text
                    Supervisor
                         │
                  决定调用谁
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
      Researcher       Writer        Reviewer
          │
          ↓
        Agent
      ┌────┼────┐
      ↓    ↓    ↓
   Search  DB  Retrieval
      │    │     │
      └────┴─────┘
         Tools
```

从上往下：

```text
Supervisor
    ↓
调度 Worker

Worker
    ↓
自己完成专业任务

Agent
    ↓
做内部决策

Tool
    ↓
提供具体能力
```

因此：

> **Supervisor 调的是“更大的能力单元”。**

---

# 十九、这让我重新理解“Agent 可以当成 Tool”

这个想法其实非常重要。

对于 Supervisor 来说：

```text
Researcher
```

可以被看成一个能力：

> “帮我完成研究。”

所以 Supervisor 看：

```text
Researcher ≈ 一个高级 Tool
```

但对于 Researcher 自己：

```text
Researcher
 ↓
LLM
 ↓
Search Tool
 ↓
Retrieval Tool
 ↓
内部 State
 ↓
循环
```

它又是一个完整 Agent。

于是：

> **一个组件是不是 Tool 或 Agent，有时候取决于你站在哪一层看它。**

从上层看：

```text
Worker = 能力模块
```

从 Worker 自己看：

```text
Worker = Agent
```

这就是层级化 Agent 的感觉。

---

# 二十、④ 系统怎么停？

Supervisor 最容易出现的问题之一就是：

```text
Supervisor
 ↓
Worker
 ↓
Supervisor
 ↓
Worker
 ↓
……
```

如果没有停止机制：

> 就可能形成死循环。

因此要有至少三层防线。

---

## 第一层：语义结束

例如：

```text
FINISH
```

意思是：

> 模型认为任务完成。

但这是 LLM 判断。

所以：

> 它会飘。

---

## 第二层：质量门槛

例如：

```text
Reviewer
 ↓
评分
 ↓
通过？
```

通过：

```text
FINISH
```

不通过：

```text
Reviser
```

这比完全依赖 Supervisor 的主观判断更好。

但：

> 它仍然是模型判断。

---

## 第三层：确定性硬上限

例如：

```text
MAX_ROUNDS
```

无论模型怎么判断：

> 到达最大轮数，必须停止。

所以：

```text
语义停止
+
质量停止
+
硬停止
```

这才是比较完整的停止设计。

---

# 二十一、为什么硬上限特别重要？

因为：

```text
FINISH
```

是模型判断。

```text
Reviewer >= 7
```

也是模型判断。

只有：

```text
MAX_ROUNDS = N
```

这种机制是确定性的。

所以：

> **不要把“模型应该会停”当作系统一定会停。**

以后设计 Agent Loop 时，都应该问一句：

> “如果模型判断错了，我的系统还能不能强制停下来？”

---

# 二十二、Worker 的权限为什么也是系统设计的一部分？

因为：

> **能做什么和负责什么，是两个不同的问题。**

例如：

```text
Publisher
```

负责发布内容。

但不一定意味着：

```text
Publisher
=
自动拥有发布权限
```

如果操作会产生副作用：

```text
写文件
修改数据库
调用外部 API
发送消息
删除内容
发布结果
```

可以设计：

```text
Worker
 ↓
Permission / Approval Gate
 ↓
允许 / 拒绝
```

所以：

```text
Role Boundary
=
谁负责什么

Permission Boundary
=
谁有资格真的做什么
```

这两个边界不能混成一个。

---

# 二十三、失败路径也必须设计

一个完整的多 Agent 系统不能只考虑：

```text
成功
```

还要考虑：

```text
失败
部分成功
超时
工具失败
结果无效
重复失败
```

例如：

```text
Researcher
 ↓
Search 失败
```

系统可以：

```text
失败
 ↓
重试
```

也可以：

```text
失败
 ↓
换工具
```

或者：

```text
失败
 ↓
回 Supervisor
 ↓
重新派任务
```

再或者：

```text
失败次数超过上限
 ↓
终止
```

所以 Worker 的设计不仅是：

> “它成功以后返回什么？”

还要问：

> **“它失败以后返回什么？”**

---

# 二十四、Supervisor 系统的完整骨架

把今天所有知识合起来，可以得到：

```text
                    ┌────────────────┐
                    │   Supervisor   │
                    │  决定 next      │
                    └───────┬────────┘
                            ↓
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
         Researcher       Writer       Reviewer
              │
              ↓
         内部 Agent
              │
       ┌──────┼──────┐
       ↓      ↓      ↓
    Search    DB   Retrieval
       │      │      │
       └──────┴──────┘
             Tools

所有 Worker
      ↓
    State
      ↓
回 Supervisor
      ↓
┌──────────────────────────┐
│ 成功？                    │
│ 质量达标？                │
│ 超过最大轮数？            │
│ 失败是否需要重试？        │
└───────────┬──────────────┘
            ↓
       FINISH / 下一轮
```

这个图里其实已经包含了今天绝大多数知识。

---

# 二十五、以后判断一个多 Agent 系统，可以问这 8 个问题

以后看到一个新架构，我不应该先问：

> “这里用了几个 Agent？”

而应该按这个顺序问：

```text
1. 为什么需要拆？
2. 每个 Worker 的专业边界是什么？
3. 每个 Worker 有什么工具？
4. State 在哪里？
5. Control Flow 怎么走？
6. Data Flow 怎么走？
7. 哪些动作需要权限 / 审批？
8. 系统怎么停止、失败怎么办？
```

如果这 8 个问题都能回答：

> 这个系统的基本架构就已经开始真正理解了。

---

# 二十六、Supervisor 和 Swarm

最后再区分一个容易混淆的概念。

## Supervisor

```text
Supervisor
   ↓
Worker
   ↓
Supervisor
   ↓
Worker
```

核心：

> **中央调度。**

---

## Swarm

更接近：

```text
Agent A
 ↓
Agent B
 ↓
Agent C
 ↓
Agent A
```

每个 Agent 都可能参与决定下一步。

因此两者真正的核心差别就是：

> **谁决定下一步。**

可以记成：

```text
Supervisor = 中央决策

Swarm = 对等协作
```

---

# 二十七、今天我真正理解到的东西

今天原本以为要学习：

> “Supervisor 怎么写。”

但真正理解以后，发现它其实和我之前学的东西串起来了。

以前：

```text
Agent
 ↓
Tool Routing
 ↓
选择 Tool
```

现在：

```text
Supervisor
 ↓
Agent Routing
 ↓
选择 Worker
```

它们底层的控制思想是相似的。

只是：

```text
Tool
=
具体能力

Worker
=
完整的专业执行单元
```

于是：

```text
Agent 调 Tool
```

和：

```text
Supervisor 调 Worker
```

其实可以看成**不同层级上的路由**。

这也让我开始理解：

> **复杂 Agent 系统不一定是在增加完全不同的新机制，很多时候是在把以前已经存在的“决策 → 调用 → 返回 → 再决策”这一套机制向更高层次扩展。**

---

# 二十八、今天的最终认知图

最后把今天压缩成一张图：

```text
                    任务
                      │
                      ↓
              是否需要拆分？
                      │
             ┌────────┴────────┐
             ↓                 ↓
           不需要             需要
             ↓                 ↓
          单 Agent         专业角色拆分
                               │
                               ↓
                         Supervisor
                               │
                        决定下一步
                               │
            ┌──────────────────┼──────────────────┐
            ↓                  ↓                  ↓
        Researcher          Writer            Reviewer
            │
            ↓
         Agent
            │
       ┌────┼────┐
       ↓    ↓    ↓
     Tool  Tool  Tool
       
        
Control Flow：
谁 → 谁

Data Flow：
数据 → 哪里

Permission：
谁 → 有资格做什么

Termination：
什么时候必须停

Failure：
出了问题怎么办
```

---

# 二十九、最后一句话

> **多 Agent 的本质不是“多几个模型”，而是把一个复杂任务变成多个具有明确职责、工具、状态和权限边界的执行单元，再通过一个控制机制把它们组织起来。**

而 Supervisor 的核心也不是：

> “我有一个主管 Agent。”

而是：

> **“我把 Agent 本身也变成了可以被上一级系统路由的能力单元。”**

所以今天最值得记住的不是某一个 API，而是这条链：

```text
Tool Routing
   ↓
选择工具

Agent
   ↓
内部决定怎么完成任务

Worker
   ↓
把一个完整专业任务封装成执行单元

Supervisor Routing
   ↓
选择哪个 Agent / Worker 执行

Multi-Agent System
   ↓
把这些执行单元组织成一个可控制的系统
```

而真正工程化以后，还必须继续回答：

```text
状态怎么流
控制怎么走
权限怎么控
失败怎么办
什么时候停
成本是否值得
```

这才是今天 W7-Day1 真正要建立起来的架构思维。
