---
description: ""
title: "Supervisor 模式：让一个 Agent 负责调度，多个 Worker 负责执行"
draft: false
date: "2026-09-15T07:16:45+08:00"
slug: "Supervisor"
categories:
 - LangGraph
 - Agent
tags:
 - Supervisor
image: ""
---

# W7-Day2｜Supervisor 模式：让一个 Agent 负责调度，多个 Worker 负责执行

昨天主要是在画 Supervisor 模式的图，今天开始真正把它写成代码。

今天的 Demo 是一个 Supervisor + 3 个 Worker 的多 Agent 流程：

```text
                    ┌──────────────┐
                    │  supervisor  │
                    │  决定下一步   │
                    └──────┬───────┘
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
        researcher       writer       reviewer
             │             │             │
             └─────────────┴─────────────┘
                           │
                           ▼
                       supervisor
                           │
                    FINISH → END
```

三个 Worker 分工：

```text
researcher → 查资料
writer     → 写文章
reviewer   → 审文章
```

最重要的不是“有三个 Agent”，而是：

> **Worker 负责执行，Supervisor 负责决定下一步。**

所以实际运行不是一条固定流水线：

```text
researcher → writer → reviewer
```

而是一个循环：

```text
supervisor
    ↓
researcher
    ↓
supervisor
    ↓
writer
    ↓
supervisor
    ↓
reviewer
    ↓
supervisor
    ↓
FINISH
    ↓
END
```

Worker 干完以后必须重新回到 Supervisor。

---

# 一、先看今天的 State

今天的 State：

```python
class SuperState(MessagesState):
    next: str
    rounds: int
```

在 `MessagesState` 自带的 `messages` 之外，又增加两个字段：

```text
messages → 工作内容和历史
next     → Supervisor 的下一步决策
rounds   → 已经循环多少次
```

这三个字段的职责并不一样。

### messages：数据流

它保存整个任务的上下文。

例如：

```text
用户：写一篇 LangGraph 介绍

researcher：
LangGraph 是……

writer：
LangGraph 是一个……
```

Worker 每次都是从这里读取前面的工作结果。

### next：控制流

例如：

```python
next = "writer"
```

表示：

> 下一步执行 writer。

所以 `next` 是给图的控制逻辑看的，而不是业务内容。

### rounds：安全控制

Supervisor 是循环：

```text
Supervisor → Worker → Supervisor → Worker → ...
```

所以必须记录轮数，防止模型一直循环。

---

# 二、为什么要有两个 LLM 变量

代码：

```python
llm = make_llm()
router_llm = make_llm()
```

当前它们使用的是同一个模型，但职责不同：

```text
llm
 ↓
Worker
 ↓
真正干活


router_llm
 ↓
Supervisor
 ↓
决定下一步
```

Worker 的任务是：

```text
查资料 / 写文章 / 审稿
```

Supervisor 的任务是：

```text
现在应该找谁？
```

虽然都是 LLM，但从系统设计上已经是两个不同角色。

---

# 三、MEMBERS：为什么专门维护一个名单

```python
MEMBERS = [
    "researcher",
    "writer",
    "reviewer"
]
```

这个列表相当于 Supervisor 的“花名册”。

后面很多代码都依赖它：

```text
Supervisor 能选择谁
        ↓
条件边能连接到谁
        ↓
Worker 的名字是否合法
        ↓
Worker 回边连接给谁
```

所以它最好成为：

> **单一事实来源。**

以后增加：

```python
"fact_checker"
```

就应该优先改这里，而不是到程序各处手写名字。

---

# 四、Route：Supervisor 怎么把“思考结果”交给程序

接下来：

```python
class Route(BaseModel):
    next: Literal[
        "researcher",
        "writer",
        "reviewer",
        "FINISH"
    ]

    reason: str = Field(
        default="",
        description="一句话理由, 方便调试"
    )
```

这里解决的是一个核心问题：

> **模型说完“下一步该谁”以后，程序怎么可靠地拿到这个答案？**

不能让模型随便返回一句：

```text
我觉得现在应该让 writer 处理一下……
```

而是约束成：

```json
{
    "next": "writer",
    "reason": "researcher 已经完成资料整理"
}
```

其中真正影响图运行的是：

```python
decision.next
```

而 `reason` 主要用于调试。

因此这里形成了一条链：

```text
LLM
 ↓
结构化输出
 ↓
Route
 ↓
decision.next
 ↓
State["next"]
 ↓
条件边
 ↓
真正执行哪个节点
```

---

# 五、为什么最后用 `json_mode`

最开始写：

```python
router_llm.with_structured_output(Route)
```

但实际跑 DeepSeek 时，默认路径触发了 `json_schema`，返回：

```text
This response_format type is unavailable now
```

后来又试了：

```python
method="function_calling"
```

结果变成：

```text
Thinking mode does not support this tool_choice
```

最后通过探针验证，当前环境真正可用的是：

```python
router = router_llm.with_structured_output(
    Route,
    method="json_mode"
)
```

所以今天这里真正学到的并不是“DeepSeek 不支持结构化输出”。

而是：

> **同一个结构化输出接口，底层可能走完全不同的协议路径。**

可以画成：

```text
with_structured_output
        │
        ├── json_schema
        │      ↓
        │    当前环境不支持
        │
        ├── function_calling
        │      ↓
        │    thinking mode 拒绝 tool_choice
        │
        └── json_mode
               ↓
             可用
```

所以以后遇到这类兼容问题，先写一个最小 Probe，确认**模型实际支持什么**，不要凭印象猜。

---

# 六、Worker 为什么用工厂函数创建

这部分源码初看有一点绕：

```python
def make_worker(name, tools, prompt):
    agent = create_react_agent(
        llm,
        tools,
        prompt=prompt
    )

    def node(state):
        ...
```

它其实是在做一件事：

> **把“创建 Worker”这件事封装成一个模板。**

然后：

```python
worker_nodes = {
    "researcher": make_worker(
        "researcher",
        [web_search],
        WORKER_PROMPTS["researcher"]
    ),

    "writer": make_worker(
        "writer",
        [],
        WORKER_PROMPTS["writer"]
    ),

    "reviewer": make_worker(
        "reviewer",
        [],
        WORKER_PROMPTS["reviewer"]
    ),
}
```

最终得到三个不同的 Worker：

```text
researcher
writer
reviewer
```

它们本质上结构类似，只是：

```text
名字不同
Prompt 不同
工具不同
```

所以这里的工厂函数主要是为了：

> **避免重复写三遍几乎一样的 Worker 创建代码。**

---

# 七、三个 Worker 为什么职责不同

Prompt：

```python
WORKER_PROMPTS = {
    "researcher": "你是研究员……",
    "writer": "你是写手……",
    "reviewer": "你是审稿人……",
}
```

工具：

```text
researcher → web_search
writer     → 无工具
reviewer   → 无工具
```

于是职责就很清楚：

```text
researcher
    ↓
产生资料

writer
    ↓
消费资料，产生文章

reviewer
    ↓
消费文章，产生审稿结果
```

这里实际上已经有一个很清楚的数据加工链：

```text
原始任务
   ↓
资料
   ↓
文章
   ↓
评审结果
```

Supervisor 只是负责控制这个加工链什么时候进入下一阶段。

---

# 八、真正执行 Worker 时，State 是怎么进去的

Worker 的核心代码：

```python
result = agent.invoke({
    "messages": state["messages"]
})
```

这里特别容易误解。

它不是：

```text
把整个 LangGraph State 原封不动塞给 Agent
```

而是：

```text
LangGraph State
      │
      └── state["messages"]
                 ↓
            Worker Agent
```

所以 Worker 真正拿到的核心数据是：

> **当前 State 中的消息历史。**

如果此时 State 是：

```text
messages:
    用户要求
    researcher 结果
```

那么 writer 就可以根据这些内容继续工作。

---

# 九、Worker 干完以后为什么还要重新包装一次消息

Worker 内部 Agent 最后得到：

```python
last = result["messages"][-1]
text = str(last.content)
```

接着：

```python
return {
    "messages": [
        HumanMessage(
            content=f"[{name} 交活]\n{text}",
            name=name
        )
    ]
}
```

这一步非常重要。

因为 Worker 内部的最终结果不能只停留在它自己的 Agent 里。

它必须进入 LangGraph 的 State。

于是：

```text
Worker 内部
    ↓
生成结果
    ↓
包装为 HumanMessage
    ↓
写回 State.messages
```

同时增加：

```python
name=name
```

于是 Supervisor 下一轮就知道：

```text
这是 researcher 交来的
```

而不是只能看到一条匿名消息。

---

# 十、这里可以第一次把“逻辑流”和“数据流”放在一起看

现在完整看一次 researcher：

### 逻辑流

```text
supervisor
    ↓
decision.next = researcher
    ↓
条件边
    ↓
researcher
```

这是：

> **决定谁执行。**

### 数据流

```text
State.messages
    ↓
researcher Agent
    ↓
web_search
    ↓
研究结果
    ↓
HumanMessage(name="researcher")
    ↓
写回 State.messages
```

这是：

> **决定执行时拿什么，执行后留下什么。**

两条流最后重新汇合：

```text
                     State
                ┌───────────────┐
                │ messages      │
                │ next          │
                │ rounds        │
                └───────┬───────┘
                        │
            ┌───────────▼───────────┐
            │      supervisor       │
            │       决定 next       │
            └───────────┬───────────┘
                        │
                     控制流
                        │
                        ▼
                      Worker
                        │
                     数据流
                        │
                        ▼
                  写回 messages
                        │
                        └────→ supervisor
```

这是今天我认为理解 Supervisor 最重要的一张图。

---

# 十一、Supervisor 节点到底在做什么

源码核心：

```python
def supervisor_node(state):
    rounds = state.get("rounds", 0) + 1

    if rounds > MAX_ROUNDS:
        return {
            "next": "FINISH",
            "rounds": rounds,
            ...
        }

    msgs = [
        SystemMessage(
            content=SUPERVISOR_PROMPT
        )
    ] + state["messages"]

    decision = router.invoke(msgs)

    return {
        "next": decision.next,
        "rounds": rounds,
    }
```

可以拆成四步。

### 第一步：计算轮数

```python
rounds += 1
```

### 第二步：检查是否超限

如果已经超限：

```text
直接 FINISH
```

连 LLM 都不再调用。

这是为了避免已经知道要结束了，还白花一轮模型费用。

### 第三步：把“Supervisor 的规则 + 当前消息历史”交给 LLM

```python
msgs = [
    SystemMessage(content=SUPERVISOR_PROMPT)
] + state["messages"]
```

模型看到：

```text
自己的职责
+
前面的工作历史
```

### 第四步：把模型决定写回 State

```python
return {
    "next": decision.next,
    "rounds": rounds,
}
```

所以 Supervisor 这一轮的作用其实非常简单：

> **读取 State → 决定下一步 → 修改 State。**

---

# 十二、条件边：模型只负责“说去哪”，图负责“真的去”

路由函数：

```python
def route_from_supervisor(state):
    nxt = state.get("next") or "FINISH"

    return (
        nxt
        if nxt in MEMBERS or nxt == "FINISH"
        else "FINISH"
    )
```

然后：

```python
builder.add_conditional_edges(
    "supervisor",
    route_from_supervisor,
    path_map
)
```

完整过程其实是：

```text
Supervisor LLM
       ↓
"writer"
       ↓
state["next"]
       ↓
route_from_supervisor()
       ↓
"writer"
       ↓
path_map
       ↓
writer 节点
```

所以一个非常值得记住的边界是：

> **LLM 做决策，Graph 执行决策。**

模型不是直接调用：

```python
writer()
```

它只是在 State 中留下：

```text
next = "writer"
```

然后 LangGraph 根据图结构真正执行 writer。

---

# 十三、为什么 Worker 必须回 Supervisor

这里：

```python
for name in MEMBERS:
    builder.add_edge(
        name,
        "supervisor"
    )
```

实际上是整个模式的循环核心。

形成：

```text
researcher ─┐
writer     ─┼──→ supervisor
reviewer   ─┘
```

如果没有这组回边：

```text
supervisor
    ↓
researcher
    ↓
结束
```

那么 Worker 做完以后，就没有人继续判断下一步。

所以 Supervisor 模式真正的图形是：

```text
Supervisor → Worker
      ↑        │
      └────────┘
```

直到：

```text
Supervisor → FINISH → END
```

---

# 十四、把一次完整运行串起来

现在假设模型做出了一个比较理想的路径：

```text
supervisor
→ researcher
→ supervisor
→ writer
→ supervisor
→ reviewer
→ supervisor
→ FINISH
```

状态变化可以粗略理解为：

### 初始

```text
messages:
    用户要求

next:
    ""

rounds:
    0
```

### Supervisor 第一次

```text
next = researcher
rounds = 1
```

### researcher 执行

```text
messages:
    用户要求
    researcher 交活
```

### Supervisor 第二次

读取：

```text
用户要求
researcher 结果
```

决定：

```text
next = writer
rounds = 2
```

### writer 执行

```text
messages:
    用户要求
    researcher 结果
    writer 文章
```

### Supervisor 第三次

决定：

```text
next = reviewer
rounds = 3
```

### reviewer 执行

```text
messages:
    用户要求
    researcher 结果
    writer 文章
    reviewer 意见
```

### Supervisor 最后决定

```text
next = FINISH
```

于是：

```text
FINISH
  ↓
END
```

整个过程中：

```text
messages
```

越来越丰富。

而：

```text
next
```

负责控制当前应该去哪。

这就是今天最核心的：

> **数据流在 State 中积累，控制流围绕 State 动态前进。**

---

# 十五、为什么要加 MAX_ROUNDS

Supervisor 本质上是动态循环。

例如 reviewer 可能说：

```text
文章需要修改
```

于是：

```text
reviewer
 ↓
writer
 ↓
reviewer
 ↓
writer
 ↓
reviewer
……
```

如果没有限制，模型可以一直绕下去。

所以：

```python
MAX_ROUNDS = 8
```

相当于：

> **给模型的自主决策设置一道硬上限。**

而且今天特意做了 fallback 模式：

```text
MAX_ROUNDS = 1
```

让流程进入：

```text
supervisor
→ researcher
→ supervisor
→ 强制 FINISH
```

这样才能真正证明兜底逻辑生效，而不是只看代码说“应该能生效”。

---

# 十六、今天的一个重要调试经验：两次运行不能拿来互相证明

之前的思路是：

```text
stream()
    ↓
拿流转链

invoke()
    ↓
拿最终 State
```

然后：

```text
第一次：4 轮
第二次：6 轮
```

再拿两者比较。

问题是：

> **它们根本不是同一次运行。**

LLM 是动态的，所以即使：

```text
temperature = 0
```

也不能把两个独立运行当成同一个实验。

最后改成：

```python
for mode, chunk in graph.stream(
    INITIAL,
    stream_mode=["updates", "values"]
):
```

于是一次运行同时得到：

```text
updates → 流程链
values  → State 快照
```

这样才能比较：

```text
这一次运行到底发生了什么。
```

这个思路其实比 Supervisor 本身还通用：

> **验证动态系统时，尽量让观察数据来自同一次实验。**

---

# 十七、今天还有一个特别隐蔽的测试坑：空集合

比如：

```python
for m in delivered:
    assert m.name in MEMBERS
```

如果：

```python
delivered = []
```

那么：

```text
循环执行 0 次
↓
assert 执行 0 次
↓
程序不报错
```

于是你会产生一种错觉：

> “检查通过了。”

实际上：

> **根本没有检查。**

所以正确做法是：

```python
assert delivered

for m in delivered:
    assert m.name in MEMBERS
```

先证明：

```text
确实存在需要验证的数据
```

再检查数据是否合法。

这就是今天说的“真空真”。

---

# 十八、验证器也可能看错地方

今天还有一个反例。

原本检测工具是否真的被调用，写成：

```python
called = any(
    "真身被调用" in str(m.content)
    for m in out["messages"]
)
```

结果：

```text
❌ 工具没有调用
```

但实际上工具真的调用了。

因为：

```python
print(...)
```

走的是：

```text
stdout
```

而：

```python
return ...
```

才会进入：

```text
ToolMessage
```

也就是：

```text
print
 ↓
终端

return
 ↓
ToolMessage
 ↓
State.messages
```

这是两条完全不同的数据通道。

所以后面才改成：

```text
CALLS
+
ToolMessage
```

两个证据一起看。

这个例子让我今天对测试又多了一层认识：

> **验证器本身也必须建立在正确的数据通道上。**

---

# 十九、今天最终形成的理解

以前看到这种代码，容易先被很多 API 吓到：

```python
StateGraph
add_node
add_edge
add_conditional_edges
with_structured_output
create_react_agent
HumanMessage
MessagesState
```

但今天把它还原成系统以后，其实就是：

```text
                 State
        ┌──────────┼──────────┐
        │          │          │
    messages      next      rounds
        │          │          │
        │          │          │
        ▼          ▼          ▼
     数据流     控制流      安全控制
        │          │
        └────┬─────┘
             ▼
        Supervisor
             │
          决定 next
             │
             ▼
          条件边
             │
             ▼
           Worker
             │
          干自己的活
             │
             ▼
      结果重新写入 State
             │
             └────→ Supervisor
```

所以今天我真正理解的，不只是：

> “Supervisor 可以调度多个 Agent。”

而是：

> **Supervisor 是一个不断读取 State、修改 State，并依据 State 决定下一跳的动态控制器。**

而整个系统可以分成两条主线：

```text
控制流：
Supervisor → next → 条件边 → Worker → Supervisor

数据流：
messages → Worker → 工作结果 → messages
```

这两条流最后都通过 State 汇合。

以后再看类似的多 Agent 代码，我觉得可以先问自己两个问题：

> **第一：谁决定下一步？**

> **第二：上一阶段产生的数据，是通过什么字段传给下一阶段的？**

先找到这两个答案，源码就不会再是一堆孤立的 API。

---

# 完整 Demo

下面保留今天最终版本的核心代码。

为了方便之后复习，完整 Demo 放在文章最后；**省略 import、包安装和断言代码**，正文重点放在运行机制和代码结构。

````python
MODEL_NAME = "deepseek-chat"
BASE_URL = "https://api.deepseek.com"
API_KEY = os.getenv("DEEPSEEK_API_KEY")

MODE = sys.argv[1] if len(sys.argv) > 1 else "normal"
MAX_ROUNDS = 1 if MODE == "fallback" else 8
FAULT = os.getenv("FAULT") or None

MEMBERS = [
    "researcher",
    "writer",
    "reviewer"
]


def make_llm(temperature: float = 0):
    return ChatOpenAI(
        model=MODEL_NAME,
        base_url=BASE_URL,
        api_key=API_KEY,
        temperature=temperature,
    )


llm = make_llm()
router_llm = make_llm()


class SuperState(MessagesState):
    next: str
    rounds: int


class Route(BaseModel):
    next: Literal[
        "researcher",
        "writer",
        "reviewer",
        "FINISH"
    ]
    reason: str = Field(
        default="",
        description="一句话理由，方便调试"
    )


router = router_llm.with_structured_output(
    Route,
    method="json_mode"
)


@tool
def web_search(query: str) -> str:
    print(f">>> [web_search] query={query!r}")

    return (
        f"[模拟检索结果] 关于「{query}」："
        "LangGraph 是用于构建有状态多步 Agent 的框架；"
        "核心包括 StateGraph、条件边和持久化能力。"
    )


WORKER_PROMPTS = {
    "researcher": (
        "你是研究员。用 web_search 查资料，"
        "把关键要点整理成 2-3 条，不要写完整文章。"
    ),

    "writer": (
        "你是写手。把对话历史里 researcher "
        "交的资料整合成一篇 300 字左右的中文文章。"
        "只写文章，不要写评论或修改意见。"
    ),

    "reviewer": (
        "你是审稿人。检查上一位 writer 写的文章，"
        "指出事实性、结构、表达上的具体问题。"
        "如果文章已经可以定稿，明确说「可以定稿」。"
    ),
}


def make_worker(name: str, tools: list, prompt: str):
    agent = create_react_agent(
        llm,
        tools,
        prompt=prompt
    )

    def node(state: SuperState) -> dict:
        result = agent.invoke({
            "messages": state["messages"]
        })

        last = result["messages"][-1]
        text = str(last.content)

        if FAULT == "no_deliver":
            return {
                "messages": []
            }

        msg_name = (
            None
            if FAULT == "no_name"
            else name
        )

        return {
            "messages": [
                HumanMessage(
                    content=f"[{name} 交活]\n{text}",
                    name=msg_name
                )
            ]
        }

    node.__name__ = name
    return node


worker_nodes = {
    "researcher": make_worker(
        "researcher",
        [web_search],
        WORKER_PROMPTS["researcher"]
    ),

    "writer": make_worker(
        "writer",
        [],
        WORKER_PROMPTS["writer"]
    ),

    "reviewer": make_worker(
        "reviewer",
        [],
        WORKER_PROMPTS["reviewer"]
    ),
}


SUPERVISOR_PROMPT = """
你是写作工作室的主编(supervisor)，手下三位成员：

- researcher：负责查资料，只交资料要点，不写正文
- writer：负责把资料整合成一篇结构清晰的中文文章
- reviewer：负责审稿，指出文章的问题

你的职责：每一步只决定“下一步交给谁”。
不要替任何成员干活，不要自己写文章。

标准流程：
researcher 拿资料
→ writer 成稿
→ reviewer 审稿
→ 若有重大问题，可以回 writer 修改
→ 都满意后输出 FINISH

判断依据：
看历史里带名的交活消息。

已经交过活的不必重复派。

只输出一个 JSON 对象，例如：

{
    "next": "researcher",
    "reason": "还没有资料"
}

next 只能是：
researcher / writer / reviewer / FINISH
"""


def supervisor_node(state: SuperState) -> dict:
    rounds = state.get("rounds", 0) + 1

    if rounds > MAX_ROUNDS:
        return {
            "next": "FINISH",
            "rounds": rounds,
            "messages": [
                (
                    "assistant",
                    f"【已达最大轮数 {MAX_ROUNDS}，强制结束】"
                )
            ],
        }

    msgs = [
        SystemMessage(
            content=SUPERVISOR_PROMPT
        )
    ] + state["messages"]

    decision = router.invoke(msgs)

    print(
        f"[supervisor] round={rounds} "
        f"next={decision.next} "
        f"({decision.reason})"
    )

    return {
        "next": decision.next,
        "rounds": rounds,
    }


def route_from_supervisor(state: SuperState):
    nxt = state.get("next") or "FINISH"

    return (
        nxt
        if nxt in MEMBERS or nxt == "FINISH"
        else "FINISH"
    )


def build_graph():
    builder = StateGraph(SuperState)

    builder.add_node(
        "supervisor",
        supervisor_node
    )

    for name, node in worker_nodes.items():
        builder.add_node(
            name,
            node
        )

    builder.add_edge(
        START,
        "supervisor"
    )

    path_map = {
        member: member
        for member in MEMBERS
    }

    path_map["FINISH"] = END

    builder.add_conditional_edges(
        "supervisor",
        route_from_supervisor,
        path_map
    )

    for name in MEMBERS:
        builder.add_edge(
            name,
            "supervisor"
        )

    return builder.compile()


INITIAL = {
    "messages": [
        (
            "user",
            "写一篇 300 字介绍 LangGraph 的文章，"
            "要求有事实依据"
        )
    ],
    "next": "",
    "rounds": 0,
}


def main():
    graph = build_graph()

    chain = []
    final_state = None

    for mode, chunk in graph.stream(
        INITIAL,
        stream_mode=["updates", "values"]
    ):
        if mode == "updates":
            for node_name in chunk:
                chain.append(node_name)

        elif mode == "values":
            final_state = chunk

    out = final_state

    print(
        "\n[流转链]",
        " → ".join(chain)
    )

    print("\n=== 成品 ===")
    print(
        str(
            out["messages"][-1].content
        )[:400]
    )

    print(
        f"[结果] "
        f"next={out['next']} "
        f"rounds={out['rounds']}"
    )

    mermaid = graph.get_graph().draw_mermaid()

    with open(
        "w7_supervisor_topology.md",
        "w",
        encoding="utf-8"
    ) as f:
        f.write(
            "```mermaid\n"
            + mermaid
            + "\n```\n"
        )


if __name__ == "__main__":
    main()
````
