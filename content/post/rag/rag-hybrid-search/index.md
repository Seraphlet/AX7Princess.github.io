---
description: ""
title: "W8-Day4：RAG 混合检索——Dense、BM25 与 RRF"
draft: false
date: "2026-10-05T06:34:40+08:00"
slug: "rag-hybrid-search"
categories:
 - RAG
tags:
 - 
image: ""
---

# W8-Day4：RAG 混合检索——Dense、BM25 与 RRF

前面几天已经把 RAG 最基本的一条链路跑通了：

```text
文档
 ↓
Chunk
 ↓
Embedding
 ↓
Chroma
 ↓
Dense Retrieval
 ↓
Top-K
 ↓
LLM
```

到这里已经可以根据“语义相似度”找到相关文档。

但今天要解决的问题是：

> **语义相似，不代表一定能把“必须精确命中的东西”找出来。**

因此今天没有推翻 Dense Retrieval，而是在它旁边增加了一条 Sparse Retrieval，最后再把两条检索路线的结果融合起来。

最终结构变成：

```text
                         Query
                           │
             ┌─────────────┴─────────────┐
             │                           │
             ↓                           ↓
       Dense Retrieval            Sparse Retrieval
        Embedding                    BM25
             │                           │
        语义相似                    词项精确匹配
             │                           │
             └─────────────┬─────────────┘
                           ↓
                          RRF
                        排名融合
                           ↓
                         Top-K
                           ↓
                          LLM
```

今天真正需要理解的就是这张图。

---

## 一、为什么 Dense Retrieval 还不够

Dense Retrieval 做的事情可以简单理解为：

```text
Query
 ↓
Embedding
 ↓
Query Vector
 ↓
和文档 Vector 比较
 ↓
找距离最近的文档
```

它最大的优势是：

> **不要求 Query 和文档使用完全相同的词，只要意思接近就可能找到。**

例如：

```text
Query：弄翻了一杯拿铁

Document：咖啡洒了
```

虽然字面不同，但 Embedding 可以把它们映射到相近的语义空间。

这正是 Dense Retrieval 的价值。

但它也带来一个问题。

假设用户查询：

```text
ERR_4517
```

或者：

```text
ZQ-7781
GB/T 22239
```

这时候用户根本不需要“意思差不多”。

他要的是：

> **就是这个编号。**

例如：

```text
ERR_4517
ERR_4518
```

从自然语言意义上看，它们可能非常接近——都是某种错误码。

但对于业务系统来说：

```text
4517 != 4518
```

差一个字符可能就是完全不同的问题。

所以 Dense 擅长回答：

```text
“这大概是在说什么？”
```

却不天然擅长回答：

```text
“到底是不是这个？”
```

这就是为什么还需要 Sparse Retrieval。

---

# 二、Sparse Retrieval：BM25

Sparse Retrieval 更接近传统搜索。

今天使用的是：

```text
BM25
```

它主要依赖词项匹配。

可以先粗略理解成：

```text
Query
 ↓
分词
 ↓
找到包含这些词的文档
 ↓
根据词的重要程度计算分数
 ↓
排序
```

但 BM25 并不只是简单地：

```python
if keyword in document:
```

它还考虑三个重要问题。

---

## 1. TF：一个词出现很多次，不应该无限加分

假设某篇文章疯狂重复：

```text
RAG RAG RAG RAG RAG RAG RAG……
```

如果简单按照出现次数计算：

```text
出现 100 次
>
出现 10 次
>
出现 1 次
```

那么只要不断堆关键词，就可以把搜索排名刷上去。

BM25 对 TF 做了饱和处理。

核心思想不是公式，而是：

```text
第一次出现       很有价值
第二次出现       还有价值
第十次出现       增益已经很小
第一百次出现     基本没什么额外价值
```

也就是：

> **词频有价值，但价值不会无限增长。**

---

## 2. IDF：越罕见的词越重要

例如一份公司文档中：

```text
系统
接口
用户
```

这些词到处都是。

它们对定位某一篇文档帮助很小。

但如果出现：

```text
ERR_4517
```

而整个知识库只有一篇文档包含它，那么这个词就具有非常强的定位能力。

所以：

```text
高频公共词 → 权重低

稀有词     → 权重高
```

这就是 IDF 的核心思想。

这也是 Sparse Retrieval 在错误码、型号、条款号、人名、专有名词等场景中的优势。

---

## 3. 文档长度归一化

还有一个问题。

假设：

```text
文档 A：100 字
文档 B：10000 字
```

文档 B 因为特别长，自然更容易包含 Query 中的词。

如果不处理这个问题，长文档可能仅仅因为：

> “话多”

就更容易获得高分。

所以 BM25 还会根据文档长度进行修正。

最终可以把 BM25 先记成：

```text
BM25
 =
词项匹配
+
TF 饱和
+
IDF
+
文档长度修正
```

不用背公式，但要知道这些机制分别解决什么问题。

---

# 三、Dense 和 Sparse 为什么要同时存在

现在两条路线的边界就清楚了。

```text
Dense
│
├─ 擅长语义
├─ 可以没有相同词
└─ “意思像”
```

而：

```text
Sparse / BM25
│
├─ 擅长词项
├─ 稀有词定位能力强
└─ “就是这个”
```

例如：

```text
“弄翻拿铁”
        ↓
Dense 更有优势
        ↓
“咖啡洒了”
```

而：

```text
“ERR_4517”
        ↓
Sparse 更有优势
        ↓
包含 ERR_4517 的文档
```

所以 Hybrid Search 并不是：

> 两种高级技术叠起来一定更强。

而是：

> **两种检索方式的失败模式不同，所以让它们互相补盲区。**

这是今天最重要的一句话。

---

# 四、为什么不能直接把两个 Score 相加

现在又出现一个问题。

Dense 会返回自己的分数：

```text
Document A    0.82
Document B    0.76
Document C    0.65
```

BM25 也会返回：

```text
Document A    8.4
Document D    6.7
Document B    4.2
```

最直觉的想法可能是：

```python
final_score = dense_score + bm25_score
```

但这样做存在问题。

因为：

```text
Dense Score
和
BM25 Score
```

不是同一种东西。

可以类比：

```text
80 摄氏度
+
80 公里
```

两个数字都叫 80，不代表可以直接相加。

Dense 的分数表达的是向量空间里的相似关系。

BM25 的分数表达的是词项匹配强度。

两边的尺度、范围和意义都不同。

因此今天采用另一个办法：

# RRF

Reciprocal Rank Fusion。

它的核心思想非常简单：

> **既然两个系统的分数不能直接比较，那我干脆不比较分数，只比较排名。**

---

# 五、RRF：只看你排第几

假设 Dense 返回：

```text
Dense

1. A
2. B
3. C
```

Sparse 返回：

```text
Sparse

1. B
2. D
3. A
```

RRF 不关心：

```text
A 的 Dense Score 是多少
B 的 BM25 Score 是多少
```

它只看：

```text
A 在 Dense 第 1
A 在 Sparse 第 3

B 在 Dense 第 2
B 在 Sparse 第 1
```

然后按照：

```text
1 / (k + rank)
```

给每次排名贡献一个分数。

因此：

```text
A
=
Dense 第1的贡献
+
Sparse 第3的贡献
```

B：

```text
B
=
Dense 第2的贡献
+
Sparse 第1的贡献
```

最后重新排序。

---

# 六、今天最值得看的代码：RRF

完整工程代码很多，但 Hybrid Search 真正新的核心算法其实非常短：

```python
def rrf_fuse(ranked_id_lists, k=60, weights=None, top_n=None):
    score = defaultdict(float)

    for i, ids in enumerate(ranked_id_lists):
        w = 1.0 if weights is None else weights[i]

        for rank, did in enumerate(ids, 1):
            score[did] += w / (k + rank)

    order = sorted(
        score.items(),
        key=lambda x: (-x[1], str(x[0]))
    )

    return order[:top_n] if top_n else order
```

先不看 Python 语法细节，把它还原成流程：

```text
准备一个 score 表
        ↓
遍历每一条检索路线
        ↓
遍历这条路线中的每个文档
        ↓
根据 rank 计算贡献
        ↓
累加到这个文档
        ↓
全部完成后重新排序
```

也就是：

```text
Dense ranking ───┐
                 ├──→ 累加排名分数 → 排序
Sparse ranking ──┘
```

这里真正重要的是：

```python
score[did] += w / (k + rank)
```

同一个 `did` 如果同时被 Dense 和 Sparse 找到，就会被累计两次。

所以：

> 两个检索器都认为不错的文档，会获得更高的融合排名。

---

# 七、完整 Hybrid 主调用链

真正把 Dense、Sparse 和 RRF 串起来的是：

```python
def hybrid(self, query: str, k: int, weights=None):
    d = self.dense(query, self.n_per_route)

    s = self.sparse_search(
        query,
        self.n_per_route
    )

    fused = rrf_fuse(
        [[i for i, _ in d],
         [i for i, _ in s]],
        k=self.rrf_k,
        weights=weights,
        top_n=k,
    )

    return fused, d, s
```

把代码压缩成数据流：

```text
query
 │
 ├──→ dense()
 │      ↓
 │   Dense Ranking
 │
 └──→ sparse_search()
        ↓
     Sparse Ranking
          │
          ↓
       rrf_fuse()
          │
          ↓
       Final Top-K
```

以后重新看这份代码，首先找这个函数。

因为它就是整个 Hybrid Retrieval 的主链。

其他代码基本都是围绕这条主链提供能力。

---

# 八、为什么两条路线必须使用相同的 ID

Hybrid Search 还有一个非常重要的工程条件：

```text
Dense 找到的 Document A

和

Sparse 找到的 Document A
```

系统必须知道：

> **这是同一个 Document A。**

所以两边必须共享同一套文档 ID。

代码因此从 Chroma 中读取：

```python
data = self.vs.get(
    include=["documents", "metadatas"]
)

self.ids = data["ids"]
self.docs = data["documents"]
```

然后 BM25 直接使用：

```python
self.sparse = SparseIndex(
    self.ids,
    self.docs,
    TOKENIZERS[tok]
)
```

也就是：

```text
               Chroma
                 │
          ids + documents
                 │
        ┌────────┴────────┐
        ↓                 ↓
      Dense             BM25
        │                 │
        └──── 同一 ID ────┘
```

否则可能出现：

```text
Dense 的 id=1
```

代表文档 A，

但：

```text
Sparse 的 id=1
```

代表文档 B。

这时候 RRF 虽然还能计算：

```python
score[1] += ...
```

但这个结果已经没有意义。

所以：

> **融合之前首先要保证两条路线处在同一个 ID 空间。**

这是一个很典型的工程边界。

---

# 九、为什么中文 BM25 还需要分词

Dense 使用 Embedding。

BM25 使用的是词项。

所以：

```text
“接口返回 ERR_4517 怎么办”
```

必须先被拆成 token。

例如：

```text
接口
返回
ERR_4517
怎么办
```

代码里因此提供：

```python
tok_jieba()
```

和：

```python
tok_bigram()
```

这里真正需要理解的不是两个 tokenizer 的所有实现。

而是：

> **Sparse Retrieval 的效果依赖“系统认为一个词是什么”。**

特别是：

```text
ERR_4517
ZQ-7781
```

这种标识符。

如果被错误拆成：

```text
ERR
_
4517
```

原本一个非常稀有、定位能力极强的 token 就被破坏了。

所以代码额外使用正则保护标识符。

这也是为什么：

> 中文 Sparse Retrieval 中，分词不是一个无关紧要的预处理步骤，而是检索系统的一部分。

---

# 十、Retriever 和 Reranker 不是一回事

今天还有一个概念只需要知道位置：

```text
Retriever
```

负责：

```text
10 万条
 ↓
找 20～50 条候选
```

然后：

```text
Reranker
```

负责：

```text
50 条
 ↓
重新精细判断
 ↓
5 条
```

所以完整结构可以变成：

```text
                    Query
                      │
          ┌───────────┴───────────┐
          ↓                       ↓
        Dense                   Sparse
          │                       │
          └──────────┬────────────┘
                     ↓
                    RRF
                     ↓
                候选 20～50 条
                     ↓
                 Reranker
                     ↓
                   Top-5
```

今天不需要实现复杂 Reranker。

只需要知道：

> **Retriever 负责大范围粗筛，Reranker 负责小范围精排。**

---

# 十一、一个很容易踩的坑：RRF Score 不是相似度

Day3 使用 Dense Retrieval 时，可以根据相似度设置拒答阈值。

例如：

```text
similarity < threshold
        ↓
      拒答
```

但进入 RRF 后：

```text
RRF Score
```

已经不再表示：

> Query 和 Document 有多相似。

它表示的是：

> 这个 Document 在多个 ranking 中综合排得怎么样。

所以：

```text
Dense similarity
```

和：

```text
RRF score
```

不能共用一个 threshold。

这其实是一个很重要的工程原则：

> **数据经过一层转换以后，要重新确认这个数字的语义，不能因为它还叫 score 就继续沿用上一层的规则。**

---

# 十二、什么时候不需要 Hybrid

Hybrid 并不是默认正确答案。

如果：

```text
语料很小
```

或者：

```text
Query 几乎全部是自然语言问题
```

又或者：

```text
文档中根本没有
错误码 / 型号 / 条款号 / 人名 / 专有标识符
```

那么 Dense Retrieval 可能已经足够。

增加 Hybrid 意味着同时增加：

```text
Dense Index
+
Sparse Index
+
Tokenizer
+
ID 一致性
+
RRF 参数
+
额外延迟
+
更多测试
```

所以它真正的取舍是：

```text
              Hybrid Search

        更好的互补召回
               ↑
               │
     ───────────────────
               │
               ↓
        更高的系统复杂度
```

只有 Dense 和 Sparse 的互补价值足够大时，这笔复杂度成本才值得付。

---

# 十三、今天最终要记住什么

今天不用记住所有代码。

只需要把下面这条链留下来：

```text
                     Query
                       │
           ┌───────────┴───────────┐
           ↓                       ↓
         Dense                   Sparse
      语义相似                    BM25
           │                       │
           ↓                       ↓
       Ranking A               Ranking B
           └───────────┬───────────┘
                       ↓
                      RRF
                  只使用排名融合
                       ↓
                    Final Top-K
```

Dense 解决：

> **“意思像不像？”**

Sparse / BM25 解决：

> **“是不是这个词？”**

RRF 解决：

> **“两套分数不能直接比较，那怎么把两个排名合起来？”**

Hybrid Search 解决的则是更上一层的问题：

> **当真实 Query 同时存在“语义型”和“精确词项型”需求时，单一 Retriever 的失败模式无法覆盖全部查询，因此让两种具有互补盲区的 Retriever 并行召回，再进行融合。**

这就是今天真正需要掌握的知识骨架。

---

## 面试时至少要能沿着这条链回答

```text
为什么需要 Hybrid？
 ↓
Dense 有什么盲区？
 ↓
BM25 为什么能补？
 ↓
BM25 为什么不只是关键词 contains？
 ↓
为什么不能直接加两个 score？
 ↓
为什么选择 RRF？
 ↓
RRF 到底做了什么？
 ↓
为什么两路 ID 必须一致？
 ↓
什么时候反而不应该使用 Hybrid？
```

如果这条链能自己解释下来，今天的核心知识就已经掌握了。

至于完整工程代码，不要求闭卷重新写出来。

但应该能够重新找到：

```python
dense()
sparse_search()
rrf_fuse()
hybrid()
```

并根据这四个函数恢复整个系统的数据流。