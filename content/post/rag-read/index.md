---
description: ""
title: "Embedding + Chroma，把文档变成可以按语义检索的向量库"
draft: false
date: "2026-09-24T06:00:58+08:00"
slug: "rag-read"
categories:
 - Rag
tags:
 - 
image: ""
---

# W8D2：Embedding + Chroma，把文档变成可以按语义检索的向量库

## 一、先把今天放回整条链路里

W8D1 做的是：

```text
原始文档
   ↓
读取
   ↓
切分
   ↓
Document 块
```

W8D2 做的是：

```text
Document 块
   ↓
Embedding
   ↓
向量
   ↓
Chroma 向量库
   ↓
落盘
```

W8D3 才会真正做：

```text
用户问题
   ↓
Embedding
   ↓
查询向量
   ↓
向量库寻找相近向量
   ↓
返回相关文档块
```

所以今天其实不是单纯在学一个 `Chroma` API。

我今天真正要建立的模型是：

> **Day1 解决“文档怎么变成块”，Day2 解决“这些块怎么变成可以比较语义距离的数字，并且保存下来”，Day3 再解决“怎么拿问题去找这些数字”。**

整个过程可以先记成：

```text
文字
 ↓
Chunk
 ↓
Embedding
 ↓
Vector
 ↓
Vector Store
 ↓
Persistence
 ↓
Query
```

今天主要停在 `Vector Store + Persistence`。

---

## 二、Embedding 到底在解决什么问题

我之前容易把 Embedding 理解成：

> “把文字变成数字。”

这个说法没错，但其实不够。

真正需要理解的是：

> **Embedding 是把文字映射成一个向量，让原本不能直接进行数值比较的文本，进入一个可以计算“距离”的空间。**

例如：

```text
“我想喝咖啡”
“给我来一杯美式”
```

如果做普通关键词匹配，它们的字面差异很大。

但 Embedding 的目标不是比较“字长得像不像”，而是希望：

```text
意思接近
   ↓
向量空间中的位置也更接近
```

所以可以把它想象成给每个文本一个“语义坐标”。

```text
                美食
                 ↑
                 │     “咖啡”
                 │       ●
                 │    ●
                 │ “美式”
                 │
                 └────────────→
```

这样到了 Day3，用户的问题也会被变成一个向量：

```text
问题 → 向量

文档块1 → 向量
文档块2 → 向量
文档块3 → 向量
...
```

然后就可以比较：

```text
问题向量
   ↓
谁离它最近？
   ↓
把这些文档块找出来
```

所以我现在对 Embedding 的理解是：

> **它不是为了“存数字”，而是为了把文本转换成一种可以计算相似程度的表示。**

---

## 三、向量库又是在干什么

到这里还有一个容易混的地方：

Embedding 已经把文字变成向量了，为什么还需要 Chroma？

因为：

> **Embedding 负责“算出向量”，Chroma 负责“把这些向量和原文保存起来，并提供检索能力”。**

所以两者职责并不是一回事。

```text
Embedding Model
    │
    │ 文本 → 向量
    ↓
[0.02, -0.18, 0.42, ...]
    │
    ↓
Chroma
    │
    ├── 向量
    ├── 原始文本
    └── metadata
```

可以把它简单理解成：

```text
Embedding = 翻译器
Chroma    = 仓库
```

Embedding 把“人类语言”翻译成“机器可以比较的坐标”。

Chroma 则负责把这些坐标、对应文本和 metadata 存起来，以后查。

这也是为什么 W8D2 的核心不是只有 Embedding。

真正完整的是：

```text
文本块
 ↓
Embedding
 ↓
向量
 ↓
向量 + 文本 + metadata
 ↓
Chroma
 ↓
持久化
```

---

## 四、今天代码里最值得看的部分：HashEmbeddings

今天没有一上来就直接调用真正的 Embedding API，而是先写了一个 `HashEmbeddings`。

这个设计其实很值得学。

关键代码只有这一小段：

```python
def _vec(self, text: str) -> list[float]:
    v = [0.0] * self.dim

    for i in range(len(text) - 1):
        gram = text[i:i + 2]
        h = int(
            hashlib.md5(
                gram.encode("utf-8")
            ).hexdigest()[:8],
            16
        )
        v[h % self.dim] += 1.0

    norm = sum(x * x for x in v) ** 0.5 or 1.0
    return [x / norm for x in v]
```

我不需要记住这一套算法，只需要理解它在干什么。

### 1. 先准备一个固定长度的向量

```python
v = [0.0] * self.dim
```

默认：

```python
dim = 256
```

所以每段文字最后都会得到：

```text
256维向量
```

无论原文是 20 个字还是 200 个字，最终表示长度都是一样的。

---

### 2. 把文本拆成两个字一组

```python
gram = text[i:i + 2]
```

比如：

```text
“我喜欢咖啡”
```

大致会得到：

```text
我喜
喜欢
欢咖
咖啡
```

然后每一个二元组经过 hash，被映射到 256 个位置中的某一个位置。

所以最后会得到一个类似：

```text
[0, 0, 2, 0, 1, 0, ..., 3, 0]
```

的向量。

这里最重要的不是 Hash 算法本身，而是理解：

> **代码正在把文本中的局部特征，映射到固定维度的数字空间。**

---

### 3. 为什么还要归一化

最后：

```python
norm = sum(x * x for x in v) ** 0.5 or 1.0
return [x / norm for x in v]
```

就是把向量做 L2 归一化。

目的可以先简单理解成：

> 把不同长度、不同大小的向量调整到统一尺度，方便后面的距离/相似度计算。

所以这段代码的完整链路其实是：

```text
文本
 ↓
切成 2-gram
 ↓
hash
 ↓
映射到 256 维
 ↓
累计
 ↓
归一化
 ↓
最终向量
```

这就是今天代码里我真正需要理解的一段。

不是为了让我以后手写一个 Hash Embedding，而是为了让我看到：

> **“Embedding”在程序里最终就是一个“文本 → 向量”的过程。**

---

## 五、为什么还专门做了一个假的 Embedding

这部分是今天我觉得最有工程意味的地方。

代码里有：

```python
def get_embeddings(kind: str) -> Embeddings:
    if kind == "smoke":
        return HashEmbeddings(dim=256)

    if kind == "openai":
        ...

    if kind == "bge":
        ...
```

也就是说，同一个向量库构建流程，可以替换不同的 Embedding 后端：

```text
             ┌── HashEmbeddings
             │
文本 → Embedding ── BGE
             │
             └── OpenAI
```

为什么不直接用真正的 BGE？

因为今天首先要证明的是：

> **这套系统的“管道”到底通不通。**

而不是马上判断：

> “这个模型的语义效果到底好不好。”

这两个问题其实完全不同。

### 链路问题

例如：

```text
文本
 ↓
Embedding
 ↓
Chroma
 ↓
落盘
 ↓
重新加载
 ↓
查询
```

这里任何一步写错，都可能导致整个程序失败。

### 模型质量问题

即使整条链路完全正确，也可能出现：

```text
查询：
“我想喝咖啡”

结果：
完全不相关的文档
```

这时候不一定是 Chroma 有问题，也可能只是 Embedding 模型本身不适合当前数据。

所以 `--smoke` 的价值就是：

```text
先把“系统有没有坏”验证掉
        ↓
再研究“模型聪不聪明”
```

这其实是一个很重要的测试思想：

> **不要把工程故障和模型能力混成一个问题。**

---

## 六、今天第二个真正重要的概念：Persistence

假设现在有 100 个文档块。

第一次运行：

```text
100 个 Chunk
 ↓
100 次 Embedding
 ↓
存进 Chroma
```

如果程序结束以后，下一次启动又重新做：

```text
100 个 Chunk
 ↓
重新 Embedding
 ↓
重新建库
```

那前一次做的工作就没有真正被复用。

所以今天有一个非常重要的工程思想：

> **建一次，落盘，以后加载。**

代码里的核心就是：

```python
PERSIST_DIR = str(HERE / "chroma_db")
```

然后：

```python
vs = Chroma(
    collection_name=COLLECTION,
    embedding_function=emb,
    persist_directory=PERSIST_DIR
)
```

这里真正重要的并不是记住 `persist_directory` 这个参数。

而是理解：

```text
内存中的对象
      ↓
持久化
      ↓
磁盘上的数据库
      ↓
下次程序启动
      ↓
重新打开
```

于是向量库就从：

```text
“这次程序运行期间的变量”
```

变成：

```text
“可以跨程序运行复用的数据”
```

这才是今天“入库 + 落盘”的真正意义。

---

## 七、所以代码为什么要先判断库存不存在

这个分支是今天主流程最值得理解的地方：

```python
if index_exists() and not reset:
    vs = Chroma(
        collection_name=COLLECTION,
        embedding_function=emb,
        persist_directory=PERSIST_DIR
    )

    n = count_of(vs)

    return vs, 0, []
```

整个逻辑其实非常简单：

```text
程序启动
   ↓
检查 chroma_db 是否存在
   │
   ├── 有
   │    ↓
   │   直接加载
   │
   └── 没有
        ↓
      读取文档
        ↓
      分块
        ↓
      Embedding
        ↓
      Chroma
        ↓
      落盘
```

这里最重要的是：

> **“有没有库”决定后面走哪条路。**

这其实已经不是单纯的 API 使用了，而是在写一个有状态的程序。

第一次运行：

```text
build
→ create
→ embed
→ persist
```

第二次运行：

```text
build
→ detect existing
→ load
```

这和之前我学过的很多“运行一次就结束”的脚本不太一样。

程序开始需要考虑：

> **系统上一次运行留下了什么？**

这就是持久化系统开始出现后的一个重要思维变化。

---

## 八、为什么 `--reset` 也很重要

既然“存在就直接加载”，那旧库有问题怎么办？

比如：

```text
旧 Embedding
    ↓
旧向量库
```

现在我把模型换了：

```text
新 Embedding
```

但程序发现 `chroma_db` 已经存在，于是直接加载旧库。

这样就可能出现：

```text
现在用的是模型 B
但是库里存的是模型 A 生成的向量
```

所以代码提供：

```bash
python w8\build_index.py --reset
```

它做的事情就是：

```text
删除旧 chroma_db
        ↓
重新读取文档
        ↓
重新 Embedding
        ↓
重新建库
```

因此 `reset` 可以理解成：

> **“放弃旧状态，从头构建一份新的向量库。”**

所以我现在可以把两个参数理解成：

```text
默认运行
→ 尽量复用已有状态

--reset
→ 明确放弃旧状态重新建立
```

这已经很接近真实工程里的缓存、索引、数据库重建问题了。

---

## 九、入库前为什么还要过滤 Chunk

Day1 留下了一个问题。

之前的分块结果里有：

```text
[190, 200, 200, 200, 1]
```

最后一块只有：

```text
“。”
```

Day1 里它只是一个“不太好看的分块”。

到了今天，它开始真的产生后果。

因为如果它进入向量库：

```text
“。”
 ↓
Embedding
 ↓
向量
 ↓
进入检索库
```

那么以后查询时，它也有可能参与召回。

于是：

```text
真正有用的文档块
+
一个只有“。”的垃圾块
```

一起进入向量空间。

所以今天增加了：

```python
MIN_CHARS = 20

kept = [
    c for c in chunks
    if len(c.page_content.strip()) >= MIN_CHARS
]
```

这里我学到的不是：

> “以后都设置 20 字。”

这个 `20` 本身只是这次代码里的一个工程取值。

真正值得记住的是：

> **数据清洗发生在进入向量库之前。**

也就是说：

```text
原始文档
 ↓
分块
 ↓
清洗
 ↓
Embedding
 ↓
入库
```

而不是：

```text
原始文档
 ↓
直接全部 Embedding
 ↓
再想办法补救
```

---

## 十、metadata 为什么还需要单独处理

Chunk 不只有正文。

它通常还会带：

```text
page_content
metadata
```

例如：

```python
{
    "source": "xxx.md",
    "start_index": 200
}
```

这些 metadata 对后面定位来源很重要。

但代码里又专门做了一步：

```python
def clean_meta(m: dict) -> dict:
    return {
        k: v
        for k, v in m.items()
        if isinstance(v, (str, int, float, bool))
    }
```

我需要理解的不是这个字典推导式，而是一个边界：

> **进入 Chroma 的 metadata 不能随便塞任意 Python 对象。**

所以这里实际上又经历了一次“边界转换”：

```text
Python 中的 metadata
        ↓
清洗
        ↓
Chroma 能接受的 metadata
```

这也是今天比较典型的一种工程工作：

> **上一个组件允许的东西，不一定就是下一个组件允许的东西。**

所以在系统串联时，中间经常需要做：

```text
转换
清洗
裁剪
适配
```

---

## 十一、为什么要分批 `add_texts`

真正入库的关键代码是：

```python
for i in range(0, len(texts), BATCH):
    vs.add_texts(
        texts=texts[i:i + BATCH],
        metadatas=metas[i:i + BATCH]
    )
```

这里的核心并不是 Python 的 `range`。

而是：

> **不要一次把所有文本都塞进去，而是按批次处理。**

今天：

```python
BATCH = 64
```

于是：

```text
1～64
65～128
129～192
...
```

这样做的工程意义主要是：

```text
避免一次处理的数据过大
      ↓
控制请求 / 内存 / 任务规模
      ↓
更容易观察进度
      ↓
更容易处理失败
```

今天的数据量很小，所以这个设计看起来可能有些“多余”。

但数据量一大，这种批处理就会变成很自然的工程需要。

---

## 十二、Chroma 这里还有一个我容易忽视的概念：距离

今天最后真正开始碰到“检索”了：

```python
hits = vs.similarity_search_with_score(probe, k=k)
```

这里的 `score` 不应该简单理解成：

```text
分数越大越好
```

因为当前 Chroma 返回的是：

> **距离（distance）**

所以这里是：

```text
距离越小
→ 越接近
→ 越相似
```

当前代码创建集合时还指定：

```python
collection_metadata={"hnsw:space": "cosine"}
```

所以这里采用的是 cosine distance 的空间。

今天不用深入 HNSW 内部算法。

我现在只需要建立这个关系：

```text
Embedding
 ↓
向量
 ↓
向量之间可以计算距离
 ↓
距离近
 ↓
认为更相似
```

这一步非常关键，因为它把前面的 Embedding 和后面的 Retrieval 真正接上了。

---

## 十三、今天的 `self_check` 为什么比“程序没报错”重要

以前我容易觉得：

```text
程序运行结束
没有异常
```

≈

```text
程序做对了
```

今天这个观点被进一步打破。

代码最后专门做了四层检查：

```text
① 文件真的落盘了吗？
        ↓
② 库里的数量对吗？
        ↓
③ 重新打开之后数量还一样吗？
        ↓
④ 查询真的能把自己的数据找回来吗？
```

对应：

```python
assert "chroma.sqlite3" in files
```

然后：

```python
assert n == n_added
```

再：

```python
vs2 = Chroma(...)
n2 = count_of(vs2)

assert n2 == n
```

最后：

```python
hits = vs.similarity_search_with_score(...)
```

看自己的 Chunk 能不能被找回来。

这里我觉得最值得记住的是第三层：

```text
写入
 ↓
关闭程序
 ↓
重新打开
 ↓
还能看到原来的数据
```

因为这才真正证明：

> **我不是刚刚把数据放在内存里，而是真的完成了持久化。**

---

## 十四、为什么“自己搜自己”不能代表语义质量

这里又有一个很容易误解的地方。

代码最后会拿自己的文本做查询：

```python
probe = texts[0][:30]
hits = vs.similarity_search_with_score(probe, k=k)
```

然后检查：

```text
自己的块有没有被召回
```

这个测试能证明：

```text
Embedding 能跑
 ↓
向量写进库了
 ↓
查询能执行
 ↓
向量检索链路基本通了
```

但它不能证明：

```text
“咖啡”真的能找到“美式”
```

因为今天的 `HashEmbeddings` 本来就不是为了理解语义。

所以必须区分：

```text
链路正确性
≠
模型语义质量
```

这两个问题要分开测。

今天：

```text
--smoke
→ 测链路
```

Day3：

```text
真实问题
→ 测召回效果
```

这个思维以后不只是 RAG 能用。

任何 AI 系统都可以拆成：

```text
系统有没有正常运行？
```

和：

```text
系统运行出来的结果好不好？
```

前者是工程正确性，后者才是能力质量。

---

## 十五、把今天所有知识重新串起来

现在再回头看 `build_index.py`，它其实没有那么复杂。

整个程序就是：

```text
启动
 ↓
选择 Embedding
 ↓
检查旧向量库
 │
 ├── 已存在
 │      ↓
 │    直接加载
 │
 └── 不存在 / --reset
        ↓
      读取 Day1 文档
        ↓
      切成 Chunk
        ↓
      过滤无效 Chunk
        ↓
      清洗 metadata
        ↓
      Embedding
        ↓
      变成向量
        ↓
      分批写入 Chroma
        ↓
      持久化
        ↓
      自检
```

所以如果让我现在用一句自己的话总结 W8D2：

> **今天是在把 Day1 得到的一堆文本块，经过 Embedding 转成可以计算距离的向量，再连同文本和 metadata 一起放入 Chroma，并把结果持久化，这样下一次运行时可以直接加载，而不是重新做一遍 Embedding。**

---

## 十六、今天真正学到的，其实不只是 RAG

这一天表面上是在学：

```text
Embedding
Chroma
Vector Store
Persistence
```

但我觉得更重要的是开始出现几个工程思维：

### 1. 一个组件解决一个问题

```text
Embedding
→ 负责表示

Chroma
→ 负责存储与检索

Persistence
→ 负责跨运行复用
```

不要把它们混成一个东西。

### 2. 先把“系统正确”与“结果优秀”分开

```text
先证明：
链路通

再证明：
效果好
```

### 3. 数据进入下一个系统前，要注意边界

```text
Chunk
 ↓
过滤

metadata
 ↓
清洗

text
 ↓
Embedding

vector
 ↓
Chroma
```

每一步都有自己的输入输出约束。

### 4. 程序开始处理“历史状态”

今天第一次明显感觉到：

```text
程序不是只处理当前输入
```

而是开始处理：

```text
当前输入
+
过去运行留下的数据
```

这就是为什么会出现：

```text
存在 → 加载
不存在 → 创建
reset → 重建
```

这其实已经开始进入真正工程系统的思维了。

---

## 十七、W8D2 最终留下的知识骨架

最后不记代码，记这张图就够了：

```text
                W8D2
                  │
        ┌─────────┴─────────┐
        ↓                   ↓
     Embedding           Persistence
        │                   │
        ↓                   ↓
   文本 → 向量          建一次 → 落盘
        │                   │
        ↓                   ↓
   可以计算距离          下次直接加载
        │
        ↓
      Chroma
        │
        ├── vector
        ├── text
        └── metadata
        │
        ↓
    similarity search
```

而整个 W8 到这里可以串成：

```text
D1
文档 → Chunk
   ↓
D2
Chunk → Embedding → Vector Store → Persistence
   ↓
D3
Query → Embedding → Similarity Search → Relevant Chunks
```

所以我现在终于可以把 RAG 前半段看成一条完整的数据流，而不是几个零散的 API：

```text
文档
 ↓
Chunk
 ↓
Embedding
 ↓
Vector
 ↓
Vector Store
 ↓
Query
 ↓
Distance
 ↓
Retrieval
```

这条链真正跑通之后，后面才开始进入：

```text
Retrieved Context
        ↓
       LLM
        ↓
      Answer
```

也就是 RAG 真正“把检索结果交给模型”的部分。
