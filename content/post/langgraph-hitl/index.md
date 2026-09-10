---
description: ""
title: " LangGraph_HITL"
draft: false
date: "2026-09-10T09:54:52+08:00"
slug: "LangGraph_HITL"
categories:
 - 
tags:
 - 
image: ""
---

# LangGraph_HITL

> W6 · Day 4 · 审批三态 · update_state 改参 · 时间旅行
> 阅读时长约 15 分钟

D3 搭出来的那道人机审批，只有「过」和「不过」两态——像个门卫，只会放行或拦人。但真实的审批场景里，人还想要第三种权力：**「不是删这个文件，是删那个」**。

而 LangGraph 里「模型想调工具」这件事，在工具真正执行之前，只是 state 里一条 `AIMessage` 身上的 `tool_calls` 字段——**它是可以被改的**。

这中间隔着一层「怎么在工具执行前，把人改过的参数塞回去，并且让模型知道这是人改的」。

D4 要做的，就是补上这层。而补这层只需要三件东西，刚好对应下面第五节的三个新节点：

- `update_state` = 图外改档（**涂改笔**：改 pending 的工具参数）
- `audit` 节点 = 审计留痕（**登记簿**：给模型看的审批记录）
- `summarize` 节点 = 结构收口（**拆按钮**：不绑工具的 LLM，物理上产不出 `tool_calls`）

> 一句话：**D3 的门卫升级成能改单子的审批员——能放行、能打回、能手改单子再放行，外加一台时光机（时间旅行）回档。**

* * *

## 目录

1. [一、大局：审批接在图的哪个位置](#一)
2. [二、`interrupt()` 是「抛异常」不是「return」：为什么必须拆节点](#二)
3. [三、`update_state` 改参：复制一份再改 + `id` 生死线](#三)
4. [四、`get_state` vs `invoke`：读状态别把图跑起来](#四)
5. [五、模型说「我误删了」：一次审计缺失事故](#五)
6. [六、「看起来跑完了」≠ 跑完了：`next` 判据 + 结构收口](#六)
7. [七、完整可运行代码 + 8 项断言](#七)
8. [八、时间旅行：`checkpoint_id` = 存档点](#八)
9. [九、踩坑记 & 知识点](#九)
10. [十、总结](#十)
11. [十一、下一篇预告](#十一)

* * *

<h2 id="一">一、大局：审批接在图的哪个位置</h2>

先看整张图的数据流向——审批是「决策之后、执行之前」的那道闸：

```text
用户输入 {"messages": [("user", "帮我删除 /tmp/a.txt")]}
   ↓
decide (绑工具的 LLM)      ← ① 模型决策：产出带 tool_calls 的 AIMessage，★必须 return
   ↓ 有 tool_calls?
approve (interrupt 问人)   ← ② 踩刹车：放行 / 拒绝 / 放行「改过之后」的参数
   ↓ 放行?
tools (ToolNode)           ← ③ 工具真身执行（这一行 print 是全部判据的来源）
   ↓
audit (注入审批留痕)        ← ④ 治叙事：让模型「看得见」人改过
   ↓
summarize (不绑工具的 LLM)  ← ⑤ 治重试：物理上产不出 tool_calls → 必进 END
   ↓
END
```

拆开看五个部件各自在哪、干嘛：

| 部件 | 角色 | 一句话 |
|---|---|---|
| `decide` | 决策 | 只管「想」——调 LLM，**必须 return**，AIMessage 才有机会落进 state |
| `approve` | 审批 | 只管「问人」——`interrupt()` 挂起，放行就原样返回，拒绝就返回不带 `tool_calls` 的消息 |
| `tools` | 执行 | `ToolNode`，工具真身在这里跑 |
| `audit` | 留痕 | 把「人改过参数」写成一条模型看得见的消息 |
| `summarize` | 收口 | **不绑工具的 LLM** → 结构上不可能再调工具 → 必然 END |

对比 D3 那张「agent → tools → agent」的两节点图，D4 多出来的三个节点各有各的病要治：

- `decide` / `approve` 拆开 → 治「改参时 `st.values["messages"][-1]` 是 `HumanMessage`」的报错
- `audit` → 治「模型编出『我误删了文件』这种灾难性叙事」
- `summarize` → 治「模型自纠、想再删一次，图卡在第二次审批上空转」

接下来逐个拆。

* * *

<h2 id="二">二、`interrupt()` 是「抛异常」不是「return」：为什么必须拆节点</h2>

**一句话：`interrupt()` 让节点从头重跑，而你在 `interrupt()` 之前算出来的东西，如果没 `return` 过，就从来没进过 state。**

### 报错现场

D4 第一版代码是把「决策」和「问人」塞进同一个节点的：

```python
def agent_node(state: S) -> dict:
    ai_msg = llm.invoke(state["messages"])          # ① 算出来了，但还没交出去
    if ai_msg.tool_calls and ai_msg.tool_calls[0]["name"] == "delete_file":
        decision = interrupt({                       # ② 喊停
            "question": f"模型想删除 {ai_msg.tool_calls[0]['args']['path']}, 批准吗?",
            "options": ["approve", "reject", "edit"],
        })
        ...
    return {"messages": [ai_msg]}
```

然后外面想改参数：

```python
st = graph.get_state(cfg_c)
last = st.values["messages"][-1]     # ❌ 拿到的是 HumanMessage
new_tc = [{**tc, "args": {"path": "/tmp/old.txt"}} for tc in last.tool_calls]
# AttributeError: 'HumanMessage' object has no attribute 'tool_calls'
```

### 两条铁证

你的终端输出里其实已经写了答案，只是当时没读出来：

| 证据 | 说明 |
|---|---|
| B 场景打了**两次** `ai_msg.tool_calls`，`id` 还不一样（`call_00_sN7...` / `call_00_1RZ...`） | `llm.invoke` 被调用了 **2 次** → resume 时节点**真的从头重跑了** |
| C 场景 `st.values` 里**只有 1 条 HumanMessage** | `interrupt` 走异常通道，`ai_msg` 没来得及合并进 state |

**核心机制**：`interrupt()` 是「抛异常」，不是「return」。节点的返回值只在 **return 那一刻**才被 reducer 合并进 state。所以 `interrupt()` 之前的计算（本地变量）**全丢**，resume 时节点从第一行重新执行。

> 类比：节点是一个工人，干完活要把工单**交回柜台**（`return`）才算登记。`interrupt()` 是他在交单前突然举手喊停——手里的工单**没有登记**，柜台（state）上还是上一张（`HumanMessage`）。等他被叫回来，只能**从头重做**。

### 根因一图看清

```text
❌ 原写法（一个节点同时干两件事）
   agent: llm.invoke() → ai_msg(有tool_calls) → interrupt() ← 喊停，ai_msg 没交回
   state: [Human]                                            ← 最后一条是 Human，改不了

✅ 正确写法（拆成两个节点）
   decide :  llm.invoke() → return {"messages":[ai_msg]}     ← 先交回柜台
   state:   [Human, AI(带tool_calls)]                         ← 现在最后一条是 AI，可以改 ✅
   approve: 读 state 最后一条 → interrupt() → 问人
```

**再加一个隐患**：就算你 `update_state` 侥幸成功了，`edit` 分支 `return {"messages": [ai_msg]}` 返回的是**刚刚重新 invoke 出来的那条**（原始 `/tmp/a.txt`）——正好把改好的覆盖掉。所以**必须拆节点**，且 edit 分支不能返回新 invoke 的 msg。

### 修法：决策与审批拆成两个节点

```python
def decide(state: S) -> dict:
    """只管「想」。★必须 return，AIMessage 才有机会落进 state"""
    ai_msg = llm_decide.invoke(state["messages"])
    print(f"[decide] tool_calls={len(ai_msg.tool_calls)}")   # 计数用：看它跑了几次
    return {"messages": [ai_msg]}


def approve(state: S) -> dict:
    """只管「问人」。★这里不能放有副作用的代码（见第九节 ⑤）"""
    last = state["messages"][-1]
    assert isinstance(last, AIMessage) and last.tool_calls, \
        f"approve 必须接在 decide 之后，但最后一条是 {type(last).__name__}"

    tc = last.tool_calls[0]
    print(f"[approve] 从 state 读到的待审参数: {tc['args']}")

    answer = interrupt({
        "question": f"模型想删除 {tc['args']['path']}, 批准吗?",
        "options": ["approve", "reject", "edit"],
    })

    if answer == "reject":
        return {"messages": [AIMessage(content="已取消删除。")]}   # 无 tool_calls → END
    return {}          # approve / edit 都放行：state 里那条 AI 原样流向 tools
```

拆完之后的额外收益，是**省掉一半的 LLM 调用**：`[decide]` 每轮只打印一次 → resume 只重跑被中断的那个节点（`approve`），`decide` 的 checkpoint 已保存、不重跑。

* * *

<h2 id="三">三、`update_state` 改参：复制一份再改 + `id` 生死线</h2>

**一句话：工具调用还没发生时，它只是 state 里一条 AIMessage 上的 `tool_calls` 字段——改它就是改模型即将执行的动作。**

### 为什么必须「复制一份再改」

因为 checkpoint 里存的是**不可变快照**，直接改原消息对象会污染历史存档（你改了「过去」，回放就乱了）。正确姿势是造一个新对象替换它。

> 类比：不能拿笔涂改**已归档的档案原件**（会毁掉存档），要**复印一张、在复印件上改**、把复印件放回档案夹。

### 三行代码

```python
st = graph.get_state(cfg_c)                    # ① 只读偷看（不是 invoke！见第四节）
last = st.values["messages"][-1]               # ② 最后一条 = 带 tool_calls 的 AIMessage

new_tc = [{**tc, "args": {**tc["args"], "path": "/tmp/old.txt"}} for tc in last.tool_calls]
fixed = AIMessage(id=last.id, content=last.content, tool_calls=new_tc)   # ★ id 必须保留

graph.update_state(cfg_c, {"messages": [fixed]})   # ③ 外部改档
out_c = graph.invoke(Command(resume="edit"), cfg_c) # ④ 再恢复
```

三个容易错的点：

**① `{**tc["args"], "path": ...}` 而不是 `{"path": ...}`**
前者是在原参数字典上**覆盖一个字段**，后者是**整体替换**——如果工具还有 `force`、`recursive` 之类别的参数，整体替换会把它们全丢掉。

**② `id=last.id` 是生死线**
`add_messages` reducer 按 `id` 做 upsert：

- 带原 `id` → **原地替换**（旧消息被覆盖）
- 不传 `id` → 自动生成新 UUID → 变成**追加**，state 里会有两条 AI 消息，tools 大概率跑的是旧的那条（你的 edit 就白改了）

**③ 顺序铁律：先 `update_state` 改好 → 再 `Command(resume)`**
反了的话，resume 时节点重跑会用原始参数把你的改动覆盖掉。

* * *

<h2 id="四">四、`get_state` vs `invoke`：读状态别把图跑起来</h2>

**一句话：`graph.invoke(cfg)` 不是「读状态」，是「让图继续跑」。读状态请用 `graph.get_state(cfg)`。**

这一节单独拎出来，因为它坑得最隐蔽——**不报错，但悄悄把工具跑了**。

### 事故现场

```python
graph.invoke({"messages": [("user", "帮我删除 /tmp/a.txt")]}, cfg_c)   # 停在 interrupt
st = graph.invoke(cfg_c)          # ❌ 你以为在读状态，其实在恢复执行
last = st.values["messages"][-1]  # 拿到的是「跑完之后」的普通 AIMessage
```

于是日志里出现了这样一份 state：

```text
StateSnapshot(values={'messages': [
    HumanMessage(帮我删除 /tmp/a.txt),
    AIMessage(带 tool_calls ...),
    ToolMessage('文件已删除：/tmp/a.txt'),   # ← 工具已经执行过了！
    AIMessage(没有 tool_calls ...),          # ← 图已经跑完
]}, next=(), interrupts=())
```

三个信号一起说明「图已经不在审批点了」：

1. `ToolMessage` 已存在 → 工具**真跑过了**
2. 最后一条 `tool_calls=[]` → 走到终点了
3. `interrupts=()` → 没有挂起的中断

所以 `last.tool_calls` 为空 → `new_tc` 为空 → `update_state` 等于什么都没改 → 最终删的还是 `/tmp/a.txt`。

**为什么会这样？** 因为没传 `Command(resume=...)`，`interrupt()` 返回了 `None`，走到 `else`（edit）分支 `return {"messages": [ai_msg]}`，图就继续往下跑了。

### 对比表

| 调用 | 性质 | 会不会执行节点 | 用在哪 |
|---|---|---|---|
| `graph.get_state(cfg)` | 只读 | ❌ 不执行 | **偷看**状态、取待审批的 `tool_calls` |
| `graph.invoke(None, cfg)` | 恢复 | ✅ 从暂停处继续跑 | 真的要让图往下走 |
| `graph.invoke(Command(resume=...), cfg)` | 恢复 + 注入人的决定 | ✅ | 标准 HITL 恢复姿势 |
| `graph.invoke({"messages": [...]}, cfg)` | 新一轮输入 | ✅ | 开启新的一轮对话 |

⚠️ 还有个配套报错值得一提：`graph.invoke(cfg_c)` 把 config 当成了**输入数据**传给第一个位置参数，于是 checkpointer 找不到 `thread_id`：

```text
ValueError: Checkpointer requires one or more of the following
'configurable' keys: thread_id, checkpoint_ns, checkpoint_id
```

**config 永远是第二个位置参数**，记牢。

* * *

<h2 id="五">五、模型说「我误删了」：一次审计缺失事故</h2>

**一句话：模型没有「误删」——它在编一个合理的故事，因为 state 里没有任何「人改过参数」的记录。**

### 模型看到的证据链

改参跑通之后，模型下一轮总结时说了一句灾难性的话：**「抱歉，我调用工具时把路径写成了 `/tmp/old.txt`，删错了。」**

它为什么会这么想？看第 2 轮 `decide` 拿到的 messages：

```text
[0] Human : 帮我删除 /tmp/a.txt              ← 用户要的是 a.txt
[1] AI    : tool_calls{path: /tmp/old.txt}   ← 但记录里「我」调的是 old.txt
[2] Tool  : 文件已删除: /tmp/old.txt          ← 而且真删成功了
```

模型一比对：**用户要 a.txt，可「我」调了 old.txt，还删成了。** 按它唯一能想到的解释——「我手滑打错了路径」。

所以那句「我误删了」不是谎言，是**在缺信息条件下最合理的推断**。你的 `update_state` 是「上帝视角」的操作，模型不在场、没收到通知。

> 类比：车间主任（你）深夜偷偷改了工单，第二天工人看记录——**客户要 A 件，可记录上我做的是 B 件**——他只能认为「昨晚我打盹看错单子了」，然后郑重道歉。他不是糊涂，是**档案里没写「主任改过」**。

> **能复述给面试官的话**：LLM 不是知道真相，它只知道 state 里写了什么。**上下文里缺席的角色，在模型的叙事里就不存在**——所以任何人工干预都必须留痕，否则会被模型误读成自己的失误。

### 修法：加一个 `audit` 节点

问题根源是**审计缺失**，补法就是「把人的动作写进模型看得见的对话里」。

**① state 加一个普通字段（无 reducer，返回即覆盖）：**

```python
class S(TypedDict):
    messages: Annotated[list, add_messages]   # 有 reducer：只能追加/按 id 替换
    audit_note: str                           # 无 reducer：承载人工审批留痕
```

**② 改参的人负责留痕**（和改参在**同一次** `update_state` 里写下去，避免两次写入不同步）：

```python
graph.update_state(cfg_c, {
    "messages": [fixed],
    "audit_note": "审批人把删除目标从 /tmp/a.txt 改成了 /tmp/old.txt",
})
```

**③ `audit` 节点把它转成模型看得见的一条消息：**

```python
def audit(state: S) -> dict:
    note = state.get("audit_note", "")
    if not note:
        return {}
    return {
        "messages": [HumanMessage(content=(
            f"【审批人操作记录】{note}。"
            "这是审批人主动、有意做出的决定，已获批准执行；"
            "这不是错误，无需道歉、无需重试其他路径，请直接据实向用户总结结果。"
        ))],
        "audit_note": "",      # 用完清空，避免污染后续轮次
    }
```

**④ 改边：** `builder.add_edge("tools", "audit")` → `builder.add_edge("audit", "summarize")`

### 两个顺序要点

- **为什么用 `HumanMessage` 而不是 `SystemMessage`**：语义上它就是「审批人对你说的话」；工程上，`SystemMessage` 插在 messages 中段**部分 API 会拒绝或忽略**。
- **`audit` 必须放在 `tools` 之后**：如果在 `AI(tool_calls)` 和 `ToolMessage` 之间插消息，会破坏「tool 消息必须紧跟在带 `tool_calls` 的 assistant 消息后」这条约束，API 可能直接报错。

⚠️ **已知取舍**：这里用 `HumanMessage`「冒充」人在说话，在 API 眼里和真人发言无法区分。单机 demo 可以；多用户产品要换成可区分的通道（自定义消息类型 / 独立字段）。

* * *

<h2 id="六">六、「看起来跑完了」≠ 跑完了：`next` 判据 + 结构收口</h2>

**一句话：任何「看起来像最终回复」的文本，都不能替代 `state.next` 这个判据——`next == ()` 才算真跑完。**

### 铁证：`next = ('approve',)`

加了 audit 之前，打印了一份「C 最终」看起来挺正常。但补两行实测：

```python
st_c = graph.get_state(cfg_c)
print("C 跑完后 next =", st_c.next)
# C 跑完后 next = ('approve',)
print("最后一条 tool_calls:", getattr(st_c.values["messages"][-1], "tool_calls", None))
# 最后一条 tool_calls: [{'path': '/tmp/a.txt'}]
```

**场景 C 根本没有收敛**——图正停在第二次审批上等你点头，而模型手里攥着的是一张「重新删除 a.txt」的新工单。

| # | 问题 | 面向谁 | 严重度 |
|---|---|---|---|
| 1 | 模型对用户说「我误删了文件」 | **用户** | 🔴 高——对客户说「我删错了」是灾难性表述 |
| 2 | 模型自纠、想再删一次 a.txt | **系统** | 🟡 中——好消息：第二次仍被 `approve` 拦住，**没有绕过审批**；坏消息：空转、多烧一次 LLM 调用 |

第 2 点要特别表扬这个结构：因为 `approve` 节点**每次**都 interrupt，所以「模型自纠」撞上的是一道必须人点头的门——**没有越权执行**。安全边界是对的，坏的只是叙事和效率。

> 类比：保安没放错人（权限正确），但前台给客户打了电话说「我们同事刚才砸错了你的车」（叙事错误）。两件事，要用两把不同的钥匙修。

### 两条独立的路，各治一个病

| 路 | 治什么 | 手段 | 强度 |
|---|---|---|---|
| **A · 审计留痕** | 模型的错误叙事 | 人的审批动作写成一条消息，让模型「看得见」 | 提示词级（软） |
| **B · 结构收口** | 模型反复重试 | **总结用不绑工具的 LLM** → 物理上产不出 `tool_calls` | 结构级（硬） |

**路 B 是 D4 真正值钱的一招**：

```python
llm_decide    = ChatOpenAI(**_cfg).bind_tools([delete_file])   # 绑工具 → 决策用
llm_summarize = ChatOpenAI(**_cfg)                             # ★不绑工具★ → 收口用
```

工具跑完 → 进 `summarize` 节点 → 这个 LLM **根本没绑工具** → 它在物理上**无法**生成 `tool_calls` → 图必然进 END。**不是「劝模型不要重试」，是让重试这条路在代码里不存在。**

> 类比：A 是在车间贴告示「请勿重复报修」；B 是把重复报修的**按钮拆了**。这就是 W5-D6 那句「赌恶不赌善」——**别靠提示词祈祷，靠结构装防火墙**。

**魔鬼代言人**：别以为「不绑工具的总结节点」是万灵药——它只保证**这一步**不产工具调用，不保证模型**不说错话**；而且如果业务需要「总结后继续下一轮决策」，这个 END 就是硬截断。真正的审计要靠日志 / 数据库，不是靠往 messages 里塞一句话——那一层留到 D6 做可观测性时再补。

* * *

<h2 id="七">七、完整可运行代码 + 8 项断言</h2>

把 A（放行）/ B（拒绝）/ C（改参）合到一个文件、一次跑完，末尾带**断言核验**——跑完你会看到 `✅ D4 整合版 8 项断言全绿`，而不是靠眼睛数 print。

**文件**：`W6/lg_d4_abc_full.py`　**运行**：`python lg_d4_abc_full.py`

```python
"""W6-D4 整合版: 审批三态 A/B/C 一次跑完
    A = 放行(执行原参数)   B = 拒绝(工具不执行)   C = 改参(update_state 改写 pending 参数 + 审计留痕)
"""

import os
import io
import contextlib
from typing import Annotated, TypedDict

from dotenv import load_dotenv
load_dotenv(override=True)

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver
from langgraph.prebuilt import ToolNode
from langgraph.types import interrupt, Command
from langchain_core.tools import tool
from langchain_core.messages import AIMessage, HumanMessage
from langchain_openai import ChatOpenAI


# ═══════════════════ 1. 工具 + 状态 ═══════════════════
@tool
def delete_file(path: str) -> str:
    """【危险】删除本地文件(演示版, 不真删)"""
    # ★ 这行 print 是"验真身"用的: 工具到底跑没跑、跑的是哪个路径, 全看它。
    #   断言也靠捕获这行来判定 —— 不要删。
    print(f">>> 工具真身执行: 删除 {path}")
    return f"文件已删除: {path}"


class S(TypedDict):
    messages: Annotated[list, add_messages]   # 有 reducer: 只能追加/按 id 替换
    audit_note: str        # ★ 无 reducer 的普通字段, 用来承载"人工审批留痕"


# ════════ 2. 两个 LLM: 决策的绑工具, 收口的不绑 ════════
_cfg = dict(model="deepseek-v4-flash",
            api_key=os.getenv("DEEPSEEK_API_KEY"),
            base_url="https://api.deepseek.com",
            temperature=0)

llm_decide = ChatOpenAI(**_cfg).bind_tools([delete_file])

# ★ 关键设计: 收口的 LLM 【不绑工具】→ 物理上产不出 tool_calls → 图必然进 END。
#   这不是"劝模型别重试", 是让重试这条路在代码里不存在(赌恶不赌善)。
llm_summarize = ChatOpenAI(**_cfg)


# ═══════════════════ 3. 五个节点 ═══════════════════
def decide(state: S) -> dict:
    """只干一件事: 让模型决策。★必须 return, AIMessage 才有机会落进 state"""
    ai_msg = llm_decide.invoke(state["messages"])
    print(f"[decide] tool_calls={len(ai_msg.tool_calls)}")
    return {"messages": [ai_msg]}
    # ⚠️ 上一版把 llm.invoke 和 interrupt 塞在同一个节点里 → interrupt 走异常通道,
    #    ai_msg 没机会 return → state 最后一条还是 HumanMessage → 改参时 AttributeError。
    #    拆成 decide / approve 两个节点就是为了修这个。


def approve(state: S) -> dict:
    """只干一件事: 问人。★这里【不能】放有副作用的代码"""
    last = state["messages"][-1]
    if not (isinstance(last, AIMessage) and last.tool_calls):
        return {}                              # 没有 tool_calls, 没什么可审的

    tc = last.tool_calls[0]
    print(f"[approve] 从 state 读到的待审参数: {tc['args']}")

    # ★ interrupt 之前的代码在 resume 时会【重放】, 所以本函数会打印两次。
    #   结论: interrupt 之前只能放幂等逻辑(读 state / print)。
    #   把"发邮件""扣款"放这里 = resume 时重复执行。
    answer = interrupt({
        "question": f"模型想删除 {tc['args']['path']}, 批准吗?",
        "options": ["approve", "reject", "edit"],
    })

    if answer == "reject":
        # ★ 返回不带 tool_calls 的 AIMessage → 路由判定为 END → 工具一次都不执行
        return {"messages": [AIMessage(content="已取消删除，本次未执行任何操作。")]}

    # approve / edit 都放行: state 里那条 AIMessage 原样流向 tools。
    # edit 时它的参数已被外部 update_state 改写过了。
    return {}


def audit(state: S) -> dict:
    """★治叙事★ 把"人改过参数"这件事, 写成模型看得见的一条消息"""
    note = state.get("audit_note", "")
    if not note:
        return {}
    print(f"[audit] 注入留痕: {note}")
    return {
        "messages": [HumanMessage(content=(
            f"【审批人操作记录】{note}。"
            "这是审批人主动、有意做出的决定，已获批准执行；"
            "这不是错误，无需道歉、无需重试其他路径，请直接据实向用户总结结果。"
        ))],
        "audit_note": "",          # 用完清空, 避免污染后续轮次
    }
    # ⚠️ 已知取舍: 这里用 HumanMessage "冒充"人在说话, 在 API 眼里和真人发言无法区分。
    #    单机 demo 可以; 多用户产品要换成可区分的通道(自定义消息类型/独立字段)。


def summarize(state: S) -> dict:
    """★治重试★ 不绑工具的 LLM 收口 → 只能说话, 不可能再调工具"""
    ai_msg = llm_summarize.invoke(state["messages"])
    print("[summarize] 收口完成, tool_calls=0 (结构保证, 非提示词保证)")
    return {"messages": [ai_msg]}


# ═══════════════════ 4. 路由 + 构图 ═══════════════════
def route_after_decide(state: S):
    last = state["messages"][-1]
    return "approve" if (isinstance(last, AIMessage) and last.tool_calls) else END


def route_after_approve(state: S):
    last = state["messages"][-1]
    return "tools" if (isinstance(last, AIMessage) and last.tool_calls) else END


builder = StateGraph(S)
builder.add_node("decide", decide)
builder.add_node("approve", approve)
builder.add_node("tools", ToolNode([delete_file]))
builder.add_node("audit", audit)
builder.add_node("summarize", summarize)

builder.add_edge(START, "decide")
builder.add_conditional_edges("decide",  route_after_decide,  {"approve": "approve", END: END})
builder.add_conditional_edges("approve", route_after_approve, {"tools": "tools",     END: END})
builder.add_edge("tools", "audit")        # 工具跑完 → 先留痕(治叙事)
builder.add_edge("audit", "summarize")    # 再收口(不绑工具) → 必然 END
builder.add_edge("summarize", END)

# 一张图 + 一个 checkpointer; 会话隔离靠 invoke 时的 thread_id
graph = builder.compile(checkpointer=MemorySaver())

# 拓扑(先看地图再看代码):
#   START → decide ─(有tool_calls)→ approve ─(放行)→ tools → audit → summarize → END
#                  └(无)──────────→ END      └(拒绝)→ END


# ═══════════════ 5. 三个场景 ═══════════════════
def run_a():
    """A 放行: 执行模型原本给的参数 /tmp/a.txt"""
    print("\n" + "=" * 18 + " A 放行 " + "=" * 18)
    cfg = {"configurable": {"thread_id": "state-a"}}
    graph.invoke({"messages": [("user", "帮我删除 /tmp/a.txt")]}, cfg)  # 停在 approve 的 interrupt
    out = graph.invoke(Command(resume="approve"), cfg)                  # 放行 → tools 执行
    print("A 最终:", out["messages"][-1].content)
    return out, graph.get_state(cfg)


def run_b():
    """B 拒绝: 取消, 工具一次都不执行"""
    print("\n" + "=" * 18 + " B 拒绝 " + "=" * 18)
    cfg = {"configurable": {"thread_id": "state-b"}}
    graph.invoke({"messages": [("user", "帮我删除 /tmp/b.txt")]}, cfg)
    out = graph.invoke(Command(resume="reject"), cfg)                   # 拒绝 → END
    print("B 最终:", out["messages"][-1].content)
    return out, graph.get_state(cfg)


def run_c():
    """C 改参: 人在图外把 pending 参数从 a.txt 改成 old.txt, 再放行"""
    print("\n" + "=" * 18 + " C 改参 " + "=" * 18)
    cfg = {"configurable": {"thread_id": "state-c"}}
    graph.invoke({"messages": [("user", "帮我删除 /tmp/a.txt")]}, cfg)  # 停在 approve

    # ── ① 读档, 确认最后一条是 AIMessage(这是能改参的前提) ──
    st = graph.get_state(cfg)
    last = st.values["messages"][-1]
    print("最后一条类型:", type(last).__name__)          # 期望 AIMessage
    assert isinstance(last, AIMessage) and last.tool_calls, \
        "最后一条不是带 tool_calls 的 AIMessage → 改参无从下手"

    # ── ② 复制一份再改(不能直接改原对象: checkpoint 是不可变快照) ──
    new_tc = [{**tc, "args": {**tc["args"], "path": "/tmp/old.txt"}} for tc in last.tool_calls]

    # ★ id=last.id 是生死线: add_messages 按 id 做 upsert
    #   带 id → 原地替换; 不带 → 自动生成新 UUID → 变成"追加", 会有两条 AI 消息
    fixed = AIMessage(id=last.id, content=last.content, tool_calls=new_tc)

    # ── ③ 改参 + 留痕 在同一次 update_state 里写下去(避免两次写入不同步) ──
    graph.update_state(cfg, {
        "messages": [fixed],
        "audit_note": "审批人把删除目标从 /tmp/a.txt 改成了 /tmp/old.txt",
    })
    print("已把删除目标改为 /tmp/old.txt")

    # ★ 顺序铁律: 先 update_state 改好, 再 Command(resume) 恢复。反了改参会被覆盖。
    out = graph.invoke(Command(resume="edit"), cfg)
    print("C 最终:", out["messages"][-1].content)
    return out, graph.get_state(cfg)


# ═══════════════ 6. 断言核验(把"我看过"变成"可重复验证") ═══════════════
def check(txt, out_a, st_a, out_b, st_b, out_c, st_c):
    print("\n" + "=" * 16 + " 断言核验 " + "=" * 16)

    # ── A: 放行 → 原参数被执行, 且图真收敛 ──
    assert ">>> 工具真身执行: 删除 /tmp/a.txt" in txt, "A: 原参数 a.txt 没被执行"
    assert st_a.next == (), f"A: 图没收敛, 还停在 {st_a.next}"

    # ── B: 拒绝 → 走拒绝分支, 且工具一次都没执行 ──
    assert "已取消" in out_b["messages"][-1].content, "B: 没走拒绝分支"
    assert ">>> 工具真身执行: 删除 /tmp/b.txt" not in txt, "B: 工具竟然执行了"
    assert st_b.next == (), f"B: 图没收敛, 还停在 {st_b.next}"

    # ── C: 改参 → 执行的是 old.txt, 且审计留痕生效 ──
    assert ">>> 工具真身执行: 删除 /tmp/old.txt" in txt, "C: 改后的 old.txt 没被执行"
    assert "[audit] 注入留痕" in txt,        "C: 审计留痕没注入 → 模型可能又编'我误删了'"
    assert "[summarize] 收口完成" in txt,    "C: 收口节点没跑到"
    assert st_c.next == (), f"C: 图没收敛, 还停在 {st_c.next}"
    assert getattr(st_c.values["messages"][-1], "tool_calls", []) == [], \
        "C: 最后一条不该带 tool_calls"

    # ★ "重试被掐死"的判据: A/B/C 各进一次 decide, 共 3 次。若变 4 次 = 又空转重试了
    n_decide = txt.count("[decide]")
    assert n_decide == 3, f"decide 应跑 3 次(每场景 1 次), 实际 {n_decide} 次 → 有重试"

    # ── 软检查: 模型的话术会飘, 不该让脚本崩, 但要提醒你回看 ──
    if "审批" not in out_c["messages"][-1].content:
        print("⚠️ 软检查未过: C 的总结没提到'审批', 叙事可能又飘了 —— 回看上面 C 最终")

    print("✅ D4 整合版 8 项断言全绿(A 2 项 / B 3 项 / C 4 项)")


def main():
    # 把三个场景的终端输出全部捕获下来 → 断言靠它判定, 不靠眼睛数
    log = io.StringIO()
    with contextlib.redirect_stdout(log):
        out_a, st_a = run_a()
        out_b, st_b = run_b()
        out_c, st_c = run_c()

    txt = log.getvalue()
    print(txt)                    # 原始日志照打, 方便你回看模型说了什么
    check(txt, out_a, st_a, out_b, st_b, out_c, st_c)


if __name__ == "__main__":
    main()
```

### 跑完要核对的五个数（数字，不是感觉）

| 场景 | 硬证据 | 期望 | 证明了什么 |
|---|---|---|---|
| A | `>>> 工具真身执行: 删除` 的路径 | `/tmp/a.txt` | 放行分支走通、原参数被执行 |
| B | B 段里有没有工具 print | **一次都没有** | 拒绝分支真的拦住了工具 |
| C | 工具 print 的路径 | **`/tmp/old.txt`** | `update_state` 真生效（不是 a.txt 才算） |
| 全部 | `[decide]` 总次数 | **3** | 重试被结构掐死（变 4 就是又空转了） |
| 全部 | `next` | 全为 `()` | 图真收敛，不是「看起来跑完」 |

⚠️ **两个已知的「软失败」**（不是你的 bug，别慌）：

1. 模型偶尔**不肯调工具**（说「我无法删除文件」）→ A 的断言会红。这是模型行为飘，重跑即可；要稳定就把 `temperature` 保持 0 并放松措辞。
2. 若 `[audit]` 没打印，说明 `audit_note` 没跨过 interrupt/resume 活下来（版本差异）。备用方案：把改参消息塞进 `approve` 的 edit 分支直接 return，**不走 `update_state`**：

```python
if answer.get("decision") == "edit":
    new_tc = [{**t, "args": {**t["args"], "path": answer["new_path"]}} for t in last.tool_calls]
    return {"messages": [AIMessage(id=last.id, content=last.content, tool_calls=new_tc)]}
# 调用：graph.invoke(Command(resume={"decision": "edit", "new_path": "/tmp/old.txt"}), cfg)
```

**为什么这个写法更稳**：改动在节点内完成、随 return 一起落盘，不涉及「在 interrupt 挂起时改档」这个版本敏感操作。代价是要自定义 resume 值结构。

* * *

<h2 id="八">八、时间旅行：`checkpoint_id` = 存档点</h2>

**一句话：每个节点执行后自动存一个 checkpoint，`config` 里同时带 `thread_id` + `checkpoint_id` 就是「读档」。**

用**确定性线性图**（不用 LLM）验证机制最干净：

```python
"""W6-D4 实验2: 时间旅行 —— checkpoint_id = 存档点, 改 config 读档"""
from typing import Annotated, TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.checkpoint.memory import MemorySaver


class T(TypedDict):
    messages: Annotated[list, add_messages]


def step_a(state: T) -> dict: print("→ A 执行");  return {"messages": [("ai", "A 完成")]}
def step_b(state: T) -> dict: print("→ B 执行");  return {"messages": [("ai", "B 完成")]}
def step_c(state: T) -> dict: print("→ C 执行");  return {"messages": [("ai", "C 完成")]}


builder = StateGraph(T)
builder.add_node("a", step_a); builder.add_node("b", step_b); builder.add_node("c", step_c)
builder.add_edge(START, "a"); builder.add_edge("a", "b"); builder.add_edge("b", "c"); builder.add_edge("c", END)
graph = builder.compile(checkpointer=MemorySaver())

cfg = {"configurable": {"thread_id": "tt-1"}}
graph.invoke({"messages": [("user", "完整跑一遍 A B C")]}, cfg)
print("完整跑完, 当前消息数:", len(graph.get_state(cfg).values["messages"]))

# 看时间线(新→旧)
print("\n=== 时间线(新→旧) ===")
snaps = list(graph.get_state_history(cfg))
for i, s in enumerate(snaps):
    cid = s.config["configurable"]["checkpoint_id"][:8]
    print(f"#{i} checkpoint={cid}  next={s.next}  消息数={len(s.values['messages'])}")

# 挑一个 A 刚跑完、B 还没跑的存档(next 含 'b')→ 读档分支重跑
target = next(s for s in snaps if "b" in s.next)
past_cid = target.config["configurable"]["checkpoint_id"]
print(f"\n回到 A 之后 B 之前 (checkpoint={past_cid[:8]})")

fork_cfg = {"configurable": {"thread_id": "tt-1", "checkpoint_id": past_cid}}
graph.invoke({"messages": [("user", "从这重来, B C 重跑一遍")]}, fork_cfg)
print("\n重跑后消息数:", len(graph.get_state(cfg).values["messages"]))
```

> 类比：游戏读档。`checkpoint_id` = 存档槽里的某一格。当前进度 = 最新存档；想试「如果当时选另一条路」，就读回那个存档点，新操作会覆盖旧未来、长出新分支。

**判据（两个证据缺一个都算验证缺口）**：

1. fork 后终端**只**打印 `→ B 执行`、`→ C 执行`（**A 不出现**）——证明没有从头跑
2. 消息数 +2——数量也对得上

**口述版**：`get_state_history` 拿到历史存档点（**新→旧**排列），config 里带上 `checkpoint_id` 就是读档，从那里重新 invoke 会分支出新未来——调试「如果当时走了另一条路」不用删库重跑。

* * *

<h2 id="九">九、踩坑记 & 知识点</h2>

**① `interrupt({...})` 里的 `question` / `options` 是自定义键名，不是关键字参数**
你传给 `interrupt` 的是一个**字典字面量**，键名完全由你决定。写成 `interrupt({"提示信息": ..., "可选操作": [...]})` 效果完全一样，传一个纯字符串也行。
区别在于：**关键字参数**的键名是函数形参名，写错报 `TypeError: unexpected keyword argument`；**字典的键名**是数据字段名，写什么都行，LangGraph 只负责把它透传给前端。真正决定分支的是 `Command(resume=...)` 的值。

**② 同一节点里出现 3 次 `return {"messages": [ai_msg]}` 不是啰嗦，是三个出口**
`approve` 分支放行、`edit` 分支放行（参数已被外部改过）、以及最外层兜底（模型压根不想调危险工具）——三条不同路径的「放行」。只有 `reject` 返回的是**不带 `tool_calls`** 的普通消息，路由才会走 END。

**③ `graph.invoke(cfg)` 报 `Checkpointer requires ... thread_id` → config 传错位置了**
config 是**第二个位置参数**。要恢复执行就写 `graph.invoke(None, cfg)` 或 `graph.invoke(Command(resume=...), cfg)`。但更推荐 `get_state(cfg)`——**读状态本来就不该触发执行**。

**④ `update_state` 时 `id` 没沿用 → 变追加而不是替换**
`add_messages` 按 `id` 做 upsert。不带 `id` 会生成新 UUID，state 里同时存在新旧两条 AI 消息，tools 可能跑旧的那条。写法：`AIMessage(id=last.id, content=..., tool_calls=new_tc)`。

**⑤ `interrupt()` 之前的代码在 resume 时会重放 → 只能放幂等逻辑**
证据：`[approve]` 打印了两次。这是预期行为，不是 bug——resume 时节点从函数头部重放到 `interrupt()`（interrupt 的返回值被缓存，不会再问一次）。
但**重放时读的是最新 state，不是当时的旧值**（第二次打印读到了 `old.txt`）。所以：**如果那行不是 `print` 而是「发邮件」/「扣款」，你每次 resume 都会重发一次。**

**⑥ `audit_note` 用普通字段（无 reducer）**
有 reducer（`Annotated[list, add_messages]`）的字段只能追加/按 id 替换；像 `audit_note` 这种「用完即弃」的标记，用裸 `str` 类型，返回即覆盖，节点里再 `""` 清空避免污染后续轮次。

**⑦ `next` 非空 = 图还活着、还在等输入**
`next=('approve',)` 说明图停在审批点；`next == ()` 才是真跑完。**任何「看起来像最终回复」的文本，都不能替代 `next` 这个判据。**

**⑧ 待验证假设：`audit_note` 能否跨 interrupt/resume 存活**
不同 langgraph 版本在「interrupt 挂起期间调 `update_state`」这件事上行为有差异。判据就是 C 段有没有打印 `[audit] 注入留痕`。没打印就换第九节末尾那个「节点内 return」的备用法，并在笔记里记一行「版本差异」。

* * *

<h2 id="十">十、总结</h2>

今天把 D3 的「两态门卫」升级成了「三态审批员」，靠的是五个节点各司其职——**而且 D3 的 interrupt 机制一行没改，只是换了组织方式**：

- **`decide` / `approve` 拆开** = 治报错：**状态必须存在，才谈得上被审批**。`interrupt()` 是抛异常，没 return 的东西进不了 state
- **`update_state` 改参** = 涂改笔：复制一份再改、`id` 必须沿用、先改后 resume
- **`audit` 节点** = 登记簿：**审计留痕不只是给人看的，更是给模型看的**——上下文里缺席的角色，在模型的叙事里就不存在
- **`summarize` 节点** = 拆按钮：**决策的 LLM 绑工具，收口的 LLM 不绑工具**，用能力边界而不是提示词来保证终止（赌恶不赌善）
- **时间旅行** = 时光机：`checkpoint_id` = 存档点，改 config 就是读档

整张图：`START → decide → approve → tools → audit → summarize → END`——审批从此成为「执行前的一道可改单的闸」，而不是执行后的事后追认。

> 一句话收尾：**留痕治叙事，不绑工具治重试，两者都不能替代「审批动作本身是否被正确记录」**——真正的审计要靠日志 / 数据库，不是靠往 messages 里塞一句话。

* * *

<h2 id="十一">十一、下一篇预告</h2>

**D5/D6：把审批装进真实项目（写记忆前问人确认）**

今天这套「三态 + 留痕 + 收口」是**零件**，D6 的项目要把它装到真实的业务里——比如 Agent 想往长期记忆写一条用户画像时，先 `interrupt` 问一句「这条要存吗 / 存成这样对吗」。

那会儿你会遇到今天没解决的问题：

- **多用户场景**下，`HumanMessage` 冒充审批人就不再安全了，得换成可区分的通道
- **审计留痕**要从「塞进 messages」升级成**独立的日志 / 数据库表**，并做可观测性
- **时间旅行**会从「演示」变成刚需——真实事故要靠它回档重跑

再往后，**W7 的 Tool Registry / Permission** 要解决今天这层防不住的事：模型被别处的提示注入带跑。今天的审批是「一道门」，W7 是「门的权限系统」。

> 一句话：**D1-D4 你学会了「记忆怎么存、审批怎么拦、参数怎么改、历史怎么回档」；D5/D6 轮到「这些零件怎么拼成一个能交付的项目」。**
