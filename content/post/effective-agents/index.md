---
description: "文章来源：https://www.anthropic.com/engineering/building-effective-agents"
title: "WorkFlow and agents"
draft: false
date: "2026-08-26T00:23:57+08:00"
slug: "Effective-agents"
categories:
 - Agent
tags:
 - 
image: ""
---

# 如何构建高效的 Agent —— 基于 Anthropic《Building Effective Agents》的学习笔记

> 原文：*Building effective agents*（Anthropic 工程博客，2024-12-19，作者 Erik S. / Barry Zhang）
> 笔记定位：把官方"agent 设计圣经"拆成可对照、可落地的工程笔记，并在末尾用文中框架回答一个高频问题——**怎么区分 chatbot / workflow / agent / multi-agent**。
> 配套：你已写过的 `LangChain教学笔记` / `LangGraph教学笔记`，本文是它们的"设计哲学上游"。

---

## 目录

- [一、核心经验：简单优先](#一核心经验简单优先)
- [二、agentic systems 的两大分支：Workflow vs Agent](#二agentic-systems-的两大分支workflow-vs-agent)
- [三、什么时候该用 / 不该用 Agent](#三什么时候该用--不该用-agent)
- [四、构建块：Augmented LLM（增强型 LLM）](#四构建块augmented-llm增强型-llm)
- [五、五种 Workflow 模式（含示例）](#五五种-workflow-模式含示例)
- [六、自主 Agent 模式](#六自主-agent-模式)
- [七、工具设计与 ACI 原则](#七工具设计与-aci-原则)
- [八、三大设计原则 + 框架使用建议](#八三大设计原则--框架使用建议)
- [九、如何区分 chatbot / workflow / agent / multi-agent](#九如何区分-chatbot--workflow--agent--multi-agent)
- [十、小结与对照表](#十小结与对照表)

---

## 一、核心经验：简单优先

Anthropic 团队在大量客户落地后，得出一句反直觉的结论：

> "Consistently, the most successful implementations weren't using complex frameworks or specialized libraries. Instead, they were building with simple, composable patterns."
> （最成功的实现，几乎都没用复杂框架或专用库，而是用**简单、可组合的模式**搭出来的。）

> "Success in the LLM space isn't about building the most sophisticated system. It's about building the *right* system for your needs. Start with simple prompts, optimize them with comprehensive evaluation, and add multi-step agentic systems only when simpler solutions fall short."
> （成功不在于最复杂，在于"刚好够用"。先写简单 prompt，用评测把它打磨好；只有简单方案不够用时，才加多步 agentic 系统。）

**知识点**：
- 这是全文的"总纲"。它直接反对"一上来就上框架、上多 Agent"的冲动。
- 和你 W1~W3 手写轮子、W4 才上 LangChain/LangGraph 的路线**完全一致**——先理解本质，再借框架标准化。

⚠️ **坑**：很多人把"agent"当成默认选项。原文明确说，很多任务一个 LLM 调用 + 检索 + 几个上下文示例就解决了，根本不需要 agent。

---

## 二、agentic systems 的两大分支：Workflow vs Agent

这是全文最重要的一个二分法，也是后面所有模式的划分依据。

### Workflow —— "代码定死的路径"

> "Workflows are systems where LLMs and tools are orchestrated through **predefined code paths**."
> （workflow = LLM 和工具被**预定义的代码路径**编排。）

- 流程是**你写死的**：先调谁、后调谁、条件怎么走，都在代码里固定。
- 模型在里面的角色是"被安排的执行者"，不是"决策者"。

### Agent —— "模型自己掌舵"

> "Agents, on the other hand, are systems where LLMs **dynamically direct their own processes and tool usage**, maintaining control over how they accomplish tasks."
> （agent = LLM **动态主导自己的流程和工具调用**，自己掌控怎么完成任务。）

- 流程是**模型运行时决定的**：它自己判断要不要调工具、调哪个、调几次、下一步干嘛。
- 适用于"无法预判要走几步、无法写死路径"的开放性问题。

### 一句话对照

| 维度 | Workflow | Agent |
|------|----------|-------|
| 谁决定流程 | 你（代码写死） | 模型（运行时动态） |
| 步数 | 可预判、固定 | 不可预判、动态 |
| 模型角色 | 被编排的执行者 | 自主决策者 |
| 可预测性 | 高 | 低 |
| 成本/延迟 | 可控 | 偏高（自主换来的） |

> 注：原文把"完全自主"和"走预定义工作流的规范实现"都归在 **agentic systems（智能体系统）** 这个大词下。所以平时口语说的"agent"可能指两者之一，工程上要分清。

---

## 三、什么时候该用 / 不该用 Agent

### 总原则

> "When building applications with LLMs, we recommend finding the **simplest solution possible**, and only increasing complexity when needed. This might mean **not building agentic systems at all**."
> （找最简单的方案；可能根本不建 agentic 系统。）

- **单 LLM 调用**就能解决的：直接做。
- **Workflow**：任务定义良好，要可预测、一致性高 → 用 workflow。
- **Agent**：需要大规模灵活性、模型自主决策 → 用 agent。

### Agent 的专用适用条件

> "Agents can be used for **open-ended problems** where it's difficult or impossible to predict the required number of steps, and where you **can't hardcode a fixed path**."

- 模型可能跑很多轮（many turns）。
- 你必须**信任它的决策**（autonomy 带来的代价）。
- 风险：更高成本 + **错误复利（compounding errors）**——一步错，后面跟着错。
- 建议：在**沙盒**广泛测试，加适当 guardrails（护栏）。

⚠️ **坑**：agent 用"延迟 + 成本"换"任务表现"。上 agent 前先问自己：这个 tradeoff（权衡）真的值吗？你的任务能不能写死步骤？

---

## 四、构建块：Augmented LLM（增强型 LLM）

> "The basic building block of agentic systems is an **LLM enhanced with augmentations** such as retrieval, tools, and memory."
> （agentic 系统的基本构件 = 一个**被增强了的 LLM**：检索、工具、记忆。）

```
        ┌─────────────────────────────┐
        │       Augmented LLM          │
        │  ┌────────┐  ┌─────────────┐ │
        │  │  model │  │ retrieval   │ │
        │  └────────┘  │ tools       │ │  ← 模型可主动调用
        │              │ memory      │ │
        │              └─────────────┘ │
        └─────────────────────────────┘
```

**知识点**：
- 这就是你 W4 学的那套：Model（加工机）+ Prompt + OutputParser + Retriever（档案室）+ Memory（记忆）+ Tool（工具箱）。Anthropic 把它们统一叫"增强型 LLM"。
- 模型不再只是"生成文本"，它能**主动生成搜索查询、选工具、决定保留哪些信息**。
- 第三方工具推荐用 [Model Context Protocol (MCP)](https://www.anthropic.com/news/model-context-protocol) 接入——这是统一工具接口协议，和你 W2 `@tool` 注册表是同一类思路的标准化版。

---

## 五、五种 Workflow 模式（含示例）

下面 5 种都是**你写死流程**的模式，按复杂度递增。每种都有"何时用 + 示例 + 最小示意"。

### 1. Prompt chaining（提示链）

> "decomposes a task into a sequence of steps, where each LLM call processes the output of the previous one."
> （把任务拆成一串步骤，每个 LLM 调用处理上一个的输出。）

- **何时用**：任务能干净拆成固定子任务，用一点延迟换更高准确度。
- **可加 gate**：在步骤间加程序化检查，不过关就截断。
- **示例**：生成营销文案 → 翻译成另一语言；写大纲 → 检查标准 → 按大纲写正文。

```python
# Prompt chaining 最小示意：先写，再译
def chain_write_then_translate(topic: str) -> str:
    draft = llm(prompt_write, topic)          # ① 写中文文案
    if not passes_gate(draft):                # 可选 gate：质量不过就停
        return "草稿未达标"
    translated = llm(prompt_translate, draft) # ② 把草稿译文
    return translated                          # 串起来，固定两步
```

### 2. Routing（路由）

> "classifies an input and directs it to a **specialized followup task**."
> （把输入分类，导向专门的后续任务。）

- **何时用**：任务有不同类别、必须分开处理，且分类能准确完成。
- **示例**：客服查询分入 通用 / 退款 / 技术支持；简单问题 → Haiku，难题 → Sonnet。

```python
def route(query: str):
    kind = llm_classify(query)          # 先分类
    if kind == "refund":   return refund_flow(query)    # 走退款专用流
    if kind == "tech":     return tech_flow(query)      # 走技术支持流
    return general_flow(query)                          # 走通用流
# 路径是"分类结果 → 固定分支"，写死的
```

### 3. Parallelization（并行化）

两种变体：
- **Sectioning**：独立子任务并行跑。
- **Voting**：同一任务跑多次拿多样结果。

> "effective when the divided subtasks can be parallelized for speed, or when multiple perspectives or attempts are needed for higher confidence."
> （子任务能并行提速，或需要多视角/多次尝试提置信度时用。）

- **示例**：一个模型答用户、另一个并行做不当内容筛查；多提示审代码漏洞（voting）；多提示评估内容是否不当以平衡误报漏报。

```python
import concurrent.futures
def parallel_guard(user_q: str):
    with concurrent.futures.ThreadPoolExecutor() as ex:
        ans  = ex.submit(llm_answer, user_q)     # 分支A：回答
        safe = ex.submit(llm_safety_check, user_q)  # 分支B：并行筛查
    return ans.result() if safe.result().ok else "被拦截"
```

### 4. Orchestrator-workers（编排者 - 工作者）

> "A central LLM **dynamically breaks down tasks**, delegates them to worker LLMs, and synthesizes their results."
> （一个中枢 LLM **动态拆任务**，派给 worker，再综合结果。）

- **和并行化的区别**：子任务**不是预定义**的，由编排者按输入**动态决定**。
- **何时用**：无法预判子任务（如改几个文件、搜几个源都不知道）。
- **示例**：跨多文件的复杂代码改动；多源信息搜索分析。
- 这其实就是 **multi-agent** 的一种标准形态（见第九节）。

```python
def orchestrator_workers(task: str):
    plan = llm_plan(task)                 # 中枢动态拆：返回 ["改a.py","改b.py",...]
    results = [worker(p) for p in plan]   # 派给 workers（可并行）
    return llm_synthesize(results)        # 综合
```

### 5. Evaluator-optimizer（评估者 - 优化者）

> "One LLM call generates a response while **another provides evaluation and feedback in a loop**."
> （一个生成，另一个在循环里评估 + 给反馈。）

- **何时用**：有明确评估标准、迭代细化有价值、LLM 能凭反馈改善。
- **示例**：文学翻译里 evaluator 给批评；复杂搜索多轮，evaluator 决定还要不要再搜。

```python
def evaluator_optimizer(task: str, rounds=3):
    resp = llm_generate(task)
    for _ in range(rounds):
        feedback = llm_evaluate(task, resp)   # 另一模型评估
        if feedback.good_enough: break
        resp = llm_generate(task, feedback)   # 带反馈重生成
    return resp
```

---

## 六、自主 Agent 模式

> 起点：用户命令或交互讨论 → 规划并**独立运行** → 可回头找人类要信息/判断。
> 关键：每步从环境拿 **"ground truth"**（工具结果 / 代码执行）来评估进展。

- **示例（Anthropic 自家）**：
  - 编码 Agent 解 [SWE-bench](https://www.anthropic.com/research/swe-bench-sonnet) 真实 GitHub issue。
  - ["computer use" 参考实现](https://github.com/anthropics/anthropic-quickstarts/tree/main/computer-use-demo)：Claude 直接操作电脑。

```python
# 自主 Agent 的骨架（概念性）：模型自循环，直到自己觉得完成
def autonomous_agent(goal: str):
    state = init(goal)
    while not done(state):
        action = llm_decide(state)        # 模型自己决定下一步
        if action.is_tool_call:
            state = run_tool(action)      # 调工具，拿 ground truth
        elif action.need_human:
            state = ask_human(action)     # 必要时找人
        else:
            return action.final_answer    # 自己判断完成
# 注意：循环边界、何时停，都是模型 runtime 决定的
```

> 你 W2 的 `fc_loop` 就是这种 agent 的雏形：模型决定调哪个工具 → 执行 → 结果喂回 → 再决定。LangChain 的 `AgentExecutor` / LangGraph 的 `StateGraph` 把它标准化了（见你的 `LangGraph教学笔记`）。

---

## 七、工具设计与 ACI 原则

工具让模型通过 API 规范调用外部服务。工具定义要像写 prompt 一样精心。

> "Think about how much effort goes into human-computer interfaces (HCI), and plan to invest **just as much effort** in creating good **agent-computer interfaces (ACI)**."
> （你在人机界面 HCI 上花多少精力，就该在**智能体-计算机界面 ACI** 上花同样多。）

**ACI 要点**：
- 给模型足够 token "思考"，别把角落写死。
- 工具格式贴近互联网自然文本，无格式开销（别让它数 JSON 引号）。
- 配用法示例、边界清晰；改参数名像写 docstring；workbench 测试；**poka-yoke（防错）**。
- 案例：SWE-bench 工具把**相对路径改成绝对路径**，避免模型用错路径。

⚠️ **坑**：工具接口烂 = 模型反复踩坑 = 错误复利。ACI 投入和模型能力一样重要。

---

## 八、三大设计原则 + 框架使用建议

### 三大原则

1. **Simplicity（简单）**：保持设计简单，复杂度只在"实测能改善"时才加。
2. **Transparency（透明）**：显式展示规划步骤，让人能看懂模型在干嘛（呼应你手册里"操作透明日志"那条）。
3. **ACI（好工具界面）**：工具要精心设计 + 充分测试。

### 框架建议

> "We suggest that developers **start by using LLM APIs directly**: many patterns can be implemented in a few lines of code."
> （建议先用 LLM API 直接写，很多模式几行代码就能实现。）

- 框架（Claude Agent SDK、LangGraph、AWS Strands、Rivet、Vellum）易上手，但**加了抽象层难调试**。
- 用框架前必须懂底层代码。
- 这些构建块不是死规矩，是可**塑形、组合**的常见模式；成功的关键是**评测 + 迭代**。

> 这和你 W1~W3 手写 `llm_client` / `fc_loop` / `MemoryManager` 再上框架的理念同源：先懂本质，框架才不是黑盒。

---

## 九、如何区分 chatbot / workflow / agent / multi-agent

这是你专门问的。下面用**上文的完整框架**给四个概念下可操作的定义，并给"一眼识别"的判断树。

### 1. Chatbot（聊天机器人）—— 非 agentic 的最简形态

- **定义**：基于单个（或简单多轮）LLM 调用的对话界面，**没有动态工具编排、没有自循环**。
- **本质**：就是第一节说的"单 LLM 调用 + 检索 + 上下文示例"的最简应用。模型只在"被问→答"之间往返，不主动决定调工具、不自我循环。
- **特征**：
  - 无 function calling / 或只有极简单工具
  - 流程固定：输入 → 输出
  - 不维护跨多步的"自主状态机"
- **对照**：它**不在** agentic systems 两大分支里，是比 workflow 还简单的一层。

```python
# Chatbot：一次调用，完事
def chatbot(user_msg, history):
    return llm(chat_prompt, history + user_msg)   # 说完就完，不决定下一步
```

### 2. Workflow（工作流）—— 代码写死路径

- **定义**：见第二节。"LLM 和工具被预定义代码路径编排"。
- **本质**：你是导演，模型是演员，台词和走位都排好了。
- **识别**：看到 `prompt chaining / routing / parallelization / orchestrator / evaluator-optimizer` 任一模式，且流程是**写死的** → 就是 workflow。
- **关键判据**：**步数和分支是你能预判的**，模型不动态决定"还要不要继续"。

### 3. Agent（智能体）—— 模型自己掌舵

- **定义**：见第二节。"模型动态主导自己的流程和工具使用。"
- **本质**：你是给目标的人，模型是自己找路到达的司机。
- **识别**：
  - 模型**运行时决定**调不调工具、调哪个、调几次
  - 有一个**自循环**（while not done），终止由模型判断
  - 步数不可预判
- **关键判据**：你能信任它"自己决定怎么完成"，并接受更高成本 / 错误复利。

### 4. Multi-agent（多智能体）—— 多个 agent 协作

- **定义**：多个 agent（或"一个编排 agent + 多个 worker agent"）协同解决一个任务。
- **Anthropic 的官方姿态**：文中**最推崇的形态是 Orchestrator-workers**（一个中枢动态拆任务 + 多个 worker 执行）；并明确提醒——多 agent **peer-to-peer 直接互相协作**往往"用性能换了一点收益"，多数场景不如 orchestrator-workers。
- **识别**：
  - 有 ≥2 个"会自主决策"的角色
  - 它们之间有**任务委派 / 信息传递**关系
  - 典型结构：Supervisor（编排者）↔ Workers（工作者）

```python
# Multi-agent 的标准形态 = Orchestrator-workers
def multi_agent(task):
    plan = supervisor(task)              # agent A：动态拆
    parts = [worker(p) for p in plan]    # agent B/C/D：各自执行（可并行）
    return supervisor.synthesize(parts)  # agent A：综合
```

### 一眼识别判断树

```
用户提问/任务进来
   │
   ├─ 一次 LLM 调用就能答，无工具自循环？
   │     └─ YES → Chatbot
   │
   ├─ 流程写死（步数/分支可预判）？
   │     └─ YES → Workflow（5 模式之一）
   │
   ├─ 模型运行时自决流程、有自循环、步数不可预判？
   │     └─ YES，只有 1 个决策者 → Agent
   │
   └─ 有 ≥2 个自主决策者、彼此委派任务？
         └─ YES → Multi-agent（优先 Orchestrator-workers）
```

**对照表（四者横评）**

| 维度 | Chatbot | Workflow | Agent | Multi-agent |
|------|---------|----------|-------|-------------|
| 在 agentic 体系里？ | 否（最简层） | 是（分支一） | 是（分支二） | 是（分支二扩展） |
| 谁定流程 | 无（固定） | 你写死 | 模型动态 | 多个模型分别动态 |
| 自循环 | 无 | 无（或 gate 固定） | 有（模型判停） | 有（多个 agent 各自/协同） |
| 工具调用 | 极少/无 | 有，按预定顺序 | 有，模型自选 | 有，分布到各 agent |
| 步数可预判 | 是 | 是 | 否 | 否 |
| 典型代表 | 客服问答壳 | 5 种 workflow 模式 | `fc_loop` / computer use | orchestrator-workers |
| 你代码里的对应 | 单 `llm.chat` | LCEL 链 / `RunnableBranch` | W2 `fc_loop` / LangGraph 自环 | W5 `StateGraph` + ToolNode 多角色 |

---

## 十、小结与对照表

**全文精华三句**：
1. 最简单的方案往往最好——先写 prompt，评测打磨，**不够才上 agentic**。
2. Workflow（你写死）和 Agent（模型掌舵）是 agentic 系统的两根柱子，区分只在"谁定流程"。
3. 框架不是答案，**简单可组合的模式 + 好 ACI + 充分评测**才是。

**和你的学习路线对接**：

| 本文概念 | 你已写的代码 / 笔记 |
|----------|---------------------|
| Augmented LLM | W4 Model+Prompt+Parser+Retriever+Memory+Tool |
| Workflow（5 模式） | LCEL 链、`RunnableParallel`/`Branch` |
| Agent | W2 `fc_loop`、LangGraph 自环 `StateGraph` |
| Multi-agent | LangGraph `orchestrator-workers` / ToolNode |
| 工具设计 ACI | 你 W2 `@tool` 注册表、手册"密钥红线" |
| 简单优先 + 透明 | 你手册"阶段零 操作透明日志" |

> 一句话收尾：**别把 agent 当默认值。** 能 chatbot 解决的别上 workflow，能 workflow 的别上 agent，能单 agent 的别上 multi-agent。每升一级，都用"评测证明值得"来买单。

---

## 参考原文关键句（保真）

- *"the most successful implementations weren't using complex frameworks... simple, composable patterns"*
- *"Workflows are systems where LLMs and tools are orchestrated through predefined code paths."*
- *"Agents... LLMs dynamically direct their own processes and tool usage"*
- *"Agents can be used for open-ended problems where it's difficult or impossible to predict the required number of steps"*
- *"invest just as much effort in creating good agent-computer interfaces (ACI)"*
- *"start by using LLM APIs directly: many patterns can be implemented in a few lines of code"*
