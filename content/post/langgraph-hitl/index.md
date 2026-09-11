---
description: "D4"
title: "从让图停下来，到回到过去"
draft: false
date: "2026-09-10T09:54:52+08:00"
slug: "LangGraph_HITL"
categories:
 - LangGraph
tags:
 - HITL
image: ""
---

# W6-D4 · 从让图停下来，到回到过去

> D3 学的是：
>
> **怎么让图停下来问人。**
>
> D4 是同一天的第二份代码。
>
> 今天不再往 HITL 上堆新按钮，而是顺着 D3 往前走一步：
>
> ```text
> D3：
> 图停下来
> ↓
> 人回答
> ↓
> 从 checkpoint 继续
>
> D4：
> 图已经有 checkpoint
> ↓
> 我能不能自己挑一个过去的 checkpoint？
> ↓
> 从那里重新跑？
> ```
>
> 答案是：
>
> **可以。**
>
> 这就是 LangGraph 的 Time Travel。

---

# 一、先看今天到底学什么

今天的实验故意不用 LLM。

因为我真正想验证的不是：

> “模型会说什么。”

而是：

> **“到底哪些节点被重新执行了？”**

所以把图简化成：

```text
START
  ↓
  A
  ↓
  B
  ↓
  C
  ↓
 END
```

完整跑一次以后，LangGraph 会留下多个 checkpoint。

于是我现在可以：

```text id="xq9c1m"
历史：

START → A → B → C → END
          ↑
       这个 checkpoint
```

然后从这里重新跑：

```text id="x9m2ax"
checkpoint
   ↓
   B
   ↓
   C
```

**A 不应该再跑。**

这就是今天最重要的验证目标。

---

# 二、完整代码先看一遍

今天的实验代码很短。

```python id="u3pm8a"
"""W6-D4：时间旅行 —— Replay + Fork"""

import io
import contextlib

from typing import Annotated, TypedDict

from langgraph.graph import (
    StateGraph,
    START,
    END,
)

from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver


class T(TypedDict):
    messages: Annotated[list, add_messages]

    # 普通字段：
    # 专门用来测试“改档以后未来会不会发生变化”
    way: str


def step_a(state: T) -> dict:
    print("→ A 执行")

    return {
        "messages": [
            ("ai", "A 完成")
        ]
    }


def step_b(state: T) -> dict:
    way = (
        state.get("way")
        or "默认方式"
    )

    print(
        f"→ B 执行 ({way})"
    )

    return {
        "messages": [
            ("ai", f"B 完成 ({way})")
        ]
    }


def step_c(state: T) -> dict:
    print("→ C 执行")

    return {
        "messages": [
            ("ai", "C 完成")
        ]
    }


builder = StateGraph(T)

builder.add_node("a", step_a)
builder.add_node("b", step_b)
builder.add_node("c", step_c)

builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("b", "c")
builder.add_edge("c", END)

graph = builder.compile(
    checkpointer=MemorySaver()
)


cfg = {
    "configurable": {
        "thread_id": "tt-1"
    }
}


# ① 先完整跑一遍
log = io.StringIO()

with contextlib.redirect_stdout(log):
    r0 = graph.invoke(
        {
            "messages": [
                (
                    "user",
                    "完整跑一遍 A B C"
                )
            ]
        },
        cfg,
    )

print(
    "原始跑完：",
    log.getvalue()
        .replace("\n", " | ")
        .strip()
)

print(
    "原始消息数：",
    len(r0["messages"])
)


# ② 查看历史 checkpoint
snaps = list(
    graph.get_state_history(cfg)
)

for i, s in enumerate(snaps):

    cid = (
        s.config[
            "configurable"
        ][
            "checkpoint_id"
        ]
    )

    print(
        f"#{i} "
        f"cid={cid[:8]} "
        f"next={s.next} "
        f"消息数="
        f"{len(s.values['messages'])}"
    )


# ③ 找到：
#    A 已完成
#    B 还没执行
#
#    也就是：
#
#    next == ("b",)

target = next(
    s
    for s in snaps
    if s.next == ("b",)
)

tcid = (
    target.config[
        "configurable"
    ][
        "checkpoint_id"
    ]
)


# ④ Replay
#
# None = 不传新输入
# target.config = 从这个 checkpoint 开始

log2 = io.StringIO()

with contextlib.redirect_stdout(log2):

    r_replay = graph.invoke(
        None,
        target.config,
    )

txt_replay = log2.getvalue()

print(
    "回放日志：",
    txt_replay
        .replace("\n", " | ")
        .strip()
)


# ⑤ Fork
#
# 从同一个历史 checkpoint
# 修改 way

new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    },
)

log3 = io.StringIO()

with contextlib.redirect_stdout(log3):

    r_fork = graph.invoke(
        None,
        new_cfg,
    )

txt_fork = log3.getvalue()

print(
    "分叉日志：",
    txt_fork
        .replace("\n", " | ")
        .strip()
)


# ⑥ 断言
#
# Replay：
# B / C 必须出现
# A 必须不能出现

assert (
    "→ B 执行 (默认方式)"
    in txt_replay
)

assert (
    "→ C 执行"
    in txt_replay
)

assert (
    "→ A 执行"
    not in txt_replay
)


# Fork：
# B 应该读到修改后的 way

assert (
    "→ B 执行 (另一种方式)"
    in txt_fork
)

assert (
    "→ A 执行"
    not in txt_fork
)


# 不要检查 messages[-1]
# 因为最后一条消息是 C。
#
# 应该检查 B 那条消息。

fork_text = " | ".join(
    str(m.content)
    for m in r_fork["messages"]
)

assert (
    "B 完成 (另一种方式)"
    in fork_text
)


# 原 checkpoint 还应该存在
snaps_after = list(
    graph.get_state_history(cfg)
)

cids_after = [
    s.config[
        "configurable"
    ][
        "checkpoint_id"
    ]
    for s in snaps_after
]

assert tcid in cids_after

assert (
    len(snaps_after)
    > len(snaps)
)


print(
    "✅ 时间旅行实验通过"
)
```

先不要急着理解每一行。

只看：

```text id="7g48ru"
完整运行
 ↓
保存 checkpoint
 ↓
找到 B 前面的 checkpoint
 ↓
Replay
 ↓
Fork
 ↓
断言
```

这就是今天的骨架。

---

# 三、第一块：为什么实验故意不用 LLM？

```python id="6p5lax"
def step_a(state):
    print("→ A 执行")
```

```python id="5d1l0c"
def step_b(state):
    print("→ B 执行")
```

```python id="7z8j51"
def step_c(state):
    print("→ C 执行")
```

这里没有模型，没有 API，没有随机性。

因为我要证明：

> **“Replay 到底跳过了哪些节点？”**

如果 B/C 出现，A 没出现，我就能比较干净地证明：

```text id="y6r29n"
B、C 重跑
A 没重跑
```

官方对 Replay 的定义也是：从历史 checkpoint 重新执行后续节点，checkpoint 之前的节点不重新执行。

所以今天这个实验越简单越好。

---

# 四、第二块：为什么需要 `way`？

```python id="3f0lhr"
class T(TypedDict):
    messages: Annotated[list, add_messages]
    way: str
```

A/C 不关心 `way`。

只有 B：

```python id="12l1n7"
way = (
    state.get("way")
    or "默认方式"
)
```

然后打印：

```text id="p2k5jx"
→ B 执行 (默认方式)
```

这样我以后修改：

```python id="6l1jpn"
way = "另一种方式"
```

就可以观察：

> **未来的 B 有没有真的发生变化。**

所以：

```text id="1d6dwy"
way
```

不是业务需求。

它只是一个：

> **“改档到底有没有影响未来”的实验开关。**

---

# 五、第三块：第一次完整运行，制造历史

```python id="qlinhs"
graph.invoke(
    {
        "messages": [
            (
                "user",
                "完整跑一遍 A B C"
            )
        ]
    },
    cfg,
)
```

先让它完整跑：

```text id="0r0k3x"
A
↓
B
↓
C
↓
END
```

这一跑非常重要。

因为：

> **没有历史，就没有时间旅行。**

现在 checkpoint 历史里应该已经保存了多个状态。

---

# 六、第四块：`get_state_history()` = 打开存档列表

```python id="ehz4c1"
snaps = list(
    graph.get_state_history(cfg)
)
```

这就是今天第一个新 API。

它的意思：

> **把这个 thread 的历史 checkpoint 全拿出来。**

当前 LangGraph 文档说明，历史是按**新 → 旧**返回的。

所以：

```text id="0wzvzz"
index = 0
```

不是最早。

反而是：

> **最新。**

这就是一个特别容易记反的坑。

---

# 七、第五块：每个 checkpoint 只先看三个东西

```python id="4d7hsg"
s.config
s.values
s.next
```

这三个东西是今天最重要的。

可以直接记：

```text id="92qw7f"
.config
→ 钥匙

.values
→ 内容

.next
→ 待办
```

官方 `StateSnapshot` 也是这样定义的：`.values` 是当前状态值，`.next` 是下一步要执行的节点，`.config` 是读取该快照时使用的配置。

---

# 八、`next` 是今天挑档的关键

假设时间线：

```text id="7s2igx"
START
  ↓
 A
  ↓
 B
  ↓
 C
  ↓
END
```

某个 checkpoint：

```text id="v1db6u"
A 已经完成
B 还没开始
```

它的：

```python id="vvk4py"
s.next
```

就是：

```python id="cw3v6n"
("b",)
```

这句话其实非常好懂：

> **下一步该执行 B。**

所以我想：

> “如果我要从 B 开始重跑，那就找 `next == ("b",)` 的 checkpoint。”

于是：

```python id="dq6vxv"
target = next(
    s
    for s in snaps
    if s.next == ("b",)
)
```

这就是：

> **挑档。**

---

# 九、为什么 `next == ()` 的档不要选？

如果：

```python id="f7ek31"
s.next == ()
```

意思是：

> **已经没有下一步。**

也就是：

```text id="lc3yz2"
A
↓
B
↓
C
↓
END
```

已经全部完成。

这个 checkpoint 拿来 replay：

```python id="b3t2x7"
graph.invoke(
    None,
    s.config
)
```

不会有什么可执行的东西。

所以：

> **挑 checkpoint 时，先看 `next`。**

想重跑 B/C：

```text id="by8hqm"
找：

next == ("b",)
```

---

# 十、这里再回头看 D3：其实你早就在用“时间旅行”

D3 的：

```python id="5cjcd3"
Command(
    resume="approve"
)
```

当时我是：

> **让 LangGraph 自动找到应该恢复的 checkpoint。**

D4 做的只是：

```text id="e1l8tz"
把 checkpoint 历史列表打开
↓
自己选哪一档
```

所以可以这样理解：

```text id="w81qqc"
D3：
自动读档

D4：
手动选档
```

这就是为什么 D4 看起来像新知识，但其实和 D3 是一条线。

---

# 十一、第六块：Replay

真正关键的一行：

```python id="7n5k8o"
r_replay = graph.invoke(
    None,
    target.config,
)
```

这里：

```text id="3qg6ae"
None
```

非常重要。

它表示：

> **不传新的输入。**

而：

```text id="iz2h0k"
target.config
```

表示：

> **从 target 这个 checkpoint 开始。**

所以整句话就是：

> **“把存档读到这里，从这里继续玩。”**

官方文档的 Replay 示例也是 `graph.invoke(None, checkpoint.config)`；checkpoint 之前的节点不会重新执行，之后的节点重新执行。

---

# 十二、今天最重要的问题：怎么证明真的只重跑 B/C？

这里不能只看：

```text id="yq4cwp"
→ B 执行
→ C 执行
```

因为：

> A 也可能偷偷跑过。

所以我专门设计三个断言：

```python id="e4u9j1"
assert "→ B 执行 (默认方式)" in txt_replay

assert "→ C 执行" in txt_replay

assert "→ A 执行" not in txt_replay
```

前两个证明：

> B/C 确实执行了。

第三个最关键：

> **A 没执行。**

所以今天最值钱的结论其实是：

> **“没有发生什么”，也是证据。**

这和普通“跑通代码”的区别很大。

---

# 十三、消息数为什么还能作为第二层证据？

如果目标 checkpoint：

```text id="b4v3e1"
messages = 2
```

里面是：

```text id="3l4p8v"
Human
A
```

Replay 后：

```text id="v5e6f5"
Human
A
B
C
```

所以：

```text id="98ftv1"
2 → 4
```

这可以作为第二层证据。

也就是说：

```text id="cok1d5"
日志：
没有 A，只有 B/C

消息：
2 → 4
```

两边都支持：

> **这是真 Replay，不是从头跑。**

---

# 十四、第七块：Fork

Replay 是：

```text id="8by8vo"
过去是什么
↓
原样重新走
```

Fork 是：

```text id="dzc5bn"
过去是什么
↓
我改一点
↓
看看另一条未来
```

代码：

```python id="11kohr"
new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    },
)
```

注意：

> 这里的 `target.config` 是过去的存档钥匙。

然后 `update_state()` 在这个历史点上改 state，并返回新的 config。

官方文档明确说明，`update_state()` 不会覆盖原来的 checkpoint，而是创建新的分支；之后再用这个新 config `invoke(None, new_cfg)` 继续执行。

---

# 十五、Fork 为什么能改变未来？

原来：

```text id="c8j3n1"
A
 ↓
B（默认方式）
 ↓
C
```

改完以后：

```text id="9smp53"
A
 ↓
B（另一种方式）
 ↓
C
```

因为 B 每次都会读：

```python id="zylhjp"
state["way"]
```

所以：

> **改变过去的 state，未来节点就会看到新的值。**

这就是 Fork 最核心的意思。

不是：

> “把历史改掉。”

而是：

> **“从这个历史点长出另一条未来。”**

---

# 十六、这里为什么一定使用 `new_cfg`？

不要这样：

```python id="z7kmee"
graph.update_state(
    target.config,
    {"way": "另一种方式"}
)

graph.invoke(
    None,
    target.config,
)
```

而要：

```python id="ojgd9h"
new_cfg = graph.update_state(
    target.config,
    {"way": "另一种方式"},
)

graph.invoke(
    None,
    new_cfg,
)
```

因为：

```text id="cmmh95"
target.config
→ 原来的过去

new_cfg
→ 新分支的钥匙
```

可以理解成：

```text id="k6b4af"
旧 checkpoint
    │
    ├── 原分支
    │
    └── new_cfg → 新分支
```

---

# 十七、今天真正坑我的地方：断言也会写错

这一次非常值得单独记录。

原本写了：

```python id="7zt3z4"
assert (
    "另一种方式"
    in r_fork["messages"][-1].content
)
```

结果红了。

一开始我以为：

> “难道 Fork 没生效？”

但看日志：

```text id="w3u2k8"
→ B 执行 (另一种方式)
→ C 执行
```

已经明确证明：

> **Fork 生效了。**

真正的问题是：

```python id="m0zjwy"
r_fork["messages"][-1]
```

最后一条是：

```text id="c5n6h4"
C
```

而不是：

```text id="lq4k88"
B
```

所以：

> **断言检查错地方了。**

这件事让我重新意识到：

> **AssertionError 不等于业务代码有 bug。**

首先要排查三件事：

```text id="unb3oj"
① 机制错了？

② 断言写错了？

③ 验证目标本身就错了？
```

这次属于：

> **第三种。**

---

# 十八、而且我后来又连续猜错了

第一轮：

> `messages[-1]` 看错了。

这个判断是对的。

然后我又直接猜了一版字符串断言。

结果又红。

接着猜：

> “是不是中文全角括号？”

还是错。

最后把字符串真正打印出来，再看字符：

```python id="7xlc7a"
repr(text)
```

发现真正的问题是：

> **多了两个空格。**

这次才真正找到原因。

所以今天留下了一条非常值钱的纪律：

> **不要猜字符串为什么不相等，把“真身”打印出来。**

例如：

```python id="3mwlzp"
print(
    repr(
        r_fork["messages"][2].content
    )
)
```

必要时甚至可以：

```python id="8gkt9s"
print(
    [
        hex(ord(c))
        for c in text
    ]
)
```

机器算出来的东西，比肉眼看：

```text
B完成(另一种方式)
```

可靠得多。

---

# 十九、所以断言最好断结构，不要硬编码整句话

例如：

```python id="9m5xqd"
assert (
    r_fork.get("way")
    == "另一种方式"
)
```

这种结构断言比较稳。

如果还想证明：

> B 真的使用了这个值。

再加行为断言：

```python id="g1n10r"
assert (
    "→ B 执行"
    in txt_fork
)

assert (
    "另一种方式"
    in txt_fork
)
```

于是：

```text id="4yb7ua"
结构证据：
state 已经改了

行为证据：
B 真读到了修改后的值
```

两个一起才完整。

---

# 二十、最终的 8 项验证在证明什么？

这次实验不是“写几个 assert 装样子”。

每一个断言都对应一个机制。

### Replay

```text id="fy9m3f"
B 出现
```

→ B 确实重跑。

```text id="fchc8r"
C 出现
```

→ C 确实重跑。

```text id="xswj2c"
A 不出现
```

→ A 没重跑。

```text id="x46cb9"
消息数只增加 2
```

→ 确实只是补上 B/C。

### Fork

```text id="o2qz4k"
B 读到“另一种方式”
```

→ 修改真的影响未来。

```text id="qef9uz"
A 不出现
```

→ 没有从头跑。

```text id="v6t8f8"
原 checkpoint 还存在
```

→ 没有覆盖历史。

```text id="2iybry"
checkpoint 数量增加
```

→ 新分支真的产生了。

所以：

> **断言不是为了证明“程序没报错”，而是为了证明“机制就是我以为的机制”。**

---

# 二十一、Time Travel 的两个玩法

现在终于可以把今天的两个 API 摆在一起。

|      | Replay                     | Fork                                          |
| ---- | -------------------------- | --------------------------------------------- |
| 做什么  | 原样回放                       | 改状态后再走                                        |
| API  | `invoke(None, old_config)` | `update_state()` → `invoke(None, new_config)` |
| 前置节点 | 不重跑                        | 不重跑                                           |
| 后续节点 | 重跑                         | 重跑                                            |
| 原历史  | 保留                         | 保留                                            |
| 用途   | 调试 / 复现                    | what-if / 替代方案                                |

官方也是这么定义 Replay 与 Fork 的。

类比 RPG：

```text id="h2v4c8"
Replay：
读档重玩


Fork：
读档
+
开修改器
+
再玩一遍
```

---

# 二十二、这就是 D4 和 D3 的连接

现在回头看 D3。

D3：

```text id="y3q5x3"
interrupt()
 ↓
checkpoint
 ↓
Command(resume)
 ↓
继续
```

D4：

```text id="q6k5bq"
get_state_history()
 ↓
看到所有 checkpoint
 ↓
自己挑一个
 ↓
invoke(None, target.config)
```

所以：

> **D3 是自动选档。**
>
> **D4 是手动选档。**

这也是为什么我现在觉得 Time Travel 没有那么神秘。

它只是把：

```text id="p2f842"
“框架替我找哪份存档”
```

变成：

```text id="ntf9j0"
“我自己决定回到哪份存档”
```

---

# 二十三、最后一个边界：回档不是撤销现实世界

这个必须单独记。

如果一个工具已经：

```text id="e9y1ci"
真的删除文件
```

然后我回到删除之前的 checkpoint。

这不代表：

```text id="xq4w21"
文件自动恢复。
```

Time Travel 管的是：

> **Agent 的状态和决策历史。**

不是：

> **现实世界的副作用。**

所以：

```text id="5x8t4b"
回档
≠
撤销真实操作
```

这也是为什么生产系统里，Time Travel 和：

```text
idempotency
side effect
compensation
```

必须分开考虑。

---

# 二十四、今天真正要记的其实只有这一张图

```text id="x7s2n4"
             D3
              │
       interrupt + resume
              │
              ▼
        自动找到 checkpoint
              │
              ▼
             继续


             D4
              │
      get_state_history()
              │
              ▼
        打开历史存档列表
              │
              ▼
             挑档
          /        \
         /          \
    Replay          Fork
      │                │
      │         update_state()
      │                │
      │                ▼
      │             新分支
      │                │
      └──────┬─────────┘
             ▼
          invoke(None)
             │
             ▼
          重跑后半段
```

再压缩成一句：

> **D3 是自动读档，D4 是手动读档。**

---

# 二十五、今天的收尾复习卡

### 我只需要记住 5 个东西

```text id="0f8r3m"
thread_id
→ 哪条历史


checkpoint_id
→ 哪一帧


snapshot.next
→ 下一步该谁


Replay
→ 原样读档


Fork
→ 改档后长出另一条未来
```

再记一个验证原则：

> **“B/C 出现”证明它们跑了；“A 没出现”才证明它没有从头跑。**

最后再记今天最真实的一课：

> **断言失败以后，不要立刻改断言。**
>
> 先问：
>
> ```text
> 机制错了吗？
> 断言错了吗？
> 验证目标错了吗？
> ```
>
> 这次真正让我翻车的，不是 LangGraph，而是我自己写了一个**未经验证的断言**。

---

# 二十六、完整源码

下面这份就是今天的最终学习版。

我把你原来的实验保留了下来，只修正了那个会因为检查 `messages[-1]` 而误报的断言，并把失败信息尽量写得更容易排查。

```python id="v9a8s2"
"""
W6-D4 实验：Time Travel
=======================

目标：

1. 完整运行 A → B → C
2. 查看历史 checkpoint
3. 找到 next == ("b",) 的 checkpoint
4. Replay：
   - B 重跑
   - C 重跑
   - A 不重跑
5. Fork：
   - 修改 way
   - B 使用新的 way
   - A 不重跑
6. 验证：
   - 原 checkpoint 仍然存在
   - checkpoint 总数增加

注意：

这个实验故意不用 LLM。

因为今天真正要验证的是：
“到底哪些节点被重新执行”。

确定性越高，证据越干净。
"""

import io
import contextlib

from typing import Annotated, TypedDict

from langgraph.graph import (
    StateGraph,
    START,
    END,
)

from langgraph.graph.message import (
    add_messages,
)

from langgraph.checkpoint.memory import (
    MemorySaver,
)


# ============================================================
# 1. State
# ============================================================

class T(TypedDict):

    # 消息历史
    messages: Annotated[
        list,
        add_messages,
    ]

    # 普通字段。
    #
    # 专门用于：
    # “修改历史 state 后，
    #  未来节点是不是会看到新值？”
    way: str


# ============================================================
# 2. A 节点
# ============================================================

def step_a(state: T) -> dict:

    # 这句 print 是实验判据。
    #
    # Replay 时：
    #
    # 如果这里再次出现，
    # 就说明 A 被重新执行了。
    print(
        "→ A 执行"
    )

    return {
        "messages": [
            (
                "ai",
                "A 完成",
            )
        ]
    }


# ============================================================
# 3. B 节点
# ============================================================

def step_b(state: T) -> dict:

    # 默认情况下：
    #
    # way = ""
    #
    # 所以显示“默认方式”。
    #
    # Fork 后：
    #
    # update_state()
    # 把 way 改成“另一种方式”
    #
    # 那么 B 就会看到新值。
    way = (
        state.get("way")
        or "默认方式"
    )

    print(
        f"→ B 执行 ({way})"
    )

    return {
        "messages": [
            (
                "ai",
                f"B 完成 ({way})",
            )
        ]
    }


# ============================================================
# 4. C 节点
# ============================================================

def step_c(state: T) -> dict:

    print(
        "→ C 执行"
    )

    return {
        "messages": [
            (
                "ai",
                "C 完成",
            )
        ]
    }


# ============================================================
# 5. 构建图
# ============================================================

builder = StateGraph(T)

builder.add_node(
    "a",
    step_a,
)

builder.add_node(
    "b",
    step_b,
)

builder.add_node(
    "c",
    step_c,
)


# ------------------------------------------------------------
# 图结构：
#
# START → A → B → C → END
# ------------------------------------------------------------

builder.add_edge(
    START,
    "a",
)

builder.add_edge(
    "a",
    "b",
)

builder.add_edge(
    "b",
    "c",
)

builder.add_edge(
    "c",
    END,
)


# ============================================================
# 6. 编译
# ============================================================
#
# checkpointer 是 Time Travel 的基础。
#
# 没有 checkpoint：
# 就没有历史可以选。
#

graph = builder.compile(
    checkpointer=MemorySaver()
)


# ============================================================
# 7. thread_id
# ============================================================
#
# thread_id：
# “我是哪条历史？”
#

cfg = {
    "configurable": {
        "thread_id": "tt-1"
    }
}


# ============================================================
# 8. 第一次完整运行
# ============================================================
#
# 先把 A/B/C 全部跑一遍，
# 制造出后面可以回去看的历史。
#

print(
    "\n" + "=" * 60
)

print(
    "第一次运行："
    "完整执行 A → B → C"
)

print(
    "=" * 60
)


log = io.StringIO()

with contextlib.redirect_stdout(
    log
):

    r0 = graph.invoke(
        {
            "messages": [
                (
                    "user",
                    "完整跑一遍 A B C",
                )
            ],

            # 初始化 way
            "way": "",
        },
        cfg,
    )


print(
    "原始跑完：",
    log.getvalue()
        .replace("\n", " | ")
        .strip(),
)


# 预期：
#
# Human
# A
# B
# C
#
# = 4 条消息

print(
    "原始消息数：",
    len(
        r0["messages"]
    ),
)


# ============================================================
# 9. 查看全部 checkpoint
# ============================================================
#
# get_state_history()
# 返回这一条 thread 的历史。
#
# 当前 LangGraph 是：
# 新 → 旧
#

print(
    "\n" + "=" * 60
)

print(
    "历史 checkpoint："
    "新 → 旧"
)

print(
    "=" * 60
)


snaps = list(
    graph.get_state_history(
        cfg
    )
)


print(
    "checkpoint 数量：",
    len(snaps),
)


for i, s in enumerate(
    snaps
):

    # ------------------------------------------
    # config
    #
    # 里面有：
    #
    # thread_id
    # checkpoint_id
    # ------------------------------------------

    cid = (
        s.config[
            "configurable"
        ][
            "checkpoint_id"
        ]
    )

    # ------------------------------------------
    # next
    #
    # 下一步该执行谁
    # ------------------------------------------

    next_nodes = s.next

    # ------------------------------------------
    # values
    #
    # 当前 state 内容
    # ------------------------------------------

    message_count = len(
        s.values[
            "messages"
        ]
    )

    print(
        f"#{i} "
        f"cid={cid[:8]} "
        f"next={next_nodes} "
        f"消息数={message_count}"
    )


# ============================================================
# 10. 找目标 checkpoint
# ============================================================
#
# 我想：
#
#     A 不重跑
#     B/C 重跑
#
# 所以我要找：
#
#     next == ("b",)
#
# 这意味着：
#
#     A 已完成
#     B 还没执行
#

target = next(
    s
    for s in snaps
    if s.next == ("b",)
)


target_checkpoint_id = (
    target.config[
        "configurable"
    ][
        "checkpoint_id"
    ]
)


print(
    "\n目标 checkpoint：",
    target_checkpoint_id[:8],
)

print(
    "目标 next：",
    target.next,
)

print(
    "目标消息数：",
    len(
        target.values[
            "messages"
        ]
    ),
)


# ============================================================
# 11. Replay
# ============================================================
#
# 关键：
#
#     invoke(None, target.config)
#
# None：
#     不提供新输入
#
# target.config：
#     从这一个 checkpoint 开始
#

print(
    "\n" + "=" * 60
)

print(
    "Replay："
    "从 B 前面的 checkpoint 继续"
)

print(
    "=" * 60
)


log2 = io.StringIO()

with contextlib.redirect_stdout(
    log2
):

    r_replay = graph.invoke(
        None,

        target.config,
    )


txt_replay = (
    log2.getvalue()
)


print(
    "回放日志：",
    txt_replay
        .replace("\n", " | ")
        .strip(),
)


print(
    "回放后消息数：",
    len(
        r_replay[
            "messages"
        ]
    ),
)


# ============================================================
# 12. Replay 断言
# ============================================================
#
# 这部分是今天最重要的。
#
# 不是：
#     “看起来像 replay”
#
# 而是：
#     “用断言证明 replay”
#

print(
    "\n" + "=" * 60
)

print(
    "Replay 断言"
)

print(
    "=" * 60
)


# B 必须重跑
assert (
    "→ B 执行 (默认方式)"
    in txt_replay
), (
    "B 没有重跑"
)


# C 必须重跑
assert (
    "→ C 执行"
    in txt_replay
), (
    "C 没有重跑"
)


# ------------------------------------------------------------
# ★ 最关键
#
# A 不能重跑。
#
# 如果出现：
#
#     → A 执行
#
# 就说明这不是我们想验证的 replay。
# ------------------------------------------------------------

assert (
    "→ A 执行"
    not in txt_replay
), (
    "A 竟然重跑了："
    "这说明没有从目标 checkpoint replay"
)


# ------------------------------------------------------------
# 消息数：
#
# 原来：
#
#     2
#
# Replay B/C：
#
#     +2
#
# 最终：
#
#     4
# ------------------------------------------------------------

expected_count = (
    len(
        target.values[
            "messages"
        ]
    )
    + 2
)

assert (
    len(
        r_replay[
            "messages"
        ]
    )
    == expected_count
), (
    "Replay 消息数量不符合预期："
    f"expected={expected_count}, "
    f"actual={len(r_replay['messages'])}"
)


print(
    "✅ Replay 断言通过"
)


# ============================================================
# 13. Fork
# ============================================================
#
# 从同一个历史 checkpoint：
#
#     修改 way
#
# 原来：
#     默认方式
#
# 新分支：
#     另一种方式
#

print(
    "\n" + "=" * 60
)

print(
    "Fork："
    "修改过去，再发展新的未来"
)

print(
    "=" * 60
)


new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    },
)


new_checkpoint_id = (
    new_cfg[
        "configurable"
    ][
        "checkpoint_id"
    ]
)


print(
    "原 checkpoint：",
    target_checkpoint_id[:8],
)

print(
    "新 checkpoint：",
    new_checkpoint_id[:8],
)


# ============================================================
# 14. 从 fork 后的新 config 继续
# ============================================================

log3 = io.StringIO()

with contextlib.redirect_stdout(
    log3
):

    r_fork = graph.invoke(
        None,
        new_cfg,
    )


txt_fork = (
    log3.getvalue()
)


print(
    "分叉日志：",
    txt_fork
        .replace("\n", " | ")
        .strip(),
)


# ============================================================
# 15. Fork 断言
# ============================================================

print(
    "\n" + "=" * 60
)

print(
    "Fork 断言"
)

print(
    "=" * 60
)


# ------------------------------------------------------------
# B 必须看到修改后的 way
# ------------------------------------------------------------

assert (
    "→ B 执行 (另一种方式)"
    in txt_fork
), (
    "B 没有读取到 fork 后的新值"
)


# ------------------------------------------------------------
# A 不应该重新执行
# ------------------------------------------------------------

assert (
    "→ A 执行"
    not in txt_fork
), (
    "A 不应该因为 fork 又重跑"
)


# ------------------------------------------------------------
# 不要写：
#
#     r_fork["messages"][-1]
#
# 因为最后一条消息是 C。
#
# 我要验证的是：
#     B 那一条消息
# ------------------------------------------------------------

fork_text = " | ".join(
    str(
        message.content
    )
    for message
    in r_fork["messages"]
)


assert (
    "B 完成 (另一种方式)"
    in fork_text
), (
    "Fork 结果中没有找到 "
    "B 使用新 way 的证据："
    f"{fork_text!r}"
)


print(
    "✅ Fork 断言通过"
)


# ============================================================
# 16. 验证原历史仍然存在
# ============================================================
#
# Fork：
#
#     不是覆盖原历史
#
# 而是：
#
#     从过去长出新分支
#

snaps_after = list(
    graph.get_state_history(
        cfg
    )
)


checkpoint_ids_after = [
    s.config[
        "configurable"
    ][
        "checkpoint_id"
    ]

    for s in snaps_after
]


# 原 checkpoint 必须还存在
assert (
    target_checkpoint_id
    in checkpoint_ids_after
), (
    "原 checkpoint 消失了"
)


# Fork 后应该出现更多 checkpoint
assert (
    len(snaps_after)
    >
    len(snaps)
), (
    "Fork 没有产生新的 checkpoint"
)


# ============================================================
# 17. 最终结果
# ============================================================

print(
    "\n" + "=" * 60
)

print(
    "✅ Time Travel 实验全部通过"
)

print(
    "Replay："
    "B/C 重跑，A 没重跑"
)

print(
    "Fork："
    "B 使用新 way，原历史仍然存在"
)

print(
    "=" * 60
)
```

---

# 二十六、最后收成一张卡

```text id="c6t6e0"
W6-D4
────────────────────────

D3：
自动恢复 checkpoint

D4：
手动挑 checkpoint


get_state_history()
        ↓
    存档列表

snap.next
        ↓
    下一步是谁

snap.config
        ↓
    这份存档的钥匙

invoke(None, snap.config)
        ↓
      Replay

update_state(...)
        ↓
       Fork

Replay：
原样重跑未来

Fork：
改档后重跑未来


最重要的验证：

B 出现 ✅
C 出现 ✅
A 不出现 ✅
```

还有今天额外学到的一条：

> **断言失败，不要马上改断言。**
>
> 先问：
>
> **机制错了？断言错了？还是验证目标错了？**

这一天真正收尾的地方，不是：

```text
✅ 8 个断言全绿
```

而是：

> **我开始知道，连“验证代码”本身也需要被验证。**

D3 学的是：

> **让图停下来问人。**

D4 学的是：

> **既然图会存档，那我就可以自己挑过去，重新走未来。**

到这里，W6 前四天其实已经连成一条线：

```text id="v2e7hz"
存档
 ↓
停下来
 ↓
人干预
 ↓
改状态
 ↓
回到过去
 ↓
重新走未来
```

这就是这一天的新知识，也是 HITL 这一章的真正收尾。
