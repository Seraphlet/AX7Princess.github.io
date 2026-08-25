---
description: ""
title: "LangChain_Memory"
draft: false
date: "2026-08-25T07:44:10+08:00"
slug: "LangChain-MemoryManager"
categories:
 - LangChain
 - Agent
tags:
 - Memory
image: ""
---

辛辛苦苦写的 `MemoryManager`（短期窗口 + 长期 Chroma + 滚动压缩），它的 `get_context()` 返回的是一串 **dict 消息**。而 LangChain 的 `prompt` 里 `("placeholder", "{history}")` 需要的，是 **`BaseMessage` 对象**（`HumanMessage` / `AIMessage` / `SystemMessage`）。

这中间隔着一层「格式不对」——dict 直接喂进去，prompt 展开不了，链就哑火。

D3 要做的，就是补上这层转换，让「取记忆」成为链的头一个环节。而补这层只需要三件东西，刚好对应你给的 `lc_memory.py` 里「**入口适配层**」的三大件：

- `_to_lc` = 把 dict 翻成 LangChain 消息对象（格式转换器）
- `lambda` = 一次性匿名函数 + 字典解包（在链首「加料」）
- `RunnableLambda` = 把函数包成能入 `|` 管道的零件（适配器）

> 一句话：**W3 的决策逻辑一行不改，只在外面套一层「适配器」，就能进 LangChain 的链。** 这和你 W3 博客里说的「最小侵入」是一回事。

---

## 一、大局：记忆接在链的哪个位置

先看整条链的数据流向——记忆是「链首的第一道工序」：

```
用户输入 {"question": "我叫什么？"}
   ↓
load_mem (RunnableLambda)   ← ① 给输入"加料"：补上 history 字段
   ↓
prompt (ChatPromptTemplate) ← ② {history} 和 {question} 这下都有值了
   ↓
model → parser              ← ③ 生成回答
```

拆开看三大件各自在哪、干嘛：

| 部件 | 角色 | 一句话 |
|---|---|---|
| `_to_lc(m: dict)` | 格式转换器 | dict 消息 → LangChain `BaseMessage` 对象 |
| `lambda x: {...}` | 一次性函数 | 在链首把 `history` 字段「加」进输入 |
| `RunnableLambda(...)` | 适配器 | 把上面的函数包成能进 `|` 的零件 |

接下来逐个拆。

---

## 二、`_to_lc`：dict → LangChain 消息对象（快递打包）

**一句话：把「散装 dict 消息」翻译成「LangChain 标准消息对象」。**

你的 `MemoryManager.get_context()` 返回的是普通 dict 列表：

```python
[{"role": "user", "content": "我叫小明"},
 {"role": "assistant", "content": "你好小明"}]
```

但 LangChain 的 prompt 里 `("placeholder", "{history}")` 需要的是**消息对象**，不是 dict。所以必须转换：

```python
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")   # 取出角色和内容，role 缺省默认 user
    if role == "system":
        return SystemMessage(content=content)       # system → SystemMessage
    if role == "assistant":
        return AIMessage(content=content)           # assistant → AIMessage
    return HumanMessage(content=content)            # 其他（user）→ HumanMessage
```

> 类比：**快递打包**。你的记忆模块产出的是「散装货物」（dict：一张纸写着「谁说的 + 说了啥」），而 LangChain 的传送带只认「标准包装箱」（`BaseMessage` 对象）。`_to_lc` 就是打包员——按「谁说的」（`role`）选不同规格的箱子（`Human`/`AI`/`System`），把货装进去。

**为什么 `m.get("role", "user")` 要带默认值？** 因为 `get` 是「有就拿，没有给默认」——万一某条消息漏写了 `role`，就按用户消息处理，不崩。这是防御性写法，跟你在 `auto_select` / `MemoryManager` 里用的「宽容解析」一个思路。

**对照自研**：你 W3 写 `compress.py` 时，把消息喂给 LLM 做摘要前，其实也隐含了这一层「dict → 文本」的转换（你当时用了 `_fmt` 拍平成纯文本）。只不过 LangChain 这边的「标准箱」是对象而不是字符串——道理一样：**框架要的是它认得的格式，你得先把货装进它的箱子**。

---

## 三、`lambda` 表达式 + 字典解包（链首加料）

**`lambda` = 一次性匿名函数**。语法只有一句话：`lambda 参数: 返回值表达式`。

```python
# 你的代码里：
lambda x: {**x, "history": get_history(x)}
#    ↑参数    ↑ 返回表达式（一个字典）

# 它等价于这个 def：
def 匿名函数(x):
    return {**x, "history": get_history(x)}
```

> 类比：**便利贴 vs 正式文件**。`def` 是正式文件（有名字、能反复用）；`lambda` 是便利贴——写完就用、用完就扔，适合「只在这一个地方用一次」的小函数。

**为什么用 `lambda` 而不是 `def`？** 因为 `RunnableLambda` 只需要一个「输入→输出」的函数，而这个函数只在这一行用一次，没必要单独起个名字占地方。

**`{**x, "history": ...}` 字典解包是什么？** `**x` 是把字典 `x` 的所有键值对「展开」，然后放进一个新字典：

```python
x = {"question": "我叫什么？"}

{**x, "history": ["..."]}
# 展开 x → {"question": "我叫什么？", "history": ["..."]}
```

**为什么不直接写 `x["history"] = ...`？** 因为那会**修改原字典**（副作用）。链式调用里要保持「纯函数」——输入不被改动，只产出新结果。`{**x, ...}` 是**复制一份再追加**，原输入 `x` 完好无损。

> 类比：**复印并加批注**——原文件不动，复印件上多了一行批注。这正是 D2 学的 `RunnablePassthrough.assign` 干的事，这里用字典解包手动实现了一遍。

所以这一行 `lambda x: {**x, "history": get_history(x)}` 的含义就是：**拿到用户的 `{"question": ...}`，复印一份，再往里塞一个 `history` 字段（内容是记忆系统的召回结果），原样返回给下一个环节**。

---

## 四、`RunnableLambda`：把函数变成能入链的零件（插头转换器）

**一句话：把普通 Python 函数包装成「能参与 `|` 管道的零件」。**

LangChain 的 `|` 传送带有规矩：**每个零件都必须实现 Runnable 协议**（有 `invoke` / `stream` / `batch` 方法）。你的 `get_history()` 是普通函数，没有这些方法，直接放管道里会崩：

```python
# ❌ 直接放普通函数：
chain = get_history | prompt | model | parser   # 报错：get_history 不是 Runnable

# ✅ 包一层 RunnableLambda：
load_mem = RunnableLambda(get_history)
chain = load_mem | prompt | model | parser      # 能跑！
```

> 类比：**插头转换器**。你的函数是「两脚插头」（普通函数），LangChain 的管道是「三脚插座」（Runnable 协议）——直接插不进去，套个转换器（`RunnableLambda`）就通了。

**这你其实在 D2 步骤 E 已经亲手跑过**：`RunnableLambda(TextCleansing)`——当时是给链**尾**加了个「清洗加工站」，现在是给链**首**加了个「加载历史加工站」，**同一个东西，位置不同**。所以 D3 的桥，D2 就已经修好了，你只是把它从链尾挪到了链首。

> ⚠️ 类型对接（跟 D2 踩坑记④一致）：`RunnableLambda` 收到的是管道里**上一个**的输出类型。这里 `load_mem` 是链首，它收到的是你 `.invoke({"question": ...})` 传进去的 **dict**，所以 `lambda x` 的参数 `x` 是 dict，`x.get("question")` 才对得上。如果上一个产出是 str，参数就得是 str。

---

## 五、完整可运行演示（不需要 API key）

先跑这个，把三个概念一次看懂——它不调模型，纯演示「转换 + 加料 + 入链」：

```python
# concepts_demo.py —— 三个概念一次看懂（无需 API key）
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage
from langchain_core.runnables import RunnableLambda

# ---------- 1. _to_lc：dict → 消息对象 ----------
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "assistant":
        return AIMessage(content=content)
    return HumanMessage(content=content)

raw = [{"role": "user", "content": "我叫小明"},
       {"role": "assistant", "content": "你好小明"}]
msgs = [_to_lc(r) for r in raw]
for m in msgs:
    print(type(m).__name__, "→", m.content)
# HumanMessage → 我叫小明
# AIMessage → 你好小明

# ---------- 2. lambda 语法 ----------
f = lambda x: x + "！"               # lambda 参数: 返回表达式
def f_def(x): return x + "！"        # 等价 def
print(f("你好"), "==", f_def("你好"))  # 你好！ == 你好！

# ---------- 3. RunnableLambda + 字典解包 ----------
def get_history(x):
    return [_to_lc(r) for r in raw]   # 返回消息对象列表（就是 _to_lc 的用武之地）

load_mem = RunnableLambda(lambda x: {**x, "history": get_history(x)})

result = load_mem.invoke({"question": "我叫什么？"})
print(result.keys())     # dict_keys(['question', 'history'])
print(result["history"]) # [HumanMessage, AIMessage] —— prompt 的 {history} 就能展开了
```

跑一下，你会看到：**`lambda` 收到 `{"question": ...}`，返回一个新字典，原字段保留、`history` 字段追加**——然后 prompt 里的 `{question}` 和 `{history}` 就都有值了。

---

## 六、`lc_memory.py` 完整代码（真实接 MemoryManager）

下面是把 W3 的 `MemoryManager` 真正接进链的版本。`get_history` 调的是 `_mm.get_context(query=...)`（带长期召回），`_to_lc` 负责格式转换，`load_mem` 用 `RunnableLambda` 包起来放到链首：

```python
# lc_memory.py —— W3 MemoryManager 接进 LCEL 链（入口适配层）
import os
from dotenv import load_dotenv
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnableLambda
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage

load_dotenv()

# 桥的包装器：dict 消息 → LangChain BaseMessage
def _to_lc(m: dict):
    role, content = m.get("role", "user"), m.get("content", "")
    if role == "system":
        return SystemMessage(content=content)
    if role == "assistant":
        return AIMessage(content=content)
    return HumanMessage(content=content)

# 接入 W3 自研 MemoryManager（失败则降级最小窗口版）
# ⚠️ 注意：直接 `python lc_memory.py` 运行时，下面这行相对导入会找不到包——
#    因为直接运行脚本时 Python 把它当顶层模块，无法解析 "..memory" 这类上级路径。
#    正确做法（二选一）：
#      1) 把项目根目录加入 sys.path 再用绝对导入（见下方注释）；
#      2) 用 `python -m 包名.模块名` 启动，而非直接执行脚本文件。
try:
    from memory.memory_manager import MemoryManager
    _mm = MemoryManager(window=6, max_tokens=500)
    def get_history(x):
        # 真实版：query=本轮问题 → 长期召回 + 滚动窗口
        return [_to_lc(m) for m in _mm.get_context(query=x.get("question"))]
except Exception as e:
    print(f"[降级]未找到memory包({e}),使用最小窗口版")
    class _MinilMM:
        def __init__(self, window=6):
            self.buf, self.window = [], window
        def get_context(self, query=None):
            return self.buf[-self.window:]
        def add_user(self, t):
            return self.buf.append({"role": "user", "content": t})
        def add_ai(self, t):
            self.buf.append({"role": "assistant", "content": t})
    _mm = _MinilMM()
    def get_history(x):
        return [_to_lc(m) for m in _mm.get_context()]

# 链：链首选历史，再 prompt | model | parser
model = ChatOpenAI(
    model="deepseek-v4-flash",
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    base_url="https://api.deepseek.com",
)

prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个有记忆的助手，记得之前聊过的内容。"),
    ("placeholder", "{history}"),            # 关键：展开历史 messages
    ("human", "{question}"),
])

load_mem = RunnableLambda(lambda x: {**x, "history": get_history(x)})
chain = load_mem | prompt | model | StrOutputParser()

def ask(question: str) -> str:
    ans = chain.invoke({"question": question})
    if hasattr(_mm, "add_user"):
        _mm.add_user(question); _mm.add_ai(ans)
    else:
        _mm.add({"role": "user", "content": question})
        _mm.add({"role": "assistant", "content": ans})
    return ans

if __name__ == "__main__":
    # 验收：第 2、3 问能答出「篮球」「小明」=> 记忆生效
    for q in ["我叫小明，喜欢打篮球", "我刚说的爱好是什么？", "我名字叫什么？"]:
        print(f"Q: {q}\nA: {ask(q)}\n")

    # 确认 load_mem 在链首 + 看历史在累积
    print("=== 链结构图（load_mem 应排最前）===")
    chain.get_graph().print_ascii()
    print("=== 当前记忆历史 ===")
    for m in _mm.get_context():
        print(f"  {m.get('role')}: {m.get('content')}")
```

`("placeholder", "{history}")` 是 LangChain 专门用来「展开一个消息列表」的占位符——它和 `("human", "{question}")` 不同：后者填的是字符串，前者填的是**消息对象列表**，会被逐条展开成多轮对话历史。这就是 `_to_lc` 存在的意义。

---

## 七、踩坑记 & 知识点

**① 相对导入在「直接运行脚本」时会炸**
`from memory.memory_manager import MemoryManager` 只在「包上下文」里成立。你 `python lc_memory.py` 直接跑，Python 把文件当顶层脚本，`__package__` 为空，解析不了 `..memory` 这种上级路径，就抛 `ModuleNotFoundError`。

两种修法（二选一）：
- **加 sys.path 补丁**：在导入前把项目根目录塞进 `sys.path`，再绝对导入（注释里写了模板）；
- **用 `-m` 启动**：`cd` 到 `AgentLearn\` 后 `python -m memory.lc_memory`（或对应包路径），Python 基于 cwd 找顶层包，自然能 import。

我上面的代码用 `try/except` 做了**优雅降级**——找不到真实 `MemoryManager` 就退回到一个最小窗口版，保证链一定能跑通、你能先看效果。但生产里你还是得用上面两种修法之一把真实版接上。

**② `placeholder` 占位符要配「消息对象列表」，不是字符串**
`{history}` 用 `("placeholder", ...)` 声明，传进去的必须是 `[HumanMessage, AIMessage, ...]` 这种列表。如果你直接传字符串，LangChain 会不知道怎么展开。所以 `_to_lc` 把 dict 翻成对象这步，**不能省**。

**③ `lambda` 里用 `{**x, ...}` 而不是 `x["k"]=...`（纯函数）**
前面讲过：前者复制一份再追加，原输入不动；后者直接改原 dict，会在链式调用里留下副作用，导致后面环节拿到被偷偷改过的输入，极难排查。进 `|` 管道的函数，尽量写成纯函数。

**④ RunnableLambda 的「输入输出类型」要对接上一个 Runnable**
`load_mem` 是链首，收到的是你 `.invoke({"question": ...})` 传的 **dict**，所以 `lambda x` 里 `x` 是 dict、`x.get("question")` 对得上。若把 `load_mem` 挪到链中间（比如接在 `prompt` 后面），它收到的就会是 `prompt` 的输出了——类型全变，得重写函数。改完链先 `get_graph().print_ascii()` 看一眼流向（跟 D1/D2 踩坑记一脉相承）。

**⑤ `_mm.add_user / add_ai` 是降级版的「自造」方法**
真实 `MemoryManager` 的接口是 `add({"role":..., "content":...})`（你 W5 博客里写过）。降级类 `_MinilMM` 是我手写的最小版，用了 `add_user`/`add_ai` 方便演示——所以 `ask()` 里用 `hasattr(_mm, "add_user")` 来区分「降级版」和「真实版」，分别走不同写入路径。这提醒你：**接第三方/自研模块时，先确认它的真实接口长啥样，别凭想象调方法。**

---

## 八、总结

今天把 W3 的 `MemoryManager` 接进了 LangChain 链，靠的是「入口适配层三件套」——**而且 W3 的决策逻辑一行没改，只是外面套了层适配器**：

- **`_to_lc`** = 格式转换器（dict → `BaseMessage`），快递打包员：按 `role` 选箱子
- **`lambda` + `{**x, ...}`** = 链首加料（保留原输入 + 追加 `history`），便利贴 + 复印加批注
- **`RunnableLambda`** = 适配器（普通函数 → 能进 `|` 管道），插头转换器：D2 的加工站从链尾挪到了链首

整条链：`load_mem | prompt | model | parser`——记忆从此成为「链的首道工序」，而不是链外的黑盒。

**这印证了你 W3 博客那句结论：「最小侵入」——框架化不是重写逻辑，是把旧轮子套上标准接口。** 你前三周手写的所有零件，到 W4 一个个「认领」进 LangChain，没有任何一个被推倒重来。

---

## 九、下一篇预告

**D4：把 Retriever / RAG 也接进链（长期记忆的「知识版」）**

今天接的是「对话记忆」（短期窗口 + 你手动落的事实）。但 W3 你还有一个更重的资产——**Chroma 长期向量记忆**，那东西在 LangChain 里叫 `Retriever`。

D4 要做的，是把 `MemoryManager` 里「长期召回」那部分，用 LangChain 的 `Retriever` 标准接口重写一遍——让你 W3 手写的向量召回，变成 `chain` 里的一环。而「对话记忆」和「知识召回」这两种 `history`，会在链首被 `RunnableParallel` / `assign` 一起拼进 prompt。

再往后，**W5 LangGraph** 会把今天这条「线性带记忆的链」升级成「有状态图」：每个零件变成 node，`MemoryManager` 变成图里一个会读写状态的节点。你 W4 这四天铺的路，正好接上 W5 的 node/edge。

> 一句话收尾：**D1-D3 你学会了「零件怎么串、函数怎么进链、记忆怎么接入」；D4 补上「知识怎么召回」，W5 就轮到「流程怎么分叉」。**
