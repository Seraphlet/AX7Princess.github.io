---
description: ""
title: "LangChain"
draft: false
date: "2026-08-25T07:56:58+08:00"
slug: "LangChain-Note"
categories:
 - LangChain
tags:
 - Note
image: ""
---

# LangChain 

## 目录

- [一、LangChain 是什么（架构总览）](https://blog.noahanna.online/p/langchain-note/#一langchain-是什么架构总览)
- [二、安装与环境](https://blog.noahanna.online/p/langchain-note/#二安装与环境)
- [三、Model 模型：ChatModel](https://blog.noahanna.online/p/langchain-note/#三model-模型chatmodel)
- [四、Prompt 提示词模板：ChatPromptTemplate](https://blog.noahanna.online/p/langchain-note/#四prompt-提示词模板chatprompttemplate)
- [五、OutputParser 输出解析器](https://blog.noahanna.online/p/langchain-note/#五outputparser-输出解析器)
- [六、LCEL 链：Runnable 与管道](https://blog.noahanna.online/p/langchain-note/#六lcel-链runnable-与管道)
- [七、Runnable 组合件](https://blog.noahanna.online/p/langchain-note/#七runnable-组合件)
- [八、Memory 记忆](https://blog.noahanna.online/p/langchain-note/#八memory-记忆)
- [九、Retriever 检索与 RAG](https://blog.noahanna.online/p/langchain-note/#九retriever-检索与-rag)
- [十、Tool 工具与 @tool](https://blog.noahanna.online/p/langchain-note/#十tool-工具与-tool)
- [十一、Agent 智能体](https://blog.noahanna.online/p/langchain-note/#十一agent-智能体)
- [十二、Callbacks 回调](https://blog.noahanna.online/p/langchain-note/#十二callbacks-回调)
- [十三、LangGraph 状态图（进阶）](https://blog.noahanna.online/p/langchain-note/#十三langgraph-状态图进阶)
- [十四、常用技巧与踩坑](https://blog.noahanna.online/p/langchain-note/#十四常用技巧与踩坑)
- [十五、小结](https://blog.noahanna.online/p/langchain-note/#十五小结)

* * *

## 一、LangChain 是什么（架构总览）

### 一句话定位

**LangChain = 调模型之上的"编排层"。** 它不替你调模型（那还是 OpenAI/DeepSeek 的活），它解决的是"调完模型之后那一堆 plumbing"：拼提示词、解析输出、接记忆、接检索、接工具、把流程串成管道。

> 你之前 W1-W3 手写过的 `RealLLM` / `prompt_lib` / `fc_loop` / `MemoryManager` / `Chroma`，在 LangChain 里都有标准件对应。学 LangChain 不是学新东西，是把你的"手动接线"换成"标准管道"。

### 全景架构图

```
┌──────────────────────────────────────────────────────────────┐
│                      你的应用 / Agent                          │
├──────────────────────────────────────────────────────────────┤
│  LangChain 编排层（本文重点）                                  │
│                                                                │
│   Prompt ──▶ Model ──▶ OutputParser        （基础链 LCEL）     │
│      │         │            │                                    │
│      ▼         ▼            ▼                                    │
│   Memory   Retriever      Tool ◀── Agent（决定调谁）            │
│   (记忆)    (RAG检索)     (工具)                                 │
│                                                                │
│   RunnableParallel / Passthrough / Lambda / Branch（组合件）    │
│   LangGraph（状态图：把上面这些编排成有环的流程）               │
├──────────────────────────────────────────────────────────────┤
│  langchain-core / langchain-openai / community（接入层）        │
├──────────────────────────────────────────────────────────────┤
│  模型供应商 API（DeepSeek / OpenAI / 通义 / Kimi ...）          │
└──────────────────────────────────────────────────────────────┘
```

### 包结构 —— 别一股脑 `pip install langchain` 就完事

```bash
pip install langchain-core          # 核心抽象：Runnable / Prompt / Parser / 基础接口（必装）
pip install langchain-openai        # OpenAI 及兼容协议（DeepSeek 走这）的模型接入
pip install langchain-community     # 社区集成：各种向量库、文档加载器、第三方工具
pip install langchain               # 高层封装：agent / chain / 一些 conveniences（可选）
pip install langgraph              # 状态图编排（W5 用，独立包）
pip install python-dotenv          # 读 .env 里的密钥
```

| 包 | 干啥 | 你现在用得到吗 |
|---|---|---|
| `langchain-core` | Runnable 协议、Prompt、Parser、Message 类型 | ✅ 必用 |
| `langchain-openai` | `ChatOpenAI` 模型封装 | ✅ 必用 |
| `langchain-community` | Chroma / FAISS / 各类 loader | ✅ 做 RAG 用 |
| `langchain` | 旧的 `LLMChain`、`Agent` 封装 | ⚠️ 历史包袱多，新项目优先用 core + LCEL |
| `langgraph` | 状态图 / 多轮循环编排 | ✅ W5 进阶 |

> ⚠️ **版本兼容是头号坑**：LangChain 迭代极快，旧博客里的 `LLMChain(...)`、`ConversationChain` 在新版已弃用。本文全部基于 **LCEL（core 0.2+）** 写法，这是目前最稳、官方主推的范式。

### 裸调 vs LangChain 对照（先理解痛点）

```python
# ===== 裸调（你 W1 的写法）=====
import openai
resp = openai.ChatCompletion.create(
    model="deepseek-chat",
    messages=[{"role": "user", "content": "你好"}]
)
text = resp["choices"][0]["message"]["content"]   # 每次都要抠这一长串
```

```python
# ===== LangChain（声明式）=====
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

chain = ChatPromptTemplate.from_messages([("user", "{q}")]) | ChatOpenAI(...) | StrOutputParser()
text = chain.invoke({"q": "你好"})   # 一行拿到字符串，不用抠 dict
```

差别：**裸调你写 plumbing，LangChain 你只声明"要什么流程"**。

* * *

## 二、安装与环境

### `pip install` —— 装依赖

```bash
pip install -U langchain langchain-openai langchain-core langchain-community python-dotenv
```

`-U` 升级到最新版；建议锁版本做笔记：`pip freeze > requirements.txt`。

### `.env` + `load_dotenv` —— 密钥分离

```python
import os
from dotenv import load_dotenv

load_dotenv()                          # 自动读取项目根目录的 .env 文件
api_key = os.getenv("DEEPSEEK_API_KEY")  # 从环境变量取，绝不硬编码进代码
```

`.env` 文件内容（放项目根目录）：

```fallback
DEEPSEEK_API_KEY=sk-xxxxxxx
```

> ⚠️ **密钥红线**：`.env` 必须写进 `.gitignore`，绝对不能 commit 进仓库。一旦泄露，第一时间去平台注销密钥并换新的（你一直强调的安全意识，这里再次强调）。

* * *

## 三、Model 模型：ChatModel

### `ChatOpenAI` —— 统一接口调任意兼容模型

```python
from langchain_openai import ChatOpenAI
import os

# DeepSeek 走 OpenAI 兼容协议：换 base_url 就行
model = ChatOpenAI(
    model="deepseek-chat",                       # 模型名（DeepSeek 用 deepseek-chat / deepseek-reasoner）
    api_key=os.getenv("DEEPSEEK_API_KEY"),       # 从环境变量取
    base_url="https://api.deepseek.com",         # 兼容协议入口
    temperature=0.7,                             # 随机性：0= deterministic，1= 发散
    max_tokens=1024,                             # 单次最多生成多少 token
)
```

| 参数 | 作用 | 常用值 |
|---|---|---|
| `model` | 模型名 | `deepseek-chat` / `gpt-4o` / `qwen-max` |
| `temperature` | 采样温度 | 0（严谨）/ 0.7（创意）/ 1（发散） |
| `max_tokens` | 上限 | 按任务定，默认往往偏小 |
| `streaming` | 是否流式 | `True` 时支持 `.stream()` |
| `base_url` | 兼容入口 | DeepSeek=`https://api.deepseek.com` |

> 你 W1 的 `RealLLM` 其实就是这个 `ChatOpenAI` 的封装。所谓"统一厨师证"——不管底层是 DeepSeek 还是通义，接口都长一样，换模型只改两个字段。

### `invoke` / `stream` / `batch` —— 三种调用方式

```python
# 1) invoke：一次拿完整结果（最常用）
msg = model.invoke("你好")          # 返回 AIMessage 对象
print(msg.content)                  # 取文本用 .content

# 2) stream：流式，一个字一个字冒（适合聊天界面）
for chunk in model.stream("写一首诗"):
    print(chunk.content, end="", flush=True)

# 3) batch：一次塞多个输入，并发处理（比循环 invoke 快）
results = model.batch(["1+1=?", "2+2=?"])
print([r.content for r in results])
```

> 这三个方法来自 **Runnable 协议**，后面所有组件（Prompt/Parser/Chain）都支持，这是 LCEL 能串起来的底层原因。

* * *

## 四、Prompt 提示词模板：ChatPromptTemplate

### `ChatPromptTemplate.from_messages` —— 构造消息模板

```python
from langchain_core.prompts import ChatPromptTemplate

# 用 (角色, 文本) 元组列表；{q} 是占位变量
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个乐于助人的助手。"),   # system 角色
    ("user", "{q}"),                          # user 角色，{q} 待填
])

# 调用时传字典填充变量
messages = prompt.invoke({"q": "什么是 LCEL？"})
print(messages.to_messages())   # 得到 [SystemMessage, HumanMessage]
```

你 W1 的 `prompt_lib` 干的就是这事——`ChatPromptTemplate` 是它的框架标准版，支持 `{变量}` 插值。

### `MessagesPlaceholder` —— 历史消息占位

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是有记忆的助手。"),
    MessagesPlaceholder("history"),     # 这里会展开成一串历史消息对象
    ("user", "{question}"),
])

# 调用时 history 必须传"消息对象列表"（不是 dict）
from langchain_core.messages import HumanMessage, AIMessage
chain.invoke({
    "question": "我叫什么？",
    "history": [HumanMessage(content="我叫小明"), AIMessage(content="你好小明")]
})
```

> ⚠️ **`placeholder` 要配消息对象列表**，不是 dict 列表。你自研记忆返回的是 dict，必须先用 `_to_lc` 转换（见第八节）。

### `partial` —— 预填部分变量

```python
# 有些变量每次都一样（如固定语言），可以提前 partial 掉
prompt = ChatPromptTemplate.from_messages([("system", "用{lang}回答：{q}")])
prompt_zh = prompt.partial(lang="中文")   # 预填 lang，之后只需传 {q}
print(prompt_zh.invoke({"q": "你好"}).to_messages())
```

* * *

## 五、OutputParser 输出解析器

### `StrOutputParser` —— 把 AIMessage 变成纯字符串

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()
text = parser.invoke(model.invoke("你好"))   # AIMessage → str
print(type(text))   # <class 'str'>
```

### `JsonOutputParser` —— 解析 JSON 输出

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import ChatPromptTemplate

parser = JsonOutputParser()
prompt = ChatPromptTemplate.from_messages([
    ("system", "只输出 JSON，不要解释。{format_instructions}"),
    ("user", "列出三种水果和颜色"),
])

# JsonOutputParser 会给你一段"格式说明"塞进 prompt
chain = prompt | model | parser
data = chain.invoke({"format_instructions": parser.get_format_instructions()})
print(data)   # 已经是 dict，不用自己 json.loads
```

### `PydanticOutputParser` —— 强类型结构化（进阶）

```python
from pydantic import BaseModel, Field
from langchain_core.output_parsers import PydanticOutputParser

class Person(BaseModel):
    name: str = Field(description="姓名")
    age: int = Field(description="年龄")

parser = PydanticOutputParser(pydantic_object=Person)
# parser.get_format_instructions() 会生成"必须输出符合 Person schema 的 JSON"
```

> 你平时手动 `json.loads(...)`，就是 `StrOutputParser`/`JsonOutputParser` 的活。框架的价值：连"格式说明"都帮你自动生成。

> ⚠️ **验收坑**：`StrOutputParser` 在某些版本内部可能把结果裹一层包装类型（如旧版的 `TextAccessor`）。**稳健验收写 `isinstance(result, str)`，别断言具体类名**——类名会变，str 子类关系不会变。

* * *

## 六、LCEL 链：Runnable 与管道

### `chain = prompt | model | parser` —— 一条最小链

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
import os

prompt = ChatPromptTemplate.from_messages([("system", "你是一个简洁助手"), ("user", "{question}")])
model = ChatOpenAI(model="deepseek-chat", api_key=os.getenv("DEEPSEEK_API_KEY"),
                   base_url="https://api.deepseek.com")
parser = StrOutputParser()

# | 就是传送带：前一个输出自动变成后一个输入
chain = prompt | model | parser

# 调用
print(chain.invoke({"question": "什么是 LCEL？"}))   # 直接拿到 str
```

**工厂流水线类比**：
- `prompt` = 配料机（dict → 消息列表）
- `model` = 加工机（消息 → AIMessage）
- `parser` = 包装机（AIMessage → 纯字符串）
- `|` = 传送带（自动流转）

### `get_graph().print_ascii()` —— 可视化整条链

```python
chain.get_graph().print_ascii()
# 输出：
#   PromptInput
#      |
#   ChatPromptTemplate
#      |
#   ChatOpenAI
#      |
#   StrOutputParser
#      |
#   StrOutputParserOutput
```

> 排错神器：链不工作时先打印结构图，看数据流向对不对。

### Runnable 协议 —— 为什么能 `|`

所有 LangChain 组件都实现 **Runnable 接口**，至少有 `invoke` / `stream` / `batch` 三个方法。`|` 运算符本质是"把左边输出喂给右边输入"。普通 Python 函数没有这接口，必须先包成 `RunnableLambda`（见第七节）才能入链。

* * *

## 七、Runnable 组合件

### `RunnableParallel` —— 一条输入，并行出多个结果

```python
from langchain_core.runnables import RunnableParallel

# 同一条 question 同时喂给"回答链"和"关键词链"
parallel = RunnableParallel(
    answer=answer_chain,        # 键名 answer 会成为结果 dict 的键
    keywords=kw_chain,
)
res = parallel.invoke({"question": "什么是 LCEL？"})
# res = {"answer": "...", "keywords": "..."}  返回 dict
```

> 你 W1-W3 没有"并行"概念，这是框架新增能力：一条输入复制多份，同时跑不同子链。

### `RunnablePassthrough.assign` —— 保留原输入，再追加新字段

```python
from langchain_core.runnables import RunnablePassthrough

# 原输入 {"question": "..."} 被保留，再追加 长度 / 前5字 两个字段
enrich = RunnablePassthrough.assign(
    长度=lambda x: len(x["question"]),
    前5字=lambda x: x["question"][:5],
)
out = enrich.invoke({"question": "什么是LCEL？"})
# out = {"question": "什么是LCEL？", "长度": 7, "前5字": "什么是LC"}
```

> 这等价于字典解包 `{**x, "新字段": ...}`——你 W3 `get_context` 里"保留原消息 + 插新消息"的思路，框架用 `.assign` 标准化了。

### `RunnableLambda` —— 任意 Python 函数入链（适配器）

```python
from langchain_core.runnables import RunnableLambda

def 清洗(text: str) -> str:
    return " ".join(text.split())[:50]     # 普通函数，参数类型要对接上一个 Runnable 的输出

chain = answer_chain | RunnableLambda(清洗)   # 把函数包成 Runnable，才能用 |
print(chain.invoke({"question": "介绍 LangChain"}))
```

> 类比"插头转换器"：普通函数是两脚插头，LangChain 管道是三脚插座，套个 `RunnableLambda` 就通了。**这也是你把 W3 自研 `MemoryManager` / `compress()` 接进链的桥**。

### `RunnableBranch` —— 条件分支（按判断走不同子链）

```python
from langchain_core.runnables import RunnableBranch

branch = RunnableBranch(
    (lambda x: "天气" in x["question"], weather_chain),   # 条件1 成立 → 天气链
    (lambda x: "计算" in x["question"], calc_chain),      # 条件2 成立 → 计算链
    default_chain,                                        # 都不成立 → 默认链（必须放最后）
)

for q in ["北京天气？", "计算 12*8", "你好"]:
    print(branch.invoke({"question": q}))
```

> 条件**短路**：从上到下匹配，命中第一个就走，所以默认链必须置底。这是 W5 LangGraph 条件 `edge` 的雏形。

* * *

## 八、Memory 记忆

### 三种记忆模式

| 模式 | 存啥 | 对应你 W3 |
|---|---|---|
| 短期（窗口） | 最近 N 轮对话 | `ShortTermMemory` |
| 长期（向量） | 用户事实 / 知识 | `LongTermMemory` + Chroma |
| 压缩 | 老历史 → 一条摘要 | `compress.py` |

### 用 `RunnableLambda` 接入自研 `MemoryManager`（重点）

你的 `MemoryManager.get_context()` 返回 **dict 列表**，但 prompt 的 `MessagesPlaceholder` 要 **消息对象列表**。中间必须有一道"格式转换"——这就是 `_to_lc`：

```python
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from langchain_core.runnables import RunnableLambda

# 桥的包装器：dict 消息 → LangChain BaseMessage
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "assistant":
        return AIMessage(content=content)
    return HumanMessage(content=content)

# 接入 W3 自研 MemoryManager（失败则降级最小窗口版）
try:
    from memory.memory_manager import MemoryManager
    _mm = MemoryManager(window=6, max_tokens=500)

    def get_history(x):
        # 真实版：query=本轮问题 → 长期召回 + 滚动窗口
        return [_to_lc(m) for m in _mm.get_context(query=x.get("question"))]
except Exception as e:
    print(f"[降级]未找到 memory 包({e})，使用最小窗口版")
    class _MiniMM:
        def __init__(self, window=6):
            self.buf, self.window = [], window
        def get_context(self, query=None):
            return self.buf[-self.window:]
        def add_user(self, t):
            self.buf.append({"role": "user", "content": t})
        def add_ai(self, t):
            self.buf.append({"role": "assistant", "content": t})
    _mm = _MiniMM()
    def get_history(x):
        return [_to_lc(m) for m in _mm.get_context()]

# 链首用 RunnableLambda 加料：保留 question，追加 history 字段
load_mem = RunnableLambda(lambda x: {**x, "history": get_history(x)})
chain = load_mem | prompt | model | StrOutputParser()

def ask(question: str) -> str:
    ans = chain.invoke({"question": question})
    if hasattr(_mm, "add_user"):
        _mm.add_user(question); _mm.add_ai(ans)
    else:
        _mm.add({"role": "user", "content": question})
        _mm.add({"role": "assistant", "content": ans})
    return ans

for q in ["我叫小明，喜欢打篮球", "我刚说的爱好是什么？", "我名字叫什么？"]:
    print(f"Q: {q}\nA: {ask(q)}\n")
```

> ⚠️ **相对导入坑**：上面的 `from memory.memory_manager import ...` 直接 `python lc_memory.py` 跑会炸（脚本被当顶层模块，`..` / 包路径解析不了）。修法有二：① `sys.path.insert(0, 项目根目录)` + 绝对导入；② 用 `python -m 包名.模块名` 启动。你 W3 调试时踩过同款坑，这里是标准解法。

> 关键设计：**`MemoryManager` 不重写，只包一层**——延续你 W3 的"调度者不重写逻辑"原则。`get_context` / `compress` 原样复用，框架只是多了一个入口适配器。

* * *

## 九、Retriever 检索与 RAG

### RAG 流程：文档 → 切分 → 向量化 → 存入 → 检索 → 拼进 prompt

```python
from langchain_community.vectorstores import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

# 1) 切分文档
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
docs = splitter.create_documents([open("知识库.txt", encoding="utf-8").read()])

# 2) 向量化 + 存入 Chroma（你 W3 的 Chroma 长期记忆就是它）
embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-small-zh-v1.5")
vectordb = Chroma.from_documents(docs, embeddings, persist_directory="./chroma_db")

# 3) 检索器：query → 相关片段
retriever = vectordb.as_retriever(search_kwargs={"k": 3})
hits = retriever.invoke("退款政策是什么？")
print([d.page_content for d in hits])
```

> 你 W3 的"长期向量记忆"和 RAG 的"知识检索"是**同一个东西**——Chroma 接一次复用，不用另起炉灶。

### 接进链（检索增强问答）

```python
from langchain_core.runnables import RunnablePassthrough

# 用 .assign 把检索结果拼进上下文，原 question 保留
rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | rag_prompt | model | parser
)
```

* * *

## 十、Tool 工具与 @tool

### `@tool` —— 把函数变成模型可调用的工具

```python
from langchain_core.tools import tool

@tool
def calculator(expr: str) -> str:
    """计算一个数学表达式。"""
    return str(eval(expr))

@tool
def get_weather(city: str) -> str:
    """查询某城市天气。"""
    return f"{city} 今天晴，25℃"

# 模型能"看到"工具的描述和参数，自己决定调哪个
tools = [calculator, get_weather]
model_with_tools = model.bind_tools(tools)
```

> 你 W2 的 `fc_loop` 手写了"拼 tool schema + 解析 tool_calls + 循环"。`@tool` 装饰器把函数自动变成标准接口，省掉手写 schema。

### `bind_tools` + 工具调用循环

```python
from langchain_core.messages import HumanMessage, ToolMessage

msg = model_with_tools.invoke("北京天气怎么样？计算 12*8")
# msg.tool_calls 里有模型想调的工具和参数
for call in msg.tool_calls:
    fn = {"calculator": calculator, "get_weather": get_weather}[call["name"]]
    result = fn.invoke(call["args"])          # 执行工具
    # 把结果回灌给模型，让它继续决定下一步（这就是 Agent 循环）
    follow = model_with_tools.invoke([
        HumanMessage("北京天气怎么样？计算 12*8"),
        msg,
        ToolMessage(content=str(result), tool_call_id=call["id"]),
    ])
```

* * *

## 十一、Agent 智能体

### 本质：模型决定调哪个工具 → 执行 → 结果喂回 → 再决定

你 W2 的 `fc_loop` 就是这个循环的标准化版本。LangChain 提供 `create_react_agent` 等高级封装：

```python
from langchain.agents import create_react_agent, AgentExecutor

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个助手，可用工具完成任务。"),
    ("user", "{input}"),
    ("assistant", "{agent_scratchpad}"),   # 思考轨迹占位
])
agent = create_react_agent(model_with_tools, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

print(executor.invoke({"input": "北京天气怎么样？再算下 12*8"})["output"])
```

> `AgentExecutor` 帮你把"调模型 → 解析 tool_calls → 执行 → 回灌"的循环跑完。你自研的 `fc_loop` 逻辑完全等价，只是它更通用。

* * *

## 十二、Callbacks 回调

### 监听链路每一步事件（调试 / 日志 / 计费）

```python
from langchain_core.callbacks import StdOutCallbackHandler

# verbose 等价写法：把回调传进 invoke
chain.invoke({"question": "你好"}, config={"callbacks": [StdOutCallbackHandler()]})
```

> 生产环境常用自定义 callback 统计 token 消耗、记录每次 LLM 调用——对应你 Vibe Coding 手册里"操作透明、禁止黑盒"的纪律。

* * *

## 十三、LangGraph 状态图（进阶）

### `StateGraph` —— 把流程编排成"有状态 + 可有环"的图

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class State(TypedDict):
    question: str
    answer: str
    steps: Annotated[list, operator.add]   # 累加型字段，每步追加

def call_model(state: State):
    state["answer"] = chain.invoke({"question": state["question"]})
    state["steps"].append("call_model")
    return state

def should_continue(state: State):
    return END if "结束" in state["answer"] else "call_model"   # 条件边

graph = StateGraph(State)
graph.add_node("call_model", call_model)
graph.add_edge("call_model", "call_model")          # 自环：可多轮
graph.add_conditional_edges("call_model", should_continue)
app = graph.compile()

print(app.invoke({"question": "你好", "steps": []}))
```

> LangGraph = LCEL 的"有环升级版"。你 W4 学的 `RunnableLambda`（node）、`RunnableBranch`（conditional edge）在这里成了 `add_node` / `add_conditional_edges` 的一等公民。**W5 你会把手写 Agent 的循环搬进状态图**。

* * *

## 十四、常用技巧与踩坑

### 技巧 1：流式 + 解析器组合

```python
for chunk in chain.stream({"question": "数到5"}):
    print(chunk, end="", flush=True)   # StrOutputParser 保证每块是 str
```

### 技巧 2：batch 并发提速

```python
results = chain.batch([{"question": "1+1"}, {"question": "2+2"}])   # 比循环 invoke 快
```

### 技巧 3：降级保护

生产代码用 `try/except` 包住导入，导入失败就退回最小实现（见第八节 `_MiniMM`），保证程序不崩。

### 踩坑 1：相对导入直接跑脚本会炸 ⚠️

`from ..memory import X` 在 `python xxx.py` 下必然失败。改用 `sys.path.insert(0, 根目录)` + 绝对导入，或 `python -m 包.模块`。

### 踩坑 2：版本弃用 ⚠️

旧教程的 `LLMChain`、`ConversationChain`、`initialize_agent` 已弃用。坚持用 **LCEL（core）+ `create_react_agent` + LangGraph**。

### 踩坑 3：parser 输出类型 ⚠️

验收用 `isinstance(result, str)`，别写死类名（跨版本包装层会变）。

### 踩坑 4：流式不能用普通函数截断 ⚠️

`RunnableLambda` 包的函数在 `.stream()` 下要支持流式透传，否则丢字。简单场景用 `StrOutputParser` 最稳。

* * *

## 十五、小结

LangChain 看着庞大，日常 80% 的场景只用这几样：**`ChatPromptTemplate` 拼提示 + `ChatOpenAI` 调模型 + `StrOutputParser` 解析 + `|` 串成链 + `RunnableLambda` 接你的老代码**。先把这条主线跑熟，记忆/检索/工具/Agent 都是往这条主线上挂模块。

回顾你 W1-W3 的手写轮子，对应关系是：

| 你自研的 | LangChain 标准件 |
|---|---|
| `RealLLM` / `llm_client` | `ChatOpenAI` |
| `prompt_lib` | `ChatPromptTemplate` |
| 手动取 `.content` / `json.loads` | `StrOutputParser` / `JsonOutputParser` |
| `fc_loop` 工具循环 | `Agent` / `bind_tools` / `ToolMessage` |
| `MemoryManager` | `Memory` + `RunnableLambda` 接入 |
| Chroma 长期记忆 | `VectorStore` / `Retriever` |
| `tools/` 注册表 | `@tool` 装饰器 |

**框架化不是学新东西，是把手动接线变成标准管道。** 你已有的底层认知（token、记忆、压缩、工具循环）全部复用，只是换了更通用的语法。

下一篇建议深入 **LangGraph**：把 Agent 的多轮循环、条件分支、状态共享，用状态图标准化——那是 W5 的主战场，也是你手写 `fc_loop` / `MemoryManager` 的自然归宿。

**记住一条铁律**：凡是接你自研代码的，先想"它实现 Runnable 了吗？没实现就包一层 `RunnableLambda`"；凡是接 dict 消息的，先想"prompt 要的是消息对象吗？要就过一道 `_to_lc`"。这两道桥通了，LangChain 的大门就彻底打开了。
