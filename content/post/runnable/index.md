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





