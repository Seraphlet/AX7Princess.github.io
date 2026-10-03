---
description: ""
title: "Manager"
draft: false
date: "2026-08-19T15:48:04+08:00"
slug: "memory-manager"
categories:
 - Agent
tags:
 - memory
image: ""
---

# 给 AI Agent 的记忆装个"调度台"：MemoryManager 为什么"薄"才是对的

前两天我们分别造好了短期窗口、长期档案、上下文压缩三块积木。今天把它们拼起来——但拼的方式不是"再写一大坨逻辑"，而是放一个"调度台"上去。这一课比跑通测试更值钱的，是藏在最后那条报错里的一句话：**压缩没触发不是 bug，是没到压缩点。**

## 一、先搞清楚：为什么需要一个调度台

前两天我们写了三个能独立干活的模块：

| 模块 | 干啥的 | 形态 |
|------|--------|------|
| `ShortTermMemory` | 短期窗口，记最近 N 条对话 | 内存里的 `buffer` 列表 |
| `LongTermMemory` | 长期档案，按语义召回用户事实 | Chroma 向量库 |
| `compress` | 上下文压缩，老历史浓缩成一条摘要 | 纯函数 |

三块积木各自都能跑。但一个真正的对话 Agent 需要回答一个问题：**"此刻该让谁上场？"**

- 用户说了句重要的事，要不要落进长期档案？还是只在短期窗口待一会儿？
- 这轮要回复了，要不要去长期档案里翻翻有没有相关的？
- 窗口是不是太长了，要不要压缩？

这些"决策"如果散落在业务代码里，每接一个场景（测试、对话、demo）都得重写一遍接线。所以今天加一层 `MemoryManager`——它是**总开关**，把三个模块的插头接到一个开关上。

一句话记住：**模块是电器，MemoryManager 是接线板。接线板本身不生产电，但没它，电器各干各的。** 它的代码"薄"不是偷懒，而是因为它本就不该有逻辑——逻辑都在三个模块里，它只负责"何时让谁干活"。

---

## 二、热身：两个视角的文件，别被"蒙"

上手整合时最容易懵：为啥 `test_manager.py` 和 `chat_with_memory.py` 看起来"长得不像同一套东西"？因为**它们俩角色完全不同**：

```
┌─────────────────────────────────────────────────┐
│  test_manager.py（质检员）                        │
│  目的: 不接真 LLM, 用假数据验证 MemoryManager     │
│        这个"管家"的 5 个功能没写错                 │
│  特点: 全是 assert(断言), 没有真对话               │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  chat_with_memory.py（真店员）                    │
│  目的: 把管家接进真正的对话循环, 用户一句句聊       │
│  特点: input() 接收输入, 调真 LLM 回答             │
└─────────────────────────────────────────────────┘
```

> 类比：**汽车出厂前**。`test_manager.py` 是质检员在车间里"踩油门、转方向盘、测刹车"（不真上路）；`chat_with_memory.py` 是车交付后你真正开上路。

### 质检员在查什么（test_manager.py 的 5 项检查）

整个测试就一句话：**"管家，我考考你——存了的能想起来吗？窗口会爆吗？"**

```
开始
 │ 造一个管家 mm（窗口=8条, 超2000 token才压缩）
 ├─① mm.add(system)              → 交代人格"你是客服小M"
 ├─① mm.add("我叫小明喜欢蓝色", persist=True)
 │                               → 这句"重要", 抄进长期档案室
 ├─② 聊 20 轮, 每轮都 mm.maybe_compress()
 ├─③ mm.get_context("用户叫什么名字?")
 │      assert "小明" 在里面  → 检查①: 长期召回生效(没失忆)
 ├─④ assert buffer里没有"已知用户长期事实"
 │      → 检查②: 查完档案不留痕迹(不污染工作台)
 ├─⑤ assert 摘要条数==1         → 检查③: 压缩生效, 摘要恒1条
 ├─⑥ 新建 mm2（模拟重启）
 │     assert mm2.recall("用户喜欢什么颜色?") 有"蓝色"
 │      → 检查④: 关电脑再开, 档案室的东西还在
 └─⑦ assert system提示还在 buffer → 检查⑤: 人格永留

终点 → 5 项全过 → ✅
```

**为什么全用假数据 + assert？** 因为要在"没有 LLM、不花钱、可重复"的情况下验证管家逻辑。真实对话里 LLM 的回答不可控，没法断言"模型答对没"；但"该不该召回、该不该压缩、摘要几条"是**确定的逻辑**，可以用断言死死卡住。

### 真店员的一天（chat_with_memory.py 的循环）

```
while True（营业中）
  ├─ u = input("你: ")            → 顾客说话
  └─ bot.chat(u)（服务一次 = 5 步）
       ├─ ① mm.add(用户消息)       → 先记在工作台便签(短期窗口)
       ├─ ② ctx = mm.get_context(u)→ 管家去档案室翻相关事实, 拼进上下文
       ├─ ③ resp = llm.chat(ctx)   → 把【档案+最近对话】交给 LLM 回答
       ├─ ④ mm.add(助手回复)       → 回答也记进便签
       └─ ⑤ mm.maybe_compress()    → 每轮后问: 便签太多了吗? 超了就浓缩
```

**对比你之前写的 LLM 对话**：以前是 `messages.append(...)` 自己管列表，现在**列表变成了管家的 buffer**——你只通过 `mm.add`（记）和 `mm.get_context`（取）间接操作。这就是"最小侵入"接法：把"取上下文"这一步换成管家来取，业务循环几乎不用改。

---

## 三、MemoryManager 到底在干嘛：就三个"决定"

| 方法 | 它做的"决定" | 问自己的问题 |
|------|--------------|--------------|
| `add(msg, persist)` | 这条消息**只进短期窗口，还是也要落长期档案**？ | "这是重要事实吗？" |
| `get_context(query)` | 现在**要不要去档案室召回**？召回结果放哪？ | "现在要用以前的资料吗？" |
| `maybe_compress()` | 工作台**满了吗**？满了才压缩 | "是不是该收拾了？" |

关键认知：**`MemoryManager` 代码"薄"是它正确的标志**。它不该有逻辑，逻辑都在三个模块里，它只做"决定"。你读它时看不到"拧螺丝"的动作，只看到"调用"——这不是空，是**项目经理的活**。

> 类比：以前你是**自己开餐厅**（自己记账、自己端菜）；现在你**雇了经理**（`MemoryManager`），你只需要说"客人来了"（`add`）、"上菜"（`chat`）。经理怎么查 VIP 档案、怎么整理库存，你不需要插手。

---

## 四、代码长什么样（逐块讲解，最后放完整代码）

`memory/manager.py` 一共一个类，下面逐方法拆，文末给完整源码。

### 顶部：导入 + 稳定 ID 生成器

```python
from memory.compress import compress, maybe_compress as _auto_compress
from memory.shortmemory import ShortTermMemory
from memory.long_term import LongTermMemory
import hashlib

def _fid(text: str) -> str:
    """内容哈希生成稳定ID：不重复、可跨进程（比 hash() 稳，hash有随机盐）"""
    return "f" + hashlib.md5(text.encode("utf-8")).hexdigest()[:12]
```

四个 import，正好对应"三个电器 + 一个压缩纯函数"。`_fid` 用 `md5(内容)` 的前 12 位当事实 ID——同一个事实内容永远得到同一个 ID，**跨进程也稳定**。这点很重要：Python 内置的 `hash()` 每次进程启动都带随机盐（PYTHONHASHSEED），同一个字符串在不同进程里 hash 值不同，做长期事实的 ID 会乱套。所以用 `md5` 给内容一个"指纹"。

### `__init__` —— 把三个电器接上电

```python
class MemoryManager:
    def __init__(self, window=8, max_tokens=2000, keep=4, llm=None):
        self.stm = ShortTermMemory(window=window, maxtokens=max_tokens)  # 短期
        self.ltm = LongTermMemory()                                     # 长期(Chroma)
        self.keep, self.max_tokens, self.llm = keep, max_tokens, llm
```

三个参数全透传给底层：`window`/`keep` 给短期窗口，`max_tokens` 给压缩阈值，`llm` 是压缩时用的摘要器。调度台自己不存任何业务状态，状态都在 `stm` 和 `ltm` 里。

### `add` —— 客人说的话，记哪？

```python
def add(self, msg: dict, persist: bool = False):
    """① 永远进短期窗口; 只有明确用户事实才落长期(省嵌入开销)"""
    self.stm.add(msg)
    if persist and msg.get("role") == "user":
        self.ltm.add_fact(msg["content"], fid=_fid(msg["content"]))
```

两个动作，决策点在 `persist`：

- `self.stm.add(msg)`——**永远执行**，先记在工作台便签（短期窗口）。
- 只有当 `persist=True` **且** 是用户消息时，才 `ltm.add_fact(...)` 抄进长期档案。

**为什么 persist 默认 False？** 因为每次落长期都要算 embedding（花钱花时间）。普通闲聊不值得进档案室，只有明确"这是用户事实"才值得。这行决策让"落长期"成为有意识的选择，而不是默认行为。

### `get_context` —— 上菜前，先查 VIP 档案？

```python
def get_context(self, query: str = None) -> list[dict]:
    """② 取上下文: 有query才召回; 召回结果临时拼接, 不污染buffer(防重复累加)"""
    base = self.stm.context()          # [sys提示...] + [窗口内消息]
    if not query:
        return base
    facts = self.ltm.recall(query, k=3)
    if not facts:
        return base
    facts_msg = {"role": "system",
                 "content": "已知用户长期事实: " + " | ".join(facts)}
    sys_msgs  = [m for m in base if m.get("role") in self.stm.system_roles]
    others    = [m for m in base if m.get("role") not in self.stm.system_roles]
    return sys_msgs + [facts_msg] + others   # 系统提示 → 长期事实 → 窗口
```

三个决策：

1. **没 query？** 直接返回 `base`，连档案室都不进（省一次检索）。
2. **召回到了吗？** 没相关事实也返回 `base`，绝不硬塞。
3. **拼在哪？** 召回结果放在 `[系统提示]` 和 `[窗口对话]` 之间，形成"系统提示 → 长期事实 → 最近对话"的结构。

**面试考点就在这**：召回结果是**临时拼进去的，不写回 buffer**。如果写回，每次 `get_context` 都会往 `buffer` 加一条 system，多轮后越堆越多——这就是需求文档里明说的"召回结果要防重复累加"。`buffer` 里只有"真实发生的对话 + 压缩摘要"，长期事实只是"借来用一下"就还回去了。

### `maybe_compress` —— 打烊前，工作台满了吗？

```python
def maybe_compress(self, llm=None):
    """③ 超阈值才压; 压完写回buffer(下次context取到压缩后结果)"""
    llm = llm or self.llm
    if llm is None:
        return self.stm.buffer
    msgs = _auto_compress(self.stm.buffer, llm,
                          max_tokens=self.max_tokens, keep=self.keep)
    if msgs is not self.stm.buffer:      # 真的压缩了才写回
        self.stm.buffer = msgs
    return self.stm.buffer
```

注意它**调用了 `compress` 模块里的 `_auto_compress` 纯函数**，自己只做两件事：

1. 处理 `llm` 兜底（调用方没传就用初始化时的）。
2. 判断"是不是真的压了"——`compress` 函数没超阈值时会**原样返回同一个 list 对象**，所以 `msgs is not self.stm.buffer` 是 False，不写回；只有真的压了才把新 list 写回 `buffer`。

**这是个精巧的"零成本判断"**：用 `is`（身份比较）而不是 `==`（值比较）来检测"有没有动过"，避免了无意义的 buffer 重赋值。

### `recall` —— 对外暴露长期检索

```python
def recall(self, query: str, k: int = 3) -> list[str]:
    """④ 对外暴露长期检索(验收: 重启后仍能recall到DeepSeek)"""
    return self.ltm.recall(query, k=k)
```

纯粹的转发——把长期检索能力暴露成管家的公开 API，让外部（比如测试里的"模拟重启"`mm2.recall(...)`）能直接查档案室，而不用知道底层是 Chroma。

### 完整代码

四个方法拼起来就是 `memory/manager.py` 的全部，可直接抄：

```python
# memory/manager.py —— 记忆总调度台：只做决策，不重写逻辑
from memory.compress import compress, maybe_compress as _auto_compress
from memory.shortmemory import ShortTermMemory
from memory.long_term import LongTermMemory
import hashlib

def _fid(text: str) -> str:
    """内容哈希生成稳定ID：不重复、可跨进程（比 hash() 稳，hash有随机盐）"""
    return "f" + hashlib.md5(text.encode("utf-8")).hexdigest()[:12]

class MemoryManager:
    def __init__(self, window=8, max_tokens=2000, keep=4, llm=None):
        self.stm = ShortTermMemory(window=window, maxtokens=max_tokens)  # 短期
        self.ltm = LongTermMemory()                                     # 长期(Chroma)
        self.keep, self.max_tokens, self.llm = keep, max_tokens, llm

    def add(self, msg: dict, persist: bool = False):
        """① 永远进短期窗口; 只有明确用户事实才落长期(省嵌入开销)"""
        self.stm.add(msg)
        if persist and msg.get("role") == "user":
            self.ltm.add_fact(msg["content"], fid=_fid(msg["content"]))

    def get_context(self, query: str = None) -> list[dict]:
        """② 取上下文: 有query才召回; 召回结果临时拼接, 不污染buffer(防重复累加)"""
        base = self.stm.context()          # [sys提示...] + [窗口内消息]
        if not query:
            return base
        facts = self.ltm.recall(query, k=3)
        if not facts:
            return base
        facts_msg = {"role": "system",
                     "content": "已知用户长期事实: " + " | ".join(facts)}
        sys_msgs  = [m for m in base if m.get("role") in self.stm.system_roles]
        others    = [m for m in base if m.get("role") not in self.stm.system_roles]
        return sys_msgs + [facts_msg] + others   # 系统提示 → 长期事实 → 窗口

    def maybe_compress(self, llm=None):
        """③ 超阈值才压; 压完写回buffer(下次context取到压缩后结果)"""
        llm = llm or self.llm
        if llm is None:
            return self.stm.buffer
        msgs = _auto_compress(self.stm.buffer, llm,
                              max_tokens=self.max_tokens, keep=self.keep)
        if msgs is not self.stm.buffer:      # 真的压缩了才写回
            self.stm.buffer = msgs
        return self.stm.buffer

    def recall(self, query: str, k: int = 3) -> list[str]:
        """④ 对外暴露长期检索(验收: 重启后仍能recall到DeepSeek)"""
        return self.ltm.recall(query, k=k)
```

---

## 五、踩坑记：maybe_compress 没压 ≠ bug

这堂课最值钱的一课，藏在昨天那条测试失败里。

一开始你写 `test_e2e.py` 断言"20 轮后 buffer ≤ 11"，结果断言挂了——因为 20 轮短对话**根本没超过 2000 token 阈值**，管家"正确地"没压。`buffer` 里 20 多条全是原文，自然 > 11。

第一次看到这个，本能反应是"压缩坏了"。但真相是：**管家没触发压缩是对的。**

> **"maybe_compress 没压 ≠ bug，而是没到压缩点。"**

这句话面试官超爱听，因为它证明你理解的是**决策逻辑**，不是"会调用函数"。展开说就是：

- 压缩是**阈值驱动**的——`token 数 > max_tokens` 才压，跟"聊了几轮"无关。
- 20 轮如果每句都短，总 token 没超，就不该压；反过来哪怕只聊了 5 轮但每句都很长，也该压。

所以端到端测试要**分两个场景**验证，而不是断言"20 轮后一定被压"：

```python
# 场景1：没超阈值 → 不该压（这"不压"是正确行为）
mm = MemoryManager(window=8, max_tokens=2000, keep=4, llm=FakeSummarize())
... 20 轮 ...
assert len([m for m in mm.stm.buffer if m.get("is_summary")]) == 0
# ✅ 摘要0条 = 正确

# 场景2：阈值调小到300，20轮必超 → 该压（摘要恒1条）
mm2 = MemoryManager(window=8, max_tokens=300, keep=4, llm=FakeSummarize())
... 20 轮 ...
assert len([m for m in mm2.stm.buffer if m.get("is_summary")]) == 1
# ✅ 摘要1条 = 正确
```

**心智模型**：压缩不是"清理工"，是"应急收纳"——没满就别收拾。把断言从"一定会压"改成"该压时压、不该压时不压"，测试才真正卡住了管家的决策边界。

---

## 六、知识点补充：调度层背后的设计

### 1. 调度者模式（Coordinator / Facade）

`MemoryManager` 是典型的**协调者模式**：它不实现具体能力，而是编排已有能力。好处是**边界清晰**——三个小模块的逻辑各管各的，调度层只决定"何时让谁上"。W5（或更后面）把它搬进 LangGraph 的状态节点时，内部逻辑一行不用改，只改"谁调用谁"的接线。

### 2. 同名不同层：`compress.maybe_compress` vs `MemoryManager.maybe_compress`

这俩最容易混，其实层级完全不同：

| 名字 | 身份 | 干什么 |
|------|------|--------|
| `compress.maybe_compress` | **纯函数** | 给它消息列表，它根据 token 数决定压不压、回不回原列表 |
| `MemoryManager.maybe_compress` | **调度方法** | 调用上面的纯函数 + 把结果写回 `self.stm.buffer` |

纯函数不知道"管家"的存在，它只认消息；调度方法知道 buffer，它把纯函数的结果接回状态。职责切得干净，测试能单独测纯函数，也能单独测调度逻辑。

### 3. 临时拼接 vs 写回——信息流的两种走向

| 操作 | 走向 | 风险 |
|------|------|------|
| `get_context` 召回结果 | **临时拼进返回**，不写 buffer | 写回会"重复累加"，buffer 越堆越大 |
| `maybe_compress` 摘要 | **写回 buffer** | 不写回，下次 `context()` 取到的是未压缩的旧列表 |

这两条规则一正一反，正好是记忆系统的"呼吸节奏"：**召回是吸气（借来用，不囤），压缩是呼气（旧的出去，留浓缩版）**。

### 4. 内容哈希做稳定 ID

`hashlib.md5(text).hexdigest()[:12]` 给每条长期事实一个**内容指纹**：

- 同一句话多次 `add`，`fid` 相同 → Chroma 的 `add(ids=[fid])` 会覆盖而非重复，天然去重。
- 跨进程稳定（不像 `hash()` 带随机盐），会话重启后"小明喜欢蓝色"还是同一个 ID，召回一致。

---

## 七、面试一句话复述

> "MemoryManager 是调度者不是实现者——它只做三个决定：`add` 时决定是否落长期（`persist` 默认 `False` 省嵌入开销）、`get_context` 时决定是否召回（有 query 才查，结果临时拼接不污染 buffer 防重复累加）、`maybe_compress` 时决定是否压缩（超阈值才压，压完写回窗口）。它的代码很薄，因为逻辑都在三个小模块里，它只负责'何时让谁干活'。我之前整合时踩过一个坑：断言'20 轮后必然被压'结果挂了——后来明白压缩是阈值驱动不是轮数驱动，没超 `max_tokens` 不压才是正确的。这证明我测的是决策边界，不是函数会被调用。"

---

## 八、总结

1. **MemoryManager 是接线板不是电器**——它不实现记忆能力，只编排短期/长期/压缩三块的调用时机。
2. **代码"薄"是正确标志**——决策层不该有业务逻辑，逻辑都在底层模块。
3. **三个决策点**：`add` 落不落长期、`get_context` 召不召回、`maybe_compress` 压不压。
4. **召回临时拼接、压缩写回 buffer**——一吸一呼，信息流不重复累加。
5. **压缩没触发 ≠ bug**——阈值驱动，没超 `max_tokens` 不压才是正确行为。

给记忆系统装好调度台，三个模块就不再是散兵游勇，而是听一个开关指挥的流水线。你之前觉得"整合没思路"，本质是还在用"工人的眼睛"看"经理的活"——经理的活看起来像什么都没做，但恰恰是它让整条线转起来。

下一步（Day6）：做一个「带记忆的多轮 Agent demo」——跨会话记住偏好（长期）、长对话自动压缩（压缩）、必要时 Function Calling 调工具。到那时你会看到：**没有 MemoryManager，demo 里每个功能都要手动接线；有了它，ChatSession 只需要几行代码。**

---

## 附：远端文件（无需修改，仅参考）

- `D:\Desktop\AgentLearn\memory\manager.py` —— 本文第四节的完整源码
- `D:\Desktop\AgentLearn\test_manager.py` —— 质检员视角，5 项断言验证管家行为
- `D:\Desktop\AgentLearn\test_e2e.py` —— 端到端，分"不压/该压/召回/系统提示永留"四场景验收
