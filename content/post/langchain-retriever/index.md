---
description: ""
title: "LangChain Retriever"
draft: false
date: "2026-09-01T11:20:01+08:00"
slug: "langchain-Retriever"
categories:
 - LangChain
 - Agent
tags:
 - Memory
image: ""
---

---
title: "W4-D5 · Retriever 转正 + Tool 调用：把 W3 的记忆和 W2 的工具全部接进 LangChain"
slug: "lc-tools-retriever"
date: 2026-09-01
description: "D4 收尾：把 RunnableLambda 手工包的假 retriever 升级成官方 BaseRetriever 子类；D5 正篇：用 @tool + bind_tools 把 W2 手写的 fc_loop 翻译成框架标准写法，含三个工具的完整实战与五个踩坑。"
categories:
  - Agent
  - LangChain
tags:
  - LCEL
  - BaseRetriever
  - bind_tools
  - Function Calling
  - DeepSeek
---

# W4-D5 · Retriever 转正 + Tool 调用

> **关键词**：`BaseRetriever` / `@tool` / `bind_tools` / `create_retriever_tool` / 工具调用循环
> **前置**：W1 `llm_client`、W2 `fc_loop`、W3 `MemoryManager`、W4-D1~D3（LCEL 三件套 + 组合件）
> **本篇覆盖**：D4 的"剩下部分"（临时工转正） + D5 正篇（Tools & Tool Calling）
> **涉及文件**：`W4/lc_retriever_w3.py`、`W4/retriever.py`、`W4/lc_tools_multi.py`
> **收尾预告**：D6 把 `memory_tool` 和多工具一起挂进 Agent，届时 W2 的自研循环彻底退休

* * *

## 零、先定位：今天要补的是哪两块

先把范围钉死，免得学晕。

你 D4 前半段为了**快速跑通**，用 `RunnableLambda` 手动包了一个"假 retriever"。能用，但那玩意儿在 LangChain 生态里**没有名分**——框架不知道它是个检索器，所以官方的工具封装不认它。

今天两件事：

```
┌──────────────────────────────────────────────────────────────┐
│  第一块  D4 收尾：临时工 → 正式工                              │
│                                                                │
│  RunnableLambda 手工包          ──▶   BaseRetriever 子类        │
│  （能用，但没工牌）                    （活一样干，门禁全开）    │
├──────────────────────────────────────────────────────────────┤
│  第二块  D5 正篇：Tool & Tool Calling                          │
│                                                                │
│  W2 手写的 fc_loop              ──▶   @tool + bind_tools        │
│  （自己写 schema、自己写循环）         （schema 自动生成，        │
│                                         循环逻辑原样保留）      │
└──────────────────────────────────────────────────────────────┘
```

**这两块是咬合的**——第一块转正后的 retriever，正是第二块里 `memory_tool` 的直接原料。没有第一块"转正"，第二块的长期记忆工具就得自己手写适配。这也是为什么我把它们放一篇讲。

* * *

## 一、D4 收尾：临时工 → 正式工

### 1.1 先看你现在的"临时工"（`W4/lc_retriever_w3.py` 实况）

你 D4 前半段写的代码长这样：

```python
def make_retriever(mm, k=3):
    """把 mm.recall 包成 LangChain 标准 Retriever：query -> list[Document]"""
    def recall_docs(query: str) -> list[Document]:
        facts = mm.recall(query, k=k)
        return [Document(page_content=f, metadata={"source": "lang_term"}) for f in facts]
    return RunnableLambda(recall_docs)

retriever = make_retriever(mm, k=3)
docs = retriever.invoke("用户喜欢什么模型？")
```

**它干得挺好**——查询进、Document 列表出，还能直接进 LCEL 链：

```python
chain = (
    {"context": retriever | _ftm, "question": RunnablePassthrough()}
    | prompt
    | model
    | StrOutputParser()
)
```

> 💡 顺带一个你代码里已经在用的细节：`retriever | _ftm` 里的 `_ftm` 是**普通函数**，没包 `RunnableLambda`。这不是笔误——LCEL 的 `|` 遇到普通函数会**自动 coerce** 成 `RunnableLambda`。手写包装只在你想显式表达时才需要。

**但它是临时工。** 为什么？往下看。

### 1.2 为什么要转正（三个理由，面试能讲）

| 理由 | 说明 |
|---|---|
| **接口白送** | D1 学的 `invoke / stream / batch` 三接口免费获得 |
| **生态识别** | 能被 `create_retriever_tool`、LangGraph 节点、`RetrievalQA` 等标准组件直接使用 |
| **类型安全** | 字段用 pydantic 声明（`mm`、`k`），有类型检查、能序列化保存 |

**第一条要说明白**：临时工其实也有 `invoke/stream/batch`——因为 `RunnableLambda` 本身就是 Runnable。所以"接口白送"的准确说法不是"从无到有"，而是**"从借用身份到原生身份"**。

真正的分水岭是**第二条：生态识别**。

```
临时工 RunnableLambda              正式工 BaseRetriever
─────────────────────              ─────────────────────
框架眼里：一个普通函数               框架眼里：一个检索器
  · 能 invoke            ✅           · 能 invoke               ✅
  · 能进链               ✅           · 能进链                  ✅
  · create_retriever_tool ❌ 不认      · create_retriever_tool    ✅ 认
  · LangGraph 检索节点    ❌ 不认      · LangGraph 检索节点       ✅ 认
  · RetrievalQA          ❌ 不认      · RetrievalQA             ✅ 认
```

> **类比**：临时工和正式工都干活，但正式工有工牌（`BaseRetriever` 这个父类）——门禁（`invoke/stream/batch`）、培训体系（LangGraph 节点、工具封装）都对他开放。没工牌的临时工，每进一道门都要你亲自带路。

### 1.3 正式版骨架：`MemoryRetriever(BaseRetriever)`

你本地 `W4/retriever.py` 的完整实现：

```python
# BaseRetriver 子类
from typing import Any
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from pydantic import Field, ConfigDict


class MemoryRetriever(BaseRetriever):
    """把 W3 的 Memory Manager.recall 包装成官方的 Retriever
       BaseRetriever 是个 pydantic 模型，所以字段要声明类型 + Field
    """

    model_config = ConfigDict(arbitrary_types_allowed=True)  # 允许非 pydantic 对象
    mm: Any = Field(description="带 recall(query, k) 方法的记忆管理器")
    k: int = Field(default=3, description="召回条数")

    def _get_relevant_documents(self, query: str, **kwargs) -> list[Document]:
        """唯一必须实现的抽象方法：query → Document 列表。

        **kwargs 兼容新旧版本签名（旧版无 run_manager，新版有）。
        """
        facts = self.mm.recall(query, k=self.k)          # W3 的召回
        return [
            Document(page_content=f, metadata={"source": "长期记忆"})
            for f in facts
        ]
```

**注意两个细节**（你的代码已经做对了）：

- `metadata` 从临时工版的 `"lang_term"` 改成了 `"长期记忆"`。转正了嘛，档案袋上的标签也得写正楷。
- `k` 从**闭包变量**变成了**类字段**——这是"可配置"和"硬编码"的区别，正式工的参数是登记在册的。

* * *

### 1.4 🎯 关键点 1：`_get_relevant_documents` 是唯一必须实现的抽象方法

它回答一句话：**"给我一句 query，还你相关文档列表。"**

LangChain 内部所有调用（`invoke` / `stream` / `batch`）**最终都会走到这个方法**——你只写这一个，剩下的框架替你管。

```
  retriever.invoke("用户喜欢什么颜色")
            │
            ▼
  BaseRetriever.invoke()          ← 框架自带：参数校验、回调、配置注入
            │
            ▼
  _get_relevant_documents(query)  ← 🎯 只有这一块是你写的
            │
            ▼
  [Document, Document, Document]
```

> **类比**：你只负责"**按关键词翻档案柜**"这一个动作（`_get_relevant_documents`），至于"客人怎么下单（invoke）、怎么批量查（batch）、怎么跟踪调用记录"——那是工牌（`BaseRetriever`）自带的，不用你写。

#### 为什么签名里必须有 `**kwargs`

这是**版本兼容保险**：

- 老版本签名：`_get_relevant_documents(self, query: str)`
- 新版本签名：`_get_relevant_documents(self, query: str, *, run_manager: CallbackManagerForRetrieverRun)`

新版多塞了一个 `run_manager` 参数进来。如果你只写 `(self, query)`，新版调用时会因为你"接不住"而报 `TypeError`。

**写 `**kwargs` 两种都能跑**——多传的参数被 kwargs 兜住，不传也不影响。

> ⚠️ 这和你 D1 踩的 `StrOutputParser` 返回类型坑是**同一类问题**：框架内部实现会随版本变，但**对外的调用约定不会变**。防御性写法就是在边界处留 `**kwargs`，把变化挡在门外。

### 1.5 🎯 关键点 2：invoke / stream / batch 白送，且原生进链

`BaseRetriever` 本身就继承 `Runnable`（和 `prompt | model | parser` 里的每个零件同源），所以 D1 学的三接口直接能用：

```python
docs = retriever.invoke("用户喜欢什么颜色")       # list[Document]
docs_list = retriever.batch(["问题1", "问题2"])   # 批量，框架自动并发
```

**而且能直接进 LCEL 链**——D2 学的组合件全部通用：

```python
from langchain_core.runnables import RunnableLambda

chain = retriever | RunnableLambda(
    lambda docs: "；".join(d.page_content for d in docs)
)
result = chain.invoke("用户喜欢什么模型？")   # 返回纯字符串
```

> **类比**：正式工（`BaseRetriever`）**自带门禁卡**（Runnable 协议），D1 你学的 `invoke/stream/batch` 三孔插头直接能插；临时工（`RunnableLambda` 手工包）得你每次自己手动接电。

这里要说句公道话：临时工也能进链（你 `lc_retriever_w3.py` 里就进了）。**区别在于"谁的身份"**——临时工进链是"借了个 Runnable 的马甲"，正式工进链是"以检索器的身份进的"，后者才能被检索器专属组件识别。

### 1.6 🎯 关键点 3：为什么字段要 `Field` + `ConfigDict`

这是新手最容易卡住的地方。`BaseRetriever` **不是普通类，是 pydantic 模型**——所以：

```
普通类                          pydantic 模型（BaseRetriever）
─────────                       ────────────────────────────
self.mm = mm                    mm: Any = Field(description="...")
self.k = 3                      k: int = Field(default=3, ...)
（__init__ 里随便赋值）          （类级别声明类型 + 元数据）
不做校验                         实例化时校验类型
不能序列化                       能 .model_dump() 存下来
```

**两道门槛**：

**① `mm` 和 `k` 必须声明类型 + Field**，否则实例化报错。pydantic 要求字段在类层面声明——你不能在 `__init__` 里 `self.mm = mm` 了事，那样 pydantic 根本不知道有这个字段。

**② `ConfigDict(arbitrary_types_allowed=True)`**——这道更关键。

pydantic 默认**只接受它能理解的类型**（str / int / list / dict 以及嵌套的 pydantic 模型）。你的 `MemoryManager` 是个普通 Python 类，pydantic 不认识它，默认会直接抛错：

```
pydantic.errors.PydanticSchemaGenerationError:
  Unable to generate pydantic-core schema for <class 'memory.memory_manager.MemoryManager'>
```

加上 `arbitrary_types_allowed=True` 等于告诉 pydantic：**"`mm` 里装的不是 pydantic 对象，但请放行。"**

> **类比**：正式工的工牌信息表（pydantic）要求"姓名、工号"必须填标准格式；`arbitrary_types_allowed` 就是备注栏——"允许填非标准信息"（比如你的 `MemoryManager` 实例）。

**为什么 `mm: Any` 而不是 `mm: MemoryManager`？**

你已经猜到了：如果写 `mm: MemoryManager`，除了 `arbitrary_types_allowed` 之外还得保证 import 路径稳定，而且后面写单元测试时**塞不进 `FakeMemory`**——类型不匹配。写 `Any` 是刻意的松绑：

- 生产环境塞真 `MemoryManager`
- 测试环境塞 `FakeMemory`（见 1.8）

真正约束 `mm` 行为的不是类型注解，而是 `Field(description="带 recall(query, k) 方法的记忆管理器")` 这句**契约说明**——这是鸭子类型：我不在乎你是什么类，我只在乎你有没有 `recall(query, k)`。

### 1.7 无缝替换：你已有的链一行不用改

```python
# 之前（临时工）:
# make_retriever = RunnableLambda(lambda q: [Document(page_content=f) for f in mm.recall(q)])

# 现在（正式工）:
from W4.retriever import MemoryRetriever

mm = MemoryManager(window=8, max_tokens=1500, keep=4)   # 不需要 llm，纯召回
retriever = MemoryRetriever(mm=mm, k=3)

# 用法完全一样，甚至更简单：
docs = retriever.invoke("用户喜欢什么颜色")
```

**你 `lc_retriever_w3.py` 里那条链一行都不用改**——因为 `retriever` 本身就是 Runnable，之前链里怎么用 `make_retriever(mm)` 就怎么用它：

```python
# 原来：retriever = make_retriever(mm, k=3)
# 现在：retriever = MemoryRetriever(mm=mm, k=3)
# 下面这段原封不动
chain = (
    {"context": retriever | _ftm, "question": RunnablePassthrough()}
    | prompt | model | StrOutputParser()
)
```

> ✅ **这就是"接口守恒"的价值**：你升级的是身份，不是接口。调用方毫无感知——好的抽象就该这样。

### 1.8 验证：不需要 API key 也能测

这是转正带来的**隐藏福利**——因为 `mm: Any`，你可以塞一个假记忆管理器进去：

```python
# test_retriever.py
from W4.retriever import MemoryRetriever

class FakeMemory:
    """模拟 MemoryManager.recall —— 不联网也能测"""
    def recall(self, query, k=3):
        return ["用户叫小明", "喜欢蓝色", "预算2000"]

r = MemoryRetriever(mm=FakeMemory(), k=3)

# 测试1：invoke 返回 list[Document]
docs = r.invoke("用户喜欢什么")
assert isinstance(docs, list) and len(docs) == 3
print(f"✓ invoke 返回 {len(docs)} 条 Document")

# 测试2：page_content 和 metadata 都在
print(f"✓ 内容: {docs[1].page_content} | 来源: {docs[1].metadata['source']}")

# 测试3：能直接进 LCEL 链
from langchain_core.runnables import RunnableLambda
chain = r | RunnableLambda(lambda ds: "；".join(d.page_content for d in ds))
print(f"✓ 链输出: {chain.invoke('用户喜欢什么')}")
```

预期输出：

```fallback
✓ invoke 返回 3 条 Document
✓ 内容: 喜欢蓝色 | 来源: 长期记忆
✓ 链输出: 用户叫小明；喜欢蓝色；预算2000
```

> 💡 **测试不需要真实记忆库**这件事很重要：`MemoryManager` 要连 Chroma、要读 `.env`、要落盘，任何一环出问题都会让"retriever 本身的逻辑对不对"这个判断变得模糊。**用 `FakeMemory` 把依赖切断，测的才是你真正想测的东西。**

### 1.9 转正对照表（背这一张够了）

| | `RunnableLambda` 手工包 | `BaseRetriever` 子类 |
|---|---|---|
| 三接口 | 借用 RunnableLambda 的 | 原生（Runnable 协议） |
| 进 LCEL 链 | 能用 | 原生支持 |
| 官方工具封装 | ❌ 不认 | ✅ `create_retriever_tool` |
| 类型安全 | 无 | pydantic 字段 |
| 可序列化 | ❌ | ✅ `.model_dump()` |
| 依赖注入测试 | 靠闭包 | 靠字段替换（`FakeMemory`） |
| 版本坑 | 少 | `**kwargs` 兼容即可 |

* * *

## 二、D5 核心心智：你 W2 的 fc_loop 就是今天的框架版

### 2.1 先吃定心丸：这不是新知识

**好消息：你 W2 已经写过 `fc_loop`，今天学的就是"用框架语法重新声明你做过的事"。**

看对照表——**你 W2 写过的每一块，今天都有对应物**：

| 你 W2 手写的 | LangChain 的 D5 对应物 | 本质 |
|---|---|---|
| `TOOLS_SCHEMA`（给模型看的说明书） | `@tool` 装饰器 + 函数 docstring | 工具说明（docstring 就是说明书） |
| `TOOLS_FUNC`（名字→函数的查表） | `tools_by_name = {t.name: t for t in tools}` | 查表执行 |
| `chat_for_tools(messages, tools)` | `model.bind_tools(tools)` | 把工具绑给模型 |
| `fc_loop`（while 循环解析 tool_calls） | `while ai.tool_calls: ...` 循环 | **同一个循环** |
| `{"role": "tool", "tool_call_id": tc.id, ...}` | `ToolMessage(content=..., tool_call_id=...)` | 结果回填 |

> **类比**：你 W2 是**自己开餐厅**——自己写菜单（schema）、自己当服务员（fc_loop）。D5 是**买了餐饮行业标准设备**——菜单用模板打印（`@tool`），服务员换成标准流程（`bind_tools`），但"点菜→上菜→结账"的流程还是你熟悉的那套。

**框架替你省掉的是"说明书的编写"，不是"循环的设计"。** 循环逻辑你一个字都不用改思路——只是换了个名字。

### 2.2 `@tool`：docstring 就是给模型看的说明书

```python
@tool
def get_weather(city: str) -> str:
    """查询指定城市的天气。参数 city: 城市名。"""
    return f"{city} 今天晴，26℃"
```

三件事在这一段里同时发生：

1. **函数被包装成 LangChain 的 `Tool` 对象**——从此它有 `.name`、`.description`、`.args_schema`、`.invoke()`。
2. **docstring 变成"工具说明书"**——模型靠它决定"这个问题该不该调这个工具、参数怎么填"。
3. **函数签名 `city: str` 自动生成参数 schema**——不用再手写 JSON Schema 了。

```
  你 W2 的手工活                      D5 的自动活
  ──────────────                      ────────────
  TOOLS_SCHEMA = [{                   def get_weather(city: str):
      "name": "get_weather",              """查询指定城市的天气。
      "description": "查询天气",             参数 city: 城市名。"""
      "parameters": {                           │
          "type": "object",                     │  @tool 自动生成
          "properties": {                       ▼
              "city": {"type": "string",    name="get_weather"
                       "description": ...}  description="查询指定城市的天气..."
          },                                args_schema={city: str}
          "required": ["city"]
      }
  }]
```

**写 docstring 的两条硬规矩**（这是实战经验，不是文档里的）：

- 🚫 **别只写"查询天气"**。要写清**什么时候该调**——模型判断调用时机靠的就是这句。
- ✅ **写"当用户询问……时调用"**。你 `make_web_search_tool` 里那句就写得很好：
  ```python
  """搜索网页信息。当用户询问需要联网查询的问题、最新动态时，调用此工具。
     参数 query: 搜索关键词; max_results: 返回条数(默认三条)"""
  ```
  前半句说"干什么"，后半句说"什么时候干"——**这就是一份合格的说明书**。

> ⚠️ **docstring 写含糊的代价**：模型要么该调不调（直接瞎编天气），要么乱调（问"你好"也去搜网页）。这不是模型的锅，是说明书没写清。

### 2.3 `bind_tools`：把工具"绑"给模型

```python
model = ChatOpenAI(...).bind_tools(tools)
```

- 等价于你 W2 的 `llm.chat_for_tools(messages, tools)`——把工具列表**预先绑**在模型上。
- 之后每次 `model.invoke(messages)`，模型**自己决定**要不要返回 `tool_calls`。
- 绑定后，模型返回的消息里出现 `ai.tool_calls` 就是"模型发起了工具调用"的信号。

```
  model = ChatOpenAI(...)                    ← 普通模型：只能说话
           │
           │  .bind_tools(tools)
           ▼
  model_with_tools                           ← 带登记簿的模型：能说话，也能填单
           │
           │  .invoke(messages)
           ▼
  AIMessage(content="", tool_calls=[...])    ← 想调工具时，content 为空、tool_calls 有货
  或
  AIMessage(content="北京今天晴", tool_calls=[])  ← 不用工具时，直接给答案
```

> **类比**：给服务员一本**登记簿**（`bind_tools`），他看到"需要查资料"就会主动填表（返回 `tool_calls`）——不用你每次手动递登记簿。**注意 `bind_tools` 返回的是新对象**，原 `model` 不受影响，这符合 LCEL 一贯的不可变风格。

### 2.4 工具调用循环：就是你 W2 的 fc_loop，原样搬

```python
ai = model.invoke(messages)          # ① 让模型回答
while ai.tool_calls:                 # ② 模型要调工具？
    for call in ai.tool_calls:
        result = tools_by_name[call["name"]].invoke(call["args"])              # ③ 执行工具
        messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))  # ④ 回填
    ai = model.invoke(messages)      # ⑤ 带着结果再问模型
return ai.content                    # ⑥ 无 tool_calls → 最终答案
```

**逐行对照你 W2 的 `fc_loop`**：

| W2 的写法 | D5 的写法 |
|---|---|
| `resp.choices[0].message` | `ai` |
| `tc.function.name` | `call["name"]` |
| `tc.function.arguments` | `call["args"]` |
| `{"role": "tool", "tool_call_id": tc.id, ...}` | `ToolMessage(content=..., tool_call_id=...)` |

**流程一模一样，只是名字换成框架的。**

> **类比**：循环的本质是"**模型 → 工具 → 回填 → 再问**"——就像你打电话问同事（工具）要数据，拿到后转述给领导（模型）做决定。D5 把"打电话"封装成了标准动作，但你还是在打这个电话。

* * *

## 三、D5 实战：三个工具 + 一个循环（`W4/lc_tools_multi.py`）

原理讲完，看你本地这份代码——它比任务表的示例**实战得多**，我逐块拆。

### 3.0 全景：这份文件在干什么

```
┌──────────────────────────────────────────────────────────────────┐
│                     W4/lc_tools_multi.py                          │
│                                                                    │
│   【三个工具】                                                      │
│   ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────┐  │
│   │ get_weather     │ │ search_web      │ │ memory_tool      │  │
│   │ @tool 直出      │ │ 工厂函数 + 闭包  │ │ 由 D4 正式工      │  │
│   │ 无状态，最简单   │ │ 注入 TavilyClient│ │ retriever 转换来  │  │
│   └────────┬────────┘ └────────┬────────┘ └────────┬─────────┘  │
│            └───────────────────┼───────────────────┘              │
│                                ▼                                   │
│                     tools = [三个工具]                             │
│                     tools_by_name = {名字: 工具}                   │
│                                │                                   │
│                                ▼                                   │
│                  model.bind_tools(tools)                           │
│                                │                                   │
│                                ▼                                   │
│   ┌────────────────────────────────────────────────────────────┐ │
│   │  run(query) —— 手写循环                                      │ │
│   │  while ai.tool_calls and round_ < MAX_ROUNDS:               │ │
│   │      args = _normalize_args(call["args"])   ← 参数归一化     │ │
│   │      result = tools_by_name[名字].invoke(args)              │ │
│   │      messages.append(ToolMessage(...))                       │ │
│   └────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

### 3.1 工具一：`get_weather`——最简形态

```python
@tool
def get_weather(city: str) -> str:
    """查询指定城市的天气。参数 city: 城市名。"""
    return f"{city} 今天晴，26℃"
```

**这是 `@tool` 的最小可跑形态**：一个函数 + 一句 docstring + 类型注解。适合无状态、无外部依赖的工具。

> 💡 真实项目里这里该调天气 API。学习期用假数据**完全没问题**——你要练的是"工具怎么接进循环"，不是"天气怎么查"。**先把管道跑通，再换真实数据源**，这也是你 Vibe Coding 手册里"小步快跑"的原则。

### 3.2 🎯 工具二：`search_web`——工厂函数 + 闭包注入（重点）

这是整份代码里**最有技术含量的一块**，也是你 W2 没遇到过的场景：**工具有状态时怎么办？**

`TavilyClient` 需要先拿 API key 初始化，是个活对象。但 `@tool` 装饰的函数**签名必须是模型能填的参数**——你不可能让模型去填 `client` 这个参数。

**解法：工厂函数 + 闭包。**

```python
def make_web_search_tool(client=None):
    """工厂函数：把有状态的 client 闭包捕获，返回 @tool 工具"""
    if client is None:
        from tavily import TavilyClient
        client = TavilyClient(api_key=os.environ.get("TAVILY_API_KEY"))

    @tool
    def search_web(query: str, max_results: int = 3) -> str:
        """搜索网页信息。当用户询问需要联网查询的问题、最新动态时，调用此工具。
           参数 query: 搜索关键词; max_results: 返回条数(默认三条)"""
        result = client.search(query, max_results=max_results)
        return json.dumps(result, ensure_ascii=False)

    return search_web
```

拆开看这三层：

```
  第一层：make_web_search_tool(client)      ← 工厂：负责"生"工具
              │
              │  client 被闭包捕获（不在 @tool 的签名里）
              ▼
  第二层：@tool def search_web(query, max_results)  ← 模型只看见这两个参数
              │
              │  函数体内部直接用外层的 client
              ▼
  第三层：return search_web                  ← 交出一个"带着 client 的工具"
```

**为什么必须这么绕？** 因为模型填参数的能力边界就到 `query` 和 `max_results` 为止。让模型填 `client`，它只会编一个字符串出来，然后 `client.search()` 直接炸。

```
  ❌ 错误想法：把 client 当参数
     @tool
     def search_web(client, query): ...     # 模型不知道 client 该填什么

  ✅ 正确做法：闭包在"定义时"注入，模型只在"调用时"填业务参数
     def make_web_search_tool(client):      # 定义时注入依赖
         @tool
         def search_web(query): ...         # 调用时只需业务参数
```

> **类比**：这就像**给快递员配好电动车再派单**——工厂函数负责"配车"（把 `TavilyClient` 装进闭包），`@tool` 函数负责"接单跑腿"（只接收地址 `query`）。你不会让客户在下单时顺便指定"用哪辆电动车"。

**这个模式要记牢**——它是"有状态依赖注入 LangChain 工具"的标准解法，后面接数据库、接向量库、接你自研的 `MemoryManager` 全用这一招。

> 💡 `json.dumps(result, ensure_ascii=False)` 里的 `ensure_ascii=False`——你 `os_json` 那篇速查里专门写过：**存中文必加**。这里同理，不加的话搜到的中文内容会变成 `\u4e2d\u6587`，模型读着费劲。

### 3.3 🎯 工具三：`memory_tool`——D4 转正的直接红利

**第一块"转正"的回报在这一刻兑现**：

```python
from W4.retriever import MemoryRetriever
try:
    from langchain.tools.retriever import create_retriever_tool
except ImportError:
    from langchain_core.tools.retriever import create_retriever_tool

mm = MemoryManager(window=8, max_tokens=1500, keep=4)
mm.add({"role": "user", "content": "用户喜欢Deekseek，预算2000元"}, persist=True)
mm.add({"role": "user", "content": "用户叫小明，喜欢蓝色"}, persist=True)

memory_retriever = MemoryRetriever(mm=mm, k=3)
memory_tool = create_retriever_tool(
    memory_retriever,
    name="memory_retriever",
    description="从长期记忆检索用户偏好、历史事实。当用户问起以前说过/聊过的事情时调用"
)
```

**就这四行，你的 W3 长期记忆变成了一个模型能主动调用的工具。**

回想一下：如果 retriever 还停在临时工阶段（`RunnableLambda`），`create_retriever_tool` 根本不认它，你得自己手写工具 schema、自己拼返回值——**这正是"转正"省下的活**。

> **类比**：临时工版**没工牌**，官方工具封装（`create_retriever_tool`）不认它；正式版**有工牌**，官方接口直接认。**这就是"生态识别"的价值**——它不是个抽象概念，是实打实少写的代码。

**注意 `description` 的写法**：「当用户问起以前说过/聊过的事情时调用」——**明确告诉模型调用时机**。模型看到"用户喜欢什么模型，预算多少？"这种问题，才知道该翻记忆而不是去搜网页。

**三个工具合流**：

```python
search_web = make_web_search_tool()          # 闭包捕获 TavilyClient
tools = [get_weather, search_web, memory_tool]
tools_by_name = {t.name: t for t in tools}
```

`tools_by_name` 这一行看着不起眼，却是循环能跑通的关键——**模型返回的是工具名字（字符串），你得有张表把它映射回工具对象**。这和你 W2 的 `TOOLS_FUNC` 字典是同一个东西。

### 3.4 `bind_tools` 接 DeepSeek（你的环境必须改这一处）

任务表示例用的是 `gpt-4o-mini` + `OPENAI_API_KEY`，**你要换成 DeepSeek**：

```python
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,                    # ← 工具调用场景建议 0：要稳定，不要发散
).bind_tools(tools)
```

**DeepSeek 支持 tool-calling**（OpenAI 兼容协议的一部分），所以 `bind_tools` + `tool_calls` 循环直接能用——你 W1 靠 `base_url` 切到 DeepSeek 的那一招，到 D5 还在生效。

> ⚠️ **`temperature=0` 不是随手写的**：工具调用场景里，你要模型**稳定地决定"调不调、调哪个、参数填什么"**，发散的采样会带来随机性——同一句问话这次调工具下次不调，调试时能把人逼疯。**工具调用场景，温度一律压到 0。**

### 3.5 🎯 循环体：`MAX_ROUNDS` 与 `_normalize_args`（两个实战细节）

你本地的循环比任务表示例多两样东西，**这两样都是踩过坑才会加的**：

```python
def _normalize_args(args: dict) -> dict:
    for k, v in list(args.items()):
        while isinstance(v, dict) and v:
            v = next(iter(v.values()))
        args[k] = v
    return args

MAX_ROUNDS = 3

def run(query: str, verbose: bool = True):
    messages = [HumanMessage(query)]
    ai = model.invoke(messages)
    messages.append(ai)

    round_ = 0
    while ai.tool_calls and round_ < MAX_ROUNDS:
        round_ += 1

        if verbose:
            print(f"\n[第{round_}轮] 模型发起 {len(ai.tool_calls)} 个工具调用:")
            for call in ai.tool_calls:
                print(f"  工具: {call['name']}  参数: {call['args']}  解析后: {_normalize_args(call['args'])}")

        for call in ai.tool_calls:
            args = _normalize_args(call["args"])
            result = tools_by_name[call["name"]].invoke(args)
            messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
            if verbose:
                print(f"  工具返回 {str(result)[:80]}...")

        ai = model.invoke(messages)          # 带着结果再问模型
        messages.append(ai)

    if ai.tool_calls:
        print(f"\n[第{round_}轮] 模型发起 {len(ai.tool_calls)} 个工具调用，"
              f"但已达到最大轮数 {MAX_ROUNDS}，停止调用。")
    return ai.content
```

#### 细节一：`MAX_ROUNDS = 3`——给循环装刹车

任务表的裸 `while ai.tool_calls:` 是个**理想化写法**。真实情况：模型可能陷入"调工具 → 结果不满意 → 再调 → 再不满意"的循环，每一轮都在烧 token。

```
  裸 while ai.tool_calls:            带 MAX_ROUNDS 的循环
  ─────────────────────             ────────────────────
  ✅ 逻辑正确                         ✅ 逻辑正确
  ❌ 可能无限循环烧钱                  ✅ 最多 3 轮，强制退出
  ❌ 生产环境不敢上线                  ✅ 有兜底返回值
```

**这是把"研究代码"变成"生产代码"的分界线。** 任何交给真实用户的自动循环，都要有轮次上限或超时——**AI 系统的成本和失控风险，都藏在无界循环里**。

> 💡 循环退出后如果 `ai.tool_calls` 还有货（说明被截断），代码特意打印了提示。这个"知道自己被截断了"的意识很重要——**静默失败是最难 debug 的失败**。

#### 细节二：`_normalize_args`——给 DeepSeek 的参数擦屁股

这是**只有真跑过 DeepSeek 才会写出来的防御代码**。

问题现象：模型返回的 `call["args"]` 有时不是干净的 `{"query": "AI新闻"}`，而是**嵌套了一层字典**，类似：

```fallback
{"query": {"description": "搜索关键词", "type": "string"}}
```

——模型把 schema 的描述信息当成值返回了。这时直接 `.invoke(args)`，`client.search(query={...})` 必然炸。

解法：**反复剥洋葱，直到剥出最里层的标量**。

```python
def _normalize_args(args: dict) -> dict:
    for k, v in list(args.items()):
        while isinstance(v, dict) and v:      # 只要还是非空 dict，就继续剥
            v = next(iter(v.values()))        # 取第一个 value 接着剥
        args[k] = v
    return args
```

```
  输入: {"query": {"description": "搜索关键词", "type": "string"}}
         │
         │  isinstance(v, dict) and v  → True
         │  v = next(iter(v.values())) = "搜索关键词"
         ▼
  输出: {"query": "搜索关键词"}          ← 剥到非 dict 就停
```

**为什么用 `while` 而不是 `if`？** 因为嵌套可能不止一层。`while` 保证剥到最里层为止，同时 `and v` 防止空 dict 死循环。

> ⚠️ 这个函数的定位是**兜底，不是主力**。别指望它解决所有参数问题——它只处理"值被套了一层字典"这一种畸形。真正稳定的做法是**把 docstring 写清楚 + `temperature=0`**，从源头减少畸形输出。`_normalize_args` 是安全气囊，不是方向盘。

**你在循环里还特意打印了"解析前 / 解析后"的对照**：

```python
print(f"  工具: {call['name']}  参数: {call['args']}  解析后: {_normalize_args(call['args'])}")
```

> ✅ 这个习惯非常好——**参数畸形是个概率问题，不打日志你根本不知道它什么时候发生**。这也是你 Vibe Coding 手册里"操作透明、禁止黑盒"的又一次落地。

### 3.6 跑起来看效果

```python
if __name__ == "__main__":
    print("\n=== 测试1：天气 ===")
    print(run("北京天气怎么样?"))
    print("\n=== 测试2：联网搜索 ===")
    print(run("今天的最新AI新闻是什么?"))
    print("\n========== 测试3：长期记忆 ==========")
    print(run("用户喜欢什么模型，预算多少?"))
```

**这三个测试是精心设计的，各自验证一件事**：

| 测试 | 验证什么 |
|---|---|
| 天气 | 最简工具的调用链路通不通 |
| 联网搜索 | 有状态工具（闭包注入）能不能跑 |
| 长期记忆 | **D4 转正的 retriever → `create_retriever_tool` → 进循环** 全链路 |

第三个测试跑通那一刻，你 W3 写的记忆系统、D4 转的 retriever、D5 学的工具调用，**三段工作连成了一条线**。

* * *

## 四、踩坑记

### 坑 1：DeepSeek 返回参数嵌套 dict ⚠️

- **现象**：`client.search()` 报错，参数里混进了 `{"description": ..., "type": ...}`
- **原因**：模型偶发把 schema 描述当值返回
- **解法**：`_normalize_args` 循环剥字典 + docstring 写清 + `temperature=0`
- **定位技巧**：循环里同时打印"解析前/解析后"，一眼看出畸形长什么样

### 坑 2：`create_retriever_tool` 导入路径变了 ⚠��

你代码里已经处理了，但值得单独讲：

```python
try:
    from langchain.tools.retriever import create_retriever_tool
except ImportError:
    from langchain_core.tools.retriever import create_retriever_tool
```

- **原因**：这个函数在版本迁移中从 `langchain` 挪到了 `langchain_core`
- **解法**：双路径 fallback
- 💡 **这个 try/except 模式可以套用到所有"路径不确定"的 LangChain 导入上**——和你在第八节记忆篇用的 `_MiniMM` 降级是同一套路：**能用就用，用不了就退，绝不崩**

### 坑 3：`sys.path` 硬编码 + `.env` 覆盖加载 ⚠️

你两份文件顶部都有这两句：

```python
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))
...
project_root = Path(__file__).parent.parent
load_dotenv(project_root / ".env", override=True)
```

- **为什么需要**：直接 `python lc_tools_multi.py` 时，脚本被当顶层模块，`from memory.memory_manager import ...` 找不到包路径
- **两个解法**：① `sys.path.insert(0, 项目根)` + 绝对导入（你用的这招）；② `python -m W4.lc_tools_multi` 启动
- **`override=True` 的意义**：强制用项目 `.env` 覆盖可能已存在的系统环境变量，避免本地装的别的 key 抢戏
- ⚠️ `sys.path.insert` 是**学习期的权宜之计**——正式项目该用 `pip install -e .` 或规范包结构

### 坑 4：不带 `MAX_ROUNDS` 的循环会烧钱 ⚠️

- **现象**：模型反复调工具不收敛，token 消耗失控
- **解法**：轮次上限 + 超时兜底 + 截断时明确提示
- 🔴 **红线**：任何上生产的自动循环，必须有界

### 坑 5：docstring 写太短，模型乱调工具 ⚠️

- **现象**：问"你好"也触发搜索，或该调的时候不调
- **原因**：说明书里只有"干什么"，没有"什么时候干"
- **解法**：docstring 固定写成 **「干什么 + 什么时候调用 + 参数说明」** 三段式

### 坑 6：`bind_tools` 返回新对象，别以为改了原模型

```python
model = ChatOpenAI(...)           # 这个 model 没有工具
model.bind_tools(tools)           # ❌ 返回值被丢弃，等于没绑
model = ChatOpenAI(...).bind_tools(tools)   # ✅ 接住返回值
```

- **原因**：LCEL 全系不可变风格，`bind_tools` 返回新实例
- **排查**：如果 `ai.tool_calls` 永远是空列表，先检查这一行

* * *

## 五、速查卡片（复习直接看这）

```python
# ===== D4：Retriever 转正 =====
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from pydantic import Field, ConfigDict

class MemoryRetriever(BaseRetriever):
    model_config = ConfigDict(arbitrary_types_allowed=True)   # 放行非 pydantic 对象
    mm: Any = Field(description="带 recall(query, k) 的记忆管理器")
    k: int = Field(default=3, description="召回条数")

    def _get_relevant_documents(self, query: str, **kwargs) -> list[Document]:
        # 🎯 唯一必须实现的方法；**kwargs 兼容版本差异
        return [Document(page_content=f, metadata={"source": "长期记忆"})
                for f in self.mm.recall(query, k=self.k)]

# ===== D5：工具三件套 =====
from langchain_core.tools import tool

@tool
def my_tool(city: str) -> str:
    """干什么。什么时候调用。参数 city: 城市名。"""   # ← docstring = 说明书
    return "..."

def make_stateful_tool(client):          # 有状态依赖：工厂 + 闭包
    @tool
    def inner(query: str) -> str:
        """..."""
        return client.search(query)      # client 从闭包来
    return inner

model = ChatOpenAI(...).bind_tools(tools)          # ⚠️ 记得接住返回值
tools_by_name = {t.name: t for t in tools}         # 名字 → 工具对象的查表

# ===== 循环 =====
ai = model.invoke(messages)
while ai.tool_calls and round_ < MAX_ROUNDS:       # 🎯 必须有界
    for call in ai.tool_calls:
        args = _normalize_args(call["args"])                        # 剥嵌套 dict
        result = tools_by_name[call["name"]].invoke(args)
        messages.append(ToolMessage(content=str(result), tool_call_id=call["id"]))
    ai = model.invoke(messages)                    # 带结果再问
return ai.content

# ===== retriever → tool（转正的红利）=====
try:
    from langchain.tools.retriever import create_retriever_tool
except ImportError:
    from langchain_core.tools.retriever import create_retriever_tool
memory_tool = create_retriever_tool(memory_retriever, name="memory_retriever",
                                    description="从长期记忆检索……当用户问起以前聊过的事时调用")
```

* * *

## 六、一句话总结

**D4 的"剩下部分"是把 `RunnableLambda` 手工包的临时工转正成 `BaseRetriever` 子类——只写 `_get_relevant_documents` 这一个方法，换回"生态识别"这张工牌；D5 是把 W2 的 `fc_loop` 翻译成框架语言——`@tool` 用 docstring 自动生成说明书，`bind_tools` 预先挂载工具，`while ai.tool_calls` 的循环逻辑一个字没变。**

两件事指向同一个结论：**框架化省的是"声明"的活，不是"设计"的活。** schema 不用手写了，但"什么时候调什么工具"仍然是你说了算；retriever 的接口白送了，但"召回什么、怎么召"仍然是你 W3 的 `MemoryManager` 在干。

> 回顾这条线：W1 `RealLLM` → W2 `fc_loop` → W3 `MemoryManager` → D4 `MemoryRetriever` → D5 `@tool` + `bind_tools`。**每一个 LangChain 标准件背后，都有一个你已经手撸过的轮子。** 框架化不是推翻重来，是给旧轮子换上标准轴距。