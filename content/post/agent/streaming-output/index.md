---
description: ""
title: " 流式输出：模型怎么\"一个字一个字\"说话"
draft: false
date: "2026-08-07T23:42:45+08:00"
slug: "streaming-output"
categories:
 - null
tags:
 - null
image: ""
---


# W1 · 流式输出：模型怎么"一个字一个字"说话

> **关键词**：`stream=True` / chunk / `choices[0]` / `delta` / `reasoning_content`
> **前置**：会写一行 `client.chat.completions.create(...)`
> **收尾预告**：拿到完整文字后，下一步是记住多轮对话（Few-shot / prompt 组装见下一篇）

* * *

## 零、为什么需要流式输出

先想一个朴素的问题：**你等模型回答时，愿意干等吗？**

非流式（默认）的回答方式是：模型把整段话**全部生成完**，一次性返回给你。生成一段 200 字的话可能要好几秒——这几秒里你的屏幕一片空白，用户以为程序死了。

流式（`stream=True`）不一样：**模型生成一个词，就立刻吐一个词给你**。

```fallback
  非流式：  [.......... 等待 5 秒 ..........] → "北京今天晴，26℃"
  流式：    "北" → "京" → "今" → "天" → "晴" → ...  → 全程秒出第一个字
```

> **类比**：非流式是**餐厅攒齐一整桌菜才上**；流式是**做好一道上一道**——客人（用户）边吃边等，感知上快得多。这就是你在各种 AI 界面上看到的"打字机效果"的来源。

**流式的本质**：把"一次完整的响应"拆成"一截一截的增量"（chunk）逐个返回。本文就把这个机制拆开看。

* * *

## 一、先解剖响应结构：`response.choices[0].delta`

### 1.1 剥洋葱：三层路径

流式循环里你写的 `chunk.choices[0].delta.content`，是一条**三层深的路径**。先看清每一层是什么：

```fallback
  chunk（一截增量）
    │
    ├── choices  ← 数组！为什么是数组？
    │     │
    │     └── [0]  ← 取第 0 个候选
    │           │
    │           └── delta  ← 增量，不是完整答案
    │                 │
    │                 ├── content            正式回答的文字增量
    │                 ├── reasoning_content  思考过程的文字增量（DeepSeek）
    │                 └── role               角色标记（只在第一截出现）
    │
    └── finish_reason  ← 流快结束时才有："stop" / "length" ...
```

### 1.2 为什么 `choices` 是数组

OpenAI 系 SDK 支持**一个请求同时生成多个候选答案**——参数 `n=2` 就是让它给两份不同回答，每个候选是 `choices` 列表里的一项。

```python
# n=1（默认）：只生成一份答案 → choices 只有一项
# n=3：生成三份 → choices 有三项，你可以挑最好的那份
```

但**你日常流式场景只有一个候选**，所以永远取第 0 个：

```python
chunk.choices[0].delta.content   # 永远 [0]
```

> 💡 记不住就记住一句：**`choices[0]` 是个"礼貌性下标"**——框架把"可能有很多个答案"设计成了列表，而你几乎总是只用第一个。

### 1.3 `delta` 是什么：增量

剥到 `choices[0]` 之后，里面那个 `delta` **才是真正装着文字的地方**。

`delta` 的英文原意就是**"增量"（Δ）**——它不是完整答案，而是**这一截新长出来的那一小块**。

```fallback
  模型的完整答案："北京今天晴，26℃"

  流式吐出来是：
    chunk 1: delta.content = "北京"
    chunk 2: delta.content = "今天"
    chunk 3: delta.content = "晴，26"
    chunk 4: delta.content = "℃"

  你做的事：content += chunk 的 delta   → 最终拼回 "北京今天晴，26℃"
```

**`delta` 的关键认知**：**它永远只是"这一截的增量"，不是"到目前为止的全部"。** 所以你不能把每个 `delta` 直接当最终结果——**必须手动累加**，这一截加一截，最后才是完整回答。

* * *

## 二、最小流式代码（逐行注释）

```python
import os
from openai import OpenAI

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),     # 密钥走环境变量，不硬编码
    base_url="https://api.deepseek.com",       # DeepSeek 走 OpenAI 兼容协议
)

messages = [{"role": "user", "content": "你是哪个大模型"}]   # 初始对话：只有一条用户消息

# 关键就一个参数：stream=True —— 让响应变成"一截一截吐"
response = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=messages,
    stream=True,                    # 🎯 开启流式
    reasoning_effort="low",         # DeepSeek 思考强度
    extra_body={"thinking": {"type": "enabled"}},   # 开启深度思考（reasoning 通道）
)

# 两个累加器：一个收思考过程，一个收正式回答
reasoning_content = ""
content = ""

for chunk in response:                        # 遍历每一截 chunk
    # 思考过程的文字增量（DeepSeek 的"草稿纸"）
    if chunk.choices[0].delta.reasoning_content:
        reasoning_content += chunk.choices[0].delta.reasoning_content
    # 正式回答的文字增量（给用户看的"最终稿"）
    if chunk.choices[0].delta.content:
        content += chunk.choices[0].delta.content

print("Reasoning Content:", reasoning_content)   # 模型自己想的（可选展示）
print("Content:", content)                       # 正式回答
```

> 💡 **先判空再累加**：`if delta.content:` 里的判空不是防御性装饰，是**必须的**——下一节解释为什么。

* * *

## 三、🎯 DeepSeek 双通道：草稿纸与正式稿

开流式 + 深度思考后，DeepSeek 会**同时开两条文字流**：

```fallback
  通道 A  reasoning_content   模型的思考过程（草稿纸）
        —— 模型在心里推理：用户问"你是哪个模型"，
           先想"他可能在测我有没有系统提示泄露"……
        —— 这条流不该直接给用户看（可能泄露 prompt、产生废话）

  通道 B  content             正式回答（最终稿）
        —— "我是 DeepSeek 大模型，由深度求索公司开发……"
        —— 这条才是要展示给用户的
```

**为什么要分两条流**？因为思考过程是"内部草稿"，正式回答是"对外成品"。**把思考流直接刷给用户，体验很差**（一堆"嗯，用户想…""首先我要…"），而且**思考里可能复述了你的 system 提示词**——有泄露风险。

> **类比**：你写作文，先打草稿（reasoning_content），再誊抄到答题卡（content）。交上去的是答题卡，不是草稿纸。

### 什么时候只有草稿纸、没有正式稿

有些"纯推理"场景，模型思考了一大段，最后只给个极简结论甚至没有 content——此时 `content` 累加器是空的。代码里的兜底处理：

```python
if content:
    return content            # 有正式稿，用正式稿
return reasoning_content      # 没有正式稿（纯思考题），把思考结果当答案
```

> ⚠️ 这个兜底是**演示用**。生产环境要给用户看什么、能看什么，是产品决策——但技术前提是：**你得先能收到这两条流，才有得选。**

* * *

## 四、🎯 流式解析里的经典坑：三种"空 chunk"

流式返回里**不是每个 chunk 都带文字**。开头只有角色标记、收尾只有结束标志——`delta.content` 可能是 `None`。

直接 `content += delta.content` 会炸：

```fallback
TypeError: can only concatenate str (not "NoneType") to str
```

所以拼接前必须先判空：`if delta.content:` 再累加。**这是流式解析的标准写法。**

### 为什么会有空 chunk：三种典型情况

```fallback
  ① 流刚开始
     chunk = {delta: {role: "assistant"}}      ← 只有角色标记，没有 content
     模型先"开口"声明身份，还没开始说话

  ② 思考阶段
     chunk = {delta: {reasoning_content: "..."}}  ← 只有草稿纸，content 是空的
     模型正在想，还没落到正式稿

  ③ 流快结束
     chunk = {finish_reason: "stop"}           ← 只带结束标志，content 是 None
     模型说完了，最后补一个"我说完了"的信号
```

**把三种空 chunk 画成一条时间线**：

```fallback
  chunk 1      →  role="assistant"              （① 开口）
  chunk 2..n   →  reasoning_content="..."       （② 思考，content 为空）
  chunk n+1..  →  content="..."                 （正式开始说人话）
  chunk ...    →  content=None + finish_reason  （③ 收尾）
```

**判断口诀**：

```python
# 收思考：   delta.reasoning_content 有值 → 累加
# 收正文：   delta.content 有值 → 累加
# 收尾信号： chunk 里没有 delta 文字了，或 finish_reason 出现 → 循环该结束
```

### 一个隐藏的坑：`reasoning_content` 不是每个模型都有

```python
if delta.reasoning_content:      # ← 这个属性，普通模型（不开 thinking）上不存在
```

DeepSeek 不开 `thinking` 时，chunk 的 `delta` 里**根本没有 `reasoning_content` 这个字段**——直接访问是安全的（返回 `None`），但要小心某些 SDK 版本行为不一。**稳妥写法是先判空再累加，不要假设它一定存在**。

* * *

## 五、messages 里三种角色各管什么

流式循环外面的 `messages` 结构，是**发请求时**的对话历史。三种角色分工：

| 角色 | 干什么 | 谁写的 |
|---|---|---|
| `system` | **规则制定者**：你设定的人设/规则 | 你 |
| `user` | **用户说的话** | 用户 / 你替用户组装 |
| `assistant` | **模型之前说过的话** | 上一轮响应的内容 |

```python
messages = [
    {"role": "system", "content": "你是耐心耐心的 Agent 老师，擅长用通俗语言解释。"},
    {"role": "user", "content": "什么是流式输出？"},       # 本轮问题
    # ← 下一轮：把模型上一轮的回答 append 成 assistant
]
```

**为什么要区分角色**：模型是"接着往下说"，不是"每次都重新开始"。**它需要知道哪些是你说的、哪些是它自己说过的**——角色标签就是它的"记忆上下文"。

> 💡 这是流式输出的"另一半"：**流式解决"怎么收"，messages 解决"怎么发"。** 下一篇讲多轮对话与 prompt 组装时，`assistant` 角色会被反复追加——现在先记住分工。

* * *

## 六、踩坑记

### 坑 1：直接 `+= delta.content` 报 `TypeError` 🔴

- **现象**：`TypeError: can only concatenate str (not "NoneType") to str`
- **原因**：三种"空 chunk"（开头只有 role / 思考阶段 / 收尾标志）里 `delta.content` 是 `None`
- **解法**：**先判空再累加** `if delta.content: content += delta.content`——这是流式解析的标准写法，别省略

### 坑 2：把 `reasoning_content` 直接刷给用户 ⚠️

- **现象**：界面上全是"首先我要…""用户可能在问…"的碎碎念
- **原因**：把思考流（草稿纸）当成正文展示
- **解法**：两条流分开收，**只展示 `content`**；思考流用于调试/可选展示。理由：思考会复述 prompt、产生废话

### 坑 3：以为每个 chunk 都是完整句子 ⚠️

- **现象**：拼出来的结果字序错乱，或 chunk 单独看不成句
- **原因**：误解 `delta` 语义——它是**增量**不是**当前全部**
- **解法**：记住 `delta` 英文原意 Δ：**一截新长出来的字**，必须累加才能还原全文

### 坑 4：`api_key=""` 硬编码 ⚠️

- **现象**：代码能跑，但 key 写死在源码里，一 push 就泄露
- **解法**：`os.getenv("DEEPSEEK_API_KEY")` + `.env` 文件（key 进 `.gitignore`）

### 坑 5：不开 `thinking` 却访问 `reasoning_content` ⚠️

- **现象**：属性访问行为随 SDK 版本飘
- **原因**：普通模型的 chunk 里没有这个字段
- **解法**：`if delta.reasoning_content:` 判空；不要假设字段一定存在

* * *

## 七、速查卡片（复习直接看这）

```python
# ===== 最小流式骨架 =====
import os
from openai import OpenAI

client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"),
                base_url="https://api.deepseek.com")

def get_response(messages, **kwargs):
    """流式请求 + 双通道收集（思考 + 正文）。返回完整正文。"""
    response = client.chat.completions.create(
        model=kwargs.get("model", "deepseek-v4-flash"),
        messages=messages,
        stream=True,                                  # 🎯 流式开关
        reasoning_effort=kwargs.get("reasoning_effort", "low"),
        max_tokens=kwargs.get("max_tokens", 500),
    )
    reasoning_content, content = "", ""
    for chunk in response:
        delta = chunk.choices[0].delta                 # 三层路径：choices[0].delta
        if delta.reasoning_content:                    # 草稿纸（思考）
            reasoning_content += delta.reasoning_content
        if delta.content:                              # 正式稿（正文）—— 先判空！
            content += delta.content
    return content or reasoning_content                # 没有正式稿就把思考当答案

# ===== 三条认知 =====
# 1. delta = 增量（Δ），不是完整答案，必须累加
# 2. choices[0] 是礼貌性下标：n 候选里你只用第一个
# 3. 空 chunk 有三种：开头 role / 思考阶段 / 收尾 finish_reason —— 先判空再拼接
```

* * *

## 八、一句话总结

**流式输出 = 模型把一句完整的话，拆成一截一截的 `chunk` 逐个吐给你；你沿 `chunk.choices[0].delta.content` 这条三层路径，把每一截的增量累加起来，还原成完整回答。**

三个必须记住的认知：

1. **`delta` 是增量不是全部**——不累加就拿不到完整答案
2. **`delta.content` 经常是 `None`**——开头/思考/收尾三种空 chunk，先判空再拼接
3. **DeepSeek 有两条流**——`reasoning_content`（草稿纸）和 `content`（正式稿），给用户看后者

**流式解决的是"怎么把答案收下来"。但要让对话有来有回，还得知道"怎么把消息发出去"**——`messages` 的角色组装、让模型记住前面说过什么，就是下一篇的内容了。

* * *
