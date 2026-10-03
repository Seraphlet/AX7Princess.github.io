---
description: ""
title: "子图与 Send"
draft: false
date: "2026-09-12T04:13:55+08:00"
slug: "langraph-subagnet"
categories:
 - langgraph
tags:
 - Send
 - SubAgent
image: ""
---

# 子图与 Send

## 把图当零件，把任务动态派出去

这一天学的是 W6-D5，主题只有两个：

> **子图（Subgraph）**：把一整张图当成一个零件，装进另一张图。
> **Send**：一个节点根据运行时的数据，动态派出 N 份独立任务。

一开始看起来这两个知识点没有太大关系，但实际实验下来，它们都在解决同一个工程问题：

> **当一个 LangGraph 开始变复杂时，怎么把复杂性拆开，同时又让任务能够动态扩展？**

子图负责**拆复杂度**，Send 负责**扩任务**。

---

# 一、子图：把一整张图当成一个节点

以前学 LangGraph 时，直觉上会把节点理解成：

```text
START → nodeA → nodeB → nodeC → END
```

每个节点就是一项工作。

但当逻辑越来越复杂时，一个节点里面可能又有很多步骤。这个时候，可以把一整张小图封装起来：

```text
父图
START
  ↓
entry
  ↓
inner
  ↓
parent_end
  ↓
END
```

其中：

```text
inner
```

并不是普通函数，而是一整张已经编译好的子图。

因此可以把子图理解成：

> **把一台复杂机器先独立制造、独立测试，然后把它作为一个标准零件装进另一台机器。**

这样做有三个直接好处：

### 1. 复用

一张子图可以被多个地方使用。

### 2. 隔离

子图拥有自己的内部 State 和执行过程，父图不需要知道它内部每一步怎么做。

### 3. 分层

复杂逻辑可以收进子图，父图只需要关心：

> “这个零件输入什么，输出什么？”

---

# 二、一个最小的子图实验

先单独定义子图：

```python
class SubState(TypedDict):
    text: str
    result: str


def shout(state: SubState) -> dict:
    return {"result": state["text"].upper() + "!"}


sub_b = StateGraph(SubState)
sub_b.add_node("shout", shout)
sub_b.add_edge(START, "shout")
sub_b.add_edge("shout", END)

subgraph = sub_b.compile()
```

这里最重要的一点是：

```python
subgraph = sub_b.compile()
```

编译之后，`subgraph` 才可以作为父图里的节点使用。

父图：

```python
class ParentState(TypedDict):
    text: str
    result: str
    final: str


def entry(state: ParentState) -> dict:
    return {"text": state["text"].strip()}


def parent_end(state: ParentState) -> dict:
    return {"final": "父图收到: " + state["result"]}


p = StateGraph(ParentState)

p.add_node("entry", entry)
p.add_node("inner", subgraph)
p.add_node("parent_end", parent_end)

p.add_edge(START, "entry")
p.add_edge("entry", "inner")
p.add_edge("inner", "parent_end")
p.add_edge("parent_end", END)

parent = p.compile()
```

真正把子图装进去的只有这一行：

```python
p.add_node("inner", subgraph)
```

所以可以记成：

```text
父图
START
  ↓
entry
  ↓
┌───────────────┐
│    inner      │   ← 整张子图
│      ↓        │
│    shout      │
└───────────────┘
  ↓
parent_end
  ↓
END
```

---

# 三、子图不是“看起来像一层”，而是真的有层级

这次实验里最值得注意的，不是代码能跑，而是怎么证明：

> **inner 真的包含了 shout，而不是只是在父图里有一个叫 inner 的普通节点。**

开启：

```python
subgraphs=True
```

之后，拿到的事件中出现了类似：

```text
(
    ('inner:e7b812b0-ca7d-1088-81ae-c15b84d74fd3',),
    {'shout': {'result': 'HI!'}}
)
```

这里有两个非常关键的证据。

第一：

```text
('inner:xxxx',)
```

说明当前事件带有子图的命名空间。

第二：

```python
{'shout': {'result': 'HI!'}}
```

`shout` 是子图内部节点的名字。

所以：

> **“非空命名空间 + 子图内部独有的节点名”**

比简单地判断：

```python
"inner" in str(event)
```

更有证明力。

---

# 四、这次让我意识到：断言“绿了”不等于验证成功

这是 D5 一个比代码本身更重要的收获。

比如之前写：

```python
assert any("inner" in str(ev) for ev in raw)
```

它确实可能变绿。

但这个断言的问题是：

```text
inner
```

本来就是我自己给父图节点起的名字。

所以即使子图内部的 `shout` 完全没有暴露出来，这个断言也可能变绿。

也就是说：

> **断言通过，不代表它证明了我要证明的东西。**

这是一种典型的“假绿”。

真正好的验证应该问：

> **如果实现是错的，这条断言还有可能通过吗？**

如果答案是“有可能”，那这条断言就还不够强。

---

# 五、我给自己总结了一条“判据强度阶梯”

```text
精确值比较
    ↓
结构化集合比较
    ↓
计数 / 长度
    ↓
子串包含
    ↓
只判断非空
```

例如：

```python
assert result == "HELLO!"
```

通常比：

```python
assert "HELLO" in str(result)
```

更有证明力。

又比如并行结果：

```python
assert sorted(actual) == sorted(expected)
```

比：

```python
assert actual == expected
```

更合理。

因为并行执行下：

> **完成顺序通常不能成为业务正确性的判断条件。**

所以这次 D5 又强化了一个以后写测试时很重要的习惯：

> **不要问“这条断言能不能绿”，而要问“它能排除哪些错误实现”。**

---

# 六、Send：一个节点动态派 N 份活

子图解决的是“拆”。

Send 解决的是“派”。

假设有：

```python
topics = [
    "RAG",
    "Agent记忆",
    "多Agent",
    "MCP",
]
```

目标是：

> 每个 topic 都交给一个 worker 单独处理。

这时候可以：

```python
Send("worker", {"topic": t})
```

再配合：

```python
for t in state["topics"]
```

动态产生：

```text
Send("worker", {"topic": "RAG"})
Send("worker", {"topic": "Agent记忆"})
Send("worker", {"topic": "多Agent"})
Send("worker", {"topic": "MCP"})
```

最终形成：

```text
                 ┌── worker(RAG) ──────┐
                 │                     │
topics ── router ─┼── worker(Agent记忆) ─┤
                 │                     ├──→ join
                 ├── worker(多Agent) ───┤
                 │                     │
                 └── worker(MCP) ──────┘
```

这里最关键的一点：

> **N 是运行时由数据决定的。**

不是写死：

```text
A → B
A → C
A → D
```

而是：

```text
topics 有几条
        ↓
就派几份任务
```

所以可以用一句话记：

> **普通条件边更像“固定分岔”，Send 更像“看订单有多少行就派多少工人”。**

---

# 七、Send 最容易理解错的地方：worker 的 state 到底是什么？

这是我这次真正卡住的地方。

假设主 State：

```python
class MapState(TypedDict):
    topics: list[str]
    results: Annotated[list[str], add]
    summary: str
```

这里定义的是：

```text
topics
results
summary
```

那么问题来了：

为什么 worker 可以写：

```python
topic = state["topic"]
```

明明 `MapState` 根本没有：

```text
topic
```

这个字段？

答案是：

> **Send 并不是给原来的 MapState 增加一个 `topic` 字段。**

这是整个 Send 机制最容易混淆的地方。

---

# 八、Send 不修改主 State，而是给一次节点执行提供输入

例如：

```python
Send("worker", {"topic": "RAG"})
```

这里：

```python
{"topic": "RAG"}
```

不是：

```python
{
    "topics": [...],
    "results": [...],
    "summary": ...,
    "topic": "RAG"
}
```

它也不会跑进：

```python
state["topics"]
```

里面。

更准确的理解是：

```text
主 State
{
    topics: [...]
    results: [...]
    summary: ...
}
        │
        │ router 计算
        ↓
Send
        │
        ├── 任务 1 → {"topic": "RAG"}
        ├── 任务 2 → {"topic": "Agent记忆"}
        ├── 任务 3 → {"topic": "多Agent"}
        └── 任务 4 → {"topic": "MCP"}
```

到了 worker：

```python
def worker(state: dict) -> dict:
    topic = state["topic"]
```

这里的 `state` 指的是：

> **这一次 Send 任务传给 worker 的输入字典。**

也就是：

```python
{"topic": "RAG"}
```

而不是原来的：

```python
MapState
```

---

# 九、于是两个 `state` 其实根本不是一个东西

这是以后看 LangGraph 代码时非常值得警惕的一点。

例如：

```python
def route_to_workers(state: MapState):
    ...
```

这里的：

```python
state
```

是父图当前的完整 State。

所以：

```python
state["topics"]
```

可以取到。

但：

```python
def worker(state: dict):
    ...
```

这里的：

```python
state
```

是 Send 提供给这一次 worker 执行的数据。

所以：

```python
state["topic"]
```

可以取到。

虽然变量都叫：

```python
state
```

但它们不是同一个东西。

以后为了防止自己混淆，我甚至可以主动写成：

```python
def route_to_workers(map_state: MapState):
    return [
        Send("worker", {"topic": t})
        for t in map_state["topics"]
    ]


def worker(worker_input: dict):
    topic = worker_input["topic"]
```

这样从变量名上就能直接看出来：

```text
map_state
   ↓
主 State

worker_input
   ↓
某一次 Send 的输入
```

---

# 十、Send 的 payload 是“任务输入”，不是 State 字段更新

所以这一条应该牢牢记住：

```python
Send("worker", {"topic": t})
```

里面：

```python
{"topic": t}
```

是：

> **这个 worker 本次执行要拿到的数据。**

而不是：

> **给主 State 增加一个 topic 字段。**

这也是为什么 worker 想拿什么上下文，就必须在 Send 时考虑进去。

例如 worker 需要：

```text
topic
user_id
检索范围
某些配置
```

那么这些需要的东西就要在任务输入里体现出来。

---

# 十一、为什么 Send 和普通边不是一回事

这次还踩到了一个很典型的坑。

最开始容易想写：

```python
g.add_edge("distributor", "worker")
```

然后希望：

```python
distributor
    ↓
Send(...)
    ↓
worker
```

但这混淆了两个职责。

可以先用一个非常实用的心智模型理解：

> **Node 负责干活，路由负责决定下一步去哪。**

普通节点返回的是：

```python
dict
```

例如：

```python
return {"results": [r]}
```

这是：

> “我做完了，这些是我对 State 的更新。”

而路由函数返回：

```python
list[Send]
```

代表的是：

> “下一步要派这些任务出去。”

所以最终形成：

```text
Node
 ↓
返回数据
 ↓
State 更新
```

而：

```text
Router
 ↓
返回 Send
 ↓
动态产生下一批任务
```

这两个职责不能混为一谈。

---

# 十二、推荐的 Send 结构

因此最终使用的是：

```python
def distributor(state: MapState) -> dict:
    print(f"[distributor] 扇出 {len(state['topics'])} 份活")
    return {}
```

真正负责动态派工的是：

```python
def route_to_workers(state: MapState) -> list[Send]:
    return [
        Send("worker", {"topic": t})
        for t in state["topics"]
    ]
```

构图：

```python
g.add_edge(START, "distributor")

g.add_conditional_edges(
    "distributor",
    route_to_workers,
    ["worker"],
)

g.add_edge("worker", "join")
g.add_edge("join", END)
```

这里的职责就非常清楚：

```text
distributor
    │
    │ 干准备工作
    ↓
route_to_workers
    │
    │ 动态计算 N
    ↓
Send × N
    │
    ↓
worker × N
    │
    ↓
reducer
    │
    ↓
join
```

---

# 十三、Reducer：并行结果怎么重新汇总？

4 个 worker 会分别产生：

```python
{"results": ["RAG 完成"]}
{"results": ["Agent记忆 完成"]}
{"results": ["多Agent 完成"]}
{"results": ["MCP 完成"]}
```

如果 `results` 是普通字段，多份更新会发生冲突。

所以定义：

```python
results: Annotated[list[str], add]
```

意思可以先理解成：

> **多个 worker 更新这个字段时，用 `add` 把它们合并。**

最终得到：

```python
[
    "RAG 完成",
    "Agent记忆 完成",
    "多Agent 完成",
    "MCP 完成",
]
```

因此 Send 这条链的完整逻辑是：

```text
一份 State
    ↓
动态扇出
    ↓
N 份独立任务
    ↓
N 个 worker
    ↓
N 个结果
    ↓
Reducer 汇聚
    ↓
一个统一结果
```

这实际上就是经典的：

> **Map → Reduce**

---

# 十四、一个很容易踩的 Reducer 坑

`results` 已经有：

```python
Annotated[list[str], add]
```

所以 join 不应该再写：

```python
return {"results": state["results"]}
```

因为这相当于：

> 已经有这一批结果了，我再把同一批结果交一次。

于是可能变成：

```text
4 条
 ↓
再加一次自己
 ↓
8 条
```

因此 join 只需要读取：

```python
state["results"]
```

然后产出另外一个字段：

```python
return {
    "summary": " | ".join(sorted(state["results"]))
}
```

---

# 十五、为什么并行结果不能直接比较顺序？

这次实验还暴露了另一个很重要的测试问题。

错误倾向：

```python
assert out["results"] == expected
```

因为 worker 是并行运行的。

哪一个 worker 先完成，并没有业务上的必要保证。

所以更合理的是：

```python
assert sorted(out["results"]) == sorted(expected)
```

这表达的意思是：

> **我只关心有没有这四个结果，不关心谁先回来。**

这也是一个很重要的工程原则：

> **不要把没有业务意义的执行顺序写进断言。**

否则今天绿，明天可能突然红，而代码实际上完全没坏。

---

# 十六、并行到底有没有真的发生？不能靠“看起来快”

这次实验故意让：

```python
def do_work(topic):
    time.sleep(1)
    return f"{topic} 完成"
```

有 4 个任务。

串行：

```text
1s + 1s + 1s + 1s
≈ 4s
```

并行：

```text
任务1 ─┐
任务2 ─┤
任务3 ─┤ → 大约同时等待 1s
任务4 ─┘
```

所以实测：

```text
串行 ≈ 4.00s
并行 ≈ 1.02s
加速比 ≈ 3.93x
```

这比一句：

> “看起来快很多。”

有意义得多。

---

# 十七、一个很重要的测试设计教训：门槛也需要验证

之前使用：

```python
assert speedup > 2.0
```

看起来合理，但实际跑出来：

```text
3.93x
```

于是发现：

> `2.0x` 这个门槛其实太松。

假如实际只跑出了：

```text
2.1x
```

这个断言照样会绿。

但 4 个任务的理想情况明明接近：

```text
4x
```

所以：

> **一个测试阈值不能只是“差不多够用”，还要问它能不能排除明显的错误实现。**

这和前面的：

```python
"inner" in str(event)
```

其实是同一种问题。

表面上是两个完全不同的 bug，底层却是同一种思维错误：

> **判据太松，错误实现也可能通过。**

---

# 十八、因此我现在更倾向于这样设计并行实验

如果单任务耗时是自己控制的：

```python
time.sleep(1)
```

而且任务数量是：

```text
4
```

那么理论上：

```text
串行 ≈ 4s
四路并行 ≈ 1s
```

此时直接检查并行耗时：

```python
assert par < 1.5
```

反而比：

```python
assert speedup > 2.0
```

更直接。

因为这里已经知道单任务耗时。

但如果是真实系统，任务耗时本身不确定，那么绝对值阈值就不够稳定，这时才更适合做：

```python
speedup = serial / parallel
```

再配合一个合理阈值。

所以：

> **测试阈值不是越严越好，而是在“误报”和“漏报”之间取舍。**

---

# 十九、这次实验还让我看到 Send 的真正价值

不要把 Send 简单记成：

> “Send 可以并行，所以更快。”

这个说法太粗。

真正重要的是：

> **Send 允许工作量 N 随运行时数据变化。**

例如：

```python
topics = ["RAG", "MCP"]
```

派：

```text
2 个 worker
```

如果变成：

```python
topics = ["RAG", "MCP", "Memory", "Agent", "Tools"]
```

就可以动态派：

```text
5 个 worker
```

所以 Send 的核心价值不是“固定并行 4 个任务”。

而是：

> **数据有多少份工作，就动态生成多少份执行任务。**

这才是它和固定条件边真正拉开差距的地方。

---

# 二十、这次 D5 最终形成了一套统一心智模型

现在可以把整个 D5 压缩成这张图：

```text
                     主 State
                        │
                        │
                 ┌──────▼──────┐
                 │    Router   │
                 └──────┬──────┘
                        │
                  根据数据计算 N
                        │
             ┌──────────┼──────────┐
             ↓          ↓          ↓
         Send(A)     Send(B)    Send(C)
             ↓          ↓          ↓
          worker      worker     worker
             │          │          │
             └──────────┼──────────┘
                        ↓
                    reducer
                        ↓
                       join
```

而子图是另一条维度：

```text
父图
│
├── entry
│
├── inner
│    │
│    ├── shout
│    ├── ...
│    └── END
│
└── parent_end
```

所以可以用两句话记：

> **子图 = 把复杂逻辑封装成零件。**

> **Send = 把运行时出现的多份工作动态派出去。**

---

# 二十一、D5 最终复盘

这一天真正值得留下来的不是几个 API，而是下面这几层理解。

### 第一层：会用

知道：

```python
p.add_node("inner", subgraph)
```

可以把子图装进父图。

知道：

```python
Send("worker", {"topic": t})
```

可以动态创建任务。

---

### 第二层：知道数据怎么走

主 State：

```text
topics
results
summary
```

经过 router：

```text
topics
 ↓
Send payload
```

进入 worker：

```text
{"topic": "..."}
```

worker 完成：

```text
{"results": [...]}
```

Reducer：

```text
多个 results
 ↓
一个 results
```

join：

```text
results
 ↓
summary
```

---

### 第三层：理解 State 的边界

最容易产生错觉的是：

```python
state
```

这个名字。

但：

```python
router(state)
```

里的 `state`

和：

```python
worker(state)
```

里的 `state`

不一定是同一个 State。

**变量叫 state，不代表它一定是整张图的共享 State。**

在 Send 场景里：

```python
Send("worker", {"topic": t})
```

里的 payload 就是这次 worker 执行的输入数据。

所以：

```python
worker_input["topic"]
```

拿到的是 Send 给这一份任务的数据，而不是主 State 新增的字段。

---

### 第四层：真正开始理解“验证设计”

这次最重要的额外收获，可能甚至不是 LangGraph。

而是：

> **“程序跑通”和“我证明它正确”是两回事。**

例如：

```python
assert "inner" in str(ev)
```

程序可以绿。

但不代表真的证明了子图内部存在。

例如：

```python
assert speedup > 2.0
```

程序可以绿。

但不代表并发度足够。

所以以后写实验时，我需要问自己：

```text
我要证明什么？
        ↓
什么现象只能由“正确实现”产生？
        ↓
错误实现有没有可能也出现这个现象？
        ↓
如果能，当前判据就不够强。
```

这其实比“会写断言”更重要。

---

# 二十二、最后把 D5 压缩成一张脑图

```text
                    LangGraph D5
                         │
          ┌──────────────┴──────────────┐
          ↓                             ↓
        子图                            Send
          │                             │
    图 → 零件                     数据 → N 份任务
          │                             │
    独立构建/测试                     动态扇出
          │                             │
    装入父图                         worker × N
          │                             │
       隔离/分层                         ↓
          │                           reducer
          ↓                             │
     父图 ← 子图结果                     ↓
                                      join
```

最后只记三句话：

> **子图是封装复杂度。**

> **Send 是动态扩展执行任务。**

> **Reducer 是把并行产生的多个结果重新汇成一个 State。**

而贯穿整个实验的第四句话是：

> **“全绿”不是证据，能排除错误实现的判据才是证据。**
