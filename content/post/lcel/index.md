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
LCEL（LangChain Expression Language）= 用 | 把零件串成流水线。

```
chain = prompt | model | parser   # 一条链
```
类比工厂流水线。prompt 是"配料机"（把问题变成消息格式），model 是"加工机"（LLM 思考），parser 是"包装机"（把 AI 的复杂输出变成纯文本）。| 就是传送带——配料出来自动进加工机，加工完自动进包装机。