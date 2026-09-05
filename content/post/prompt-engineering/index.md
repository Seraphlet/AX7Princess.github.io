---
description: ""
title: "Prompt 四模式：CoT / Few-shot / ReAct / ToT"
draft: false
date: "2026-08-08T02:58:24+08:00"
slug: "Chain of Thought"
categories:
 - null
tags:
 - null
image: ""
---


# W1 · Prompt 四模式：CoT / Few-shot / ReAct / ToT

> **关键词**：Prompt 及格线 / CoT / Few-shot / ReAct / ToT / 代码路由
> **前置**：会流式收答案（见《初识流式输出》）
> **收尾预告**：ReAct 的"Observation"要真工具结果，下一站就是手撸 Function Calling

* * *

## 零、什么样的 prompt 算"过及格线"

先说个反直觉的判断：**Prompt 工程有没有及格，和技巧高不高级无关。**

一段"请一步步思考"能及格，一段花哨的角色扮演也能及格——判据不是"看起来专业"，而是**可验证**。三个硬指标：

| 指标 | 白话版 | 怎么检验 |
|---|---|---|
| **角色任务清晰** | 模型知道"你是谁、要干什么" | 换个人看这段 prompt，也能说出它的用途 |
| **输入输出边界明确** | 输入什么、输出什么格式说死了 | 连续跑 3 次，格式稳定 |
| **可复用可测试** | 换参数能复用，不是一次性 | 同一个函数换 3 组参数都能跑 |

**及格线 = 角色清晰 + 边界明确 + 可复用可测试。** 与技巧花哨与否无关。

> **为什么 Agent 工程日常做的是 prompt 工程，而不是微调？**
>
> 微调贵、慢、且改不动"工具调用"这种动态行为。prompt 工程便宜、灵活、可迭代——**先 prompt，不够再考虑微调**（那是模型工程师的活，不是 Agent 工程师的）。

> ⚠️ 顺带泼一盆冷水：**AI 是加速器，不是替代者。** 它能写代码，但架构设计、工具拆分、安全审查、调试评估、部署运维这些工程决策必须由人来做——**你学这些，是为了能判断它写得好不好、能在它之上搭产品。**

* * *

## 一、先有个统一底座：`get_response`

后面四种模式都只是"拼不同的 messages"，最后都要过一个**统一的请求函数**。上一篇文章详细拆过流式，这里直接给出完整版作为全篇的地基：

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"),
                base_url="https://api.deepseek.com")


def get_response(messages, **kwargs):
    """统一请求入口：流式接收 + 双通道收集（思考 + 正文）。
    注意：如果生成了正式 content，会把 assistant 消息存回 messages（见多轮对话）。"""
    response = client.chat.completions.create(
        model=kwargs.get("model", "deepseek-v4-flash"),
        messages=messages,
        stream=True,                                  # 流式：一截一截收
        reasoning_effort=kwargs.get("reasoning_effort", "low"),
        max_tokens=kwargs.get("max_tokens", 500),
    )
    reasoning_content, content = "", ""
    for chunk in response:
        delta = chunk.choices[0].delta
        if delta.reasoning_content:                   # 草稿纸（思考）
            reasoning_content += delta.reasoning_content
        if delta.content:                             # 正式稿（正文）
            content += delta.content
    if content:
        messages.append({"role": "assistant", "content": content})   # 存回历史
        return content
    return reasoning_content                          # 纯思考题兜底
```

> ⚠️ **注意那个副作用**：`get_response` 生成正文后会把 assistant 消息 `append` 回 `messages`。这是刻意设计——**多轮对话的"记忆"就是这么累积的**：每轮 user 进来、assistant 存回去，`messages` 越来越长，模型越来越"记得"。但它让一个纯函数带了副作用，后面模式工厂里你会看到两种用法打架。**先记住这个设计，第七章会回来处理它。**

* * *

## 二、CoT：把思考过程写出来

### 2.1 是什么

**CoT（Chain of Thought，思维链）= 别让模型"张口就答"，让它先把推理步骤写出来，再给结论。**

就像做数学题老师要求"写出解题过程"，不许直接填答案。

```python
def cot_prompt(question, system_prompt="你是一个逻辑缜密，思维严谨的推理助手"):
    """CoT：要求一步步思考再回答"""
    u = f"请一步步思考，再给出答案。\n问题:{question}"
    return [
        {"role": "system", "content": system_prompt},   # 人设（规则）
        {"role": "user", "content": u},                 # 任务（要求一步步想）
    ]
```

### 2.2 解决什么问题

**多步推理、数学逻辑题**——模型直接答容易"跳步出错"：

```fallback
  小明有 3 个苹果，又买了 2 打（12 个/打），分给 4 个朋友，每人几个？

  ❌ 直接答：5 个            ← 跳步：忘了把"2 打"换算成 24
  ✅ 一步步想：
     1) 3 + 2×12 = 27 个
     2) 27 ÷ 4 = 6.75
     3) 每人 6.75 个
```

**逼它写过程，就能大幅提高正确率**——因为每一步都摊开在眼前，跳步会被"自己看见"。

### 2.3 CoT 的本质：一条指令

CoT 不是某种固定格式，**它本质就是一条指令（"请一步步思考"）**：

- 放 `system` → 当**规则**（全程都这么答）
- 放 `user` → 当**任务要求**（这题这么答）

放哪都行，效果差异不大——选哪个取决于你要"全局生效"还是"单题生效"。

> 💡 **当你已经开了 DeepSeek 的 thinking（reasoning_content）时，还需要 prompt 层的 CoT 吗？** 两条路都能让模型"多想一步"，但机制不同：thinking 是模型厂商在推理层实现的；CoT 是你在 prompt 层要求的。**追求稳妥就两者都开**——模型层面多想 + 文本层面要求，双保险。

* * *

## 三、Few-shot：给范例，让模型照葫芦画瓢

### 3.1 是什么

**Few-shot = 在问题前面放几个「输入 → 输出」的例子，让模型模仿。**

```python
DEFAULT_EXAMPLES = [
    {"question": "我明天下午3点约了张医生复诊",
     "answer": "时间:明天下午3点;人物:张医生;事项:复诊"},
    {"question": "周五晚上和李总在望江楼吃饭",
     "answer": "时间:周五晚上;人物:李总;事项:吃饭"},
]

def few_shot_prompt(question, examples=DEFAULT_EXAMPLES,
                    system_prompt="你是擅长信息抽取的助手"):
    """Few-shot：先给 2 个问答范例，再问真问题。"""
    msgs = [{"role": "system", "content": system_prompt}]
    for ex in examples:
        msgs.append({"role": "user", "content": ex["question"]})    # 示例问题
        msgs.append({"role": "assistant", "content": ex["answer"]})  # 示例答案
    msgs.append({"role": "user", "content": question})               # 真问题
    return msgs
```

> **类比**：新员工入职，先看老员工处理过的几单（输入 → 标准输出），再自己上手。**给的是"范例"，不是"规则"。**

### 3.2 解决什么问题

**格式固定、样本少的任务**——比如"把消息分类成 情绪 / 任务 / 闲聊"。规则说不清（什么算"任务"？边界很模糊），但**给 2~3 个例子它立刻就会**。

```fallback
  用规则描述：  "判断用户意图，可能是任务、闲聊或情绪表达……"   ← 说不清
  用例子示范：
    user: "帮我订明天去上海的机票"    assistant: 任务
    user: "今天好累啊"                assistant: 情绪
    user: "讲个笑话吧"                assistant: 闲聊
    user: "提醒我晚上买牛奶"          → 模型自然会模仿成: 任务
```

### 3.3 🎯 Few-shot 的本质：示例必须组装成消息对

Few-shot 与 CoT 有个关键差别——**CoT 是一条指令，Few-shot 是一组"问答对"**。

而 API 只认 `user/assistant` 交替的 messages 结构，所以示例**必须拆成多轮消息对**：

```python
# ❌ 错误：把示例塞进一段 user 话里
{"role": "user", "content": "示例：Q:... A:... 示例2：Q:... A:... 现在回答：..."}

# ✅ 正确：示例拆成 user/assistant 交替（见 few_shot_prompt）
[user 示例1问题] [assistant 示例1答案] [user 示例2问题] [assistant 示例2答案] [user 真问题]
```

> 🎯 **这就是"prompt 必须用代码实现"的第一个理由**：文字是 content，代码是组装——**API 只接收带 role 的 messages 结构，Few-shot 的"一问一答"格式必须由代码拆成多轮消息**，手写文本做不到。

### 3.4 多轮对话练习：让"记忆"自然发生

把第一节那个"自动存回 assistant"的设计用起来，就得到了**天然带记忆的多轮对话**：

```python
def get_response(messages, **kwargs):
    # ……（同第一节：流式收集 + 存回 assistant）……

# system 定义人设：Agent 老师
messages = [{"role": "system", "content":
    "你是一个耐心 Agent 老师，擅长用通俗的语言解释学生提到的问题。"
    "请根据问题一步步分析学生当前的学习阶段，给出最适合当前阶段的回答。"}]

while True:
    user_input = input("User: ")
    if user_input.lower() in ["exit", "quit"]:
        break
    messages.append({"role": "user", "content": user_input})   # 本轮问题入队
    response = get_response(messages)                          # 内部会把 assistant 存回
    print("Agent导师:", response)
```

**看消息列表怎么自己长胖的**：

```fallback
  第 1 轮前：  [system, user₁]
  第 1 轮后：  [system, user₁, assistant₁]
  第 2 轮前：  [system, user₁, assistant₁, user₂]
  第 2 轮后：  [system, user₁, assistant₁, user₂, assistant₂]
  …… messages 每轮 +2，模型每轮都"记得"前面聊过什么
```

> 💡 **这就是最简单的记忆**：把历史全塞进上下文。它的上限是 token 窗口——聊长了会爆。怎么在"全记住"和"装不下"之间取舍，是 W3 `MemoryManager` 要解决的事，这里先建立直觉。

* * *

## 四、ReAct：边想边做（Agent 的前身）

### 4.1 是什么

**ReAct = Reason + Act（边思考边行动）**——模型不闷头答，而是"想一步 → 做一步 → 看结果 → 再想下一步"。

```
Thought: 我先想清楚要做什么
Action: 调哪个工具
Action Input: 传什么参数
Observation: 工具返回了什么
Answer: 最终答案
```

### 4.2 文本版 prompt（第一步：先让模型"假装"会调工具）

```python
def react_prompt(question, system_prompt="你是多种能力的小助手", tools="搜索引擎、计算器"):
    """ReAct 文本版：让模型按 Thought/Action/Observation/Answer 格式回答。
    注意：这里 Observation 是模型自己"编"的——它还没真工具可用。"""
    u = (f"可用工具:{tools}。请严格按格式回答：\n"
         f"Thought: 先根据用户问题想清楚要干什么\n"
         f"Action: 调用哪个工具更适合该问题\n"
         f"Action Input: 参数\n"
         f"Observation: 工具返回结果\n"
         f"Answer: 最终答案\n\n问题:{question}")
    return [{"role": "system", "content": system_prompt},
            {"role": "user", "content": u}]
```

**跑一次你会看到什么**——模型确实会按格式输出：

```fallback
  Thought: 用户问天气，需要调用天气查询工具
  Action: get_weather
  Action Input: {"city": "北京"}
  Observation: 北京今天晴，26℃          ← ⚠️ 它自己编的！没人真查
  Answer: 北京今天晴，26℃
```

**模型演了一出完整的工具调用戏——但 Observation 是它编的。** 它没有真的查天气，它"假装查了"。

> 🎯 **这就是 ReAct 文本版的天花板，也是下一篇的起点**：格式已经对了，但"Observation"必须是**真工具的真结果**——谁来执行 `Action`、谁来填真的 `Observation`？**答案就是 Function Calling**：模型输出 Action 意图，你的代码执行工具，把真结果回填。**下一篇《手撸 Function Calling》就是把 ReAct 里那句"假装查了"变成"真查了"。**

### 4.3 为什么说 ReAct 是 Agent 的前身

```fallback
  普通对话：  用户 → 模型 → 答案            （一次问答）
  ReAct：     用户 → 想一步 → 做一步 → 看结果 → 再想 → 答案   （循环）

  ReAct 第一次把"行动"和"观察"装进了回答格式里
  → 它差的就是"真能行动"——那就是 Agent
```

* * *

## 五、ToT：发散 → 评估 → 收敛

### 5.1 是什么

**ToT（Tree of Thoughts，思维之树）** 的核心是三步循环：

```fallback
  发散：生成多条路径（多个方案）
    ↓
  评估：给每条路径打分（优缺点/风险）
    ↓
  收敛：选出最优路径（并给出实施步骤）
```

> **类比**：不是只画一条路走到底，而是**先在纸上画好几条路，逐一评估，再选最好那条走**。

### 5.2 工程上两种做法

| | 单 prompt 版 | 多轮循环版（真 ToT） |
|---|---|---|
| 做法 | 让模型一口气走完发散→评估→收敛 | 分三次调用 API，每步一个专家 |
| API 调用 | 1 次 | 3 次 |
| 成本 | 低 | 高（约 3 倍） |
| 可控性 | 低（三步混在一次回答里） | **高**（每步可单独调 prompt、可干预） |
| 适合 | 快速出方案 | 关键决策 |

**选型看任务**：快速出方案用单 prompt，关键决策用多轮循环。

### 5.3 单 prompt 版

```python
def tot_prompt(system_prompt, question, n=3):
    """ToT 单调用版：把"发散→评估→收敛"全塞进一个问题里"""
    u = (f"请针对以下问题，提供{n}个不同的解决方案，"
         f"然后逐步分析每个解决方案的优缺点，最后给出最优方案，"
         f"并说明理由。\n\n问题：{question}")
    return [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": u},
    ]

# 使用：
msgs = tot_prompt("你是资深的 Agent 架构师，擅长多种角度对比方案", user_input)
response = get_response(msgs, max_tokens=1000)
```

**特点**：一次调用、三步混在一次回答里。步骤之间的"节奏"模型自己掌握，你插不上手。适合"随便给个像样的方案就行"的场景。

### 5.4 多轮循环版（真 ToT）

三个独立调用，每个一个"专家人设"：

```python
def ask(question, system_prompt="你是擅长多种角度对比方案的资深 Agent 架构师",
        max_tokens=500):
    """单次问答：组装 system + user 并调用"""
    msgs = [{"role": "system", "content": system_prompt},
            {"role": "user", "content": question}]
    return get_response(msgs, max_tokens=max_tokens)


def tot_solve(question, n=3):
    """真 ToT：三次调用，三个专家接力。每步的结果喂给下一步。"""
    # ① 发散：规划专家给 n 条思路
    schemes = ask(
        f"请针对以下问题给出{n}种不同解决思路，编号列出：\n{question}\n",
        system_prompt="你是擅长发散思维的规划专家",
        max_tokens=600,
    )
    # ② 评估：评审专家逐条打分（上一步的产出喂进来）
    scores = ask(
        f"请对以下 {n} 个方案逐一打分(1-10分),并各用一句话说明理由:\n{schemes}",
        system_prompt="你是严格的评审专家,打分要客观",
        max_tokens=500,
    )
    # ③ 收敛：决策专家选最优（评分喂进来）
    best = ask(
        f"综合以下评分,选出最优方案,并给出具体实施步骤:\n{scores}",
        system_prompt="你是决策专家,直接给最终选择",
        max_tokens=600,
    )
    return schemes, scores, best


# 使用：
s1, s2, s3 = tot_solve(user_input)
print("方案:", s1)
print("评审:", s2)
print("决策:", s3)
```

**看它的"数据流"**：`schemes` 是 ② 的输入，`scores` 是 ③ 的输入——**每一步的产出，是下一步的上下文**。这就是"多步 Agent 编排"的最早形态：

```fallback
  question ──▶ [规划专家] ──schemes──▶ [评审专家] ──scores──▶ [决策专家] ──▶ best
                发散                   评估                   收敛
```

> 💡 **每个"专家"其实只是换了个 system_prompt**——同一个模型，人设一变，就"扮演"不同角色。**角色分离（人设）是编排复杂任务的成本最低的手段。**

> ⚠️ **ToT 的 token 陷阱**：输出量大（三份长文本），最容易触 `max_tokens` 上限。判断信号是"**结尾突兀 / 结构缺块 / 方案数量不足**"。对策三件套：给足 token（`max_tokens=600`+）、拆多轮循环（每步独立控制）、约束格式。

* * *

## 六、🎯 为什么必须用代码路由，而不是手动粘贴

四种模式都有了，还差一个问题：**谁来决定"这题用哪个模式"？**

### 6.1 手动粘贴的局限

手动粘贴 prompt = 单次、人工、不可复用。你在聊天框里试出一个好 prompt，下次还得手动复制——**它活在对话里，不活在系统里**。

### 6.2 代码路由的三层价值

```fallback
  ① 自动化     系统自己判断并调用，不用每次人工选
  ② 执行动作   路由选中的不只是"一段文字"，而是一整套执行流程
     —— ReAct 会启动"调 API → 执行工具 → 结果回填"的循环
     —— ToT 是三次调用的编排
     —— 这是手动粘贴永远做不到的
  ③ 可观测    能打日志、能批量测试、能调策略
```

**一句话**：单次聊天靠 LLM 自动发挥就够了；**做 Agent 系统，必须用代码路由。**

### 6.3 模式注册表 + 路由函数

```python
# ── 模式注册表：名字 → prompt 构建函数 ──
BUILDERS = {
    "cot": cot_prompt,
    "few_shot": few_shot_prompt,
    "react": react_prompt,        # 文本版（Observation 还是编的，真工具见 FC 篇）
}


def auto_select(question: str) -> str:
    """让 LLM 判断这题适合哪种模式，返回模式名。
    规则写在 prompt 里；识别用关键词兜底（tot→react→few_shot→cot）。
    匹配顺序从长到短，避免 'cot' 误吞 'react'。"""
    content = (
        f"判断下面用户的问题适合哪种回答模式，输出：tot 或 react 或 few_shot 或 cot\n"
        f"规则:需要一步步推理/数学解题/公式推导的输出 cot;"
        f"固定模板化抽取信息的输出 few_shot;"
        f"需要调工具(天气/计算等)的输出 react;"
        f"开放性问题/多方案对比找最优的输出 tot\n"
        f"问题:{question}"
    )
    text = get_response([{"role": "user", "content": content}],
                        max_tokens=50).lower()
    for mode in ["tot", "react", "few_shot", "cot"]:   # 先长后短
        if mode in text:
            return mode
    print("[警告] 未识别到模式名,兜底 cot")
    return "cot"


def run_mode(mode: str, question: str, **kw):
    """按模式执行。tot 特判（要拼三段输出），其余走 BUILDERS。"""
    if mode == "tot":
        schemes, scores, best = tot_solve(question, n=kw.get("n", 3))
        return f"【方案】\n{schemes}\n\n【评审】\n{scores}\n\n【决策】\n{best}"
    if mode not in BUILDERS:
        raise ValueError(f"未知模式:{mode},可选 {list(BUILDERS) + ['tot']}")
    msgs = BUILDERS[mode](question, **kw)
    return get_response(msgs, **kw)


# ── 主循环：选模式 → 执行 → 输出（Agent 的第一版驾驶舱）──
while True:
    user_input = input("User: ")
    if user_input.lower() in ["exit", "quit"]:
        break
    mode = auto_select(user_input)
    print(f"[使用模式] {mode}")
    result = run_mode(mode, user_input)
    print("Agent导师:", result)
```

**看主循环那三行**——不再是"你选模式"，而是**模型看问题自己选**：

```fallback
  以前：你写死一种方式 → 调用
  现在：模型看问题 → 自己选 cot/few_shot/react/tot → 执行
```

> 🎯 **这就是"会选模式"的 Agent 雏形**。它和《手撸 Function Calling》里那套 `fc_loop` 合体后，就是能"自己选怎么答 + 真调工具"的完整 Agent——**下一篇收这个口**。

> ⚠️ **诚实提醒**：`auto_select` 用关键词兜底而非纯靠 LLM，是有意的。纯靠 LLM 选模式，选错一次整个回答就废了；**关键词兜底虽笨，但可预测、可调试**。先跑通规则，再放开给模型。

* * *

## 七、踩坑记

### 坑 1：把 Few-shot 示例堆进一段 user 话 ⚠️

- **现象**：给了示例模型还是不按格式走
- **原因**：API 只认 user/assistant 交替的 messages——示例得拆成"一问一答"的消息对
- **解法**：示例逐个 `user(问题) + assistant(答案)` 交替插入，真问题放最后

### 坑 2：CoT 的"一步步思考"放错层 ⚠️

- **现象**：想让全局按 CoT，结果只对单题生效（或反之）
- **原因**：指令放 `system` 是规则（全局），放 `user` 是任务（单题）
- **解法**：想清楚作用域再放。多数场景放 user 更干净——不至于让所有回答都啰嗦

### 坑 3：`get_response` 的"自动存回"副作用踩多轮 ⚠️

- **现象**：纯函数场景（如 auto_select 只问一句话）messages 被悄悄 append 了 assistant
- **原因**：`get_response` 把存回历史写死在函数里——多轮对话需要它，一次性问答不需要
- **解法**：要么拆出 `ask_once`（不存回）和 `ask_chat`（存回）两个版本；要么用参数 `save=True/False` 控制

### 坑 4：ToT 输出被 `max_tokens` 截断 🔴

- **现象**：方案列到一半没了 / 结尾突兀 / 结构缺块
- **原因**：ToT 输出量大，token 上限不够
- **解法**：给足 token（600+）、拆多轮循环（每步独立）、约束格式（编号列出）
- **自检信号**：结尾突兀 / 结构缺块 / 方案数量不足——三选一即中招

### 坑 5：ReAct 文本版把"编的 Observation"当真的 ⚠️

- **现象**：模型输出 `Observation: 北京晴 26℃`，你当真了
- **原因**：文本版没有真工具，Observation 是模型自己编的（幻觉的温床）
- **解法**：**记住 Observation 必须来自真工具执行**——这正是下一站 Function Calling 要解决的。在接真工具前，别把文本版 ReAct 用于任何需要真实数据的场景

### 坑 6：`auto_select` 匹配顺序反了 ⚠️

- **现象**：模型输出 "react_prompt" 却被匹配成 "cot"（`"cot" in "react_prompt"` → False？对，但 `"cot" in "few_shot_cot..."` 会误伤）
- **注意**：子串匹配的坑在于**短模式是长模式的子串**（如 `cot` ⊂ `scotland`）
- **解法**：**从长到短匹配**（`tot → react → few_shot → cot`），或要求模型输出纯模式名后做精确匹配

### 坑 7：react 返回字符串、其他模式返回消息 ⚠️

- **现象**：`run_mode` 里 react 特判，其他走 `BUILDERS`——接口不齐
- **原因**：react 直接进 `fc_loop` 拿最终答案，其余返回消息列表
- **解法**：演进过程中先跑通、后统一（把 react 也包成"返回字符串"的一致接口）。**接口不齐是演进的常态，别为了好看卡住进度**

* * *

## 八、速查卡片（复习直接看这）

```python
# ===== 三种模式一句话 =====
# CoT      要求一步步思考 → 适合数学/推理（本质：一条指令）
# Few-shot 给范例照葫芦画瓢 → 适合格式固定任务（本质：user/assistant 问答对）
# ReAct    想一步做一步看一步 → Agent 前身（Observation 要真工具，见 FC 篇）

# ===== 模式构建函数 =====
def cot_prompt(q, sys="你是逻辑缜密的推理助手"):
    return [{"role": "system", "content": sys},
            {"role": "user", "content": f"请一步步思考，再给出答案。\n问题:{q}"}]

def few_shot_prompt(q, examples=[("小明约了张医生", "时间:明天;人物:张医生")], sys="..."):
    msgs = [{"role": "system", "content": sys}]
    for question, answer in examples:
        msgs.append({"role": "user", "content": question})       # 一问
        msgs.append({"role": "assistant", "content": answer})     # 一答
    msgs.append({"role": "user", "content": q})                   # 真问题
    return msgs

def react_prompt(q, tools="搜索引擎、计算器"):
    u = (f"可用工具:{tools}。请严格按格式回答：\nThought:...\nAction:...\n"
         f"Action Input:...\nObservation:...\nAnswer:...\n\n问题:{q}")
    return [{"role": "system", "content": "你是多种能力的小助手"},
            {"role": "user", "content": u}]

# ===== ToT 两种做法 =====
# 单调用：tot_prompt 一发带走（发散→评估→收敛 混在一次回答）
# 多轮：  tot_solve 三连调（规划→评审→决策，每步产出喂下一步）→ 关键决策用这个

# ===== Prompt 及格线 =====
# 角色任务清晰 + 输入输出边界明确 + 可复用可测试（与技巧高级无关）
```

* * *

## 九、一句话总结

**Prompt 四模式不是四种"写法技巧"，而是四种"思考方式"**：CoT 让模型把推理摊开、Few-shot 让模型按范例模仿、ReAct 让模型边想边做、ToT 让模型多条路评估后择优。**及格的标准永远是"角色清晰 + 边界明确 + 可复用可测试"。**

**而你这一篇真正学会的是**：

```fallback
  ① 文字（prompt 内容）和代码（messages 组装）缺一不可
     —— API 只收带 role 的结构，Few-shot 的问答对必须代码拆
  ② 模式可以注册成表（BUILDERS），用路由（auto_select）选
     —— 代码路由 = 自动化 + 执行动作 + 可观测，手动粘贴做不到
  ③ ReAct 的 Observation 必须是真工具的真结果
     —— 文本版只是"演戏"，接真工具才是 Agent
```

**下一篇《手撸 Function Calling》**：把 ReAct 里那句"假装查了"变成"真查了"——模型输出 Action 意图，代码执行工具，把真 Observation 回填，再让模型基于真结果作答。到那时，你从"让模型按格式说话"（本篇）进到"让模型驱动真实世界"（下一篇）。

* * *
