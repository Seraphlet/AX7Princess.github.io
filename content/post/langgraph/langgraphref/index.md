---
description: ""
title: "反思循环：Critique-Revise，让 Agent 自己检查、修改，再决定要不要继续"
draft: false
date: "2026-09-16T14:35:03+08:00"
slug: "LangGraphref"
categories:
 - LangGraph
tags:
 - Critique-Revise
image: ""
---

# W7-Day3｜反思循环：Critique-Revise，让 Agent 自己检查、修改，再决定要不要继续

昨天是 Supervisor：一个 Agent 负责调度，多个 Worker 负责执行。

今天把昨天的 `reviewer` 往前再推进一步：

```text
generator
    ↓
critic
    ↓
┌───────────────┬────────────────┬─────────────────┐
│ 达标          │ 次数用尽       │ 质量回退        │
│ score >= 8    │ revise >= 3    │ score 下降      │
│      ↓        │      ↓         │       ↓          │
│     END       │     END        │      END         │
└───────────────┴────────────────┴─────────────────┘
                    ↑
                 未达标
                    ↓
                 reviser
                    ↓
                 critic
                    ↓
                  循环
```

今天的核心不是“让模型多改几遍”。

而是：

> **让模型的评审结果进入程序控制流，再由程序决定继续修改还是结束。**

所以今天实际上是在做一个完整的：

```text
评审 → 修改 → 再评审 → 决定是否继续
```

---

# 一、先看今天的 State

今天的 State：

```python
class RState(MessagesState):
    topic: str
    draft: str
    score: int
    issues: list[str]
    revise_count: int

    # 每轮评审轨迹
    trajectory: Annotated[list, operator.add]

    # 历史最优稿
    best_draft: str
    best_score: int
```

如果使用最终版本，还会增加硬伤字段：

```python
class RState(MessagesState):
    topic: str
    draft: str
    score: int
    issues: list[str]
    fatal_issues: list[str]
    revise_count: int
    trajectory: Annotated[list, operator.add]
    best_draft: str
    best_score: int
```

所以 State 可以先理解成：

```text
messages       → 工作历史
topic          → 当前任务
draft          → 当前文章
score          → 当前评分
issues         → 当前普通问题
fatal_issues   → 当前硬伤
revise_count   → 已经修改几次
trajectory     → 历史评审轨迹
best_draft     → 历史最优稿
best_score     → 历史最高分
```

这里和之前学习 State 时的理解是一致的：

> **State 不是“变量大杂烩”，每一个字段都应该承担明确职责。**

---

# 二、为什么今天不直接让 `critic` 返回一句话

最简单的 critic 可能写成：

```text
文章还可以，但是第二段缺少例子，结尾也比较仓促。
```

对人来说没问题。

但是程序马上会遇到几个问题：

```text
到底多少分？
是否达标？
有什么问题？
应该继续改吗？
```

如果返回自由文本，程序就还要自己从文本里猜。

于是今天改成结构化输出：

```python
class Critique(BaseModel):
    score: int = Field(
        ge=1,
        le=10,
        description="1-10分，8分及以上达标",
    )

    issues: list[str] = Field(
        default_factory=list,
        description="软问题，影响分数",
    )

    passed: bool = Field(
        description="是否达标",
    )
```

最终版本再增加：

```python
fatal_issues: list[str] = Field(
    default_factory=list,
    description=(
        "硬伤：编造/无来源的具体数据、事实错误、违规内容。"
        "只要非空，无论多少分都不达标"
    ),
)
```

于是模型的输出不再只是：

```text
一段话
```

而变成：

```text
score
issues
fatal_issues
passed
```

这样程序才能继续处理。

所以这里形成了一个非常重要的链：

```text
LLM
 ↓
结构化输出
 ↓
State
 ↓
条件边
 ↓
控制流
```

---

# 三、先看 `Critique`

代码：

```python
class Critique(BaseModel):
    """② 强制 critic 输出可判断的结果，而不是一段自由文本"""

    score: int = Field(
        ge=1,
        le=10,
        description="1-10分，8分及以上达标",
    )

    issues: list[str] = Field(
        default_factory=list,
        description="软问题，影响分数",
    )

    fatal_issues: list[str] = Field(
        default_factory=list,
        description=(
            "硬伤：编造/无来源的具体数据、事实错误、违规内容。"
            "只要非空，无论多少分都不达标"
        ),
    )

    passed: bool = Field(
        description="是否达标",
    )
```

这里的几个字段并不是同一种东西。

---

## `score`

```python
score: int
```

代表：

> 当前稿件的总体评分。

并且通过：

```python
ge=1
le=10
```

把合法范围限制在：

```text
1 ~ 10
```

也就是说，模型不能正常返回：

```text
score = 15
```

---

## `issues`

```python
issues: list[str]
```

代表：

> 普通问题。

例如：

```text
第二段缺少案例
结论过于仓促
表达略显重复
```

它们的特点是：

```text
发现问题
    ↓
修改
    ↓
可能解决
```

所以这是一个**软问题集合**。

---

## `fatal_issues`

```python
fatal_issues: list[str]
```

这里和 `issues` 不一样。

例如：

```text
编造统计数据
事实错误
无来源的具体事实
违规内容
```

这类问题不能简单用：

```text
“多写一点”
```

解决。

所以它单独成为一个硬门槛。

---

## `passed`

```python
passed: bool
```

这个字段看起来很有用。

但今天后来会发现：

> **不能直接相信模型自己给出的 `passed`。**

因为它也是模型输出的。

真正控制程序的最终判断，还是应该由程序自己做。

后面会再看。

---

# 四、为什么 `score` 还要手动校验

结构化输出并不意味着所有业务约束都自动解决。

例如：

```python
def _parse_critique(raw) -> Critique:
    """手动校验：json_mode 不保证数值范围"""

    if not isinstance(raw, Critique):
        raw = Critique.model_validate(raw)

    if not (1 <= int(raw.score) <= 10):
        raise ValueError(
            f"score 越界: {raw.score}"
        )

    return raw
```

这里其实是两层校验。

第一层：

```python
Critique
```

负责定义结构。

第二层：

```python
_parse_critique()
```

负责程序自己的业务检查。

可以理解为：

```text
模型输出
    ↓
结构化解析
    ↓
程序二次校验
    ↓
真正进入 State
```

所以：

> **Schema 是第一道门，程序逻辑是第二道门。**

---

# 五、先看 `generator`

代码：

```python
def generator(state: RState) -> dict:
    """初稿"""

    r = llm.invoke(
        f"请写一篇约200字的中文短文，"
        f"主题：{state['topic']}。只输出正文。"
    )

    draft = str(r.content)

    print(
        f"[generator] 初稿 {len(draft)}字"
    )

    return {
        "draft": draft,
        "revise_count": 0,
        "best_draft": draft,
        "best_score": 0,
    }
```

它其实非常简单。

输入：

```text
State
```

读取：

```python
state["topic"]
```

生成：

```text
draft
```

然后返回：

```python
{
    "draft": draft,
    "revise_count": 0,
    "best_draft": draft,
    "best_score": 0,
}
```

---

## 为什么初始化 `best_draft`

因为从系统角度看，初稿也是一个候选版本。

例如：

```text
v1 = 7分
```

后面可能：

```text
v2 = 8分
v3 = 7分
```

那么最终需要知道：

```text
历史最好的是 v2
```

所以从第一轮开始，就应该有：

```python
best_draft = draft
best_score = 0
```

第一次 critic 之后再真正更新最佳分数。

---

# 六、再看 `JUDGE_SYS`

代码：

```python
JUDGE_SYS = """你是严格但公正的中文写作评审。

评分维度：
① 观点清晰度
② 论据支撑
③ 结构层次
④ 表达流畅

评分标准：
8分及以上为达标。

issues 必须写「具体可改的问题」，
不要写「可以更好」这类空话。

issues 最多3条，
每条不超过15字。

输出格式：
只输出一个 JSON 对象，形如：

{
  "score": 7,
  "issues": [
      "第2段缺少具体例子",
      "结论过于仓促"
  ],
  "fatal_issues": [],
  "passed": false
}
"""
```

这一部分是：

> **告诉 critic 应该如何评价。**

这里其实包含三层信息。

第一层：

```text
评什么
```

例如：

```text
观点
论据
结构
表达
```

第二层：

```text
怎么给分
```

例如：

```text
8分以上达标
```

第三层：

```text
输出什么结构
```

例如：

```json
{
    "score": 7,
    "issues": [],
    "fatal_issues": [],
    "passed": false
}
```

于是模型输出就有了一个固定接口。

---

# 七、critic 节点到底做什么

先看主体：

```python
def critic(state: RState) -> dict:
    """评审：打分+列问题。同时更新轨迹与最优稿"""

    judge = llm.with_structured_output(
        Critique,
        method="json_mode",
    )

    prompt = (
        f"{JUDGE_SYS}\n\n"
        f"文稿:\n{state['draft']}"
    )

    ...
```

这里其实可以拆成：

```text
当前 State
    ↓
取 draft
    ↓
交给 critic
    ↓
得到 Critique
```

---

# 八、为什么这里重新调用 `with_structured_output`

```python
judge = llm.with_structured_output(
    Critique,
    method="json_mode",
)
```

这个 `judge` 和普通的：

```python
llm
```

区别在于：

```text
普通 llm
    ↓
自由文本

judge
    ↓
按照 Critique 结构输出
```

所以从角色上可以理解成：

```text
llm
 ↓
生成 / 修改文章

judge
 ↓
评价文章
```

即使底层使用的是同一个模型，也可以通过不同的输出契约，把它变成不同角色。

---

# 九、为什么这里用 `json_mode`

之前已经实测过几条结构化输出路径。

默认：

```python
with_structured_output(Critique)
```

在当前环境走到了 `json_schema`，不支持。

`function_calling` 又遇到了：

```text
Thinking mode does not support this tool_choice
```

最后使用：

```python
method="json_mode"
```

所以今天的代码：

```python
judge = llm.with_structured_output(
    Critique,
    method="json_mode",
)
```

这里真正值得记住的是：

> **接口名字一样，不代表底层协议路径一样。**

以后遇到模型兼容问题，不能简单理解成：

```text
“这个模型不支持结构化输出”
```

而应该进一步问：

```text
到底是哪一种结构化输出方式不支持？
```

---

# 十、critic 为什么要尝试三次

代码：

```python
c = None

for attempt in range(3):
    try:
        c = _parse_critique(
            judge.invoke(prompt)
        )
        break

    except Exception as e:
        print(
            f"[warn] 第 {attempt + 1} 次评审解析失败: "
            f"{type(e).__name__}: {str(e)[:80]}"
        )
```

这里处理的是：

```text
调用失败
解析失败
字段不符合预期
```

而不是评分逻辑本身。

所以：

```text
第一次失败
    ↓
重试

第二次失败
    ↓
重试

第三次失败
    ↓
进入保守兜底
```

---

# 十一、为什么解析失败以后不能默认“通过”

代码：

```python
if c is None:
    c = Critique(
        score=1,
        issues=["评审解析失败，无法判断"],
        fatal_issues=[],
        passed=False,
    )

    print(
        "[warn] 三次都失败 → "
        "保守判定为不达标"
    )
```

这里采用的是：

> **宁可多改一轮，也不能因为评审失败就直接放行。**

如果解析失败还继续：

```text
score = 8
passed = True
```

那就会变成：

```text
没有得到有效评审
        ↓
却被当成通过
```

这属于非常危险的默认行为。

所以这里是：

```text
无法判断
    ↓
保守处理
    ↓
不通过
```

---

# 十二、为什么 `passed` 不是程序最终事实来源

代码：

```python
passed_by_code = c.score >= PASS_SCORE

if passed_by_code != c.passed:
    print(
        f"[warn] 模型说 passed={c.passed}，"
        f"但 score={c.score} 对照阈值 "
        f"{PASS_SCORE} 应为 {passed_by_code}"
        f" → 以程序的判定为准"
    )
```

这里非常值得理解。

假设：

```text
模型：
score = 8
passed = True
```

如果：

```python
PASS_SCORE = 11
```

程序自己计算：

```text
8 >= 11
↓
False
```

于是：

```text
模型：True
程序：False
```

这时候程序不去争论模型为什么这么判断。

直接：

```text
程序阈值
    ↓
作为控制流事实
```

所以：

> **模型给的是判断建议，程序给的是执行规则。**

---

# 十三、把“模型判断”和“程序判断”彻底拆开

现在可以画成：

```text
                critic
                  │
          ┌───────┴────────┐
          ↓                ↓
     模型判断            程序判断
     passed=True       score >= threshold
          │                │
          │                ↓
          │             控制流
          │
          └────→ 仅用于交叉检查
```

这其实是 Agent 系统里一个非常重要的思想：

> **不要让模型自己返回一个 `approved=True`，然后程序看到 True 就无条件执行。**

因为：

```text
approved
```

本身也是模型产生的数据。

---

# 十四、critic 如何更新 `best_draft`

代码：

```python
score = int(c.score)

best_score = state.get(
    "best_score",
    0,
)

best_draft = state.get(
    "best_draft",
    "",
)

if score > best_score:
    best_score = score
    best_draft = state["draft"]
```

这里就是：

```text
当前分数
    ↓
和历史最高分比较
    ↓
如果更高
    ↓
更新 best
```

例如：

```text
第1轮 7
best = 7

第2轮 8
best = 8

第3轮 7
best 仍然 = 8
```

所以：

```text
draft
```

和：

```text
best_draft
```

是两个不同概念。

```text
draft
    = 当前正在修改的版本

best_draft
    = 到目前为止历史最好版本
```

---

# 十五、为什么这个机制很重要

因为 Self-Refine 并不是单调优化。

可能：

```text
7 → 8 → 7
```

如果没有 `best_draft`：

```text
最终 = 7
```

有了以后：

```text
最终 = 历史最佳 8
```

所以它实际上给循环增加了一层：

> **回退保护。**

可以理解成：

```text
当前版本
   ↓
如果更好
   ↓
保存

如果更差
   ↓
保留历史最优
```

这和 Git、checkpoint、best model 等很多工程思想其实是同一类思路：

> **不要只记住“现在是什么”，还要记住“最好是什么”。**

---

# 十六、critic 为什么要记录 `trajectory`

代码：

```python
entry = dict(
    round=len(state.get("trajectory", [])),
    score=score,
    issues=c.issues,
    fatal_issues=c.fatal_issues,
    draft_len=len(state["draft"]),
    draft_head=(
        state["draft"][:60]
        .replace("\n", " ")
    ),
)
```

这里保存：

```text
第几轮
多少分
有什么问题
有什么硬伤
稿子多长
稿子开头
```

然后：

```python
return {
    "score": score,
    "issues": c.issues,
    "fatal_issues": c.fatal_issues,
    "trajectory": [entry],
    "best_score": best_score,
    "best_draft": best_draft,
    ...
}
```

这里最容易忽略的是：

```python
trajectory: Annotated[list, operator.add]
```

这意味着：

```text
不是覆盖
而是追加
```

所以：

```text
第1轮
→ [entry1]

第2轮
→ [entry1, entry2]

第3轮
→ [entry1, entry2, entry3]
```

这就是为什么最后可以打印：

```text
7 → 8 → 7
```

而不是只能看到最后一次评分。

---

# 十七、为什么 `trajectory` 要用 reducer

如果没有：

```python
Annotated[list, operator.add]
```

State 更新更接近：

```text
旧 trajectory
    ↓
新 trajectory
    ↓
覆盖
```

那就很难保留完整轨迹。

现在通过：

```python
operator.add
```

表达：

> 每次节点返回一个新的历史片段，把它追加进旧历史。

所以：

```text
trajectory
```

实际上是一个典型的：

> **State 累积字段。**

---

# 十八、critic 最终返回的 State 更新

完整看：

```python
return {
    "score": score,
    "issues": c.issues,
    "fatal_issues": c.fatal_issues,
    "trajectory": [entry],
    "best_score": best_score,
    "best_draft": best_draft,
    "messages": [
        (
            "assistant",
            f"[评审] {score} 分 | "
            f"问题: {c.issues} | "
            f"硬伤: {c.fatal_issues}"
        )
    ],
}
```

注意它没有返回：

```python
draft
```

因为：

```text
critic 不负责修改文章
```

它只负责：

```text
评审
```

所以 State 中：

```text
draft
```

仍然是当前稿子。

下一步：

```text
route
```

决定要不要进入：

```text
reviser
```

这就是很清楚的职责边界：

```text
generator
    → 生成

critic
    → 评价

reviser
    → 修改
```

---

# 十九、再看 `reviser`

代码：

```python
def reviser(state: RState) -> dict:
    """按评审意见重写"""

    r = llm.invoke(
        "请根据评审意见改进以下文稿，"
        "保持主题不变、篇幅相近，"
        "只输出改进后的正文。\n\n"

        f"原稿:\n{state['draft']}\n\n"

        f"评审意见:\n{state['issues']}"
    )

    draft = str(r.content)

    n = (
        state.get("revise_count", 0)
        + 1
    )

    print(
        f"[reviser] 第{n}次修改，"
        f"新稿{len(draft)}字"
        f"(原 {len(state['draft'])}字)"
    )

    return {
        "draft": draft,
        "revise_count": n,
    }
```

它和 critic 的区别非常明显。

critic：

```text
输入：
draft

输出：
score
issues
fatal_issues
trajectory
```

reviser：

```text
输入：
draft
issues

输出：
new draft
revise_count
```

所以：

```text
critic
    ↓
结构化意见

reviser
    ↓
新的文章
```

---

# 二十、为什么 reviser 不带完整历史

代码：

```python
f"原稿:\n{state['draft']}\n\n"
f"评审意见:\n{state['issues']}"
```

没有把：

```text
第一轮 draft
第一轮 issues
第二轮 draft
第二轮 issues
...
```

全部塞进去。

而是只给：

```text
当前稿
+
当前意见
```

因此每轮都像：

```text
上一版
+
这一轮建议
    ↓
下一版
```

这样做的直接目的：

> **不让上下文随着迭代不断膨胀。**

但也会带来一个特点：

> reviser 做的是“重写”，不是“打补丁”。

所以才会出现：

```text
243字
→
400字
→
645字
```

模型为了满足当前意见，可能顺手重写很多内容。

这也是今天观察到的一个真实问题：

> **Prompt 里说“篇幅相近”，并不等于模型一定会严格遵守。**

---

# 二十一、为什么 `revise_count` 要自己加

代码：

```python
n = (
    state.get("revise_count", 0)
    + 1
)
```

这里记录：

> reviser 节点实际执行了多少次。

比如：

```text
generator
→ critic
→ reviser
→ critic
→ reviser
→ critic
```

那么：

```text
revise_count = 2
```

注意：

```text
critic 次数
≠
reviser 次数
```

如果第一次 critic 是检查初稿：

```text
critic = 1
```

那么：

```text
reviser = critic - 1
```

在正常完整路径下可以大致这样理解。

---

# 二十二、真正控制循环的是 `route_after_critic`

代码：

```python
def route_after_critic(
    state: RState,
) -> Literal["reviser", "end"]:

    score = state["score"]
    count = state.get("revise_count", 0)
    traj = state.get("trajectory", [])
    fatal = state.get("fatal_issues") or []

    if count >= MAX_REVISE:
        return "end"

    if fatal:
        return "reviser"

    if score >= PASS_SCORE:
        return "end"

    if (
        STALL_STOP
        and len(traj) >= 2
        and traj[-1]["score"]
        < traj[-2]["score"]
    ):
        return "end"

    return "reviser"
```

这个函数其实就是：

> **今天整个图最核心的控制器。**

可以翻译成人话：

```text
修改次数到了吗？
    ↓
    是 → END

存在硬伤吗？
    ↓
    是 → reviser

分数达标了吗？
    ↓
    是 → END

这一轮变差了吗？
    ↓
    是 → END

否则
    ↓
  reviser
```

---

# 二十三、为什么条件顺序很重要

注意：

```python
if count >= MAX_REVISE:
    return "end"

if fatal:
    return "reviser"
```

这不是随便排列的。

因为假设：

```text
fatal_issues 一直存在
```

那么如果没有：

```text
MAX_REVISE
```

系统可能：

```text
reviser
→
critic
→
reviser
→
critic
→
...
```

无限循环。

所以：

> **成本上限必须能够最终把循环截断。**

这就是 `MAX_REVISE` 的真正意义：

```text
不是质量规则
而是成本规则
```

---

# 二十四、为什么 `MAX_REVISE` 必须存在

Self-Refine 如果只有：

```text
没达到标准
    ↓
继续修改
```

理论上可以：

```text
无限继续
```

例如：

```text
critic
 ↓
reviser
 ↓
critic
 ↓
reviser
 ↓
critic
 ↓
...
```

每一轮都要调用模型。

所以：

```python
MAX_REVISE = 3
```

本质是在声明：

> **系统最多允许自动修改几次。**

这不是“模型觉得够了”。

而是：

> **程序给模型设置了硬边界。**

---

# 二十五、为什么不能用 `<=` 检测停滞

原本容易写：

```python
traj[-1]["score"] <= traj[-2]["score"]
```

意思是：

```text
没有提高
    ↓
停
```

但这样会误杀一种情况：

```text
7
↓
7
↓
7
↓
9
```

前几轮没有变好。

但不能因此推导：

```text
以后也不会变好
```

所以改成：

```python
traj[-1]["score"] < traj[-2]["score"]
```

只在：

```text
7 → 6
```

这种真正变差的时候停。

于是两个出口开始分工：

```text
score 下降
    → STALL_STOP

修改次数达到上限
    → MAX_REVISE
```

可以把它们分别理解成：

```text
质量保护
+
成本保护
```

---

# 二十六、为什么这两个条件测试时会互相掩盖

假设：

```text
7 → 8 → 7
```

原本想测试：

```text
MAX_REVISE = 3
```

但是第三轮分数：

```text
7 < 8
```

已经触发了：

```python
STALL_STOP
```

于是直接：

```text
END
```

最终：

```text
revise_count = 2
```

根本没走到：

```text
MAX_REVISE = 3
```

所以：

> **一个机制是否正确，不能只看最终“停了”。**

因为你还要知道：

> **到底是哪一个出口让它停下来的？**

---

# 二十七、所以 `forced` 模式是干什么的

代码：

```python
MODE = (
    sys.argv[1]
    if len(sys.argv) > 1
    else "normal"
)

FORCED = MODE == "forced"

PASS_SCORE = (
    11
    if FORCED
    else 8
)

STALL_STOP = (
    not FORCED
    and os.getenv("STALL", "1") != "0"
)
```

普通模式：

```text
PASS_SCORE = 8
STALL_STOP = True
```

forced 模式：

```text
PASS_SCORE = 11
STALL_STOP = False
```

而评分合法范围只有：

```text
1 ~ 10
```

所以：

```text
最高只能 10
目标是 11
```

意味着：

> **这一轮不可能因为“达标”结束。**

同时：

```text
STALL_STOP = False
```

又关闭了停滞出口。

这样剩下的主要出口就只剩：

```text
MAX_REVISE
```

于是才能干净地测试：

```text
次数用尽
```

这就是今天很重要的测试思想：

> **测试一个出口时，把其他出口尽量关掉。**

---

# 二十八、为什么不能单纯依赖 LLM 运行来测试条件边

因为：

```text
调用 LLM
+
测试条件边
```

成本高，而且结果有随机性。

但是：

```python
route_after_critic(state)
```

其实是纯逻辑。

给它：

```python
{
    "score": 7,
    "revise_count": 1,
    "trajectory": [
        {"score": 7},
        {"score": 7},
    ],
}
```

就可以直接得到：

```text
reviser
```

完全不需要调用模型。

所以可以写：

```python
def mk(score, count, scores):
    return {
        "score": score,
        "revise_count": count,
        "trajectory": [
            {"score": s}
            for s in scores
        ],
    }
```

---

# 二十九、直接测试“平局”这个特殊情况

例如：

```python
(
    "★ 平局(验证 < 的那行)",
    mk(
        7,
        1,
        [7, 7],
    ),
    "reviser",
)
```

现在：

```text
上一轮 = 7
这一轮 = 7
```

使用：

```python
<
```

那么：

```text
7 < 7
↓
False
```

所以：

```text
不会触发停滞
↓
reviser
```

但如果代码写成：

```python
<=
```

则：

```text
7 <= 7
↓
True
↓
end
```

所以这一条测试是非常好的：

> **可证伪测试。**

因为错误版本必然失败。

---

# 三十、`test_router.py` 为什么可以做到 0 成本

完整逻辑：

```python
fail = 0

print(f"[STALL_STOP={stall}]")

for name, st, expect in cases:
    got = rg.route_after_critic(st)

    ok = got == expect

    print(
        f"{'✅' if ok else '❌'} "
        f"{name}: "
        f"期望 {expect}, "
        f"实际 {got}"
    )

    fail += (not ok)

print(
    f"\n"
    f"{'全部通过' if not fail else f'{fail} 项失败'}"
)
```

没有：

```python
llm.invoke(...)
```

没有：

```text
API
```

所以：

```text
0 token
0 调用费用
```

但仍然可以验证：

```text
达标出口
次数出口
停滞出口
平局
正常进步
单轮场景
```

这就是今天开始形成的一个很重要的习惯：

> **纯函数逻辑优先用确定性测试，不要拿 LLM 运行充当单元测试。**

---

# 三十一、但单测全绿也不代表整个图一定正确

这里还有一层。

你现在测试的是：

```python
route_after_critic()
```

假设：

```text
7/7 全部通过
```

说明：

> **这个函数在这些输入下是对的。**

但还不能推出：

> **线上 Graph 一定按这个函数运行。**

因为还存在：

```text
函数正确
    ↓
接线错误
```

比如真实构图：

```python
builder.add_conditional_edges(
    "critic",
    route_after_critic,
    {
        "reviser": "reviser",
        "end": END,
    },
)
```

如果这里写错了，单独的 `route_after_critic()` 即使 100% 正确，线上依然可能错误。

所以：

```text
函数层
```

和：

```text
接线层
```

不是一回事。

---

# 三十二、再看构图

代码：

```python
def build_graph():
    builder = StateGraph(RState)

    builder.add_node(
        "generator",
        generator,
    )

    builder.add_node(
        "critic",
        critic,
    )

    builder.add_node(
        "reviser",
        reviser,
    )

    builder.add_edge(
        START,
        "generator",
    )

    builder.add_edge(
        "generator",
        "critic",
    )

    builder.add_conditional_edges(
        "critic",
        route_after_critic,
        {
            "reviser": "reviser",
            "end": END,
        },
    )

    builder.add_edge(
        "reviser",
        "critic",
    )

    return builder.compile()
```

先看固定边：

```text
START
 ↓
generator
 ↓
critic
```

这部分没有选择。

---

# 三十三、然后是最核心的条件边

```python
builder.add_conditional_edges(
    "critic",
    route_after_critic,
    {
        "reviser": "reviser",
        "end": END,
    },
)
```

可以拆成：

```text
critic
   ↓
route_after_critic
   ↓
返回一个字符串
   ↓
查 path_map
   ↓
真正去哪个节点
```

也就是：

```text
critic
  ↓
route_after_critic(state)
  ↓
┌────────────┬─────────────┐
│ "reviser"  │ "end"       │
└─────┬──────┴──────┬──────┘
      ↓             ↓
  reviser          END
```

所以：

> **模型本身并没有直接调用 `reviser()`。**

模型产生的是数据：

```text
score
issues
...
```

程序再根据这些数据决定：

```text
reviser
```

还是：

```text
END
```

---

# 三十四、最后一条边：`reviser → critic`

代码：

```python
builder.add_edge(
    "reviser",
    "critic",
)
```

这条边就是整个循环真正闭合的地方。

没有它：

```text
generator
 ↓
critic
 ↓
reviser
 ↓
结束
```

那就只能修改一次。

有了它：

```text
generator
 ↓
critic
 ↓
reviser
 ↓
critic
 ↓
reviser
 ↓
critic
 ↓
...
```

再由：

```python
route_after_critic()
```

决定什么时候退出。

因此可以把：

```python
builder.add_edge(
    "reviser",
    "critic",
)
```

理解成：

> **反思循环的回边。**

---

# 三十五、把整个图重新画一次

```text
                 ┌──────────────┐
                 │  generator   │
                 │    生成初稿   │
                 └──────┬───────┘
                        ↓
                 ┌──────────────┐
                 │    critic    │
                 │ score        │
                 │ issues       │
                 │ fatal_issues │
                 └──────┬───────┘
                        ↓
              ┌─────────────────────┐
              │ route_after_critic  │
              └─────────┬───────────┘
                   ┌────┴────┐
                   │         │
                   ↓         ↓
                reviser     END
                   │
                   ↓
                 critic
                   │
                   └────────────→ ...
```

所以今天整个图真正的循环关系其实就是：

```text
critic
  ↓
条件边
  ↓
reviser
  ↓
critic
```

---

# 三十六、初始化 State

代码：

```python
INITIAL = {
    "topic": "为什么 Agent 需要记忆系统",

    "draft": "",
    "score": 0,
    "issues": [],
    "fatal_issues": [],
    "revise_count": 0,

    "trajectory": [],

    "best_draft": "",
    "best_score": 0,
}
```

这里提前把 State 的字段初始化好。

例如：

```text
draft = ""
```

意味着：

> 现在还没有初稿。

```text
score = 0
```

意味着：

> 现在还没有评审。

```text
trajectory = []
```

意味着：

> 现在还没有任何历史。

```text
revise_count = 0
```

意味着：

> 还没有进行修改。

这和 Day2 的 State 初始化思想一样：

> **节点运行时如果需要读取某个字段，就应该提前让这个字段有明确初始状态。**

---

# 三十七、完整运行：先启动 Graph

代码：

```python
graph = build_graph()
```

这里做的事情就是：

```text
State Schema
    +
节点
    +
边
    ↓
compile()
    ↓
可执行 Graph
```

所以 `build_graph()` 负责的是：

> **定义图的结构。**

而：

```python
graph.stream(...)
```

才是真正运行。

---

# 三十八、为什么这里同时拿 `updates` 和 `values`

代码：

```python
for mode, chunk in graph.stream(
    INITIAL,
    stream_mode=["updates", "values"],
):

    if mode == "updates":
        for node_name in chunk:
            chain.append(node_name)

    elif mode == "values":
        final_state = chunk
```

这里一次运行同时拿两类信息：

```text
updates
    ↓
节点更新了什么

values
    ↓
当前完整 State
```

今天真正需要两个东西：

```text
第一：
节点到底怎么走？

第二：
最后 State 是什么？
```

于是：

```text
updates
→
流转链

values
→
最终状态
```

这样避免为了拿不同信息而重新运行第二次。

---

# 三十九、为什么“同一次运行”很重要

LLM 是动态系统。

比如：

```text
第一次运行：
7 → 8 → 7

第二次运行：
7 → 8
```

这两个结果可能完全不同。

因此不能：

```text
第一次运行拿轨迹
+
第二次运行拿最终 State
```

然后假设：

```text
它们属于同一个实验
```

实际上不是。

所以现在：

```python
graph.stream(
    INITIAL,
    stream_mode=["updates", "values"],
)
```

一次运行同时拿：

```text
updates
+
values
```

这样：

> **所有观察数据都来自同一次实验。**

这是测试动态 Agent 时非常重要的一点。

---

# 四十、最后怎么输出分数轨迹

代码：

```python
traj = out["trajectory"]

for t in traj:
    print(
        f"第 {t['round']} 轮: "
        f"{t['score']} 分, "
        f"{t['draft_len']} 字"
    )
```

于是：

```text
第 0 轮：7 分，243 字
第 1 轮：8 分，400 字
第 2 轮：7 分，645 字
```

这些信息就能让人真正看到：

```text
系统到底发生了什么
```

而不是只知道：

```text
程序最后 END 了
```

这也是为什么：

```text
trajectory
```

不仅是业务数据，也是非常重要的：

> **调试数据。**

---

# 四十一、今天一个非常值得注意的现象：分数提高，不代表内容一定变好

例如：

```text
7 → 8
```

表面上看：

```text
变好了
```

但是实际可能是：

```text
critic 要求“增加数据”
        ↓
reviser 编了一个数据
        ↓
文本形式更具体
        ↓
评分提高
```

所以：

```text
score ↑
```

不能自动推出：

```text
真实质量 ↑
```

这就是今天最重要的发现之一：

> **Self-Refine 会优化评分标准，但评分标准如果不能覆盖真实质量，就可能被“钻空子”。**

---

# 四十二、这就是为什么需要硬门槛

把：

```text
score
```

和：

```text
fatal_issues
```

分开之后，逻辑变成：

```text
score = 9
fatal = []
    ↓
可以通过

score = 9
fatal = ["数据无来源"]
    ↓
不能通过
```

所以：

```text
score
```

是：

> **软质量指标**

而：

```text
fatal_issues
```

是：

> **硬约束**

两个不能互相替代。

---

# 四十三、但当前 `fatal_issues` 这条数据流要特别注意

原始 Day3 代码已经有：

```python
class Critique(BaseModel):
    ...
    fatal_issues: list[str]
```

路由也有：

```python
fatal = state.get("fatal_issues") or []
```

但如果 `RState` 没有：

```python
fatal_issues: list[str]
```

同时 `critic()` 也没有：

```python
"fatal_issues": c.fatal_issues
```

那么这条链就没有真正接通：

```text
Critique
  ↓
fatal_issues
  X
State
  ↓
route
```

所以这里一定要分清：

> **代码里写了这个名字，不等于这个字段真的参与了 State 流动。**

完整接通以后才应该是：

```text
Critique.fatal_issues
        ↓
critic()
        ↓
State.fatal_issues
        ↓
route_after_critic()
        ↓
控制流
```

这也是今天特别重要的一类工程问题：

> **定义 ≠ 数据流 ≠ 控制流。**

---

# 四十四、把今天的 Data-flow 单独画出来

```text
topic
  ↓
generator
  ↓
draft
  ↓
critic
  ↓
┌───────────────────────┐
│ score                 │
│ issues                │
│ fatal_issues          │
│ trajectory            │
│ best_score            │
│ best_draft            │
└───────────┬───────────┘
            ↓
          State
            ↓
         reviser
            ↓
        new draft
            ↓
          critic
```

这里主要关注：

> **数据产生以后去了哪里。**

---

# 四十五、再把 Control-flow 单独画出来

```text
critic
  ↓
route_after_critic
  │
  ├── count >= MAX_REVISE
  │          ↓
  │         END
  │
  ├── fatal
  │    ↓
  │  reviser
  │
  ├── score >= PASS_SCORE
  │          ↓
  │         END
  │
  ├── score 下降
  │          ↓
  │         END
  │
  └── 其他
       ↓
     reviser
```

所以：

```text
Data-flow
```

告诉我们：

> 发生了什么。

而：

```text
Control-flow
```

告诉我们：

> 接下来去哪。

这两个概念今天需要彻底分开。

---

# 四十六、今天最值得记住的整体模型

以前容易把 Self-Refine 想成：

```text
模型
 ↓
修改
 ↓
模型
 ↓
修改
```

现在应该理解成：

```text
                  State
                    │
                    ↓
                ┌───────┐
                │ critic│
                └───┬───┘
                    │
          产生结构化评审数据
                    │
                    ↓
             route_after_critic
                    │
         ┌──────────┼───────────┐
         ↓          ↓           ↓
       END       reviser      END
                    │
                    ↓
                  draft
                    │
                    └────→ critic
```

真正的核心不是：

```text
模型自己反思
```

而是：

> **模型产生结构化判断，State 保存判断，程序根据判断控制循环。**

---

# 四十七、最终可以把 Day3 压缩成四个概念

```text
① Critique
   → 结构化评价

② Revise
   → 根据评价重写

③ Route
   → 决定继续还是结束

④ Guard
   → 防止无限循环、质量回退和硬伤放行
```

于是：

```text
Critique
   ↓
State
   ↓
Route
   ↓
Revise / END
```

这就是今天整张图的骨架。

---

# 四十八、完整代码

下面只保留 Day3 的核心代码，省略 import 部分。

```python
# ══════════ 0. 配置 ══════════

MODEL_NAME = "deepseek-v4-flash"
BASE_URL = "https://api.deepseek.com"
API_KEY = os.getenv("DEEPSEEK_API_KEY")

MODE = sys.argv[1] if len(sys.argv) > 1 else "normal"
FORCED = MODE == "forced"

PASS_SCORE = 11 if FORCED else 8
MAX_REVISE = 3

STALL_STOP = (
    not FORCED
    and os.getenv("STALL", "1") != "0"
)

llm = ChatOpenAI(
    model=MODEL_NAME,
    base_url=BASE_URL,
    api_key=API_KEY,
    temperature=0,
)
```

这里主要是运行配置：

```text
MODE
PASS_SCORE
MAX_REVISE
STALL_STOP
```

其中：

```text
PASS_SCORE
```

决定多少分可以结束。

```text
MAX_REVISE
```

决定最多改几次。

```text
STALL_STOP
```

决定是否启用质量回退出口。

---

```python
# ══════════ 1. Critique + State ══════════

class Critique(BaseModel):
    score: int = Field(
        ge=1,
        le=10,
        description="1-10分，8分及以上达标",
    )

    issues: list[str] = Field(
        default_factory=list,
        description="软问题，影响分数",
    )

    fatal_issues: list[str] = Field(
        default_factory=list,
        description=(
            "硬伤：编造/无来源数据、"
            "事实错误、违规内容。"
            "只要非空，无论多少分都不达标"
        ),
    )

    passed: bool = Field(
        description="是否达标",
    )


class RState(MessagesState):
    topic: str
    draft: str
    score: int
    issues: list[str]
    fatal_issues: list[str]
    revise_count: int

    trajectory: Annotated[
        list,
        operator.add,
    ]

    best_draft: str
    best_score: int
```

这一块定义：

```text
模型评价的数据结构
+
Graph 运行的数据状态
```

两者不要混淆。

```text
Critique
    → 模型一次评审的结果

RState
    → 整个循环长期保存的状态
```

---

```python
# ══════════ 2. generator ══════════

def generator(state: RState) -> dict:

    r = llm.invoke(
        f"请写一篇约200字的中文短文，"
        f"主题：{state['topic']}。"
        f"只输出正文。"
    )

    draft = str(r.content)

    return {
        "draft": draft,
        "revise_count": 0,
        "best_draft": draft,
        "best_score": 0,
    }
```

这里：

```text
topic
 ↓
LLM
 ↓
draft
```

同时初始化：

```text
revise_count
best_draft
best_score
```

---

```python
# ══════════ 3. critic ══════════

JUDGE_SYS = """你是严格但公正的中文写作评审。

评分维度：
① 观点清晰度
② 论据支撑
③ 结构层次
④ 表达流畅

评分标准：
8分及以上为达标。

issues 最多3条，
每条不超过15字。

输出格式：
只输出一个 JSON 对象，形如：

{
  "score": 7,
  "issues": ["第二段缺少具体例子"],
  "fatal_issues": [],
  "passed": false
}
"""


def _parse_critique(raw) -> Critique:

    if not isinstance(raw, Critique):
        raw = Critique.model_validate(raw)

    if not (1 <= int(raw.score) <= 10):
        raise ValueError(
            f"score 越界:{raw.score}"
        )

    return raw
```

这一块：

```text
Prompt
+
Schema
+
程序校验
```

共同组成 critic 的输入输出契约。

---

```python
def critic(state: RState) -> dict:

    judge = llm.with_structured_output(
        Critique,
        method="json_mode",
    )

    prompt = (
        f"{JUDGE_SYS}\n\n"
        f"文稿:\n{state['draft']}"
    )

    c = None

    for attempt in range(3):
        try:
            c = _parse_critique(
                judge.invoke(prompt)
            )
            break
        except Exception as e:
            print(
                f"[warn] 第{attempt + 1}次评审解析失败"
            )

    if c is None:
        c = Critique(
            score=1,
            issues=[
                "评审解析失败，无法判断"
            ],
            fatal_issues=[],
            passed=False,
        )

    passed_by_code = (
        c.score >= PASS_SCORE
    )

    if passed_by_code != c.passed:
        print(
            "[warn] 模型 passed "
            "与程序阈值不一致"
        )

    score = int(c.score)

    best_score = state.get(
        "best_score",
        0,
    )

    best_draft = state.get(
        "best_draft",
        "",
    )

    if score > best_score:
        best_score = score
        best_draft = state["draft"]

    entry = {
        "round": len(
            state.get(
                "trajectory",
                [],
            )
        ),
        "score": score,
        "issues": c.issues,
        "fatal_issues": c.fatal_issues,
        "draft_len": len(
            state["draft"]
        ),
        "draft_head": (
            state["draft"][:60]
            .replace("\n", " ")
        ),
    }

    return {
        "score": score,
        "issues": c.issues,
        "fatal_issues": c.fatal_issues,
        "trajectory": [entry],
        "best_score": best_score,
        "best_draft": best_draft,
    }
```

这一块是今天最重要的节点。

它完整做：

```text
读取 draft
 ↓
调用 critic
 ↓
解析结构化结果
 ↓
业务校验
 ↓
程序重新计算 passed
 ↓
更新 best
 ↓
追加 trajectory
 ↓
写回 State
```

---

```python
# ══════════ 4. reviser ══════════

def reviser(state: RState) -> dict:

    r = llm.invoke(
        "请根据评审意见改进以下文稿。"
        "保持主题不变，只输出改进后的正文。\n\n"
        f"原稿:\n{state['draft']}\n\n"
        f"普通问题:\n{state['issues']}\n\n"
        f"硬伤:\n"
        f"{state.get('fatal_issues') or []}"
    )

    draft = str(r.content)

    n = (
        state.get(
            "revise_count",
            0,
        )
        + 1
    )

    return {
        "draft": draft,
        "revise_count": n,
    }
```

这里和 critic 完全相反：

```text
critic
    ↓
产生评价

reviser
    ↓
消费评价
    ↓
产生新稿
```

所以：

> **一个节点负责观察，一个节点负责修改。**

---

```python
# ══════════ 5. 条件边 ══════════

def route_after_critic(
    state: RState,
) -> Literal[
    "reviser",
    "end",
]:

    score = state["score"]

    count = state.get(
        "revise_count",
        0,
    )

    traj = state.get(
        "trajectory",
        [],
    )

    fatal = state.get(
        "fatal_issues"
    ) or []

    if count >= MAX_REVISE:
        return "end"

    if fatal:
        return "reviser"

    if score >= PASS_SCORE:
        return "end"

    if (
        STALL_STOP
        and len(traj) >= 2
        and traj[-1]["score"]
        < traj[-2]["score"]
    ):
        return "end"

    return "reviser"
```

这是控制流核心。

注意这里没有：

```python
llm.invoke(...)
```

所以它本质上是：

> **纯函数控制器。**

这也是为什么它非常适合单元测试。

---

```python
# ══════════ 6. 构图 ══════════

def build_graph():

    builder = StateGraph(RState)

    builder.add_node(
        "generator",
        generator,
    )

    builder.add_node(
        "critic",
        critic,
    )

    builder.add_node(
        "reviser",
        reviser,
    )

    builder.add_edge(
        START,
        "generator",
    )

    builder.add_edge(
        "generator",
        "critic",
    )

    builder.add_conditional_edges(
        "critic",
        route_after_critic,
        {
            "reviser": "reviser",
            "end": END,
        },
    )

    builder.add_edge(
        "reviser",
        "critic",
    )

    return builder.compile()
```

这部分就是把前面的代码真正组成 Graph：

```text
START
 ↓
generator
 ↓
critic
 ↓
route
 ├── END
 └── reviser
        ↓
      critic
```

---

```python
# ══════════ 7. INITIAL ══════════

INITIAL = {
    "topic": "为什么 Agent 需要记忆系统",
    "draft": "",
    "score": 0,
    "issues": [],
    "fatal_issues": [],
    "revise_count": 0,
    "trajectory": [],
    "best_draft": "",
    "best_score": 0,
}
```

这是图真正运行前的初始 State。

整个循环中的所有数据，最后都是围绕这里不断更新。

---

```python
# ══════════ 8. 运行 ══════════

graph = build_graph()

chain = []
final_state = None

for mode, chunk in graph.stream(
    INITIAL,
    stream_mode=[
        "updates",
        "values",
    ],
):

    if mode == "updates":

        for node_name in chunk:
            chain.append(node_name)

    elif mode == "values":

        final_state = chunk
```

这里的重点不是语法，而是：

```text
updates
    → 看图怎么走

values
    → 看 State 变成什么
```

一次运行同时观察：

```text
Control-flow
+
State
```

这样才可以把：

```text
程序到底走了哪？
```

和：

```text
State 最后变成了什么？
```

对应起来。

---

# 总结

今天的图表面上只是：

```text
generator
→ critic
→ reviser
→ critic
```

但真正需要掌握的是里面的四层关系：

```text
第一层：Critique
    LLM 负责评价

第二层：State
    保存评价结果和历史

第三层：Route
    程序根据 State 决定下一步

第四层：Guard
    防止无限循环、质量回退和硬伤放行
```

最终就变成：

```text
                  State
                    │
                    ↓
                ┌───────┐
                │ critic│
                └───┬───┘
                    │
              结构化评价结果
                    │
                    ↓
             route_after_critic
                    │
             ┌──────┴──────┐
             ↓             ↓
          reviser         END
             │
             ↓
           critic
             │
             └──────────→ ...
```

所以今天真正理解的不是：

> **“Self-Refine 就是让模型自己修改。”**

而是：

> **“LLM 负责产生评价，State 负责保存评价，程序负责决定控制流，停止条件负责限制自主循环。”**

而今天最需要记住的一个工程规律是：

> **模型输出可以参与决策，但最终的控制权应该落在程序定义的边界上。**
