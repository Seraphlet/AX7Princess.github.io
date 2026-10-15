---
description: ""
title: "从 HTTP 接口到有状态的 LangGraph Agent 服务"
draft: false
date: "2026-10-15T11:53:25+08:00"
slug: "fastapi-langgraph-agent-service"
categories:
 - FastAPI
tags:
 - null
image: ""
---

# W10｜FastAPI：从 HTTP 接口到有状态的 LangGraph Agent 服务

> **学习内容**：FastAPI、Pydantic、APIRouter、Depends、lifespan、async/await、LangGraph Checkpointer
>
> **源码仓库**：[AgentLearn / W10](https://github.com/Seraphlet/AgentLearn/tree/main/W10)
>
> **学习目标**：理解如何将原本只能在 Python 中调用的 Agent，封装成能够通过 HTTP 请求访问、支持多轮会话和状态持久化的 Web 服务。

## 一、这两天到底学了什么？

在之前的 LangGraph 学习中，我们已经可以构建 Agent、调用工具、管理 State，并通过 Checkpointer 保存会话。

但这些程序通常是在 Python 内部调用的：

```python
result = graph.invoke(
    {"messages": [("user", "你好")]},
    config={"configurable": {"thread_id": "001"}}
)
```

如果现在希望网页、手机 App 或其他服务调用这个 Agent，就需要提供一个统一的外部入口。

FastAPI 解决的正是这个问题。

它可以把 Python 函数暴露为 HTTP 接口，让外部客户端通过网络发送请求、获取结果。

这两天的学习可以概括为：

- **Day1**：理解 HTTP 接口、路由、请求验证和错误处理。
- **Day2**：将 LangGraph 接入 FastAPI，管理 Agent 生命周期，并通过 SQLite Checkpointer 保存会话。

最终形成的系统结构：

```text
客户端（浏览器 / 其他程序）
          │
          │ HTTP POST /chat
          ▼
┌─────────────────────────┐
│         FastAPI         │
│                         │
│  Pydantic 请求验证       │
│           │             │
│           ▼             │
│  Router 路由处理         │
│           │             │
│           ▼             │
│  Depends 获取 Graph      │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│        LangGraph        │
│                         │
│  State → Node → State   │
│           │             │
│      Checkpointer       │
└───────────┬─────────────┘
            │
            ▼
      SQLite 数据库
      保存 / 恢复会话
            │
            ▼
      返回 ChatResponse
            │
            ▼
         客户端
```

这里最重要的是区分职责：

**FastAPI 负责对外提供服务，LangGraph 负责 Agent 的执行，Checkpointer 负责状态持久化。**

它们不是相互替代的关系，而是不同层次的组件。

---

## 二、Day1：FastAPI 如何把 Python 函数变成 HTTP 接口？

### 2.1 FastAPI、Uvicorn 和路由函数

先看最简单的代码：

```python
from fastapi import FastAPI

app = FastAPI(
    title="My Agent API",
    version="0.1.0"
)

@app.get("/ping")
def ping():
    return {"status": "ok"}
```

这段代码包含三个重要概念。

**① FastAPI：Web 应用对象**

```python
app = FastAPI()
```

创建的是一个 ASGI 应用对象。

它负责组织路由、请求处理、响应生成等功能，但它本身并不是负责监听网络端口的服务器。

**② Uvicorn：ASGI 服务器**

启动命令：

```bash
uvicorn main:app --reload
```

其中：

- `main`：Python 模块名，对应 `main.py`。
- `app`：模块中的 FastAPI 对象。
- `--reload`：开发模式下监听文件变化并重新启动服务。

Uvicorn 接收 HTTP 请求，再交给 FastAPI 应用处理。

**③ 路由：HTTP 请求与 Python 函数之间的映射**

```python
@app.get("/ping")
def ping():
    return {"status": "ok"}
```

`@app.get("/ping")` 是一个装饰器。

在模块加载时，它会向 FastAPI 注册路由，让框架知道：

```text
HTTP 请求
    │
    ├── Method = GET
    │
    └── Path = /ping
              │
              ▼
           ping()
              │
              ▼
      {"status": "ok"}
```

这里并不是浏览器直接调用了 Python 函数。

而是 Uvicorn 接收请求，FastAPI 根据请求方法和路径找到对应函数，执行后把返回值转换成 HTTP 响应。

### 2.2 GET 和 POST 有什么区别？

Day1 使用了两个 HTTP 方法：

```python
@app.get("/health")
def health():
    return {
        "status": "healthy",
        "service": "agent-api"
    }
```

```python
@app.post("/echo")
def echo(req: EchoRequest):
    return {
        "you_said": req.message * req.times,
        "times": req.times
    }
```

GET 通常用于获取资源或查询信息，POST 通常用于提交数据并触发处理。

例如 Agent 服务可以设计为：

```text
GET  /health   查询服务状态

POST /chat     提交消息，执行 Agent
```

HTTP 方法表达的是接口的语义，而不是 Python 函数本身的能力限制。

### 2.3 Pydantic：请求进入业务之前的验证

Day1 定义了一个请求模型：

```python
from pydantic import BaseModel, Field

class EchoRequest(BaseModel):
    message: str = Field(
        ...,
        min_length=1,
        description="要回显的内容"
    )

    times: int = Field(
        default=1,
        ge=1,
        le=5
    )
```

`BaseModel` 用来定义数据结构及验证规则。

例如客户端发送：

```json
{
  "message": "hello",
  "times": 2
}
```

FastAPI 会根据 `EchoRequest` 解析和验证请求体。

字段含义：

| 写法 | 作用 |
|---|---|
| `message: str` | 字段需要是字符串 |
| `Field(...)` | 必填字段 |
| `min_length=1` | 至少一个字符 |
| `default=1` | 默认值为 1 |
| `ge=1` | 大于等于 1 |
| `le=5` | 小于等于 5 |

请求处理过程：

```text
客户端 JSON
     │
     ▼
FastAPI 接收请求
     │
     ▼
Pydantic 解析与验证
     │
     ├── 验证失败
     │       └── HTTP 422
     │
     └── 验证成功
             │
             ▼
       EchoRequest 对象
             │
             ▼
          echo(req)
```

这里和之前学习的 LLM 结构化输出有相似之处。

它们都通过 Schema 约束数据结构，只不过验证对象不同：

- LLM 结构化输出：约束或验证模型输出。
- FastAPI 请求模型：验证客户端输入。

本质上都是在系统边界建立数据契约。

### 2.4 HTTPException：数据合法不代表业务允许

Day1 还有这段代码：

```python
@app.post("/echo")
def echo(req: EchoRequest):
    if req.times > 3:
        raise HTTPException(
            status_code=400,
            detail="times 太大"
        )

    return {
        "you_said": req.message * req.times,
        "times": req.times
    }
```

Pydantic 允许 `times` 最大为 5。

但是业务代码要求不能超过 3。

因此：

```text
times = 2
    │
    └── 验证通过 → 业务通过 → HTTP 200

times = 4
    │
    └── 验证通过 → 业务拒绝 → HTTP 400

times = 6
    │
    └── 验证失败 → HTTP 422
```

这是两个不同层次的验证。

**Pydantic 判断数据是否符合契约，业务逻辑判断操作是否被允许。**

这个区别在 Agent 工程中同样重要。

例如工具调用参数格式正确，不代表当前用户有权限执行该工具。

### 2.5 Swagger UI：接口怎么测试？

FastAPI 默认提供交互式 API 文档。

启动服务后访问：

```text
http://127.0.0.1:8000/docs
```

可以直接选择接口、填写请求参数并执行。

这比每次编写一个 Python 测试客户端更直观，适合刚开始学习 HTTP 接口时使用。

但 `/docs` 只是接口交互与调试工具，不能替代自动化测试。

---

## 三、Day2：为什么一个接口突然拆成四个文件？

Day1 只有一个 `main.py`。

Day2 则拆成：

```text
w10-day2/
│
├── main.py
│   └── 应用入口与生命周期
│
├── models.py
│   └── 请求 / 响应的数据契约
│
├── routes.py
│   └── HTTP 路由与业务调用
│
└── graph_builder.py
    └── LangGraph 状态、节点与图构建
```

拆分的目的不是让代码看起来更复杂，而是隔离不同职责。

例如以后要将演示节点替换成真正的 ReAct Agent，只要外部接口契约不变，主要修改就可以集中在 `graph_builder.py`。

不过，阅读 Day2 有一个容易忽略的问题：

**文件之间的依赖关系，不等于程序实际运行时的调用顺序。**

必须区分两个阶段：

```text
【服务启动阶段】

加载 main.py
      │
      ▼
创建 FastAPI 应用
      │
      ▼
进入 lifespan
      │
      ▼
创建 SQLite Checkpointer
      │
      ▼
build_graph()
      │
      ▼
compile()
      │
      ▼
app.state.graph = graph
      │
      ▼
服务准备接收请求


【请求处理阶段】

POST /chat
      │
      ▼
验证 ChatRequest
      │
      ▼
Depends(get_graph)
      │
      ▼
从 app.state 取出 Graph
      │
      ▼
graph.ainvoke()
      │
      ▼
LangGraph 执行节点
      │
      ▼
返回 ChatResponse
```

Day2 的核心就是把这两条执行链连接起来。

---

## 四、main.py：lifespan 与 Agent 的生命周期

源码中的核心结构：

```python
from contextlib import asynccontextmanager

from fastapi import FastAPI
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

from graph_builder import build_graph
from routes import router

DB_PATH = "sessions.db"

@asynccontextmanager
async def lifespan(app: FastAPI):

    async with AsyncSqliteSaver.from_conn_string(
        DB_PATH
    ) as saver:

        app.state.graph = build_graph(
            checkpoint=saver
        )

        yield

app = FastAPI(
    title="My Agent API",
    version="0.2.0",
    lifespan=lifespan
)

app.include_router(router)
```

这里第一次接触到三个新概念：

- `asynccontextmanager`
- `lifespan`
- `app.state`

### 4.1 为什么需要 lifespan？

在普通 Python 脚本中，通常这样执行：

```text
创建资源
    ↓
执行任务
    ↓
释放资源
    ↓
程序结束
```

但 Web 服务需要持续接收请求。

如果每来一个请求都重新创建数据库连接、构建并编译 Agent，就会产生不必要的初始化开销。

因此需要一个与应用生命周期绑定的资源管理机制。

FastAPI 的 `lifespan` 正是用于管理启动与关闭阶段。

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    # 启动阶段

    yield

    # 关闭阶段
```

可以这样理解：

```text
服务启动
    │
    ▼
执行 yield 之前的代码
    │
    ▼
yield
    │
    │  服务开始处理 HTTP 请求
    │
    │  请求 1
    │  请求 2
    │  请求 3
    │  ...
    │
    ▼
服务关闭
    │
    ▼
执行 yield 之后的代码
```

`yield` 在这里不是为了给客户端返回数据。

它是异步上下文管理器的生命周期分界点。

`asynccontextmanager` 将异步生成器包装成符合异步上下文管理器协议的对象。

### 4.2 `async with` 为什么能自动释放资源？

```python
async with AsyncSqliteSaver.from_conn_string(
    DB_PATH
) as saver:
    ...
```

这里创建并管理异步 SQLite Checkpointer。

进入 `async with` 时获得资源，离开时执行对应的清理逻辑。

因此当前代码即使没有在 `yield` 后显式写关闭数据库的语句，外层 `async with` 仍然负责释放它管理的资源。

要注意，数据库资源的生命周期应该覆盖 Graph 的使用期间。

如果在启动时打开连接，却在请求执行之前关闭，那么后续 Graph 即使还存在，也可能无法正常使用 Checkpointer。

### 4.3 为什么要把 Graph 放到 app.state？

```python
app.state.graph = build_graph(
    checkpoint=saver
)
```

`app.state` 是 FastAPI 应用对象上的共享状态容器。

这里把已经编译好的 Graph 存入应用级状态，供后续请求使用。

它与 LangGraph 的 `State` 不是一回事。

| 概念 | 保存什么 |
|---|---|
| `app.state` | FastAPI 应用级共享资源 |
| `ChatState` | LangGraph 执行中的业务状态 |
| SQLite Checkpointer | LangGraph 的持久化状态 |

例如：

```text
app.state.graph
       │
       └── 保存可执行 Graph 对象

ChatState["messages"]
       │
       └── 保存 Agent 的消息状态

SQLite
       │
       └── 持久化指定 thread 的状态
```

**app.state 保存的是 Graph 对象，不是所有用户共用的一份聊天记录。**

不同会话的状态由 `thread_id` 区分，并由 Checkpointer 管理。

---

## 五、routes.py：Depends 是怎样工作的？

Day2 的路由文件先创建了一个 Router：

```python
from fastapi import APIRouter

router = APIRouter()
```

`APIRouter` 用来组织路由。

Day1 直接通过：

```python
@app.get("/ping")
```

注册接口。

Day2 则可以通过：

```python
@router.post("/chat")
```

先在独立模块中定义路由，再由 `main.py` 注册：

```python
app.include_router(router)
```

这样就不需要把所有接口都写在应用入口文件里。

### 5.1 get_graph() 做了什么？

```python
def get_graph(request: Request):
    graph = getattr(
        request.app.state,
        "graph",
        None
    )

    if graph is None:
        raise HTTPException(
            status_code=503,
            detail="graph 尚未就绪"
        )

    return graph
```

这里的 `Request` 是 FastAPI 提供的请求对象。

通过：

```python
request.app
```

可以访问处理该请求的应用对象。

然后：

```python
getattr(request.app.state, "graph", None)
```

尝试取得启动阶段保存的 Graph。

`getattr()` 的三个参数分别是：

```python
getattr(对象, 属性名, 默认值)
```

如果找不到 `graph` 属性，就返回 `None`。

这时主动抛出 HTTP 503，表示服务当前无法处理该请求。

### 5.2 Depends 为什么不是普通函数调用？

```python
async def chat(
    req: ChatRequest,
    graph=Depends(get_graph)
):
    ...
```

这是 Day2 最需要理解的 FastAPI 机制之一。

`Depends(get_graph)` 并不是直接执行 `get_graph()`，然后把结果作为普通默认参数。

它声明的是一个依赖关系：

> 执行 chat 之前，需要先通过 get_graph 获得 graph。

请求到达时，FastAPI 会解析依赖：

```text
FastAPI 准备调用 chat()
          │
          ▼
发现 graph 依赖 get_graph
          │
          ▼
调用 get_graph(request)
          │
          ▼
获取 app.state.graph
          │
          ▼
把返回值注入 graph 参数
          │
          ▼
执行 chat(req, graph)
```

这种机制称为依赖注入（Dependency Injection）。

它的价值是让业务函数只关心自己需要什么，而不必自己创建所有依赖。

例如 `chat()` 不需要知道：

- SQLite 在什么位置打开。
- Graph 在什么时候编译。
- Graph 使用什么 Checkpointer。

它只需要拿到一个可以调用的 Graph。

这样未来测试 `chat()` 时，也可以替换依赖，使用模拟 Graph，而不必运行真实的 Agent。

**依赖注入解决的是依赖的提供与管理问题，而不是 Agent 的状态持久化问题。**

---

## 六、models.py：请求与响应为什么都需要模型？

Day2 定义了两个 Pydantic 模型。

### 6.1 ChatRequest：输入契约

```python
class ChatRequest(BaseModel):
    message: str = Field(
        ...,
        min_length=1,
        max_length=4000
    )

    session_id: str = Field(
        default="default",
        min_length=1,
        max_length=64,
        pattern=r"^[A-Za-z0-9_-]+$"
    )
```

客户端请求示例：

```json
{
  "message": "你好",
  "session_id": "user_001"
}
```

`message` 是用户消息。

`session_id` 是会话标识。

这里使用正则表达式：

```python
r"^[A-Za-z0-9_-]+$"
```

限制会话 ID 只能包含字母、数字、下划线和连字符。

但它只是输入格式限制，不是身份认证。

真实多用户服务还需要确认当前用户是否有权访问该会话。

### 6.2 ChatResponse：输出契约

```python
class ChatResponse(BaseModel):
    reply: str
    session_id: str
    turns: int
```

它规定接口返回：

```json
{
  "reply": "你好",
  "session_id": "user_001",
  "turns": 2
}
```

路由通过：

```python
@router.post(
    "/chat",
    response_model=ChatResponse
)
```

声明响应模型。

FastAPI 会根据响应模型验证并序列化返回数据。

因此 Pydantic 不仅能保护输入边界，也能约束输出边界。

这对 Agent 服务很重要。

即使内部模型或 Agent 的执行过程不断变化，对外 API 仍然可以保持稳定的数据结构。

---

## 七、graph_builder.py：将已经学过的 LangGraph 接入服务

这一部分没有引入新的 LangGraph 架构，仍然是之前学过的 StateGraph。

### 7.1 定义 State

```python
class ChatState(TypedDict):
    messages: Annotated[
        list,
        add_messages
    ]
```

`messages` 是 Graph 的状态字段。

`add_messages` 是 Reducer，负责合并消息更新。

它不是普通的列表追加操作，而是按消息 ID 进行合并：相同 ID 可以替换，新的消息通常会追加。

### 7.2 定义节点

```python
def respond(state: ChatState):
    turns = len(state["messages"])

    text = _text_of(
        state["messages"][-1]
    )

    return {
        "messages": [
            (
                "ai",
                f"（第 {turns} 条消息）你说的是：{text}"
            )
        ]
    }
```

这里的 `respond()` 只是一个演示节点。

它读取最新消息，再返回一条 AI 消息，并没有真正调用大模型。

执行过程：

```text
ChatState
    │
    ▼
respond(state)
    │
    ├── 读取 messages
    │
    ├── 提取最新消息
    │
    └── 返回 AI 消息
            │
            ▼
      add_messages
            │
            ▼
      更新后的 State
```

注意：

```python
turns = len(state["messages"])
```

统计的是消息数量，而不是真正的对话轮数。

一轮用户提问和 AI 回复通常会产生两条消息。

因此当前变量名 `turns` 不够精确，但仍可用于观察状态是否累积。

### 7.3 构建 Graph

```python
def build_graph(checkpoint=None):
    g = StateGraph(ChatState)

    g.add_node("respond", respond)

    g.add_edge(START, "respond")
    g.add_edge("respond", END)

    return g.compile(
        checkpointer=checkpoint
    )
```

执行拓扑：

```text
START
  │
  ▼
respond
  │
  ▼
 END
```

这里的重点是：

```python
g.compile(checkpointer=checkpoint)
```

Graph 编译时接入 Checkpointer。

以后执行时，LangGraph Runtime 会根据配置中的 `thread_id` 管理对应的持久化状态。

节点本身不需要编写 SQL。

这也是 LangGraph 把业务逻辑与状态持久化分离的一个体现。

---

## 八、最关键的主链路：POST /chat 是怎样执行的？

现在把前面的所有组件串起来。

客户端发送：

```json
{
  "message": "你好",
  "session_id": "demo_001"
}
```

### 第一步：FastAPI 匹配路由

```python
@router.post("/chat")
async def chat(...):
    ...
```

FastAPI 找到处理函数。

### 第二步：Pydantic 验证请求

```python
req: ChatRequest
```

将 JSON 解析成请求模型。

### 第三步：Depends 注入 Graph

```python
graph=Depends(get_graph)
```

FastAPI 调用 `get_graph()`，取得启动时已经编译好的 Graph。

### 第四步：生成 LangGraph 配置

```python
config = {
    "configurable": {
        "thread_id": req.session_id
    }
}
```

这里建立了：

```text
HTTP session_id
       │
       ▼
LangGraph thread_id
       │
       ▼
Checkpointer 查找对应状态
```

`session_id` 是 API 对外暴露的字段名。

`thread_id` 是 LangGraph Checkpointer 用来标识线程状态的配置字段。

它们可以使用同一个字符串，但属于不同层次。

### 第五步：执行 Graph

以下是修正消息结构后的调用示意：

```python
result = await graph.ainvoke(
    {
        "messages": [
            ("user", req.message)
        ]
    },
    config
)
```

这里有两个关键输入：

- `messages`：本次新增的用户消息。
- `config`：指定要使用哪个 thread。

如果该 thread 已有保存的状态，LangGraph 可以从 Checkpointer 恢复之前的 State，并应用本次输入。

然后执行节点。

### 第六步：返回响应

Graph 执行完成后：

```python
msgs = result.get("messages", [])
```

得到消息列表。

再提取 AI 的回复，构造：

```python
ChatResponse(
    reply=reply,
    session_id=req.session_id,
    turns=len(msgs)
)
```

FastAPI 最终将响应序列化为 JSON，返回给客户端。

完整主链路：

```text
HTTP POST /chat
        │
        ▼
ChatRequest 验证
        │
        ▼
Depends(get_graph)
        │
        ▼
获取 app.state.graph
        │
        ▼
构造 thread_id
        │
        ▼
graph.ainvoke()
        │
        ▼
Checkpointer 恢复状态
        │
        ▼
respond(state)
        │
        ▼
Reducer 合并更新
        │
        ▼
Checkpointer 保存状态
        │
        ▼
提取 AI 回复
        │
        ▼
ChatResponse
        │
        ▼
HTTP JSON 响应
```

这就是 Day2 的完整运行逻辑。

---

## 九、async / await：为什么 Day2 要使用异步？

Day1 的路由函数是：

```python
def echo(req: EchoRequest):
    ...
```

Day2 则是：

```python
async def chat(...):
    result = await graph.ainvoke(...)
```

这里使用异步，不是因为 LangGraph 比普通函数更高级，而是因为 Web 服务可能需要同时处理多个请求，并且请求执行期间可能存在等待操作。

例如：

```text
请求 A → 等待模型 API 返回
请求 B → 等待数据库操作
请求 C → 处理其他工作
```

如果这些等待操作支持异步，事件循环可以在等待期间调度其他任务。

`async def` 定义协程函数。

`await` 等待一个可等待对象完成，并允许事件循环在适当的时候执行其他任务。

但需要明确：

**异步不等于并行，也不意味着 CPU 密集型代码会自动加速。**

如果在异步函数中直接执行大量阻塞计算，仍可能影响其他请求。

在 Day2 中，使用 `AsyncSqliteSaver` 和 `graph.ainvoke()`，是为了让服务的异步调用链保持一致。

---

## 十、代码检查：当前源码存在的问题

这两天的学习重点是理解架构，因此暂时不直接重构代码。

但阅读时需要知道，当前 Day2 代码存在几处问题。

| 文件 | 问题 | 原因 |
|---|---|---|
| `routes.py` | `["user", req.message]` | 消息结构不符合预期 |
| `routes.py` | `mession_id` | 字段拼写错误，缺少必填的 `session_id` |
| `graph_builder.py` | `_text_of(m).split()` | 把字符串变成列表 |
| `graph_builder.py` | `from cv2 import add` | 不必要的依赖，可能造成导入失败 |
| `graph_builder.py` | `pyexpat.errors.messages` | 无关且可能失败的导入 |
| `graph_builder.py` | 消息提取未判断 AI 类型 | 可能选中非 AI 消息 |

其中前三个尤其值得注意。

**消息结构错误**

```python
{"messages": ["user", req.message]}
```

这里把角色和正文放成两个列表元素。

正确的单条消息表示形式可以是：

```python
{"messages": [("user", req.message)]}
```

外层列表表示消息集合，内层元组表示一条消息。

**响应字段拼写错误**

```python
ChatResponse(
    reply=...,
    mession_id=req.session_id,
    turns=len(msgs)
)
```

模型定义的是 `session_id`，不是 `mession_id`。

Pydantic 会在创建响应对象时发现缺少必填字段。

**返回值类型错误**

```python
text = _text_of(m).split()
```

`.split()` 返回的是列表。

但 `ChatResponse.reply` 要求字符串。

另外，当前代码中从后往前寻找消息的逻辑没有判断是否属于 AI，未来接入真实 Agent 时也需要修正。

这些问题并不改变 Day2 的整体设计思路。

它们可以在后续测试中逐项修复，并通过 pytest 验证。

---

## 十一、从脚本到服务：工程上到底改变了什么？

以前构建 Agent 时，主要考虑：

```text
输入
  ↓
Graph
  ↓
输出
```

现在还需要考虑：

```text
外部请求
    │
    ▼
请求契约
    │
    ▼
路由与依赖
    │
    ▼
Agent Runtime
    │
    ▼
状态持久化
    │
    ▼
响应契约
    │
    ▼
外部客户端
```

这种变化不仅是增加了 FastAPI。

更重要的是，系统开始有明确的外部边界。

例如：

**输入边界**：请求是否合法，参数是否满足要求？

**资源边界**：数据库连接何时创建，何时关闭？

**状态边界**：不同会话如何区分，谁负责持久化？

**业务边界**：FastAPI 是否需要知道 Agent 内部节点？

**输出边界**：Agent 返回的数据能否满足 API 契约？

这些问题是将 Agent 从学习脚本转变为实际应用时必须面对的。

不过，当前 Day2 仍然只是一个教学级服务骨架。

它尚未实现完整的用户认证、并发控制、生产级错误处理、日志与监控，也没有真正接入 LLM。

因此不能仅凭它使用了 FastAPI 和 SQLite，就认为已经具备生产部署能力。

---

## 十二、最后总结：必须记住的知识

### FastAPI 的核心职责

```text
Uvicorn
   │
   └── 接收网络请求
          │
          ▼
FastAPI
   │
   ├── 路由匹配
   ├── 请求验证
   ├── 依赖注入
   ├── 调用处理函数
   └── 响应序列化
```

### Day2 的关键概念

| 概念 | 核心作用 |
|---|---|
| `FastAPI()` | 创建 Web 应用 |
| `APIRouter` | 组织与注册路由 |
| `BaseModel` | 定义输入输出数据契约 |
| `Depends` | 声明并解析依赖 |
| `lifespan` | 管理应用启动与关闭 |
| `asynccontextmanager` | 定义异步资源管理区间 |
| `app.state` | 保存应用级共享资源 |
| `async / await` | 支持异步调用与等待 |
| `graph.ainvoke()` | 异步执行 LangGraph |
| `thread_id` | 标识 LangGraph 会话线程 |
| `Checkpointer` | 保存与恢复 Graph 状态 |

### 我对这两天学习的理解

Day1 让我知道，FastAPI 可以通过路由把 Python 函数提供给外部调用，并通过 Pydantic 在请求进入业务逻辑之前验证数据。

Day2 则把原本独立运行的 LangGraph 接入 HTTP 服务。

整个系统分成两个阶段：

**启动阶段**负责创建数据库资源、编译 Graph，并将它保存为应用级共享对象。

**请求阶段**负责验证输入、取得 Graph、指定会话、执行 Agent，再将结果转换成 HTTP 响应。

其中，FastAPI 不负责管理 Agent 的内部 State，LangGraph 也不负责直接处理 HTTP 请求。两者通过路由函数连接起来。

目前最需要进一步理解的是 `lifespan`、`Depends` 和 `async/await`，因为它们分别改变了资源管理、对象获取和函数执行的方式。

**最终记住一句话：FastAPI 负责让 Agent 能被外部调用，LangGraph 负责 Agent 怎样执行，Checkpointer 负责执行状态怎样跨请求保存。**

这就是 W10 前两天建立起来的完整知识骨架。


