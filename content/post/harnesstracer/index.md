---
description: ""
title: "从 Session、Context Compaction、Tracer 理解 Agent Harness 工程化"
draft: false
date: "2026-09-20T07:52:49+08:00"
slug: "Harnesstracer"
categories:
 - Harnesser
 - LangGraph
tags:
 - Session
 - Context Compaction
 - Tracer
image: ""
---

# W7-D5：从 Session、Context Compaction、Tracer 理解 Agent Harness 工程化

这一天开始，我感觉自己学的东西开始从“LangGraph API”慢慢变成了真正的工程代码。

前面学 LangGraph 的时候，我更关注：

> 一个节点怎么写？
> 一个 State 怎么传？
> 一个 `Send` 怎么并发？
> 一个 `interrupt` 怎么暂停？

但 Agent 真正跑起来以后，很快会遇到另一类问题：

* 程序重启了，之前的对话怎么办？
* 对话越来越长，模型上下文装不下怎么办？
* Agent 明明执行了很多步骤，但出了问题以后根本不知道哪里慢、哪里炸？
* 这些能力如果都塞进 Agent 主逻辑，代码最后会变成什么样？

所以 Day5 做的，其实不是“再学三个 Python 类”。

而是在学习一个更重要的问题：

> **怎么把一个 Agent 从“能跑”，逐渐变成“可以长期运行、可以维护、可以排查问题”的系统。**

Day5 的三个模块分别是：

| 模块                 | 类比     | 解决的问题                  |
| ------------------ | ------ | ---------------------- |
| Session Store      | 游戏存档   | 重启以后状态不丢               |
| Context Compaction | 行李超重打包 | 长任务不会因为上下文越来越大而爆掉      |
| Tracer             | 外卖配送轨迹 | 出问题以后知道谁调用了谁、花了多久、哪里失败 |

而整个 Harness 的整体设计里，它们分别属于第 3、4、5 个模块。

---

# 一、先不要看代码：我先建立整个系统的骨架

先把它画出来。

```text
                    Agent / LangGraph
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
    Session Store      Compaction        Tracer
      存全量历史        生成模型视图        记录执行过程
          │                │                │
          ▼                ▼                ▼
      JSONL / 内存       历史摘要           span
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                    LLM / Agent 执行
```

这里有一个非常重要的思想：

## 这三个模块并不是 Agent 的“业务逻辑”

例如：

```text
Agent：
    分析舆情
    调用搜索工具
    判断结果
    生成回答
```

这是业务逻辑。

而：

```text
Session：
    把执行状态存下来

Compaction：
    控制发给模型的上下文

Tracer：
    记录执行过程
```

属于**基础设施能力**。

也就是：

> Agent 负责“做什么”，Harness 负责“让它能够稳定地做”。

这个思路非常重要。

因为如果这些东西全部写进 Agent：

```python
def agent():
    # 先读历史
    # 判断是不是太长
    # 如果太长就压缩
    # 调模型
    # 记录耗时
    # 记录异常
    # 把状态保存
    # ...
```

最后 Agent 会越来越胖。

以后想：

* 换存储方式
* 换压缩策略
* 换日志系统
* 删除 Trace
* 增加监控

都会直接碰业务代码。

这就是工程化最核心的问题之一：

> **把容易变化的基础设施能力，从核心业务逻辑里拆出去。**

---

# 二、第一个模块：Session Store

## 2.1 Session 到底是什么？

先不要管 Python。

假设你正在和 Agent 聊天。

```text
用户：
    帮我分析今天的舆情

Agent：
    好

用户：
    再按照昨天那个格式整理

Agent：
    好，我记得
```

这里 Agent 为什么知道“昨天那个格式”？

因为它需要历史状态。

如果程序运行到一半：

```text
进程退出
服务器重启
```

内存中的：

```python
messages = [...]
```

没了。

那么下一次启动：

```text
用户：
    按照昨天的格式整理

Agent：
    什么格式？
```

这就是 Session Store 要解决的问题。

可以直接把它理解成：

> **游戏存档点。**

代码注释本身就是这么定义的：打到一半关机，下次读档还在原来的位置。

---

# 三、SessionStore 的第一个工程问题：存在哪里？

代码：

```python
class SessionStore:

    def __init__(self, path=None):
        self.path = path
        self.bad_lines = 0
        self._mem = {}
```

这里最重要的不是 Python 语法。

而是这个设计：

```python
path=None
```

和：

```python
path="session.jsonl"
```

代表两种后端。

### 内存模式

```text
SessionStore(None)
```

数据：

```text
Python 内存
```

程序一退出：

```text
数据没了
```

适合：

```text
测试
临时任务
单次运行
```

### 文件模式

```text
SessionStore("session.jsonl")
```

数据：

```text
文件
```

程序重启：

```text
还能读回来
```

这就是第一个非常典型的工程化思维：

> **调用方只关心“我能不能保存”，不应该关心你底层到底是内存、文件还是数据库。**

以后：

```text
JSONL
   ↓
SQLite
   ↓
PostgreSQL
```

如果上层接口没变：

```python
session.append(...)
session.load(...)
session.messages(...)
```

调用方就不用跟着一起重写。

这就是：

> **抽象接口，隔离实现。**

---

# 四、JSONL 是什么？

这一段第一次看到的时候容易懵。

JSON：

```json
{
  "thread_id": "123",
  "messages": []
}
```

JSONL 就是：

> **一行一个 JSON。**

例如：

```text
{"thread_id":"123","seq":0,"messages":[...]}
{"thread_id":"123","seq":1,"messages":[...]}
{"thread_id":"123","seq":2,"messages":[...]}
{"thread_id":"456","seq":0,"messages":[...]}
```

它有一个特点：

> **特别适合追加。**

因为新增一轮消息时，不需要重新把整个文件改一遍，只需要：

```text
在文件末尾再写一行
```

这也就对应代码里的：

```python
with open(self.path, "a", encoding="utf-8") as f:
```

这里的：

```text
"a"
```

就是 append：

> 追加。

---

# 五、为什么这里选择 append-only？

`append-only` 的意思就是：

> **历史记录只增加，不修改。**

例如：

```text
第1轮
第2轮
第3轮
第4轮
```

不会写成：

```text
第1轮被修改
第2轮被修改
第3轮被修改
```

而是：

```text
第1轮
第2轮
第3轮
第4轮
```

一直往后追加。

为什么？

因为它对崩溃更友好。

假设正在写：

```text
第4轮
```

程序突然：

```text
断电
```

最坏情况：

```text
第4轮坏了
```

但是：

```text
第1轮
第2轮
第3轮
```

还在。

而如果你每次都：

```text
把整个历史文件重新写一遍
```

那么中途崩溃，就可能直接把整个文件破坏。

所以这个设计本质上是在做：

> **把“修改旧数据”的风险，变成“追加新数据”的风险。**

这是非常典型的工程化取舍。

---

# 六、SessionStore 怎么恢复数据？

看这个函数：

```python
def _read_all(self):
```

它做的事情其实非常简单：

```text
文件
 ↓
一行一行读取
 ↓
JSON 解析
 ↓
取 thread_id
 ↓
按照 thread_id 分组
 ↓
得到：

{
    "thread1": [记录1, 记录2],
    "thread2": [记录1, 记录2]
}
```

核心代码：

```python
out.setdefault(tid, []).append(rec)
```

这里的：

```python
setdefault
```

可以先理解成：

> 如果这个 key 没有，就先创建一个空列表。

例如第一次看到：

```python
tid = "A"
```

相当于：

```python
out["A"] = []
```

然后：

```python
append(rec)
```

所以最终：

```python
{
    "A": [rec1, rec2, rec3],
    "B": [rec1, rec2]
}
```

---

# 七、为什么读取坏行不直接报错？

这里是一个很重要的工程思维。

代码：

```python
try:
    rec = json.loads(line)
    tid = rec["thread_id"]
except (json.JSONDecodeError, KeyError, TypeError):
    self.bad_lines += 1
    continue
```

如果文件中间有一行：

```text
{"thread_id":"123"
```

它不是完整 JSON。

普通程序可能：

```text
报错
→ 整个 Session 不能读
→ Agent 整个挂掉
```

但这里的设计是：

```text
坏这一行
    ↓
记录 bad_lines
    ↓
跳过
    ↓
继续读取后面的数据
```

也就是：

> **局部损坏，不要扩大成整个系统不可用。**

这是非常典型的容错设计。

---

# 八、append 里还有一个容易被忽略的东西：fsync

代码：

```python
f.flush()
os.fsync(f.fileno())
```

第一次看到特别容易觉得：

> 不就是写文件吗？

其实这里是在解决：

> **“我写成功了”和“数据真的已经落到持久化介质”是不是一回事？**

简单理解：

```text
Python
  ↓
操作系统缓冲区
  ↓
文件系统
  ↓
存储设备
```

`flush()` 更接近：

> 把 Python/文件对象里的缓冲往操作系统推。

`fsync()` 则是：

> 请求操作系统把这个文件的数据同步到持久化存储。

所以这里的思想是：

> **宁愿每次写慢一点，也希望 Session 的持久化可靠性更高。**

代码注释也明确写了：这是以每次写更慢为代价来换取更强的持久化保障。

---

# 九、为什么要保存 `ts / thread_id / seq / messages / meta`？

最终写入的结构：

```python
rec = {
    "ts": time.time(),
    "thread_id": thread_id,
    "seq": self._count(thread_id),
    "messages": list(messages),
    "meta": dict(meta or {}),
}
```

可以把它理解成：

```text
ts
↓
什么时候存的

thread_id
↓
属于哪个会话

seq
↓
这个会话的第几次快照

messages
↓
真正的数据

meta
↓
额外信息
```

这里值得注意：

```python
messages = list(messages)
```

不是直接把原列表对象塞进去，而是复制一层。

同理：

```python
dict(meta or {})
```

也复制了一层。

这背后的工程意图可以理解为：

> **Session 保存的是“这一时刻的快照”，而不是把调用方未来还会修改的容器直接绑进来。**

---

# 十、Session Store 的公开 API 为什么这么设计？

代码最后有：

```python
append()
load()
latest()
messages()
list_threads()
delete()
```

可以画成：

```text
append
  ↓
保存一份快照

load
  ↓
拿到这个会话全部快照

latest
  ↓
只拿最后一份

messages
  ↓
只拿最后一份里的 messages

list_threads
  ↓
有哪些会话

delete
  ↓
删除一个会话
```

这里有一个很典型的代码组织方式：

> **底层能力少，上层 API 组合。**

比如：

```python
def latest(self, thread_id):
    rs = self.load(thread_id)
    return rs[-1] if rs else None
```

它没有重新实现“怎么读文件”。

而是直接使用：

```python
load()
```

所以：

```text
load()
   ↓
latest()
   ↓
messages()
```

是一层一层建立起来的。

这就是为什么代码会比“每个函数自己重新读取文件”更容易维护。

---

# 十一、delete 为什么这么麻烦？

前面说：

> append-only。

那为什么删除还要重新写文件？

因为你不能从一个文本文件中间：

```text
把 thread-A 的几行直接擦掉
```

所以做法变成：

```text
原文件
   ↓
读取全部数据
   ↓
删掉指定 thread
   ↓
写到临时文件
   ↓
os.replace()
   ↓
替换原文件
```

代码：

```python
tmp = self.path + ".tmp"

with open(tmp, "w", encoding="utf-8") as f:
    ...

os.replace(tmp, self.path)
```

重点不是 `.tmp`。

重点是：

```python
os.replace()
```

它的目标是让外部尽量只看到两种状态：

```text
旧文件
```

或者：

```text
新文件
```

而不是：

```text
写到一半的破文件
```

所以这里体现的是另一个工程化关键词：

> **原子性。**

---

# 十二、SessionStore 的整体流程

把这一整个类压缩成一张图：

```text
                    SessionStore
                         │
             ┌───────────┴───────────┐
             │                       │
          path=None               path=x.jsonl
             │                       │
           _mem                    文件
             │                       │
             └───────────┬───────────┘
                         │
                      append()
                         │
                         ▼
                      快照 rec
                         │
               ┌─────────┼─────────┐
               ▼         ▼         ▼
              ts      thread_id   messages
               │         │         │
               └─────────┼─────────┘
                         ▼
                       JSONL
                         │
                         ▼
                      load()
                         │
                         ▼
                     latest()
                         │
                         ▼
                     messages()
```

现在再看 SessionStore，就不容易迷路了。

---

# 十三、第二个模块：Context Compaction

Session 有了以后，新的问题出现了。

假设 Agent 执行了：

```text
第1轮
第2轮
第3轮
...
第100轮
```

Session 说：

> 没问题，我全部给你保存。

但是模型说：

> 等一下，我上下文不是无限的。

于是：

```text
Session Store
    ↓
保存了全部历史
```

并不等于：

```text
LLM 每次都要看到全部历史
```

这就是 Context Compaction。

---

# 十四、Compaction 到底在干什么？

代码注释用了一个很好理解的比喻：

```text
行李箱超重
```

假设：

```text
你有 40 件衣服
```

但是飞机：

```text
只允许带 10 件
```

最粗暴的方法：

```text
直接扔掉 30 件
```

问题：

> 你以后可能还需要那 30 件。

另一种方法：

```text
写一张清单：

5件外套
3件衬衫
2件裤子
...
```

然后：

```text
只带最近最重要的东西
+
一张早期历史摘要
```

这就是压缩。

代码文件也明确把它定义成：

> 压缩是“投影”，不是“销毁”。原始消息仍然保存在 SessionStore 中。

---

# 十五、这里有一个特别重要的工程决策

问题：

> 压缩后的消息，要不要直接覆盖 Session？

答案：

> **不要。**

采用：

```text
存全量
+
读时压缩
```

也就是：

```text
Session
  ↓
原始历史
  ↓
Compactor
  ↓
给 LLM 的历史视图
```

而不是：

```text
Session
  ↓
压缩
  ↓
保存压缩版
```

为什么？

因为：

```text
今天：
max_messages = 20

明天：
max_messages = 10

后天：
max_messages = 50
```

如果原始数据没了：

```text
20 → 10
```

以后就回不来了。

如果原始数据一直保存：

```text
原始历史
   ↓
不同 Compaction 策略
   ↓
不同模型视图
```

随时可以重新计算。

这其实就是一个非常重要的工程模式：

> **原始数据和派生数据分开。**

可以把它理解成：

```text
Source of Truth
      ↓
   原始历史

Projection
      ↓
   当前需要的视图
```

这也是这三个文件里非常重要的一条设计思想。

---

# 十六、`needs_compaction()` 为什么这么简单？

代码：

```python
def needs_compaction(self, messages):
    return len(messages) > self.max_messages
```

现在我们只按：

```text
消息条数
```

判断。

比如：

```python
max_messages = 20
```

那么：

```text
20 条 → 不压
21 条 → 压
```

但是这里也留下了一个非常重要的工程问题：

> **消息条数 ≠ Token 数。**

例如：

```text
消息1：
“好”
```

只有很少内容。

但另一条：

```text
消息2：
10000 字符的大段代码
```

依然只算：

```text
1 条消息
```

所以以后更真实的生产设计，触发条件可能应该是：

```text
消息条数
+
Token 数
+
工具结果大小
```

这里代码自己的注释其实也已经意识到这个问题。

这就是一个非常好的工程学习方式：

> **不是看到现有代码就结束，而是继续问：这个设计在哪种情况下会失效？**

---

# 十七、为什么 `maybe_compact()` 不直接返回原消息？

代码：

```python
if not self.needs_compaction(messages):
    return None
```

这一点特别值得学。

很多初学者会写：

```python
if 不需要压缩:
    return messages
```

但这里选择：

```python
return None
```

为什么？

因为调用方可以明确区分：

```text
None
→ 没有发生压缩

list
→ 真的产生了新的压缩视图
```

这其实是在设计一个更清晰的函数契约：

```text
返回 None：
    没变化

返回 list：
    有新结果
```

这比：

```text
无论有没有压缩都返回一个 list
```

更容易让上层判断发生了什么。

---

# 十八、system 消息为什么单独拿出来？

代码：

```python
head, body = [], list(messages)

if self.keep_system and body and _role(body[0]) == "system":
    head, body = [body[0]], body[1:]
```

这里是在做：

```text
system
   ↓
单独保存

其他消息
   ↓
参与压缩
```

因为 `system` 通常承担：

```text
角色设定
系统约束
行为规则
工具规则
输出格式
```

如果把它当普通历史消息一起删掉，整个 Agent 的行为边界可能变化。

所以工程上经常会看到：

```text
系统级数据
用户历史
工具历史
当前上下文
```

分层处理。

---

# 十九、整个压缩核心其实就是这几行

```python
cut = max(0, len(body) - self.keep_recent)

cut = _safe_cut(body, cut)

old, recent = body[:cut], body[cut:]

summary = self._summarize(old, summarize)

return (
    head
    + [{"role": "system", "content": f"[历史摘要] {summary}"}]
    + recent
)
```

翻译成人话：

```text
先算：

我准备留下最近多少条？

↓

剩下的是旧历史

↓

检查切分位置安全不安全

↓

旧历史做摘要

↓

最终：

system
+
历史摘要
+
最近 K 条
```

可以画成：

```text
原始消息

system
1
2
3
4
5
6
7
8
9
10
11
12

          ↓

          cut
           │
           ▼

system | old old old old | recent recent

          ↓

system | summary         | recent recent
```

---

# 二十、这里出现了一个真正的坑：tool 消息不能随便切

这是这份代码里最值得我记住的一点之一。

Agent 的消息不只是：

```text
user
assistant
```

还可能有：

```text
assistant 发起 tool call
        ↓
tool 返回结果
```

它们实际上是一个逻辑整体。

例如：

```text
assistant:
    我要调用 search

tool:
    search 返回结果
```

如果你压缩的时候变成：

```text
tool:
    search 返回结果
```

但是：

```text
assistant 发起调用
```

被删掉了。

那模型拿到的就是一个：

> **孤儿 tool 结果。**

所以：

```python
def _safe_cut(messages, cut):
    cut = max(0, min(cut, len(messages)))

    while (
        cut > 0
        and cut < len(messages)
        and _is_tool_result(messages[cut])
    ):
        cut -= 1

    return cut
```

它的思想非常简单：

> **如果切点正好落在 tool 结果上，就往前退。**

也就是：

```text
准备在这里切：

assistant(tool_calls)
tool(result)
↑
切点

不安全

↓

往前退
```

这就是工程化和“只是会写列表切片”的区别。

普通程序员可能想：

```python
messages[-6:]
```

工程代码会继续问：

> **这样切有没有违反上层协议？**

这就是非常重要的思维。

---

# 二十一、`_summarize()` 为什么接受一个 `summarize` 参数？

代码：

```python
def _summarize(self, old, summarize):
    if summarize is not None:
        return summarize(old)
```

这里其实是在做：

> **依赖注入。**

意思是：

```text
Compactor 不负责决定“到底用哪个 LLM”。

它只规定：

“你给我一个 summarize 函数，我调用它。”
```

所以以后可以传：

```python
summarize = my_llm_summarizer
```

也可以：

```python
summarize = test_summarizer
```

或者：

```python
summarize = mock_summarizer
```

这样：

```text
Compactor
```

不需要知道：

```text
OpenAI
Claude
本地模型
LangChain
LangGraph
```

都是什么。

这就是：

> **让模块依赖“接口/能力”，而不是依赖具体实现。**

---

# 二十二、为什么默认摘要不是 LLM？

代码里面还提供了一个无 LLM 的版本：

```python
roles = {}

for m in old:
    r = _role(m)
    roles[r] = roles.get(r, 0) + 1
```

最后得到类似：

```text
已折叠 16 条早期消息
(assistantx8, userx8)
```

这其实是一个很好的工程设计。

为什么？

因为：

```text
核心逻辑
```

和：

```text
真正调用 LLM 的能力
```

被分开了。

这样测试的时候：

```text
不用 API
不用网络
不用模型
```

一样可以验证：

```text
有没有触发压缩
有没有减少消息数量
有没有保留 system
有没有破坏 tool 边界
```

这是非常典型的：

> **把外部依赖隔离出去，先测试自己的逻辑。**

---

# 二十三、这里有一个我在最新版代码里发现的问题

当前代码：

```python
summary = self._summarize(old, summarize)

self.compactions += 1
self.last_summary = summarize
```

从前面的变量命名和最后返回结果来看，代码意图明显是：

```python
self.last_summary = summary
```

而当前写的是：

```python
self.last_summary = summarize
```

这意味着：

### 如果没有传 summarize：

```python
summarize = None
```

那么：

```python
last_summary = None
```

即使刚刚已经生成了摘要。

### 如果传了 summarize：

那么：

```python
last_summary
```

保存的是：

```text
函数对象
```

而不是：

```text
摘要结果
```

所以这不是语法错误，而是一个非常典型的：

> **“代码能运行，但是状态语义错了”的 Bug。**

这其实比语法错误更值得学习。

因为工程里最危险的问题之一就是：

```text
程序没炸
测试可能也没炸
但是状态已经错了
```

---

# 二十四、第三个模块：Tracer

Session 解决：

```text
历史在哪里？
```

Compaction 解决：

```text
历史太长怎么办？
```

但是现在还有一个问题：

> **Agent 出问题以后，我怎么知道到底发生了什么？**

例如：

```text
Agent
 ↓
LLM
 ↓
Tool A
 ↓
Tool B
 ↓
Tool C
```

最终：

```text
失败
```

你只知道：

```text
失败了
```

但不知道：

```text
哪个慢？
哪个炸？
谁调用了谁？
到底执行到了哪一步？
```

Tracer 就是干这个的。

代码把它比喻成：

> 外卖配送轨迹。你能看到“接单 → 到店 → 配送 → 送达”，Tracer 则给 Agent 的每一步记录开始、耗时、成功与否。

---

# 二十五、Tracer 和 Audit 有什么区别？

这两个特别容易混。

可以这样记：

```text
Audit：
    谁做了什么？
    为什么做？
    能不能追责？

Trace：
    运行到了哪里？
    花了多久？
    谁调用了谁？
    哪里失败？
```

也就是：

```text
Audit
    = 决策/行为记录

Trace
    = 执行链路记录
```

所以：

> Audit 更偏安全、审计。

> Trace 更偏性能、排障、可观测性。

---

# 二十六、Tracer 最核心的概念：Span

一个 span 可以先理解成：

> **一次函数/步骤执行的记录。**

例如：

```json
{
    "span_id": "s1",
    "parent_id": null,
    "name": "agent",
    "ms": 100,
    "ok": true
}
```

意思：

```text
agent 这一步
执行了 100ms
成功了
```

如果 agent 又调用了：

```text
tool
```

那么：

```json
{
    "span_id": "s2",
    "parent_id": "s1",
    "name": "tool"
}
```

意思：

```text
tool 是 agent 的孩子
```

于是就形成：

```text
agent
 ├── tool-A
 ├── tool-B
 │    └── helper
 └── tool-C
```

这就是 Trace 树。

---

# 二十七、这里第一次遇到装饰器

你之前没有专门学过装饰器，所以这里单独补。

最简单的装饰器其实就是：

```python
def log_call(fn):

    def wrapper(*args, **kwargs):
        print("开始")

        result = fn(*args, **kwargs)

        print("结束")

        return result

    return wrapper
```

然后：

```python
@log_call
def add(a, b):
    return a + b
```

它实际上相当于：

```python
def add(a, b):
    return a + b

add = log_call(add)
```

所以：

```text
原函数
   ↓
交给装饰器
   ↓
获得 wrapper
   ↓
以后调用 add
   ↓
实际上调用 wrapper
```

这就是装饰器。

---

# 二十八、为什么 Tracer 特别适合用装饰器？

因为我们不想每个节点都写：

```python
start = time.perf_counter()

try:
    ...
finally:
    ...
```

例如：

```python
def node_a():
    ...

def node_b():
    ...

def node_c():
    ...
```

如果每个函数都手写 Trace：

```text
node_a
  写一遍

node_b
  再写一遍

node_c
  再写一遍
```

太重复了。

所以变成：

```python
@tracer.trace()
def node_a():
    ...

@tracer.trace()
def node_b():
    ...

@tracer.trace()
def node_c():
    ...
```

业务函数不用关心 Trace。

这就是：

> **横切关注点。**

业务逻辑：

```text
我到底做什么
```

Trace：

```text
我怎么记录你怎么做
```

两者分开。

---

# 二十九、把当前 Tracer 的装饰器拆开

代码核心：

```python
def trace(self, name=None):

    def deco(fn):

        label = name or getattr(fn, "__name__", "anonymous")

        @functools.wraps(fn)
        def wrapper(*a, **kw):
            ...
        
        return wrapper

    return deco
```

这里有三层：

```text
trace()
 ↓
deco(fn)
 ↓
wrapper(*a, **kw)
```

第一次看到非常绕。

实际上可以这样理解：

## 第一层

```python
trace(name=None)
```

是：

> 我准备制作一个“装饰器”。

## 第二层

```python
deco(fn)
```

是真正接收到：

> 原来的那个函数。

## 第三层

```python
wrapper(*a, **kw)
```

是真正以后执行的函数。

所以：

```text
@tracer.trace()
def foo():
    pass
```

相当于：

```text
tracer.trace()
      ↓
    deco
      ↓
    foo
      ↓
   wrapper
```

---

# 三十、`*a, **kw` 是什么？

这一句：

```python
def wrapper(*a, **kw):
```

意思是：

> 不管原函数接收什么参数，我都先接住。

例如原函数：

```python
def foo(x):
```

可以。

原函数：

```python
def foo(x, y):
```

也可以。

原函数：

```python
def foo(x, y, name="a"):
```

也可以。

所以包装器不需要知道每个业务函数到底有几个参数。

这里：

```python
*a
```

接收：

```text
位置参数
```

这里：

```python
**kw
```

接收：

```text
关键字参数
```

然后：

```python
fn(*a, **kw)
```

再把它原样交给原函数。

---

# 三十一、`functools.wraps` 为什么要写？

代码：

```python
@functools.wraps(fn)
def wrapper(*a, **kw):
```

装饰器一个常见问题是：

```text
原来的函数名字
原来的 docstring
原来的元信息
```

可能被 wrapper 覆盖掉。

`wraps` 可以帮助 wrapper 保留原函数的元信息。

对现在学习阶段来说，记一句即可：

> **写装饰器时，通常把 `functools.wraps` 一起写上。**

它属于装饰器的工程规范，而不仅仅是为了“能运行”。

---

# 三十二、Tracer 为什么先取 parent？

代码：

```python
parent = _current.get()

parent_id = parent["span_id"] if parent else None
```

意思：

> 看看“现在是不是已经有一个 Trace 在执行”。

如果没有：

```text
parent = None
```

那么当前 span 是根节点。

如果有：

```text
parent = 某个外层 span
```

那么：

```python
parent_id = parent["span_id"]
```

当前这个 span 就知道：

> 我是谁的孩子。

---

# 三十三、这就是 `contextvars`

代码：

```python
_current = contextvars.ContextVar(
    "current_span",
    default=None
)
```

它保存的是：

> 当前执行上下文中的 span。

然后：

```python
_current.set(holder)
```

就是：

> 把当前 span 设置成我。

之后内部再调用一个被 Trace 包住的函数：

```python
parent = _current.get()
```

就能读到：

> 当前外层是谁。

于是：

```text
parent
   ↓
child
   ↓
grandchild
```

自然就串起来了。

---

# 三十四、为什么不用一个普通的全局变量？

如果简单写：

```python
current_span = None
```

单线程可能看不出问题。

但真实 Agent 系统可能是：

```text
请求 A
请求 B
请求 C
同时执行
```

如果大家都共用：

```python
current_span
```

就可能出现：

```text
A 的 Trace
    被 B 改掉

B 的 Trace
    又被 C 改掉
```

于是父子关系串线。

`contextvars` 就是专门为这种：

> **上下文相关、并发场景下需要隔离的数据**

设计的。

所以这里实际上不是“为了炫一个 Python API”。

而是：

> **因为系统未来会并发，所以现在不使用一个简单但不安全的全局变量。**

这就是设计和 API 的区别。

---

# 三十五、Tracer 最关键的一段：try / except / finally

代码：

```python
try:
    out = fn(*a, **kw)
    return out

except Exception as e:
    ok, err = False, f"{type(e).__name__}: {e}"
    raise

finally:
    _current.reset(token)

    self.record(
        label,
        ms=(time.perf_counter() - t0) * 1000,
        ok=ok,
        err=err,
        parent_id=parent_id,
        span_id=holder["span_id"],
    )
```

这一段值得仔细学。

---

# 三十六、为什么 `return out` 不能漏？

假设：

```python
def foo():
    return {"x": 1}
```

Tracer 包装以后：

```python
def wrapper():
    out = foo()
```

如果后面没有：

```python
return out
```

那么：

```python
wrapper()
```

得到的就是：

```python
None
```

原函数明明返回了：

```python
{"x": 1}
```

结果被装饰器吃掉了。

这就是为什么注释里专门写：

```python
return out
```

因为：

> **一个装饰器应该在增加能力的同时保持原函数的行为。**

这是工程上非常重要的原则：

> **非侵入式增强。**

---

# 三十七、为什么异常要 `raise`？

代码：

```python
except Exception as e:
    ok, err = False, ...
    raise
```

很多初学者会觉得：

> 都捕获了，为什么还要再抛？

因为 Trace 的职责是：

```text
记录异常
```

而不是：

```text
吞掉异常
```

比如业务代码：

```python
raise ValueError("参数错误")
```

Tracer 应该：

```text
记录：
    ok = False
    err = ValueError...

然后：
    异常继续抛出去
```

所以：

```text
Tracer：
    我负责记录

业务：
    我负责处理异常
```

职责不能混在一起。

---

# 三十八、为什么日志放到 `finally`？

因为：

```python
finally
```

无论：

```text
成功
失败
return
```

最终都会执行。

所以：

```python
return out
```

之后：

```python
finally
```

仍然会先完成 Trace 记录，再真正返回。

这样：

```text
成功
   ↓
record

失败
   ↓
record
   ↓
继续抛异常
```

两条路径都能留下记录。

这就是为什么：

```text
try
except
finally
```

这里其实已经不是简单语法题了。

它是在实现一个工程保证：

> **无论业务成功还是失败，都尽量记录一次执行结果。**

---

# 三十九、为什么 `_current.reset(token)` 也必须做？

之前：

```python
token = _current.set(holder)
```

相当于：

```text
当前上下文：
旧值 → holder
```

那么结束以后：

```python
_current.reset(token)
```

就是：

```text
恢复原来的上下文
```

否则下一次执行的时候：

```text
旧 span
```

可能还残留。

所以这是另一种工程思维：

> **设置了上下文，就必须负责恢复上下文。**

很像：

```text
打开资源
  ↓
使用
  ↓
关闭资源
```

---

# 四十、Tracer 怎么得到父子树？

每条 span 都保存：

```python
{
    "span_id": ...,
    "parent_id": ...,
    "name": ...,
    "ms": ...,
    "ok": ...,
    "err": ...,
    "ts": ...
}
```

所以：

```text
s1
├── s2
├── s3
│   └── s4
└── s5
```

其实不是提前存了一棵树。

而是：

> **每个节点只保存“我的父亲是谁”。**

最后：

```python
children.setdefault(s["parent_id"], []).append(s)
```

把：

```text
parent_id
```

重新组织成：

```text
树
```

这叫：

> **扁平记录，树形重建。**

这是一种很常见的数据设计。

因为写入的时候：

```text
每条 span 独立
```

比较简单。

真正展示的时候：

```text
再组装成树
```

比较灵活。

---

# 四十一、为什么 `total_ms()` 只统计顶层 span？

代码：

```python
return round(
    sum(
        s["ms"]
        for s in self.spans
        if s["parent_id"] is None
    ),
    3
)
```

为什么不把所有 span 都加起来？

假设：

```text
agent       100ms
 ├── tool    30ms
 └── llm     50ms
```

如果全加：

```text
100 + 30 + 50 = 180ms
```

但实际上：

```text
用户看到的总执行时间
```

更接近：

```text
100ms
```

因为：

```text
tool
llm
```

已经包含在 agent 的执行时间里面。

所以：

> **统计指标必须先定义语义，再写代码。**

这也是工程化非常重要的一点。

不是：

```text
能算出来
```

就代表：

```text
这个数字有意义
```

---

# 四十二、把三个模块连起来

现在把整个流程画出来。

```text
                 Agent / LangGraph
                        │
                        │
                        ▼
                SessionStore
                        │
                  保存全量历史
                        │
                        ▼
                 session.messages()
                        │
                        ▼
                 ContextCompactor
                        │
               ┌────────┴─────────┐
               │                  │
          不超过阈值            超过阈值
               │                  │
               ▼                  ▼
            原消息        摘要 + 最近消息
               │                  │
               └────────┬─────────┘
                        ▼
                       LLM
                        │
                        ▼
                   Agent 执行
                        │
                        ▼
                     Tracer
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
        name            ms            ok
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                    Trace Tree
```

所以现在可以把三者记成一句话：

> **Session 管“数据有没有”，Compaction 管“这次给模型看多少”，Tracer 管“这次到底发生了什么”。**

---

# 四十三、真正的工程化思维：不是“我会不会写这个类”

我觉得 Day5 最应该学的，其实不是：

```python
json
os
time
contextvars
functools
```

而是下面这个思考顺序。

---

## 第一步：先找失败场景

不要问：

> 我要写一个 Session 类。

先问：

> 如果 Agent 重启，会发生什么？

于是得到：

```text
需要持久化
```

不要问：

> 我要写一个 Compactor。

先问：

> 如果对话越来越长，会发生什么？

于是得到：

```text
需要压缩
```

不要问：

> 我要写一个 Tracer。

先问：

> Agent 出问题以后，我怎么排查？

于是得到：

```text
需要 Trace
```

---

# 四十四、第二步：给每个模块划边界

### Session

负责：

```text
保存
读取
删除
```

不负责：

```text
摘要
LLM
模型调用
```

### Compactor

负责：

```text
决定什么时候压
怎么切
怎么摘要
```

不负责：

```text
保存原始历史
```

### Tracer

负责：

```text
记录执行
```

不负责：

```text
改变业务结果
```

边界越清楚：

```text
模块之间越容易替换
测试越容易写
出错越容易定位
```

---

# 四十五、第三步：定义“不变量”

工程代码真正重要的，不只是“正常情况下怎么跑”。

而是：

> **什么事情无论如何都不能发生？**

这三个模块里，其实有很多这样的“不变量”。

### Session

```text
不能因为坏一行，导致整个历史不可读
```

### Compaction

```text
不能把 assistant.tool_calls 和 tool result 拆开
```

### Tracer

```text
不能因为记录日志，改变业务函数返回值
```

### Tracer

```text
不能因为 Trace 出错，让业务逻辑一起失败
```

### Compaction

```text
不能为了压缩，把原始 Session 数据永久删掉
```

一旦开始用：

> **“这个系统必须永远保证什么？”**

去看代码。

你就开始进入工程思维了。

---

# 四十六、第四步：再考虑异常

继续拿三个模块举例。

### Session

如果文件坏了一行：

```text
跳过
继续恢复
```

### Delete

如果重写过程中挂了：

```text
临时文件
+
原子替换
```

### Tracer

如果日志落盘失败：

```python
except OSError:
    pass
```

为什么？

因为：

> **日志系统不能反过来杀死主业务。**

### Tracer

如果业务抛异常：

```python
raise
```

为什么？

因为：

> **Trace 不能吞掉业务错误。**

这些都是非常典型的：

> **失败隔离。**

---

# 四十七、第五步：再考虑“以后会不会变”

这就是为什么这里大量使用：

```text
path
summarize
parent_id
contextvars
```

例如：

### Session

今天：

```text
JSONL
```

以后：

```text
SQLite
```

### Compactor

今天：

```text
规则摘要
```

以后：

```text
LLM 摘要
```

### Tracer

今天：

```text
JSONL
```

以后：

```text
真正的 Observability 系统
```

好的工程代码不是：

> “今天刚好能跑。”

而是：

> **在合理范围内，为未来变化预留替换点。**

---

# 四十八、这三个文件里我最值得记住的 8 个工程设计

## 1. 原始数据和派生数据分开

```text
Session
   ↓
原始历史

Compaction
   ↓
模型视图
```

---

## 2. append-only 优先

```text
新增记录
```

比：

```text
修改旧记录
```

更容易处理崩溃。

---

## 3. 局部失败不要扩大

```text
坏一行
```

不应该变成：

```text
整个 Session 不能恢复
```

---

## 4. 基础设施不能改变业务语义

Tracer：

```text
记录业务函数
```

不能：

```text
修改业务函数返回值
```

---

## 5. 业务异常不能被基础设施吞掉

```python
raise
```

必须保留。

---

## 6. 依赖注入

```python
summarize=...
```

意味着：

```text
逻辑和具体 LLM 解耦
```

---

## 7. 原子替换

```python
tmp
↓
os.replace
```

避免中间态破坏真实文件。

---

## 8. 先定义指标含义再计算

```python
total_ms()
```

没有简单地：

```text
所有 span 全加
```

而是只加：

```text
root span
```

因为“总耗时”有明确的业务语义。

---

# 四十九、三个文件的最新版代码

下面代码保持你上传的三个最新版文件内容不变，方便以后复习时直接对照。

## `session.py`

```python
"""SessionStore —— 会话状态持久化(Harness 模块③)

类比: 游戏存档点。
  打到一半关机, 下次读档还在原地 —— 这就是 session 的作用。
  没有它, 每次打开都是新游戏: 第一关重来。

★ 三个必须回答的设计问题:
  1) 存哪?   内存 / JSONL 文件 / SQLite ...
             -> 同一个接口, 换后端不改调用方
  2) 怎么存? append-only(只追加, 不改旧行)
             -> 崩溃安全: 写到一半挂了, 前面的行仍然完整可读
  3) 存什么? 每轮的消息快照 + 元数据(ts / seq / meta)

★ 核心决策(面试会问): 压缩后存, 还是存全量、读时再压?
  答案: 存全量, 读时压(投影)。
  理由: 压缩策略是会变的。今天 max=20, 明天想改成 10 —— 存全量还能改,
       存压缩版就永久丢了。
  类比: 数据库的 WAL(先如实记日志) + 视图(按需投影)。
"""
from __future__ import annotations

import json
import os
import time


class SessionStore:
    """会话状态持久化。thread_id -> 一串快照。"""

    def __init__(self, path=None):
        self.path = path
        self.bad_lines = 0
        self._mem = {}

    def _read_all(self):
        if self.path is None:
            return {k: list(v) for k, v in self._mem.items()}

        out = {}

        if not os.path.exists(self.path):
            return out

        with open(self.path, encoding="utf-8") as f:
            for line in f:
                line = line.strip()

                if not line:
                    continue

                try:
                    rec = json.loads(line)
                    tid = rec["thread_id"]

                except (json.JSONDecodeError, KeyError, TypeError):
                    self.bad_lines += 1
                    continue

                out.setdefault(tid, []).append(rec)

        return out

    def _count(self, thread_id):
        if self.path is None:
            return len(self._mem.get(thread_id, []))

        n = 0

        if not os.path.exists(self.path):
            return 0

        with open(self.path, encoding="utf-8") as f:
            for line in f:
                line = line.strip()

                if not line:
                    continue

                try:
                    if json.loads(line)["thread_id"] == thread_id:
                        n += 1
                except Exception:
                    pass

        return n

    def append(self, thread_id, messages, meta=None):
        rec = {
            "ts": time.time(),
            "thread_id": thread_id,
            "seq": self._count(thread_id),
            "messages": list(messages),
            "meta": dict(meta or {}),
        }

        if self.path is None:
            self._mem.setdefault(thread_id, []).append(rec)
            return rec

        d = os.path.dirname(os.path.abspath(self.path))

        if d:
            os.makedirs(d, exist_ok=True)

        tail_complete = True

        if os.path.exists(self.path) and os.path.getsize(self.path) > 0:
            with open(self.path, "rb") as fr:
                fr.seek(-1, os.SEEK_END)
                tail_complete = fr.read(1) == b"\n"

        with open(self.path, "a", encoding="utf-8") as f:
            if not tail_complete:
                f.write("\n")

            f.write(json.dumps(rec, ensure_ascii=False) + "\n")
            f.flush()
            os.fsync(f.fileno())

        return rec

    def load(self, thread_id):
        return self._read_all().get(thread_id, [])

    def latest(self, thread_id):
        rs = self.load(thread_id)
        return rs[-1] if rs else None

    def messages(self, thread_id):
        rec = self.latest(thread_id)
        return list(rec["messages"]) if rec else []

    def list_threads(self):
        return sorted(self._read_all().keys())

    def delete(self, thread_id):
        if self.path is None:
            self._mem.pop(thread_id, None)
            return

        data = self._read_all()
        data.pop(thread_id, None)

        tmp = self.path + ".tmp"

        with open(tmp, "w", encoding="utf-8") as f:
            for recs in data.values():
                for r in recs:
                    f.write(json.dumps(r, ensure_ascii=False) + "\n")

        os.replace(tmp, self.path)
```

---

## `compact.py`

```python
"""ContextCompactor —— 上下文压缩(Harness 模块④)

类比: 行李箱超重。
  要带 40 件衣服上飞机, 航空公司只让带 10 件。两种做法:
    (a) 直接扔掉 30 件                          -> 你会忘事(不可逆丢失)
    (b) 写张清单"带了 5 件外套 3 件衬衫…", 实物少带 -> 摘要
  压缩就是 (b)。

★ 关键陷阱: assistant.tool_calls 和它对应的 tool 结果是一对, 不能拆开!
  类比: 你不能只寄出"问路的话"却不寄"对方的回答" —— 很多 API 会直接 400。
  真实踩过的坑: 删旧消息时把 assistant 的 tool_calls 删了、留下 tool 结果,
  模型收到"孤儿 tool 结果" -> 报错。
  修法: 切分点如果正好落在 tool 消息上, 就往前退到发起它的 assistant 消息。

★ 压缩是"投影"不是"销毁": 原始消息仍在 SessionStore 里(存全量)。
  好处: 改策略(max 20 -> 10)不用回填历史; 压缩算错也不会不可逆。
"""

from __future__ import annotations

from regex import R

def _role(m):
    return m.get("role") or m.get("type") or "?"

def _is_tool_result(m):
    return _role(m) =="tool"

def _safe_cut(messages,cut):
    """★ 把切分点往前挪, 保证不把 assistant.tool_calls 和 tool 结果拆开"""
    cut = max(0,min(cut,len(messages)))

    while cut>0 and cut <len(messages) and _is_tool_result(messages[cut]):
        cut -=1

    return cut


class ContextCompactor:
    def __init__(self,max_messages=20,keep_recent=6,keep_system=True):
        self.max_messages = max_messages
        self.keep_recent = keep_recent
        self.keep_system = keep_system
        self.compactions = 0
        self.last_summary = None

    def needs_compaction(self,messages):
        return len(messages)> self.max_messages

    def maybe_compact(self,messages,summarize=None):
        if not self.needs_compaction(messages):
            return None 

        head,body =[],list(messages)

        if self.keep_system and body and _role(body[0]) =="system":
            head,body=[body[0]],body[1:]

        cut =max(0,len(body) - self.keep_recent)
        cut=_safe_cut(body,cut)

        if cut ==0:
            return None

        old, recent = body[:cut], body[cut:]
        summary=self._summarize(old,summarize)

        self.compactions+=1
        self.last_summary=summarize

        return head + [
            {
                "role": "system",
                "content": f"[历史摘要] {summary}"
            }
        ] + recent

    def _summarize(self,old,summarize):
        if summarize is not None:
            return summarize(old)

        roles ={}

        for m in old:
            r=_role(m)
            roles[r] =roles.get(r,0)+1

        detail = ", ".join(
            f"{k}x{v}"
            for k, v in sorted(roles.items())
        )

        return f"已折叠 {len(old)} 条早期消息 ({detail})"
```

---

## `trace.py`

```python
"""Tracer —— 调用链追踪(Harness 模块⑤)

类比: 外卖配送轨迹。
  点外卖能看到: 商家接单 -> 骑手到店 -> 配送中 -> 已送达。
  Tracer 给 Agent 的每一步盖一个"开始 / 结束 / 耗时 / 成功没有"。

★ 和审计日志(audit)的区别 —— 两者都要, 不能互相替代:
    audit : 谁做了什么【决策】(安全视角, 用于追责/合规)
    trace : 每一步花了多久、成功没有、谁调用了谁(性能/排障视角)
  一句话: audit 回答"该不该做", trace 回答"为什么慢 / 为什么炸"。

★ 关键陷阱(踩过): 装饰器把返回值吃掉 -> 图直接报错。
  因为 LangGraph 节点必须返回 dict, 装饰器里少写一句 return 就变成返回 None。
  修法: try 里 return out, finally 里记日志 —— 两者互不干扰。
"""
from __future__ import annotations

import contextvars
import functools
import json
import os
import time

_current = contextvars.ContextVar("current_span", default=None)


class Tracer:
    def __init__(self, path=None):
        self.path = path
        self.spans = []
        self._seq = 0

    def _next_seq(self):
        self._seq += 1
        return self._seq

    def new_span_id(self):
        return f"s{self._next_seq()}"

    def record(self, name, *, ms, ok, err=None, parent_id=None, span_id=None):
        span = {
            "span_id": span_id or self.new_span_id(),
            "parent_id": parent_id,
            "name": name,
            "ms": round(float(ms), 3),
            "ok": bool(ok),
            "err": err,
            "ts": time.time(),
        }

        self.spans.append(span)

        if self.path:
            d = os.path.dirname(os.path.abspath(self.path))

            if d:
                os.makedirs(d, exist_ok=True)

            try:
                with open(self.path, "a", encoding="utf-8") as f:
                    f.write(
                        json.dumps(span, ensure_ascii=False) + "\n"
                    )
            except OSError:
                pass

        return span

    def trace(self, name=None):
        def deco(fn):
            label = name or getattr(fn, "__name__", "anonymous")

            @functools.wraps(fn)
            def wrapper(*a, **kw):
                parent = _current.get()
                parent_id = parent["span_id"] if parent else None

                holder = {
                    "span_id": self.new_span_id(),
                    "name": label
                }

                token = _current.set(holder)
                t0 = time.perf_counter()

                ok, err = True, None

                try:
                    out = fn(*a, **kw)
                    return out

                except Exception as e:
                    ok, err = False, f"{type(e).__name__}: {e}"
                    raise

                finally:
                    _current.reset(token)

                    self.record(
                        label,
                        ms=(time.perf_counter() - t0) * 1000,
                        ok=ok,
                        err=err,
                        parent_id=parent_id,
                        span_id=holder["span_id"]
                    )

            return wrapper

        return deco

    def summary(self):
        children = {}

        for s in self.spans:
            children.setdefault(
                s["parent_id"],
                []
            ).append(s)

        lines = []

        def walk(pid, depth):
            for s in children.get(pid, []):
                mark = "OK " if s["ok"] else "ERR"
                extra = f"  <- {s['err']}" if s["err"] else ""

                lines.append(
                    f"{'  ' * depth}"
                    f"{mark} "
                    f"{s['name']}  "
                    f"{s['ms']}ms"
                    f"{extra}"
                )

                walk(s["span_id"], depth + 1)

        walk(None, 0)
        return "\n".join(lines)

    def total_ms(self):
        return round(
            sum(
                s["ms"]
                for s in self.spans
                if s["parent_id"] is None
            ),
            3
        )

    def clear(self):
        self.spans.clear()
        self._seq = 0
```

---

# 五十、最后重新理解一下：“工程化”到底是什么？

以前我可能会把工程化理解成：

```text
目录规范
命名规范
class
日志
测试
```

现在理解得更准确一点。

工程化并不是：

> **把代码写得更复杂。**

而是：

> **面对真实运行环境中的失败、变化、并发、维护问题，提前设计边界和保护机制。**

比如：

```text
程序重启
    ↓
Session

上下文过长
    ↓
Compaction

数据压坏
    ↓
跳过坏行

删除过程崩溃
    ↓
tmp + os.replace

日志系统失败
    ↓
不能影响主流程

业务函数异常
    ↓
Trace 记录后继续 raise

嵌套调用
    ↓
parent_id

未来换摘要模型
    ↓
summarize 注入
```

你会发现：

> **工程代码很多时候不是在写“正常情况”。**

而是在写：

```text
如果坏了怎么办？
如果未来变了怎么办？
如果并发了怎么办？
如果中途退出怎么办？
如果一个模块挂了怎么办？
```

这才是今天真正应该带走的东西。

---

# 五十一、Day5 我现在真正学到的，不是三个类

可以最后把今天压缩成这一张图：

```text
              “Agent 怎么长期稳定运行？”
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ▼                 ▼                 ▼
    数据问题           上下文问题          排障问题
       │                 │                 │
       ▼                 ▼                 ▼
   Session Store      Compaction          Tracer
       │                 │                 │
       ▼                 ▼                 ▼
    存全量              读时压              记执行
       │                 │                 │
       └─────────────────┼─────────────────┘
                         ▼
                    Agent Harness
```

真正的工程思维则是：

```text
先发现问题
    ↓
定义模块边界
    ↓
定义不变量
    ↓
选择数据结构
    ↓
考虑失败场景
    ↓
隔离外部依赖
    ↓
保留替换空间
    ↓
最后才是写代码
```

所以以后再看到一个陌生的工程文件，我不能只问：

> “这个函数是干嘛的？”

还应该继续问：

> **“它防的是哪个问题？”**

再继续：

> **“如果删掉这段，会坏在哪里？”**

再继续：

> **“为什么不使用更简单的写法？”**

最后：

> **“这个模块和其他模块之间，边界在哪里？”**

这应该就是我这一阶段真正要练的东西。

---

# 五十二、复习时我只需要记住这 12 句话

1. **Session 是存档，不是 Agent 业务逻辑。**
2. **原始历史尽量保存，压缩属于派生视图。**
3. **append-only 的价值是降低修改历史数据带来的崩溃风险。**
4. **坏一条数据，不应该让整个系统一起死。**
5. **Compaction 不是删除历史，而是减少这次送给模型的数据。**
6. **消息条数不等于 Token 数。**
7. **tool call 和 tool result 不能随便拆。**
8. **装饰器本质是“用一个 wrapper 包住原函数”。**
9. **Tracer 必须透传原函数返回值。**
10. **记录异常和处理异常是两回事，所以要 `raise`。**
11. **`contextvars` 是为了让嵌套、并发执行下的上下文不串线。**
12. **工程化真正关注的是：失败、边界、变化和可维护性。**
