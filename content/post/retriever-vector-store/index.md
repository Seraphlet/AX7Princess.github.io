---
description: ""
title: "Retriever / Vector Store"
draft: true
date: "2026-08-28T07:25:20+08:00"
slug: "Retriever-Vector Store"
categories:
 - 
tags:
 - 
image: ""
---

Vector Store（仓库）负责"存"，Retriever（取货员）负责"取"，把两者接进 LCEL 链，就做出最朴素的 RAG（检索增强生成）

原理：五个概念，每个都对应你已经会的东西

Chroma 里存的不是裸字符串，是 Document(page_content, metadata)：

Document(page_content="公司年假政策:入职满1年享5天",
         metadata={"部门": "HR", "更新时间": "2026"})

Vector Store 的三个动作（这就是"仓库"的完整用法）

| 动作 | 方法 | 类比 |
|---|---|---|
| 存 | `store.add_documents([...])` | 进货入库 |
| 查 | `store.similarity_search(query, k=2)` | 按相关度捞货 |
| 转 | `store.as_retriever(...)` | 给取货员发工作证（变成 LangChain 标准接口） |

 Retriever 在 LCEL 里的接法

chain = (
    {"context": retriever | _fmt, "question": RunnablePassthrough()}   # ← D2 学的并行+透传+加工
    | prompt
    | model
    | parser
)
retriever 的输出自动塞进 prompt 的 {context} 占位符——你 D2 学的 RunnableParallel（并行拿 context 和 question）、RunnablePassthrough（原样传问题）、RunnableLambda（_fmt 加工文档），今天全用上了

检索参数：k 和 score_threshold

k=2：召回几条
score_threshold=0.3：相似度低于这个值的不要

最重要：和你 W3 的记忆同源
任务表原话：RAG 的"知识库"本质就是 W3 的"长期向量记忆（Chroma）"——今天练的 retriever，之后可以直接复用 W3 的 LongTermMemory，不必另起炉灶 
W4_D4任务表.md
。

你 W3 的 LongTermMemory.recall(query, k=3) 就是"取货员"，只是它返回 list[str]，LangChain 要的是 list[Document]——包一层就行。