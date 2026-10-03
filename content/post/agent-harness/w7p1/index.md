---
description: ""
title: " W7 总览：从多 Agent 流程到 Agent Harness"
draft: false
date: "2026-09-21T14:36:46+08:00"
slug: "W7P1"
categories:
 - Harness
 - Agent
tags:
 - 
image: ""
---

# W7 总览：从多 Agent 流程到 Agent Harness

这一周的学习内容很多。

刚开始看，每天似乎都是一个新的知识点：

- Supervisor
- 多 Agent
- Structured Output
- Reflection
- Critic / Reviser
- Tool Registry
- Permission Gate
- Session Store
- Context Compaction
- Tracer
- Facade
- 测试
- 故障注入
- 最后还有一个 `writer_pipeline.py`

如果把这些东西一个个记，很容易变成：

> “这些我都见过，但是如果让我从头解释整个 W7，我不知道它们为什么会出现在一起。”

所以 Day6 做的事情，其实就是让我重新回答一个问题：

> **这一周到底在造什么？**

我现在对 W7 的理解是：

> **W7 不是单纯学习 Multi-Agent，而是在尝试把一个“会自己决定下一步的 Agent 系统”，逐渐变成一个具有路由、反思、工具权限、持久化、上下文管理和可观测能力的 Agent 应用。**

也就是说，这一周其实有两条线。

**W7 一页脑图版**
```
                         W7
                          │
          ┌───────────────┴────────────────┐
          │                                │
       Agent 大脑                       Agent 身体
          │                                │
          ▼                                ▼
     Supervisor                         Harness
          │                                │
     决定下一步                      ┌─────┼─────┐
          │                         ↓     ↓     ↓
   ┌──────┼──────┐               Registry Gate Session
   ↓      ↓      ↓                   │      │      │
Research Writer Reviewer             │      │      ↓
          │                          │      │  Compactor
          ↓                          │      │      │
      Reflection                     │      │      ↓
          │                          │      │    LLM
     ┌────┴────┐                     │      │
     ↓         ↓                     │      ↓
  Critic    Reviser                  │    Executor
     ↑         │                     │
     └─────────┘                     │
                                     ↓
                                   Tracer

                     ↓
                 Day6 Integration
                     ↓
       ┌─────────────┼─────────────┐
       ↓             ↓             ↓
    消息适配       窄接口        五模块实际生效
       ↓             ↓             ↓
 LangChain msg   ReflectionResult  文件/日志/trace
       │             │             │
       └─────────────┼─────────────┘
                     ↓
                  端到端
                     ↓
                故障注入
                     ↓
                  测试证据
```
---


## 一、第一条线：Agent 的“脑子”越来越复杂

这一条线是：

```text
Supervisor
    ↓
多个 Worker
    ↓
Reflection
    ↓
Critic
    ↓
Reviser
    ↓
重新评审
```

它解决的是：

> **一个 LLM 不够用了，如何让多个角色协作，并且让结果能够自我检查和修改？**

---

## 二、第二条线：Agent 的“身体”越来越完整

另一条线是 Harness：

```text
Tool Registry
      ↓
Permission Gate
      ↓
Executor

Session Store
Context Compactor
Tracer
```

它解决的是另外一类问题：

> **Agent 会思考还不够，它真正运行的时候，工具怎么管理？危险操作怎么办？上下文太长怎么办？重启之后怎么办？出了问题怎么查？**

所以我现在可以把 W7 看成：

```text
                  ┌──────────────────────┐
                  │       Agent 大脑      │
                  │                      │
用户任务 → Supervisor → Worker → Reflection
                  │                      │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │      Agent Harness   │
                  │                      │
                  │ Registry             │
                  │ Permission Gate      │
                  │ Session Store        │
                  │ Context Compactor    │
                  │ Tracer               │
                  └──────────────────────┘
```

前半部分解决：

> **怎么想、怎么分工、怎么反思。**

后半部分解决：

> **怎么安全、稳定、可持续地运行。**

而 Day6 的任务就是：

> **把这两个世界真正接起来。**

---

# 二、Day1：先设计“机器长什么样”

W7 的第一天实际上更像架构设计，而不是继续学习某个 API。

这一阶段最重要的不是代码，而是先确定：

```text
用户
 ↓
Supervisor
 ↓
Researcher
 ↓
Writer
 ↓
Reviewer
 ↓
是否需要修改？
 ├─ 是 → Writer
 └─ 否 → FINISH
```

同时开始考虑：

```text
Agent
 ↓
Tools
 ↓
权限
 ↓
状态
 ↓
上下文
 ↓
追踪
```

所以 Day1 给后面留下了两张图：

### 第一张图：Agent 编排图

```text
Supervisor
   │
   ├── Researcher
   │
   ├── Writer
   │
   └── Reviewer
          │
          └── 需要修改 → Writer
```

### 第二张图：Harness 图

```text
Agent
 │
 ├── Registry
 ├── Gate
 ├── Session
 ├── Compactor
 └── Tracer
```

后面所有代码其实都是在把这两张图变成真实程序。

所以我发现：

> **复杂项目不能一上来就读代码。**

先画结构，才能知道代码里的函数到底属于哪一个盒子。

---

# 三、Day2：先解决“多个 Agent 怎么协作”

Day2 开始真正写 Supervisor。

核心文件是：

```text
supervisor_demo.py
```

它解决的问题非常明确：

> **不是让三个 Agent 自己聊天，而是增加一个 Supervisor，决定下一步应该谁工作。**

整体结构：

```text
START
  ↓
supervisor
  ↓
根据 next 决定
  ├── researcher
  ├── writer
  ├── reviewer
  └── FINISH
       ↑
       │
worker ─┘
```

这里有一个非常重要的理解：

### Worker 不负责决定流程

Worker 只负责：

```text
收到任务
 ↓
干自己的活
 ↓
交活
```

Supervisor 负责：

```text
看当前状态
 ↓
判断缺什么
 ↓
决定下一步
```

所以：

```text
Worker = 干活

Supervisor = 调度
```

这其实已经非常接近真实 Agent 系统里的：

```text
执行者
vs
Orchestrator
```

---

# 四、为什么 Supervisor 需要 Structured Output？

Supervisor 的输出不能是：

```text
“我觉得下一步应该让研究员继续。”
```

因为程序没法稳定解析。

所以 Day2 定义：

```python
class Route(BaseModel):
    next: Literal[
        "researcher",
        "writer",
        "reviewer",
        "FINISH"
    ]
    reason: str
```

于是模型输出变成：

```json
{
    "next": "researcher",
    "reason": "还没有资料"
}
```

这时候：

```text
LLM
 ↓
Route
 ↓
state["next"]
 ↓
conditional edge
 ↓
worker
```

整个链条就变成程序可以控制的东西。

这也是我之前学习 Structured Output 时理解的东西，在 W7 真正落到了实际项目里：

> **LLM 可以负责做判断，但程序不能直接把自然语言判断当成控制流。**

所以：

```text
自然语言
    ↓
结构化结果
    ↓
程序控制流程
```

是 Agent 工程里非常重要的一层。

---

# 五、Day2 真正让我学到的不是 Supervisor，而是“控制流”

如果只记：

> Supervisor 是一个调度 Agent。

其实还是浅。

真正应该记住的是：

```text
State
 ↓
Supervisor 判断
 ↓
next
 ↓
Conditional Edge
 ↓
Worker
 ↓
State 更新
 ↓
Supervisor 再判断
```

所以 Supervisor 本质上是在控制：

> **Control Flow**

而 `messages` 等字段承载的是：

> **Data Flow**

这两个东西必须分开看。

例如：

```text
next = "writer"
```

是控制流。

而：

```text
messages = [...]
```

是数据流。

这是 W7 开始变复杂以后，非常重要的一种阅读代码方式。

---

# 六、Day3：发现一个新问题——Agent 会一直改

Day2 解决：

> 多个 Agent 怎么协作？

Day3 开始解决：

> **一个 Agent 产出的结果，怎么知道够不够好？**

于是出现：

```text
generator
    ↓
critic
    ↓
reviser
    ↓
critic
    ↓
reviser
    ↓
...
```

也就是 Reflection。

核心文件：

```text
reflection_graph.py
```

---

# 七、Reflection 本质上是一个反馈控制回路

如果把它抽象出来：

```text
生成
 ↓
评价
 ↓
发现问题
 ↓
修改
 ↓
重新评价
 ↓
是否达标？
 ├── 是 → END
 └── 否 → 修改
```

这其实不是简单的：

> “再调用一次 LLM。”

而是：

> **让一次 Agent 执行拥有反馈回路。**

所以 Day3 的重点其实是：

```text
生成器
+
评价器
+
修改器
+
停止条件
```

---

# 八、为什么 Reflection 必须有停止条件？

因为：

```text
critic → reviser → critic → reviser
```

天然就是一个循环。

如果没有出口：

```text
无限调用
无限 Token
无限成本
```

所以 Day3 设计了三个出口条件。

---

## 1. 分数达标

```python
if score >= PASS_SCORE:
    return "end"
```

比如：

```text
8 分以上 → 结束
```

---

## 2. 修改次数达到上限

```python
if count >= MAX_REVISE:
    return "end"
```

例如：

```text
最多修改 3 次
```

这解决的是：

> 即使永远达不到标准，也不能无限烧钱。

---

## 3. 停滞

例如：

```text
第一次 7 分
第二次 7 分
第三次 6 分
```

说明继续改可能没有价值。

于是：

```text
改了
 ↓
没有变好
 ↓
停止
```

---

# 九、Day3 又学到一个非常重要的东西：不要完全相信模型自己

Critic 有：

```python
passed: bool
```

模型可以告诉程序：

```text
passed = True
```

但代码没有直接相信它。

而是：

```python
passed_by_code = score >= PASS_SCORE
```

也就是说：

```text
模型负责提供：
score

程序负责决定：
passed
```

这和之前 Structured Output 的思想是一致的：

> **让模型提供信息，但关键控制权留在程序。**

---

# 十、Day3 又增加了“硬伤”

这比普通评分更重要。

比如一篇文章：

```text
评分：9 分
```

但是：

```text
编造了一个数据
```

这时候不能说：

> 9 分，所以通过。

因此增加：

```python
fatal_issues
```

形成：

```text
score >= 8
       │
       ├── 没硬伤 → 可以通过
       │
       └── 有硬伤 → 不通过
```

所以：

> **分数是质量指标，fatal issue 是硬约束。**

这已经开始接近真正的 Agent 评估系统：

```text
软指标
+
硬约束
```

---

# 十一、Day4：Agent 不只是“会用工具”，还必须“受管控”

到了 Day4，问题又变化了。

前面我们讨论：

> Agent 怎么思考？

现在开始讨论：

> **Agent 想调用工具的时候，真的可以随便调用吗？**

答案当然不是。

例如：

```text
search_web
```

和：

```text
delete_file
publish_article
send_money
```

风险完全不同。

所以 Day4 开始建立 Harness 的前两块：

```text
Tool Registry
      ↓
Permission Gate
      ↓
Executor
```

---

# 十二、Tool Registry 是什么？

我现在更倾向于把 Registry 理解成：

> **Agent 的工具通讯录。**

里面记录：

```text
工具名
工具函数
描述
权限等级
标签
```

例如：

```text
search_web
level = safe

delete_file
level = dangerous
```

Registry 负责回答：

```text
有什么工具？
这个工具存在吗？
这个工具是什么？
它危险吗？
```

但它不负责：

> “这个 Agent 现在能不能用。”

这个事情交给 Gate。

所以：

```text
Registry = 能力目录

Gate = 权限判断
```

这个边界非常重要。

---

# 十三、为什么需要 Permission Gate？

如果没有 Gate：

```text
LLM
 ↓
工具
```

模型一旦输出：

```text
delete_file(...)
```

程序就可能直接执行。

所以加：

```text
LLM
 ↓
Tool Registry
 ↓
Permission Gate
 ↓
Executor
 ↓
真正工具
```

Gate 有三个结果：

```text
ALLOW
DENY
NEED_APPROVAL
```

也就是：

```text
普通工具
    ↓
直接允许

危险工具
    ↓
需要人工确认

黑名单工具
    ↓
直接拒绝
```

---

# 十四、Day4 真正重要的是“拒绝以后真的不能执行”

测试不能只写：

```python
assert result == "拒绝"
```

因为可能出现：

```text
返回结果说拒绝
但是工具实际上已经执行
```

所以测试里专门准备：

```python
CALLS = []
```

工具真正执行的时候：

```python
CALLS.append("delete_file")
```

然后验证：

```text
Gate 拒绝
 ↓
CALLS == []
```

这才证明：

> **拒绝不是嘴上拒绝，而是执行路径真的没有发生。**

这是我这一周对“测试”的一个重要理解：

> **不要只测返回值，要测副作用。**

---

# 十五、Day5：Harness 开始从“安全”变成“完整运行环境”

Day4 主要解决：

```text
工具
权限
审批
审计
```

Day5 又增加：

```text
Session Store
Context Compactor
Tracer
Harness Facade
```

于是五模块终于形成：

```text
             Harness
                │
 ┌──────────────┼──────────────┐
 ↓              ↓              ↓
Registry       Gate         Session
                               
              ↓
          Compactor
                               
              ↓
            Tracer
```

这时候我才真正理解：

> **Harness 不是 Agent 的“大脑”。**

它更像：

> **Agent 运行时的基础设施层。**

---

# 十六、Session Store：让 Agent 能“记住运行状态”

Session Store 解决：

> 程序重启之后，之前的会话还在不在？

如果只有：

```python
self._mem = {}
```

那么：

```text
程序退出
 ↓
内存消失
 ↓
全部历史消失
```

所以 Day5 增加 JSONL：

```text
writer_sessions.jsonl
```

存：

```text
thread_id
seq
messages
meta
timestamp
```

于是：

```text
第一次进程
 ↓
append
 ↓
JSONL

程序退出

第二次进程
 ↓
SessionStore(path)
 ↓
load
 ↓
恢复历史
```

---

# 十七、为什么 Session Store 存“全量”，而不是存压缩结果？

这是 Day5 一个非常值得记住的设计决策。

假设：

```text
今天压缩策略：
max_messages = 20
```

明天发现：

```text
20 太多了
改成 10
```

如果数据库里已经保存的是压缩结果：

```text
早期消息已经永久丢失
```

但如果：

```text
Session Store
    ↓
保存全量历史

Compactor
    ↓
读取时做投影
```

那么：

```text
原始数据
   ↓
策略 A → 一个视图
   ↓
策略 B → 另一个视图
```

所以：

> **Session Store 是事实来源，Compactor 是视图。**

这个思想其实非常像：

```text
原始日志
+
按需投影
```

---

# 十八、Context Compactor：不是删除历史，而是改变给模型看的视图

假设有：

```text
41 条消息
```

但模型只需要：

```text
最近 6 条
+
早期内容摘要
```

于是：

```text
41 条原始消息
       ↓
     Compactor
       ↓
┌───────────────┐
│ system        │
│ 历史摘要       │
│ recent 6 条    │
└───────────────┘
```

关键点：

> **压缩不是销毁。**

原始数据仍然在 Session Store。

---

# 十九、Context Compactor 最容易出现的坑：Tool Call 不能拆

比如：

```text
assistant
  tool_calls = call_1

tool
  tool_call_id = call_1
```

这两个实际上是一对。

如果压缩的时候：

```text
删掉 assistant
保留 tool
```

就变成：

```text
tool(call_1)
```

但是找不到：

```text
谁发起了 call_1？
```

于是就可能出现 API 错误。

所以 `_safe_cut()` 做的事情不是：

> “随便从第 N 条切。”

而是：

> **如果切点正好落在 tool 结果上，就往前退，确保完整保留这一组。**

这个地方让我开始真正意识到：

> **上下文压缩不是简单的 list[:N]。**

它已经涉及消息协议的结构约束。

---

# 二十、Tracer：解决“为什么慢、为什么炸”

Tracer 和 Audit 很容易混。

但它们解决的问题完全不同。

### Audit

回答：

> **谁做了什么决定？**

例如：

```text
agent
delete_file
approval_granted
```

主要用于：

```text
安全
审计
追责
```

---

### Trace

回答：

> **程序到底怎么跑的？哪一步慢？谁调用了谁？**

例如：

```text
supervisor  300ms
  writer    200ms
    critic  150ms
```

于是可以看到：

```text
父节点
 └── 子节点
      └── 更深子节点
```

所以：

> Audit 是安全视角。

> Trace 是运行视角。

---

# 二十一、Tracer 为什么需要 contextvars？

因为嵌套调用的时候，需要知道：

```text
当前是谁？
```

比如：

```python
outer()
    ↓
inner()
```

Tracer 要记录：

```text
outer
 └── inner
```

所以使用：

```python
_current
```

保存当前 span。

进入：

```text
outer
 ↓
_current = outer
```

再进入：

```text
inner
 ↓
parent = outer
```

于是：

```text
inner.parent_id = outer.span_id
```

最后就能恢复出树。

---

# 二十二、Day5 结束时，我实际上已经拥有了一套 Agent 基础设施

到这里：

```text
             Agent
               │
       ┌───────┴───────┐
       ↓               ↓
    Tools            State
       │               │
    Registry         Session
       │               │
      Gate          Compactor
       │
    Executor
       
       + Tracer
       + Audit
```

但是：

> **这些东西还是零件。**

它们之间还没有真正装起来。

这就是 Day6。

---

# 二十三、Day6：真正的任务不是“继续写功能”

Day6 的一句话我现在可以重新理解成：

> **Day1–Day5 造零件，Day6 装机器。**

Day6 材料里明确把今天的交付定义成：

```text
① 端到端跑通
② 拓扑图和设计一致
③ 五模块全部实际生效
④ 故障注入
⑤ README
⑥ 3 分钟讲法
```

而且特别强调：

> “接了”不等于“有用”。

代码里：

```python
import Tracer
```

只能证明：

```text
Tracer 被导入了
```

不能证明：

```text
Tracer 真正在记录
```

真正的验收应该是：

```text
trace.jsonl
audit.log
writer_sessions.jsonl
```

这些东西真的产生了。

---

# 二十四、所以我现在可以把整个 W7 串成一条线

```text
Day1
架构设计
   ↓
Day2
Supervisor
   ↓
解决“多人怎么协作”
   ↓
Day3
Reflection
   ↓
解决“结果怎么自我检查”
   ↓
Day4
Registry + Gate + Executor
   ↓
解决“Agent 怎么安全使用工具”
   ↓
Day5
Session + Compactor + Tracer + Facade
   ↓
解决“Agent 怎么长期、稳定、可观察地运行”
   ↓
Day6
Integration
   ↓
解决“这些零件怎么真正接在一起”
```

所以 W7 的真正主线不是：

```text
Supervisor
Reflection
Registry
Gate
Session
Compactor
Tracer
```

这些知识点的简单堆积。

而是：

```text
一个 Agent 系统
       ↓
会思考
       ↓
会协作
       ↓
会反思
       ↓
会使用工具
       ↓
不会随便使用危险工具
       ↓
能保存状态
       ↓
能控制上下文
       ↓
能追踪运行过程
       ↓
最后把这些东西组合成完整系统
```

这才是 W7。

---

# 二十五、我现在对 W7 最大的理解变化

以前我容易把 Agent 理解成：

```text
LLM
 ↓
Tool
 ↓
Result
```

W7 之后，这个模型已经不够了。

现在更接近：

```text
                   ┌───────────────┐
                   │   Supervisor  │
                   └───────┬───────┘
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
         Researcher      Writer       Reviewer
                           │
                           ↓
                     Reflection
                           │
                    ┌──────┴──────┐
                    ↓             ↓
                 Critic        Reviser
                           
                           │
                           ↓
                    ┌─────────────┐
                    │   Harness   │
                    ├─────────────┤
                    │ Registry    │
                    │ Gate        │
                    │ Session     │
                    │ Compactor   │
                    │ Tracer      │
                    └─────────────┘
```

也就是说：

> **LLM 只是系统中的一个决策组件。**

真正的 Agent 应用，是：

```text
LLM
+
Workflow
+
State
+
Tools
+
Permissions
+
Persistence
+
Context Management
+
Observability
+
Evaluation
```

这也是我觉得 W7 比前面的单个 LangGraph API 更重要的地方。

它开始从：

> “我会不会写一个 Agent？”

进入：

> **“我能不能设计一个真正可以运行、可以验证、可以排错的 Agent 系统？”**

---

# 二十六、W7 最后留下的一个工程思维

这周我越来越明显地感觉到：

> **真正复杂的不是代码本身，而是模块之间的边界。**

Day6 就是最典型的例子。

每一个模块单独看都不复杂：

```text
Supervisor
Reflection
Registry
Gate
Session
Compactor
Tracer
```

真正困难的是：

```text
A 输出什么？
 ↓
B 接受什么？
 ↓
两边的数据结构是不是一样？
 ↓
谁负责转换？
 ↓
谁负责判断？
 ↓
谁负责执行？
 ↓
谁负责记录？
```

所以以后再看到复杂代码，我不能只问：

> “这个函数是干什么的？”

还应该问：

> **“它和前后模块的接缝是什么？”**

这可能是 W7 最重要的一次学习升级。