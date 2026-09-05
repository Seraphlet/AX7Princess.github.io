---
description: ""
title: "Function Calling"
draft: false
date: "2026-08-10T02:58:11+08:00"
slug: "Function Calling"
categories:
 - null
tags:
 - null
image: ""
---


# W2 · Function Calling 演进记：从最小闭环到模型路由

> **关键词**：`tool_calls` / `fc_loop` / 工具分发 / MockLLM / 模型路由 / `tool_records`
> **前置**：W1（`llm_client`、Prompt 四模式）
> **本篇内容**：FC 闭环五版演进，从"能跑"到"能测"到"能防幻觉"
> **收尾预告**：W3 MemoryManager——让这辆能调工具的车，拥有记忆

* * *

## 零、先说清楚：Function Calling 到底在解决什么

### 0.1 一个扎心的事实

**大模型不会执行工具。** 你让它查天气，它只会**编**一个天气给你——因为它压根没法联网。

但模型有一个能力：**它能读懂"有哪些工具可用、各要什么参数"，然后输出一句"我要调 `get_weather`，参数是 `{"city": "北京"}`"**——注意，它只是**说要调**，不会真的调。

```fallback
  模型的真实输出（不是答案，是"意图"）：
  {
    "name": "get_weather",
    "arguments": "{\"city\": \"北京\"}"
  }
  ← 这是一张"点菜单"，不是"菜"
```

**Function Calling（FC）= 你在模型和真实世界之间，当那个"接单的人"。**

### 0.2 闭环四步（全文的总纲）

```fallback
  ① 函数注册  告诉模型："我有这些工具、各要什么参数"（tools 列表）
        │
        ▼
  ② 模型决定  输出"我要调 get_weather，参数 city=北京"（不是直接回答）
        │
        ▼
  ③ 代码执行  解析函数名和参数，真正调用 get_weather("北京")
        │
        ▼
  ④ 结果回填  把返回值作为 tool 消息喂回模型 → 模型基于真实结果回答
        │              ↑ 如果还需要别的信息，重复 ②③④
        └──────────────┘ 直到模型直接给最终答案
```

> **类比**：模型是**客人**，只会看菜单点菜，不会进厨房。FC 闭环里：
> - ① 菜单（tools 注册）
> - ② 客人下单（`tool_calls`）
> - ③ 厨房做菜（你的代码执行工具）
> - ④ 上菜给客人看（结果回填），客人尝过再决定要不要加点别的

**本文就干一件事：把这个闭环从零手写出来，并让它一步步变皮实。** 五版进化，每一版解决上一版的一个痛点。

* * *

## 一、版本 1：最小可跑闭环（先让循环转起来）

### 1.1 FC 循环的"骨架清单"

写任何 FC 循环前，先在心里过这五条：

```fallback
  ① 开头：messages = [{"role": "user", ...}]     ← 只有一条用户消息
  ② 循环：for _ in range(MAX_ROUNDS):            ← 最多转几圈（防死循环）
  ③ 判断：if not msg.tool_calls: return           ← 模型没要工具 → 结束
  ④ 有工具：append assistant(带tool_calls)
           + 执行工具 + append tool 消息
  ⑤ 结果：模型看到 tool 结果再回答 → 回到 ③
```

**这五条就是 `fc_loop` 的全部。** 后面所有版本都是在这五条上做加固，骨架从没变过。

### 1.2 最小代码（逐行注释）

```python
import json
import os
from openai import OpenAI


# ── 工具本体：真正干活的函数 ──
def get_weather(city: str) -> str:
    """天气实际查询函数（版本1先用假数据，把链路跑通再说）"""
    return f"{city}今天台风，注意安全"


# ── ① 函数注册：手写 tools schema，告诉模型有哪些工具 ──
# 这段 JSON 就是"菜单"：模型靠它知道 get_weather 要一个 city 参数
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市天气",          # 模型靠这句决定何时调用
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"}
                },
                "required": ["city"],
            },
        },
    }
]

messages = [{"role": "user", "content": "今天北京天气怎么样"}]

client = OpenAI(
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",       # DeepSeek 走 OpenAI 兼容协议
)

# ── ② 模型决定：带 tools 参数发起请求 ──
res = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=messages,
    tools=tools,          # ← 关键：不传 tools，模型永远不会"点菜"
    max_tokens=100,
)
msg = res.choices[0].message

# ── ③ 判断 + 执行 + ④ 回填 ──
if msg.tool_calls:
    # 把模型这条"带点菜意图"的消息原样追加进历史
    messages.append(msg)
    for tc in msg.tool_calls:
        # tc.function.arguments 是 JSON 字符串，要解析成 dict 再展开传参
        result = get_weather(**json.loads(tc.function.arguments))
        messages.append({
            "role": "tool",
            "tool_call_id": tc.id,     # ← 必须和模型给的 id 一一对应
            "content": result,
        })
    # 带着工具结果再问一次模型，这次它该给最终答案了
    final = client.chat.completions.create(
        model="deepseek-v4-flash", messages=messages, tools=tools
    )
    print(final.choices[0].message.content)
```

### 1.3 两个必须记牢的协议细节

**① `tool_call_id` 必须一一对应。**

模型发了几个 `tool_calls`，你就必须回填几条 `tool` 消息，每条带着模型的 `id`。**少一条或 id 对不上，下一轮请求就报 400**——这是协议，不是建议。

**② 模型的那条 `tool_calls` 消息必须原样 append 进 messages。**

```python
messages.append(msg)   # msg 是带 tool_calls 的 assistant 消息，不能丢
```

如果不把它加回去，模型下一轮就"失忆"了——它不知道自己刚才点过菜。

> ⚠️ **这个最小版有个明显的短板：只能处理一轮工具调用。** 如果模型第一次要查两个城市、或者查完还要算数，这个 `if` 结构就转不动了。版本 2 解决它。

* * *

## 二、版本 2：真实 API + 循环 + 工具分发（让循环自己转）

### 2.1 三个升级

```fallback
  版本 1                    版本 2
  ───────                  ───────
  if 结构，单轮            for 循环，最多 max_runs 轮
  get_weather 假数据        wttr.in 真实天气 API
  执行写死在 if 里           run_status 统一分发（名字 → 函数）
  （无出错保护）             出错不崩，把错误喂回给模型自纠
```

**版本 2 引进了 FC 的第二个核心抽象：工具分发器 `run_status`。**

### 2.2 完整代码（逐行注释）

```python
import json
import os
import urllib.parse
import urllib.request
from openai import OpenAI


# ── 工具本体：这次是真 API（wttr.in，免 key）──
def get_weather(city: str) -> str:
    city_encoded = urllib.parse.quote(city)     # 中文 → URL 编码：深圳 → %E6%B7%B1%E5%9C%B3
    url = f"https://wttr.in/{city_encoded}?format=3"
    return urllib.request.urlopen(url).read().decode()   # 返回简短天气文本


# ── ① 函数注册（schema 和版本1一样）──
TOOLS_SCHEMA = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市天气",
            "parameters": {
                "type": "object",
                "properties": {"city": {"type": "string"}},
                "required": ["city"],
            },
        },
    }
]

# ── 工具查表：名字 → 函数（模型只给名字，代码负责翻译）──
TOOLS_FUNC = {"get_weather": get_weather}


def run_status(name: str, args: dict) -> str:
    """工具分发器：模型说"调 get_weather"，这里负责真的调。
    原则：出错不崩，把错误信息返回给模型，让它自己修正。"""
    try:
        if name not in TOOLS_FUNC:
            return f"未知工具:{name}"          # 模型也可能幻觉出不存在的工具
        return TOOLS_FUNC[name](**args)        # 按名字查到函数，展开参数调用
    except Exception as e:
        return f"工具执行失败:{e}"             # 失败也要"上菜"，上的是错误信息


def fc_loop(user_input: str, max_runs: int = 3) -> str:
    """FC 主循环：②③④ 反复转，直到模型直接给答案，或转满上限。"""
    messages = [{"role": "user", "content": user_input}]
    client = OpenAI(
        api_key=os.getenv("DEEPSEEK_API_KEY"),
        base_url="https://api.deepseek.com",
    )

    for _ in range(max_runs):                  # ② 最多转几圈
        res = client.chat.completions.create(
            model="deepseek-v4-flash",
            messages=messages,
            tools=TOOLS_SCHEMA,
            max_tokens=100,
        )
        msg = res.choices[0].message

        if not msg.tool_calls:                 # ③ 模型没要工具 → 给最终答案
            return msg.content

        messages.append(msg)                   # ④ 把带 tool_calls 的消息存进历史
        for tc in msg.tool_calls:
            try:
                arg = json.loads(tc.function.arguments)   # arguments 是 JSON 字符串
            except Exception as e:
                # 参数解析失败也回填——让模型看到"你给的东西我解析不了"
                messages.append({"role": "tool", "tool_call_id": tc.id,
                                 "content": f"参数解析失败:{e}"})
                continue
            result = run_status(tc.function.name, arg)    # 统一分发执行
            messages.append({"role": "tool", "tool_call_id": tc.id,
                             "content": result})          # 结果回填（在 for 内！）

    return "达到最大轮数,已停止"               # 转满上限，兜底返回


if __name__ == "__main__":
    print(fc_loop(input("天气助手:")))
```

### 2.3 🎯 版本 2 最值钱的设计：出错也回填

```python
except Exception as e:
    return f"工具执行失败:{e}"
```

**为什么工具出错也要把错误喂回给模型，而不是自己 raise？**

```fallback
  ❌ 自己抛异常：程序崩了，用户只看到 traceback
  ✅ 错误回填：  模型看到"工具执行失败:xxx"，会自己换个说法重试、
                 换个参数再调、或直接道歉给答案

  本质：把"调试期你自己盯着"的事，变成"模型现场自愈"
```

> **类比**：服务员下单后厨房说"这道菜做不了"。**你不能假装没下过单**——正确做法是把"这道菜没了"也端上桌，让客人（模型）自己决定换菜还是结账。**工具挂了，对话不能挂。**

### 2.4 ⚠️ 顺带记住一个坑：多工具回填的缩进

版本 1 那种 `if` 写法一旦改成 `for` 循环，**回填 `messages.append` 必须在 `for tc` 循环体内部**：

```python
for tc in msg.tool_calls:
    result = run_status(tc.function.name, arg)
    messages.append({...})    # ✅ 在这里：每个 tc 都回填一条
# messages.append({...})      # ❌ 在这里：只有最后一条被回填，其余 400
```

模型一次发了 3 个 `tool_calls`，你只回填 1 条，协议就不完整。**这段代码你以后在 LangGraph 里会再遇到一次——`ToolNode` 替你把这个 for 循环写了。**

> 💡 你现在手写踩的每个坑（回填不齐、id 对不上、解析失败），都是将来理解 `ToolNode`"自动替你做了什么"的素材。**先手写，再框架，你才知道框架替你省了什么。**

* * *

## 三、版本 3：MockLLM——不烧 token 的假模型

### 3.1 为什么要造一个假模型

版本 2 能跑了，但**每测一次就烧一次 API 钱**。而且真模型是"概率输出"——这次返回 2 个 tool_calls，下次可能直接回答，**你没法稳定复现要测的分支**。

> **类比**：你要测试"客人点菜 → 厨房做菜 → 上菜"这条流水线，不该每次都请真客人来——**请个假人按剧本点菜**，才能稳定验证流水线本身有没有 bug。

版本 3 的思路：**仿照 OpenAI 的返回结构，写一个假模型 `MockLLM`。** 它不联网、不烧 token，根据输入关键词**按剧本返回** `tool_calls` 或最终回答。

### 3.2 抽象基类：先把"模型"的接口钉死

```python
class LLMClient:
    """模型客户端抽象：真模型、假模型都实现 chat()，调用方不关心是谁"""
    def chat(self, message, tools=None):
        raise NotImplementedError("子类必须实现 chat()")


class RealLLM(LLMClient):
    """真模型：对接 DeepSeek（OpenAI 兼容协议）"""
    def __init__(self, provide="deepseek"):
        cfg = {
            "deepseek": {"base_url": "https://api.deepseek.com",
                         "key": "DEEPSEEK_API_KEY", "model": "deepseek-v4-flash"},
            "openai":   {"base_url": None,
                         "key": "OPENAI_API_KEY", "model": "gpt-4o-mini"},
        }[provide]
        self.client = OpenAI(api_key=os.getenv(cfg["key"]), base_url=cfg["base_url"])
        self.model = cfg["model"]

    def chat(self, messages, tools=None):
        return self.client.chat.completions.create(
            model=self.model, messages=messages, tools=tools
        )
```

**为什么值得抽这个基类**：`fc_loop` 只认 `llm.chat(...)`，不管背后是真模型还是假模型。**接口钉死了，测试才能替换**——这就是依赖注入的最小形态。

### 3.3 仿造 OpenAI 的返回结构（Fake 系列）

`fc_loop` 里访问的是 `rep.choices[0].message.tool_calls`。要造假模型，就得把这条访问链上的每一层都造出来：

```python
# ── 仿照 openai SDK 的返回对象，逐层构造 ──
class FakeMessage:
    """仿 message：有 content 和 tool_calls 两个属性"""
    def __init__(self, content=None, tool_calls=None):
        self.content, self.tool_calls = content, tool_calls

class FakeChoice:
    """仿 choices[0]"""
    def __init__(self, message):
        self.message = message

class FakeResponse:
    """仿 response：.choices 是个列表，装着 message"""
    def __init__(self, message):
        self.choices = [FakeChoice(message)]

class FakeFunction:
    """仿 tool_calls[i].function：.name 和 .arguments"""
    def __init__(self, name, arguments):
        self.name, self.arguments = name, arguments

class FakeToolCall:
    """仿 tool_calls[i]：.id 和 .function"""
    def __init__(self, name, arguments):
        self.id = "call_mock_001"
        self.function = FakeFunction(name, arguments)
```

> 💡 **为什么要仿到这么细？** 因为 `fc_loop` 读的是 `tc.function.arguments`、`tc.id` 这种深层路径。**假对象只要把循环要访问的路径都提供出来，`fc_loop` 一行都不用改就能跑。** 这就是"面向协议编程"——循环只认结构，不认实现。

### 3.4 MockLLM：按剧本演戏的假模型

```python
class MockLLM(LLMClient):
    """假模型：不联网、不烧 token，按关键词返回写死的 tool_calls"""
    def chat(self, message, tools=None):
        # 剧本①：最后一条是 tool 结果 → 模型"看到结果"，给出最终回答
        if message[-1]["role"] == "tool":
            result = message[-1]["content"]
            return FakeResponse(FakeMessage(
                content=f"模型看到工具结果[{result}]后给出的最终回答"
            ))
        # 剧本②：问题里带"天气" → 返回一个调用 get_weather 的意图
        q = message[-1]["content"]
        if "天气" in q:
            return FakeResponse(FakeMessage(
                tool_calls=[FakeToolCall("get_weather", '{"city":"天津"}')]
            ))
        # 剧本③：问题里带"等于/计算" → 返回一个调用 calculator 的意图
        if "等于" in q or "计算" in q:
            return FakeResponse(FakeMessage(
                tool_calls=[FakeToolCall("calculator", '{"expr": "1+1"}')]
            ))
        # 剧本④：兜底 → 直接给回答
        return FakeResponse(FakeMessage(content="Mock回答"))
```

**看它的"剧本"设计**——每个分支对应 `fc_loop` 的一条路径：

```fallback
  剧本       触发条件                    验证的路径
  ─────     ──────────                  ──────────────
  ①  tool 消息结尾       → fc_loop 的"结果回填后"分支
  ②  含"天气"           → fc_loop 的"发 tool_calls"分支（get_weather）
  ③  含"等于/计算"      → fc_loop 的"发 tool_calls"分支（calculator）
  ④  其余               → fc_loop 的"直接返回"分支
```

> 🎯 **这就是 Mock 的核心价值：不是"假装有模型"，而是"让模型按剧本走你指定的路径"。** 你想测"多轮工具调用"，就让剧本 ① 之后返回剧本 ②——循环自动转两圈。真模型做不到这么听话。

### 3.5 安全版计算器（版本 3 的暗雷）

剧本 ③ 会触发 `calculator('{"expr": "1+1"}')`。**注意：`ast.literal_eval` 只吃字面量，不吃表达式**——`literal_eval("1+1")` 会抛 `ValueError`，因为 `1+1` 是运算不是字面量。

正确做法是白名单式 AST 求值：

```python
import ast
import operator as op

# 白名单：只允许这些运算符进"厨房"
_OPS = {
    ast.Add: op.add, ast.Sub: op.sub, ast.Mult: op.mul, ast.Div: op.truediv,
    ast.USub: op.neg, ast.Pow: op.pow,
}

def _safe_eval(node):
    """遍历 AST 求值：只放行数字和 + - * / 幂、取负，其余一律拒绝"""
    if isinstance(node, ast.Expression):
        return _safe_eval(node.body)
    if isinstance(node, ast.Constant) and isinstance(node.value, (int, float)):
        return node.value
    if isinstance(node, ast.BinOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.left), _safe_eval(node.right))
    if isinstance(node, ast.UnaryOp) and type(node.op) in _OPS:
        return _OPS[type(node.op)](_safe_eval(node.operand))
    raise ValueError(f"不支持的表达式节点: {type(node).__name__}")

def calculator(expr: str) -> str:
    """四则运算：不用 eval，白名单遍历 AST——安全且能算 '1+1' 这类表达式"""
    return str(_safe_eval(ast.parse(expr, mode="eval")))
```

```fallback
  为什么不能直接 eval("1+1")？
    eval 会执行任意代码——用户输入 "1; import os; os.remove(...)" 就完蛋了
    白名单 AST 只认数字和 +-*/，其余节点直接抛错

  literal_eval 呢？
    它只解析"字面量"，'1+1' 是表达式不是字面量 → ValueError
    所以要算表达式，得自己遍历 AST 求值
```

### 3.6 同一个 fc_loop，换引擎跑

```python
def fc_loop(question: str, llm: LLMClient, max_rounds: int = 3):
    """和版本 2 几乎一样——唯一区别：llm 从外面传进来，真假都行"""
    messages = [{"role": "user", "content": question}]
    for _ in range(max_rounds):
        rep = llm.chat(messages, tools=TOOLS_SCHEMA)
        msg = rep.choices[0].message
        if not msg.tool_calls:
            return msg.content                 # 模型直接回答
        messages.append(msg)                   # 存下带 tool_calls 的消息
        for tc in msg.tool_calls:
            try:
                arg = json.loads(tc.function.arguments)
            except Exception as e:
                messages.append({"role": "tool", "tool_call_id": tc.id,
                                 "content": f"参数解析失败:{e}"})
                continue
            result = run_tool(tc.function.name, arg)
            messages.append({"role": "tool", "tool_call_id": tc.id,
                             "content": result})   # ✅ for 内回填
    return "达到最大访问次数"


def run_tool(name: str, arg: dict) -> str:
    """工具分发（和版本2的 run_status 同款）"""
    try:
        return TOOLS_FUNC[name](**arg)
    except Exception as e:
        return f"参数出错了:{e}"
```

**测试时传 `MockLLM()`，上线传 `RealLLM("deepseek")`，循环一个字不用改：**

```python
if __name__ == "__main__":
    # 假模型：零成本验证循环逻辑
    print(fc_loop("天津天气怎么样", MockLLM()))
    # 真模型：验证真实 API 链路
    print(fc_loop("天津天气怎么样", RealLLM("deepseek")))
```

> 💡 **这版演进教会你的事**：**把"模型"抽成接口，是 FC 工程化的第一课。** 之后你写的任何 Agent 代码，都应该问一句"这里能不能塞个假模型进去测？"——能，说明解耦到位；不能，说明逻辑和实现粘死了。

* * *

## 四、版本 4：真实天气 + 模型路由 + 工具记录（防幻觉）

### 4.1 三个新东西

```fallback
  ① get_weather 换 open-meteo：免费、免 key、按经纬度返回结构化天气
  ② route_model：从问题里识别要哪个模型（kimi/deepseek/...）→ 模型硬路由
  ③ fc_loop 返回 {answer, tool_records}：记录每次工具调用 → 检测幻觉
```

前两个是"更好用"，第三个才是这版的重点——**它回答了一个尖锐的问题：模型没调工具就说出了答案，你信吗？**

### 4.2 天气工具：open-meteo（按经纬度查）

```python
import json
import urllib.request

# open-meteo 的天气码 → 中文描述（官方文档的码表）
WEATHER_CODE_DESC = {
    0: "晴朗的天空", 1: "主要晴朗", 2: "局部多云", 3: "阴天",
    45: "雾气", 48: "霜雾沉积",
    51: "毛毛雨：轻度", 53: "毛毛雨：中度", 55: "毛毛雨：密集",
    61: "降雨：轻度", 63: "降雨：中度", 65: "降雨：强雨",
    71: "降雪量：轻度", 73: "降雪量：中度", 75: "降雪量：重度",
    80: "阵雨：轻度", 81: "阵雨：中度", 82: "阵雨：猛烈",
    95: "雷暴：轻度或中度", 96: "雷暴伴轻微冰雹", 99: "雷暴伴强烈冰雹",
}

def get_weather(latitude: float, longitude: float) -> str:
    """按经纬度查实时天气（open-meteo，免 key）。返回中文 dict 的 JSON。"""
    url = (f"https://api.open-meteo.com/v1/forecast?latitude={latitude}"
           f"&longitude={longitude}"
           f"&current=temperature_2m,relative_humidity_2m,wind_speed_10m"
           f"&daily=weather_code,temperature_2m_max,temperature_2m_min,"
           f"sunrise,sunset,precipitation_probability_max"
           f"&timezone=Asia%2FShanghai")
    data = json.loads(urllib.request.urlopen(url).read())
    today = {k: v[0] for k, v in data["daily"].items()}   # daily 数组取今天
    cur = data["current"]
    return json.dumps({
        "天气": WEATHER_CODE_DESC.get(today["weather_code"], "未知天气码"),
        "当前温度_C": cur["temperature_2m"],
        "湿度_%": cur["relative_humidity_2m"],
        "今日最高_C": today["temperature_2m_max"],
        "今日最低_C": today["temperature_2m_min"],
        "日出": today["sunrise"][11:16],
        "日落": today["sunset"][11:16],
        "降雨概率_%": today["precipitation_probability_max"],
    }, ensure_ascii=False)      # ensure_ascii=False：中文原样输出，别变 \uXXXX
```

> ⚠️ 注意 schema 也跟着变了：`city: string` 变成了 `latitude / longitude: number`。**工具的输入从"城市名"变成"经纬度"后，模型的负担也变了**——它得自己把"北京"换算成坐标。这正是版本 4 里模型路由要配合的原因之一（有地理能力的大模型才能干这活）。真实项目里，更常见的是你提供"城市名 → 经纬度"的第二个工具。

### 4.3 模型硬路由：route_model

```python
def route_model(question: str) -> str:
    """模型硬路由：从用户输入里找模型名，找不到用默认。
    简单粗暴但够用——先让"选模型"这件事可复现，再谈智能路由。"""
    for name in ["kimi", "deepseek", "qwen", "gpt"]:
        if name in question.lower():
            return name
    return "deepseek"    # 默认


# 使用时：
llm = RealLLM(route_model(user_input))
```

**为什么在 FC 循环外面加路由？** 因为不同任务适合不同模型——**简单对话用便宜快的，复杂推理用贵的**。硬路由是第一步：规则写在明面上、结果可预测。等哪天规则不够用了，再让模型自己路由（那就是 `auto_select`，版本 5 的主角）。

### 4.4 🎯 tool_records：给 Agent 装"行车记录仪"

```python
def fc_loop(question: str, llm: LLMClient, max_rounds: int = 3) -> dict:
    messages = [{"role": "user", "content": question}]
    tool_records = []                 # 🎯 新：记录每一次工具调用

    for _ in range(max_rounds):
        rep = llm.chat(messages, tools=TOOLS_SCHEMA)
        msg = rep.choices[0].message
        if not msg.tool_calls:
            return {
                "answer": msg.content,          # 最终答案
                "tool_records": tool_records,   # [] = 没调工具（模型猜的！）
            }
        messages.append(msg)
        for tc in msg.tool_calls:
            try:
                arg = json.loads(tc.function.arguments)
            except Exception as e:
                messages.append({"role": "tool", "tool_call_id": tc.id,
                                 "content": f"参数解析失败:{e}"})
                continue
            result = run_tool(tc.function.name, arg)
            tool_records.append({                # 🎯 每次调用都记一笔
                "tool": tc.function.name,
                "args": arg,
                "result": result,
            })
            messages.append({"role": "tool", "tool_call_id": tc.id,
                             "content": result})
    return {"answer": "达到最大轮数", "tool_records": tool_records}
```

**返回值从字符串升级成字典，是这版最重要的设计决策**：

```python
out = fc_loop("天津天气怎么样", RealLLM("deepseek"))

if out["tool_records"]:
    # 正常：回答确实经过工具调用
    print("工具调用记录:", out["tool_records"])
else:
    # ⚠️ 红灯：模型没调任何工具就给了答案 —— 很可能在编！
    print("警告: 模型未调用任何工具, 回答可能为模型推断")
```

> **类比**：`tool_records` 就是给 Agent 装的**行车记录仪**。车撞了（答错了），你有录像能查"它当时到底有没有看路（调工具）"。**没有记录仪，你连"它在编"都分不出来。**

> 🎯 **一句话记忆**：**模型没调工具 ≠ 答对了，也可能是在猜。** 把 `tool_records` 暴露出来，幻觉风险至少能被看见——看见是治理的第一步。

* * *

## 五、版本 5：prompt 工厂 + auto_select（把 W1 和 W2 焊在一起）

### 5.1 这版在干什么

W1 你写了四种 Prompt 模式（CoT / Few-shot / ReAct / ToT），还写了个路由器选模式。W2 你写完了 FC 闭环。

**版本 5 = 路由器 + FC 的合体**：一个 `auto_select` 让模型判断"这问题适合哪种模式"，然后把对应模式的答案拼装成最终回复。**这是"会选模式 + 会调工具"的第一次合流——它就是你 Agent 的雏形。**

```fallback
  用户输入
    │
    ▼
  auto_select(question)   ← LLM 判断：cot / few_shot / react / tot
    │
    ▼
  run_mode(mode, question)
    ├─ tot       → 生成 n 方案 → 打分评审 → 选最优（ToT 三连问）
    ├─ react     → fc_loop + 工具调用 + 行车记录仪检查
    ├─ few_shot  → 按示例模板抽取信息
    └─ cot       → 一步步思考再回答
```

### 5.2 基础设施：流式 + 思考链收集

```python
from openai import OpenAI
import os

client = OpenAI(api_key=os.getenv("DEEPSEEK_API_KEY"),
                base_url="https://api.deepseek.com")


def get_response(messages, **kwargs):
    """带思考链收集的流式请求。
    DeepSeek 的深度思考模型会把"内部推理"放在 reasoning_content 里，
    和最终 content 分开流式返回——这里两个都收。"""
    response = client.chat.completions.create(
        model=kwargs.get("model", "deepseek-v4-flash"),
        messages=messages,
        stream=True,
        reasoning_effort=kwargs.get("reasoning_effort", "low"),
        max_tokens=kwargs.get("max_tokens", 500),
    )
    reasoning_content, content = "", ""
    for chunk in response:
        delta = chunk.choices[0].delta
        if delta.reasoning_content:       # 思考过程（不展示给用户）
            reasoning_content += delta.reasoning_content
        if delta.content:                 # 最终回答（展示给用户）
            content += delta.content
    # 有些模型只给思考不给 content（比如纯推理题），那就把思考当答案
    if content:
        return content
    return reasoning_content
```

> 💡 `reasoning_content` 和 `content` 分流，是 DeepSeek 系列（reasoner）的特色——思考过程单独流式返回。**这让"隐藏思维链"成了协议级能力，而不是 prompt 里求它"一步步想"。**

### 5.3 模式工厂：四个 Prompt 构建器 + 一个特殊出口

```python
def cot_prompt(question, system_prompt="你是一个逻辑缜密，思维严谨的推理助手"):
    """CoT：引导一步步思考再回答（适合数学/推理/公式推导）"""
    u = f"请一步步思考，再给出答案。\n问题:{question}"
    return [{"role": "system", "content": system_prompt},
            {"role": "user", "content": u}]


DEFAULT_EXAMPLES = [
    {"question": "我明天下午3点约了张医生复诊",
     "answer": "时间:明天下午3点;人物:张医生;事项:复诊"},
    {"question": "周五晚上和李总在望江楼吃饭",
     "answer": "时间:周五晚上;人物:李总;事项:吃饭"},
]

def few_shot_prompt(question, examples=DEFAULT_EXAMPLES,
                    system_prompt="你是擅长信息抽取的助手"):
    """Few-shot：先给 2 个"问题→格式化答案"的范例，再问真问题。
    适合固定模板的抽取任务（时间/人物/事项）。"""
    msgs = [{"role": "system", "content": system_prompt}]
    for ex in examples:
        msgs.append({"role": "user", "content": ex["question"]})
        msgs.append({"role": "assistant", "content": ex["answer"]})
    msgs.append({"role": "user", "content": question})
    return msgs


def react_prompt(question, system_prompt="你是一位有多种能力的小助手"):
    """ReAct：不是拼 prompt，而是直接进 fc_loop（W2 的主角）。
    工具调用模式的"提示词"就是 tools schema + 循环本身。"""
    out = fc_loop(question, RealLLM(route_model(question)))
    if out["tool_records"]:
        print("工具调用记录:", out["tool_records"])
    else:
        print("警告: 模型未调用任何工具, 回答可能为模型推断")
    return out["answer"]


def tot_solve(question, n=3):
    """ToT：方案生成 → 打分评审 → 选最优，三连问（W1 的内容）"""
    schemes = ask(
        f"请针对以下问题给出{n}种不同解决思路，编号列出：\n{question}\n",
        system_prompt="你是擅长发散思维的规划专家",
        max_tokens=600,
    )
    scores = ask(
        f"请对以下 {n} 个方案根据实行难度，风险评估逐一打分(1-10分),"
        f"并各用一句话说明理由:\n{schemes}",
        system_prompt="你是严格的评审专家,打分要客观",
        max_tokens=500,
    )
    best = ask(
        f"综合以下评分,选出最优方案,并给出具体实施步骤:\n{scores}",
        system_prompt="你是决策专家,直接给最终选择",
        max_tokens=600,
    )
    return schemes, scores, best
```

> ⚠️ 注意 `react_prompt` 的返回值：**它是字符串**（`out["answer"]`），而其他模式返回**消息列表**。为了让 `run_mode` 统一处理，版本 5 里 react 走了特判（见 5.5）。这种"接口不齐"是演进过程中的常态——先跑通，再谈统一。

### 5.4 调度总入口：BUILDERS 查表

```python
BUILDERS = {"cot": cot_prompt, "few_shot": few_shot_prompt,
            "react": react_prompt}
```

`react` 也放进 BUILDERS 是为了让 `auto_select` 的匹配逻辑统一——**但它在 run_mode 里被特判，因为它的返回类型和别人不一样**（见下）。

### 5.5 auto_select + run_mode：会选模式的 Agent

```python
def auto_select(question: str) -> str:
    """让 LLM 判断这问题适合哪种模式，返回模式名。
    规则写在 prompt 里；识别用关键词兜底（tot→react→few_shot→cot）。
    注意匹配顺序：tot 最长，cot 最短——先匹配长的，避免 'cot' 误吞 'react'。"""
    content = (
        f"判断下面用户的问题适合哪种回答模式，输出：tot_solve 或 react_prompt "
        f"或 few_shot_prompt 或 cot_prompt\n"
        f"规则:需要一步步推理/数学解题/公式推导的输出 cot;"
        f"固定模板化抽取信息的输出 few_shot;"
        f"需要调工具(天气/计算等)的输出 react;"
        f"开放性问题/多方案对比找最优的输出 tot\n"
        f"问题:{question}"
    )
    text = get_response([{"role": "user", "content": content}],
                        max_tokens=50).lower()
    for mode in ["tot", "react", "few_shot", "cot"]:
        if mode in text:
            return mode
    print("[警告] 未识别到模式名,兜底 cot")
    return "cot"


def run_mode(mode: str, question: str, **kw):
    """按模式执行。react 和 tot 是特判（有副作用/多步），其余走 BUILDERS。"""
    if mode == "tot":
        schemes, scores, best = tot_solve(question, n=kw.get("n", 3))
        return f"【方案】\n{schemes}\n\n【评审】\n{scores}\n\n【决策】\n{best}"
    if mode == "react":
        return react_prompt(question)
    if mode not in BUILDERS:
        raise ValueError(f"未知模式:{mode},可选 {list(BUILDERS) + ['tot']}")
    msgs = BUILDERS[mode](question, **kw)
    return get_response(msgs, **kw)


# ── 主循环：选模式 → 执行 → 输出（Agent 的第一版驾驶舱）──
if __name__ == "__main__":
    while True:
        user_input = input("User: ")
        if user_input.lower() in ["exit", "quit"]:
            break
        mode = auto_select(user_input)
        print(f"[使用模式] {mode}")
        result = run_mode(mode, user_input)
        print("Agent:", result)
```

### 5.6 🎯 这版的地位：你第一次"让模型决定怎么答"

看主循环那三行——**它不再是你选 prompt，而是模型自己选 prompt**：

```fallback
  版本 4 及以前：你写死一种方式 → 调用
  版本 5：      模型看问题 → 自己选 cot/few_shot/react/tot → 执行

  本质：把"怎么答"这个决策，也交给了模型。
  ← 这就是 Agent 与"调 API"的分界线
```

> ⚠️ **诚实提醒**：`auto_select` 用"关键词兜底"而非纯靠 LLM，是有意的。纯靠 LLM 选模式，选错一次整个回答就废了；**关键词兜底虽然笨，但可预测、可调试**。等你把每种模式的触发规则摸清了，再放开让 LLM 全权决定也不迟。

* * *

## 六、五版演进总对照（一张表看懂）

| 版本 | 解决什么 | 新增的关键物 | 一句话 |
|---|---|---|---|
| **v1 最小闭环** | FC 是什么 | `tools` schema + `if tool_calls` | 先让"点菜→做菜→上菜"转起来 |
| **v2 真 API + 分发** | 单轮变多轮 | `fc_loop` 循环 + `run_status` 分发 + 出错回填 | 循环自己转，工具挂了对话不挂 |
| **v3 MockLLM** | 测试烧钱、不可复现 | `LLMClient` 抽象 + `Fake*` 结构 + 剧本 | 假模型按剧本演戏，零成本测循环 |
| **v4 路由 + 记录** | 模型会编答案 | `route_model` + `tool_records` | 行车记录仪：没调工具 = 可能在猜 |
| **v5 prompt 工厂** | 一种方式答所有题 | `auto_select` + `run_mode` + BUILDERS | 让模型自己决定"怎么答" |

**演进主线其实是两条**：

```fallback
  纵向：让循环更皮实
    v1 if 单轮 → v2 for 多轮 → v3 可测 → v4 可查 → v5 会选

  横向：抽象边界越来越清晰
    tools（菜单）→ run_status（分发）→ LLMClient（引擎接口）
    → tool_records（记录）→ BUILDERS（模式注册表）
```

> 🎯 **回头看，你其实在无意识地实践三层抽象**：**数据（messages/tools）、执行（分发/循环）、策略（路由/选模式）**。这三层分开，是后面一切 Agent 工程的地基。

* * *

## 七、踩坑记

### 坑 1：手写 tools schema 结构记错 ⚠️

```python
# ❌ 少了 "type": "function" 这层
{"function": {"name": "get_weather", "parameters": {...}}}

# ✅ 标准结构：type → function → name/description/parameters
{"type": "function", "function": {"name": "get_weather", "description": "...",
  "parameters": {"type": "object", "properties": {...}, "required": [...]}}}
```

- **现象**：请求 400 或模型不识别工具
- **解法**：对照官方文档逐层写。**这也是为什么后来 LangChain 的 `@tool` 值钱**——它从函数签名自动生成这一整坨，不用你手写（W4-D5 见）。

### 坑 2：`tool_call_id` 不匹配 → 400 ⚠️

- **现象**：下一轮请求报 `insufficient tool messages` 之类的错
- **原因**：模型发了 N 个 `tool_calls`，你回填了 M 条，或 id 抄错
- **解法**：**每条 `tool_calls` 都必须有对应的一条 `tool` 消息**，id 原样带回。多工具时注意 `append` 在 `for` 循环内部

### 坑 3：忘了把带 `tool_calls` 的 assistant 消息存回 messages ⚠️

- **现象**：模型第二轮好像"失忆"，重复发同一个工具调用
- **原因**：`messages.append(msg)` 被跳过，模型不知道自己点过菜
- **解法**：先 append 再执行工具，顺序不能反

### 坑 4：`ast.literal_eval` 不认表达式 🔴

- **现象**：`calculator("1+1")` 抛 `ValueError`
- **原因**：`literal_eval` 只解析字面量（数字/字符串/列表/dict），`1+1` 是表达式
- **解法**：白名单 AST 求值（见 3.5），**别用裸 `eval`**——它会执行任意代码

### 坑 5：工具执行出错直接 raise，把整个对话搞崩 ⚠️

- **现象**：一个工具挂了，用户只看到 traceback
- **解法**：`run_status` 用 `try/except` 兜住，**把错误信息当结果回填**——模型看到错误自己会修正
- 🎯 **原则**：工具是增强项不是可用性依赖，工具挂了对话得继续

### 坑 6：`urllib.parse.quote` 忘 import（隐式依赖）⚠️

- **现象**：本地能跑、换环境就 `NameError: name 'urllib' is not defined`（或半可用）
- **原因**：`import urllib.request` 内部顺带加载了 `urllib.parse`，你"蹭"到了它——这是隐式依赖，脆弱
- **解法**：用哪个子模块就显式 `import urllib.parse` / `import urllib.request`

### 坑 7：模型没调工具就给了答案，你还真信了 🔴

- **现象**：问"北京天气"，模型答"今天晴，25℃"，但它压根没调工具
- **原因**：**模型不会的也会编**（幻觉），没调工具 = 没有真实数据来源
- **解法**：`fc_loop` 返回 `tool_records`，**空记录时打警告**——先看见幻觉，才能治理幻觉

### 坑 8：把"选模式"也交给 LLM，选错一次全废 ⚠️

- **现象**：`auto_select` 判断失误，数学题走了聊天模式
- **解法**：**先用关键词兜底规则**（可预测、可调试），摸清触发边界后再放开给 LLM
- 💡 规则的优先级：能写死的别让模型猜——**模型只做规则做不了的事**

* * *

## 八、速查卡片（复习直接看这）

```python
# ===== FC 闭环四步 =====
# ① 注册：tools = [{"type": "function", "function": {name/description/parameters}}]
# ② 模型：res = client.chat.completions.create(..., tools=tools)  # 不传 tools 永远不点菜
# ③ 执行：result = TOOLS_FUNC[name](**json.loads(tc.function.arguments))
# ④ 回填：messages.append({"role": "tool", "tool_call_id": tc.id, "content": result})

# ===== fc_loop 骨架（五要素）=====
def fc_loop(question: str, llm: LLMClient, max_rounds: int = 3):
    messages = [{"role": "user", "content": question}]
    tool_records = []
    for _ in range(max_rounds):                     # ② 有界循环
        rep = llm.chat(messages, tools=TOOLS_SCHEMA)   # 引擎可替换：真/假都行
        msg = rep.choices[0].message
        if not msg.tool_calls:                      # ③ 没点菜 → 直接回答
            return {"answer": msg.content, "tool_records": tool_records}
        messages.append(msg)                        # 存下点菜单（别忘了！）
        for tc in msg.tool_calls:
            try:
                arg = json.loads(tc.function.arguments)
            except Exception as e:
                messages.append({"role": "tool", "tool_call_id": tc.id,
                                 "content": f"参数解析失败:{e}"})
                continue
            try:
                result = TOOLS_FUNC[tc.function.name](**arg)
            except Exception as e:
                result = f"工具执行失败:{e}"         # 出错也回填
            tool_records.append({"tool": tc.function.name, "args": arg,
                                 "result": result})
            messages.append({"role": "tool", "tool_call_id": tc.id,
                             "content": result})     # ✅ for 内回填
    return {"answer": "达到最大轮数", "tool_records": tool_records}

# ===== 三条铁律 =====
# 1. 发几个 tool_calls 就回填几条 tool 消息，id 一一对应
# 2. 出错也回填错误信息，让模型自纠，别 raise
# 3. 没调工具就给的答案，默认当"猜测"处理（tool_records 留痕）
```

* * *

## 九、一句话总结

**Function Calling 就是"模型只会点菜、不会做菜"这件事的解法：tools 菜单注册、模型输出 `tool_calls` 点菜单、你的代码进厨房执行、结果回填上菜——循环到模型直接给答案为止。**

**五版演进你做的事其实是一条线**：

```fallback
  先把闭环转起来（v1）
  → 让循环自己转、出错不崩（v2）
  → 造个假模型零成本测循环（v3）
  → 装上行车记录仪防幻觉（v4）
  → 让模型自己选怎么答（v5）
```

**回头看，抽象边界在每一版里悄悄清晰**：菜单（schema）、厨房（分发器）、引擎（`LLMClient`）、记录（`tool_records`）、策略（路由/`BUILDERS`）。**这三五层分开的那一刻，你已经不是在"调 API"，而是在"组装 Agent"了。**

> 再往后看两件事的伏笔：
> 1. `run_status` + 出错回填 → W4 的 `ToolMessage` 失败回填、W5 的 `ToolNode`（把 for 循环替你写了）
> 2. `tool_records` → W5 的 `stream` 看节点流转（同一个"看得见才可信"的思路）
>
> **下一篇：W3 MemoryManager**——现在这辆车会调工具了，但它没有记忆。下一周给它装上短期窗口、长期向量库和滚动压缩。

* * *
