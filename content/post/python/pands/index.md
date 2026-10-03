---
description: ""
title: "Pands"
draft: false
date: "2026-08-16T08:17:42+08:00"
slug: "pands"
categories:
 - 
tags:
 - 
image: ""
---

# Pandas 增删改查完全指南：从入门到实战（附示例）

## 一、准备：创建示例数据

```python
import pandas as pd

data = {
    "name":  ["Alice", "Bob", "Charlie", "David"],
    "age":   [20, 21, 19, 22],
    "score": [85, 92, 78, 88],
}
df = pd.DataFrame(data)
print(df)
#       name  age  score
# 0    Alice   20     85
# 1      Bob   21     92
# 2  Charlie   19     78
# 3    David   22     88
```

> 下文所有「增删改查」都基于这份 `df`。

## 二、查（Read / 查询）

### 1. 从外部读取数据
```python
df = pd.read_csv("data.csv")        # CSV
df = pd.read_excel("data.xlsx")     # Excel
df = pd.read_json("data.json")      # JSON
```

### 2. 快速预览
```python
df.head(3)        # 前 3 行
df.tail(2)        # 后 2 行
df.info()         # 字段、类型、非空数
df.describe()     # 数值列统计（均值/最值等）
```

### 3. 选择列
```python
df["score"]            # 单列（返回 Series）
df[["name", "score"]]  # 多列（返回 DataFrame）
```

### 4. 选择行：`loc` 按标签，`iloc` 按位置
```python
df.loc[0]              # 索引为 0 的那一行
df.loc[0:2]            # 索引 0~2（含 2）
df.iloc[0]             # 第 1 行（按位置）
df.iloc[0:2]           # 前 2 行（左闭右开）
```

### 5. 条件筛选
```python
df[df["score"] > 80]                      # 分数大于 80
df[(df["age"] > 20) & (df["score"] > 80)] # 多条件用 & | ~，每个条件加括号
df[df["name"].isin(["Alice", "Bob"])]     # 在列表中
```

## 三、增（Create / 新增）

### 1. 新增一列（最常用）
```python
df["grade"] = df["score"].apply(lambda x: "A" if x >= 90 else "B")
```

### 2. 新增一行
```python
# 推荐：pd.concat（pandas 2.x 中 append 已弃用）
new_row = {"name": "Eve", "age": 20, "score": 95}
df = pd.concat([df, pd.DataFrame([new_row])], ignore_index=True)

# 或：用 loc 直接追加到末尾
df.loc[len(df)] = ["Frank", 23, 90]
```

## 四、改（Update / 修改）

### 1. 修改单个单元格
```python
df.loc[0, "score"] = 90
```

### 2. 批量修改（向量化 / apply）
```python
df["score"] = df["score"] + 5                     # 整列 +5
df["name"]  = df["name"].str.upper()              # 字符串列转大写
df["score"] = df["score"].apply(lambda x: x * 1.1)  # apply 逐元素
```

### 3. 替换与填充
```python
df.replace({"grade": "B"}, "C", inplace=True)  # 值替换
df.fillna(0, inplace=True)                     # 缺失值填 0
```

### 4. 重命名列
```python
df.rename(columns={"score": "成绩"}, inplace=True)
```

## 五、删（Delete / 删除）

### 1. 删除列
```python
df.drop("age", axis=1, inplace=True)   # axis=1 表示列
# 等价写法： del df["age"]
```

### 2. 删除行
```python
df.drop(0, axis=0, inplace=True)       # axis=0 表示行（默认）
```

### 3. 删除缺失值 / 重复行
```python
df.dropna(inplace=True)                # 删除含空值的行
df.drop_duplicates(inplace=True)       # 删除完全重复的行
```

> `inplace=True` 会直接修改原 DataFrame；不想改动原数据就去掉它，并用 `df_new = ...` 接收返回值。

## 六、高级查询：条件组合、多重筛选与字符串搜索

基础筛选 `df[df["score"] > 80]` 只够入门。真实分析里，你要的是「多条件组合 + 模糊匹配 + 跨列逻辑」。下面全部基于这份数据：

```python
import pandas as pd
df = pd.DataFrame({
    "name":  ["Alice", "Bob", "Charlie", "David", "Eve"],
    "age":   [20, 21, 19, 22, 20],
    "score": [85, 92, 78, 88, 95],
    "city":  ["BJ", "SH", "BJ", "GZ", "SH"],
})
```

### 1. 多条件组合：`&`（且）、`|`（或）、`~`（非）
```python
# 分数>80 且 年龄>20
mask = (df["score"] > 80) & (df["age"] > 20)
df[mask]

# 分数<60 或 分数>95
df[(df["score"] < 60) | (df["score"] > 95)]

# 非：不在 90 分及以上
df[~(df["score"] >= 90)]
```
> ⚠️ 每个条件必须单独用括号包起来，`&`/`|` 优先级高于比较运算，不加括号会算错。

### 2. `df.query()`：用字符串表达式查询（推荐，可读性强）
```python
df.query("score > 80 and age > 20")
df.query("city == 'BJ'")                       # 字符串值加引号
df.query("score > @t", local_dict={"t": 80})   # 引用外部变量加 @
df.query("score > 80 and city in ['BJ','SH']") # in 多值
```
`query` 支持 `and / or / not / in`，写法接近 SQL，复杂条件尤其清爽，性能也不错。

### 3. `isin()`：多值匹配（等于列表里任意一个）
```python
df[df["name"].isin(["Alice", "Bob", "David"])]   # 在列表中
df[~df["name"].isin(["Alice"])]                  # 不在列表中
```

### 4. `between()`：闭区间范围查询
```python
df[df["score"].between(80, 90)]   # 80 <= score <= 90
```

### 5. 字符串模糊与正则搜索：`str.contains` 等
```python
df[df["name"].str.contains("li")]               # 包含 "li"
df[df["name"].str.contains("li", case=False)]   # 忽略大小写
df[df["name"].str.contains(r"^A", regex=True)]  # 正则：以 A 开头
df[df["name"].str.startswith("A")]              # 以 A 开头
df[df["name"].str.endswith("e")]                # 以 e 结尾
```

### 6. 跨列自定义函数筛选
```python
df[df.apply(lambda r: r["score"] > r["age"] * 4, axis=1)]
```
> 能用向量化就用向量化，`apply` 逐行执行，数据量大时慢很多。

### 7. 多条件赋值：`np.where`（二分类）/ `np.select`（多分类）
```python
import numpy as np
df["pass"] = np.where(df["score"] >= 60, "及格", "不及格")

conditions = [
    df["score"] >= 90,
    (df["score"] >= 80) & (df["score"] < 90),
    df["score"] < 80,
]
choices = ["A", "B", "C"]
df["level"] = np.select(conditions, choices, default="?")
```

### 8. 取最大 / 最小 N 行
```python
df.nlargest(3, "score")     # 分数最高的 3 行
df.nsmallest(3, "score")    # 分数最低的 3 行
```

### 9. 跨表关联查询：`merge` 再筛选
```python
other = pd.DataFrame({"name": ["Alice", "Bob"], "status": ["active", "off"]})
merged = pd.merge(df, other, on="name", how="left")
merged[merged["status"] == "active"]   # 筛出活跃用户
```

---

## 七、综合实战

需求：给不及格（<80）的同学每人加 5 分，新增「等级」列，删掉重复行，最后保存。

```python
import pandas as pd

df = pd.DataFrame({
    "name":  ["Alice", "Bob", "Charlie", "David", "Alice"],
    "score": [85, 92, 78, 88, 85],
})

# 改：不及格 +5
df.loc[df["score"] < 80, "score"] += 5

# 增：等级列
df["level"] = df["score"].apply(lambda x: "A" if x >= 90 else "B")

# 删：重复行（Alice 出现两次）
df.drop_duplicates(inplace=True)

# 查：看结果
print(df)

# 存盘
df.to_csv("result.csv", index=False)
```

## 小结

| 操作 | 核心 API |
|------|----------|
| 查 | `[]`、`loc`、`iloc`、`df[条件]`、`query`、`isin`、`between`、`str.contains`、`read_*` |
| 增 | `df["新列"]=`、`pd.concat`、`df.loc[len]` |
| 改 | `df.loc[行,列]=`、`apply`、`replace`、`fillna`、`rename`、`np.where`/`np.select` |
| 删 | `drop`、`dropna`、`drop_duplicates` |
| 高级查询 | `&`/`|`/`~`、`query()`、`isin`、`between`、`str.contains`、跨列 `apply`、`merge` |

记住一个原则：**查询多用布尔索引与 `query()`，新增多用 concat，修改多用 loc + 向量化，删除多用 drop 系列，复杂分类用 `np.select`**。把本文示例跑一遍，pandas 的增删改查与高级搜索就真正上手了。
