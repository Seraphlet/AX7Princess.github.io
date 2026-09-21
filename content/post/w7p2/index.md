---
description: ""
title: "W7 核心机制：Supervisor、Reflection 与 Harness"
draft: false
date: "2026-09-21T14:40:28+08:00"
slug: "W7P2"
categories:
 - Harness
 - Agent
tags:
 - 
image: ""
---

# W7 核心机制：Supervisor、Reflection 与 Harness

第一篇把 W7 串成了一条主线。

这一篇不再按 Day1、Day2、Day3 去记，而是把整个系统拆成三个核心问题：

```text
① 谁决定下一步？
② 谁负责把结果变好？
③ Agent 运行时需要哪些基础设施？
```

对应：

```text
① Supervisor
② Reflection
③ Harness
```

---

# 一、先看完整系统

先不看代码。

只看这张图：

```text
                    用户任务
                       │
                       ▼
                ┌─────────────┐
                │ Supervisor  │
                └──────┬──────┘
                       │
             决定下一步是谁
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
   Researcher        Writer        Reviewer
                       │              │
                       │              │
                       └──────┬───────┘
                              ↓
                         Reflection
                              │
                     ┌────────┴────────┐
                     ↓                 ↓
                   Critic            Reviser
                     │                 │
                     └──────→ Critic ←─┘
                              │
                              ↓
                            END


                 下面所有节点都运行在
                       Harness
                          │
        ┌─────────────────┼─────────────────┐
        ↓                 ↓                 ↓
    Registry             Gate            Session
        │                                   │
        ↓                                   ↓
    Executor                           Compactor
                                           
                                           
                       Tracer
```

这时候要特别注意：

> Supervisor、Reflection、Harness 不是三个平级功能。

它们解决的是三个不同层次的问题。

---

# 二、Supervisor：解决“下一步干什么”

Supervisor 的输入是：

```text
当前 State
```

输出是：

```text
next
```

例如：

```json
{
    "next": "researcher",
    "reason": "目前还没有资料"
}
```

然后 LangGraph：

```text
supervisor
    ↓
conditional edge
    ↓
researcher
```

所以 Supervisor 本身不负责：

```text
查资料
写文章
审稿
```

它只负责：

> **选择下一个执行者。**

---

# 三、Supervisor 为什么需要 State？

因为它不能凭空判断。

例如：

```text
第一次：
没有 researcher 资料
→ researcher

第二次：
有资料，没有文章
→ writer

第三次：
有文章，没有审核
→ reviewer

第四次：
审核通过
→ FINISH
```

因此 Supervisor 实际上是在：

```text
读取 State
   ↓
理解当前进度
   ↓
做下一步决策
```

这就是：

> **State → Decision → Control Flow**

---

# 四、Supervisor 的代码怎么读？

先找到：

```python
def supervisor_node(state):
```

不要一行一行读。

先画：

```text
state
 ↓
rounds + 1
 ↓
是否超限？
 ├─ 是 → FINISH
 └─ 否
      ↓
System Prompt + messages
      ↓
router.invoke()
      ↓
Route
      ↓
next
```

然后再看：

```python
router = router_llm.with_structured_output(
    Route,
    method="json_mode"
)
```

这句话真正的作用是：

```text
LLM
 ↓
JSON
 ↓
Pydantic Route
 ↓
decision.next
```

所以它不是为了“写 JSON 好看”。

它是为了：

> **把模型的自然语言决策变成程序可以消费的控制数据。**

---

# 五、为什么 `Literal` 很重要？

```python
class Route(BaseModel):
    next: Literal[
        "researcher",
        "writer",
        "reviewer",
        "FINISH"
    ]
```

这里实际上是在告诉程序：

```text
允许的控制流只有这几个
```

如果模型输出：

```text
"next": "delete_database"
```

理论上就不应该进入合法流程。

所以：

```text
Literal
=
控制流白名单
```

---

# 六、Reflection：解决“结果够不够好”

Supervisor 解决：

```text
谁干活
```

Reflection 解决：

```text
干出来的东西怎么样
```

它的流程：

```text
generator
    ↓
critic
    ↓
route
    ├── end
    └── reviser
            ↓
          critic
```

---

# 七、Reflection 的 State 为什么这么多？

Day3 的 State：

```text
topic
draft
score
issues
fatal_issues
revise_count
trajectory
best_draft
best_score
```

刚开始看会觉得：

> 为什么要这么多字段？

其实可以分成四组。

### 第一组：当前任务

```text
topic
draft
```

### 第二组：当前评估

```text
score
issues
fatal_issues
```

### 第三组：循环控制

```text
revise_count
```

### 第四组：历史信息

```text
trajectory
best_draft
best_score
```

这样一下就清楚了。

---

# 八、`trajectory` 为什么要用 Reducer？

```python
trajectory: Annotated[list, operator.add]
```

普通字段：

```text
新值
 ↓
覆盖旧值
```

Reducer：

```text
旧值
+
新值
 ↓
合并
```

所以：

```text
第1轮 → 7分
第2轮 → 8分
第3轮 → 8分
```

最终：

```python
trajectory = [
    {"round": 0, "score": 7},
    {"round": 1, "score": 8},
    {"round": 2, "score": 8}
]
```

这时候它就不只是：

> 当前分数。

而是：

> **整个 Reflection 的历史轨迹。**

---

# 九、为什么要保存 `best_draft`？

因为：

```text
修改
```

不一定：

```text
越改越好
```

可能：

```text
v1 = 7分
v2 = 8分
v3 = 6分
```

如果最终直接拿：

```text
v3
```

就把好结果改坏了。

所以保存：

```text
best_draft = v2
best_score = 8
```

最后：

```text
终稿 = best_draft
```

这个设计特别重要。

因为它承认一个事实：

> **LLM 的修改不是单调优化过程。**

---

# 十、Reflection 的条件边怎么理解？

```python
def route_after_critic(state):
```

不要先看代码。

先画决策树：

```text
critic
  │
  ├── 修改次数 >= MAX_REVISE
  │        ↓
  │       END
  │
  ├── 有 fatal issue
  │        ↓
  │      reviser
  │
  ├── score >= PASS_SCORE
  │        ↓
  │       END
  │
  ├── 分数下降
  │        ↓
  │       END
  │
  └── 其他
           ↓
        reviser
```

所以 Reflection 的本质就是：

> **一个有边界的反馈控制回路。**

---

# 十一、Harness 五模块应该怎么理解？

现在进入另一条线。

不要死记五个名字。

按照 Agent 生命周期记：

```text
Agent 想做什么？
       ↓
Registry
       ↓
允许不允许？
       ↓
Gate
       ↓
真的执行
       ↓
Executor

执行过程中
       ↓
Session 保存状态
       ↓
Compactor 控制上下文
       ↓
Tracer 记录过程
```

这样五个模块就不会混。

---

# 十二、Registry：工具“有什么”

Registry 的职责：

```text
注册
查询
搜索
执行入口
```

但它最重要的一条边界是：

> **Registry 不决定权限。**

例如：

```python
registry.is_dangerous("delete_file")
```

只是回答：

```text
它危险吗？
```

不是：

```text
现在允许执行吗？
```

后者是 Gate。

---

# 十三、为什么 Registry 要提供 `invoke()`？

代码里有：

```python
registry.invoke(name, args)
```

而不是让外面：

```python
spec.fn(...)
```

这是一个非常典型的封装边界。

如果以后：

```text
ToolSpec.fn
```

改成：

```text
ToolSpec.func
```

如果所有调用方都直接访问：

```python
spec.fn
```

整个项目都要改。

但如果统一：

```python
registry.invoke()
```

调用方不知道内部到底叫什么。

所以：

```text
内部实现
    ↓
Registry
    ↓
统一接口
    ↓
外部调用方
```

这就是：

> **封装。**

---

# 十四、Gate：工具“能不能用”

Gate 的判断：

```text
工具存在？
   ↓
黑名单？
   ↓
危险工具？
   ↓
白名单？
   ↓
是否需要审批？
```

因此：

```text
Registry = capability

Gate = authorization
```

这两个概念以后做 Agent 很容易混淆。

---

# 十五、Executor：工具“真的执行了吗”

Executor 是：

```text
Gate
 ↓
真正执行
```

它特别强调：

```python
{
    "ok": True,
    "executed": True,
    "result": ...
}
```

为什么要有：

```text
executed
```

而不仅仅是：

```text
ok
```

因为：

```text
ok=False
```

可能有很多原因：

```text
权限拒绝
人工拒绝
工具异常
```

所以：

```text
ok
```

和：

```text
executed
```

是两个不同维度。

---

# 十六、Session：Agent“经历过什么”

Session 保存：

```text
thread_id
messages
meta
seq
timestamp
```

它回答：

> **这个 Agent 之前发生过什么？**

注意：

```text
Session
```

不是：

```text
Memory
```

至少在这一周的实现里，它首先解决的是：

> **运行状态持久化。**

比如：

```text
thread_id = user-001
```

第一次：

```text
你好
```

程序重启。

第二次：

```text
继续刚才的事情
```

Session 可以恢复之前的状态。

---

# 十七、Compactor：Agent“现在需要看到什么”

Session 是：

```text
全部历史
```

Compactor 是：

```text
当前给 LLM 看的历史
```

所以：

```text
Session
   ↓
全量历史
   ↓
Compactor
   ↓
当前上下文
   ↓
LLM
```

这就是为什么：

> Session 和 Compactor 不能合并成一个东西。

因为：

```text
事实存储
```

和：

```text
视图生成
```

是两个不同职责。

---

# 十八、Tracer：Agent“刚才是怎么跑的”

Tracer 记录：

```text
谁
什么时候
耗时
成功还是失败
父节点是谁
```

例如：

```text
supervisor 300ms
  writer 200ms
    critic 150ms
```

于是可以定位：

```text
哪里慢？
哪里炸？
谁调用谁？
```

---

# 十九、Audit 和 Trace 再区分一次

这是以后非常容易被问到的问题。

```text
Audit
 ↓
“谁做了什么？”
```

```text
Trace
 ↓
“系统是怎么跑的？”
```

例如：

```text
Audit:
agent → delete_file → approval_granted

Trace:
supervisor 100ms
  writer 500ms
  critic 200ms
```

所以：

```text
Audit = 安全 / 合规

Trace = 性能 / 排障
```

---

# 二十、Facade：为什么又套一个 Harness？

如果直接让业务代码：

```python
registry = ...
gate = ...
session = ...
compactor = ...
tracer = ...
```

每个地方都自己组装。

以后会变成：

```text
业务代码
到处 new 五个模块
```

所以：

```python
Harness(...)
```

统一装配。

它不是：

> 第六个功能模块。

而是：

> **五个模块的装配入口。**

这就是 Facade。

---

# 二十一、Facade 最重要的两个设计

### 第一：模块之间不要互相乱 import

理想结构：

```text
Registry
Gate
Session
Compactor
Tracer
   ↓
 Harness
```

而不是：

```text
Registry → Gate
Gate → Session
Session → Tracer
Tracer → Registry
```

否则容易循环依赖。

---

### 第二：LLM 从外面注入

不要：

```python
class Harness:
    def __init__():
        self.llm = ChatOpenAI(...)
```

而是：

```python
Harness(llm=llm)
```

这样：

```text
DeepSeek
 ↓
换模型
```

只需要改装配层。

这就是依赖注入。

---

# 二十二、把三条线合在一起

现在可以画出 W7 真正的系统：

```text
                     用户
                      │
                      ▼
                ┌─────────────┐
                │ Supervisor  │
                └──────┬──────┘
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
     Researcher      Writer      Reviewer
                         │
                         ↓
                  Reflection
                         │
              ┌──────────┴──────────┐
              ↓                     ↓
           Critic                 Reviser
              │                     │
              └──────────→──────────┘


             所有运行都依赖 Harness

 ┌─────────────────────────────────────────┐
 │                 Harness                │
 │                                         │
 │ Registry → Gate → Executor              │
 │                                         │
 │ Session → Compactor                     │
 │                                         │
 │ Tracer                                  │
 └─────────────────────────────────────────┘
```

这时候 W7 的架构就真正清楚了。

---

# 二十三、这也是为什么 Day6 才真正困难

因为 Day2：

```text
Supervisor 自己能跑
```

Day3：

```text
Reflection 自己能跑
```

Day4：

```text
Registry/Gate 自己能跑
```

Day5：

```text
Session/Compactor/Tracer 自己能跑
```

但是：

```text
Supervisor
+
Reflection
+
Harness
```

真正接起来的时候，突然出现：

```text
消息格式不同
State 不同
调用接口不同
依赖不同
```

这就是 Day6。

所以：

> **“单独能跑”不代表“系统能组合”。**

这其实是 W7 最工程化的一课。