---
description: ""
title: ""
draft: true
date: "2026-08-24T00:12:57+08:00"
slug: "Runnable"
categories:
 - 
tags:
 - 
image: ""
---

RunnableParallel:把多个 Runnable 并行跑,输入自动透传、结果合并成 dict。一条输入喂给 N 条链,同时拿 N 个产出。
RunnablePassthrough:原样透传数据,常用于「保留原始输入 + 追加新字段」(RunnablePassthrough.assign(...))。
RunnableLambda:把任意 Python 函数包成 Runnable——这是 D3 把 W3 MemoryManager 接进链的桥,今天先认识它。
RunnableBranch:条件分支,按判断走不同子链(W5 LangGraph 条件 edge 的雏形)。
多步链组装:用 | 把上面这些拼成 pipeline,理解数据在链里如何被变换、分流、合并。
可视化调试:chain.get_graph().print_ascii() 看整条链的结构。

RunnableParallel —— 并行流水线（一条输入，同时出多个结果）
D1 的链是串行：prompt | model | parser，一步接一步。RunnableParallel 让你一条输入同时喂给多条链，最后汇总成一个 dict

RunnablePassthrough —— 复印机（原样保留 + 追加新字段）
RunnablePassthrough 把输入原样传给下一步，不改动。它有个升级版 .assign()：保留原输入，再追加新字段

RunnableLambda —— 临时加工站（把任意函数变成链的一环）
RunnableLambda 把任意 Python 函数包装成 Runnable，塞进链里

RunnableBranch —— 岔路口（按条件走不同分支）
RunnableBranch 按条件把输入分到不同子链


| 新武器 | 你写过的对应物 |
|---|---|
| RunnableParallel | 无（这是新能力：并行） |
| RunnablePassthrough | `get_context` 里"保留原消息+插新消息" [1] |
| RunnableLambda | 你的 `compress()` / `recall()` 自研函数 |
| RunnableBranch | 你的 `auto_select` 路由 [2] |

import os 
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import(
    RunnableParallel,
    RunnablePassthrough,
    RunnableLambda,
    RunnableBranch,
)
load_dotenv()
model=ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)

parser=StrOutputParser()

answer_prompt=ChatPromptTemplate.from_messages([
    ("system","你是一个简洁回答的助手"),
    ("user","{question}")
])

kw_prompt=ChatPromptTemplate.from_messages([
    ("system","你是关键词提取器，请输出3个关键词，用逗号分隔"),
    ("user","{question}")
])
answer_chain = answer_prompt | model | parser
kw_chain = kw_prompt | model | parser

parallel_chain=RunnableParallel(
    answer=answer_chain,
    keywords=kw_chain,
)
print("=== 步骤A1：RunnableParallel 并行 ===")
res = parallel_chain.invoke({"question": "什么是LCEL？"})
print(res)

print("类型:", {k: type(v).__name__ for k, v in res.items()})

print("\n=== 步骤A2：RunnablePassthrough.assign 追加字段 ===")

enrich=RunnablePassthrough.assign(
    长度=lambda x: len(x["question"]),
    前5个字=lambda x: x["question"][:5],
)
out=enrich.invoke({"question":"什么是LCEL？"})
print(out)

# RunnableBranch 分支
weather_prompt=ChatPromptTemplate.from_messages([
    ("system","你是天气助手，用一句话回答天气问题"),
    ("user","{question}"),
])

calc_prompt=ChatPromptTemplate.from_messages([
    ("system","你是计算小助手，只输出计算结果"),
    ("user","{question}"),
])

default_prompt=ChatPromptTemplate.from_messages([
    ("system","你只智能助手,简要回答"),
    ("user","{question}"),
])

weather_chain = weather_prompt | model | parser
calc_chain = calc_prompt | model | parser
default_chain = default_prompt | model | parser

branch=RunnableBranch(
     (lambda x: "天气" in x["question"], weather_chain),   # 条件1 → 天气链
    (lambda x: "计算" in x["question"], calc_chain),      # 条件2 → 计算链
    default_chain,                                   
)

print("\n=== 步骤C：RunnableBranch 分支 ===")
for q in ["北京天气怎么样？", "计算 12*8", "你好呀"]:
    ans = branch.invoke({"question": q})
    print(f"问: {q}\n答: {ans}\n")


# 步骤 D：get_graph().print_ascii() 可视化

print("=== 步骤D：并行链结构图 ===")
parallel_chain.get_graph().print_ascii()

print("\n=== 步骤D：分支链结构图 ===")
branch.get_graph().print_ascii()

# 步骤 E：RunnableLambda 接自研函数

def TextCleansing(text:str)->str:
    """模拟自研函数，无实际逻辑处理"""
    return " ".join(text.split())[:50]

chain_with_func=answer_prompt | model | parser | RunnableLambda(TextCleansing)

print("\n=== 步骤E：RunnableLambda 加工站 ===")

out_e=chain_with_func.invoke({"question": "用一句话介绍 LangChain"})
print(out_e)
print("类型:", type(out_e).__name__)



