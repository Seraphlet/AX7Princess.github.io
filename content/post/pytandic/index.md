---
description: ""
title: "Pydantic 模型定义、校验与序列化"
draft: false
date: "2026-09-06T13:21:43+08:00"
slug: "Pytandic"
categories:
 - 
tags:
 - 
image: ""
---



# Pydantic 模型定义、校验与序列化（v2 详解）

> **关键词**：`BaseModel` / 宽松·严格模式 / `Field` / `field_validator` / `model_validator` / `ConfigDict` / `model_dump` / `TypeAdapter` / `json_schema`
> **适用**：任何想让"外部进来的数据"在入口处被检查、被清洗、被安全使用的场景——LLM 输出解析、配置文件、API 请求、LangGraph State
> **版本**：本文全部基于 **Pydantic v2**（当前主流；v1 语法如 `parse_obj` / `@validator` 已弃用/移除）

* * *

## 一、Pydantic 是什么：一句话 + 一个痛点

**Pydantic = 给 Python 数据上"安检"的库。** 你声明"数据应该长什么样"（字段名 + 类型 + 约束），它负责三件事：

```fallback
  ① 校验   进来的数据不符合声明 → 报错（ValidationError）
  ② 转换   能安全转的自动转（"20" → 20、123 → "123"）
  ③ 序列化 模型 ↔ dict ↔ JSON ↔ 模型，随时互转
```

**没有它时你写的是这种代码**（每个入口都要自己手写防御）：

```python
def register(user_data: dict):
    # 手写防御三连：判存在 → 判类型 → 判值域，然后一个个手转
    if "age" not in user_data:
        raise ValueError("缺 age")
    age = user_data["age"]
    if not isinstance(age, int):
        try:
            age = int(age)          # "20" 能转，但 "abc" 呢？还得再包 try
        except ValueError:
            raise ValueError("age 必须是数字")
    if age < 0 or age > 150:
        raise ValueError("age 范围不对")
    name = str(user_data.get("name", "")).strip()
    ...
```

**有它之后**：声明一份模型，上面全部防御 + 转换交给框架：

```python
from pydantic import BaseModel, Field

class UserInput(BaseModel):
    name: str = Field(min_length=1, max_length=50)
    age: int = Field(ge=0, le=150)
    email: str | None = None          # 可空

# 一行过安检：
u = UserInput.model_validate({"name": " Alice ", "age": "20"})
```

**核心理念**：**在数据入口花 5 分钟声明规则，省掉后面 50 处"字段可能为空 / 类型不对"的防御代码。** 你的 Agent 从模型输出拿 JSON、从配置读参数、收 API 请求时，先过一道安检，后面全程类型安全。

**安装**：

```bash
pip install "pydantic[email]"   # email 可选：EmailStr 需要 email-validator
```

> ⚠️ v2 与 v1 的关键差异（从旧教程迁移时最常踩）：
> | 旧写法（v1） | 新写法（v2） |
> |---|---|
> | `parse_obj()` / `parse_raw()` | `model_validate()` / `model_validate_json()` |
> | `.dict()` / `.json()` | `.model_dump()` / `.model_dump_json()` |
> | `@validator` | `@field_validator`（语义有变，见第七章） |
> | `__fields__` | `model_fields` |
> v2 底层用 Rust 写的 pydantic-core，**校验速度比 v1 快 5~50 倍**。

* * *

## 二、BaseModel 入门

### 2.1 定义一个模型

```python
from pydantic import BaseModel

class Student(BaseModel):
    name: str            # 字段 = 类属性 + 类型注解
    age: int
    score: float = 0.0   # 带默认值 = 可选字段

# 实例化：位置参数 / 关键字 / dict 都行
s = Student("Alice", 20, 85.5)                 # 位置参数
s = Student(name="Bob", age=21)                # 关键字（score 用默认 0.0）
s = Student.model_validate({"name": "Cara", "age": 19, "score": 90})   # 从 dict 过安检

# 访问：就是普通属性
s.name     # 'Cara'
s.age      # 19
```

**三个细节**：

- **类型注解即规则**：`name: str` 不只是给 IDE 看的——它在运行时真的会检查。
- **实例化就有校验**：`Student(name=123)` 会怎样？`123` 会被**宽松转换**成 `"123"` 存进去（str 字段吃下大多数值）；但 `Student(age="abc")` 会抛 `ValidationError`。
- **每个字段一个必填/可选属性**：无默认值 = 必填，漏了报错；有默认值 = 可选。

### 2.2 查看模型声明：`model_fields`

想知道"这个模型有哪些字段、各自什么规则"，不用翻源码：

```python
for name, field in Student.model_fields.items():
    print(name, "→", field.annotation, "| 默认:", field.default)
# name → <class 'str'> | 默认: PydanticUndefined   （undefined = 必填）
# age → <class 'int'>  | 默认: PydanticUndefined
# score → <class 'float'> | 默认: 0.0
```

> 💡 调试"为什么这个字段没过校验"时，先看 `model_fields` 确认声明本身没问题。

### 2.3 repr 与比较

```python
s = Student(name="Alice", age=20)
s                    # Student(name='Alice', age=20, score=0.0)  清晰 repr
s == Student(name="Alice", age=20)   # True  ← 模型自动实现值相等（不是身份相等）
```

> **应用**：测试里断言 `result == Student(name=...)` 直接过，不用逐个字段比。

* * *

## 三、校验的核心机制：宽松 / 严格模式

### 3.1 宽松模式（默认）：能转就转

Pydantic 默认**宽松（lax）**：输入类型和声明不完全一致时，先尝试语义无损的转换。

| 字段类型 | 传什么 | 存成什么 | 规则 |
|---|---|---|---|
| `int` | `"20"` | `20` | 纯数字字符串可转 |
| `float` | `"85.5"` / `85` | `85.5` | 数字字符串 / int 都可转 |
| `str` | `123` | `"123"` | 多数标量会转字符串 |
| `bool` | `"true"` / `1` / `"yes"` | `True` | 常见真值写法都认 |
| `datetime` | `"2026-09-06T20:00:00"` | `datetime(...)` | ISO 字符串自动解析 |

```python
class Demo(BaseModel):
    a: int
    b: float
    c: str
    d: bool

d = Demo(a="42", b="3.14", c=99, d="yes")
d.model_dump()
# {'a': 42, 'b': 3.14, 'c': '99', 'd': True}
```

### 3.2 什么时候宽松会失败

宽松不是万能——**转不了就报错**，并且**能列出所有错**：

```python
try:
    Demo(a="abc", b="x", c=None, d="maybe")
except ValidationError as e:
    print(e)
```

```fallback
4 validation errors for Demo
a
  Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='abc', input_type=str]
b
  Input should be a valid number, unable to parse string as a number [type=float_parsing, ...]
c
  Input should be a valid string [type=string_type, input_value=None, input_type=NoneType]
d
  Input should be a valid boolean, unable to interpret input [type=bool_parsing, ...]
```

**每条错误的结构**（结构化，可用于程序处理）：

```python
except ValidationError as e:
    for err in e.errors():            # 每条的完整结构
        print(err["loc"],    # ('a',)         ← 哪个字段（嵌套时是元组路径）
              err["type"],   # 'int_parsing'  ← 错误类型码
              err["msg"],    # 人类可读信息
              err["input"])  # 原始输入
```

> 💡 `loc` 在嵌套模型里是**路径元组**，如 `('items', 1, 'price')` = "第 2 个商品的 price 字段"。错误定位到树叶，前端表单或日志都能直接用。

### 3.3 严格模式：只认声明的类型

有时候你要的就是**严格**：`"20"` 是字符串，不是整数，别给我转。

三种开启方式（作用范围从大到小）：

```python
from pydantic import StrictInt, ConfigDict

# 方式① 全局：整个模型严格
class S1(BaseModel):
    model_config = ConfigDict(strict=True)
    age: int

# 方式② 单字段严格：用 Strict* 类型
from typing import Annotated
class S2(BaseModel):
    age: StrictInt            # 只这个字段严格

# 方式③ 调用时严格：单次校验覆盖
Student.model_validate({"name": "A", "age": 20}, strict=True)
```

```python
S2(age="20")     # ❌ ValidationError：strict 模式下 str 不再转 int
S2(age=20)       # ✅
```

| 模式 | 行为 | 用在哪 |
|---|---|---|
| 宽松（默认） | 能安全转就转 | 外部输入（JSON、表单、LLM 输出）——**默认用它** |
| 严格 | 类型必须精确匹配 | 内部契约、不允许隐式转换的地方 |

> 🎯 **一条工程经验**：**对外（宽松）进，对内（严格）用**。外部数据（API/JSON）宽容一点，转完存进内部模型后再处理的就是干净类型了。如果你全程严格，前端多传一个字符串年龄就直接 400，很烦。

* * *

## 四、字段类型全览

### 4.1 标准类型与容器

```python
from pydantic import BaseModel
from typing import Optional, Union, Literal  # 或直接用 |（3.10+）

class Everything(BaseModel):
    id: int
    ratio: float
    name: str
    flag: bool
    tags: list[str]                      # 元素也校验：list 里每个都必须是 str
    attrs: dict[str, int]                # 键 str、值 int
    coords: tuple[float, float]          # 长度、元素类型都校验
    unique_ids: set[int]                 # 自动去重 + 元素校验
    anything: Any                        # 不校验，原样存
```

```python
Everything(
    id=1, ratio=0.5, name="x", flag=True,
    tags=["a", "b"], attrs={"k": 1},
    coords=(1.0, 2.0), unique_ids=[1, 1, 2],   # set 自动去重 → {1, 2}
    anything={"随便": [1, 2, "什么"]},
)
```

> ⚠️ `list[Model]` / `dict[str, Model]` 同样**逐元素递归校验**——容器里装模型，每个都过安检（见第八章）。

### 4.2 Optional / Union / None

```python
class Opt(BaseModel):
    maybe: int | None = None      # 可空：int 或 None
    a_or_b: int | str             # 联合类型：是 int 就给 int，否则试 str
    kind: Literal["dev", "prod", "test"] = "dev"   # 字面量枚举
```

- `int | None`（= `Optional[int]`）：值可以是 int 也可以是 `None`。**注意**：给 `None` 时不再尝试转 int——`None` 有自己明确的语义。
- `int | str`：Pydantic 按顺序尝试，int 能转就用 int。
- `Literal["dev", "prod"]`：只接受列出的值——**比枚举(Enum)更轻量的白名单**。

### 4.3 特殊类型（按需）

```python
from pydantic import EmailStr, HttpUrl, UUID4
from datetime import datetime, date
from decimal import Decimal

class Profile(BaseModel):
    email: EmailStr            # ✅ 语法校验（需要 pip install email-validator）
    url: HttpUrl               # ✅ 必须是合法 http(s) URL
    uid: UUID4                 # ✅ 必须是 UUID v4
    created: datetime          # ✅ 自动解析 ISO 字符串
    day: date
    amount: Decimal            # ✅ 金额用 Decimal，别用 float（精度）
```

```python
Profile(email="not-an-email", url="ftp://x", uid="abc", created="2026-09-06T20:00:00")
# ❌ email / url / uid 三条都报
```

| 类型 | 校验什么 | 备注 |
|---|---|---|
| `EmailStr` | 邮箱语法 | 需要 `email-validator` |
| `HttpUrl` | 合法 URL | 自动校验协议 |
| `UUID` / `UUID4` | UUID 格式 | |
| `datetime` / `date` / `time` | 时间 | 自动解析 ISO 字符串 |
| `Decimal` | 高精度小数 | 算钱别用 float |
| `IPvAnyAddress` | IP | 需看子模块 |

> ⚠️ 特殊类型的校验是**运行时依赖格式库**的（EmailStr 要 email-validator），没装会在导入时报错——所以安装用 `pydantic[email]`。

* * *

## 五、必填 / 可选 / 默认值（进阶）

### 5.1 默认值三种写法

```python
class Conf(BaseModel):
    env: str = "dev"                       # ① 字面量默认
    tags: list[str] = Field(default_factory=list)      # ② 工厂默认（可变对象必用）
    extra: dict[str, Any] = Field(default_factory=dict)
    created_at: datetime = Field(default_factory=datetime.now)   # 每次实例化取当前时间
```

> ⚠️ **可变默认值必须用 `default_factory`**：直接写 `tags: list[str] = []` 会直接抛 `ValidationError`（"field default is not allowed to be mutable"）。原因：共享同一个 list 会让一个实例的修改污染所有实例。`default_factory=list` 每次实例化**新建**一个空列表。

### 5.2 缺省（None）与"没传"的区别

```python
class Req(BaseModel):
    a: int = Field(default_factory=lambda: 0)     # 缺省时生成 0
    b: int | None = None                          # 缺省时是 None
```

```python
r = Req()          # a=0, b=None（都"没传"但拿到了值）
r2 = Req(a=5)      # a=5
r.model_dump(exclude_unset=True)   # {}  ← 只导出"显式传过"的字段！
```

> 💡 `exclude_unset=True` 是 **PATCH 类接口的利器**：客户端只传要改的字段，你只更新传过的字段，没传的不动。默认值和显式传值在"是否需要序列化"上有本质区别。

### 5.3 缺省校验

```python
class C(BaseModel):
    code: str = Field(default_factory=lambda: gen_code())

# 默认值要不要也过校验？默认不校验（省性能）。想校验：
class C2(BaseModel):
    model_config = ConfigDict(validate_default=True)   # 默认值也过安检
```

* * *

## 六、Field：全部常用参数

`Field(...)` 是给单个字段声明规则的集中地。完整常用清单：

```python
from pydantic import BaseModel, Field

class Product(BaseModel):
    # —— 标识 ——
    id: int = Field(description="数据库主键")            # 描述（进 JSON Schema）
    code: str = Field(title="货号", alias="product_code")  # title + 别名

    # —— 值域 ——
    price: float = Field(ge=0, le=100000)              # ge≥ le≤（gt/lt 不含等号）
    discount: float = Field(ge=0, multiple_of=0.1)     # multiple_of：必须是 0.1 的倍数
    name: str = Field(min_length=2, max_length=50)     # 长度
    sn: str = Field(pattern=r"^[A-Z]{2}-\d{4}$")       # 正则

    # —— 行为 ——
    stock: int = Field(default=0)                      # 默认值
    tags: list[str] = Field(default_factory=list)      # 工厂默认
    secret: str = Field(repr=False)                    # repr 里隐藏（打印模型时不显示密码）
    frozen_flag: bool = Field(frozen=True)             # 该字段赋值后不可改
```

**用法演示**：

```python
Product(id=1, product_code="AB-1234", price=99.5)     # ✅ 用别名 product_code 传入
Product(id=1, product_code="AB-1234", price=-1)       # ❌ price < 0
Product(id=1, product_code="XX-1234", sn="AB-1234" )  # ❌ 如果 sn 必填却没给
```

**记住这张表**：

| 参数 | 管什么 | 例子 |
|---|---|---|
| `default` / `default_factory` | 默认值 | 见第五章 |
| `alias` | 输入/输出别名 | 见第十一章 |
| `title` / `description` | 元信息（生成 JSON Schema） | **LLM 工具绑定会读它** |
| `gt` `ge` `lt` `le` | 数字范围 | `ge=0` 非负 |
| `multiple_of` | 倍数 | `multiple_of=0.1` |
| `min_length` / `max_length` | 长度（str/list） | `max_length=5000` |
| `pattern` | 正则 | `pattern=r"^\d{11}$"` |
| `frozen` | 只读 | 赋值即报错 |
| `repr` | 是否进 repr | 密码字段 `repr=False` |
| `exclude` | 序列化时排除 | 见第十章 |

> 🎯 **对 Agent 开发者最重要的一行**：`description` 不只是注释——**它是给 LLM 看的**。LangChain 的 `bind_tools` 生成工具 schema 时，`Field(description=...)` 会变成工具参数的描述。**写 Pydantic 模型 = 在给模型写说明书**（回指 W4-D5 的 docstring 精神）。

* * *

## 七、校验器：自定义规则（field_validator / model_validator）

类型和 Field 覆盖不了逻辑时，写函数。

### 7.1 `field_validator`：单字段

```python
from pydantic import field_validator

class User(BaseModel):
    username: str
    password: str
    email: str

    @field_validator("username")           # 指定要管的字段
    @classmethod
    def check_username(cls, v: str) -> str:      # v = 该字段的输入值
        v = v.strip()                            # 清洗：去空格
        if len(v) < 3:
            raise ValueError("用户名至少 3 个字符")   # 抛 ValueError → 校验失败
        return v                                 # 返回值会覆盖原输入（存进模型）
```

```python
User(username="  ab ", password="x", email="a@b.c")   # ❌ strip 后 "ab" < 3
User(username="  alice ", password="x", email="a@b.c").username   # ✅ → "alice"
```

**三个要点**：

1. `@field_validator` 加 `@classmethod`，签名是 `(cls, v)`——v 是该字段的输入。
2. **抛 `ValueError` = 校验失败**；返回的值 = 该字段最终存进去的值（所以能用来清洗）。
3. 想校验多个字段可以传列表：`@field_validator("a", "b")`。

**before / after 模式**：默认 after（转换后校验）。想**在类型转换前**看原始输入（比如把 `"未知"` 转成 None）：

```python
class N(BaseModel):
    age: int | None = None

    @field_validator("age", mode="before")
    @classmethod
    def blank_to_none(cls, v):
        if isinstance(v, str) and v.strip() in ("", "未知"):
            return None            # 转换前拦截：空串 → None
        return v
```

### 7.2 `model_validator`：跨字段

单字段管不了"A 和 B 要一致"这种关系，用模型级校验器：

```python
from pydantic import model_validator

class Order(BaseModel):
    coupon: str | None = None
    total: float
    discount: float = 0

    @model_validator(mode="after")                 # after：所有字段都校验完后跑
    def check_discount(self) -> "Order":
        """跨字段规则：有券才有折扣，且折扣不能超总额"""
        if self.discount > 0 and not self.coupon:
            raise ValueError("有折扣必须提供优惠券")
        if self.discount > self.total:
            raise ValueError("折扣不能超过总额")
        return self                                  # after 模式返回整个 self
```

```python
Order(total=100, discount=10)                 # ❌ 有折扣没券
Order(coupon="SAVE10", total=100, discount=10)  # ✅
```

> ⚠️ `mode="after"` 的 `model_validator` 返回 `self`（或返回 dict 的 `mode="before"` 版）。**after 是最常用的**——此时所有字段已校验完、类型已转换，你可以放心用 `self.xxx` 跨字段比较。

### 7.3 校验执行顺序

```fallback
  原始输入
    │
    ▼
  field_validator(mode="before")     ← 每个字段：转换前的自定义拦截
    │
    ▼
  类型转换 + Field 约束               ← 框架内建（str→int、ge=0、pattern…）
    │
    ▼
  field_validator(mode="after")      ← 每个字段：转换后的自定义检查
    │
    ▼
  model_validator(mode="after")      ← 模型级：跨字段规则（最后）
```

> 💡 调试顺序问题就按这张流水线走：**想拦"原始脏输入"用 before，想在"转完类型后"补充规则用 after**。别在 after 里假设能看到原始字符串——那时已经是转换后的类型了。

* * *

## 八、嵌套模型与复杂结构

### 8.1 模型套模型：安检自动递归

```python
class Address(BaseModel):
    city: str
    zipcode: str = Field(pattern=r"^\d{6}$")

class User(BaseModel):
    name: str
    address: Address                       # 字段类型是另一个模型
    tags: list[str] = Field(default_factory=list)

u = User(
    name="Alice",
    address={"city": "北京", "zipcode": "100000"},   # dict 自动转 Address
    tags=["vip"],
)
u.address.city      # 链式访问：u.address.city → '北京'
```

**嵌套错误定位**：

```python
try:
    User(name="Bob", address={"city": "上海"})    # 内层缺 zipcode
except ValidationError as e:
    print(e.errors()[0]["loc"])      # ('address', 'zipcode')  ← 路径元组
```

### 8.2 容器装模型

```python
class Item(BaseModel):
    sku: str
    qty: int = Field(ge=1)

class Cart(BaseModel):
    user_id: int
    items: list[Item] = Field(default_factory=list)      # 每个元素都过 Item 安检

cart = Cart.model_validate({
    "user_id": 1,
    "items": [
        {"sku": "A1", "qty": 2},
        {"sku": "A2", "qty": 0},       # ❌ 第二个商品 qty < 1（ge 约束）
    ],
})
# 报错：items[1].qty ... Input should be greater than or equal to 1
# 错误路径 ('items', 1, 'qty') 精确定位到"第 2 个商品的 qty"
```

> 💡 错误路径 `('items', 1, 'qty')` 直接告诉你：**第 2 个商品的 qty 有问题**。LLM 返回一个含 N 个商品的结构时，这种定位能力无价。

### 8.3 自引用模型（树 / 链表）

```python
class TreeNode(BaseModel):
    value: int
    children: list["TreeNode"] = Field(default_factory=list)   # 引用自身（前向引用）

# 类体内引用自身，v2 需要构建后重建（或在定义后用）
TreeNode.model_rebuild()     # ← 前向引用解析
```

```python
tree = TreeNode(value=1, children=[TreeNode(value=2)])
tree.model_dump()
# {'value': 1, 'children': [{'value': 2, 'children': []}]}
```

> 💡 写前向引用时用**字符串注解** `"TreeNode"` + 类定义后调一次 `model_rebuild()`。若用了 `from __future__ import annotations`，很多情况下自动处理。

* * *

## 九、ConfigDict：模型的全局设置

`Field` 管单字段，`ConfigDict` 管整张模型：

```python
from pydantic import ConfigDict

class Settings(BaseModel):
    model_config = ConfigDict(
        # —— 输入侧 ——
        str_strip_whitespace=True,      # 所有 str 字段自动去首尾空格
        str_to_lower=False,             # 可开：str 自动转小写
        extra="forbid",                 # "ignore" 默认忽略 / "forbid" 多传报错 / "allow" 收进 __pydantic_extra__
        populate_by_name=True,          # 允许同时用别名和字段原名传入
        validate_assignment=True,       # 赋值时也校验（默认只在构造时校验）
        validate_default=True,          # 默认值也过校验

        # —— 输出侧 ——
        frozen=True,                    # 模型整体只读（类似 @dataclass(frozen=True)）
        arbitrary_types_allowed=True,   # 允许字段类型是任意普通类（非 pydantic 模型）

        # —— 行为 ——
        strict=False,                   # 默认宽松
        extra="forbid",
    )
    api_key: str = Field(min_length=10)
    env: str = "dev"
```

**extra 三种策略**（最常见也最容易懵的配置）：

```python
# extra="ignore"（默认）：多传的字段悄悄丢掉
# extra="forbid"：多传直接报错 —— 配置/契约类用它，写错名字立刻暴露
# extra="allow"：多传的收进 obj.__pydantic_extra__（dict），不丢

class A(BaseModel):
    model_config = ConfigDict(extra="forbid")
    x: int

A(x=1, y=2)     # ❌ Extra inputs are not permitted
```

**validate_assignment 演示**：

```python
class B(BaseModel):
    model_config = ConfigDict(validate_assignment=True)
    age: int = Field(ge=0)

b = B(age=20)
b.age = -5        # ❌ 赋值也会被拦（默认配置下赋值不校验，只有构造时校验）
```

**frozen 演示**：

```python
class C(BaseModel):
    model_config = ConfigDict(frozen=True)
    v: int

c = C(v=1)
c.v = 2           # ❌ ValidationError: Instance is frozen
```

> ⚠️ `arbitrary_types_allowed=True` 是你在 LangGraph/LangChain 里最可能遇到的：**BaseRetriever / State 里塞普通类对象（如你的 MemoryManager）时，必须开它**——否则 Pydantic 不知道该怎么给这个字段建 schema。

* * *

## 十、序列化：模型 ↔ dict ↔ JSON

### 10.1 四件套

```python
from datetime import datetime

class Event(BaseModel):
    name: str
    when: datetime

e = Event(name="发布", when="2026-09-06T20:00:00")

e.model_dump()                  # ① dict（Python 原生类型）
# {'name': '发布', 'when': datetime.datetime(2026, 9, 6, 20, 0)}

e.model_dump(mode="json")       # ② dict，但值已转成"可 JSON 化"的
# {'name': '发布', 'when': '2026-09-06T20:00:00'}   ← datetime → str

e.model_dump_json()             # ③ JSON 字符串
# '{"name":"发布","when":"2026-09-06T20:00:00"}'

Event.model_validate(e.model_dump())         # ④ 再过一遍安检（round-trip）
```

| 方法 | 用途 | 关键参数 |
|---|---|---|
| `model_dump()` | → dict（Python 值） | `mode="json"` 值转成 JSON 化类型 |
| `model_dump_json()` | → JSON 字符串 | |
| `model_validate(obj)` | dict / 模型 → 模型 | 输入可以是 dict 或另一个同模型实例 |
| `model_validate_json(s)` | JSON 字符串 → 模型 | 一步到位 |
| `model_copy(update=...)` | 复制并改字段（不修改原对象） | 见 10.3 |

### 10.2 序列化过滤参数

```python
u = User(name="Alice", age=20, email=None, tags=["vip"])

u.model_dump(exclude={"tags"})                # 排除指定字段
u.model_dump(exclude_none=True)               # 跳过值为 None 的字段
u.model_dump(exclude_unset=True)              # 只导出"显式传过"的字段（PATCH 用）
u.model_dump(include={"name", "age"})         # 白名单
u.model_dump(by_alias=True)                   # 用别名当 key（见第十一章）
```

### 10.3 `model_copy`：不可变更新

```python
e2 = e.model_copy(update={"name": "更新版"})    # 返回新对象，e 原样不动
e.name      # '发布'（没变）
e2.name     # '更新版'
```

> 🎯 **这条对 LangGraph 特别重要**：LangChain/LangGraph 的对象普遍走"复制更新"风格（和 `bind_tools` 返回新对象、`Runnable` 不可变是同一个设计哲学）——**想改一个字段就 `model_copy(update=...)`，别原地赋值**，这在多节点共享 state 时避免"你改我坏"。

* * *

## 十一、别名（Alias）：字段名和外部名不一样

数据库字段叫 `user_name`，前端传 `userName`，JSON 给 `username`——**外部命名和 Python 命名冲突时用别名**：

```python
from pydantic import Field

class ApiUser(BaseModel):
    model_config = ConfigDict(populate_by_name=True)   # 允许原名也能传入
    user_name: str = Field(alias="userName")
    api_key: str = Field(alias="apiKey")

# 输入走别名：
u = ApiUser.model_validate({"userName": "alice", "apiKey": "sk-xxx"})
u.user_name        # 'alice'（内部还是 Python 命名）

# 输出走别名：
u.model_dump(by_alias=True)
# {'userName': 'alice', 'apiKey': 'sk-xxx'}
```

**别名相关配置**：

| 配置/参数 | 作用 |
|---|---|
| `Field(alias="xxx")` | 校验时读 xxx，序列化默认仍用原名 |
| `ConfigDict(populate_by_name=True)` | 原名也能传入（双通道） |
| `model_dump(by_alias=True)` | 输出用别名做 key |
| `validation_alias` / `serialization_alias` | 输入/输出别名分开设 |

> 💡 典型场景：**给 LLM 的 tool schema 用别名**（模型输出 `userName`，内部字段 `user_name`），你的 Python 代码全程舒服命名，只在边界处转换。

* * *

## 十二、TypeAdapter：不建模型也能校验

有时候你只是要校验**一个值或一个简单结构**，为它建整个 `BaseModel` 小题大做——用 `TypeAdapter`：

```python
from pydantic import TypeAdapter

# 校验单值
ta = TypeAdapter(int)
ta.validate_python("42")       # → 42（宽松转换）
# ta.validate_python("abc")    # → ValidationError

# 校验容器结构
items_ta = TypeAdapter(list[str])
items_ta.validate_python(["a", "b"])     # ✅
# items_ta.validate_python(["a", 1])     # ❌ 1 不会转成 str？会——lax 下 1 → "1"
# 想严格：TypeAdapter(list[StrictStr]) 之类

# JSON 输入一步到位
TypeAdapter(list[dict[str, int]]).validate_json('[{"a": 1}]')
```

> 💡 在 Agent 里常见的用法：**校验 LLM 输出的轻量结构**（比如"给我返回 3 个关键词的 JSON 数组"），用 `TypeAdapter(list[str])` 就够了，不必每次建模型。TypeAdapter 和模型共享同一套校验引擎。

* * *

## 十三、与 JSON Schema / LLM 工具的联动

### 13.1 模型 → JSON Schema

Pydantic 模型**天生能生成 JSON Schema**（描述数据结构的标准格式）：

```python
Movie.model_json_schema()
# {'properties': {'title': {'title': 'Title', 'type': 'string'}, ...}, 'required': ['title'], 'type': 'object'}
```

**这就是你的 Agent 栈里 schema 的最终来源**：LangChain 的 `@tool` 参数、`bind_tools` 的工具描述、`PydanticOutputParser` 的格式要求——底层都是"Pydantic 模型 → JSON Schema → 塞给模型"。

```python
# 结构化输出（W4 见过，这里看全链路）：
from langchain_core.output_parsers import PydanticOutputParser
from langchain_core.prompts import ChatPromptTemplate

class Movie(BaseModel):
    title: str = Field(description="电影标题")
    rating: float = Field(description="评分（0-10）", ge=0, le=10)
    genres: list[str] = Field(description="类型标签，2-4 个")

parser = PydanticOutputParser(pydantic_object=Movie)

prompt = ChatPromptTemplate.from_messages([
    ("system", "根据描述提取信息。只输出 JSON。\n{format_instructions}"),
    ("user", "{input}"),
])
chain = prompt | llm | parser      # 模型输出 → JSON → 校验 → Movie 对象
# 输出格式不对时 parser 会抛 ValidationError —— 你能看见并重试
```

### 13.2 用模型直接当工具参数

```python
# 定义一个"参数模型"，交给 @tool：
from langchain_core.tools import tool

class WeatherArgs(BaseModel):
    city: str = Field(description="城市名，如 北京")
    days: int = Field(default=1, ge=1, le=7, description="预报天数")

@tool(args_schema=WeatherArgs)          # 模型自动帮你生成参数 schema
def get_weather(city: str, days: int = 1) -> str:
    """查询城市天气"""
    return f"{city} 未来{days}天天气……"
```

> 🎯 **这是 Pydantic 对 Agent 开发者最大的隐藏红利**：`args_schema=SomeModel` 时，**类型注解、description、ge/le 约束全部自动变成工具 schema 的一部分**——模型看到的参数说明，就是你 Field 里写的 description。**你在写模型，同时也在给模型写说明书。**

* * *

## 十四、综合实战：完整链路

把本篇知识串成一个真实场景：**LLM 返回一段含订单的 JSON → 校验 + 清洗 → 算价格 → 序列化输出**。

```python
from datetime import date
from decimal import Decimal
from pydantic import BaseModel, Field, ConfigDict, field_validator, model_validator

# ① 定义模型（含别名、清洗、跨字段规则）
class Item(BaseModel):
    model_config = ConfigDict(str_strip_whitespace=True)

    name: str = Field(min_length=1, max_length=50)
    price: Decimal = Field(ge=0, description="单价")
    qty: int = Field(ge=1)

class OrderInput(BaseModel):
    model_config = ConfigDict(extra="forbid")          # 不收未声明字段

    customer: str = Field(alias="customerName")        # 外部叫 customerName
    ship_date: date | None = None
    items: list[Item]

    @field_validator("customer", mode="before")
    @classmethod
    def strip_customer(cls, v):
        return v.strip() if isinstance(v, str) else v

    @model_validator(mode="after")
    def ensure_not_empty(self) -> "OrderInput":
        if not self.items:
            raise ValueError("订单至少要有一个商品")
        return self

    def total(self) -> Decimal:                        # 模型上可以带方法
        return sum((i.price * i.qty for i in self.items), Decimal("0"))

# ② LLM/接口返回的"脏 JSON" → 一行过安检
raw = {
    "customerName": "  Alice  ",
    "shipDate": "2026-09-10",
    "items": [
        {"name": "键盘", "price": "199.5", "qty": "2"},    # 价格/数量是字符串！
        {"name": "鼠标", "price": 59, "qty": 1},
    ],
}

order = OrderInput.model_validate(raw)     # 别名解析 + 去空格 + 字符串转 Decimal/int
order.customer      # 'Alice'（自动去空格）
order.items[0].price  # Decimal('199.5')（字符串自动转 Decimal）
order.total()         # Decimal('458.0')

# ③ 序列化出去（供存储 / 回给前端）
order.model_dump(mode="json", by_alias=True, exclude_none=True)
# {'customerName': 'Alice', 'items': [{'name': '键盘', 'price': '199.5', ...}], ...}
```

**这条链路的每一环都在用前面某节的能力**：

```fallback
  model_validate + alias      → 第十一章  外部名 → Python 名
  str_strip_whitespace        → 第九章    ConfigDict
  price: Decimal + 字符串自动转 → 第三章   宽松转换
  @field_validator / @model_validator → 第七章  自定义规则
  带方法 total()              → 第二章   模型=数据+行为
  model_dump(mode="json", by_alias=True) → 第十章  序列化
```

* * *

## 十五、常见坑与排查清单

1. **可变默认值**：`tags: list[str] = []` → 必须 `Field(default_factory=list)`。
2. **v1 语法照抄**：`parse_obj` / `.dict()` / `@validator` → 全按 v2 改（`model_validate` / `model_dump` / `field_validator`）。旧教程十有八九是 v1。
3. **宽松转换比预期激进**：`"yes"` → `True`、`123` → `"123"`。要精确就上严格模式（`StrictInt` / `strict=True`）。
4. **`None` 与必填混用**：`x: int = None` 是错的（类型注解得是 `int | None`）；`x: int | None` 不带默认 = 必填但可传 None，与"可选不传"是两回事。
5. **多字段规则放错位置**：单字段用 `field_validator`，跨字段必须 `model_validator(mode="after")`，放在 field_validator 里读不到别的字段。
6. **校验器不改就不知道有没有执行**：before/after 顺序、返回值的"覆盖输入"语义，出问题时按第七章的顺序表推演。
7. **email 相关字段报 ImportError**：`EmailStr` 需要装 `email-validator`。
8. **嵌套报错找不到是哪层**：看 `err["loc"]`——它是路径元组（`('items', 1, 'price')`），不是字符串。
9. **`repr=False` 没设，密码打印出来了**：敏感字段设 `Field(repr=False)`。
10. **直接赋值不校验**：默认只在构造时校验；要赋值也校验开 `validate_assignment=True`。
11. **前向引用报 NameError**：类体内引用自身/后定义的类，用字符串注解 + `model_rebuild()`。
12. **性能担忧**：v2 的校验在 Rust 里跑，很快。真正的开销来自"反复把同一条数据校验 N 遍"——在入口校验一次，别在每层都 validate。

* * *

## 小结：API 速查表

| 要干什么 | 用什么 |
|---|---|
| 定义模型 | `class X(BaseModel)` + 字段类型注解 |
| 校验入口 | `X.model_validate(dict)` / `X.model_validate_json(str)` |
| 宽松转换 | 默认开启：`"20"`→int、`123`→str、`"2026-.."`→datetime |
| 严格模式 | `StrictInt` 等 / `ConfigDict(strict=True)` / 调用时 `strict=True` |
| 校验失败 | `except ValidationError as e` → `e.errors()`（含 `loc` 路径） |
| 默认值 | 字面量 / `Field(default_factory=...)`（可变必用） |
| 值域约束 | `Field(ge/le/gt/lt, multiple_of, min_length, pattern)` |
| 白名单 | `Literal["a","b"]` |
| 自定义校验 | `@field_validator`（单字段）/ `@model_validator(mode="after")`（跨字段） |
| 全局配置 | `ConfigDict(strict / extra / frozen / validate_assignment / str_strip_whitespace / arbitrary_types_allowed)` |
| 序列化 | `model_dump(mode="json", exclude_none, by_alias)` / `model_dump_json()` |
| 复制更新 | `model_copy(update={...})` |
| 别名 | `Field(alias=...)` + `model_dump(by_alias=True)` + `populate_by_name` |
| 嵌套/递归 | 模型套模型 / `list[Model]` / 字符串自引用 + `model_rebuild()` |
| 单值校验 | `TypeAdapter(T).validate_python(x)` |
| 生成 schema | `X.model_json_schema()` → LangChain `bind_tools` / `args_schema` / PydanticOutputParser 的原料 |
| 字段元数据 | `X.model_fields` |

**记住一个原则**：**Pydantic = 在数据入口处声明规则一次，让"非法数据进不来、脏数据变干净、好数据随时能序列化"**。凡是"外部数据进系统"的地方——LLM 输出、配置文件、API 请求、工具参数——都值得先过一道模型。

> 下一篇（W5-D5）预告：**State Schema 进阶**——把 LangGraph 的 State 从 TypedDict 升级成 Pydantic 模型。到时候你会发现：这一篇的 `Field`、`ConfigDict`、`model_validator` 全部原样搬进 State，给"整张工单"上安检。
