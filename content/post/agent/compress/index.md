---
description: ""
title: "上下文压缩，提取摘要"
draft: false
date: "2026-08-18T14:52:23+08:00"
slug: "compress"
categories:
 - Agent
tags:
 - memory
image: ""
---

# 把老历史浓缩成一条线：滚动摘要式上下文压缩的设计复盘

## 一、先搞清楚：为什么要压缩

Agent 跑久了，消息列表会无限增长。每轮对话都往 `messages` 里追加 user 和 assistant 的内容，几十轮下来，几个问题接踵而至：

| 问题 | 后果 |
|------|------|
| **Token 超限** | 大模型 context window 有上限（GPT-4o 128K，DeepSeek 64K），超了直接报错 |
| **成本飙升** | API 按 token 计费，每轮都把全部历史重发一遍，费用线性增长 |
| **延迟增大** | prompt 越长，推理越慢，用户等得越久 |
| **注意力稀释** | 上下文太长，模型对早期信息的关注度下降（"Lost in the Middle"问题） |

新手的第一反应是**截断**——砍掉老消息，只留最近几轮。简单粗暴，但问题是：**砍掉的信息就彻底没了**。用户十轮前说的偏好、做过的决策，全丢了。

所以我们需要一种更聪明的办法：**压是浓缩，不是丢掉**。

---

## 二、压缩策略选型：三种思路对比

在写代码之前，先看看业界主流的上下文压缩方案：

| 策略 | 做法 | 优点 | 缺点 |
|------|------|------|------|
| **截断（Truncation）** | 只留最近 N 条，老消息直接扔 | 零成本、零延迟 | 信息完全丢失 |
| **提取（Extraction）** | 从老消息中抽取关键实体/事实，结构化存储 | 信息精确、可检索 | 需要 NER 或规则引擎，实现复杂 |
| **摘要（Summarization）** | 把老历史交给 LLM 浓缩成一段摘要 | 保留语义、实现简单 | 每次压缩多一次 LLM 调用；有信息损失 |

我们选的是**摘要**，具体来说是**滚动摘要（Rolling Summary）**——不积累多条摘要，而是永远只维护一条，新摘要覆盖旧摘要。

一句话记住：**截断是删聊天记录，提取是做笔记，摘要是写周报。** 周报写完，旧的聊天记录可以删，但信息以浓缩形式留住了。

---

## 三、核心设计：滚动摘要的三条铁律

### 铁律一：压什么——只压窗口外的老历史

消息列表不是铁板一块，压缩前先分三类：

```
┌─────────────────────────────────────────────┐
│  messages 列表                                │
│                                               │
│  ┌──────────────┐  ← 原系统提示（人格/角色设定）│
│  │ system (原)   │     永远不动，不参与压缩      │
│  ├──────────────┤                             │
│  │ 历史摘要      │  ← 上一轮压缩产生的摘要        │
│  │ (is_summary)  │     每次压缩会被新摘要替换     │
│  ├──────────────┤                             │
│  │ old messages  │  ← 滚出窗口的老对话           │
│  │ (要被压缩的)   │     压完就扔，只留浓缩版       │
│  ├──────────────┤                             │
│  │ recent × K    │  ← 最近 K 条原文              │
│  │ (保留不压)     │     模型对"刚说的话"不能断片   │
│  └──────────────┘                             │
└─────────────────────────────────────────────┘
```

关键：**最近 K 条（4~6 轮）原文必须保留**。为什么？因为模型需要对"刚刚的上下文"保持连贯理解。你压缩了上一轮的对话，模型回复就会前言不搭后语。

### 铁律二：什么时候压——懒触发

不是每轮对话都压缩。压缩要调一次 LLM（摘要本身也是一次推理），有成本有延迟。所以策略是：

> **超过 token 阈值才触发，没超就原样返回。**

这就是 `maybe_compress` 干的事——先估算当前消息列表的 token 数，超了才调 `compress`，没超直接返回原列表。省一次 LLM 调用。

### 铁律三：压完别二次压缩

这是最容易被忽略的坑。压缩后，列表变成：

```
[原system, 摘要(is_summary=True), recent×4]
```

下次再触发压缩时，如果不把 `is_summary` 的消息单独拎出来，它会被当成"普通老消息"再压一遍。**摘要的摘要，越压越短，信息指数级丢失**——就像你把周报再写成周报的周报，最后只剩一句话。

解决办法：用 `is_summary` 标记区分"摘要"和"真实系统提示"，压缩时把旧摘要单独取出来，和新滚出窗口的原文**合并成一条新摘要**，旧的扔掉。永远只有一条摘要。

---

## 四、代码长什么样

`memory/compress.py` 一共四个函数，各管一段。下面逐块拆解，文末给出完整源码。

### `_estimate_tokens` —— 零依赖的 token 估算

```python
def _estimate_tokens(text: str) -> int:
    """粗略估算 token 数（无外部依赖）：中文约 0.6 token/字，英文约 4 字符/token。"""
    return max(1, int(len(text) * 0.6))
```

一行核心逻辑：`max(1, int(len(text) * 0.6))`。不引入 `tiktoken` 等外部依赖，按"中文约 0.6 token/字"粗估，够用于阈值判断。

**知识点补充：Token 是什么？**

Token 是大模型处理文本的最小单位。它不是"一个字"也不是"一个词"，而是介于两者之间的子词单元（subword）。OpenAI 用 BPE（Byte Pair Encoding）分词，DeepSeek 用类似的 tokenizer。大致规律：

| 语言 | 分词粒度 | 估算系数 |
|------|----------|----------|
| 中文 | 1 字 ≈ 0.6~1 token | `len(text) * 0.6` 粗估 |
| 英文 | 1 token ≈ 4 字符 | `len(text) / 4` 粗估 |
| 代码 | 符号多，1 token ≈ 3~4 字符 | 比英文略碎 |

精确计数要用 `tiktoken`（OpenAI）或对应模型的 tokenizer，但做阈值判断时粗估就够了——我们不需要知道"到底 2999 还是 3001"，只需要知道"是不是该压了"。

### `_fmt` —— 消息列表转纯文本

```python
def _fmt(messages) -> str:
    """消息列表 → 纯文本（喂给 LLM 做摘要的原料）。"""
    return "\n".join(f"{m['role']}: {m['content']}" for m in messages)
```

一行 join：把结构化的 `messages`（OpenAI 格式的 `list[dict]`）拍平成 `"role: content"` 每条一行的纯文本，方便塞进一个 prompt 让 LLM 做摘要，也顺带作为 token 估算的输入。

### `compress` —— 核心压缩逻辑

```python
def compress(messages: list[dict], llm, keep: int = 4) -> list[dict]:
    """滚动摘要版压缩。llm 需要提供 summarize(text) -> str 方法。"""
    # ① 三类消息分开：原系统提示 / 旧摘要 / 其余对话
    #    用 is_summary 标记区分"摘要"和"真实系统提示"，防止混为一谈
    sys_msgs = [m for m in messages
                if m.get("role") == "system" and not m.get("is_summary")]
    old_sums = [m for m in messages if m.get("is_summary")]
    others   = [m for m in messages
                if m.get("role") != "system" and not m.get("is_summary")]

    if len(others) <= keep:
        return messages          # 还没长到需要压，原样返回

    old, recent = others[:-keep], others[-keep:]

    # ② 滚动关键：旧摘要 + 新滚出窗口的原文 → 合并成一条新摘要
    #    这就是"写周报"：把上周周报和这周的事浓缩成一份新周报，旧的扔掉
    text = _fmt(old)
    if old_sums:
        text = f"已有摘要: {old_sums[-1]['content']}\n新增对话:\n{text}"

    # ③ 交给 LLM 浓缩。生产环境的提示词要强调：保留事实/决策/偏好、控制长度
    summary = llm.summarize(text)
    summary_msg = {"role": "system",
                   "content": f"历史摘要: {summary}",
                   "is_summary": True}
    return sys_msgs + [summary_msg] + recent
```

逐步拆解三个步骤：

**① 分流**：用列表推导把消息分成三桶——原系统提示、旧摘要、其余对话。`is_summary` 是我们自定义的消息字段——OpenAI 的消息格式只认 `role` 和 `content`，但你可以往 dict 里塞额外字段，模型不会报错（它只读 `role` 和 `content`，多余的忽略）。我们利用这点来做内部标记。分流后如果 `len(others) <= keep`，说明还没长到需要压，原样返回。

**② 滚动合并**：这是整个设计的核心。`others[:-keep]` 是滚出窗口的老对话，`others[-keep:]` 是最近 K 条原文。如果已有旧摘要，就把旧摘要内容拼到新对话前面，一起喂给 LLM——LLM 看到的是"已有摘要: xxx + 新增对话: yyy"，输出的是合并后的新摘要。旧摘要被新摘要替代，**永远只有一条**。

**③ 重组**：最终消息列表 = 原系统提示 + 一条摘要（`is_summary=True`）+ 最近 K 条原文。结构清晰，角色明确。

### `maybe_compress` —— 自动触发器

```python
def maybe_compress(messages, llm, max_tokens: int = 3000, keep: int = 4):
    """自动触发：超阈值才压，没超原样返回。"""
    if (_estimate_tokens(_fmt(messages)) > max_tokens
            and len(messages) > keep + 1):
        return compress(messages, llm, keep=keep)
    return messages
```

外层的守门员，正常对话循环里每轮调一次它。两个条件**同时满足**才压：
1. token 总量超过阈值（`max_tokens`）
2. 消息条数大于 `keep + 1`（至少有一条可以压的旧消息）

为什么用 `and` 而不是 `or`？因为如果消息条数不够（比如就 3 条但每条特别长），没有足够的内容可以压缩，强行压只会把最近对话也塞进摘要，得不偿失。

### 完整代码

四个函数拼起来就是 `memory/compress.py` 的全部，可直接抄：

```python
# memory/compress.py
"""Day4 上下文压缩：老历史 → 一条滚动摘要，永远只有一条。"""

def _estimate_tokens(text: str) -> int:
    """粗略估算 token 数（无外部依赖）：中文约 0.6 token/字，英文约 4 字符/token。"""
    return max(1, int(len(text) * 0.6))

def _fmt(messages) -> str:
    """消息列表 → 纯文本（喂给 LLM 做摘要的原料）。"""
    return "\n".join(f"{m['role']}: {m['content']}" for m in messages)

def compress(messages: list[dict], llm, keep: int = 4) -> list[dict]:
    """滚动摘要版压缩。llm 需要提供 summarize(text) -> str 方法。"""
    # ① 三类消息分开：原系统提示 / 旧摘要 / 其余对话
    #    用 is_summary 标记区分"摘要"和"真实系统提示"，防止混为一谈
    sys_msgs = [m for m in messages
                if m.get("role") == "system" and not m.get("is_summary")]
    old_sums = [m for m in messages if m.get("is_summary")]
    others   = [m for m in messages
                if m.get("role") != "system" and not m.get("is_summary")]

    if len(others) <= keep:
        return messages          # 还没长到需要压，原样返回

    old, recent = others[:-keep], others[-keep:]

    # ② 滚动关键：旧摘要 + 新滚出窗口的原文 → 合并成一条新摘要
    #    这就是"写周报"：把上周周报和这周的事浓缩成一份新周报，旧的扔掉
    text = _fmt(old)
    if old_sums:
        text = f"已有摘要: {old_sums[-1]['content']}\n新增对话:\n{text}"

    # ③ 交给 LLM 浓缩。生产环境的提示词要强调：保留事实/决策/偏好、控制长度
    summary = llm.summarize(text)
    summary_msg = {"role": "system",
                   "content": f"历史摘要: {summary}",
                   "is_summary": True}
    return sys_msgs + [summary_msg] + recent

def maybe_compress(messages, llm, max_tokens: int = 3000, keep: int = 4):
    """自动触发：超阈值才压，没超原样返回。"""
    if (_estimate_tokens(_fmt(messages)) > max_tokens
            and len(messages) > keep + 1):
        return compress(messages, llm, keep=keep)
    return messages
```

---

## 五、LLM 摘要提示词怎么写

代码里 `llm.summarize(text)` 是个接口约定——`llm` 对象需要提供 `summarize` 方法。生产环境里，这个方法的提示词至关重要：

```python
def summarize(self, text: str) -> str:
    """将对话历史浓缩为摘要。"""
    prompt = f"""请将以下对话历史浓缩为一段摘要。
要求：
1. 保留关键事实、用户偏好、已做决策、未完成任务
2. 省略寒暄、重复内容、无信息量的对话
3. 控制在200字以内
4. 用第三人称客观陈述，不要编造未提及的信息

对话内容：
{text}"""
    return self.chat([{"role": "user", "content": prompt}])
```

四条要求各有深意：

| 要求 | 为什么 |
|------|--------|
| 保留事实/决策/偏好 | 这些是 Agent 后续行为依赖的信息，丢了就失忆 |
| 省略寒暄/重复 | "你好""谢谢"这类没有信息量，浪费摘要空间 |
| 控制长度 | 摘要本身也要占 token，不限制就失去了压缩的意义 |
| 第三人称客观陈述 | 防止 LLM 编造或脑补，只复述已有内容 |

---

## 六、踩坑记：二次压缩的信息坍缩

这个坑如果不踩一次，很难意识到。

假设没有 `is_summary` 标记，消息列表压缩后长这样：

```
# 第一次压缩后
[system(原), system(摘要), msg_5, msg_6, msg_7, msg_8]

# 又聊了4轮，触发第二次压缩
# 如果不区分摘要和普通消息，摘要会被当 old 再压一次
# 摘要的摘要 → 信息再砍一刀
```

每压一次，信息量打一个折扣。假设每次摘要保留 70% 的信息，压 5 次后只剩 `0.7^5 ≈ 23%`。这就是**信息坍缩**——和反复存 JPEG 一样，每存一次画质掉一档。

`is_summary` 标记的作用就是**把旧摘要从"待压缩"队列里拎出来**，单独和新的老对话合并成一条新摘要。旧摘要不会自己被再压，而是作为"已有摘要"参与新摘要的生成——信息有损失，但只损失一轮，不会指数级衰减。

**心智模型**：摘要不是消息列表里的普通一员，它是"历史浓缩版"，身份特殊。每次压缩时，它和新的老对话一起被重新浓缩，生成一条全新的摘要替代它。就像写周报：不是在旧周报上追加，而是把旧周报和这周的事一起重新写一份。

---

## 七、知识点补充：上下文压缩的更多背景

### 1. Context Window 与"Lost in the Middle"

大模型的 context window 不是"填多少都能用好"。研究表明，模型对 context 中间部分的信息关注度显著低于开头和结尾——这就是 **"Lost in the Middle"** 问题（Liu et al., 2023）。

这意味着：即使你的 context window 够大（128K），塞满历史消息也不一定有好效果。压缩不只是省 token，更是**把模型注意力集中在最近对话上**。

### 2. 摘要有界性

滚动摘要能控制 token 总量，前提是**摘要本身有界**。如果 LLM 每次摘要都写 1000 字，那摘要本身就成了新的负担。所以提示词里必须限制长度（如 200 字），保证：

```
总 token ≈ system(固定) + 摘要(有界) + recent×K(有界)
```

三项都有上界，总 token 就是 O(1) 级别的常数，不随对话轮次增长。

### 3. 其他压缩方案（了解即可）

| 方案 | 代表 | 思路 |
|------|------|------|
| **LLMLingua** | Microsoft | 用小模型做 token 级别的"软删除"——删掉对输出影响小的 token，保留重要的 |
| **MemGPT** | UC Berkeley | 操作系统式记忆管理——主内存(context) + 外存(磁盘)，按需"换页"加载 |
| **CompAct** | LangChain | 检测对话主题切换点，在主题边界做压缩，避免跨主题摘要信息混乱 |

我们的滚动摘要是最简方案：实现 40 行代码，够用，够理解原理。生产环境可以考虑接入 LLMLingua 做更细粒度的压缩。

---

## 八、总结

1. **压缩是浓缩不是丢掉**——滚动摘要把老历史浓缩成一条 system 消息，信息以压缩形式保留。
2. **三类消息分流是防错的基础**——原系统提示永留、摘要单独标记、普通对话才参与压缩。
3. **懒触发省成本**——没超阈值就不压，避免每轮对话都多一次 LLM 调用。
4. **`is_summary` 标记防止信息坍缩**——旧摘要不二次压缩，而是和新老对话合并成新摘要，永远只有一条。

给消息列表做个"收纳箱"：最近的东西放台面上随手拿（recent × K），老东西装箱贴标签（摘要），装箱后不重复装箱（is_summary）。箱子永远只有一个，但里面的东西是最新的浓缩版。