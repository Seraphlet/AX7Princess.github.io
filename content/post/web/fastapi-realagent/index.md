---
description: ""
title: "LangGraph Agent 服务的流式输出、会话管理与并发隔离"
draft: false
date: "2026-10-18T09:39:41+08:00"
slug: "fastapi-langgraph-streaming-session-concurrency"
categories:
 - 
tags:
 - 
image: ""
---

# FastAPI × LangGraph 进阶：异步流式输出、SSE、会话管理与并发隔离

> **学习主题**：Python 异步生成器、LangGraph Streaming、SSE、HTTP 流式响应、Checkpointer 状态管理、并发隔离与测试

## 一、先建立整体知识结构

之前已经学过如何通过 FastAPI 接收请求，再调用 LangGraph 执行 Agent。

但实际聊天服务不仅要能回答问题，还需要解决三个问题：

1. 模型生成较慢，如何让客户端边生成边接收？
2. 如何读取、保存和删除不同会话的历史状态？
3. 多个请求同时执行，怎样防止状态串台？

这三个问题分别对应三组知识：

```text
Agent Web 服务
│
├── ① 流式输出
│   ├── async / await
│   ├── 异步生成器 yield
│   ├── LangGraph astream
│   ├── chunk / metadata
│   ├── StreamingResponse
│   └── SSE 协议
│
├── ② 会话管理
│   ├── session_id / thread_id
│   ├── Checkpointer
│   ├── aget_state
│   ├── adelete_thread
│   └── 数据模型与历史分页
│
└── ③ 并发与可靠性
    ├── 事件循环
    ├── asyncio.gather
    ├── 共享可变状态
    ├── 请求隔离
    ├── 取消与异常处理
    └── 自动化测试
```

先记住整个系统的数据流：

```text
客户端
   │
   │ POST /chat/stream
   ▼
FastAPI
   │
   ├── Pydantic 验证请求
   ├── Depends 获取 Graph
   └── 生成 thread_id 配置
           │
           ▼
      graph.astream()
           │
           ├── Checkpointer 恢复状态
           ├── Graph 执行节点
           └── LLM 生成消息
                   │
                   ▼
           (chunk, metadata)
                   │
                   ▼
              提取文本
                   │
                   ▼
               SSE 编码
                   │
                   ▼
          StreamingResponse
                   │
                   ▼
             客户端接收
```

需要区分两条链路：

**执行链路**：FastAPI → LangGraph → LLM。

**传输链路**：LangGraph 消息 → SSE → HTTP → 客户端。

---

# 第一部分：Python 异步与生成器

## 二、async / await：为什么 Web 服务需要异步

### 2.1 先理解同步执行

普通 Python 函数：

```python
def get_result():
    result = call_model()
    return result
```

执行到 `call_model()` 时，当前执行流程需要等待调用完成，才能继续运行。

假设模型需要 5 秒才能返回，那么这次函数调用就要等待这 5 秒。

Web 服务可能同时收到多个请求。

如果所有等待都阻塞同一个执行线程，就会降低服务处理其他请求的能力。

### 2.2 异步执行的核心

```python
async def get_result():
    result = await call_model_async()
    return result
```

其中：

- `async def`：定义协程函数。
- `await`：等待一个可等待对象完成。
- 事件循环：负责调度能够继续运行的异步任务。

执行流程可以理解为：

```text
请求 A
   │
   ▼
调用异步模型
   │
   ▼
await 等待网络响应
   │
   └── 当前任务让出执行机会
                 │
                 ▼
              请求 B
                 │
                 ▼
              执行其他任务
                 │
                 ▼
          请求 A 的结果就绪
                 │
                 ▼
             恢复请求 A
```

关键不是异步能让模型生成速度更快。

而是**等待外部 I/O 时，程序可以处理其他任务**。

注意，`async def` 不会自动让内部所有代码变成非阻塞代码。同步阻塞调用仍可能占用执行线程。

### 2.3 async 不等于多线程

| 概念 | 主要机制 |
|---|---|
| 同步调用 | 当前执行流程等待结果 |
| 异步调用 | 任务可在等待时让出执行机会 |
| 多线程 | 多个线程分别执行任务 |
| 多进程 | 多个进程独立运行 |
| 并发 | 多个任务在同一时间段内推进 |
| 并行 | 多个任务在同一时刻实际执行 |

异步适合大量网络 I/O 等待场景，但不能直接解决 CPU 密集型计算问题。

---

## 三、yield：为什么流式输出不能只用 return

### 3.1 return 返回一次结果

```python
def numbers():
    return [1, 2, 3]
```

调用方一次得到完整列表。

### 3.2 yield 逐次产生数据

```python
def numbers():
    yield 1
    yield 2
    yield 3
```

调用：

```python
for n in numbers():
    print(n)
```

每次迭代从生成器取得一个值。

生成器在 `yield` 处暂停，下一次迭代时从暂停位置继续。

它不会像普通 `return` 那样直接结束整个函数。

### 3.3 异步生成器

当生成数据的过程中还需要异步等待时，可以组合 `async def` 和 `yield`：

```python
import asyncio

async def generate():
    for i in range(3):
        await asyncio.sleep(1)
        yield i

async def main():
    async for item in generate():
        print(item)

asyncio.run(main())
```

执行流程：

```text
generate()
    │
    ▼
await sleep(1)
    │
    ▼
yield 0
    │
    ▼
调用方收到 0
    │
    ▼
继续迭代
    │
    ▼
await sleep(1)
    │
    ▼
yield 1
    │
    ▼
...
```

这就是后面 FastAPI 流式接口使用的基础机制。

### 3.4 四个语法的区别

| 语法 | 用途 |
|---|---|
| `return` | 返回结果并结束函数 |
| `yield` | 产生一个值，保留生成器执行位置 |
| `await` | 等待可等待对象 |
| `async for` | 逐次消费异步可迭代对象 |

**记忆规则**：`await` 解决等待，`yield` 解决逐次产出。

---

# 第二部分：LangGraph 流式执行

## 四、ainvoke 和 astream 的区别

之前调用 Graph：

```python
result = await graph.ainvoke(
    {"messages": [("user", "你好")]},
    config
)
```

它返回本次执行完成后的结果。

但聊天界面通常希望模型逐步输出内容，因此可以使用：

```python
async for chunk, meta in graph.astream(
    {"messages": [("user", "你好")]},
    config,
    stream_mode="messages"
):
    print(chunk)
```

### 4.1 两种调用的执行方式

```text
ainvoke：

输入 → Graph 执行 → 最终结果


astream：

输入 → Graph 执行
             │
             ├── 事件 1
             ├── 事件 2
             ├── 事件 3
             └── 执行结束
```

它们都可以执行 Graph。

区别在于调用方怎样获取执行结果。

### 4.2 stream_mode 决定输出结构

常见流模式：

| 模式 | 返回内容 | 适用场景 |
|---|---|---|
| `values` | 执行过程中的完整 State | 观察状态 |
| `updates` | 节点产生的状态更新 | 调试节点 |
| `messages` | LLM 消息块及元数据 | 聊天输出 |
| `custom` | 自定义事件 | 进度、业务事件 |

特别注意：

`stream_mode="messages"` 下通常得到 `(message_chunk, metadata)`。

但 `updates` 和 `values` 模式的输出结构不同。

因此，**不能脱离 stream_mode 判断 astream 返回的数据类型**。

---

## 五、chunk 和 metadata 是什么

核心代码：

```python
async for chunk, meta in graph.astream(
    {"messages": [("user", req.message)]},
    config,
    stream_mode="messages"
):
    ...
```

这里使用了 Python 的序列解包。

例如：

```python
name, age = ("Alice", 20)
```

把一个包含两个元素的对象分别赋给两个变量。

LangGraph 在 `messages` 模式下，每次产生的事件通常包含：

```python
(message_chunk, metadata)
```

因此：

```python
chunk, meta = event
```

### 5.1 chunk：消息内容

`chunk` 通常是 LangChain 消息块对象，例如 `AIMessageChunk`。

可以通过：

```python
content = chunk.content
```

读取消息内容。

但 `content` 不一定总是字符串，也可能是内容块列表。

因此源码提供了一个适配函数：

```python
def text_of(message) -> str:
    content = getattr(message, "content", None)

    if isinstance(content, str):
        return content

    if isinstance(content, list):
        return "".join(
            block.get("text", "")
            for block in content
            if isinstance(block, dict)
        )

    return ""
```

它把不同形式的消息内容转换成字符串。

这里的 `getattr()` 用于读取对象属性：

```python
getattr(obj, "content", None)
```

相当于尝试获取 `obj.content`，不存在时返回 `None`。

### 5.2 metadata：消息从哪里产生

```python
node = meta.get("langgraph_node", "")
```

`meta` 通常是一个字典，包含节点来源等信息。

例如：

```python
{
    "langgraph_node": "agent"
}
```

通过元数据可以判断消息来自哪个节点。

这对复杂 Graph 尤其重要。

```text
START
  │
  ▼
agent
  │
  ▼
respond
  │
  ▼
END
```

两个节点都可能产生消息。

但并不是所有节点的消息都应该直接展示给用户。

### 5.3 为什么需要过滤

当前代码：

```python
text = text_of(chunk)

if not text:
    continue

if node and not node.startswith(
    ("agent", "respond", "chat")
):
    continue
```

第一层过滤空文本。

第二层按照节点名称过滤。

`continue` 表示跳过本次循环，继续处理下一个事件。

不过节点名称过滤只是一种简单策略。

真实 Agent 还可能产生工具调用、内部分析事件、结构化数据等内容。

更可靠的做法是明确规定哪些事件允许对外输出。

---

# 第三部分：HTTP Streaming 与 SSE

## 六、StreamingResponse：怎样将生成器连接到 HTTP

FastAPI 提供：

```python
from fastapi.responses import StreamingResponse
```

使用：

```python
return StreamingResponse(
    gen(),
    media_type="text/event-stream"
)
```

这里 `gen()` 是一个异步生成器。

`StreamingResponse` 会逐次消费它产生的数据，并将数据作为 HTTP 响应体发送。

### 6.1 它与普通响应的区别

普通响应：

```text
执行函数
   │
   ▼
得到完整结果
   │
   ▼
HTTP 返回
```

流式响应：

```text
执行生成器
   │
   ├── yield 第一段 → 发送
   │
   ├── yield 第二段 → 发送
   │
   ├── yield 第三段 → 发送
   │
   └── 结束
```

但这里有一个前提：

服务器、代理和客户端都需要正确支持流式传输。

如果中间代理缓冲响应，即使服务器逐段产生数据，客户端也可能一次性收到。

---

## 七、SSE：什么是服务端事件

SSE 全称 Server-Sent Events。

它是一种基于 HTTP 的服务端向客户端持续发送事件的机制。

它不是 LangGraph 专有的协议。

### 7.1 SSE 帧结构

```text
event: token
data: {"t":"你好"}

```

主要字段：

- `event`：事件类型。
- `data`：事件内容。
- 空行：一个事件结束。

可以定义：

```text
event: token
data: {"t":"你好"}

event: token
data: {"t":"世界"}

event: done
data: {"ok":true}

```

客户端根据事件类型处理数据。

### 7.2 SSE 编码函数

```python
import json

def sse(obj: dict, event: str | None = None) -> str:
    prefix = f"event: {event}\n" if event else ""

    data = json.dumps(
        obj,
        ensure_ascii=False
    )

    return f"{prefix}data: {data}\n\n"
```

这里需要理解两个细节。

**第一，为什么用 json.dumps？**

因为消息可能包含换行、引号等特殊字符。

JSON 编码可以保证这些字符被正确表示，避免直接拼接文本破坏当前 SSE 帧格式。

**第二，为什么结尾是两个换行？**

```python
"\n\n"
```

因为 SSE 使用空行标记事件结束。

### 7.3 SSE 和 WebSocket 的区别

| 对比 | SSE | WebSocket |
|---|---|---|
| 方向 | 服务端向客户端推送 | 双向通信 |
| 基础 | HTTP 响应流 | WebSocket 连接 |
| 典型用途 | 模型输出、通知、进度 | 实时协作、双向交互 |
| 实现复杂度 | 通常较低 | 通常较高 |

普通聊天 Agent 的逐段回答，可以先考虑 SSE。

如果需要持续的双向实时消息交互，再评估 WebSocket。

SSE 不是所有实时通信场景的唯一选择。

---

## 八、完整的流式接口

将前面的知识组合起来：

```python
@router.post("/chat/stream")
async def chat_stream(
    req: ChatRequest,
    request: Request,
    graph=Depends(get_graph)
):
    config = {
        "configurable": {
            "thread_id": req.session_id
        }
    }

    async def gen():
        n = 0

        try:
            async for chunk, meta in graph.astream(
                {"messages": [("user", req.message)]},
                config,
                stream_mode="messages"
            ):
                node = meta.get("langgraph_node", "")
                text = text_of(chunk)

                if not text:
                    continue

                n += 1

                yield sse(
                    {"i": n, "t": text, "node": node},
                    event="token"
                )

                if await request.is_disconnected():
                    break

            yield sse(
                {"done": True, "n": n},
                event="done"
            )

        except asyncio.CancelledError:
            raise

        except Exception:
            yield sse(
                {"error": "stream_failed"},
                event="error"
            )
            yield sse(
                {"done": True, "ok": False},
                event="done"
            )

    return StreamingResponse(
        gen(),
        media_type="text/event-stream",
        headers=SSE_HEADERS
    )
```

这段代码是整个流式知识的组合点。

阅读时不要逐行死记，而是识别四个阶段：

```text
① 准备
   └── 构造 thread_id

② 执行
   └── graph.astream()

③ 转换
   └── chunk → text → SSE

④ 交付
   └── StreamingResponse
```

还有一个需要注意的地方：客户端断开后，不能简单地认为已经成功完成本次执行。实际是否中断、是否保存部分状态，以及是否需要发送 `done`，都应由服务的取消策略决定。

---

# 第四部分：Checkpointer 与会话管理

## 九、session_id 与 thread_id

客户端发送：

```json
{
  "message": "你好",
  "session_id": "user_001"
}
```

FastAPI 接收到的是业务层的 `session_id`。

而 LangGraph Checkpointer 通过 `thread_id` 定位会话状态。

因此需要映射：

```python
def cfg(session_id: str) -> dict:
    return {
        "configurable": {
            "thread_id": session_id
        }
    }
```

调用：

```python
graph.ainvoke(
    {"messages": [("user", "你好")]},
    cfg("user_001")
)
```

这里可以理解为：

```text
业务会话标识
session_id
    │
    ▼
cfg()
    │
    ▼
LangGraph 配置
thread_id
    │
    ▼
Checkpointer
    │
    ▼
指定会话的状态
```

### 9.1 为什么要封装 cfg

虽然它只有几行，但它负责统一数据转换契约。

如果每个接口都自己构造：

```python
{"configurable": {"thread_id": session_id}}
```

以后修改规则时，就需要在多个位置同步调整。

封装之后，转换逻辑集中管理。

但也不意味着所有只有一行的代码都必须封装。

判断标准是：

**这段代码是否代表一个需要统一维护的规则，而不仅仅是重复的语法。**

---

## 十、aget_state：读取现有状态

```python
state = await graph.aget_state(
    cfg(session_id)
)
```

`aget_state()` 获取指定配置对应的状态快照。

读取消息：

```python
msgs = state.values.get("messages", [])
```

这里的 `state` 不是普通的 `ChatState` 字典，而是包含状态值、配置等信息的状态快照对象。

`state.values` 才是当前状态值。

### 10.1 aget_state 和 ainvoke 的区别

| 方法 | 用途 | 是否执行 Graph 节点 |
|---|---|---|
| `ainvoke()` | 执行 Graph | 是 |
| `astream()` | 执行并产生流事件 | 是 |
| `aget_state()` | 查询状态快照 | 否 |
| `adelete_thread()` | 删除指定线程的持久化数据 | 否 |

当需求只是查看历史时，不应该为了获取消息而重新执行 Graph。

### 10.2 将内部消息转换成 API 数据

```python
items = []

for m in msgs[-limit:]:
    text = text_of(m)

    items.append({
        "role": getattr(m, "type", "unknown"),
        "content": text[:300],
        "truncated": len(text) > 300,
    })
```

这里完成三件事：

1. 截取需要返回的消息。
2. 将内部消息对象转换成字符串与角色。
3. 限制单条消息的展示长度。

注意，`text[:300]` 是字符截取，不是 Token 截取。

这种转换属于数据适配层。

它让 API 不必直接暴露 LangGraph 内部对象。

---

## 十一、adelete_thread：删除会话状态

```python
await graph.checkpointer.adelete_thread(
    session_id
)
```

这个方法通过 Checkpointer 删除指定 `thread_id` 对应的持久化数据。

但实际工程还需要判断：

- 当前 Checkpointer 是否支持删除。
- 删除失败应该返回什么。
- 是否存在正在执行的同一会话请求。
- 删除后是否需要验证状态。
- 调用者是否有权删除该会话。

### 11.1 删除接口的正确性

不能仅仅写：

```python
return {"cleared": True}
```

就认为删除成功。

应该根据实际操作结果判断。

当前源码通过重新查询消息数验证删除效果：

```python
state = await graph.aget_state(cfg(session_id))
remaining = len(state.values.get("messages", []))
```

这可以检查当前可见消息是否清空。

但消息数为零并不等于已经证明数据库中所有相关记录都彻底消失。

更严格的删除验证应依据存储层的具体实现。

---

# 第五部分：并发、共享状态与竞态条件

## 十二、为什么多个请求会发生串台

考虑一个错误设计：

```python
CURRENT = {"thread_id": "default"}

@app.post("/bad")
async def bad(req: Req):
    CURRENT["thread_id"] = req.session_id

    await asyncio.sleep(0.2)

    tid = CURRENT["thread_id"]
```

`CURRENT` 是模块级全局可变字典。

多个请求会修改同一份数据。

假设：

```text
请求 A                     请求 B
   │                          │
   ▼                          │
CURRENT = A                   │
   │                          │
   ▼                          │
await sleep                   │
   │                          ▼
   │                      CURRENT = B
   │                          │
   ▼                          │
恢复执行                       │
   │                          │
   ▼                          │
读取 CURRENT = B              │
```

请求 A 读取到了请求 B 修改后的值。

这种结果取决于多个任务的执行交错顺序，属于竞态条件。

### 12.1 为什么 await 会暴露问题

在 `await` 处，当前协程可能暂停，事件循环可以运行其他任务。

其他任务修改了共享变量后，原协程恢复时就可能读到不同的数据。

但并不是 `await` 本身造成数据错误。

根本原因是：

**不同请求使用共享可变状态保存本应独立的数据。**

---

## 十三、怎样隔离请求状态

正常接口中：

```python
async def chat(req: ChatRequest, graph=Depends(get_graph)):
    config = cfg(req.session_id)

    result = await graph.ainvoke(
        {"messages": [("user", req.message)]},
        config
    )
```

每次请求都创建自己的 `config`。

```text
请求 A                  请求 B
   │                       │
   ▼                       ▼
config(A)               config(B)
   │                       │
   └──────────┬────────────┘
              │
         共享 Graph
              │
         Checkpointer
              │
      ┌───────┴───────┐
      ▼               ▼
   Thread A        Thread B
```

这里要区分两个概念：

**共享 Graph 对象**：可以作为应用级资源复用。

**共享请求状态**：必须遵守明确的隔离规则。

### 13.1 thread_id 能解决所有并发问题吗

不能。

不同 `thread_id` 可以让 Checkpointer 区分状态。

但如果同一个 `thread_id` 同时收到两个请求：

```text
请求 A：读取状态 V1
请求 B：读取状态 V1
请求 A：写入状态 V2
请求 B：写入状态 V3
```

就需要进一步考虑执行顺序、冲突和状态一致性。

具体行为取决于运行时与持久化实现，不能假设最后一定包含两次更新。

常见策略包括：

- 同一会话请求串行执行。
- 对同一会话加锁。
- 使用版本或冲突检测。
- 明确拒绝同一会话的重叠执行。

因此：

> thread_id 解决状态定位问题，并不自动解决同一会话的所有并发一致性问题。

---

## 十四、asyncio.gather：如何制造并发测试

当前测试使用：

```python
rs = await asyncio.gather(
    chat(c, "u1", "我叫小明"),
    chat(c, "u2", "我叫小红"),
    chat(c, "u1", "我今年 30"),
    chat(c, "u2", "我今年 25"),
)
```

`asyncio.gather()` 可以并发运行多个可等待对象，并在完成后按传入顺序返回结果。

这里要特别注意：

**结果列表的顺序，不等于任务实际完成的顺序。**

例如：

```text
任务 A ──────────────完成
任务 B ─────完成
任务 C ─────────完成
```

虽然 B 先完成，但 `gather(A, B, C)` 返回的结果仍按 A、B、C 的传入顺序排列。

### 14.1 并发测试需要验证什么

仅仅发出多个请求，不等于已经证明并发安全。

至少应该区分：

| 测试 | 验证目标 |
|---|---|
| 不同会话并发 | 是否出现跨会话数据 |
| 相同会话并发 | 是否丢失更新或顺序混乱 |
| 服务重启 | Checkpoint 是否持久化 |
| 请求取消 | 是否正确停止执行 |
| 删除与聊天同时发生 | 状态是否一致 |
| 模型失败 | 是否返回正确错误事件 |

自动化测试应该围绕可观察的行为和断言设计，而不是只检查程序有没有报错。

---

# 第六部分：异常、取消与可靠性

## 十五、流式接口为什么不能只依赖 HTTP 状态码

普通接口可以在执行失败时返回：

```python
raise HTTPException(
    status_code=500,
    detail="执行失败"
)
```

但流式响应不同。

一旦响应头已经发送，HTTP 状态码通常不能再修改。

因此需要在事件流中表达执行结果。

```python
try:
    async for chunk, meta in graph.astream(...):
        yield sse({"t": text_of(chunk)}, event="token")

except Exception:
    yield sse({"error": "stream_failed"}, event="error")
```

客户端必须理解：

```text
HTTP 200
    │
    └── 表示 HTTP 请求已开始成功响应

event: done
    │
    └── 表示应用层报告执行结束

event: error
    │
    └── 表示应用层报告执行错误
```

因此 HTTP 成功和 Agent 执行成功不是同一个判断。

---

## 十六、CancelledError：客户端断开后怎么办

当前代码：

```python
except asyncio.CancelledError:
    raise
```

`CancelledError` 表示异步任务收到取消请求。

重新抛出可以保留取消语义，让上层框架执行相应的清理流程。

但取消不一定代表底层模型服务已经立即停止工作。

还需要考虑：

- 正在进行的网络调用是否支持取消。
- 已产生的部分状态是否被保存。
- 是否需要清理资源。
- 客户端是否会重试。

这些属于运行时与资源生命周期的边界问题。

---

# 第七部分：代码检查与容易踩坑的地方

## 十七、文本提取不能破坏原文

当前源码中：

```python
text = "".join(text_of(m).split())
```

会删除所有空白字符。

例如：

```text
Hello world
```

会变成：

```text
Helloworld
```

如果只是想跳过空消息，可以改成：

```python
text = text_of(m)

if text.strip():
    return text
```

`strip()` 用于判断是否只有首尾空白，但最终返回原文。

另外，如果函数目标是提取 AI 回答，还应该检查消息角色，而不是简单返回最后一条非空消息。

---

## 十八、Graph 多节点输出可能导致重复回答

当前 Graph：

```text
START
  │
  ▼
agent
  │
  ▼
respond
  │
  ▼
END
```

两个节点都会向 `messages` 写入 AI 消息。

流式输出如果同时放行两个节点，客户端可能收到两段不同的回答。

这提醒我：

**Graph 内部消息与客户端可见消息不是同一个概念。**

在更复杂的 Agent 中，应定义明确的输出契约。

例如，规定只有某个最终回答节点的输出可以展示给用户。

---

## 十九、数据验证不等于权限验证

当前：

```python
session_id: str = Field(
    pattern=r"^[A-Za-z0-9_-]+$"
)
```

限制的是会话 ID 格式。

但它不能证明请求者有权访问这个会话。

真实服务还需要建立：

```text
用户身份
    │
    ▼
身份认证
    │
    ▼
会话所有权校验
    │
    ▼
允许读取 / 删除
```

格式正确，只代表输入符合规则，不代表操作被授权。

---

# 第八部分：知识速查与复习

## 二十、核心 API 速查

| API / 语法 | 作用 | 需要记住 |
|---|---|---|
| `async def` | 定义协程函数或异步生成器 | 是否含 `yield` 会影响调用方式 |
| `await` | 等待异步操作 | 不等于创建线程 |
| `yield` | 逐次产生数据 | 保留生成器执行位置 |
| `async for` | 消费异步迭代器 | 常用于流式数据 |
| `asyncio.gather` | 并发等待多个任务 | 结果按传入顺序排列 |
| `graph.ainvoke()` | 异步执行 Graph | 获取最终结果 |
| `graph.astream()` | 获取 Graph 流事件 | 输出形状取决于 stream_mode |
| `graph.aget_state()` | 查询状态快照 | 不执行节点 |
| `adelete_thread()` | 删除线程持久化数据 | 取决于 Checkpointer 支持情况 |
| `StreamingResponse` | 发送 HTTP 流式响应 | 消费迭代器产生的数据 |
| `json.dumps()` | JSON 序列化 | 处理特殊字符 |
| `request.is_disconnected()` | 检查客户端连接状态 | 不能替代完整取消策略 |
| `asyncio.CancelledError` | 任务取消信号 | 通常应保留取消语义 |

## 二十一、容易混淆的概念

| 概念 A | 概念 B | 核心区别 |
|---|---|---|
| `ainvoke` | `astream` | 最终结果 vs 流事件 |
| `await` | `yield` | 等待 vs 产出 |
| LangGraph Streaming | HTTP Streaming | 内部执行事件 vs 网络传输 |
| SSE | WebSocket | 单向推送 vs 双向通信 |
| `session_id` | `thread_id` | 业务标识 vs LangGraph 状态标识 |
| `app.state` | `ChatState` | 应用资源 vs Graph 业务状态 |
| `aget_state` | `ainvoke` | 读取状态 vs 执行 Graph |
| 并发 | 并行 | 任务交错推进 vs 同时实际执行 |
| 状态隔离 | 权限隔离 | 数据分开存储 vs 谁有权访问 |

## 二十二、复习时应该能回答的问题

**基础理解**

1. 为什么 `async def` 不代表所有代码都会自动非阻塞？
2. `yield` 与 `return` 的区别是什么？
3. 为什么异步生成器需要使用 `async for`？
4. `astream()` 中的 `chunk` 和 `metadata` 从哪里来？
5. 为什么不同 `stream_mode` 的输出不能使用相同的解析方式？

**机制理解**

6. 为什么已经有 `astream()`，还需要 `StreamingResponse`？
7. SSE 为什么需要空行结束一个事件？
8. 为什么流式接口返回 HTTP 200 后仍可能执行失败？
9. 为什么查看历史使用 `aget_state()` 而不是 `ainvoke()`？
10. 为什么共享 Graph 不一定导致会话串台？

**工程边界**

11. 为什么全局可变字典可能导致请求串台？
12. 不同 `thread_id` 与同一 `thread_id` 的并发问题有什么区别？
13. 客户端断开连接后，模型是否一定停止执行？
14. 为什么会话 ID 的格式验证不能替代权限验证？
15. 怎样设计测试才能证明会话隔离，而不只是证明程序运行过？

---

## 二十三、最后恢复一次完整主链路

以后重新看这篇文章时，可以先看这张图：

```text
HTTP POST /chat/stream
        │
        ▼
FastAPI
        │
        ├── ChatRequest 验证
        ├── Depends(get_graph)
        └── cfg(session_id)
                │
                ▼
          graph.astream()
                │
                ├── Checkpointer
                │      └── 根据 thread_id 恢复状态
                │
                ├── LangGraph Runtime
                │      └── 执行节点
                │
                └── messages 模式
                       │
                       ▼
                 chunk + metadata
                       │
                       ▼
                    text_of()
                       │
                       ▼
                     sse()
                       │
                       ▼
               StreamingResponse
                       │
                       ▼
                    客户端
```

这次学习真正需要掌握的，不是背下一个流式接口的完整代码。

而是理解三个边界：

**第一，执行边界。** LangGraph 负责执行 Graph，`ainvoke` 和 `astream` 决定调用方如何获取结果。

**第二，通信边界。** FastAPI 通过 `StreamingResponse` 将内部数据转换为 HTTP 响应，SSE 定义客户端怎样理解事件。

**第三，状态边界。** Checkpointer 根据 `thread_id` 管理状态，但并发一致性、权限、删除验证和异常处理仍需要应用层设计。

理解这些机制后，即使未来换掉 LangGraph、模型服务或前端框架，也能根据同样的问题重新设计服务。