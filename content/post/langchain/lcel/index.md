---
description: ""
title: "LangChain 入门 + LCEL"
draft: false
date: "2026-08-23T02:39:38+08:00"
slug: "LCEL"
categories:
 - Agent
 - LangChain
tags:
 - LCEL
image: ""
---

# W4 · LCEL：用管道把零件串成流水线

> 关键词：LCEL / Runnable / 声明式 / 零件库 + 组装流水线
> 前置：W1 llm_client、W2 fc_loop、W3 MemoryManager（全是手撸）
> 收尾预告：W5 LangGraph —— node/edge 的雏形

---

## 翻车前的自白

W1 你写 `llm_client`，每次调完模型都要手动抠 `resp.choices[0].message.content`；
W2 你写 `fc_loop`，自己拼 tools schema、解析 `tool_calls`、再写循环；
W3 你写 `MemoryManager`，短期窗口、长期 Chroma、滚动压缩，全自己管。

回头看，这三周你一直在做同一件事：**手动接线**。

每加一个功能，就多一串「把 A 的输出塞进 B 的输入」的胶水代码。功能一多，胶水比功能还多——这就是手写 Agent 的隐性成本。

W4 要解决的，就是这团胶水。**LCEL 不是新概念，是把你 W1-W3 的手动接线，标准化成一条「传送带」。**

---

## 零、LangChain 是什么？

如果你之前只用手写 `requests` 或 `openai` SDK 直连大模型，第一次听到「LangChain」大概率会懵：这玩意儿和「直接调 API」差在哪？

**一句话：LangChain 是一套给 LLM 应用用的「乐高积木 + 说明书」。**

### 先想清楚：不用框架，你怎么调大模型？

裸调一个模型，你其实要手写一堆「杂活」：

```python
# 没有框架时，一次问答你要操心的事
import openai

# 1. 自己拼消息格式
messages = [
    {"role": "system", "content": "你是一个简洁的助手"},
    {"role": "user",   "content": "什么是 LCEL？"},
]

# 2. 自己发起请求
resp = openai.chat.completions.create(
    model="xxx",
    messages=messages,
)

# 3. 自己从嵌套结构里抠出文本
text = resp.choices[0].message.content

# 4. 想要流式？再写一套 chunk 拼接逻辑
# 5. 想要批量？再写一套并发
# 6. 想接数据库/工具/记忆？每一块都自己接线
```

上面 6 步，每一步都是「胶水」。**功能越复杂，胶水越多**。这就是你 W1-W3 手写 Agent 的真实体感——你不是在写业务，是在写「让零件转起来的 plumbing」。

### LangChain 把这堆杂活标准化了

LangChain 做的事，本质上是**把上面那些重复的 plumbing 抽成可复用的标准零件**，再用统一的接口（`Runnable`）接起来：

```
没有框架：            有了 LangChain：
你写 plumbing  ──→   你只声明「要什么」：
requests请求           chain = prompt | model | parser
手抠 .content          框架自动：拼消息→请求→抠结果→流式/批量
自己写流式/批量
自己接工具/记忆
```

所以它不是「另一个能调模型的库」——`openai` SDK 也能调模型。**它是「调模型之上的那层编排」**：当你需要的不是「问一句话」，而是「带记忆、能查资料、会调工具、能多轮决策」的应用时，框架的价值才显出来。

### LangChain 全家福（你后面会陆续碰到）

| 包名 | 干什么 | 你现在用到没 |
|---|---|---|
| `langchain-core` | 最核心：Runnable 接口、LCEL 管道、`|` | ✅ 本文主角 |
| `langchain-openai` | 把 OpenAI / DeepSeek(兼容) 等模型接进来 | ✅ 本文用到 |
| `langchain-community` | 社区集成：各种向量库、工具、检索器 | ⏳ 后面接 Chroma/工具时 |
| `langchain` | 上层高级封装（chains/agents 老接口） | ⏳ 渐进接触 |
| `langgraph` | 有状态图编排（W5 主角） | ⏳ 下周 |

**记住一句话就够了**：LangChain = 「调模型之上的编排层」。你 W1-W3 手写的那些零件，它都有现成的；你手写那些胶水，它用 `|` 一条管道替你接好。

> 回到开头那句自白：你前三周手写的所有东西，不是白写——它们让你**亲手踩过每个零件的坑**，所以现在看 LangChain，你是在「认领」自己造过的轮子，而不是从天书开始学。

---

## 一、LCEL 到底是什么？

**LCEL（LangChain Expression Language）= 用 `|` 把零件串成流水线。**

它是 LLM 应用的「零件库 + 组装流水线」：零件（模型、提示词、解析器、记忆、检索、工具）是标准化的，组装方式（管道 `|`）也是标准化的。

一句话记忆：**`|` 就是传送带，前一个的输出自动变成后一个的输入。**

### 为什么需要它（对照你的自研）

| 你自研的（W1-W3） | LangChain 里的名字 | 一句话 |
|---|---|---|
| `RealLLM` / `llm_client` | `ChatOpenAI` / `ChatModel` | 调模型的封装 |
| `prompt_lib`（W1） | `ChatPromptTemplate` | 提示词模板 |
| 手动取 `.content` / `json.loads` | `StrOutputParser` / `JsonOutputParser` | 解析 AI 输出 |
| `fc_loop` 工具循环（W2） | `Agent` / Tool Calling | 工具调用循环 |
| `MemoryManager`（W3） | `Memory`（或包一层自研） | 记忆 |
| Chroma 长期记忆（W3） | `Retriever` / `VectorStore` | 向量召回 |
| `tools/` 注册表（天气/计算器） | `@tool` 装饰器 | 工具标准接口 |

**关键认知**：框架化不是「学新东西」，是「把你的手动接线变成标准管道」。右边每一格，你左边都写过一遍——所以你不是在从零学，是在给旧东西换统一接口。

> LangChain / LangGraph 是 Agent 岗位出现率最高的框架。会这套「通用语言」，比会某个手写实现更有市场价值。

---

## 二、八个零件：从你的自研到 LangChain

LangChain 把 LLM 应用拆成 8 个标准化零件。下面每个都对应你 W1-W3 的代码——**你早就造过它们，只是名字不同**。

```
┌──────────────────────────────────────────────────────────┐
│  1. Model（模型）= 厨师                                      │
│     你已有 RealLLM。ChatOpenAI 是"统一厨师证"——              │
│     DeepSeek/通义/Kimi 都一个接口，走 OpenAI 兼容协议，       │
│     换 base_url 就行（你早就这么干了）                       │
├──────────────────────────────────────────────────────────┤
│  2. Prompt（提示词模板）= 菜谱模板                            │
│     你 W1 的 prompt_lib。ChatPromptTemplate 支持 {变量} 插值： │
│     "请翻译成英文：{text}"，传 {"text":"你好"} 自动填          │
├──────────────────────────────────────────────────────────┤
│  3. OutputParser（输出解析器）= 装盘                          │
│     你平时手动 json.loads(...)。StrOutputParser 把复杂输出   │
│     变纯字符串；JsonOutputParser 帮你做 json.loads           │
├──────────────────────────────────────────────────────────┤
│  4. Chain / LCEL（链）= 传送带 ⭐ 本周核心                    │
│     前者的输出自动变后者的输入。"声明式"：你描述"要什么流程"， │
│     不用写"怎么一步步调"                                    │
├──────────────────────────────────────────────────────────┤
│  5. Memory（记忆）= 记忆                                      │
│     你 W3 的 MemoryManager 已经很强（短期窗口+长期Chroma+     │
│     滚动压缩）。本周 D3 把它接进链，而不是重写                │
├──────────────────────────────────────────────────────────┤
│  6. Retriever（检索器）= 档案室管理员                         │
│     你 W3 的 Chroma 就是这个。RAG 的"知识记忆"和你的          │
│     "长期向量记忆"是同一个东西——接一次复用，不另起炉灶        │
├──────────────────────────────────────────────────────────┤
│  7. Tool（工具）= 工具箱                                      │
│     你 W2 的天气、计算器。LangChain 用 @tool 装饰器把函数     │
│     变成"模型可调用的工具"                                   │
├──────────────────────────────────────────────────────────┤
│  8. Agent（智能体）= 服务员                                    │
│     你 W2 的 fc_loop：模型决定调哪个工具→执行→结果喂回→再决定。│
│     LangChain 的 Agent 把这个循环标准化了                     │
└──────────────────────────────────────────────────────────┘
```

**三个核心概念先记住**（它们是 W5 LangGraph 的 node/edge 雏形）：
- **Chain = 组合**：把零件用 `|` 串起来
- **Retriever = 取数**：从外部/向量库取上下文
- **Tool = 可调函数**：模型能主动调用的函数

---

## 三、工厂流水线：prompt | model | parser

LCEL 最小形态就是一条三件套链。`|` 是传送带——配料出来自动进加工机，加工完自动进包装机。

```
PromptInput            ← 入口：你传的 {"question": "..."}（dict）
     ↓
ChatPromptTemplate     ← 配料机：dict → [system, user] 消息列表
     ↓
ChatOpenAI             ← 加工机：消息 → AIMessage
     ↓
StrOutputParser        ← 包装机：AIMessage → 纯字符串
     ↓
StrOutputParserOutput  ← 出口：str
```

**声明式 vs 命令式**（这是 LCEL 的精髓，面试爱问）：
- 命令式（你 W1-W3 的写法）：`resp = client.chat(...); text = resp.choices[0].message.content`——你写「怎么一步步调」
- 声明式（LCEL）：`chain = prompt | model | parser`——你只描述「要什么流程」，中间的传递、类型转换全交给框架

这就是为什么叫「表达式语言」：你写的是一个**表达式**（一条链），不是一串**语句**。

---

## 四、最小可跑代码（逐块讲解）

先装依赖（一行命令）：

```bash
pip install langchain langchain-openai langchain-community python-dotenv
pip install -U langchain langchain-openai langchain-core
```

### ① 配料机：ChatPromptTemplate

```python
from langchain_core.prompts import ChatPromptTemplate

# {question} 是占位符，invoke 时传 dict 自动填
prompt = ChatPromptTemplate([
    ("system", "你是一个简洁的助手"),
    ("user", "{question}"),
])
```

对应你 W1 的 `prompt_lib`：把「系统提示 + 用户问题」拼成消息列表，但不用你手写 f-string 拼接了。

### ② 加工机：ChatOpenAI（接 DeepSeek）

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI

load_dotenv()

# DeepSeek 走 OpenAI 兼容协议：换 model 名 + base_url 即可
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)
```

**对应你 W1 的 `RealLLM`**：你当时就是靠 `base_url` 切到 DeepSeek 的。ChatOpenAI 是「统一厨师证」——DeepSeek、通义、Kimi 都是同一套接口，换 `model` + `base_url` 就行，不用为每家重写客户端。

### ③ 包装机：StrOutputParser

```python
from langchain_core.output_parsers import StrOutputParser

# AIMessage → 纯字符串，对应你平时手写的 .content 提取
parser = StrOutputParser()
```

对应你 W1 手动抠的 `resp.choices[0].message.content`——现在一句 `StrOutputParser()` 替代，而且流式/批量场景自动适配。

### ④ 传送带：一条链

```python
# 核心就这一行：前一个的输出自动喂给后一个
chain = prompt | model | parser
```

这就是本周核心。`|` 把三个零件声明式地组装成 Runnable。

### ⑤ 三种调用方式

```python
# invoke：一次性拿到完整结果
result = chain.invoke({"question": "什么是 LCEL？"})
print(isinstance(result, str))   # True ← 这才是验收

# stream：流式，一个字一个字往外冒（对应你 W3 博客里写的 streaming）
for chunk in chain.stream({"question": "数到5"}):
    print(chunk, end="", flush=True)

# batch：一次处理多个输入，框架自动并行
results = chain.batch([
    {"question": "1+1=?"},
    {"question": "2+2=?"},
])
print(results)

# get_graph：把链结构打印成 ASCII 图，肉眼确认流向
chain.get_graph().print_ascii()   # 应看到 prompt → model → parser
```

`invoke` / `stream` / `batch` 是 LCEL 所有 Runnable 的统一三件套——你写的任何链都自动获得这三种能力，不用自己实现。

---

## 五、完整代码（可直接抄）

```python
# 安装依赖
# pip install langchain langchain-openai langchain-community python-dotenv
# pip install -U langchain langchain-openai langchain-core

# 最小 LCEL 链：管道 prompt | model | parser

import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

# 配料机：{question} → [system, user] 消息
prompt = ChatPromptTemplate([
    ("system", "你是一个简洁的助手"),
    ("user", "{question}"),
])

# 加工机：ChatOpenAI 接 DeepSeek（OpenAI 兼容协议，换 base_url 即可）
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)

# 包装机：AIMessage → 纯字符串
parser = StrOutputParser()

# 传送带：一条链
chain = prompt | model | parser

# 调用
print("=== invoke ===")
result = chain.invoke({"question": "什么是 LCEL？"})
# StrOutputParser 返回 Python str
print(isinstance(result, str))   # True ← 这才是验收

print("\n=== stream（流式，一个字一个字冒）===")
for chunk in chain.stream({"question": "数到5"}):
    print(chunk, end="", flush=True)

print("\n\n=== batch（一次处理多个）===")
results = chain.batch([
    {"question": "1+1=?"},
    {"question": "2+2=?"},
])
print(results)

print("\n=== 链结构图 ===")
chain.get_graph().print_ascii()   # 应看到 prompt → model → parser 的流向
```

---

## 六、踩坑记 & 知识点

**① 验收用 `isinstance(result, str)`，别 assert 具体类名**
`StrOutputParser` 在不同 LangChain 版本里，底层返回类型可能裹一层包装类型（比如某些版本会看到 `TextAccessor` 这类名字）。**跨版本会变的是具体类名，不会变的是「它是 str 的子类」**。所以稳健验收写 `isinstance(result, str) == True`，不要写 `type(result).__name__ == 'str'`，后者会在版本升级时假红。

**② DeepSeek 是 OpenAI 兼容协议，不是特例**
你 W1 的 `RealLLM` 已经验证了 `base_url="https://api.deepseek.com"` 这套走法。ChatOpenAI 只是把这套走法做成标准接口——`model` 名按 DeepSeek 官方给的填（笔记里写的 `deepseek-v4-flash` 按你当时环境填，真实环境以官方模型名为准），`api_key` 走环境变量，别硬编码进代码（呼应你 Vibe Coding 手册里的「密钥不上库」红线）。

**③ 「声明式」省的是胶水，不是脑子**
`chain = prompt | model | parser` 看起来比手写 `resp = ...; text = ...` 短，但**它没替你设计流程**——你 still 得想清楚「先配料、再加工、再装盘」的顺序。LCEL 把「传递」自动化了，把「架构决策」留给你。这也是为什么 W3 的 `MemoryManager` 不用重写、直接包一层接进链——决策逻辑是你的，管道只是载体。

**④ `get_graph().print_ascii()` 是你的排错眼**
链一复杂（尤其后面接 Retriever / Tool / Memory），`print_ascii()` 能一眼看出「数据从哪流到哪、有没有断」。每次改完链先打一张图，比对着报错猜快十倍。

---

## 七、总结

LangChain 是 LLM 应用的**编排框架**：模型、提示词、解析器、记忆、检索、工具都标准化成 Runnable，用 LCEL 管道（`|`）声明式组合。

它和你 W1-W3 的自研**一一对应**——`RealLLM`=ChatModel、`prompt_lib`=ChatPromptTemplate、手动解析=OutputParser、`fc_loop`=Agent、`MemoryManager`=Memory、Chroma=Retriever。**所以框架化不是学新东西，是把你的手动接线变成标准管道。**

三个核心概念记住：**Chain=组合、Retriever=取数、Tool=可调函数**——这正是 W5 LangGraph 的 node/edge 雏形。

---
