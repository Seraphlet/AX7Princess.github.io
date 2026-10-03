---
description: ""
title: "一二周整理"
draft: false
date: "2026-08-21T12:49:45+08:00"
slug: "agent-project"
categories:
 - Agent
tags:
 - Memory
 - Function Calling
image: ""
---


我一直想要一个这样的 Agent：

- **能记住**：上一轮你说的话、你长期透露的偏好，它都记得；
- **会办事**：能查天气、算算术、把你的偏好存进"长期记忆"里；
- **能换脑子**：同一个问题，可以用思维链(CoT)慢慢推、用少样本(Few-shot)学样子、用 ReAct 调工具、用 Tree-of-Thought 多方案权衡。

`D:\agent_project` 就是这么一个东西。它不大，但把 Agent 工程里最该懂的几块都拆开了：**LLM 层 / 记忆层 / 工具层 / Agent 层**，外加一个 `agent/main.py` 把它们粘起来。

这篇文章把每个模块的作用、知识点、源码都摊开讲。你照着读一遍，基本就能自己复刻一个。

---

## 一、项目全景：四层 + 一个组装口

先看整体骨架。`agent_project` 的目录长这样：

```
D:\agent_project\
├── agent\                # Agent 层：调度大脑
│   ├── __init__.py       # 暴露 fc_loop / auto_select 等入口
│   ├── main.py           # 组装所有层 + 交互循环
│   ├── core.py           # Function Calling 主循环 fc_loop
│   ├── router.py         # 选模型 + 选推理模式
│   └── modes\            # 四种推理模式
│       ├── cot.py        # 思维链
│       ├── few_shot.py   # 少样本
│       ├── react.py      # 工具调用
│       └── tot.py        # 树状思维
├── llm\                  # LLM 层：模型抽象与封装
│   ├── base.py           # 抽象接口 + 配置读取
│   ├── real_llm.py       # 非流式客户端（带 .env + summarize）
│   └── stream_llm.py     # 流式客户端
├── memory\               # 记忆层：短期 + 长期 + 压缩
│   ├── paths.py          # 落盘路径集中管理
│   ├── shortmemory.py    # 短期滑动窗口
│   ├── long_term.py      # 长期向量库（Chroma）
│   ├── compress.py       # 滚动摘要压缩
│   └── memory_manager.py # 统一调度者
├── tools\                # 工具层：工具定义 + 注册 + 执行
│   ├── base.py           # 工具基类
│   ├── registry.py       # 动态注册表
│   ├── executor.py       # 带"自愈"的执行器
│   ├── weather.py        # 查天气
│   ├── calculator.py     # 计算器
│   ├── save_preference.py# 存长期偏好
│   └── recall_memory.py  # 召回长期记忆
├── config.json           # providers + tools 清单
└── .env                  # 密钥（不进 git）
```

用一张图把数据流说清楚：

```
                        用户输入 "你: ..."
                                 │
                                 ▼
        ┌─────────────────────────────────────────────┐
        │              agent/main.py  (组装口)          │
        │  1. mm.add(user_msg, persist=是否长期事实)      │
        │  2. ctx = mm.get_context(query=用户问题)        │
        │  3. out = fc_loop(llm, ctx, registry, executor)│
        │  4. mm.add(assistant_msg)                      │
        │  5. mm.maybe_compress()  ← 超阈值才压缩         │
        └───────────────────────┬─────────────────────┘
                                │
            ┌───────────────────┼───────────────────┐
            ▼                   ▼                   ▼
      ┌──────────┐       ┌──────────────┐    ┌─────────────┐
      │ 记忆层    │       │  Agent 层     │    │  工具层      │
      │ Memory   │◄──────┤ fc_loop 循环  │───►│ 调 get_weather│
      │ Manager  │       │ router 选模式 │    │ 调 calculator│
      └────┬─────┘       └──────┬───────┘    └──────┬──────┘
           │                    │                   │
           ▼                    ▼                   ▼
     短期窗口 + 长期向量库    RealLLM / StreamLLM    依赖注入 mm
           │                    (LLM 层)             (MemoryManager)
           └──────────────────────────────────────────┘
```

**铁律 1**：分层不是为了好看，是为了"换一层不影响其他层"。模型从 DeepSeek 换成 Kimi，只动 `config.json`；加一个新工具，只写一个 `tools/xxx.py` 再往 `config.json` 加一行——这就是分层架构的回报。

---

## 二、LLM 层：把"模型"抽象成可替换的零件

这一层解决两件事：**① 统一接口**，让上层不关心背后是哪家模型；**② 配置与代码分离**，密钥放 `.env`，模型地址放 `config.json`。

### 2.1 抽象基类 `llm/base.py`

```python
# llm/base.py —— LLM 抽象接口
from pathlib import Path
import json

# 配置路径：从"llm/ 的上级目录"找 config.json
CONFIG_PATH = Path(__file__).resolve().parent.parent / "config.json"

def load_provider_config(provider: str) -> dict:
    with open(CONFIG_PATH, encoding="utf-8") as f:
        return json.load(f)["providers"][provider]

class LLMClient:
    def chat(self, messages, tools=None):
        raise NotImplementedError("子类必须实现 chat()")
    def chat_stream(self, messages, **kwargs):
        raise NotImplementedError("子类必须实现 chat_stream()")
```

**知识点补充**：`load_provider_config` 拿到的 `CONFIG_PATH` 用的是 `Path(__file__).resolve().parent.parent`——这是"相对模块自身位置找文件"的写法，比 `open("config.json")` 稳，因为你从哪个目录运行脚本，`__file__` 都在，而当前工作目录 `cwd` 会漂。

### 2.2 非流式客户端 `llm/real_llm.py`

这是项目的主力客户端（main.py 用的就是它）。它比基类多了两件关键事：**读 `.env`** 和 **`summarize` 方法**（记忆压缩要用）。

```python
# llm/real_llm.py —— 非流式客户端
import os
from pathlib import Path
from openai import OpenAI
from dotenv import load_dotenv
from .base import LLMClient, load_provider_config

# 加载项目根目录的 .env（让 DEEPSEEK_API_KEY 等从文件读取，优先级高于系统环境变量）
load_dotenv(Path(__file__).resolve().parent.parent / ".env", override=True)

class RealLLM(LLMClient):
    def __init__(self, provider="deepseek"):
        cfg = load_provider_config(provider)          # 从 config.json 读
        self.client = OpenAI(api_key=os.getenv(cfg["key_env"]),
                             base_url=cfg["base_url"])
        self.model = cfg["model"]
        print(f"实际使用的 provider={provider}, "
              f"key 已加载={'是' if os.getenv(cfg['key_env']) else '否'}")

    def chat(self, messages, tools=None,**kwargs) -> str:
        """普通对话：返回纯文本（demo / 对话循环用）"""
        resp = self.client.chat.completions.create(
            model=self.model, messages=messages, tools=tools,
            max_tokens=kwargs.get("max_tokens"))
        return resp.choices[0].message.content

    def chat_for_tools(self, messages, tools):
        """工具对话：返回完整 message 对象（fc_loop 用，能拿到 tool_calls）"""
        resp = self.client.chat.completions.create(
            model=self.model, messages=messages, tools=tools)
        return resp.choices[0].message

    def summarize(self, text: str) -> str:
        """把一段对话文本压缩成简短摘要（记忆压缩模块 compress 调用）"""
        messages = [
            {"role": "system", "content": "你是一个摘要助手。请把下面的对话浓缩成简洁的中文要点，"
                                           "保留关键事实、用户偏好与决策，不要添加新内容。"},
            {"role": "user", "content": text},
        ]
        return self.chat(messages, max_tokens=500)
```

**逐行拆解**：

- 第 9 行 `load_dotenv(..., override=True)` 是**铁律级**写法。`override=True` 表示"`.env` 里的变量优先级高于系统环境变量"。否则如果你系统里曾设过一个错的 `DEEPSEEK_API_KEY`，Python 会先用系统的，`.env` 白读——这坑我踩过，报 401 查了半天才发现是系统里有个旧 key 顶掉了文件里的。
- `chat` 返回 `str`（纯文本），`chat_for_tools` 返回**完整 message 对象**。为什么要两个？因为 Function Calling 要拿 `message.tool_calls`，只有完整对象里才有，`chat` 只吐 `content` 是不够的。这俩分工是后面 `fc_loop` 能跑起来的前提。
- `summarize` 是给记忆压缩模块 `compress.py` 用的"摘要能力"。注意它本质就是一次普通 `chat`，只是 system 提示词换成了"你是摘要助手"。

### 2.3 流式客户端 `llm/stream_llm.py`

```python
# llm/stream_llm.py —— 流式客户端
import os
from openai import OpenAI
from .base import LLMClient, load_provider_config

class StreamLLM(LLMClient):
    def __init__(self, provider="deepseek"):
        cfg = load_provider_config(provider)
        self.client = OpenAI(api_key=os.getenv(cfg["key_env"]),
                             base_url=cfg["base_url"])
        self.model = cfg["model"]

    def chat_stream(self, messages, **kwargs):
        """流式：yield 逐段吐（边收边显示）"""
        stream = self.client.chat.completions.create(
            model=self.model, messages=messages, stream=True,
            reasoning_effort=kwargs.get("reasoning_effort", "low"),
            extra_body={"thinking": {"type": "enabled"}},
            max_tokens=kwargs.get("max_tokens", 500),
        )
        for chunk in stream:
            delta = chunk.choices[0].delta.content
            if delta:                     # 过滤空 chunk（思考过程不吐）
                yield delta
```

**知识点补充**：流式用 `yield` 而不是 `return`，于是它变成一个**生成器**，调用方可以 `for piece in llm.chat_stream(msgs): print(piece, end="")` 实现"边生成边打印"。`if delta:` 那行很关键——开了 thinking 的模型每个 chunk 可能先吐思考过程再吐正文，空 `content` 的 chunk 要过滤掉，否则会打印一堆 `None`。

> 注：`stream_llm.py` 没在 `main.py` 里用（主流程走 `RealLLM`），但它是 LLM 层"统一接口"的体现——`StreamLLM` 同样继承自 `LLMClient`，将来想让对话逐字蹦出来，换一行实例化就行。

### 2.4 `config.json`：模型与工具的"总清单"

```json
{
  "providers": {
    "deepseek": {
      "base_url": "https://api.deepseek.com",
      "key_env": "DEEPSEEK_API_KEY",
      "model": "deepseek-v4-flash"
    },
    "kimi": {
      "base_url": "https://api.moonshot.cn/v1",
      "key_env": "MOONSHOT_API_KEY",
      "model": "kimi-k2.6"
    }
  },
  "tools": [
    { "module": "tools.weather", "class": "WeatherTool" },
    { "module": "tools.calculator", "class": "CalculatorTool" },
    { "module": "tools.save_preference", "class": "SavePreferenceTool" },
    { "module": "tools.recall_memory", "class": "RecallMemoryTool" }
  ]
}
```

**知识点补充**：`key_env` 这种设计把"哪把钥匙对应哪个模型"写进了配置，代码里永远不直接写密钥，只写 `os.getenv(cfg["key_env"])`。密钥在 `.env` 里：`DEEPSEEK_API_KEY=sk-xxxx`，`.env` 不进 git。

**一句话记住**：LLM 层 = 一个抽象基类 + 两个实现 + 一份"模型说明书"（`config.json`）+ 一个密钥保险箱（`.env`）。

---

## 三、记忆层：短期窗口 + 长期向量库 + 一条滚动摘要

记忆层是整个项目里最"值钱"的部分，它把"对话记忆"拆成三种形态：

| 形态 | 实现 | 生命周期 | 用途 |
|------|------|----------|------|
| 短期记忆 | `ShortTermMemory` | 进程内，窗口滑动 | 最近 N 轮对话 |
| 长期记忆 | `LongTermMemory`(Chroma) | **落盘，关电脑不丢** | 用户稳定偏好/事实 |
| 压缩摘要 | `compress.py` | 进程内，一条 | 老历史浓缩，省 token |

### 3.1 路径集中管理 `memory/paths.py`

```python
"""集中管理所有落盘路径, 避免各处硬编码 './xxx' (相对路径会随 cwd 漂移)."""
# ============ 隐私分流（设计约定, 不是代码自动给的） ============
# 🔒 永远留本地、绝不进 prompt: 用户真实姓名 / 联系方式 / 密钥 / 明文敏感偏好
# 🌐 会发到云端 LLM API: 你主动塞进 system/user 消息的"召回结果"
# ⚠️ 敏感事实: 存本地, 但召回时不自动进 prompt, 需用户显式确认才带上
# ==============================================================

import os
BASE = os.path.dirname(os.path.abspath(__file__))
MEM_STORE   = os.path.join(BASE, "mem_store")    # Chroma 向量库
PROFILE_DB  = os.path.join(BASE, "memory.db")    # SQLite 用户画像
FACTS_JSON  = os.path.join(BASE, "facts.json")   # 纯本地简单事实
```

**逐行拆解**：注释里那套"隐私分流"是设计约定——`MEM_STORE` 是 Chroma 向量库路径，`PROFILE_DB` 是给 SQLite 画像预留的（当前代码用的是 `facts.json` 思路的简化版，向量库已足够）。所有路径都用 `os.path.abspath(__file__)` 算出来，保证无论从哪运行都不会找错地方。

### 3.2 短期记忆 `memory/shortmemory.py`

```python
class ShortTermMemory:
    def __init__(self,window=10,max_tokens:int=500,system_roles=("system",)):
        self.window=window          # 聊天窗口大小
        self.max_tokens=max_tokens  # 最大 tokens
        self.system_roles=system_roles
        self.buffer=[]              # 消息缓存

    def _tokens(self,msgs) ->int:   # 估算 token 数
        return sum(max(1,len(m.get("content",""))//3)for m in msgs)

    def add(self,msg:dict):         # 消息缓存
        self.buffer.append(msg)

    def context(self)->list[dict]:  # 返回最近 n 条消息，滑动窗口
        sys_msgs=[m for m in self.buffer if m.get("role") in self.system_roles]
        others=[m for m in self.buffer if m.get("role") not in self.system_roles]
        return sys_msgs+others[-self.window:]

    def trim(self):                 # 消息裁剪
        sys_msgs= [m for m in self.buffer if m.get("role") in self.system_roles]
        others =[m for m in self.buffer if m.get("role") not in self.system_roles]
        while self._tokens(others) >self.max_tokens and len(others)>1:
            others.pop(0)
        self.buffer=sys_msgs+others
        return self.buffer
```

**知识点补充**：`context()` 实现的是**滑动窗口**——永远只取最近 `window` 条非系统消息，系统消息（`system`）永远保留在最前。`trim()` 是更狠的"裁剪"：按 token 硬砍最老的，直到不超 `max_tokens`。**压缩 ≠ 裁剪**：`trim` 是"删"，`compress` 是"把老历史浓缩成一条摘要再保留"，两者互补。

### 3.3 长期记忆 `memory/long_term.py`

```python
'''
长期记忆 = 关电脑也不丢
长期记忆 = 单文档 RAG + 落盘
    嵌入/相似度召回  → 只是把"文档"换成"用户事实"且持久化
pip install chromadb
'''
import chromadb
from pathlib import Path
import chromadb, time
from memory.paths import MEM_STORE

class LongTermMemory:
    def __init__(self, path=MEM_STORE, collection_name="long_term"):
        self.client = chromadb.PersistentClient(path=path)  # 绝对路径
        self.col = self.client.get_or_create_collection(collection_name)

    def add_fact(self, text: str, fid: str, private: bool = False):
        """存一条长期事实。private=True 表示敏感, 召回时默认不出网。"""
        self.col.add(ids=[fid], documents=[text],
                     metadatas=[{"private": private, "ts": time.time()}])

    def recall(self,query:str,k:int=3,include_private=False)->list[str]:
        """召回相关事实。默认跳过 private=True 的敏感事实(不进 prompt)。"""
        res=self.col.query(query_texts=[query],n_results=k)
        facts=[]
        for doc,meta in zip(res["documents"][0],res["metadatas"][0]):
            if  meta and meta.get("private") and not include_private:
                continue
            facts.append(doc)
        return facts

    def delete(self,fid:str):
        self.col.delete(ids=[fid])
```

**知识点补充（RAG 召回）**：这其实就是一个迷你版 RAG（检索增强生成）。Chroma 的 `PersistentClient` 把向量库**落盘到 `MEM_STORE` 目录**，所以关掉程序再打开，记忆还在。`add_fact` 时 Chroma 会用内置 embedding 模型把 `text` 变成向量；`recall` 时把 `query` 也变成向量，做相似度检索，返回最像的句子。代码里那个 `from memory.paths import MEM_STORE` 就是 3.1 集中路径管理的价值——改路径只动 `paths.py` 一处。

> **踩坑记**：我第一次跑它时 `chromadb` 要联网下载 `all-MiniLM-L6-v2` 这个 embedding 模型，本地没代理直接 `httpx.ConnectTimeout`。后来走 Clash 代理（7890）把模型下到了本机缓存 `C:\Users\MLoong\.cache\chroma\onnx_models`，之后离线也能跑。

**铁律 2**：`private` 字段是隐私底线的代码化——敏感事实（真实姓名、密钥等）**存本地但不自动进 prompt**，`recall` 默认 `include_private=False` 跳过它。想让它进模型，必须显式传 `include_private=True`（且最好让用户确认）。

### 3.4 滚动摘要压缩 `memory/compress.py`

这是记忆层的"灵魂"，也是我修过 bug 的地方。

```python
"""
上下文压缩——>一条滚动摘要，永远只有一条
"""

def _estimate_tokens(text:str) ->int:#token估算
    """
    估算tokens，达到上限压缩一次。
    中文为主：1 字≈1 token，直接用字符数估算（原 *0.6 严重低估中文，
    导致上下文要累积到 2500+ 字才触发压缩，实际等于不压缩）。
    """
    return max(1, len(text))

def _fmt(messages) ->str:#历史消息拼接
    """消息列表，摘要原材料"""
    return "\n".join(f"{m['role']}:{m['content']}" for m in messages)

def compress(messages:list[dict],llm,keep:int=4) ->list[dict]:#压缩逻辑函数
    """滚动压缩版本"""
    sys_msgs=[m for m in messages if m.get("role")=="system" and not m.get("is_summary")]
    old_sums=[m for m in messages if m.get("is_summary")]
    others=[m for m in messages if m.get("role")!="system" and not m.get("is_summary")]
    if len((others))<=keep:
        return messages
    old,recent=others[:-keep],others[-keep:]

    # 滚动关键：旧摘要 + 新滚出窗口的原文 → 合并成一条新摘要
    text=_fmt(old)
    if old_sums:
        text=f"已有摘要:{old_sums[-1]['content']}\n新增对话：\n{text}"
    summary=llm.summarize(text)
    summary_msg={"role":"system","content":f"历史摘要：{summary}","is_summary":True}
    return sys_msgs + [summary_msg] + recent

def maybe_compress(messages,llm,max_tokens:int=3000,keep=4):#自动压缩检测
    """自动触发，超阈值才压缩，没超原样返回"""
    if (_estimate_tokens(_fmt(messages))>max_tokens and len(messages)> keep+1):
        return compress(messages,llm,keep=keep)
    return messages
```

**逐行拆解**：

- `_estimate_tokens` 现在是 `return max(1, len(text))`——**中文 1 字≈1 token**。原先写的是 `len(text)*0.6`，把中文 token 数严重低估，结果上下文要攒到 2500+ 字才触发压缩，等于永远不压缩。**这是我在调试时发现的一个真 bug**，改成字符数后，2520 字立刻触发了压缩，出来 5 条消息 + 1 条摘要。
- `compress` 的"滚动"精髓在第 32-36 行：**如果已经有旧摘要，就把"旧摘要 + 新滚出窗口的原文"一起喂给 `llm.summarize`**。这样无论聊多久，摘要永远只有**一条**（`is_summary=True`），不会越压越多、越压越乱。
- `is_summary` 这个标记位是防止"摘要被二次压缩"的关键。后面 `sys_msgs` 过滤时特意排除了 `is_summary` 的 system 消息，避免把摘要当普通消息再处理。

**铁律 3**：压缩是"浓缩"不是"删除"。`keep` 条最近对话永远原样保留，只有"滚出窗口的老历史"被压成摘要。所以 Agent 不会忘掉你刚说的，只会忘掉"你上周说的、但已浓缩进摘要"的细节。

### 3.5 统一调度者 `memory/memory_manager.py`

```python
"""
记忆调度：把三个记忆模块拼接起来，对外只暴露 add / get_context / maybe_compress / recall
- get_context 里召回结果是临时拼接、不写回 buffer → 避免"每次召回都追加 system 导致堆积"
- maybe_compress 用 `msgs is not self.stm.buffer` 判断"真的压了才写回" → 没超阈值不碰 buffer
- add_fact 的 fid 用内容 md5 哈希 → 稳定、不重复、跨进程
"""
from .compress import compress, maybe_compress as _auto_compress
from .shortmemory import ShortTermMemory
from .long_term import LongTermMemory
import hashlib

def _fid(text: str) -> str:
    """内容哈希生成稳定ID：不重复、可跨进程（比 hash() 稳，hash 有随机盐）"""
    return "f" + hashlib.md5(text.encode("utf-8")).hexdigest()[:12]

class MemoryManager:
    def __init__(self, window=8, max_tokens=2000, keep=4, llm=None):
        self.stm = ShortTermMemory(window=window, max_tokens=max_tokens)
        self.ltm = LongTermMemory()
        self.keep, self.max_tokens, self.llm = keep, max_tokens, llm

    def add(self, msg: dict, persist: bool = False):
        self.stm.add(msg)
        # 只有明确是用户事实才落长期（persist 默认 False，省 embedding 开销）
        if persist and msg.get("role") == "user":
            self.ltm.add_fact(msg["content"], fid=_fid(msg["content"]))

    def get_context(self, query: str = None) -> list[dict]:
        base = self.stm.context()  # [系统提示] + [窗口消息]
        if not query:
            return base
        facts = self.ltm.recall(query, k=3)
        if not facts:
            return base
        facts_msg = {"role": "system", "content": "已知用户长期事实：" + "|".join(facts)}
        sys_msgs = [m for m in base if m.get("role") in self.stm.system_roles]
        others = [m for m in base if m.get("role") not in self.stm.system_roles]
        return sys_msgs + [facts_msg] + others  # 系统提示 + 长期事实 + 窗口

    def maybe_compress(self, llm=None):
        llm = llm or self.llm
        if llm is None:
            return self.stm.buffer
        msgs = _auto_compress(self.stm.buffer, llm,
                              max_tokens=self.max_tokens, keep=self.keep)
        if msgs is not self.stm.buffer:  # 真压缩了才写回
            self.stm.buffer = msgs
        return self.stm.buffer

    def recall(self, query: str, k: int = 3) -> list[str]:
        return self.ltm.recall(query, k=k)
```

**逐行拆解**：

- `MemoryManager` 是**调度者，不是新逻辑**。它自己不存东西，只把 `ShortTermMemory` 和 `LongTermMemory` 组合起来，对外暴露干净的四方法：`add` / `get_context` / `maybe_compress` / `recall`。上层（main.py）只跟它打交道。
- `get_context` 里召回结果是**临时拼进上下文**的（第 38-40 行重新拼 `sys_msgs + facts_msg + others`），**没有写回 `buffer`**。这点很重要——否则每次召回都把"已知用户长期事实"塞进 buffer，buffer 会越滚越大、重复堆积。
- `maybe_compress` 第 48 行用 `msgs is not self.stm.buffer` 判断"**真的压了**才写回"。`is` 是身份判断：没超阈值时 `compress` 原样返回同一个 list，身份不变 → 不写回；超阈值时返回新 list → 写回。一个 `is` 省掉了无意义的覆盖。
- `_fid` 用 `hashlib.md5(text)` 生成稳定 ID，而不是 Python 内置 `hash()`——因为 `hash()` 有随机盐，每次进程重启同一个字符串 hash 值不一样，会导致 Chroma 里重复存储或删不掉。`md5` 跨进程稳定。

**一句话记住**：记忆层 = 短期窗口（删旧的）+ 长期向量库（存偏好）+ 滚动摘要（压老的），三者由一个只调度、不改写的 `MemoryManager` 串起来。

---

## 四、工具层：注册表 + 自愈执行器 + 依赖注入

工具层让 Agent "能干活"。它的设计亮点是**配置驱动注册**和**执行时自愈**。

### 4.1 工具基类 `tools/base.py`

```python
class BaseTool:
    name: str = ""              # 工具名（模型调用时用）
    description: str = ""       # 给模型看的说明（决定它会不会调）
    parameters: dict = {}       # JSON Schema（声明参数格式）

    def __init__(self, **kwargs):
        pass   # 允许注册时传入 mm=... 等依赖，不需要的工具直接忽略

    def execute(self, **kwargs) -> str:
        raise NotImplementedError("子类必须实现 execute()")
```

**知识点补充**：`__init__(self, **kwargs)` 是关键设计。注册时 `ToolRegistry` 会往所有工具里塞 `**deps`（比如 `mm=MemoryManager` 实例）。不需要依赖的工具（天气、计算器）就靠这个 `**kwargs` 把多余参数"吞掉"，不会因为多传了个 `mm` 就报错。`name` / `description` / `parameters` 三个类属性，是给 LLM 看的"说明书"——`description` 写得好不好，直接决定模型会不会调这个工具。

### 4.2 动态注册表 `tools/registry.py`

```python
# 读 config.json 动态注册工具
import json, importlib
from .base import BaseTool

class ToolRegistry:
    def __init__(self, config_path: str, deps: dict = None):
        self._deps = deps or {}          # 依赖注入：{"mm": MemoryManager实例}
        self._tools = {}
        with open(config_path, encoding="utf-8") as f:
            self._config = json.load(f)

    def register_all(self):
        """按 config.json 的 tools 列表加载所有工具"""
        for item in self._config.get("tools", []):
            mod = importlib.import_module(item["module"])   # 按路径导入模块
            cls = getattr(mod, item["class"])                # 取类
            tool = cls(**self._deps)                        # 实例化（注入依赖）
            self._tools[tool.name] = tool
        return self

    def get_schemas(self): 
        """给 LLM 的说明书，返回 openai 可读的格式"""
        return [{"type": "function", "function": {
            "name": t.name, "description": t.description, "parameters": t.parameters,
        }} for t in self._tools.values()]

    def execute(self, name: str, args: dict) -> str:
        if name not in self._tools:
            return f"未注册的工具: {name}"
        return self._tools[name].execute(**args)
```

**逐行拆解**：

- `register_all` 是**配置驱动**的典范。加一个新工具，只需：① 写 `tools/xxx.py` 定义 `XxxTool`；② 在 `config.json` 的 `tools` 里加一行 `{"module":"tools.xxx","class":"XxxTool"}`。不用改任何注册代码。`importlib.import_module` + `getattr` 把字符串变成真正的类，再 `cls(**self._deps)` 实例化。
- `get_schemas` 把每个工具的 `name/description/parameters` 包成 OpenAI Function Calling 要的 JSON 格式。**这就是"让模型知道有哪些工具可用"的关键一步**——`fc_loop` 会把它的返回值传给 `chat_for_tools`。
- `deps={"mm": mm}` 是**依赖注入（DI）**：`save_preference` 和 `recall_memory` 需要访问记忆，于是注册时把 `MemoryManager` 实例塞进去。工具自己不负责创建记忆管理器，由外面"注入"，解耦且便于测试。

### 4.3 带"自愈"的执行器 `tools/executor.py`

```python
# 出错重试 / 参数修复（跨工具通用）
from __future__ import annotations

import time

class ToolExecutor:
    def __init__(self, registry:ToolRegistry, max_retries=2):
        self._registry = registry
        self._max_retries = max_retries

    def execute_with_healing(self, name: str, args: dict) -> str:
        last_err = None  # 记录"最后一次错误"，备用
        for attempt in range(self._max_retries + 1):
            try:
                return self._registry.execute(name, args)   # 试执行工具
            except TypeError as e:                     # 模型给错参数键，修复重试
                args = self._repair_args(name, args, e) # 修复参数再试
                last_err = e
            except Exception as e:                    # 其他异常 → 重试
                last_err = e
                time.sleep(1)
        return f"[自愈失败] 工具 {name} 重试 {self._max_retries} 次仍失败: {last_err}"

    def _repair_args(self, name, args, err):
        """把模型给的别名参数映射回工具要的键名"""
        alias = {"user_fact": "fact", "preference": "fact",
                 "question": "query", "keyword": "query"}
        return {alias.get(k, k): v for k, v in args.items()}
```

**逐行拆解**：

- 第 2 行 `from __future__ import annotations` 是修过的一个 bug。原来的代码里 `ToolRegistry` 类型注解在 `ToolExecutor.__init__` 里用到了，但 `ToolRegistry` 这个类在 `registry.py` 里定义、导入顺序导致 `NameError`。加上这行后，类型注解被**延迟求值**（变成字符串），不再在导入时就真去解析 `ToolRegistry`，报错消失。这是 Python 3.10+ 处理"循环/前向引用"的标准姿势。
- `execute_with_healing` 实现了**自愈**：模型有时候会把参数名搞错（比如该传 `fact` 却传了 `user_fact`）。`TypeError` 被捕获后，`_repair_args` 用别名表把参数键"翻译"回工具要的名字再重试。这是让 Agent 在模型不完美时依然能跑通的实用技巧。

**铁律 4**：自愈只修"参数键名错配"这种廉价错误，不掩盖真正的逻辑 bug。重试耗尽会明确返回 `[自愈失败]`，而不是假装成功——这点很重要，否则你永远不知道工具其实没执行。

### 4.4 四个工具的实现

天气（真实 HTTP 调用，带天气码字典）：

```python
# tools/weather.py（节选核心）
class WeatherTool(BaseTool):
    name = "get_weather"
    description = "查天气（按经纬度），返回天气、温度、湿度、日出日落、降雨概率"
    parameters = {
        "type": "object",
        "properties": {
            "latitude": {"type": "number", "description": "纬度"},
            "longitude": {"type": "number", "description": "经度"},
        },
        "required": ["latitude", "longitude"],
    }
    def execute(self, latitude: float, longitude: float) -> str:
        try:
            url = (f"https://api.open-meteo.com/v1/forecast?latitude={latitude}"
                   f"&longitude={longitude}&current=...&daily=...&timezone=Asia%2FShanghai")
            data = json.loads(urllib.request.urlopen(url).read())
            # ... 把 weather_code 翻成中文描述，拼成 JSON 返回
        except Exception as e:
            return f"查天气失败: {e}"
```

计算器（**安全 eval**，只放行四则运算 AST 节点，防代码注入）：

```python
# tools/calculator.py（节选核心）
class CalculatorTool(BaseTool):
    name = "calculator"
    description = "四则运算（如 1+2*3）"
    parameters = {
        "type": "object",
        "properties": {"expr": {"type": "string", "description": "数学表达式"}},
        "required": ["expr"],
    }
    def execute(self, expr: str) -> str:
        try:
            tree = ast.parse(expr, mode="eval")
            # 只允许数字和四则运算，防止模型注入危险代码
            allowed = (ast.Expression, ast.BinOp, ast.UnaryOp, ast.Constant,
                       ast.Add, ast.Sub, ast.Mult, ast.Div, ast.USub)
            for node in ast.walk(tree):
                if not isinstance(node, allowed):
                    return f"表达式包含不允许的语法: {type(node).__name__}"
            result = eval(compile(tree, "<string>", "eval"),
                          {"__builtins__": {}}, {})
            return str(result)
        except Exception as e:
            return f"计算失败: {e}"
```

**知识点补充**：`CalculatorTool` 用的是 `ast` 白名单，而不是裸 `eval(expr)`。裸 `eval` 能让模型注入 `os.system("rm -rf /")` 这种东西。这里先用 `ast.parse` 把表达式解析成语法树，逐个节点检查是不是在 `allowed` 白名单里（只有数字、加减乘除），不在就拒绝。最后 `eval(..., {"__builtins__": {}}, {})` 还把内置函数全清空。这是**"让 LLM 能算数但不能搞破坏"的标准做法**。

保存偏好 / 召回记忆（依赖注入 `mm` 的范例）：

```python
# tools/save_preference.py
class SavePreferenceTool(BaseTool):
    name = "save_preference"
    description = "当用户表达稳定偏好/事实时调用，跨会话记住"
    parameters = {
        "type": "object",
        "properties": {"fact": {"type": "string", "description": "要记住的用户事实"}},
        "required": ["fact"],
    }
    def __init__(self, mm=None):
        self.mm = mm          # 依赖注入：MemoryManager 由外面传入
    def execute(self, fact: str) -> str:
        if self.mm is None:
            return "错误：记忆管理器未注入"
        self.mm.add({"role": "user", "content": fact}, persist=True)  # 落长期
        return f"已保存到长期记忆 ✓"

# tools/recall_memory.py
class RecallMemoryTool(BaseTool):
    name = "recall_memory"
    description = "需要回顾用户历史偏好/事实时调用"
    parameters = {
        "type": "object",
        "properties": {"query": {"type": "string", "description": "检索关键词"}},
        "required": ["query"],
    }
    def __init__(self, mm=None):
        self.mm = mm
    def execute(self, query: str) -> str:
        if self.mm is None:
            return "错误：记忆管理器未注入"
        facts = self.mm.recall(query, k=3)          # 召回结果拼回上下文
        return "；".join(facts) if facts else "（没有找到相关记忆）"
```

**逐行拆解**：这俩工具展示了**依赖注入的实际落点**。`__init__(self, mm=None)` 接收外面传进来的 `MemoryManager` 实例，`execute` 里就能直接 `self.mm.add(...)` / `self.mm.recall(...)`。注意 `save_preference` 调 `add(..., persist=True)`——对应 3.5 里"只有 persist 才落长期记忆"，于是这条偏好就进了 Chroma 向量库，下次会话还能回忆起来。

**一句话记住**：工具层 = 一个基类（统一接口）+ 一个注册表（配置驱动、DI 注入 mm）+ 一个自愈执行器（修参数键名错配）+ 四个干活的工具。

---

## 五、Agent 层：Function Calling 循环 + 路由器 + 四种推理模式

这一层是"大脑的决策中枢"。

### 5.1 Function Calling 主循环 `agent/core.py`

```python
import json
from .modes import BUILDERS

def fc_loop(llm, messages, registry, executor, max_rounds=3):
    """
    Function Calling 主循环
    1. llm.chat_for_tools 返回 message 对象 → 能拿到 tool_calls
    2. registry.get_schemas() 从 config.json 动态生成 schema
    3. executor.execute_with_healing 带自愈
    """
    tool_records = []
    for _ in range(max_rounds):
        msg = llm.chat_for_tools(messages, registry.get_schemas())
        if not msg.tool_calls:
            # 模型没调工具 → 返回答案 + 记录
            return {"answer": msg.content, "tool_records": tool_records}
        messages.append(msg)
        for tc in msg.tool_calls:
            try:
                args = json.loads(tc.function.arguments)
            except Exception as e:
                messages.append({"role": "tool", "tool_call_id": tc.id,
                                 "content": f"参数解析失败:{e}"})
                continue
            result = executor.execute_with_healing(tc.function.name, args)  # 自愈执行
            tool_records.append({"tool": tc.function.name, "args": args, "result": result})
            messages.append({"role": "tool", "tool_call_id": tc.id, "content": str(result)})
    return {"answer": "达到最大工具轮数", "tool_records": tool_records}

def get_response(msgs, llm, **kw):
    """调 LLM 生成回复"""
    return llm.chat(msgs, **kw)

def run_mode(mode, question, llm, builders=None, **kw):
    """按模式调度：查注册表 → 构造消息 → 调 LLM"""
    builders = builders or BUILDERS
    if mode not in builders:
        raise ValueError(f"未知模式:{mode},可选 {list(builders)}")
    msgs = builders[mode](question, **kw)
    return get_response(msgs, llm, **kw)
```

**知识点补充（Function Calling 循环）**：这是整个 Agent "会办事"的核心。流程是：

1. 把"当前消息 + 工具清单"发给模型（`chat_for_tools`）；
2. 模型如果返回 `tool_calls`，说明它想调工具——把模型这步的 `msg` 先塞回 `messages`（**必须把模型的 tool_calls 消息留在上下文里，否则模型看不到自己刚才的决定**）；
3. 逐个 `tool_calls` 解析参数 → `executor` 执行 → 把结果以 `role:"tool"` 消息塞回；
4. 带着"工具结果"再问一次模型，循环最多 `max_rounds` 轮，直到模型不再调工具、直接给答案。

**铁律 5**：`messages.append(msg)`（第 18 行）不能省。Function Calling 是"多轮对话"范式，模型返回的带 `tool_calls` 的 assistant 消息必须原样留在 `messages` 里，下一轮模型才能把 `role:"tool"` 的结果对应上。漏了这步，模型会反复"决定调工具但永远拿不到结果"。

### 5.2 路由器 `agent/router.py`

```python
# 路由：选模型 + 选推理模式
def route_model(question: str, default: str = "deepseek") -> str:
    """从用户输入里找模型名,找不到用默认"""
    for name in ["kimi", "deepseek", "qwen", "gpt"]:
        if name in question.lower():
            return name
    return default

def auto_select(question: str, llm=None) -> str:
    """
    判断问题适合哪种推理模式
    升级可选：让 LLM 输出 JSON 再解析，比关键词匹配更稳（防格式漂移）
    """
    if llm is not None:
        text = (f"判断下面用户的问题适合哪种回答，只输出以下之一：cot / few_shot / react / tot。"
                f"规则：需要动手查/调工具用 react；需要多方案权衡用 tot；"
                f"需要解释原因用 cot；需要举例对比用 few_shot。\n问题:{question}")
        out = llm.chat([{"role": "user", "content": text}]).lower()
        for mode in ["tot", "react", "few_shot", "cot"]:
            if mode in out:
                return mode

    q = question.lower()
    if any(k in q for k in ["帮我", "执行", "查询", "搜索"]):
        return "react"
    if any(k in q for k in ["如果", "方案", "计划"]):
        return "tot"
    if any(k in q for k in ["比较", "对比", "哪个"]):
        return "few_shot"
    return "cot"   # 默认
```

**知识点补充**：`auto_select` 有两套策略——**有 LLM 时用 LLM 判断**（让模型输出 `cot/few_shot/react/tot` 之一，最准），**没 LLM 时退化为关键词匹配**（"帮我/查询"→react，"方案/计划"→tot，兜底 cot）。这就是"能换脑子"的实现：同一个问题，根据类型走不同推理模式。

### 5.3 四种推理模式 `agent/modes/`

四种模式本质都是"**构造一套 messages**"，区别在 system 提示词和是否带示例：

```python
# agent/modes/cot.py —— 思维链：让模型一步步推
def cot_prompt(question: str) -> list[dict]:
    return [
        {"role": "system", "content": "你是擅长逐步推理的专家,请一步一步思考,先列思路再给结论。"},
        {"role": "user", "content": question},
    ]

# agent/modes/few_shot.py —— 少样本：给例子让它学格式
DEFAULT_EXAMPLES = [
    {"question": "我明天下午3点约了张医生复诊",
     "answer": "时间:明天下午3点;人物:张医生;事项:复诊"},
    {"question": "周五晚上和李总在望江楼吃饭",
     "answer": "时间:周五晚上;人物:李总;事项:吃饭"},
]
def few_shot_prompt(question, examples=DEFAULT_EXAMPLES,
                    system_prompt="你是擅长信息抽取的助手"):
    msgs = [{"role": "system", "content": system_prompt}]
    for ex in examples:
        msgs.append({"role": "user", "content": ex["question"]})
        msgs.append({"role": "assistant", "content": ex["answer"]})
    msgs.append({"role": "user", "content": question})
    return msgs

# agent/modes/react.py —— 工具调用：给个系统提示就完事，真正调工具靠 fc_loop
def react_prompt(question, system_prompt="你是会调用工具的智能助手，"
                 "需要查询/计算/记忆时调用工具，并把工具结果结合进回答。"):
    return [
        {"role": "system", "content": system_prompt},
        {"role": "user", "content": question},
    ]

# agent/modes/tot.py —— 树状思维：三次调用（发散→评审→决策）
def ask(llm, question, system_prompt="你是擅长多种角度对比方案的资深决策专家",
        max_tokens=500):
    msgs = [{"role": "system", "content": system_prompt},
            {"role": "user", "content": question}]
    return llm.chat(msgs, max_tokens=max_tokens)

def tot_solve(question, n=3, llm=None):
    """三段式：发散 → 评审 → 决策（三次调用）"""
    if llm is None:
        raise ValueError("tot 模式需要注入 llm")
    schemes = ask(llm, f"请针对以下问题给出{n}种不同解决思路,编号列出:\n{question}\n",
                  system_prompt="你是擅长发散思维的规划专家", max_tokens=600)
    scores = ask(llm, f"请对以下 {n} 个方案根据实行难度,风险评估逐一打分(1-10分),并各用一句话说明理由:\n{schemes}",
                 system_prompt="你是严格的评审专家,打分要客观", max_tokens=500)
    best = ask(llm, f"综合以下评分,选出最优方案,并给出具体实施步骤:\n{scores}",
               system_prompt="你是决策专家,直接给最终选择", max_tokens=600)
    return schemes, scores, best
```

**知识点补充**：CoT / Few-shot / ReAct / ToT 是 LLM 推理的四种经典范式——

| 模式 | 一句话 | 适合场景 |
|------|--------|----------|
| CoT | 一步步想 | 要解释原因、推导 |
| Few-shot | 照例子学 | 格式固定的抽取/分类 |
| ReAct | 想一步、干一步 | 要查/算/调工具 |
| ToT | 多方案打分选优 | 需要权衡决策 |

`BUILDERS` 字典（`modes/__init__.py`）把它们按名字收口，于是 `run_mode("cot", q, llm)` 一行就能切换模式。

**一句话记住**：Agent 层 = 一个 fc_loop（Function Calling 循环）+ 一个路由器（选模型/选模式）+ 四种"造消息"的范式。

---

## 六、组装口 `agent/main.py`：把四层粘起来

```python
# main.py —— 组装所有层
import sys
from pathlib import Path

# 让 agent/main.py 直接运行时也能找到项目根目录的 llm/memory/tools 包
PROJECT_ROOT = Path(__file__).parent.parent
sys.path.insert(0, str(PROJECT_ROOT))

from llm import RealLLM
from memory import MemoryManager
from tools import ToolRegistry, ToolExecutor
from agent import fc_loop, auto_select

llm = RealLLM("deepseek")
mm = MemoryManager(window=8, max_tokens=1500, keep=4, llm=llm)
registry = ToolRegistry(str(PROJECT_ROOT / "config.json"), deps={"mm": mm}).register_all()
executor = ToolExecutor(registry)

while True:
    u = input("你: ")
    if u.lower() == "exit":
        break
    mm.add({"role": "user", "content": u}, persist=u.startswith("记住:"))
    ctx = mm.get_context(query=u)
    out = fc_loop(llm, ctx, registry, executor)   # 返回 {"answer", "tool_records"}
    mm.add({"role": "assistant", "content": out["answer"]})
    mm.maybe_compress()
    print("Agent:", out["answer"])
```

**逐行拆解**：

- 第 6-7 行 `sys.path.insert(0, str(PROJECT_ROOT))` 是个**修过的坑**。直接 `python agent/main.py` 运行时，Python 的模块搜索路径默认是 `agent/` 而不是项目根，于是 `from llm import ...` 会报 `ModuleNotFoundError: No module named 'llm'`。把项目根插进 `sys.path` 就解决了。
- 第 16 行 `deps={"mm": mm}` 是把 `MemoryManager` 实例注入工具层的关键一步——这样 `save_preference` / `recall_memory` 才有记忆可用。
- 主循环（19-28 行）是整条链路的"呼吸"：`add 用户消息`（以"记住:"开头就落长期）→ `get_context`（短期窗口 + 长期召回拼接）→ `fc_loop`（可能调工具）→ `add 助手消息` → `maybe_compress`（超阈值才压）。每一轮都完整走一遍"记→取→想→答→压"。

**铁律 6**：`persist=u.startswith("记住:")` 是个巧妙的约定——用户说"记住:我喜欢用 DeepSeek"时，这句话带着"记住:"前缀，于是 `mm.add(..., persist=True)` 把它存进长期向量库；普通闲聊不落长期，省 embedding 开销。

---

## 七、知识点总盘点

把全文的知识点收个表，方便你回看：

| 知识点 | 落在哪 | 一句话 |
|--------|--------|--------|
| 分层架构 | 四目录 | 换一层不碰其他层 |
| OpenAI 兼容客户端 | `real_llm.py` | 一个 client 通吃各家模型 |
| `.env` + `override=True` | `real_llm.py` | 文件密钥优先于系统环境变量 |
| `chat` vs `chat_for_tools` | `real_llm.py` | 要 tool_calls 必须拿完整 message 对象 |
| 流式生成器 `yield` | `stream_llm.py` | 边生成边打印，过滤空 chunk |
| 配置与密钥分离 | `config.json` / `.env` | 代码里永不写密钥 |
| 滑动窗口 / 裁剪 | `shortmemory.py` | 取最近 N 条 / 按 token 硬砍老的 |
| 长期向量记忆(RAG 召回) | `long_term.py` | Chroma 落盘，相似度检索用户事实 |
| 滚动摘要压缩 | `compress.py` | 旧摘要+新原文→永远一条新摘要 |
| 中文 token 估算 | `compress.py` | 1 字≈1 token，别乘 0.6 |
| 隐私字段 `private` | `long_term.py` | 敏感事实存本地、不自动进 prompt |
| 依赖注入 DI | `registry.py`/`base.py` | 外面把 mm 塞进工具，解耦 |
| 配置驱动注册 | `registry.py` | 加工具只改 config.json |
| 执行器自愈 | `executor.py` | 修参数键名错配，重试兜底 |
| `from __future__ import annotations` | `executor.py` | 延迟类型求值，解前向引用报错 |
| 安全 eval(白名单 AST) | `calculator.py` | 只放行四则运算，防注入 |
| Function Calling 循环 | `core.py` | 模型调工具→执行→结果回灌→再问 |
| 必须回灌 assistant tool_calls 消息 | `core.py` | 否则模型拿不到工具结果 |
| 四种推理范式 | `modes/` | CoT / Few-shot / ReAct / ToT |
| `sys.path` 注入项目根 | `main.py` | 解决直接运行找不到包 |

---

## 八、踩坑记合集（ debugging 实录）

> 这一节把项目从"跑不起来"到"全功能可用"期间踩的坑一次性列出来，每一条都是真金白银换的。

1. **`ModuleNotFoundError: No module named 'llm'`**
   直接 `python agent/main.py` 时 Python 找不到项目根的 `llm` 包。→ 在 `main.py` 开头 `sys.path.insert(0, PROJECT_ROOT)`；同时补 `agent/__init__.py` 暴露 `fc_loop`/`auto_select`。

2. **`SyntaxError: f-string: unmatched '['`（`compress.py`）**
   Python 3.10 的 f-string 里不允许嵌套双引号。→ 把内层双引号改成单引号 `f"{m['role']}:{m['content']}"`。

3. **`ModuleNotFoundError: No module named 'openai'`**
   机器上有多个 Python，跑脚本的解释器没装依赖。→ 用对应解释器的 `python -m pip install openai chromadb python-dotenv`。

4. **`NameError: name 'ToolRegistry' is not defined`（`executor.py`）**
   类型注解里前向引用了还没解析的类。→ 文件头加 `from __future__ import annotations`。

5. **`TypeError: WeatherTool() takes no arguments`**
   工具实例化时被注入了 `mm=...`，但基类没接收。→ 给 `BaseTool` 加 `def __init__(self, **kwargs): pass`。

6. **`httpx.ConnectTimeout`（Chroma 下载 embedding 模型）**
   本地没代理，Chroma 联网下 `all-MiniLM-L6-v2` 超时。→ 走 Clash 代理(7890)把模型下到本机缓存，之后离线可用。

7. **`openai.AuthenticationError: 401`（key 不对）**
   系统环境变量里有个旧的 `DEEPSEEK_API_KEY` 顶掉了 `.env` 里的。→ `load_dotenv(..., override=True)`，让文件里的密钥优先级最高。

8. **`AttributeError: 'RealLLM' object has no attribute 'summarize'`**
   `compress.py` 调了 `llm.summarize`，但 `RealLLM` 没这个方法。→ 在 `RealLLM` 里补 `summarize`。

9. **压缩永远不触发**
   `_estimate_tokens` 原写 `len(text)*0.6`，中文被严重低估，要攒 2500+ 字才压。→ 改成 `max(1, len(text))`，2500 字立刻触发。

10. **长期记忆里混进了脏数据（跨会话旧事实）**
    调试期存过测试事实，污染了召回。→ 删掉 `mem_store/` 目录，下次运行自动重建。

---

## 九、总结

`D:\agent_project` 麻雀虽小，五脏俱全。它用**四层架构**把一个能"记事儿、会调工具、能换脑子"的 Agent 讲清楚了：

- **LLM 层**把模型抽象成可替换零件，密钥与模型配置分离；
- **记忆层**用短期窗口 + 长期向量库 + 滚动摘要，解决了"记性"问题，还顺手做了隐私分流；
- **工具层**用配置驱动注册 + 依赖注入 + 自愈执行器，让 Agent 安全、可扩展地"干活"；
- **Agent 层**用 Function Calling 循环把"想+干"串起来，用路由器在四种推理范式间切换；
- **`main.py`** 把这一切粘成一个能对话的循环。

**一句话记住**：Agent 不是某个神奇模型，而是"LLM（脑子）+ 记忆（记性）+ 工具（手脚）+ 循环调度（决策）"四件套的工程化组装——分层清晰，每一层都能单独换、单独测、单独升级。

> 项目路径：`D:\agent_project` ｜ 备份：`D:\agent_project_backup_20260821`
