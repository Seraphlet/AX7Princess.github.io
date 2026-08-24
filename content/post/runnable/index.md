---
description: ""
title: " Runnable"
draft: false
date: "2026-08-24T00:12:57+08:00"
slug: "Runnable"
categories:
 - Agent
 - LaangChain
tags:
 - 
image: ""
---

# Runnable 组合件——并行、透传、加工、分支

---

## 翻车前的自白

昨天你拼出第一条链：`prompt | model | parser`——一步接一步，**串行**。

但真实应用从来不是一条直道：
- 想**同时**拿到「回答」和「关键词提取」→ 要并行，别等两次
- 想在链中间**保留原始问题**、再追加加工结果 → 要透传 + 追加
- 想把 W3 自研的 `compress()` / `recall()` **塞进链里** → 要把函数变成链的一环
- 想按问题类型**走不同链路**（天气 / 计算 / 默认）→ 要条件分支

昨天那四个需求，光靠 `|` 串不起来。今天这四件「组合件」就是干这个的——它们把「单链」升级成「有分流、合并、分支的管道」。

而且你会发现：**每一件， W1-W3 都手写过一遍，只是名字不同。**

---

## 一、为什么线性链不够用？

`A | B | C` 的本质是「上一个的输出，整个喂给下一个」。问题在于：

1. **它只能串行**：要两个产出就得跑两遍链，浪费 token 和时间。
2. **它会「覆盖」输入**：B 拿到的是 A 的输出，原始输入（比如用户原话）如果 A 改了，B 就拿不到了。
3. **它接不进你的老代码**：你 W3 的 `compress()` 是个普通 Python 函数，不是 Runnable，直接 `|` 会报错。
4. **它不会「分叉」**：所有输入走同一条路，无法按内容选不同处理。

这四件组合件，正好一一补上这四个缺口。

---

## 二、四件新武器（对照你的自研）

| 新武器 | 干啥的 | 你写过的对应物 |
|---|---|---|
| `RunnableParallel` | 一条输入，同时喂给 N 条链，结果合并成 dict | **无**（这是新能力：并行） |
| `RunnablePassthrough` | 原样透传，常用 `.assign()` 保留原输入 + 追加新字段 | `get_context` 里「保留原消息 + 插新消息」 |
| `RunnableLambda` | 把任意 Python 函数包成 Runnable | 你的 `compress()` / `recall()` 自研函数 |
| `RunnableBranch` | 按条件把输入分到不同子链 | 你的 `auto_select` 路由 |

**关键认知**：`RunnableParallel` 是你之前「没有」的能力（手写并行要自己搞线程/异步）；其余三件，本质是**把你 W1-W3 手写的逻辑，套上 Runnable 外壳**——所以你不是学新东西，是在给旧轮子装「标准接口」，好让它们能进 `|` 管道。

---

## 三、逐个拆解（带代码）

### ① RunnableParallel —— 并行流水线

一条输入同时喂给多条链，最后汇总成一个 dict。键名就是你起的名字。

```python
import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableParallel

load_dotenv()
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)
parser = StrOutputParser()

# 两条子链：一个回答，一个提取关键词
answer_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个简洁回答的助手"),
    ("user", "{question}"),
])
kw_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是关键词提取器，请输出3个关键词，用逗号分隔"),
    ("user", "{question}"),
])
answer_chain = answer_prompt | model | parser
kw_chain = kw_prompt | model | parser

# 并行：一条 {"question": ...} 同时进两条链
parallel_chain = RunnableParallel(
    answer=answer_chain,
    keywords=kw_chain,
)

print("=== RunnableParallel 并行 ===")
res = parallel_chain.invoke({"question": "什么是LCEL？"})
print(res)                                       # {'answer': '...', 'keywords': '...'}
print("类型:", {k: type(v).__name__ for k, v in res.items()})   # 每个值都是 str
```

**对照你的自研**：串行链只能「一次一个产出」。昨天你若想拿回答+关键词，得跑两遍。今天一句 `RunnableParallel` 搞定——**这是你之前没有的新能力**。

---

### ② RunnablePassthrough —— 复印机（保留 + 追加）

`RunnablePassthrough` 把输入**原样**传给下一步，不改动。它的升级版 `.assign()` 更常用：**保留原输入，再追加新字段**。

```python
from langchain_core.runnables import RunnablePassthrough

# 保留 {"question": ...}，再追加「长度」「前5个字」
enrich = RunnablePassthrough.assign(
    长度=lambda x: len(x["question"]),
    前5个字=lambda x: x["question"][:5],
)
out = enrich.invoke({"question": "什么是LCEL？"})
print(out)   # {'question': '什么是LCEL？', '长度': 7, '前5个字': '什么是LC'}
```

**对照你的自研**：这跟你 W3 `get_context` 里干的完全一致——「保留原消息列表 + 插入一条『已知用户长期事实』」。`get_context` 是你手写 `sys_msgs + [facts_msg] + others` 拼接；`.assign()` 是框架版「保留 + 插字段」。**D3 接 MemoryManager 时就靠它保留窗口、追加长期事实。**

---

### ③ RunnableLambda —— 临时加工站（D3 接 MemoryManager 的桥）

`RunnableLambda` 把任意 Python 函数包装成 Runnable，塞进链里。**这是 D3 把你 W3 `compress()` / `recall()` 接进链的关键桥。**

```python
from langchain_core.runnables import RunnableLambda

def TextCleansing(text: str) -> str:
    """模拟自研后处理函数：去多余空格并截断到 50 字"""
    return " ".join(text.split())[:50]

# 函数包成 Runnable，直接接到链尾
chain_with_func = answer_prompt | model | parser | RunnableLambda(TextCleansing)

print("=== RunnableLambda 加工站 ===")
out_e = chain_with_func.invoke({"question": "用一句话介绍 LangChain"})
print(out_e)
print("类型:", type(out_e).__name__)     # str
```

**对照你的自研**：`TextCleansing` 是个普通函数。你 W3 的 `compress()`（压缩）、`recall()`（召回）也是普通函数。把它们用 `RunnableLambda(...)` 一包，就能写进 `|` 管道——**不用改函数内部一行代码**。D3 的实战就是：`get_context` 包成 Lambda 接进链，让「取记忆」成为链的一环。

> ⚠️ 注意类型对接：Lambda 拿到的是**上一个 Runnable 的输出类型**。上面 `parser` 产出 `str`，所以 `TextCleansing(text: str)` 接收 `str` 才对得上。如果上一个产出是 dict，你的函数参数就得是 dict。

---

### ④ RunnableBranch —— 岔路口（W5 LangGraph 条件 edge 雏形）

`RunnableBranch` 按条件把输入分到不同子链。条件**按顺序短路**（第一个命中就走，不走后面的），最后一个是**默认分支**（无条件的 Runnable，不是元组）。

```python
from langchain_core.runnables import RunnableBranch

weather_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是天气助手，用一句话回答天气问题"),
    ("user", "{question}"),
])
calc_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是计算小助手，只输出计算结果"),
    ("user", "{question}"),
])
default_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是智能助手，简要回答"),
    ("user", "{question}"),
])
weather_chain = weather_prompt | model | parser
calc_chain = calc_prompt | model | parser
default_chain = default_prompt | model | parser

# (条件, 子链) 元组列表 + 最后放默认链
branch = RunnableBranch(
    (lambda x: "天气" in x["question"], weather_chain),
    (lambda x: "计算" in x["question"], calc_chain),
    default_chain,
)

print("=== RunnableBranch 分支 ===")
for q in ["北京天气怎么样？", "计算 12*8", "你好呀"]:
    ans = branch.invoke({"question": q})
    print(f"问: {q}\n答: {ans}\n")
```

**对照你的自研**：这就是你 W2 的 `auto_select` 路由——「看用户输入，决定走哪条工具链」。`RunnableBranch` 把这套 if/elif 路由标准化了。**它正是 W5 LangGraph「条件 edge」的雏形**：在图里，节点之间的边可以按条件选走哪条。

---

## 四、可视化调试：get_graph().print_ascii()

链一复杂（并行、分支、Lambda），肉眼看不清数据流向。每个 Runnable 都能 `.get_graph().print_ascii()` 打印结构。

```python
print("=== 并行链结构图 ===")
parallel_chain.get_graph().print_ascii()   # 应看到 question → answer / keywords 两路

print("\n=== 分支链结构图 ===")
branch.get_graph().print_ascii()           # 应看到按条件分三路
```

**为什么必用**：你改完一条链，先打一张图，比对着报错猜「数据从哪断的」快十倍。这是你排错的第一双眼——和昨天踩坑记里说的一样。

---

## 五、完整代码（可直接抄）

```python
import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import (
    RunnableParallel,
    RunnablePassthrough,
    RunnableLambda,
    RunnableBranch,
)

load_dotenv()
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)
parser = StrOutputParser()

# ---- 两条基础子链 ----
answer_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个简洁回答的助手"),
    ("user", "{question}"),
])
kw_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是关键词提取器，请输出3个关键词，用逗号分隔"),
    ("user", "{question}"),
])
answer_chain = answer_prompt | model | parser
kw_chain = kw_prompt | model | parser

# 步骤A1：RunnableParallel 并行
parallel_chain = RunnableParallel(
    answer=answer_chain,
    keywords=kw_chain,
)
print("=== 步骤A1：RunnableParallel 并行 ===")
res = parallel_chain.invoke({"question": "什么是LCEL？"})
print(res)
print("类型:", {k: type(v).__name__ for k, v in res.items()})

# 步骤A2：RunnablePassthrough.assign 追加字段
print("\n=== 步骤A2：RunnablePassthrough.assign 追加字段 ===")
enrich = RunnablePassthrough.assign(
    长度=lambda x: len(x["question"]),
    前5个字=lambda x: x["question"][:5],
)
out = enrich.invoke({"question": "什么是LCEL？"})
print(out)

# 步骤C：RunnableBranch 分支
weather_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是天气助手，用一句话回答天气问题"),
    ("user", "{question}"),
])
calc_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是计算小助手，只输出计算结果"),
    ("user", "{question}"),
])
default_prompt = ChatPromptTemplate.from_messages([
    ("system", "你只智能助手,简要回答"),
    ("user", "{question}"),
])
weather_chain = weather_prompt | model | parser
calc_chain = calc_prompt | model | parser
default_chain = default_prompt | model | parser

branch = RunnableBranch(
    (lambda x: "天气" in x["question"], weather_chain),
    (lambda x: "计算" in x["question"], calc_chain),
    default_chain,
)
print("\n=== 步骤C：RunnableBranch 分支 ===")
for q in ["北京天气怎么样？", "计算 12*8", "你好呀"]:
    ans = branch.invoke({"question": q})
    print(f"问: {q}\n答: {ans}\n")

# 步骤D：get_graph().print_ascii() 可视化
print("=== 步骤D：并行链结构图 ===")
parallel_chain.get_graph().print_ascii()
print("\n=== 步骤D：分支链结构图 ===")
branch.get_graph().print_ascii()

# 步骤E：RunnableLambda 接自研函数
def TextCleansing(text: str) -> str:
    """模拟自研函数，无实际逻辑处理"""
    return " ".join(text.split())[:50]

chain_with_func = answer_prompt | model | parser | RunnableLambda(TextCleansing)
print("\n=== 步骤E：RunnableLambda 加工站 ===")
out_e = chain_with_func.invoke({"question": "用一句话介绍 LangChain"})
print(out_e)
print("类型:", type(out_e).__name__)
```

---

## 六、踩坑记 & 知识点

**① RunnableParallel 返回的是 dict，键名 = 你起的名**
`RunnableParallel(answer=..., keywords=...)` 的结果永远是 `{"answer": ..., "keywords": ...}`。别指望它返回 str 或 list——它是「多产出汇总」，天然是 dict。验收直接 `res["answer"]`、`res["keywords"]`。

**② RunnablePassthrough.assign 是「保留 + 追加」，不是「替换」**
`.assign(新键=...)` 会**保留**原 dict 所有字段，再**加**新键。想「替换」某字段，得显式写 `原键=lambda x: 新值`。这点跟你 W3 `get_context` 的「`sys_msgs + [facts_msg] + others` 保留原系统提示」逻辑一致。

**③ RunnableBranch 条件按顺序短路，默认分支必须放最后且无条件**
`(条件, 链)` 元组从前往后判，命中第一个就走，后面不判。最后一个参数必须是**裸 Runnable**（不是元组），它是兜底。放反了（把默认放前面）会导致永远走默认。

**④ RunnableLambda 的「输入输出类型」要对齐上一个 Runnable**
Lambda 收到的是管道里**上一个**的输出类型。`parser` 出 `str` → 你的函数参数就得是 `str`；若上一个出 `dict` → 参数得是 `dict`。类型对不上不会在定义时报错，会在 `.invoke()` 时炸——所以**写完先跑一次**。

**⑤ get_graph().print_ascii() 是排错第一双眼**
并行/分支链肉眼难 Debug。改完链先打图确认流向，再写业务逻辑。跟昨天踩坑记④一脉相承。

---

## 七、总结

今天四件组合件，把「单链」升级成「有分流、合并、分支的管道」：

- **RunnableParallel** = 并行流水线（一条输入 → N 条链 → dict 汇总），是你之前没有的新能力
- **RunnablePassthrough** = 复印机（`.assign()` 保留原输入 + 追加字段），对应你 W3 `get_context` 的「保留 + 插消息」
- **RunnableLambda** = 加工站（任意函数包成 Runnable），是 D3 把 W3 `compress()`/`recall()` 接进链的桥
- **RunnableBranch** = 岔路口（按条件走不同子链），对应你 W2 `auto_select` 路由，也是 W5 LangGraph 条件 edge 的雏形

它们和你 W1-W3 的自研**一一对应**——**框架化不是学新东西，是把你的手动逻辑套上标准接口，好让它们能进 `|` 管道。**

---
