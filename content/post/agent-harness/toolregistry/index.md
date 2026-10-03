---
description: ""
title: "W7-D4｜Tool Registry + Permission Gate：给 Agent 加上工具注册表和权限闸门"
draft: false
date: "2026-09-17T13:54:20+08:00"
slug: "ToolRegistry"
categories:
 - Harness
tags:
 - Tool Registry
 - Permission Gate
image: ""
---

# W7-D4｜Tool Registry + Permission Gate：给 Agent 加上工具注册表和权限闸门

> 今天第一次接触 Harness。
>
> 之前学 LangGraph 时，我一直在解决“Agent **怎么跑**”的问题：
>
> * State 怎么流
> * 节点怎么走
> * 条件边怎么决定下一步
> * HITL 怎么暂停和恢复
>
> 今天开始换一个问题：
>
> > **Agent 想调用工具的时候，谁来管理这些工具？谁决定这个工具能不能真的执行？**
>
> 今天做的就是两层：
>
> **Tool Registry：工具登记处**
>
> **Permission Gate：工具闸门**

---

# 一、先看整体：今天到底给 Agent 加了什么

以前可能是这种结构：

```text
Agent
  │
  ├── read_file()
  ├── search_web()
  ├── delete_file()
  └── send_email()
```

工具多起来以后，工具散落在代码各处。

Agent 知道某个函数存在，就直接调用：

```text
Agent
  ↓
直接调用工具
  ↓
执行
```

问题就来了：

* 工具在哪里？
* 工具有哪些？
* 工具的名字是什么？
* 参数怎么传？
* 这个工具危险不危险？
* 危险工具能不能直接执行？
* 谁审批过？

所以今天加一层 Harness：

```text
                    ┌───────────────┐
                    │     Agent     │
                    └───────┬───────┘
                            │
                     想调用某个工具
                            ↓
                  ┌──────────────────┐
                  │  Tool Registry   │
                  │   工具注册表      │
                  └────────┬─────────┘
                           │
                     找到工具 + 元数据
                           ↓
                  ┌──────────────────┐
                  │ Permission Gate  │
                  │     权限闸门      │
                  └────────┬─────────┘
                           │
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
           allow         deny      need_approval
             │                           │
             ↓                           ↓
          执行工具                  等待人工审批
```

所以今天新增的不是一种“新的 Agent”。

而是在：

> **Agent 和真正执行工具之间，加了一层统一管理和控制。**

---

# 二、今天解决的其实是两个不同问题

这两个东西很容易混在一起。

## 1. Registry 解决“这个工具是什么”

Registry 更像工具目录。

它负责：

```text
名字
 ↓
找到工具
 ↓
拿到工具信息
```

例如：

```text
delete_file
    ↓
找到真正的 delete_file 函数
    ↓
知道它是 high risk
```

所以 Registry 的核心问题是：

> **“我有哪些工具，它们分别是什么？”**

---

## 2. Permission Gate 解决“这个工具现在能不能执行”

Gate 不负责寻找工具。

它只判断：

```text
这个工具
   ↓
当前条件
   ↓
允许吗？
```

例如：

```text
read_file
    ↓
low risk
    ↓
allow
```

而：

```text
delete_file
    ↓
high risk
    ↓
need_approval
```

所以可以简单记成：

| 模块              | 解决的问题     |
| --------------- | --------- |
| Tool Registry   | 工具是谁      |
| Permission Gate | 工具现在能不能执行 |

这两个问题一定要分开。

---

# 三、先看 Tool Registry 的核心结构

先不管装饰器。

假设最简单的工具注册表是：

```python
class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, name, description, risk, func):
        self._tools[name] = {
            "name": name,
            "description": description,
            "risk": risk,
            "func": func,
        }

    def get(self, name):
        return self._tools[name]
```

这里其实没有什么神秘东西。

本质就是一个字典：

```text
工具名
  ↓
工具信息
```

例如：

```python
{
    "delete_file": {
        "name": "delete_file",
        "description": "删除文件",
        "risk": "high",
        "func": delete_file
    }
}
```

这样 Agent 以后就不需要到处找函数。

只需要：

```python
registry.get("delete_file")
```

就能拿到整个工具描述。

---

# 四、为什么工具不直接放函数，还要包一层 ToolSpec？

实际代码里通常不会一直使用这种裸字典。

更适合定义一个工具描述对象：

```python
from dataclasses import dataclass
from typing import Callable


@dataclass
class ToolSpec:
    name: str
    description: str
    risk: str
    func: Callable
```

然后 Registry：

```python
class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, spec: ToolSpec):
        self._tools[spec.name] = spec

    def get(self, name: str) -> ToolSpec:
        return self._tools[name]
```

现在注册表里面保存的就不是一堆散乱字段，而是完整的：

```text
ToolSpec
├── name
├── description
├── risk
└── func
```

这其实是在做一件非常工程化的事情：

> **把“一个工具”和“这个工具的描述信息”绑在一起。**

以后权限判断、日志、工具发现、参数校验都可以继续往 `ToolSpec` 上挂。

---

# 五、然后问题来了：工具一个个手动 register 很麻烦

如果每个工具都这样写：

```python
def read_file(path):
    ...


registry.register(
    ToolSpec(
        name="read_file",
        description="读取文件",
        risk="low",
        func=read_file,
    )
)
```

工具少的时候还能接受。

工具多以后就很烦。

而且：

```python
def read_file(...):
    ...

registry.register(...)
```

工具代码和“注册动作”是分开的。

能不能写成：

```python
@registry.register(...)
def read_file(...):
    ...
```

这就进入今天第一次真正遇到的东西：

# 装饰器

---

# 六、装饰器到底是什么

我以前看到：

```python
@xxx
def foo():
    ...
```

容易把它理解成：

> “这是某种特殊语法。”

实际上没有那么神秘。

装饰器最核心的思想只有一句：

> **把一个函数作为参数传进去，返回一个处理后的函数对象。**

最简单的例子：

```python
def decorator(func):
    def wrapper():
        print("调用前")
        func()
        print("调用后")

    return wrapper
```

然后：

```python
@decorator
def hello():
    print("hello")
```

它其实等价于：

```python
def hello():
    print("hello")

hello = decorator(hello)
```

这就是装饰器最关键的理解点。

---

# 七、所以 `@decorator` 到底做了什么？

假设：

```python
@decorator
def hello():
    print("hello")
```

Python 不会把 `@decorator` 当成一个神秘标签。

它实际上是在做：

```text
先创建 hello 函数
        ↓
把 hello 作为参数传给 decorator
        ↓
decorator 返回一个函数
        ↓
再把返回结果重新绑定给 hello
```

也就是：

```python
hello = decorator(hello)
```

所以：

> **装饰器的本质，就是“函数加工”。**

---

# 八、为什么今天的 Registry 特别适合用装饰器

回到今天。

我们希望：

```python
@registry.register(...)
def read_file(path):
    ...
```

意思其实非常直白：

> “定义这个函数的时候，顺手把它登记进 Registry。”

也就是说：

```text
定义工具
  +
注册工具
```

合并成了一件事。

这不是为了炫技。

它解决的是：

> **避免“工具定义”和“工具注册”分离。**

---

# 九、这里有一个比普通装饰器更重要的细节

你今天很可能会看到：

```python
@registry.register(
    name="read_file",
    description="读取文件",
    risk="low",
)
def read_file(path):
    ...
```

你可能会疑惑：

> `register()` 明明需要参数，为什么最后还能接收到 `read_file`？

这里是装饰器最容易卡住的地方。

因为这里其实有两层调用。

---

## 第一层：先执行 `register(...)`

```python
registry.register(
    name="read_file",
    description="读取文件",
    risk="low",
)
```

这一步还没有拿到 `read_file`。

它的任务只是：

> 根据这些配置，创建一个真正的装饰器。

可以理解成：

```text
register(...)
    ↓
得到 decorator
```

---

## 第二层：Python 再把函数交给这个 decorator

然后定义：

```python
def read_file(path):
    ...
```

Python 再做：

```text
decorator(read_file)
```

也就是说整个过程实际上类似：

```python
decorator = registry.register(
    name="read_file",
    description="读取文件",
    risk="low",
)

read_file = decorator(read_file)
```

这才是：

```python
@registry.register(...)
def read_file(...):
    ...
```

真正发生的事情。

---

# 十、所以 Tool Registry 的装饰器可以这样写

```python
from dataclasses import dataclass
from typing import Callable


@dataclass
class ToolSpec:
    name: str
    description: str
    risk: str
    func: Callable


class ToolRegistry:
    def __init__(self):
        self._tools = {}

    def register(self, name, description, risk):
        def decorator(func):
            spec = ToolSpec(
                name=name,
                description=description,
                risk=risk,
                func=func,
            )

            self._tools[name] = spec

            return func

        return decorator
```

这里的结构值得认真看。

```text
register(...)
    ↓
返回 decorator
    ↓
decorator(func)
    ↓
创建 ToolSpec
    ↓
放进 self._tools
    ↓
返回原函数
```

这里出现了一个新的知识：

> **函数里面还定义函数。**

这叫嵌套函数。

而 `decorator()` 能记住外面的：

```python
name
description
risk
```

是因为它形成了闭包。

你今天不需要深入学闭包底层，但至少先知道：

> `decorator()` 虽然执行时接收到的参数只有 `func`，但它还能使用外层 `register()` 的 `name / description / risk`。

这是装饰器里非常常见的写法。

---

# 十一、为什么最后要 `return func`？

这是今天一个非常值得注意的细节。

```python
def decorator(func):
    ...
    return func
```

为什么注册完以后不返回 `None`？

因为：

```python
@registry.register(...)
def read_file(...):
    ...
```

最终等价于：

```python
read_file = decorator(read_file)
```

如果：

```python
decorator(...)
```

返回 `None`：

```python
read_file = None
```

那原本的：

```python
read_file(...)
```

以后就不能调用了。

所以这里选择：

```python
return func
```

意味着：

> **我完成了登记，但不改变这个函数本身。**

因此：

```text
Registry
    ├── 保存了一份工具信息
    └── 原函数依然还是原函数
```

这是今天这个装饰器实现和很多“包装型装饰器”很重要的区别。

---

# 十二、这和 LangChain 里的 `@tool` 不完全是一回事

这里很容易产生一个误解：

> “装饰器不是就是 `@tool` 吗？”

不是。

`@tool` 只是**使用了装饰器语法**。

至于这个装饰器具体干什么，要看实现。

例如我们今天自己的：

```python
@registry.register(...)
def read_file(...):
    ...
```

本质上只是：

```text
把函数登记进 Registry
```

原函数仍然返回。

而某些框架里的：

```python
@tool
def search(...):
    ...
```

可能会：

```text
普通 Python 函数
        ↓
转换成 Tool 对象
        ↓
增加 schema
        ↓
增加描述
        ↓
变成框架可调用对象
```

所以以后看到：

```python
@xxx
```

不要先问：

> “这个装饰器是什么意思？”

应该先问：

> **“这个装饰器拿到原函数以后，对它做了什么？”**

这个思维比死记 `@tool` 有用得多。

---

# 十三、Registry 解决了工具管理，但它仍然没有权限

现在假设：

```python
@registry.register(
    name="read_file",
    description="读取文件",
    risk="low",
)
def read_file(path):
    return f"读取 {path}"
```

以及：

```python
@registry.register(
    name="delete_file",
    description="删除文件",
    risk="high",
)
def delete_file(path):
    return f"删除 {path}"
```

Registry 可以知道：

```text
read_file
    risk = low

delete_file
    risk = high
```

但是：

> **知道风险，不等于阻止执行。**

这就是 Permission Gate 出场的原因。

---

# 十四、Permission Gate 是什么

Permission Gate 本质上就是一个决策函数。

输入：

```text
工具信息
+
当前权限条件
```

输出：

```text
allow
deny
need_approval
```

可以先写成最简单的：

```python
from typing import Literal

Decision = Literal["allow", "deny", "need_approval"]


class PermissionGate:

    def check(self, tool: ToolSpec) -> Decision:
        if tool.risk == "low":
            return "allow"

        if tool.risk == "high":
            return "need_approval"

        return "deny"
```

这里其实非常简单。

它不是执行器。

它只是：

> **判断能不能执行。**

---

# 十五、为什么这里要把 Gate 和工具本身分开？

这是今天真正的工程设计。

错误的写法可能是：

```python
def delete_file(path):
    if user_is_admin():
        ...
```

这样每个工具自己负责权限。

最后就会变成：

```text
read_file       自己判断权限
delete_file     自己判断权限
send_email      自己判断权限
pay_money       自己判断权限
...
```

权限逻辑散落整个项目。

以后规则一改：

```text
所有 high-risk 工具都需要审批
```

你就得改很多地方。

而现在：

```text
Tool
  ↓
只负责“怎么做”

PermissionGate
  ↓
负责“允许不允许做”
```

职责就清楚了。

---

# 十六、所以今天真正形成了“能力”和“策略”的分离

这个可以作为今天最重要的一句话：

```text
Tool
= 我能做什么

Permission Gate
= 我现在允不允许做
```

例如：

```text
delete_file
```

本身具备：

> 删除文件的能力。

但：

```text
PermissionGate
```

可以决定：

```text
普通情况下 → deny
人工审批后 → allow
```

所以：

> **工具负责能力，Gate 负责策略。**

这也是 Harness 开始有意义的地方。

---

# 十七、把 Registry + Gate 串起来

真正的工具调用入口可以设计成：

```python
def call_tool(
    registry: ToolRegistry,
    gate: PermissionGate,
    tool_name: str,
    args: dict,
    approved: bool = False,
):
    tool = registry.get(tool_name)

    decision = gate.check(tool)

    if decision == "deny":
        return "permission denied"

    if decision == "need_approval" and not approved:
        return "approval required"

    return tool.func(**args)
```

这段代码非常重要。

因为它第一次把今天两个组件连接起来：

```text
tool_name
   ↓
Registry
   ↓
找到 ToolSpec
   ↓
PermissionGate
   ↓
判断
   ↓
allow？
   ↓
真正执行 func
```

---

# 十八、这里要特别区分“逻辑流”和“数据流”

## 逻辑流

谁决定下一步？

```text
call_tool()
   ↓
Registry.get()
   ↓
拿到 ToolSpec
   ↓
Gate.check()
   ↓
decision
   ↓
allow / deny / need_approval
   ↓
决定是否进入 func()
```

真正决定流程走向的是：

```python
decision
```

---

## 数据流

什么东西在流动？

最开始有：

```python
tool_name = "delete_file"
args = {"path": "a.txt"}
```

先经过 Registry：

```text
"delete_file"
    ↓
ToolSpec
```

然后 Gate 使用：

```text
ToolSpec
    ↓
decision
```

最后真正执行：

```python
tool.func(**args)
```

所以：

```text
tool_name
   ↓
ToolSpec
   ↓
decision
   ↓
tool.func + args
   ↓
result
```

---

# 十九、一个完整调用到底发生了什么

假设 Agent 想做：

```python
call_tool(
    registry,
    gate,
    "delete_file",
    {"path": "a.txt"},
)
```

完整执行链：

### 第一步：Agent 提出工具调用

```text
delete_file
```

同时给参数：

```text
path = a.txt
```

---

### 第二步：Registry 查询

```python
registry.get("delete_file")
```

得到：

```text
ToolSpec(
    name="delete_file",
    description="删除文件",
    risk="high",
    func=<真正的 delete_file 函数>
)
```

---

### 第三步：Gate 判断

```python
gate.check(tool)
```

因为：

```text
risk = high
```

所以：

```text
need_approval
```

---

### 第四步：流程暂停

这时候**不能直接执行**：

```python
tool.func(...)
```

而是应该进入人工审批流程。

这就连接到了你 W6 学过的：

```text
HITL
```

逻辑变成：

```text
Agent
 ↓
Tool Registry
 ↓
Permission Gate
 ↓
need_approval
 ↓
HITL
 ↓
人工决定
 ↓
allow / deny
```

所以今天没有凭空出现一个新概念。

而是：

> **W6 学过的 HITL，今天被重新利用到了工具权限控制上。**

---

# 二十、审批以后发生什么

假设人工允许：

```python
approved = True
```

再次进入：

```python
call_tool(...)
```

或者把审批结果继续传回原流程。

最终：

```text
PermissionGate
      ↓
allow
      ↓
tool.func(**args)
      ↓
delete_file("a.txt")
      ↓
result
```

所以完整链路就是：

```text
Agent
  ↓
选择工具
  ↓
Registry 找工具
  ↓
拿到 ToolSpec
  ↓
PermissionGate 判断
  ↓
┌───────────────┬──────────────┬─────────────────┐
│    allow      │     deny     │ need_approval   │
│       ↓       │       ↓      │        ↓        │
│   执行工具    │    直接拒绝   │      HITL       │
│               │              │        ↓        │
│               │              │   人工审批       │
│               │              │        ↓        │
│               │              │      allow      │
└───────────────┴──────────────┴────────┬────────┘
                                        ↓
                                   执行工具
                                        ↓
                                      result
```

这就是今天整个任务。

---

# 二十一、为什么今天的 Registry 不能顺便做权限判断？

当然可以。

从“能不能写出来”的角度：

```python
registry.get(...)
```

里面也可以判断权限。

但这样会把两个完全不同的问题混在一起：

```text
工具管理
+
权限管理
```

未来 Registry 还要负责：

* 工具发现
* 名称冲突
* schema
* 描述
* 工具列表

Permission Gate 还要负责：

* risk
* allow
* deny
* approval
* 用户权限
* 环境限制

所以把它们拆开，是为了以后能够独立演化。

今天虽然代码不多，但已经开始进入真正的工程设计：

> **不是“能不能运行”，而是“职责是不是清楚”。**

---

# 二十二、今天一个很重要的边界：Gate 不是安全魔法

这里也要保持一个工程上的清醒。

如果项目里同时存在：

```python
call_tool(...)
```

和：

```python
delete_file(...)
```

那么程序员完全可以绕过 Gate：

```python
delete_file("a.txt")
```

所以今天这种 Permission Gate 的设计，更准确地说是：

> **统一的工具执行控制入口。**

如果真正想做到严格权限控制，就必须让“工具执行”本身经过统一边界。

也就是说：

```text
不应该：

Agent → Tool
Agent → Tool
Agent → Tool

而应该：

Agent
  ↓
统一工具执行入口
  ↓
Registry
  ↓
Permission Gate
  ↓
Tool
```

否则 Gate 只是“建议”，不是唯一入口。

这个区别以后做真正 Agent 系统时非常重要。

---

# 二十三、今天的装饰器，最后再用一句话理解

看到：

```python
@registry.register(
    name="delete_file",
    description="删除文件",
    risk="high",
)
def delete_file(path):
    ...
```

脑子里不要把它看成一条特殊语法。

直接展开成：

```python
def delete_file(path):
    ...

decorator = registry.register(
    name="delete_file",
    description="删除文件",
    risk="high",
)

delete_file = decorator(delete_file)
```

然后继续往里面展开：

```text
register(...)
    ↓
返回 decorator
    ↓
decorator(delete_file)
    ↓
创建 ToolSpec
    ↓
ToolSpec 放入 Registry
    ↓
返回原来的 delete_file
```

这时候装饰器基本就不神秘了。

---

# 二十四、这次真正应该记住的不是几个 API

今天最值得留下来的其实是这几个关系：

```text
Tool
= 能力

Tool Registry
= 工具目录 / 统一发现

Permission Gate
= 权限决策

HITL
= 人工介入决策

Harness
= 把这些控制能力放到 Agent 和执行环境之间
```

再压缩成一条：

```text
Agent 想做事
    ↓
Registry：你想找哪个工具？
    ↓
拿到 ToolSpec
    ↓
Gate：这个工具现在允许做吗？
    ↓
allow / deny / need_approval
    ↓
必要时 HITL
    ↓
真正执行 Tool
```

所以今天的变化不是：

> “我又学了两个类。”

而是：

> **Agent 从“能调用工具”，开始进入“工具调用受系统控制”的阶段。**

---

# 二十五、关键坑 / 真正理解

## 1. Registry 不是执行器

```python
registry.get("read_file")
```

只是找到工具。

真正执行的是：

```python
tool.func(...)
```

---

## 2. Gate 不是工具

Gate 不负责：

```text
删除文件
发邮件
查询数据库
```

它只负责：

```text
允许
拒绝
审批
```

---

## 3. 装饰器不是“魔法标记”

```python
@xxx
```

本质上要还原成：

```python
func = xxx(func)
```

遇到陌生装饰器，先找：

> **xxx 到底返回了什么？**

---

## 4. 带参数的装饰器有两层调用

看到：

```python
@register(name="xxx")
```

不要直接理解成：

```python
register(function)
```

它通常是：

```text
register(name="xxx")
       ↓
    decorator
       ↓
decorator(function)
```

---

## 5. `return func` 有实际意义

如果装饰器最终：

```python
return None
```

那么：

```python
@decorator
def foo():
    ...
```

执行以后可能变成：

```python
foo = None
```

原函数就没了。

所以我们的 Registry 装饰器返回原函数，是因为：

> **注册工具，但不改变原工具的调用方式。**

---

# 二十六、完整 Demo

下面这个版本把今天的核心机制全部放在一起。

```python
from dataclasses import dataclass
from typing import Callable, Literal


# =========================
# Tool 定义
# =========================

@dataclass
class ToolSpec:
    name: str
    description: str
    risk: str
    func: Callable


# =========================
# Tool Registry
# =========================

class ToolRegistry:

    def __init__(self):
        self._tools: dict[str, ToolSpec] = {}

    def register(self, name: str, description: str, risk: str):
        def decorator(func: Callable):
            if name in self._tools:
                raise ValueError(f"tool already exists: {name}")

            self._tools[name] = ToolSpec(
                name=name,
                description=description,
                risk=risk,
                func=func,
            )

            return func

        return decorator

    def get(self, name: str) -> ToolSpec:
        if name not in self._tools:
            raise KeyError(f"unknown tool: {name}")

        return self._tools[name]


# =========================
# Permission Gate
# =========================

Decision = Literal["allow", "deny", "need_approval"]


class PermissionGate:

    def check(self, tool: ToolSpec) -> Decision:

        if tool.risk == "low":
            return "allow"

        if tool.risk == "high":
            return "need_approval"

        return "deny"


# =========================
# 创建 Registry
# =========================

registry = ToolRegistry()


# =========================
# 注册工具
# =========================

@registry.register(
    name="read_file",
    description="读取文件",
    risk="low",
)
def read_file(path: str):
    return f"读取文件：{path}"


@registry.register(
    name="delete_file",
    description="删除文件",
    risk="high",
)
def delete_file(path: str):
    return f"删除文件：{path}"


# =========================
# 创建 Gate
# =========================

gate = PermissionGate()


# =========================
# 统一工具执行入口
# =========================

def call_tool(
    tool_name: str,
    args: dict,
    approved: bool = False,
):
    # 1. 从 Registry 找工具
    tool = registry.get(tool_name)

    # 2. 让 Permission Gate 判断
    decision = gate.check(tool)

    # 3. 拒绝
    if decision == "deny":
        return {
            "status": "denied",
            "reason": "permission denied",
        }

    # 4. 需要人工审批
    if decision == "need_approval" and not approved:
        return {
            "status": "need_approval",
            "tool": tool.name,
        }

    # 5. 允许执行
    result = tool.func(**args)

    return {
        "status": "success",
        "result": result,
    }


# =========================
# 调用
# =========================

print(
    call_tool(
        "read_file",
        {"path": "a.txt"},
    )
)


print(
    call_tool(
        "delete_file",
        {"path": "a.txt"},
    )
)


print(
    call_tool(
        "delete_file",
        {"path": "a.txt"},
        approved=True,
    )
)
```

这个 Demo 不复杂，但已经把今天的完整结构包含进去了：

```text
@registry.register
        ↓
Tool 注册
        ↓
ToolSpec
        ↓
Registry
        ↓
call_tool()
        ↓
PermissionGate
        ↓
allow / deny / need_approval
        ↓
Tool.func()
```

---

# 二十七、今天这一层放回整个 W7

到这里可以重新看整个 W7：

```text
多 Agent
   ↓
Supervisor / Worker
   ↓
反思与停止条件
   ↓
Tool Registry
   ↓
Permission Gate
   ↓
真正可控的 Agent 执行环境
```

前面的内容更多是在解决：

> **Agent 怎么思考、怎么协作、怎么控制流程。**

今天开始增加：

> **Agent 想动手的时候，系统怎么控制它。**

所以 Harness 开始有了非常具体的形状：

```text
能力
  ↓
工具

管理
  ↓
Registry

控制
  ↓
Permission Gate

人工决策
  ↓
HITL
```

这几个东西以后还会继续往下扩展成更完整的 Agent Runtime / Harness。
