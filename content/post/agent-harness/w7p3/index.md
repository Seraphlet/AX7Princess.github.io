---
description: ""
title: "W7-Day6：把所有零件装成一台机器——完整调用链"
draft: false
date: "2026-09-21T14:41:20+08:00"
slug: "W7P3"
categories:
 - Harness
 - Agent
tags:
 - 
image: ""
---

# W7-Day6：把所有零件装成一台机器——完整调用链

前两篇已经把 W7 的知识点拆开了。

这一篇只做一件事情：

> **顺着代码真正走一遍。**

因为我发现自己阅读复杂代码时，最大的困难不是看不懂某个函数，而是：

> **不知道这个函数为什么在这里，它是谁调用的，调用完又把结果交给谁。**

所以这一次不从文件开始，而是从一次真实运行开始。

---

# 一、先看一次完整运行

用户输入：

```text
写一篇 300 字介绍 Agent 记忆系统的文章，要有事实依据
```

整个系统大致这样跑：

```text
main()
  ↓
创建 LLM
  ↓
build_harness()
  ↓
创建五模块
  ↓
build_supervisor()
  ↓
创建 supervisor + workers
  ↓
build_graph()
  ↓
graph.stream()
  ↓
START
  ↓
supervisor
  ↓
researcher
  ↓
supervisor
  ↓
writer
  ↓
Reflection
  ↓
critic
  ↓
reviser
  ↓
critic
  ↓
返回 ReflectionResult
  ↓
writer 交活
  ↓
supervisor
  ↓
reviewer
  ↓
supervisor
  ↓
FINISH
```

这张图是 Day6 最应该掌握的东西。

---

# 二、入口：`main()`

最终整合程序的入口是：

```python
def main():
```

先做：

```python
llm = ChatOpenAI(...)
```

这里创建的是整个系统共用的 LLM。

然后：

```python
h = build_harness(llm)
```

注意这个调用。

它意味着：

```text
main
 ↓
build_harness
 ↓
Harness
```

---

# 三、`build_harness()` 到底做了什么？

这里不是业务流程。

它做的是：

> **装配基础设施。**

大概：

```text
build_harness
    │
    ├── ToolRegistry
    │
    ├── PermissionGate
    │
    ├── SessionStore
    │
    ├── ContextCompactor
    │
    ├── Tracer
    │
    └── Harness
```

所以：

```python
h = build_harness(llm)
```

得到的：

```text
h
```

不是一个 Agent。

而是：

> **Agent 的运行环境。**

---

# 四、Registry 在这里注册了什么？

示例里注册两个工具：

```text
search_web
publish_article
```

其中：

```text
search_web
    level = safe

publish_article
    level = dangerous
```

于是：

```text
Registry
    │
    ├── search_web
    │
    └── publish_article
```

接着：

```python
PermissionGate(reg, audit_path=...)
```

建立：

```text
Registry
    ↓
Gate
```

---

# 五、Session / Compactor / Tracer 在这里被装配

```python
SessionStore(...)
ContextCompactor(...)
Tracer(...)
```

最终：

```python
Harness(
    registry=reg,
    gate=gate,
    session=session,
    compactor=compactor,
    tracer=tracer,
    llm=llm
)
```

这就是整个 Harness 的装配点。

以后业务代码拿：

```python
h
```

就能访问：

```text
h.registry
h.gate
h.session
h.compactor
h.tracer
h.llm
```

---

# 六、然后进入 `build_supervisor()`

这里开始真正创建 Agent 流程。

首先：

```python
Route
```

定义 Supervisor 可以输出：

```text
planner
researcher
writer
reviewer
FINISH
```

然后：

```python
router = llm.with_structured_output(...)
```

于是：

```text
LLM
 ↓
Route
```

---

# 七、`build_supervisor()` 又创建了几个节点？

可以先不看函数细节。

直接看返回值：

```text
supervisor
planner
researcher
writer
reviewer
```

所以这里实际上完成：

```text
节点工厂
 ↓
节点集合
```

然后 `build_graph()` 再把这些节点连起来。

---

# 八、为什么 `make_simple()` 是一个函数工厂？

例如：

```python
def make_simple(name, prompt):
```

它返回：

```python
def node(state):
```

所以：

```text
make_simple("planner", prompt)
```

得到：

```text
planner node
```

而：

```text
make_simple("researcher", prompt)
```

得到：

```text
researcher node
```

本质上：

```text
输入角色配置
 ↓
生成对应节点函数
```

所以这不是多了一层“炫技”。

而是在消除重复代码。

---

# 九、真正特殊的是 `writer_node`

因为：

```text
planner
researcher
reviewer
```

都是普通 worker。

但：

```text
writer
```

里面又嵌套了一套 Reflection。

也就是说：

```text
Supervisor 图
    │
    └── writer
          │
          ▼
     Reflection 图
          │
       generator
          ↓
        critic
          ↓
       reviser
          ↓
        critic
```

这就是 Day6 最复杂的一层嵌套。

---

# 十、为什么这里不能直接把 Reflection 的 State 暴露出来？

Reflection 有：

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

而 Supervisor 有：

```text
messages
next
rounds
```

如果直接把两套 State 合起来：

```text
SuperState + RState
```

马上就会产生耦合。

于是 Day6 引入：

```text
reflection_api.py
```

---

# 十一、`ReflectionResult` 就是两个系统之间的“接口”

它只有：

```python
@dataclass
class ReflectionResult:
    draft: str
    score: int
    rounds: int
    passed: bool
    notes: list
```

也就是说：

```text
Reflection 内部
    8 个字段
       ↓
Reflection API
       ↓
5 个字段
       ↓
Supervisor
```

这就是：

> **窄接口。**

---

# 十二、窄接口到底解决什么？

假设未来 Reflection 内部：

```text
revise_count
```

改成：

```text
revision_count
```

如果 Supervisor 直接访问：

```python
state["revise_count"]
```

那么 Supervisor 也要改。

但现在：

```text
Supervisor
    ↓
write_with_reflection()
    ↓
Reflection 内部
```

Supervisor 根本不知道内部字段叫什么。

所以：

> **内部实现可以变化，外部调用契约不变。**

这就是封装真正有价值的地方。

---

# 十三、`write_with_reflection()` 内部到底发生什么？

调用：

```python
write_with_reflection(topic)
```

进入：

```text
build / 注入 Reflection Graph
        ↓
graph.invoke(...)
```

传进去的是：

```python
{
    "topic": topic,
    "draft": "",
    "score": 0,
    "issues": [],
    "revise_count": 0,
    "trajectory": [],
    "best_draft": "",
    "best_score": 0
}
```

也就是说：

> Supervisor 不需要知道这些字段。

这些字段全部属于 Reflection 内部。

---

# 十四、Reflection Graph 内部再次运行

```text
START
 ↓
generator
 ↓
critic
 ↓
route_after_critic
```

如果不达标：

```text
reviser
 ↓
critic
```

继续循环。

最终：

```text
Reflection Graph
 ↓
最终 State
```

---

# 十五、然后 API 做一件很重要的事情：重新判定

这里特别值得注意。

它没有直接：

```python
passed = out["passed"]
```

而是自己计算：

```text
fatal issue?
score >= pass_score?
revise_count >= max_revise?
```

然后：

```text
程序
 ↓
ReflectionResult
```

所以：

> **模型负责评价，程序负责最终判定。**

---

# 十六、为什么返回 `best_draft` 而不是最后一稿？

例如：

```text
第一稿 6 分
第二稿 8 分
第三稿 7 分
```

最终：

```text
draft = 第三稿
best_draft = 第二稿
```

因此 API 返回：

```text
best_draft
+
best_score
```

而不是：

```text
最后一次 draft
+
最后一次 score
```

否则会出现：

```text
稿子是第二稿
分数却是第三稿
```

这种状态不一致。

所以这里其实还有一个重要工程原则：

> **相关字段必须保持一致的语义。**

---

# 十七、回到 `writer_node`

Reflection 返回：

```python
ReflectionResult
```

然后 writer 把它重新变成 Supervisor 世界里的消息：

```text
[writer 交活]
终稿内容
[内部: 2轮/8分/pass=True]
```

然后：

```python
return {
    "messages": [...]
}
```

注意这里发生了一次非常重要的转换：

```text
Reflection State
      ↓
ReflectionResult
      ↓
Supervisor Message
```

这就是系统之间的边界。

---

# 十八、第二条边界：消息格式

还有一个更隐蔽的问题。

Supervisor / LangGraph 使用：

```text
HumanMessage
AIMessage
SystemMessage
ToolMessage
```

但是 Harness 的：

```text
Session
Compactor
```

使用的是：

```python
{
    "role": "user",
    "content": "..."
}
```

所以出现：

```text
LangChain Message Object
        ↓
       ❌
       ↓
dict
```

如果直接：

```python
m.get("role")
```

而：

```text
m
```

其实是：

```text
HumanMessage
```

就会报：

```text
AttributeError
```

---

# 十九、于是 Day6 又增加了 `msg_adapter.py`

它做的事情非常简单：

```text
LangChain Message
        ↓
     normalize
        ↓
内部统一 dict
```

例如：

```text
HumanMessage
    ↓
{"role": "user", ...}

AIMessage
    ↓
{"role": "assistant", ...}

ToolMessage
    ↓
{"role": "tool", ...}
```

于是：

```text
外部世界
LangChain Message

        ↓

      Adapter

        ↓

内部世界
dict
```

---

# 二十、为什么不让 Harness 直接支持所有 Message 类型？

当然可以。

但是那样会导致：

```text
Session
    认识 HumanMessage
    认识 AIMessage
    认识 ToolMessage

Compactor
    也认识 HumanMessage
    也认识 AIMessage
    ...

Tracer
    ...
```

最终：

> 每个模块都知道 LangChain。

耦合就越来越高。

而 Adapter 的方案：

```text
LangChain
   ↓
Adapter
   ↓
内部统一格式
   ↓
Harness 五模块
```

所以 Harness 不需要知道外面的消息框架。

这就是 Day6 的第二条重要边界。

---

# 二十一、现在整个调用链可以完整画出来了

```text
用户输入
   │
   ▼
graph.stream()
   │
   ▼
supervisor
   │
   ├── 读取 messages
   │
   ├── LLM 判断 next
   │
   └── Route
        │
        ▼
   conditional edge
        │
        ├──────── researcher
        │              │
        │              └── search_web
        │
        ├──────── writer
        │              │
        │              ▼
        │        write_with_reflection()
        │              │
        │              ▼
        │        Reflection Graph
        │              │
        │       ┌──────┴──────┐
        │       ↓             ↓
        │    generator      critic
        │                     │
        │                     ↓
        │                  reviser
        │                     │
        │                     └──→ critic
        │
        └──────── reviewer
                       │
                       ▼
                   supervisor
                       │
                 FINISH / 再循环
```

而旁边始终有：

```text
             Harness
                │
     ┌──────────┼──────────┐
     ↓          ↓          ↓
 Registry      Gate      Session
     │                     │
 Executor             Compactor
                           
             Tracer
```

---

# 二十二、Tracer 是怎么插进调用链的？

Day6 使用：

```python
@h.tracer.trace("supervisor")
def supervisor_node(...):
```

于是：

```text
进入 supervisor
 ↓
Tracer 创建 span
 ↓
执行 supervisor
 ↓
记录耗时
 ↓
返回原结果
```

最关键的是：

> **装饰器必须把原函数返回值继续返回。**

否则：

```python
supervisor_node()
```

原本应该：

```python
return {"next": ...}
```

结果装饰器返回：

```python
None
```

LangGraph 就炸了。

所以 Tracer 不是业务逻辑。

它是：

> **包在业务逻辑外面的观察层。**

---

# 二十三、为什么 Day5 的 Tracer Bug 很重要？

之前出现过：

```text
父 span 一个 ID
子 span 指向另一个 ID
```

最终：

```text
trace tree
```

为空。

这个 bug 很危险，因为：

```text
程序不一定报错
```

但是：

```text
观测数据已经失真
```

所以：

> **可观测性系统自己也必须被测试。**

不能因为：

```text
Tracer 是测试工具
```

就不测试 Tracer。

---

# 二十四、Session 又在哪里真正生效？

理论上：

```text
SessionStore
```

存在还不够。

必须真的：

```text
append
 ↓
writer_sessions.jsonl
```

所以 Day6 验收强调：

```text
文件真的长出来
```

这就是：

> **“接入”和“生效”的区别。**

---

# 二十五、Compactor 怎么证明自己真的工作？

不能只写：

```python
ContextCompactor(...)
```

而应该看到：

```text
compactions > 0
```

甚至：

```text
原始：
81 条

压缩后：
9 条
```

而且：

```text
Session 中原始 81 条
仍然存在
```

所以：

```text
存储层
81 条

模型视图
9 条
```

这是最完整的证明。

---

# 二十六、Gate 怎么证明真的生效？

例如：

```text
publish_article
```

是危险工具。

正常路径：

```text
调用
 ↓
Gate
 ↓
NEED_APPROVAL
 ↓
人工批准
 ↓
Executor
 ↓
真正执行
```

拒绝路径：

```text
调用
 ↓
Gate
 ↓
NEED_APPROVAL
 ↓
人工拒绝
 ↓
Executor 不执行
```

最终应该同时看到：

```text
audit.log
```

以及真实副作用：

```text
批准 → 文件出现
拒绝 → 文件没有出现
```

所以：

> **安全机制必须验证“阻止了什么”，而不仅仅是“返回了什么”。**

---

# 二十七、Day6 最重要的测试思想：故障注入

正常运行：

```text
reviewer
 ↓
“可以定稿”
 ↓
FINISH
```

这个证明不了 Reflection 一定有效。

因为：

> 可能只是这次恰好没触发修改。

所以故意加入：

```text
fault-reviewer
```

让 reviewer：

```text
永远说：
“不达标，第2段缺案例”
```

这样才能强制系统进入：

```text
reviewer
 ↓
reviser
 ↓
reviewer
 ↓
reviser
 ↓
...
```

然后验证：

```text
真的修改了
```

以及：

```text
没有超过 MAX_REVISE
```

这就是：

> **故障注入测试。**

---

# 二十八、为什么“绿色测试”还不够？

Day5 有：

```text
21 个测试
```

但最有价值的是后来做的：

> **自杀测试 / Mutation Test。**

故意把 bug 种回去：

```text
删掉 safe_cut
    ↓
对应测试必须红

span_id 分配两次
    ↓
嵌套测试必须红

坏行修复删掉
    ↓
Session 测试必须红

装饰器不 return
    ↓
return-value 测试必须红
```

于是得到：

```text
4 个 bug
 ↓
4 个对应测试
 ↓
4/4 全部抓住
```

这才证明：

> **测试不是“跑起来是绿的”，而是“真的有牙”。**

---

# 二十九、这也是我这一周对“验证设计”的最大理解

以前可能会这样写测试：

```python
assert result == expected
```

现在开始发现：

> **测试首先要证明自己的前提成立。**

例如：

```python
assert delivered
```

为什么？

因为如果：

```python
delivered = []
```

后面的：

```python
for m in delivered:
```

一句都不会执行。

测试可能莫名其妙通过。

所以：

> **空集合测试是非常危险的。**

应该先：

```text
证明数据存在
 ↓
再证明数据满足条件
```

这就是这周不断出现的：

> **“先证明测试有牙。”**

---

# 三十、最后重新看 W7，发现它其实是在训练四层能力

### 第一层：编排

```text
Supervisor
Conditional Edge
Worker
```

解决：

> Agent 怎么协作？

---

### 第二层：反馈

```text
Critic
Reviser
Reflection
```

解决：

> Agent 怎么自我检查？

---

### 第三层：运行时基础设施

```text
Registry
Gate
Session
Compactor
Tracer
```

解决：

> Agent 怎么真正运行？

---

### 第四层：工程验证

```text
Unit Test
Integration Test
Fault Injection
Mutation Test
Evidence
```

解决：

> 我怎么知道它真的工作？

---

# 三十一、所以 W7 真正的闭环是

```text
              ┌───────────────┐
              │   Agent 决策   │
              └───────┬───────┘
                      ↓
                多 Agent 协作
                      ↓
                   Reflection
                      ↓
                 产生更好结果
                      ↓
                Harness 执行
                      ↓
          ┌───────────┼───────────┐
          ↓           ↓           ↓
        Tools       State       Trace
          ↓           ↓           ↓
        Gate       Session      Audit
          │           │
          └──────┬────┘
                 ↓
              最终结果
                 ↓
               Tests
                 ↓
          证明它真的工作
```

---

# 三十二、如果让我用一句话总结 W7

以前我可能会说：

> W7 学了 Supervisor、Reflection 和 Harness。

现在我会说：

> **W7 是第一次把“Agent 的思考逻辑”和“Agent 的运行基础设施”放到同一个系统里考虑。**

其中：

```text
Supervisor
解决“谁来做”

Reflection
解决“做得够不够好”

Registry
解决“有什么能力”

Gate
解决“能不能使用能力”

Executor
解决“到底有没有执行”

Session
解决“过去发生了什么”

Compactor
解决“现在需要看到什么”

Tracer
解决“刚才是怎么运行的”

Facade
解决“这些零件怎么装起来”
```

而 Day6 真正解决的是：

```text
这些东西之间
怎么连接
怎么转换
怎么验证
```

所以我现在再看一个复杂 Agent 项目，应该不再只是找：

```text
Agent
Tool
LLM
```

而应该主动寻找：

```text
入口
 ↓
State
 ↓
Control Flow
 ↓
Data Flow
 ↓
模块边界
 ↓
调用契约
 ↓
执行入口
 ↓
持久化
 ↓
上下文
 ↓
观测
 ↓
测试证据
```

这才是这一周真正想让我学会的东西。