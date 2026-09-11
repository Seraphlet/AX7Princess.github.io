---
description: ""
title: " LangGraph_HITL"
draft: false
date: "2026-09-10T09:54:52+08:00"
slug: "LangGraph_HITL"
categories:
 - LangGraph
tags:
 - HITL
image: ""
---

# 让图停下来问人

> 这篇不是教程。
>
> 是我给未来的自己留的一份笔记。
>
> 以后如果我又忘了：
>
> * `interrupt()` 到底是怎么停的？
> * 为什么 `LLM + interrupt` 最好拆成两个节点？
> * `edit` 到底改了什么？
> * 为什么 `AIMessage(id=last.id, ...)` 里的 `id` 不能删？
> * `audit` 到底是在解决什么问题？
> * 为什么最后一个模型不能绑工具？
> * `checkpoint_id` 到底是什么？
> * `get_state_history()` 为什么是时间旅行？
> * 怎么证明“真的只重跑了 B/C，没有从头跑 A”？
>
> 我希望重新看这篇文章的时候，不需要再从 200 行代码里硬找答案。

---

# 一、最开始，我只想让图停下来问人

最初的需求其实很简单：

```text
用户：
帮我删除 /tmp/a.txt

模型：
我要调用 delete_file("/tmp/a.txt")

程序：
等等。

这个操作危险。

先问人。
```

于是最基本的 HITL 是：

```text
用户
 ↓
模型
 ↓
interrupt()
 ↓
人
 ↓
approve / reject
```

一开始，我以为 HITL 就这么简单。

后来才发现：

> **真正麻烦的不是“停下来问人”，而是“停下来以后，人到底能改变什么”。**

---

# 二、第一层：approve / reject 其实不够

如果只有：

```text
approve
reject
```

那么人只有两个权力：

```text
允许模型做原来的事情

或者

不允许模型做
```

但现实里经常是第三种：

```text
“可以做，但参数改一下。”
```

例如：

```text
模型：

我要删除 /tmp/a.txt


人：

可以删。

但别删 a.txt。

改成 /tmp/old.txt。
```

所以审批其实应该是：

```text
approve
reject
edit
```

三种状态：

| 选择      | 意思         |
| ------- | ---------- |
| approve | 原参数直接执行    |
| reject  | 不执行        |
| edit    | 修改待执行参数后执行 |

于是我第一次真正意识到：

> **HITL 不只是“人有没有批准”，还可能意味着“人接管了模型的下一步动作”。**

---

# 三、那人到底修改什么？

这里是理解 D4 的第一个关键。

模型调用工具的时候，先产生的不是：

> “文件已经删除。”

而是一个：

> **待执行的工具调用。**

例如模型生成：

```python
AIMessage(
    tool_calls=[
        {
            "name": "delete_file",
            "args": {
                "path": "/tmp/a.txt"
            }
        }
    ]
)
```

此时工具还没真的执行。

可以把它想成一张工单：

```text
┌──────────────────────┐
│ 工具：delete_file    │
│ 参数：/tmp/a.txt     │
└──────────────────────┘
```

所以时间线上其实是：

```text
模型决定
   ↓
【现在可以人工修改】
   ↓
工具执行
```

人真正修改的是：

```python
AIMessage.tool_calls
```

而不是已经执行完的工具。

这个区别非常重要。

---

# 四、我第一次踩的大坑：`interrupt()` 不是普通暂停

我一开始很容易形成一个错误模型：

```python
def agent(state):
    ai_msg = llm.invoke(...)
    
    answer = interrupt(...)
    
    return {
        "messages": [ai_msg]
    }
```

脑子里的想象是：

```text
llm.invoke()
 ↓
ai_msg 已经产生
 ↓
interrupt()
 ↓
等人
 ↓
继续 return
```

但真实情况不是这样。

LangGraph 的 `interrupt()` 第一次被调用时，会触发一个可恢复的中断；当前节点停止执行。等以后通过 `Command(resume=...)` 恢复时，**这个节点会从头重新执行**，直到再次走到对应的 `interrupt()`，这次才拿到 resume 的值。

所以第一次运行：

```text
agent
 ↓
llm.invoke()
 ↓
ai_msg 只存在于 Python 局部变量
 ↓
interrupt()
 ↓
节点停掉
```

而：

```python
return {"messages": [ai_msg]}
```

根本还没执行。

所以：

> `ai_msg` 虽然“算出来了”，但还没有进入 state。

这就是我第一次看 HITL 时最容易错的地方。

---

# 五、所以为什么一定要拆成 `decide → approve`

正确结构是：

```text
decide
 ↓
先把 AIMessage 写进 state
 ↓
approve
 ↓
interrupt()
```

也就是：

```python
def decide(state):
    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    return {
        "messages": [ai_msg]
    }
```

然后：

```python
def approve(state):
    last = state["messages"][-1]

    tc = last.tool_calls[0]

    answer = interrupt({
        "question":
            f"模型想删除 "
            f"{tc['args']['path']}，批准吗？",
        "options": [
            "approve",
            "reject",
            "edit",
        ],
    })
```

此时流程是：

```text
decide
 ↓
return
 ↓
state 已经保存 AIMessage
 ↓
approve
 ↓
interrupt
```

所以当图停住的时候，我可以：

```python
st = graph.get_state(cfg)
```

然后放心地看：

```python
st.values["messages"][-1]
```

拿到真正的：

```text
AIMessage
  └── tool_calls
```

这张“工单”已经进入 checkpoint 了。

---

# 六、又发现一个反直觉：resume 会让节点重跑

第一次：

```text
approve()
 ↓
interrupt()
 ↓
停
```

然后：

```python
graph.invoke(
    Command(resume="approve"),
    cfg
)
```

恢复后并不是：

```text
从 interrupt 下一行继续
```

而是：

```text
approve()
 ↓
从节点开头重新执行
 ↓
执行到 interrupt()
 ↓
这次 interrupt 返回 "approve"
 ↓
继续往下
```

所以：

```text
[approve]
[approve]
```

出现两次不是 bug。

是机制。

这件事给我留下一个非常重要的工程规则：

> **interrupt 前的代码必须允许重跑。**

如果那里只是：

```python
print(...)
读取 state
做纯计算
```

问题不大。

但如果那里有：

```text
扣款
发邮件
写数据库
调用外部 API
创建订单
```

就危险了。

因为恢复时，这段代码可能重新执行。

所以我的记忆方式是：

> **interrupt 前可以“想”和“读”，不要随便“动现实世界”。**

官方文档也明确建议：interrupt 节点在恢复时会重执行，因此要避免 interrupt 前存在不可安全重放的副作用。

---

# 七、第二个关键：怎么修改参数？

拿到：

```python
last = st.values["messages"][-1]
```

假设原来：

```python
last.tool_calls
```

是：

```python
[
    {
        "name": "delete_file",
        "args": {
            "path": "/tmp/a.txt"
        }
    }
]
```

我想把：

```text
a.txt
```

改成：

```text
old.txt
```

于是复制：

```python
new_tc = [
    {
        **tc,
        "args": {
            **tc["args"],
            "path": "/tmp/old.txt",
        },
    }
    for tc in last.tool_calls
]
```

这里的逻辑其实非常简单：

```text
旧工单：

delete_file
path=/tmp/a.txt


↓

改参数


新工单：

delete_file
path=/tmp/old.txt
```

然后创建新的 `AIMessage`。

---

# 八、这里为什么死死保留 `id`？

代码是：

```python
fixed = AIMessage(
    id=last.id,
    content=last.content,
    tool_calls=new_tc,
)
```

这里：

```python
id=last.id
```

千万不要看着碍眼就删。

因为：

```python
messages: Annotated[list, add_messages]
```

不是普通的：

```python
list.append()
```

消息更新是带有 message id 语义的。

可以简单理解成：

```text
相同 id
→ 更新原消息

新 id
→ 新增一条消息
```

所以：

```python
AIMessage(id=last.id, ...)
```

表达的是：

> “我要修改刚才那张工单。”

而：

```python
AIMessage(...)
```

更像：

> “我又新建了一张工单。”

于是原来：

```text
4 条消息
```

可能变成：

```text
5 条消息
```

结果 state 里同时存在：

```text
delete /tmp/a.txt

delete /tmp/old.txt
```

这不是修改。

这是：

> **桌上多了一张工单。**

所以我以后看到：

```python
AIMessage(
    id=last.id,
    ...
)
```

应该马上想到：

> **按原 ID 替换。**

---

# 九、`update_state()` 到底是什么？

现在这条：

```python
graph.update_state(
    cfg,
    {
        "messages": [fixed]
    }
)
```

可以简单理解成：

> **直接修改当前 checkpoint 的 state，然后生成一个新的 checkpoint。**

所以它不是：

> “调用某个节点。”

而是：

> **“我现在要改存档里的内容。”**

这也就是后面时间旅行的基础。

因为 HITL 里的“改参数”，本质上已经在偷偷做一个很小的：

> **改档。**

---

# 十、为什么 `edit` 没有自己的分支？

我第一次看这里也很容易困惑：

```python
if answer == "reject":
    return {...}

return {}
```

那：

```text
edit
```

去哪儿了？

其实没有：

```python
if answer == "edit":
```

这个分支。

因为：

> **edit 的真正动作发生在图外的 `update_state()`。**

也就是：

```text
用户选择 edit
        ↓
图外修改 tool_calls
        ↓
update_state()
        ↓
Command(resume="edit")
        ↓
approve()
        ↓
return {}
        ↓
tools
```

所以 `approve` 根本不负责：

> “执行 edit。”

它只负责：

> “是不是阻止这个工具继续执行？”

因此：

```text
reject
→ 拦住

approve
→ 放行

edit
→ 图外已经改好
→ 放行
```

这句话我以后应该记住：

> **`approve()` 是审批闸门，不是编辑器。**

---

# 十一、但还有一个问题：模型不知道“谁改的”

假设：

```text
用户：

删除 a.txt


模型：

delete a.txt


人：

改成 old.txt


工具：

删除 old.txt
```

如果最后模型只看 transcript：

```text
用户：
删除 a.txt

AI：
tool_calls = old.txt

Tool：
删除 old.txt
```

模型就会产生一个疑问：

> “为什么我刚才突然改成 old.txt 了？”

它不知道：

```text
是审批人改的。
```

所以很容易变成：

```text
“抱歉，我可能操作错误了……”
```

甚至：

```text
“我再试一次……”
```

问题不在模型。

而在于：

> **人的操作发生在图外，但模型只能看到 state。**

所以：

```text
人做过什么
```

必须想办法留下记录。

---

# 十二、`audit_note`：给人留下一道脚印

于是 state 多了一个字段：

```python
class S(TypedDict):
    messages: Annotated[list, add_messages]
    audit_note: str
```

然后：

```python
graph.update_state(
    cfg,
    {
        "messages": [fixed],
        "audit_note":
            "审批人把删除目标从 "
            "/tmp/a.txt 改成了 "
            "/tmp/old.txt",
    },
)
```

这里一次写入两个东西：

```text
① 新工单
② 人改工单的原因
```

然后：

```python
def audit(state):
    note = state.get("audit_note", "")

    if not note:
        return {}

    return {
        "messages": [
            HumanMessage(
                content=
                    f"【审批人操作记录】{note}。"
                    "这是审批人主动、有意做出的决定，"
                    "已获批准执行；"
                    "这不是错误，无需道歉、"
                    "无需重试其他路径，"
                    "请直接据实向用户总结结果。"
            )
        ],
        "audit_note": "",
    }
```

现在模型就能知道：

```text
用户要求 a.txt

↓

模型原本准备 a.txt

↓

审批人主动把它改成 old.txt

↓

工具执行 old.txt
```

因果链重新闭合了。

所以：

> **audit 解决的不是“工具问题”，而是“叙事问题”。**

---

# 十三、为什么叫“治叙事”？

因为模型非常依赖上下文。

如果上下文中间缺了一块：

```text
用户：
A

模型：
B

工具：
C
```

模型一定会问：

> “B 和 C 为什么不一致？”

如果它没有看到真正原因，就会自己补故事。

于是：

```text
audit
```

本质上是在告诉模型：

> **“中间不是你犯错，是人主动改了。”**

所以我以后看到：

```python
audit_note
```

可以直接理解成：

> **给模型补一段缺失的因果链。**

---

# 十四、然后又来了第二个问题：模型可能继续重试

补上 audit 后，模型知道：

> “哦，原来参数是人改的。”

但这又可能让它发现：

> “原来真的可以改工具参数。”

于是执行完后，它可能继续产生：

```text
tool_calls
```

造成：

```text
tools
 ↓
audit
 ↓
LLM
 ↓
tools
 ↓
audit
 ↓
LLM
 ↓
...
```

这时候不能只靠：

```text
System Prompt：

请不要重试。
```

因为：

> **Prompt 是说明书，不是锁。**

模型有能力调用工具。

你只是告诉它：

> “最好别调用。”

这属于软约束。

---

# 十五、真正的硬约束：收口模型根本没有工具

所以 D4 最后专门用了两个 LLM：

```python
llm_decide = ChatOpenAI(...).bind_tools(
    [delete_file]
)

llm_summarize = ChatOpenAI(...)
```

区别只有一句：

```text
decide：
有工具

summarize：
没工具
```

于是：

```text
decide
↓
可以产生 tool_calls
```

但：

```text
summarize
↓
根本没有工具
↓
无法产生工具调用
```

最终变成：

```text
tools
 ↓
audit
 ↓
summarize
 ↓
END
```

这就是“结构收口”。

所以：

```text
audit
```

解决：

> 模型不知道为什么发生变化。

而：

```text
summarize 不绑定工具
```

解决：

> 模型不应该再启动工具流程。

一个是：

> **治叙事。**

一个是：

> **治重试。**

---

# 十六、现在 D4 的完整骨架其实只有五个人

这一张表是我最想以后回来看的。

| 节点          | 像谁  | 做什么        |
| ----------- | --- | ---------- |
| `decide`    | 决策官 | “我要调用这个工具” |
| `approve`   | 审批官 | “准不准做？”    |
| `tools`     | 执行工 | “真的执行”     |
| `audit`     | 纪要员 | “刚才是人改的”   |
| `summarize` | 签字官 | “事情到这里结束”  |

于是整个图：

```text
START
  ↓
decide
  ↓
approve
  ↓
tools
  ↓
audit
  ↓
summarize
  ↓
END
```

只有 `decide` 有工具。

这个骨架才是我真正需要记住的。

---

# 十七、然后我发现：checkpoint 本身就是“存档”

前面的 HITL 已经在使用 checkpoint 了。

因为 `interrupt()` 本身就依赖持久化状态。没有 checkpointer，就无法正常实现这种可恢复中断。

以前我只知道：

```text
图停住
 ↓
resume
 ↓
继续
```

但这实际上已经意味着：

```text
图在某个位置保存过
```

所以后来学时间旅行的时候，我才发现：

> **我之前已经在用“存档”，只是没有打开“历史存档列表”。**

---

# 十八、D3 的 `resume`，其实就是自动时间旅行

以前：

```python
graph.invoke(
    Command(resume="approve"),
    cfg
)
```

我没有指定：

```text
checkpoint_id
```

但 LangGraph 会根据当前线程的可恢复状态继续执行。

可以理解成：

```text
D3：

游戏自动读取最近一次相关存档
```

而时间旅行：

```text
D4：

我自己把存档列表打开
然后自己选一档
```

所以：

> **D3 是自动挑档。**
>
> **D4 是手动挑档。**

这个连接一旦想明白，time travel 就没有那么神秘了。

---

# 十九、Time Travel 到底是什么？

先看一张最简单的图：

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

LangGraph 会随着执行过程保存 checkpoint。

所以我脑子里应该把它想成：

```text
START
  ↓
[A 执行完]
  ↓
[B 执行完]
  ↓
[C 执行完]
  ↓
END
```

每一个关键时刻都有一个：

```text
checkpoint_id
```

这就是：

> **存档编号。**

---

# 二十、`get_state_history()` 就是打开存档列表

```python
snaps = list(
    graph.get_state_history(config)
)
```

现在我可以看到：

```text
这个 thread 历史上保存过哪些状态？
```

当前官方文档中，history 是按**最新 → 最旧**返回的。

所以：

```text
index = 0
```

不是最早。

而是：

> **最新。**

也就是说：

```text
index 越大
↓
时间越早
```

这个特别容易记反。

---

# 二十一、每个 checkpoint 我只需要先看三个东西

拿到：

```python
s
```

以后先看：

```python
s.config
s.values
s.next
```

我后来给自己总结成：

| 属性        | 我该怎么理解 |
| --------- | ------ |
| `.config` | 钥匙     |
| `.values` | 存档内容   |
| `.next`   | 待办清单   |

这个心智模型非常好用。

---

# 二十二、`config` 是位置，不是内容

例如：

```python
cfg = {
    "configurable": {
        "thread_id": "tt-1"
    }
}
```

这里最重要的是：

```text
thread_id
```

表示：

> 我在哪个线程里。

而某个 checkpoint 的：

```python
snap.config
```

里还带着：

```text
checkpoint_id
```

就变成：

> **我在这个线程的哪一档。**

所以：

```text
state
= 内容

config
= 位置
```

以后看到一大堆：

```text
cfg
target.config
new_cfg
```

不要慌。

本质就是：

> **不同时间点的“存档钥匙”。**

---

# 二十三、`values` 是真正的 state

如果：

```python
s.values["messages"]
```

有：

```text
Human
A
```

说明这一档：

```text
A 已经做了
```

如果里面有：

```text
Human
A
B
```

说明：

```text
A、B 都已经做了
```

所以：

> `.values` 是“当时图里面到底有什么”。

---

# 二十四、`next` 是“这档以后还有谁没执行”

这其实是挑 checkpoint 最有价值的信息。

假设：

```text
START → A → B → C → END
```

历史可能类似：

```text
档案        next

某档        ('a',)
某档        ('b',)
某档        ('c',)
最后        ()
```

如果我要：

> 从 B 开始重新走。

那就应该找：

```python
s.next == ("b",)
```

的 checkpoint。

于是：

```python
target = next(
    s for s in snaps
    if s.next == ("b",)
)
```

这句话的意思就是：

> **找到“B 正准备执行”的那份存档。**

---

# 二十五、为什么 `next == ()` 的存档不能用来回放？

因为：

```text
next == ()
```

就是：

> 没有下一步。

比如：

```text
A
 ↓
B
 ↓
C
 ↓
END
```

最后一个 checkpoint：

```text
next = ()
```

代表：

> 全做完了。

这个时候：

```python
graph.invoke(
    None,
    target.config
)
```

当然没有什么可做。

所以：

> **挑档的时候，首先看 `next`。**

我的判断标准应该是：

```text
我要从谁开始重跑？
        ↓
找 next 里有谁的 checkpoint。
```

---

# 二十六、纯回放：不改任何东西

找到了：

```python
target
```

以后：

```python
graph.invoke(
    None,
    target.config
)
```

这里有两个关键点。

### `None`

表示：

> 不给新的输入。

不是重新发送：

```text
“完整跑一遍 A B C”
```

而是：

> **从这个 checkpoint 继续。**

### `target.config`

表示：

> **从这一个历史存档开始。**

所以整句话就是：

> **读这个存档，然后继续往后玩。**

官方对 time travel replay 的语义也是如此：从历史 checkpoint replay 时，checkpoint 之前已经完成的节点不会再次执行，之后的节点重新执行。

---

# 二十七、最重要的问题：怎么证明它真的没重跑 A？

这里才是这次实验真正让我满意的地方。

因为：

```text
B 跑了
C 跑了
```

并不能证明：

> A 没跑。

也可能实际上：

```text
A
B
C
```

全跑了一遍。

所以我专门给节点打日志：

```python
def step_a(state):
    print("→ A 执行")
```

```python
def step_b(state):
    print("→ B 执行")
```

```python
def step_c(state):
    print("→ C 执行")
```

然后 replay。

我期待看到：

```text
→ B 执行
→ C 执行
```

但真正关键的是：

```text
不能有：

→ A 执行
```

所以最后断言：

```python
assert "→ B 执行" in txt_replay
assert "→ C 执行" in txt_replay
assert "→ A 执行" not in txt_replay
```

这句话非常重要：

> **没有发生某件事，本身也是验证证据。**

---

# 二十八、“跑通”与“验证”是两回事

以前我很容易：

```text
代码跑完
 ↓
输出看起来对
 ↓
“应该没问题”
```

现在我更希望自己形成一个习惯：

```text
先提出机制假设
 ↓
设计一个能区分真假机制的实验
 ↓
写断言
 ↓
让代码自己判定
```

例如：

### 假设 A

> replay 会从 B 开始。

那么应该看到：

```text
B
C
```

### 假设 B

> A 不会重新运行。

那么应该证明：

```text
A 不出现
```

### 假设 C

> fork 不会覆盖原历史。

那么应该证明：

```python
原 checkpoint 还在
```

### 假设 D

> fork 真的产生了新的未来。

那么应该证明：

```text
B 读到了修改后的 way
```

所以：

> **断言不是附属品。**
>
> **断言就是我证明自己理解没错的工具。**

---

# 二十九、为什么消息数也值得检查？

假设目标 checkpoint：

```text
messages = 2
```

里面：

```text
Human
A
```

然后重跑：

```text
B
C
```

两个节点各增加一条。

所以：

```text
2 + 2 = 4
```

于是可以：

```python
assert (
    len(r_replay["messages"])
    ==
    len(target.values["messages"]) + 2
)
```

这不是为了证明数学题。

而是第二条证据：

```text
日志：
没有 A，只有 B/C

消息：
原来 2 条，现在增加 2 条
```

两种证据同时支持：

> **确实从 B 开始。**

---

# 三十、改档分叉：如果过去换一个选择，未来会怎样？

纯 replay 是：

```text
过去是什么
 ↓
原样再走一遍
```

但还有一种更有意思：

```text
过去是什么
 ↓
我改一下过去那个 state
 ↓
看看另一条未来
```

比如 state 里有：

```python
way: str
```

B：

```python
def step_b(state):
    way = state.get("way") or "默认方式"

    print(f"→ B 执行 ({way})")

    return {
        "messages": [
            ("ai", f"B 完成 ({way})")
        ]
    }
```

原来：

```text
way = 默认方式
```

现在：

```python
new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    }
)
```

然后：

```python
graph.invoke(
    None,
    new_cfg
)
```

于是未来变成：

```text
原分支：

A
 ↓
B（默认方式）
 ↓
C


新分支：

A
 ↓
B（另一种方式）
 ↓
C
```

这就是：

> **fork。**

不是改写过去。

而是：

> **从过去长出一个不同的未来。**

---

# 三十一、Replay 和 Fork，一定要分清

|                   | Replay | Fork         |
| ----------------- | ------ | ------------ |
| 从历史 checkpoint 开始 | ✅      | ✅            |
| 修改 state          | ❌      | ✅            |
| 重新执行后半段           | ✅      | ✅            |
| 原历史保留             | ✅      | ✅            |
| 主要用途              | 复现、调试  | what-if、替代方案 |

所以：

```text
Replay
= 读档重玩

Fork
= 读档 + 修改属性 + 重玩
```

这两个概念以后不要混。

---

# 三十二、为什么 `update_state()` 会返回 `new_cfg`？

这个细节也很重要。

不要：

```python
graph.update_state(
    target.config,
    {"way": "另一种方式"}
)

graph.invoke(
    None,
    target.config
)
```

因为你改档以后：

> **新分支有新的 checkpoint。**

所以应该：

```python
new_cfg = graph.update_state(
    target.config,
    {
        "way": "另一种方式"
    }
)
```

然后：

```python
graph.invoke(
    None,
    new_cfg
)
```

记住：

> **旧 config = 旧档。**
>
> **`update_state()` 返回的新 config = 新分支的钥匙。**

---

# 三十三、原分支为什么还能存在？

因为时间旅行不是：

```text
撤销
```

它更像：

```text
复制过去
 ↓
从这里长一个新分支
```

所以：

```python
snaps_after = list(
    graph.get_state_history(cfg)
)
```

应该发现：

```text
历史数量增加了
```

同时：

```python
assert target_checkpoint_id in cids_after
```

说明：

> 原 checkpoint 还存在。

所以这套机制更像 Git：

```text
旧 commit
   │
   ├── 原分支
   │
   └── 新分支
```

而不是：

```text
旧 commit
↓
被覆盖
```

官方持久化 / time-travel 文档也将这种机制描述为 fork：通过更新历史 checkpoint 创建新的执行分支，而原历史仍然保留。

---

# 三十四、这时候我终于看懂了：D4 其实只有两样东西

这是我给自己留下的最重要的一条总结。

整个 time travel 里，看起来有一堆变量：

```text
cfg
snaps
s
target
target.config
r_replay
new_cfg
r_fork
```

但其实只有两类东西：

```text
① state
② config
```

### state

是：

> **图里面的内容。**

例如：

```text
messages
way
audit_note
```

### config

是：

> **图现在站在哪个位置。**

例如：

```text
thread_id
checkpoint_id
```

所以：

```text
state = 内容
config = 位置
```

这句话我希望以后忘了代码时，还能靠它重新推回来。

---

# 三十五、`.config / .values / .next` 三件套

看到：

```python
snap
```

先不要慌。

只记：

```text
snap.config
snap.values
snap.next
```

分别是：

```text
.config
↓
钥匙

.values
↓
内容

.next
↓
待办
```

所以：

```text
拿钥匙
 ↓
打开存档
 ↓
看里面还有谁没做
```

整个时间旅行 API 基本就靠这条心智模型串起来了。

---

# 三十六、这也是为什么时间旅行和 HITL 是同一个故事

到了这里，再回头看：

```text
HITL：

interrupt
 ↓
checkpoint
 ↓
Command(resume)
 ↓
继续
```

和：

```text
Time Travel：

checkpoint
 ↓
get_state_history
 ↓
挑 checkpoint
 ↓
invoke(None, checkpoint.config)
 ↓
继续
```

本质是一样的。

区别只是：

```text
HITL：

LangGraph 自动找到“该恢复哪一档”


Time Travel：

我自己从历史里挑一档
```

所以：

> **HITL 是“自动时间旅行”。**
>
> **Time Travel 是“手动时间旅行”。**

---

# 三十七、但是有一个很大的边界：时间旅行不是撤销现实

这是以后真正做项目必须牢牢记住的一句话。

假设工具已经：

```text
发送邮件
```

然后你：

```text
回到发送之前的 checkpoint
```

再 replay：

```text
再发送一次
```

这并不是：

> 把第一次发送撤销。

而是：

> **又发送了一次。**

所以 checkpoint 管的是：

```text
Agent 的状态与决策历史
```

不是：

```text
现实世界的副作用
```

文件删了：

```text
回档 ≠ 文件恢复
```

钱转了：

```text
回档 ≠ 银行退款
```

邮件发了：

```text
回档 ≠ 邮件撤回
```

所以以后做生产系统时，必须同时考虑：

```text
checkpoint
+
side effect
+
idempotency
+
compensation
```

这几个东西不能混为一谈。

---

# 三十八、今天这个时间旅行实验为什么故意不用 LLM？

这一点其实是我现在越来越喜欢的工程习惯。

如果我要证明：

> **“到底哪些节点重新执行？”**

那么用 LLM 其实会增加噪音：

```text
模型输出可能变化
工具调用可能变化
prompt 可能变化
```

所以实验故意做成：

```text
A → B → C
```

而且：

```text
没有 LLM
没有随机性
没有外部 API
没有复杂条件边
```

这样如果：

```text
→ A 执行
```

真的出现：

> 那就是图真的跑 A 了。

如果：

```text
→ A 执行
```

没有：

> 那就是强证据。

实验不是越真实越好。

有时候：

> **越简单，越容易证明底层机制。**

---

# 三十九、但也不要把这个实验过度外推

今天这个实验只证明：

```text
确定性线性图
```

里的 replay。

它不能自动证明：

```text
复杂 Agent
+
LLM
+
interrupt
+
条件分支
+
并行
+
外部副作用
```

全部都和这个实验完全一样。

尤其是：

> **带 interrupt 的历史 checkpoint replay 会发生什么？**

这个问题应该单独实验。

不能因为：

```text
A → B → C
```

是这样，

就脑补：

```text
decide → approve → tools
```

一定完全一样。

好的工程实验是：

> **只对自己实际验证过的范围下结论。**

---

# 四十、还有一个生产问题：checkpoint 会越来越多

今天：

```python
InMemorySaver()
```

非常适合学习。

因为：

```text
跑一次
 ↓
存起来
 ↓
随时看
 ↓
随时回放
```

非常爽。

但如果以后：

```text
10000 用户
×
长对话
×
大量 super-step
×
大量 checkpoint
```

那持久化存储一定会增长。

所以：

> “历史全部保留”

是调试能力。

但：

> “生产无限保留”

就要另外考虑存储成本、保留策略和清理策略。

这也是我以后做项目时需要继续学习的一层。

---

# 四十一、现在把整个 D4 压缩成五句话

如果以后完全忘了代码，只记：

### 1.

> `decide` 负责产生工具调用，并先把结果写进 state。

### 2.

> `approve` 负责暂停问人；`edit` 真正修改的是 state 里的 pending `tool_calls`。

### 3.

> `audit` 负责告诉模型“为什么参数变了”，解决上下文断裂。

### 4.

> `summarize` 不绑定工具，从结构上切断“执行后再次调用工具”的路径。

### 5.

> checkpoint 是存档，`get_state_history()` 是存档列表，`checkpoint.config` 是读档钥匙；replay 是原样读档，`update_state()` 后再运行是 fork。

---

# 四十二、最后，我想留下一个“疑惑 → 答案”速查表

以后重新看这篇文章的时候，我大概率是忘了某个局部。

那就直接查这里。

| 我的疑惑                                            | 应该想起什么                                     |
| ----------------------------------------------- | ------------------------------------------ |
| `interrupt()` 之前的代码会不会再执行？                      | 会，resume 时所在节点从头执行                         |
| 为什么 `LLM + interrupt` 要拆节点？                     | 先让 AIMessage 落进 state，再中断                  |
| 人改的是什么？                                         | pending `AIMessage.tool_calls`             |
| 为什么保留 `id`？                                     | 让 `add_messages` 更新原消息而不是追加                |
| `edit` 为什么没独立分支？                                | edit 在图外 `update_state()` 完成，approve 只负责放行 |
| 模型为什么不知道人改过？                                    | 图外动作没进入 transcript/state                   |
| `audit` 做什么？                                    | 补因果链，治叙事                                   |
| 为什么最后不能只靠 prompt？                               | Prompt 是软约束，不是锁                            |
| 为什么 summarize 不绑定工具？                            | 从结构上消灭再次调用工具的能力                            |
| `checkpoint_id` 是什么？                            | 某一份历史存档的编号                                 |
| `get_state_history()` 是什么？                      | 查看整条历史时间线                                  |
| `next` 是什么？                                     | 这份存档之后还有哪些节点要执行                            |
| 为什么 `next == ()` 不适合 replay？                    | 已经没有下一步                                    |
| `invoke(None, snap.config)` 是什么？                | 从这个 checkpoint 纯回放                         |
| `update_state()` + `invoke(None, new_cfg)` 是什么？ | 改档分叉                                       |
| 怎么证明没从头跑？                                       | 强断言：目标节点出现，前置节点不出现                         |
| Time Travel 是撤销现实吗？                             | 不是，只能回到 Agent 的历史状态；现实副作用不会自动回滚            |

---

# 四十三、我现在真正应该记住的，不是 API

以前我会想：

> “今天我学了 `interrupt`、`Command`、`update_state`、`get_state_history`……”

这种记法很容易过几天全部忘掉。

现在我更希望自己记住的是：

```text
模型想做事
 ↓
先形成一张“待执行工单”
 ↓
人可以在执行前接管这张工单
 ↓
人工修改必须留下因果记录
 ↓
执行之后要结构性收口
 ↓
整个过程都被 checkpoint 保存
 ↓
所以过去可以被重新打开
 ↓
打开后可以原样重玩
 ↓
也可以改档后探索另一条未来
```

这才是今天真正学到的东西。

---

# 四十四、最后一张图

把整篇文章压成一张图：

```text
                         ┌──────────────┐
                         │    decide    │
                         │   LLM+Tools  │
                         └──────┬───────┘
                                │
                          产生 tool_call
                                │
                                ▼
                         ┌──────────────┐
                         │    approve   │
                         │   interrupt  │
                         └──────┬───────┘
                                │
                    ┌───────────┼───────────┐
                    │           │           │
                 approve      reject       edit
                    │           │           │
                    │           ▼           │
                    │          END          │
                    │                       │
                    └───────────┬───────────┘
                                │
                           update_state
                            （改工单）
                                │
                                ▼
                         ┌──────────────┐
                         │    tools     │
                         │   真执行     │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │    audit     │
                         │  记录人工改动 │
                         └──────┬───────┘
                                │
                                ▼
                         ┌──────────────┐
                         │  summarize   │
                         │   无工具     │
                         └──────┬───────┘
                                │
                                ▼
                               END

                                │
                         checkpoint 持久化
                                │
                                ▼
                    ┌────────────────────────┐
                    │    历史存档时间线       │
                    └───────────┬────────────┘
                                │
                  ┌─────────────┴─────────────┐
                  │                           │
                Replay                      Fork
                  │                           │
           原样读档重跑                  改档再运行
                  │                           │
                  ▼                           ▼
              B → C                       B' → C'
```

---

# 四十五、完整源码

下面代码我故意保留了“学习版”的写法：

> **宁可多几行注释，也不要追求短。**

因为这里的代码不是拿去生产的，而是为了以后重新看时，能一眼知道：

> “这里到底为什么这么写？”

---

## 源码一：D4 HITL —— 三态审批 + 改参 + 审计 + 结构收口

保存成：

```text
w6_d4_hitl.py
```

依赖：

```bash
pip install langgraph langchain-core langchain-openai python-dotenv
```

环境变量：

```env
DEEPSEEK_API_KEY=你的key
DEEPSEEK_MODEL=deepseek-v4-flash
```

```python
"""
W6-D4：LangGraph HITL 学习版

今天只学 4 件事：

1. interrupt()：让图停下来问人
2. edit：修改 pending tool_calls
3. audit：告诉模型“这是人主动改的”
4. summarize：最后不绑定工具，结构性收口

注意：
这是学习/演示代码。
delete_file() 故意只 print，不真的删除文件。
"""

import os

from dotenv import load_dotenv

# 读取 .env
load_dotenv(override=True)


# ============================================================
# 1. LangGraph / LangChain imports
# ============================================================

from typing import Annotated, TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

# 当前 LangGraph 常用的内存 checkpointer
from langgraph.checkpoint.memory import InMemorySaver

# 工具节点：拿到 AIMessage.tool_calls 后真正执行工具
from langgraph.prebuilt import ToolNode

# interrupt：暂停图
# Command：恢复 interrupt
from langgraph.types import interrupt, Command

from langchain_core.tools import tool
from langchain_core.messages import AIMessage, HumanMessage

from langchain_openai import ChatOpenAI


# ============================================================
# 2. 定义危险工具
# ============================================================

@tool
def delete_file(path: str) -> str:
    """
    危险操作：删除文件。

    这里只是演示：
    不会真的删除文件，只打印“工具真身执行”。
    """

    print(f">>> 工具真身执行：删除 {path}")

    return f"文件已删除：{path}"


# ============================================================
# 3. 定义 State
# ============================================================

class S(TypedDict):
    # messages 是核心对话状态。
    #
    # add_messages 很重要：
    # LangGraph 会按照 message id 等规则处理消息更新。
    messages: Annotated[list, add_messages]

    # 普通 state 字段。
    #
    # 用来记录：
    # “审批人到底做了什么修改？”
    #
    # 它不是 messages reducer。
    # 每次 node return 新值时，普通字段可以直接覆盖。
    audit_note: str


# ============================================================
# 4. 两个模型：故意分工
# ============================================================

_common = {
    # 从环境变量读取模型。
    # 默认沿用你现在实验使用的模型名。
    "model": os.getenv(
        "DEEPSEEK_MODEL",
        "deepseek-v4-flash",
    ),

    "api_key": os.getenv("DEEPSEEK_API_KEY"),

    "base_url": os.getenv(
        "DEEPSEEK_BASE_URL",
        "https://api.deepseek.com",
    ),

    "temperature": 0,
}


# ------------------------------
# 决策模型
# ------------------------------
#
# 这个模型“有工具箱”。
#
# 它可以产生：
#
# AIMessage(
#   tool_calls=[...]
# )
#
llm_decide = ChatOpenAI(
    **_common
).bind_tools(
    [delete_file]
)


# ------------------------------
# 收口模型
# ------------------------------
#
# 这里故意没有 bind_tools。
#
# 它可以总结：
#
# “文件已经删除……”
#
# 但它不能：
#
# “我再调用 delete_file 一次。”
#
llm_summarize = ChatOpenAI(**_common)


# ============================================================
# 5. decide：决策官
# ============================================================

def decide(state: S) -> dict:
    """
    只负责：

    “模型觉得下一步应该做什么？”

    注意：
    这里没有 interrupt。

    为什么？
    因为我要先让 AIMessage 真正 return，
    进入 state/checkpoint。

    如果这里直接 interrupt，
    ai_msg 还只是 Python 局部变量，
    还没有进入 state。
    """

    ai_msg = llm_decide.invoke(
        state["messages"]
    )

    print(
        f"[decide] tool_calls="
        f"{len(ai_msg.tool_calls)}"
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 6. approve：审批官
# ============================================================

def approve(state: S) -> dict:
    """
    只负责：

    “模型刚才这张工单，人批不批？”

    注意：
    这个节点会被 interrupt/resume 机制重跑。

    所以这里不要放不可安全重放的副作用。
    """

    # 读取上一条消息。
    last = state["messages"][-1]

    # 如果上一条不是带 tool_calls 的 AIMessage，
    # 理论上就不应该走到这里。
    if not (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return {}

    # 取出模型提出的第一条工具调用。
    tc = last.tool_calls[0]

    # 打印出来，帮助我们观察运行过程。
    print(
        f"[approve] 待审批参数："
        f"{tc['args']}"
    )

    # --------------------------------------------------------
    # interrupt：
    #
    # 第一次执行到这里：
    #   图停下来
    #
    # resume 以后：
    #   当前 approve 节点会从头重跑
    #   重新来到 interrupt()
    #   这一次 interrupt() 返回 resume 值
    # --------------------------------------------------------

    answer = interrupt(
        {
            "question": (
                f"模型想删除 "
                f"{tc['args']['path']}，"
                f"批准吗？"
            ),
            "options": [
                "approve",
                "reject",
                "edit",
            ],
        }
    )

    # --------------------------------------------------------
    # reject：
    #
    # 我们故意返回一条“没有 tool_calls”的 AIMessage。
    #
    # 后面的 route_after_approve()
    # 看到最后一条消息已经没有 tool_calls，
    # 就会直接 END。
    # --------------------------------------------------------

    if answer == "reject":

        return {
            "messages": [
                AIMessage(
                    content=(
                        "已取消删除，"
                        "本次未执行任何操作。"
                    )
                )
            ]
        }

    # --------------------------------------------------------
    # approve / edit：
    #
    # 两者都走这里。
    #
    # 为什么 edit 没有自己的 if？
    #
    # 因为真正“修改 tool_calls”的动作
    # 已经在图外通过 update_state() 完成。
    #
    # approve 节点这里只负责：
    # “我不拦。”
    # --------------------------------------------------------

    return {}


# ============================================================
# 7. audit：纪要员
# ============================================================

def audit(state: S) -> dict:
    """
    解决的问题：

    “模型为什么看到的参数和原来不一样？”

    因为人可能在图外通过 update_state() 修改了参数。

    所以这里把“人的操作”补回 transcript，
    让模型知道：

    “不是你搞错了，是审批人主动改的。”
    """

    note = state.get(
        "audit_note",
        "",
    )

    # 没有审计记录，就什么也不做。
    if not note:
        return {}

    print(
        f"[audit] 注入留痕：{note}"
    )

    # --------------------------------------------------------
    # 把 audit_note 转成模型能看到的消息。
    #
    # 这里使用 HumanMessage 只是为了演示简单。
    #
    # 生产系统里应该更谨慎地区分：
    # “真实用户输入”
    # 和
    # “系统注入的审计事件”。
    # --------------------------------------------------------

    audit_message = HumanMessage(
        content=(
            f"【审批人操作记录】{note}。"
            "这是审批人主动、有意做出的决定，"
            "已获批准执行；"
            "这不是错误，无需道歉、"
            "无需重试其他路径，"
            "请直接据实向用户总结结果。"
        )
    )

    return {
        "messages": [audit_message],

        # 用完清空。
        #
        # 这样下一次流程不会重复注入同一条 audit。
        "audit_note": "",
    }


# ============================================================
# 8. summarize：签字官
# ============================================================

def summarize(state: S) -> dict:
    """
    最后只允许“说话”，不允许再调用工具。

    这是整个 D4 “结构收口”最重要的地方。
    """

    ai_msg = llm_summarize.invoke(
        state["messages"]
    )

    # 因为这个模型根本没有工具，
    # 所以这里从结构上保证：
    # 最终只产生普通 AIMessage。
    print(
        "[summarize] 收口完成，"
        "tool_calls=0"
    )

    return {
        "messages": [ai_msg]
    }


# ============================================================
# 9. decide 后的路由
# ============================================================

def route_after_decide(state: S):
    """
    如果模型产生 tool_calls：
        → approve

    如果模型只是普通回答：
        → END
    """

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "approve"

    return END


# ============================================================
# 10. approve 后的路由
# ============================================================

def route_after_approve(state: S):
    """
    如果最后一条消息仍然带 tool_calls：
        → tools

    如果没有：
        → END

    reject 时：
        approve 返回了一条没有 tool_calls 的消息
        所以这里会 END。

    approve / edit 时：
        原来的 AIMessage 仍然存在并带 tool_calls
        所以这里会进入 tools。
    """

    last = state["messages"][-1]

    if (
        isinstance(last, AIMessage)
        and last.tool_calls
    ):
        return "tools"

    return END


# ============================================================
# 11. 构建 graph
# ============================================================

builder = StateGraph(S)


# 注册节点
builder.add_node("decide", decide)
builder.add_node("approve", approve)

# ToolNode：
# 真正读取 AIMessage.tool_calls
# 并执行 delete_file
builder.add_node(
    "tools",
    ToolNode([delete_file]),
)

builder.add_node("audit", audit)
builder.add_node("summarize", summarize)


# ------------------------------------------------------------
# START → decide
# ------------------------------------------------------------

builder.add_edge(
    START,
    "decide",
)


# ------------------------------------------------------------
# decide → approve / END
# ------------------------------------------------------------

builder.add_conditional_edges(
    "decide",
    route_after_decide,
    {
        "approve": "approve",
        END: END,
    },
)


# ------------------------------------------------------------
# approve → tools / END
# ------------------------------------------------------------

builder.add_conditional_edges(
    "approve",
    route_after_approve,
    {
        "tools": "tools",
        END: END,
    },
)


# ------------------------------------------------------------
# tools → audit → summarize → END
# ------------------------------------------------------------

builder.add_edge(
    "tools",
    "audit",
)

builder.add_edge(
    "audit",
    "summarize",
)

builder.add_edge(
    "summarize",
    END,
)


# ============================================================
# 12. 编译
# ============================================================
#
# interrupt 和 time travel 都需要 checkpointer。
#
# 学习阶段使用 InMemorySaver。
#

checkpointer = InMemorySaver()

graph = builder.compile(
    checkpointer=checkpointer,
)


# ============================================================
# 13. 场景 C：修改参数
# ============================================================

def run_edit_demo():
    """
    演示：

    用户：
        删除 /tmp/a.txt

    人：
        edit

    最终：
        实际执行 /tmp/old.txt
    """

    cfg = {
        "configurable": {
            "thread_id": "w6-d4-edit",
        }
    }

    # --------------------------------------------------------
    # 第一次调用：
    #
    # graph 会运行：
    #
    # START → decide → approve → interrupt
    #
    # 然后停在 approve。
    # --------------------------------------------------------

    print("\n" + "=" * 60)
    print("第一次运行：让图停在 approve")
    print("=" * 60)

    graph.invoke(
        {
            "messages": [
                (
                    "user",
                    "帮我删除 /tmp/a.txt",
                )
            ],

            # 普通字段最好显式初始化，
            # 让 state 结构更加清楚。
            "audit_note": "",
        },
        cfg,
    )

    # --------------------------------------------------------
    # 此时图已经停下来了。
    #
    # get_state()：
    # 只是查看。
    # 不会继续执行。
    # --------------------------------------------------------

    st = graph.get_state(cfg)

    print("\n当前 next =", st.next)

    # 当前最后一条消息应该是：
    # AIMessage(tool_calls=[...])
    last = st.values["messages"][-1]

    print(
        "当前最后一条消息类型：",
        type(last).__name__,
    )

    print(
        "当前待执行 tool_calls：",
        last.tool_calls,
    )

    # --------------------------------------------------------
    # 复制原来的 tool_calls。
    #
    # 这里只修改 path，
    # 其它参数保持不变。
    # --------------------------------------------------------

    new_tool_calls = [
        {
            **tc,
            "args": {
                **tc["args"],
                "path": "/tmp/old.txt",
            },
        }
        for tc in last.tool_calls
    ]

    # --------------------------------------------------------
    # 创建“修改后的 AIMessage”。
    #
    # 非常关键：
    #
    # id=last.id
    #
    # 这样 add_messages 才知道：
    # 这是在替换原消息。
    #
    # 如果完全新建一条不带原 id 的消息，
    # 可能变成追加而不是替换。
    # --------------------------------------------------------

    fixed = AIMessage(
        id=last.id,
        content=last.content,
        tool_calls=new_tool_calls,
    )

    # --------------------------------------------------------
    # 一次 update_state 同时完成两件事：
    #
    # 1. 修改待执行工单
    # 2. 写入审计记录
    #
    # 这样“发生了什么”和“为什么发生”
    # 一起落盘。
    # --------------------------------------------------------

    graph.update_state(
        cfg,
        {
            "messages": [fixed],

            "audit_note": (
                "审批人把删除目标从 "
                "/tmp/a.txt 改成了 "
                "/tmp/old.txt"
            ),
        },
    )

    # --------------------------------------------------------
    # 现在才 resume。
    #
    # 注意：
    # update_state() 只是改状态。
    #
    # 它不会自动把图继续跑下去。
    # --------------------------------------------------------

    print("\n" + "=" * 60)
    print("恢复执行：resume('edit')")
    print("=" * 60)

    result = graph.invoke(
        Command(resume="edit"),
        cfg,
    )

    # 最终总结
    print(
        "\n最终结果：",
        result["messages"][-1].content,
    )

    # --------------------------------------------------------
    # 最终收敛检查
    # --------------------------------------------------------

    final_state = graph.get_state(cfg)

    print(
        "最终 next =",
        final_state.next,
    )

    print(
        "最后一条 tool_calls =",
        getattr(
            final_state.values["messages"][-1],
            "tool_calls",
            None,
        ),
    )


if __name__ == "__main__":
    run_edit_demo()
```

---

# 四十六、源码二：Time Travel —— Replay + Fork + 断言

这份我故意**完全不用 LLM**。

因为今天真正想验证的是：

> **到底哪些节点被重新执行？**

所以越确定性越好。

保存：

```text
w6_d4_time_travel.py
```

```python
"""
W6-D4：Time Travel 学习实验

目标：

1. 先完整跑 A → B → C
2. 查看全部 checkpoint
3. 找到 next == ("b",) 的历史存档
4. 从这里 replay
5. 证明：
   - B 重跑
   - C 重跑
   - A 没重跑
6. 再从同一个过去 checkpoint fork
7. 修改 way
8. 证明未来的 B 发生变化
9. 证明原 checkpoint 没消失

整个实验不用 LLM。
这样更容易验证底层机制。
"""

from typing import Annotated, TypedDict

from langgraph.graph import (
    StateGraph,
    START,
    END,
)

from langgraph.graph.message import add_messages

from langgraph.checkpoint.memory import InMemorySaver


# ============================================================
# 1. State
# ============================================================

class T(TypedDict):
    # 对话消息。
    #
    # add_messages：
    # 新节点返回的消息会追加/更新到消息历史。
    messages: Annotated[list, add_messages]

    # 普通字段。
    #
    # 这个字段专门拿来演示：
    #
    # “如果过去这个 state 被改掉，
    #  那么未来节点会不会跟着变化？”
    way: str


# ============================================================
# 2. A
# ============================================================

def step_a(state: T) -> dict:
    """
    A 节点。

    时间旅行 replay 时，
    如果 A 没有被重新执行，
    那么日志里就不应该出现：

        → A 执行
    """

    print("→ A 执行")

    return {
        "messages": [
            ("ai", "A 完成")
        ]
    }


# ============================================================
# 3. B
# ============================================================

def step_b(state: T) -> dict:
    """
    B 节点。

    它会读取 state["way"]。

    原始运行：
        默认方式

    fork 修改后：
        另一种方式

    所以 B 是我们验证 fork 是否成功的节点。
    """

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


# ============================================================
# 4. C
# ============================================================

def step_c(state: T) -> dict:
    """
    C 节点。

    同样作为 replay 的证明：

        B
        ↓
        C
    """

    print("→ C 执行")

    return {
        "messages": [
            ("ai", "C 完成")
        ]
    }


# ============================================================
# 5. 构图
# ============================================================

builder = StateGraph(T)

builder.add_node("a", step_a)
builder.add_node("b", step_b)
builder.add_node("c", step_c)

# 线性图：

# START
#   ↓
#   A
#   ↓
#   B
#   ↓
#   C
#   ↓
#  END

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
# 6. 编译 + checkpointer
# ============================================================

checkpointer = InMemorySaver()

graph = builder.compile(
    checkpointer=checkpointer,
)


# ============================================================
# 7. thread config
# ============================================================

cfg = {
    "configurable": {
        "thread_id": "w6-d4-time-travel"
    }
}


# ============================================================
# 8. 第一次：完整跑一遍
# ============================================================

print("\n" + "=" * 60)
print("第一次：完整运行 A → B → C")
print("=" * 60)

result = graph.invoke(
    {
        "messages": [
            (
                "user",
                "完整跑一遍 A B C",
            )
        ],

        "way": "",
    },
    cfg,
)

print(
    "\n原始运行后的消息数：",
    len(result["messages"]),
)

# 预期：
#
# Human
# A
# B
# C
#
# = 4 条


# ============================================================
# 9. 查看历史 checkpoint
# ============================================================

print("\n" + "=" * 60)
print("历史 checkpoint：新 → 旧")
print("=" * 60)

snaps = list(
    graph.get_state_history(cfg)
)

print(
    f"共有 {len(snaps)} 个 checkpoint"
)


for i, snap in enumerate(snaps):

    checkpoint_id = (
        snap.config[
            "configurable"
        ][
            "checkpoint_id"
        ]
    )

    print(
        f"#{i:02d} "
        f"cid={checkpoint_id[:8]} "
        f"next={snap.next} "
        f"messages="
        f"{len(snap.values['messages'])}"
    )


# ============================================================
# 10. 找“下一步是 B”的 checkpoint
# ============================================================
#
# 这是挑档的关键。
#
# 如果：
#
#   next == ("b",)
#
# 就意味着：
#
#   A 已经做完
#   B 还没做
#
# 所以从这里继续：
#
#   会跑 B → C
#
# 不应该再跑 A。
# ============================================================

target = next(
    snap
    for snap in snaps
    if snap.next == ("b",)
)

target_checkpoint_id = (
    target.config[
        "configurable"
    ][
        "checkpoint_id"
    ]
)

print("\n" + "=" * 60)
print("挑中的目标 checkpoint")
print("=" * 60)

print(
    "checkpoint_id =",
    target_checkpoint_id,
)

print(
    "next =",
    target.next,
)

print(
    "messages =",
    len(
        target.values["messages"]
    ),
)


# ============================================================
# 11. Replay：从这个历史 checkpoint 原样继续
# ============================================================

print("\n" + "=" * 60)
print("Replay：从 B 前面的 checkpoint 继续")
print("=" * 60)

# ------------------------------------------------------------
# 由于我们想“证明日志里有没有 A”，
# 所以这里不用复杂的日志采集。
# 直接让节点 print。
#
# 如果你的测试框架需要：
# 可以把 stdout 捕获下来做 assert。
# ------------------------------------------------------------

import io
import contextlib

replay_log = io.StringIO()

with contextlib.redirect_stdout(
    replay_log
):
    replay_result = graph.invoke(
        None,

        # ★ 关键：
        #
        # 直接使用 checkpoint 自己的 config。
        #
        # 不要自己重新拼 checkpoint_id。
        # snap.config 已经是完整读档钥匙。
        target.config,
    )


txt_replay = replay_log.getvalue()

print(
    txt_replay.strip()
)

print(
    "Replay 后消息数：",
    len(
        replay_result["messages"]
    ),
)


# ============================================================
# 12. 强断言：证明不是从头跑
# ============================================================

print("\n" + "=" * 60)
print("Replay 断言")
print("=" * 60)

# B 必须执行。
assert (
    "→ B 执行 (默认方式)"
    in txt_replay
), "B 没有重跑"


# C 必须执行。
assert (
    "→ C 执行"
    in txt_replay
), "C 没有重跑"


# ------------------------------------------------------------
# 最关键的一条：
#
# A 绝对不能出现。
#
# 如果出现：
#
#   → A 执行
#
# 就说明你并没有从 B 前面的 checkpoint replay，
# 而是从头跑了。
# ------------------------------------------------------------

assert (
    "→ A 执行"
    not in txt_replay
), (
    "A 竟然重跑了："
    "这不是我们想证明的 replay"
)


# ------------------------------------------------------------
# 原 checkpoint 如果有 2 条消息：
#
#   Human
#   A
#
# Replay 后：
#
#   Human
#   A
#   B
#   C
#
# 应该新增 2 条。
# ------------------------------------------------------------

assert (
    len(replay_result["messages"])
    ==
    len(
        target.values["messages"]
    ) + 2
), (
    "Replay 新增消息数量不对"
)


print(
    "✅ Replay 断言全部通过"
)

print(
    "关键证据："
    "B/C 出现，A 没出现。"
)


# ============================================================
# 13. Fork：先改档，再继续
# ============================================================

print("\n" + "=" * 60)
print("Fork：从同一个历史 checkpoint 改出另一条未来")
print("=" * 60)

# ------------------------------------------------------------
# 在历史 checkpoint 上改 state：
#
# way：
#   默认方式
#
# ↓
#
# way：
#   另一种方式
#
# update_state() 返回“新分支”的 config。
# ------------------------------------------------------------

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
    "原 checkpoint =",
    target_checkpoint_id,
)

print(
    "新 checkpoint =",
    new_checkpoint_id,
)


# ============================================================
# 14. 从新分支继续
# ============================================================

fork_log = io.StringIO()

with contextlib.redirect_stdout(
    fork_log
):

    fork_result = graph.invoke(
        None,

        # ★ 注意：
        #
        # 这里一定使用 update_state()
        # 的返回值 new_cfg。
        #
        # 因为这个 new_cfg 指向新的 fork。
        #
        new_cfg,
    )


txt_fork = fork_log.getvalue()

print(
    txt_fork.strip()
)

print(
    "Fork 后消息数：",
    len(
        fork_result["messages"]
    ),
)


# ============================================================
# 15. Fork 断言
# ============================================================

print("\n" + "=" * 60)
print("Fork 断言")
print("=" * 60)


# B 必须看到修改后的 way。
assert (
    "→ B 执行 (另一种方式)"
    in txt_fork
), "fork 修改后的 way 没生效"


# A 仍然不能重跑。
assert (
    "→ A 执行"
    not in txt_fork
), (
    "Fork 不应该把 A 从头重新执行"
)


# 最终消息也应该带上修改后的 way。
assert (
    "另一种方式"
    in
    fork_result[
        "messages"
    ][-1].content
), (
    "最终 state 中没有带上 fork 修改"
)


print(
    "✅ Fork 断言全部通过"
)


# ============================================================
# 16. 验证：原 checkpoint 仍然存在
# ============================================================
#
# 这是“分叉”而不是“覆盖”的证据。
# ============================================================

print("\n" + "=" * 60)
print("验证原分支仍然存在")
print("=" * 60)

snaps_after = list(
    graph.get_state_history(cfg)
)

checkpoint_ids_after = [
    snap.config[
        "configurable"
    ][
        "checkpoint_id"
    ]
    for snap in snaps_after
]


# 原 checkpoint 还存在。
assert (
    target_checkpoint_id
    in checkpoint_ids_after
), (
    "原 checkpoint 消失了："
    "这不符合我们预期的 fork 行为"
)


# checkpoint 数量应该增加。
assert (
    len(snaps_after)
    >
    len(snaps)
), (
    "Fork 后 checkpoint 数量没有增加"
)


print(
    "✅ 原 checkpoint 仍然存在"
)

print(
    "✅ Fork 产生了新的历史分支"
)


# ============================================================
# 17. 最终总结
# ============================================================

print("\n" + "=" * 60)
print("最终结论")
print("=" * 60)

print(
    "Replay："
    "从历史 checkpoint 原样重跑后半段。"
)

print(
    "Fork："
    "从历史 checkpoint 修改 state，"
    "再产生不同的未来。"
)

print(
    "最关键的 Replay 证据："
    "日志里没有 → A 执行。"
)

print(
    "✅ W6-D4 Time Travel 实验完成"
)
```

---

# 四十七、运行顺序

我以后重新学这一天的时候，建议固定按这个顺序：

### 第一步

先看文章里的：

```text
decide
→ approve
→ tools
→ audit
→ summarize
```

不要先看 200 行代码。

### 第二步

再跑：

```bash
python w6_d4_hitl.py
```

重点观察：

```text
[decide]
[approve]
>>> 工具真身执行
[audit]
[summarize]
```

然后思考：

> 为什么 approve 会再出现一次？

---

### 第三步

再跑：

```bash
python w6_d4_time_travel.py
```

重点不是看最后一句：

```text
✅
```

而是看：

```text
Replay：

→ B 执行
→ C 执行
```

同时确认：

```text
没有：

→ A 执行
```

---

# 四十八、以后忘记这一天的时候，只看这张“极简卡片”

```text
HITL
────────────────────────────

decide
  ↓
生成 tool_call

approve
  ↓
interrupt()
  ↓
approve / reject / edit

edit
  ↓
update_state()
  ↓
改 pending tool_call

tools
  ↓
真正执行

audit
  ↓
告诉模型：
“这是人改的”

summarize
  ↓
不绑定工具
  ↓
结构收口
```

然后：

```text
Time Travel
────────────────────────────

checkpoint
  ↓
存档

get_state_history()
  ↓
看历史

snap.next
  ↓
看下一步该谁

snap.config
  ↓
拿读档钥匙

invoke(None, snap.config)
  ↓
Replay

update_state(snap.config, ...)
  ↓
改档

invoke(None, new_cfg)
  ↓
Fork
```

最后只记一个验证习惯：

> **不要只证明“该发生的发生了”，还要证明“绝对不该发生的没有发生”。**

所以：

```python
assert "→ B 执行" in txt_replay
assert "→ C 执行" in txt_replay
assert "→ A 执行" not in txt_replay
```

这一条，比我单纯记住 `get_state_history()` 更值得留下来。

---

# 四十九、给未来的我

如果以后又觉得 LangGraph 的代码很乱，先不要继续往下读。

先问自己三个问题：

```text
1. 现在的 state 是什么？
2. 我现在站在哪个 checkpoint？
3. 下一步是谁？
```

然后再问：

```text
我要：

继续？
暂停？
改档？
回放？
分叉？
```

因为不管今天的代码变得多复杂，底层仍然是在处理：

```text
State
+
位置
+
下一步
```

而这一轮真正学到的，是：

> **Agent 不只是“向前跑”。**
>
> **它可以停下来，让人接管；可以把每一步存下来；可以回到过去重新观察；也可以从过去长出另一条未来。**
>
> 而工程化真正重要的是：**我不仅知道它应该这么跑，我还能设计实验和判据，证明它确实这么跑了。**
