---
description: ""
title: "LangChain 入门 + LCEL"
draft: true
date: "2026-08-23T02:39:38+08:00"
slug: "LCEL"
categories:
 - Agent
 - LangChain
tags:
 - LCEL
image: ""
---

## LCEL 到底是什么？
LCEL（LangChain Expression Language）= 用 | 把零件串成流水线。是 LLM 应用的"零件库 + 组装流水线"。
为什么需要它？

W1 你写 llm_client：每次要处理 resp.choices[0].message.content 
agent.py
W2 你写 fc_loop：要自己拼 tools schema、解析 tool_calls、循环
W3 你写 MemoryManager：自己管短期/长期/压缩

每加一个功能就要手动接线。LangChain 把这些"接线"变成标准管道，而且行业里大家都用同一套语言——LangChain/LangGraph 是 Agent 岗位出现率最高的框架 

| 你自研的（W1-W3） | LangChain 里的名字 | 一句话 |
|---|---|---|
| `RealLLM` / `llm_client` [2] | `ChatOpenAI` / `ChatModel` | 调模型的封装 |
| `prompt_lib`（W1） | `ChatPromptTemplate` | 提示词模板 |
| 手动取 `.content` / `json.loads` | `StrOutputParser` / `JsonOutputParser` | 解析 AI 输出 |
| `fc_loop` 工具循环（W2） | `Agent` / Tool Calling | 工具调用循环 |
| `MemoryManager`（W3） | `Memory`（或包一层自研） | 记忆 |
| Chroma 长期记忆（W3） | `Retriever` / `VectorStore` | 向量召回 |
| `tools/` 注册表（天气/计算器） | `@tool` 装饰器 | 工具标准接口 |


. Model（模型）= 厨师
你已有 RealLLM。LangChain 的 ChatOpenAI 是"统一厨师证"——不管 DeepSeek、通义、Kimi，都一个接口。DeepSeek 走 OpenAI 兼容协议，换 base_url 就行（你早就这么干了）。
 Prompt（提示词模板）= 菜谱模板
你 W1 的 prompt_lib。ChatPromptTemplate 支持 {变量} 插值："请翻译成英文：{text}"，传 {"text": "你好"} 自动填。

3. OutputParser（输出解析器）= 装盘
你平时手动 json.loads(...)。StrOutputParser 就是把 AI 的复杂输出变成纯字符串；JsonOutputParser 就是帮你做 json.loads。

4. Chain / LCEL（链）= 传送带 ⭐ 本周核心
前一个的输出自动变成后一个的输入。这就是"声明式"：你描述"要什么流程"，不用写"怎么一步步调"。

Memory（记忆）= 记忆

你 W3 的 MemoryManager 已经很强（短期窗口 + 长期 Chroma + 滚动压缩）。本周 D3 就是把你的 MemoryManager 接进链 
W4周任务表.md
，而不是重写。

6. Retriever（检索器）= 档案室管理员
你 W3 的 Chroma 就是这个。RAG 的"知识记忆"和你的"长期向量记忆"是同一个东西——接一次就复用，不用另起炉灶 
W4周任务表.md
。

7. Tool（工具）= 工具箱
你 W2 的天气、计算器。LangChain 用 @tool 装饰器把函数变成"模型可调用的工具"。

8. Agent（智能体）= 服务员
你 W2 的 fc_loop：模型决定调哪个工具 → 执行 → 结果喂回 → 再决定。LangChain 的 Agent 就是把这个循环标准化了。

LangChain 是 LLM 应用的编排框架：模型、提示词、解析器、记忆、检索、工具都标准化成 Runnable，用 LCEL 管道（|）声明式组合。它和我 W1-W3 自研一一对应——RealLLM=ChatModel、prompt_lib=ChatPromptTemplate、手动解析=OutputParser、fc_loop=Agent、MemoryManager=Memory、Chroma=Retriever。所以框架化不是学新东西，是把我的手动接线变成标准管道。三个核心概念：Chain=组合、Retriever=取数、Tool=可调函数，是 W5 LangGraph 的 node/edge 雏形 
```
chain = prompt | model | parser   # 一条链
```
类比工厂流水线。prompt 是"配料机"（把问题变成消息格式），model 是"加工机"（LLM 思考），parser 是"包装机"（把 AI 的复杂输出变成纯文本）。| 就是传送带——配料出来自动进加工机，加工完自动进包装机。

