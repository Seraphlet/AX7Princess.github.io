---
description: ""
title: "LangChain 总结"
draft: false
date: "2026-09-02T07:12:06+08:00"
slug: "LangChain-Summary"
categories:
 - LangChain
tags:
 - 
image: ""
---


# W4-D6 · 收官：装车

> **关键词**：闭环四步 / 链无状态·记忆有状态 / 双通道记忆 / REPL 驾驶舱 / 容错降级
> **前置**：W4-D1 LCEL、D2 Runnable 四件套、D3 记忆、D4 Retriever 转正、D5 `@tool` + `bind_tools`
> **涉及文件**：`W4/lc_agent_demo.py`、`W4/lc_tools_multi.py`、`W4/retriever.py`、`memory/memory_manager.py`
> **收尾预告**：W5 LangGraph——把今天的 `ask()` 四步拆成 node，把 `while` 拆成条件边，把 `MemoryManager` 变成 state

* * *

## 零、D6 是什么：从零件到整车

### 0.1 一句话

**把 D3（记忆）+ D4（检索）+ D5（工具）装进一个多轮 REPL 对话循环，跑通"记忆 → 模型 → 工具 → 回写记忆"的闭环，产出 `lc_agent_demo.py` + `README.md`。**

> **类比**：D1~D5 是零件——D5 是**发动机**（工具循环）、D3 是**油箱**（记忆）、D4 是**导航**（检索）、D2 是**传动轴**（Runnable）。D6 是**整车装配 + 驾驶舱**（`while True: input()` 的 REPL）。
> 你之前是"在车间里挨个测试零件"，今天是把车开上马路。

### 0.2 零件清单：你全都已经有了，只差组装

| 零件 | 来自哪一天 | 你的文件 |
|---|---|---|
| 三工具 + `tools_by_name` | D5 ✅ | `lc_tools_multi.py` |
| `MemoryManager`（短期+长期+压缩） | W3 / D3 ✅ | `memory/memory_manager.py` |
| `MemoryRetriever`（官方认证检索器） | D4 ✅ | `W4/retriever.py`（已被 `create_retriever_tool` 包成 `memory_tool`） |
| `_to_lc`（dict → BaseMessage） | D3 ✅ | 复用即可 |
| `_normalize_args` + 轮次上限 | D5 隐患修复 ✅ | 已讲过终版写法 |

**你今天不写新零件，只做两件事：① 把它们串起来；② 加一个 REPL 驾驶舱。**

### 0.3 装配图

```
┌──────────────────────────────────────────────────────────────────┐
│                     REPL 驾驶舱（while True）                      │
│                     input("用户：") → ask(q) → print              │
└───────────────────────────────┬──────────────────────────────────┘
                                ▼
┌──────────────────────────────────────────────────────────────────┐
│  ask(question) —— 四步闭环（🎯 今天的核心，面试就画这张图）         │
│                                                                    │
│   ① 取记忆   mm.get_context(query=question)                        │
│              └─ 短期窗口 + 长期召回 → dict 列表 → _to_lc 转消息     │
│                             │                                      │
│   ② 拼 prompt  prompt.format_messages(history=..., question=...)   │
│                             │                                      │
│   ③ 生成 + 工具循环  model.invoke() → _run_tools()                 │
│                             │    while ai.tool_calls:              │
│                             │      执行 → ToolMessage 回填 → 再问   │
│                             │                                      │
│   ④ 写回记忆  mm.add(user) + mm.add(assistant)                     │
└──────────────────────────────────────────────────────────────────┘
```

* * *

## 一、原理：闭环四步（面试核心图）

### 1.1 数据流

```fallback
用户输入
  ↓
① 取记忆：mm.get_context(query)          ← D3：短期窗口 + 长期召回，dict → BaseMessage
  ↓
② 拼 prompt：历史经 placeholder 展开 + 当前问题   ← D1 / D3
  ↓
③ 模型生成 + 工具循环：
     while ai.tool_calls → 执行 → ToolMessage 回填 → 再问   ← D5
  ↓
④ 写回记忆：mm.add(user) + mm.add(assistant)   ← D3：只有 ask() 能写回
  ↓
输出回答 → 回到 ①（下一轮）
```

### 1.2 🎯 关键设计：链无状态、记忆有状态

这是 D6 最值钱的一个认知，**面试必讲**。

**LangChain 的链是无状态的**——每次 `invoke` 都是一次干净的调用，链自己不记得上一轮说过什么。而**记忆是有状态的**——`MemoryManager` 里存着历史、窗口在滑动、长期事实在 Chroma 里躺着。

两者的接口对不上，所以：

```
  ┌─────────────────────────────────────────────────┐
  │  链（无状态）                                     │
  │  prompt | model | parser                         │
  │  每次 invoke 都是白纸一张，不带任何上轮痕迹        │
  └─────────────────────────────────────────────────┘
              ↕ 需要一道"人工搬运"
  ┌─────────────────────────────────────────────────┐
  │  记忆（有状态）                                   │
  │  MemoryManager：短期窗口 + 长期 Chroma + 压缩     │
  │  跨轮次持续存在                                   │
  └─────────────────────────────────────────────────┘
```

**"取记忆"和"写回记忆"必须在链外由 `ask()` 负责**——链自己不会去读记忆，也不会自动写回。这就是为什么 `ask()` 是今天的主角，而不是某条 LCEL 链。

> **类比**：链是**一次性餐具**，用完就扔，不记事；记忆是**私人饭卡**，余额一直跟着你。服务员（`ask()`）负责在每次点餐前刷一下卡（取记忆）、吃完后再刷一次（写回记忆）——餐具本身不会自己去刷卡。

### 1.3 🎯 这就是 W5 LangGraph 的预告

把这四步换个角度看：

| 今天（D6 的命令式写法） | W5（LangGraph 的声明式写法） |
|---|---|
| `ask()` 里的 ①②③④ 四步 | 四个 **node** |
| `while ai.tool_calls` 循环 | **条件边**（conditional edge） |
| `MemoryManager` | 图的 **state** |

**你今天手写的 `ask()`，本质就是一个还没被声明出来的状态图。** W5 要做的不是学新东西，是把这个你已经跑通的循环，用图的语法重新表达——node 是函数，edge 是流转条件，state 是贯穿全程的记忆。

> 💡 这个对照要记牢：**先能手写，才谈框架抽象。** 你如果没写过今天这个 `ask()`，直接看 LangGraph 只会觉得"为什么要画个图"。

* * *

## 二、实操 A：装配最小闭环（`W4/lc_agent_demo.py`）

下面按你本地文件的真实代码逐块拆。

### 2.1 路径与依赖（开场三行不能省）

```python
import os, sys
from pathlib import Path
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))  # 回到根目录，硬编码
from dotenv import load_dotenv
load_dotenv(Path(__file__).parent.parent / ".env", override=True)
```

三个动作：

- **① `sys.path.insert` 回根目录**——让 `from memory.memory_manager import MemoryManager` 找得到包。你 W3 调试时踩过同款坑，这里是标准解法。
- **② `load_dotenv(项目根 / ".env")`**——注意你用的是 `parent.parent`（根目录），和 `lc_tools_multi.py` 里的 `project_root = Path(__file__).parent.parent` **保持一致**。这个统一很关键：两份文件从不同目录启动，都指向同一个 `.env`。
- **③ `override=True`**——强制用项目 `.env` 覆盖可能已存在的系统环境变量，避免本地装过的别的 key 抢戏。

> ⚠️ `sys.path.insert` 是**学习期的权宜之计**。正式项目应该用 `pip install -e .` 或规范的包结构，别让每个文件开头都挂两行路径补丁。

### 2.2 零件初始化：模型 + 记忆 + prompt

```python
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
    temperature=0,
).bind_tools(tools, parallel_tool_calls=False)   # ← 注意这个参数

mm = MemoryManager(window=8, max_tokens=1500, keep=4)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是有记忆，能调工具的助手。历史对话和检索到用户事实、喜好和用户说记下的内容都在history里"),
    ("placeholder", "{history}"),      # 历史消息原地展开
    ("human", "{question}"),
])
```

**三个值得说的点**：

**① `parallel_tool_calls=False` 是你自己加的，加得对。**

这个参数告诉模型"一次只发一个工具调用"。DeepSeek 在并行工具调用上的兼容性和 OpenAI 不完全一致——模型同时发 3 个 `tool_calls` 时，回填协议更容易出问题（少回一条就 400）。**串行发、串行回，是 DeepSeek 上最稳的走法。**

**② `temperature=0` 在 Agent 场景是硬要求。**

工具调用要的是稳定决策——"调不调、调哪个、参数填什么"。发散的采样会让同一句问话这次调工具、下次不调，debug 时能把人逼疯。

**③ `("placeholder", "{history}")` 是 D3 的关键语法。**

它和 `MessagesPlaceholder("history")` 等价，但更简洁。作用是：把**消息对象列表**原地展开成一串真实的消息插进 prompt。

> ⚠️ **占位符要吃消息对象，不是 dict**。你 `MemoryManager` 返回的是 dict 列表，所以中间必须有 `_to_lc` 这道桥（见 2.3）。这条你在 D3 就踩过，D6 又遇到一次——**凡是接 dict 消息的，先想"prompt 要的是消息对象吗？要就过一道 `_to_lc`"**。

### 2.3 D3 的桥：`_to_lc`

```python
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "user":
        return AIMessage(content=content)      # ⚠️ 这一行有问题，见坑 5
    return HumanMessage(content=content)
```

设计意图很清楚：`MemoryManager` 吐出 `{"role": ..., "content": ...}` 的 dict，LangChain 的 prompt 要 `BaseMessage` 对象，这个函数负责翻译。

> ⚠️ 但你本地这版有个**角色写反的 bug**——`user` 映射成了 `AIMessage`。这会让你说的话被当成 AI 说的。**先往下看，坑 5 里详细定位和修法。**

### 2.4 工具循环 `_run_tools`（D5 逻辑 + 你的实战加固）

```python
MAX_ROUNDES = 3

def _normalize_args(args: dict) -> dict:
    """任意层嵌套都取第一个字符串值"""
    for k, v in list(args.items()):
        while isinstance(v, dict) and v:
            v = next(iter((v.values)))         # ⚠️ 漏了括号，见坑 6
        args[k] = v
    return args

def _run_tools(messages, verbose=True):
    """串行执行：一次只处理一个 tool_call，最稳的 DeepSeek 兼容方案"""
    round_ = 0
    ai = messages[-1]
    while ai.tool_calls and round_ < MAX_ROUNDES:
        round_ += 1
        if verbose:
            print(f"  [工具第{round_}轮] 模型发起 {len(ai.tool_calls)} 个调用:")
            for call in ai.tool_calls:
                print(f"    → {call['name']}({_normalize_args(call['args'])})")
        for call in ai.tool_calls:                        # 注意：不是 tool_calls[0]！
            args = _normalize_args(call.get("args", {}))
            try:
                result = tools_by_name[call['name']].invoke(args)
                content = str(result) if result is not None else "无结果"
            except Exception as e:
                content = f"工具调用失败: {e}"             # 失败也必须回填，否则协议违例
            messages.append(ToolMessage(content=content, tool_call_id=call['id']))
        ai = model.invoke(messages)                   # 模型看到结果后，自己决定还要不要发下一个
        messages.append(ai)
    return ai
```

**这段比 D5 的任务表示例多了四样东西，每样都是实战换来的**：

#### 加固 1：`try/except` 包住工具执行，失败也必须回填

```python
try:
    result = tools_by_name[call['name']].invoke(args)
    content = str(result) if result is not None else "无结果"
except Exception as e:
    content = f"工具调用失败: {e}"
messages.append(ToolMessage(content=content, tool_call_id=call['id']))
```

**这是这段代码里最值钱的一处，也是"400 错误"的根源解法。**

工具调用协议有条铁律：**模型发了几个 `tool_call`，你就必须回填几条 `ToolMessage`，`tool_call_id` 一一对应。** 少一条，下一轮 `model.invoke(messages)` 就会报 400——因为 messages 里有个 `tool_call` 悬着没有结果。

```
  模型: 我要调 get_weather（tool_call_id = abc123）
  你:   调失败了，跳过不回填 ❌
  下一轮 invoke(messages) → 400 Bad Request
        ↑ messages 里有个没有结果的 tool_call，协议不完整

  正确做法：
  你:   调失败，但照样回填 ToolMessage(content="工具调用失败: ...", id=abc123) ✅
  下一轮 invoke → 模型看到失败信息，自己决定换个说法或改调别的工具
```

> **类比**：服务员下单了 3 道菜，厨房第 2 道做不出来。你不能假装没下过这个单就上菜——客人（模型）会一直等着。**正确做法是把"这道菜没了"也端上桌**，让客人自己决定是换菜还是结账。
>
> 🎯 **设计原则：工具是增强项，不是可用性依赖。工具挂了，对话也得继续。**

#### 加固 2：`result is not None` 的兜底

`str(result) if result is not None else "无结果"`——有些工具（比如检索不到东西时）可能返回 `None`，直接 `str(None)` 会得到字符串 `"None"`，模型可能把它当成真内容。显式转成"无结果"，语义更干净。

#### 加固 3：`call.get("args", {})` 而不是 `call["args"]`

防止模型返回畸形消息时 KeyError 直接崩循环。`.get` 是**边界处的防御性写法**——外部输入（模型的输出就是外部输入）永远不要假设它格式完整。

#### 加固 4：`MAX_ROUNDES = 3` 给循环装刹车

任务表的裸 `while ai.tool_calls:` 是理想化写法。真实情况：模型可能陷入"调工具 → 结果不满意 → 再调"的循环，每轮都在烧 token。

> 🔴 **红线**：任何上生产的自动循环，必须有界。AI 系统的成本和失控风险，都藏在无界循环里。
>
> 💡 顺带一提：你变量名拼成了 `MAX_ROUNDES`（多了个 E）。两处引用都拼成一样所以能跑，但后面改名或复用时容易漏——**建议统一成 `MAX_ROUNDS`**。

### 2.5 🎯 核心 `ask()`：四步闭环

```python
def ask(question: str) -> str:
    history = [_to_lc(m) for m in mm.get_context(query=question)]        # ① 取记忆
    messages = prompt.format_messages(history=history, question=question)  # ② 拼 prompt
    ai = model.invoke(messages)
    messages.append(ai)
    ai = _run_tools(messages)                                            # ③ 生成 + 工具循环
    mm.add({"role": "user", "content": question})                        # ④ 写回记忆
    mm.add({"role": "assistant", "content": ai.content})
    return ai.content
```

**就这么 7 行，是整周的灵魂。** 逐行对照闭环四步：

| 行 | 对应 | 说明 |
|---|---|---|
| `history = [...]` | ① 取记忆 | `get_context(query=question)` 带 query 才能触发长期召回 |
| `format_messages(...)` | ② 拼 prompt | history 经 placeholder 展开，question 填 human 位 |
| `model.invoke` + `_run_tools` | ③ 生成 + 工具 | 工具循环可能跑 0~3 轮 |
| `mm.add` × 2 | ④ 写回 | user / assistant 各一条 |

**两个容易忽略的细节**：

**① `get_context(query=question)` 的 `query` 参数不能省。**

`MemoryManager` 的长期记忆是按 query 做向量召回的——**不传 query，长期通道就是瞎的**。你传了，所以"用户喜欢什么模型"这种问题才能召回 W3 存下的事实。

**② 写回顺序：先 user 后 assistant，且只写最终答案。**

`ai.content` 是工具循环跑完后的**最终回答**——中间的工具调用过程（ToolMessage）不进记忆。这是对的：记忆里该存"对话内容"，不是"调试日志"。

```
  记忆里存的：                        记忆里不存的：
  ├─ user: "北京天气怎么样？"          ├─ AIMessage(tool_calls=[get_weather])
  └─ assistant: "北京今天晴，26℃"      └─ ToolMessage(content="北京今天晴，26℃")
                                        ↑ 这些是过程，存在 messages 里就够了
```

> **类比**：`ask()` 就是一次**完整的呼吸**——吸气（取记忆）、憋气思考（生成 + 调工具）、呼气（回答）、然后把这个循环记进日记（写回）。少了任何一步，下一轮呼吸就接不上。

### 2.6 驾驶舱：多轮 REPL

```python
if __name__ == "__main__":
    print("=== W4-D6：带记忆 + 工具的多轮对话 demo ===")
    while True:
        question = input("\n用户：")
        if question in ["exit", "quit"]:
            break
        print("助手:", ask(question))
```

简单到没什么好讲的——**这正是好设计的标志**：复杂度全在 `ask()` 里，驾驶舱只负责 I/O。

> 💡 一个可优化点：`input()` 的结果建议 `.strip()` 一下。不然用户不小心多打了个空格，`" exit"` 就匹配不上退出条件了。

* * *

## 三、实操 B：双通道记忆——**你已经装好了**

### 3.1 好消息：这一步你不用写代码

你的 `memory_tool`（D4 的 `create_retriever_tool` 包装）**已经在 `tools` 列表里了**。所以"知识问答走检索"自动生效——模型看到"用户喜欢什么模型？"会自己决定调 `memory_retriever` 工具去查长期记忆。

**这就是 D4"转正"的复利**：当时多写的那个 `BaseRetriever` 子类，到 D6 直接省掉了手写适配。

### 3.2 🎯 双通道记忆（面试加分点）

| 通道 | 机制 | 管什么 | 触发方式 |
|---|---|---|---|
| **短期通道** | `mm.get_context` 每轮自动注入 history | 刚才聊了什么 | 每轮**无条件**注入 |
| **长期通道** | `memory_tool` → `MemoryRetriever` → Chroma | 以前说过的事实 | 模型**自主决定**调不调 |

```
  用户："我刚才说了什么？"
     │
     ├─ 短期通道：history 里已经有上一轮内容 ──▶ 直接答 ✅（无需调工具）
     │
  用户："用户喜欢什么模型，预算多少？"
     │
     ├─ 短期通道：history 里没有（这是几轮前/上次会话的事）
     ├─ 模型判断：得翻档案 ──▶ 调 memory_retriever ──▶ 长期通道命中 ✅
```

**分工一句话：闲聊走短期、事实问答走检索。**

> **类比**：短期通道是**聊天时的"刚才"**——你说话时脑子里还热乎着；长期通道是**档案室的"以前"**——得专门去翻。两者互补，缺了短期则每轮失忆，缺了长期则跨会话就断片。

### 3.3 自主性观察：模型会自己决定调用策略

实际跑起来你会看到有意思的现象——**模型理解工具能力后，会自己决定怎么调**：

- 同一个搜索需求，它可能**中英双语各搜一次**（自己判断单语结果不够全）
- 你问题里的拼写错误，它**自己去纠正后再搜**
- 它可能**连续调两个不同工具**，把结果拼起来回答

> ⚠️ 注意 `MAX_ROUNDES` 的意义在这里体现：模型的自主性越强，越可能跑多轮。没有上限，就是没有刹车。**自主性和成本，永远是一对需要权衡的矛盾。**

* * *

## 四、实操 C：README（`W4/README.md`）

```markdown
# W4 收官 Demo：带记忆 + 工具的对话 Agent

## 一句话
D1~D5 的零件装成的多轮对话闭环：记忆（短期+长期）→ 模型 → 三工具 → 回写记忆。

## 依赖
pip install langchain langchain-openai langchain-chroma python-dotenv tavily-python

## 环境变量（放项目根目录 .env，勿硬编码）
DEEPSEEK_API_KEY=sk-xxx
TAVILY_API_KEY=tvly-xxx

## 运行
python W4/lc_agent_demo.py

## 零件清单
- lc_tools_multi.py  → D5 三工具（天气/搜索/长期记忆）
- retriever.py       → D4 MemoryRetriever（官方认证检索器）
- memory/            → W3 记忆门面（短期窗口+长期Chroma+压缩）
```

> 🔴 **红线复述**：`.env` 必须写进 `.gitignore`。密钥一旦 push 上去就等于公开——第一时间去平台注销并换新的。这是你 Vibe Coding 手册里的老规矩，到 W4 依然有效。

* * *

## 五、验收清单（照着跑一遍）

| # | 输入 | 期望 | 验证什么 |
|---|---|---|---|
| 1 | `我叫小明，喜欢打篮球` | 回答确认 | 正常对话 |
| 2 | `我刚才说了什么？` | 答出"小明/打篮球" | **短期记忆生效**（D3） |
| 3 | `北京天气怎么样？` | 调 `get_weather` 并答出天气 | **工具生效**（D5） |
| 4 | `用户喜欢什么模型，预算多少？` | 调 `memory_retriever` 答出 DeepSeek / 2000 | **长期检索生效**（D4） |
| 5 | `exit` | 退出 | REPL 正常 |

**这五条的顺序是精心设计的**——从"能不能说话"到"短期记忆"到"工具"到"长期检索"，逐层验证，任何一条挂了都能精确定位是哪一层的问题：

```
  测试 1 挂 → 模型/密钥层（连不上）
  测试 2 挂 → _to_lc 或 get_context（记忆没取到 / 角色错了）← 坑 5 会挂在这里
  测试 3 挂 → bind_tools 或工具循环（D5 没通）
  测试 4 挂 → retriever 或 memory_tool（D4 没通 / docstring 没写清）
  测试 5 挂 → REPL 层（最不可能，但得验）
```

> 💡 **测试 2 和测试 4 挂了要分别对待**：测试 2 挂说明短期通道断了（查 `_to_lc` 和 `get_context`），测试 4 挂说明长期通道断了（查 `MemoryRetriever` 和 `create_retriever_tool`）。**双通道独立验证，才能定位准确。**

* * *

## 六、踩坑记

### 坑 1：工具失败不回填 → 400 Bad Request ⚠️

- **现象**：工具抛异常后，下一轮 `model.invoke(messages)` 报 400
- **原因**：messages 里有 `tool_call` 悬着没有对应 `ToolMessage`，协议不完整
- **解法**：`try/except` 包住执行，**失败也要回填** `ToolMessage(content="工具调用失败: ...")`
- 🎯 **原则**：工具是增强项不是可用性依赖，工具挂了对话得继续

### 坑 2：`parallel_tool_calls` 在 DeepSeek 上的兼容性 ⚠️

- **现象**：模型一次发多个 `tool_calls` 时，回填后报 400
- **解法**：`.bind_tools(tools, parallel_tool_calls=False)` 强制串行
- **权衡**：牺牲并行速度换稳定性。DeepSeek 上这个交换是划算的——工具调用本来就是 I/O 密集，串行那点延迟远小于一次 400 的代价

### 坑 3：长期召回失效——`get_context` 忘了传 query ⚠️

- **现象**：短期记忆正常，但长期事实怎么都召不回
- **原因**：`mm.get_context()` 不传 `query`，长期向量召回不会被触发
- **解法**：`mm.get_context(query=question)`——**query 参数是长期通道的开关**

### 坑 4：`placeholder` 吃消息对象，不是 dict ⚠️

- **现象**：`format_messages` 报类型错误，或历史消息不生效
- **原因**：`MemoryManager` 吐 dict，prompt 要 `BaseMessage`
- **解法**：中间过一道 `_to_lc`
- 💡 这条你在 D3 踩过、D6 又遇到——**凡是接 dict 消息的，先想"prompt 要的是消息对象吗"**

### 坑 5：🔴 `_to_lc` 里 user 和 assistant 写反了（你代码里的隐藏 bug）

你本地这版：

```python
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "user":
        return AIMessage(content=content)      # ❌ user 被映射成 AIMessage
    return HumanMessage(content=content)       # ❌ assistant 落到这里变成 HumanMessage
```

**后果**：**角色全反了。**

```
  实际发生的事：
  你说的 "我叫小明"      → AIMessage   → 模型以为是"自己说过的话"
  模型说的 "你好小明"    → HumanMessage → 模型以为是"用户说的话"

  多轮对话会怎么乱：
  第1轮 你：我叫小明        → [AI] 我叫小明
  第2轮 你：我叫什么？      → 模型看历史，发现"小明"是自己说的
                            → 它可能回答"我没说过你叫什么"
```

修法（改一个判断条件）：

```python
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "assistant":                    # ✅ 改成 assistant
        return AIMessage(content=content)
    return HumanMessage(content=content)       # ✅ user 落到这里
```

> ⚠️ **为什么这个 bug 难发现**：单轮对话几乎看不出来（历史是空的）。**只有跑到验收清单的测试 2（"我刚才说了什么？"）才会暴露**——这正是验收表存在的意义。

### 坑 6：🔴 `_normalize_args` 里 `v.values` 漏了括号（你代码里的第二个 bug）

你本地这版：

```python
while isinstance(v, dict) and v:
    v = next(iter((v.values)))      # ❌ 少了调用括号
```

`v.values` 是**方法对象本身**，不是调用结果。正确写法：

```python
while isinstance(v, dict) and v:
    v = next(iter(v.values()))      # ✅ 加括号调用
```

**后果**：一旦参数真的嵌套 dict，这里会抛

```fallback
TypeError: 'builtin_function_or_method' object is not iterable
```

**为什么它更隐蔽**：平时参数格式正常时 `while` 循环根本不进——**这个 bug 只在模型返回畸形参数时才炸**。也就是说，它平时装死，专等你最需要兜底的那一刻跳出来。

> 💡 **幂等性提醒**：修完这两个 bug 后，重跑验收清单的测试 2 和测试 4。坑 5 影响短期通道，坑 6 影响畸形参数兜底——**两个都在"异常路径"上，正常路径测不出来。**

### 坑 7：`MAX_ROUNDES` 拼写 ⚠️

- 拼成了 `MAX_ROUNDES`（多一个 E），两处引用一致所以能跑
- 建议统一改成 `MAX_ROUNDS`——改名时务必全局搜索，别只改定义处

* * *

## 七、W4 总盘点：一周知识点 + 语法表

> 记住一句话主线：**W4 = 用 LangChain 把"模型"变成"能干活的 Agent"。每天一个零件，D6 装车。**

### 7.1 知识点（按天）

| 天 | 知识点 | 面试一句话 |
|---|---|---|
| D1 | **LCEL 链**：`prompt \| model \| parser` 管道组合 | "LCEL 用 `\|` 把组件串成链，底层是 Runnable 协议" |
| D2 | **Runnable 四件套**：Passthrough（透传）/ Parallel（并行）/ Lambda（函数）/ Sequence（顺序） | "Runnable 是 LangChain 的统一接口：能 invoke、能组合、能并行" |
| D3 | **记忆**：消息历史、滑动窗口、压缩、dict→BaseMessage 桥 | "记忆让无状态链记住有状态的对话，窗口+压缩控制 token" |
| D4 | **Retriever**：向量检索、Chroma、embedding、`create_retriever_tool` | "Retriever 把知识库变成可查询的'外部大脑'，还能包成工具" |
| D5 | **Tools**：`@tool`、`bind_tools`、工具循环、参数归一化、轮次上限 | "模型不执行工具，只'点菜'；我们执行完把结果'上菜'回填" |
| D6 | **闭环装配**：REPL、取/写记忆、容错降级、双通道记忆 | "把零件装成能跑的 Agent demo，工具挂了对话不崩" |

### 7.2 语法清单（必须能手写）

```python
# 1. 模型绑定工具（D5）
model = ChatOpenAI(model=..., api_key=..., base_url=..., temperature=0) \
            .bind_tools(tools, parallel_tool_calls=False)   # D6 加固：DeepSeek 串行更稳

# 2. 工具声明（D5）
@tool
def get_weather(city: str) -> str:
    """查询天气。参数 city: 城市名。"""     # docstring 就是给模型看的说明书
    return f"{city} 今天晴, 26℃"

# 3. 工具查表 + 循环回填（D5 核心，D6 加固）
tools_by_name = {t.name: t for t in tools}
while ai.tool_calls and round_ < MAX_ROUNDS:
    for call in ai.tool_calls:
        try:
            result = tools_by_name[call["name"]].invoke(_normalize_args(call.get("args", {})))
            content = str(result) if result is not None else "无结果"
        except Exception as e:
            content = f"工具调用失败: {e}"      # 🎯 失败也必须回填
        messages.append(ToolMessage(content=content, tool_call_id=call["id"]))
    ai = model.invoke(messages)
    messages.append(ai)

# 4. prompt 带历史占位（D1/D3）
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是有记忆、能调工具的助手。"),
    ("placeholder", "{history}"),          # 吃消息对象列表
    ("human", "{question}"),
])

# 5. 消息桥（D3）—— 注意 assistant 的映射别写反
def _to_lc(m: dict):
    if m.get("role") == "system":    return SystemMessage(content=m["content"])
    if m.get("role") == "assistant": return AIMessage(content=m["content"])
    return HumanMessage(content=m["content"])

# 6. 参数归一化（D5 踩坑修复）—— 注意 v.values() 的括号
def _normalize_args(args: dict) -> dict:
    for k, v in list(args.items()):
        while isinstance(v, dict) and v:
            v = next(iter(v.values()))
        args[k] = v
    return args

# 7. 闭环四步（D6 灵魂）
def ask(question: str) -> str:
    history = [_to_lc(m) for m in mm.get_context(query=question)]      # ① 取（query 不能省）
    messages = prompt.format_messages(history=history, question=question)  # ② 拼
    ai = model.invoke(messages); messages.append(ai)
    ai = _run_tools(messages)                                          # ③ 工具循环
    mm.add({"role": "user", "content": question})                      # ④ 写回
    mm.add({"role": "assistant", "content": ai.content})
    return ai.content
```

* * *

## 八、默写框架：W4 收官自测

> **用法**：不看答案，照着这个骨架从头写一遍。写到 `pass` 的地方停下来默写，写不出的地方就是你的薄弱点。

```python
# ============ W4 收官自测：从零默写"带记忆 + 工具的 Agent" ============
import os, sys
from pathlib import Path
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))
from dotenv import load_dotenv
load_dotenv(Path(__file__).parent.parent / ".env", override=True)

from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage
from langchain_core.tools import tool
from langchain_core.retrievers import BaseRetriever
from langchain.tools.retriever import create_retriever_tool

# ① 模型（D5）—— 要写：bind_tools + parallel_tool_calls
model = ChatOpenAI(                      # ← 填空：model / api_key / base_url / temperature
    ...
).bind_tools(tools, parallel_tool_calls=False)

# ② 工具1：天气（D5）—— 要写：@tool + docstring
@tool
def get_weather(city: str) -> str:
    """..."""                            # ← 填空：给模型的说明书
    return f"{city} 今天晴, 26℃"

# ③ 工具2：搜索（D5 闭包工厂）—— 要写：内部 @tool
def make_web_search_tool(api_key: str):
    @tool
    def search_web(query: str, max_results: int = 3) -> str:
        """..."""
        pass                             # ← 填空：调用搜索 API
    return search_web

# ④ 工具3：记忆检索（D4）—— 要写：create_retriever_tool
def make_memory_tool(retriever: BaseRetriever):
    return create_retriever_tool(...)    # ← 填空：retriever / name / description

# ⑤ 查表（D5）
tools = [...]                            # ← 填空：三个工具
tools_by_name = {t.name: t for t in tools}

# ⑥ prompt（D1/D3）—— 要写：placeholder
prompt = ChatPromptTemplate.from_messages([...])

# ⑦ 消息桥（D3）—— 注意别把 assistant 写成 user
def _to_lc(m: dict):
    pass

# ⑧ 参数归一化（D5 踩坑）—— 注意 v.values() 括号
def _normalize_args(args: dict) -> dict:
    pass

# ⑨ 工具循环（D5 核心）—— while + 上限 + 失败也回填
MAX_ROUNDS = 3
def _run_tools(messages, verbose=True):
    pass

# ⑩ 核心 ask()（D6 收官）—— 四步闭环
def ask(question: str) -> str:
    pass

# ⑪ REPL 驾驶舱（D6）
if __name__ == "__main__":
    while True:
        q = input("\n用户：").strip()
        if q in ("exit", "quit"):
            break
        print("助手:", ask(q))
```

**自测打分标准**：

- ⑨⑩ 能默写 = **85 分**（工具循环 + 闭环是 Agent 的灵魂，面试必考）
- ①②⑤⑥⑦ 也能写 = **95 分**（LangChain 核心抽象全掌握）
- 全部写出来且能解释每行为什么 = **W4 真正收官**，放心进 W5

### 附：今日算法（呼应状态机）

对话闭环本质是**状态机**：输入 → 决策（记忆/检索/工具）→ 输出 → 回记忆。建议练 **LeetCode 207 课程表**（拓扑/状态依赖）或 **133 克隆图**（状态节点复制）。

* * *

## 九、一句话总结

**W4-D6 我做的不是学新东西，是把 D1~D5 的零件装成一条闭环：每轮 `ask()` 做四件事——取记忆（`get_context` 带长期召回，dict 转 BaseMessage 后经 placeholder 展开）、拼 prompt、生成 + 工具循环（`while ai.tool_calls` + `ToolMessage` 回填 + 轮次上限 + 参数归一化 + 失败也回填）、写回记忆。**

**而在这一整周里，LangChain 每一个标准件背后，都有一个你 W1~W3 手撸过的轮子**：

| 你自研的 | LangChain 标准件 |
|---|---|
| `RealLLM` / `llm_client` | `ChatOpenAI` |
| `prompt_lib` | `ChatPromptTemplate` |
| 手动取 `.content` / `json.loads` | `StrOutputParser` / `JsonOutputParser` |
| `fc_loop` 工具循环 | `bind_tools` + `ToolMessage` 回填 |
| `MemoryManager` | 外部记忆 + `_to_lc` 桥 |
| Chroma 长期记忆 | `MemoryRetriever` + `create_retriever_tool` |
| `tools/` 注册表 | `@tool` + `tools_by_name` |

**框架化不是推翻重来，是给旧轮子换上标准轴距。** 你踩过的每个坑（密钥、路径、参数畸形、角色转换），框架都没有替你免掉——它只是把这些坑搬到了一个大家都认的位置上。

> 🎯 **最关键的认知**：**链无状态、记忆有状态**——所以取/写记忆必须在 `ask()` 外部完成。等你进 W5 会发现，`ask()` 的四个步骤就是四个 node，`while` 就是条件边，`MemoryManager` 就是 state。**你今天手写的，是一张还没被画出来的图。**

**下一篇：W5 LangGraph**——把 `ask()` 拆成 node、把 `while` 拆成条件边、把 `MemoryManager` 升级成 state，用状态图声明式地重写今天这个循环。到那时你会感谢 D6 老老实实手写了这一遍。

* * *
