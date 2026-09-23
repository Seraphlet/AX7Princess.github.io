---
description: ""
title: "W8-D1 · RAG 地基代码详解：加载与分块"
draft: false
date: "2026-09-23T08:24:34+08:00"
slug: "rag-load-split"
categories:
 - RAG
tags:
 - split
image: ""
---

# W8-D1 · RAG 地基代码详解：加载与分块

## 零、今天这份代码在做什么

RAG 全链路七段，今天只做前两段：

```fallback
① 加载 ──► ② 分块 ──► ③ 嵌入 ──► ④ 入库 ──► ⑤ 检索 ──► ⑥ 生成
 Load      Split      Embed      Store      Retrieve    Generate
 └─今天─┘
```

一份文件 `w8/day1_load_split.py`，结构是这样的：

```fallback
day1_load_split.py
├── 配置区        CHUNK_SIZE / CHUNK_OVERLAP / SEPARATORS / HEADERS
├── 加载段        load_text_file / load_pdf_file / check_text_layer
├── 分块段        make_splitter / split_documents / split_one_text
│                split_by_headers / split_markdown_two_stage / drop_tiny_chunks
├── 度量段        measure_actual_overlap / expansion_ratio / formula_expansion
│                cut_at_punctuation
├── 报告段        report_split
├── 样例文档       SAMPLE_MD / SAMPLE_TXT / ensure_sample_docs
├── 三种模式      run_demo / run_real / run_compare
└── 入口          if __name__ == "__main__"
```

```fallback
跑法：
  python w8/day1_load_split.py            # demo：自动造样例，0 准备
  python w8/day1_load_split.py real        # real：读 w8/docs/ 里的真实文件
  python w8/day1_load_split.py compare     # compare：chunk_size 选型对照
```

下面按这份结构逐块讲：**代码 → 这块的坑 → 可优化 → 方法**。

---

## 一、配置区：那几个拍脑袋的数字

```python
from pathlib import Path

HERE = Path(__file__).resolve().parent
DOCS_DIR = HERE / "docs"

CHUNK_SIZE = 300                              # 上限，不是固定值
CHUNK_OVERLAP = 50                            # 约 size 的 15%
OVERLAP_RATIO = 0.15
SEPARATORS = ["\n\n", "\n", "。", "！", "？", "；", "，", " ", ""]
HEADERS = [("#", "h1"), ("##", "h2"), ("###", "h3")]
ENDS = ("。", "！", "？", "；", "，")
```

### 1.1 坑

**坑 1：`SEPARATORS` 结尾必须留 `""`。**

这个空字符串是"实在没辙就一个字一个字切"的**兜底**。如果列表里没有它，遇到"完全没有分隔符"的内容（比如一长串无标点文本、base64 串、压缩包里的字符串），引擎无处可切，**整块直接突破 `chunk_size`**：

```python
text = "A" * 800                                     # 800 字符，零分隔符
no_fb = ["\n\n", "\n", "。", "，"]                    # 故意不留 ""
# → 1 块，长度 800   ❌ 超预算 8 倍
# 加上 [""] 兜底后：
# → 8 块，最长 100   ✅
```

**为什么这条特别值得记**：超预算的块会**安静地**流进后续流程——嵌入时可能超模型输入上限、检索时噪声爆炸，但**任何一步都不会报错**。

**坑 2：中文标点必须自己补。**

`RecursiveCharacterTextSplitter` 的默认 `separators` 是给英文设计的（`["\n\n", "\n", " ", ""]`）。中文段落里**没有空格**，只给这些等于"退无可退就按字符硬切"，句子会被从中间砍断。

**坑 3：`CHUNK_SIZE` / `CHUNK_OVERLAP` 是经验值，不是实测值。**

300 来自"中文常见段落约 200-400 字"这个行业感觉。**它不是针对你的语料测出来的。** 合同/法条（一句话就是要点）可能太大，FAQ（一问一答很短）可能太小——所以这份代码里专门有一个 `compare` 模式来还这笔账。

### 1.2 可优化

| 现状 | 优化 | 收益 |
|---|---|---|
| 常量散在模块顶层 | 抽成 `@dataclass Config` | 多套配置（不同语料）可切换、便于单测 |
| `ENDS` 与 `SEPARATORS` 里的标点重复写 | 用 `ENDS` 生成 `SEPARATORS` | 单一来源，改一处生效 |
| 中文标点只列了 6 个 | 补 `：`、`」`、`）`、`——` 等 | 更少硬切 |
| `len` 数字符 | 换 token 计数（`tiktoken` / 模型自带） | 中英混合语料下预算更准 |

### 1.3 方法

**配置集中 + 单一来源**：注意 `SEPARATORS` 与 `ENDS` 是有关系的（前者是"切分退让顺序"，后者是"句末标点集合"）。重复维护两处，早晚会不一致——**能推导的就别手写两份**。

---

## 二、加载段

```python
from langchain_core.documents import Document


def load_text_file(path: Path) -> Document:
    """读文本/Markdown。显式 encoding，避免中文乱码"""
    text = path.read_text(encoding="utf-8")
    return Document(page_content=text, metadata={"source": path.name})


def load_pdf_file(path: Path) -> list[Document]:
    """PDF：一页 = 一个 Document，metadata 里带 page"""
    from langchain_community.document_loaders import PyPDFLoader   # pip install pypdf
    return PyPDFLoader(str(path)).load()


def check_text_layer(pages: list[Document], min_chars: int = 20) -> list[int]:
    """扫描件探测：返回疑似没有文字层的页码（空列表 = 正常）"""
    return [i for i, d in enumerate(pages)
            if len(d.page_content.strip()) < min_chars]
```

**整个 RAG 只围绕一个数据结构转**：

```python
Document(
    page_content="这一段正文……",                 # 内容
    metadata={"source": "手册.md", "page": 3}     # 从哪来 ★
)
```

### 2.1 坑

**坑 4：`encoding` 不写，中文可能乱码。**

`read_text()` 不传 encoding 时用的是**平台默认编码**——它随机器/环境而变。同一份代码在自己的机器上正常、换一台就出一堆 `\ufffd`，根源都在这。**读写文本一律显式 `encoding="utf-8"`。**

**坑 5：扫描件会"加载成功但内容为空"。**

PDF 分两种：带文字层的（能直接抽文本）和扫描件（本质是图片）。后者用 `PyPDFLoader` 读出来是**空字符串**——不报错、不抛异常，你后面所有环节都在处理空气。

所以加载完必须**探测一次**，而不是假设：

```python
pages = load_pdf_file(Path("manual.pdf"))
bad = check_text_layer(pages)
if bad:
    print(f"⚠️ 第 {bad} 页疑似扫描件，需要 OCR")
```

**判据是"字符数"而不是"有没有报错"**——加载器成不成功，和有没有文字层没关系。

**坑 6：`metadata` 里只放 `path.name` 会重名。**

两个不同目录下的 `README.md` 会撞成同一个 `source`。做引用回溯时，你分不清是哪一份。

### 2.2 可优化

| 现状 | 优化 | 收益 |
|---|---|---|
| `if suffix == ".pdf" else ...` 的 if/else 分发 | 用注册表 `{".md": ..., ".pdf": ...}` | 加新格式只改一处 |
| 返回 `list[Document]` | 返回 `Iterator[Document]`（生成器） | 大目录不吃内存 |
| `source` 只记文件名 | 记**相对路径** + 文件 `hash` | 避免重名、能判断文件有没有更新 |
| 空内容直接进入流程 | 加载后统一 `if not d.page_content.strip(): skip` | 早失败，别让空气流下去 |
| 无日志 | 用 `logging` 替代 `print` | 可控级别、能进文件 |

### 2.3 方法

**"探测而不是假设"**是这一段的核心思想：`check_text_layer` 只有三行，但它把"这份 PDF 到底能不能用"从**祈祷**变成了**判断**。

**metadata 是契约，不是装饰**：`source` / `page` 在第一天写进去，是因为第六段要"带引用回答"、第四段入库时 metadata 会跟着向量一起存。**第一天没留，后面回填等于整个库重跑。**

---

## 三、分块段

### 3.1 切分器工厂

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter, MarkdownHeaderTextSplitter


def make_splitter(size=CHUNK_SIZE, overlap=CHUNK_OVERLAP, separators=None):
    return RecursiveCharacterTextSplitter(
        chunk_size=size,
        chunk_overlap=overlap,
        separators=separators if separators is not None else SEPARATORS,
        keep_separator=True,
        length_function=len,
        add_start_index=True,
    )
```

四个参数各自的分量：

| 参数 | 作用 | 要点 |
|---|---|---|
| `chunk_size` | 块长**上限** | 引擎会尽量在分隔符处断，不会硬凑到满 |
| `chunk_overlap` | 相邻块重叠 | **是下限，不是精确值**（切分点会吸附到分隔符） |
| `keep_separator` | 分隔符归谁 | `True` → 归**下一块开头**；`"end"` → 归上一块末尾；`False` → 丢弃 |
| `add_start_index` | 记录每块在原文的起始位置 | 做引用、做定位、做判据都靠它 |

**坑 7：`overlap >= chunk_size` 会出问题**（报错或切不动），经验值取 10~15%。

**坑 8：`keep_separator` 的语义会坑掉你的判据。**

`keep_separator=True` 时，标点归**下一块的开头**，所以：

```fallback
块1 = "...到这里就结束了"        ← 结尾是"了"，不是标点
块2 = "。下一句从这里开始..."     ← 开头是句号
```

**只有最后一块可能以标点结尾**——所以"块以句号结尾"这个判据，在结构上就不可达。**判据必须和实现的语义对齐**（3.4 给了解法）。

### 3.2 三种切法

```python
def split_documents(docs, *, size=CHUNK_SIZE, overlap=CHUNK_OVERLAP, separators=None):
    """切多个 Document —— metadata 会自动复制到每一块"""
    return make_splitter(size, overlap, separators).split_documents(docs)


def split_one_text(text: str, *, size=CHUNK_SIZE, overlap=CHUNK_OVERLAP,
                   metadata=None, separators=None) -> list[Document]:
    """切一段纯文本 —— 要手动传 metadata"""
    return make_splitter(size, overlap, separators).create_documents(
        [text], metadatas=[metadata or {}])


def split_by_headers(md_text: str) -> list[Document]:
    """Markdown 一级切：按标题切章节，标题信息进 metadata"""
    sp = MarkdownHeaderTextSplitter(headers_to_split_on=HEADERS, strip_headers=False)
    return sp.split_text(md_text)


def split_markdown_two_stage(md_text: str, *, size=CHUNK_SIZE, overlap=CHUNK_OVERLAP):
    """两级切：先按标题，再按长度（heading metadata 会继承到每块）"""
    return split_documents(split_by_headers(md_text), size=size, overlap=overlap)
```

**`split_documents` 与 `split_one_text` 的关键差别**：前者**继承**原 Document 的 metadata（切完每块都带 `source`），后者**丢弃**（你不传 metadata 就没有）。**这是"metadata 跟着每块走"最容易踩空的地方。**

**两级切为什么值得**：

```fallback
一级：按标题切（# / ## / ###）  → 保证语义边界 + 章节归属
二级：超长章节再按长度递归切    → 保证不超预算
要求：二级切完，一级的 h1/h2/h3 必须还在
```

切完的每一块自带 `h1/h2/h3`——检索出来的块**知道自己属于哪一章**。这在生成阶段直接变成引用来源，在调试阶段直接告诉你是哪一节出了问题。

`strip_headers=False` 让标题留在正文里：块被单独拿出来看，也知道自己讲的是什么。

### 3.3 碎块过滤

```python
def drop_tiny_chunks(docs: list[Document], min_len: int = 10) -> list[Document]:
    """过滤碎块：切分可能产出 1 个字符的块（例如一个孤立句号），进库就是噪声"""
    return [d for d in docs if len(d.page_content.strip()) >= min_len]
```

**坑 9：会切出 1 个字符的块。**

按标点切分时，如果某段的残余正好是一个标点（比如末尾那个 `。`），它会**单独成块**。这种块进向量库是纯噪声——**任何人检索都可能召回一个句号**。

**这是产品缺陷，不是测试问题**：断言不会替你发现它，只有你亲眼扫一遍块列表才会看到。

**坑 10：两级切之后，`start_index` 的坐标基准变了。**

二级切分是**对每个 section 单独做的**，所以每块的 `start_index` 是"相对该 section 开头"的偏移，**不是相对整篇原文**。要做精确定位（比如"引用原文第几段"），需要自己把一级的偏移加回去。

### 3.4 可优化

| 现状 | 优化 | 收益 |
|---|---|---|
| `start_index` 只有一层坐标 | 两级切时累加一级偏移 | 引用能精确到原文位置 |
| `min_len=10` 写死 | 提到配置里 | 不同语料可调 |
| 只按字符判空 | 同时 strip 首尾空白再切 | 少一批"全是空格"的块 |
| 切完不校验 | 加一句"拼接能复原原文"的自检 | 早发现丢字（`keep_separator=False` 会丢） |
| 一次全量切 | 生成器逐块产出 | 大文档内存友好 |

---

## 四、度量段：把"拍脑袋"变成"算账"

```python
def measure_actual_overlap(chunks: list[str]) -> list[int]:
    """实测相邻块的真实重叠字符数"""
    out = []
    for a, b in zip(chunks, chunks[1:]):
        n = 0
        for k in range(min(len(a), len(b)), 0, -1):
            if a[-k:] == b[:k]:
                n = k
                break
        out.append(n)
    return out


def expansion_ratio(original_len: int, chunks: list[str]) -> float:
    """冗余膨胀率 = 所有块字符数 / 原文字符数 - 1"""
    return 0.0 if not original_len else sum(len(c) for c in chunks) / original_len - 1


def formula_expansion(size: int, overlap: int) -> float:
    """理论膨胀率 ≈ overlap / (size - overlap)"""
    return float("inf") if size <= overlap else overlap / (size - overlap)
```

### 4.1 方法：三个数把经验变成证据

**① 实测重叠：`chunk_overlap` 是下限。**

设定 30 字符重叠，实测往往大于 30——因为**切分点会被吸附到分隔符上**。你设的是"至少重叠这么多"，不是"恰好这么多"。

**② 膨胀率：直接乘到嵌入账单上。**

```
50 / (300 - 50) ≈ 20%
```

你多切出来的这 20% 字符，下一篇就变成 20% 的嵌入调用量。**overlap 翻一倍，嵌入成本大致也翻一倍。**

**③ 选型对照：没有"最优 size"，只有取舍。**

`compare` 模式打出来的那张表，读法是：

| 观察 | 含义 |
|---|---|
| 块数涨得快 | 召回的碎片更多、待嵌入的条数更多（贵） |
| 平均块长小 | 单块上下文少，可能丢掉"这句话在讲什么" |
| 膨胀率 | 直接乘到嵌入账单上 |

### 4.2 坐标判据：绕开实现细节

```python
def cut_at_punctuation(chunks: list[Document], original: str,
                       ends: tuple[str, ...] = ENDS) -> float | None:
    """切分点是否落在标点上——用原文坐标判断，与 keep_separator 取值无关。

    keep_separator=True  → 标点在下一块开头（original[o]）
    keep_separator="end" → 标点在上一块末尾（original[o-1]）
    """
    offs = [c.metadata.get("start_index") for c in chunks]
    if len(offs) < 2 or any(o is None for o in offs):
        return None                      # 坐标缺失 → 交给调用方报错
    hit = 0
    for o in offs[1:]:
        if o <= 0:
            continue
        if original[o - 1:o] in ends or original[o:o + 1] in ends:
            hit += 1
    return hit / (len(offs) - 1)
```

**这就是"判据与实现解耦"的写法。** 与其去猜"标点归前块还是后块"，不如**回到原文坐标**：看这个切分点（`start_index`）前后那一个字符是不是标点。

好处有三个：

- **不受 `keep_separator` 取值影响**——改实现不改判据；
- **判据有区分度**——加了中文标点的切分，切分点落在标点上接近 100%；去掉标点的对照组接近 0%；
- **坐标缺失会显式失败**（返回 `None` 而不是 0 分），避免"什么都没测到还显示通过"。

**用切片取值（`original[o-1:o]`）而不是索引取值（`original[o-1]`）**，是为了防越界——切片越界返回空串，索引越界直接抛异常。

### 4.3 可优化

| 现状 | 优化 | 收益 |
|---|---|---|
| `measure_actual_overlap` 从 `min(len(a), len(b))` 全量比对，最坏 O(n²) | 只比对前 `max_overlap*2` 个字符 | 长块下快一个量级 |
| `expansion_ratio` 只按字符 | 同时报 token 口径 | 和嵌入账单口径一致 |
| `cut_at_punctuation` 返回 `float \| None` | 返回一个小的结果对象（比例+命中数+样本） | 排查时能看到"哪几个切分点没命中" |

---

## 五、报告与三种模式

```python
def report_split(label: str, original: str, docs: list[Document]) -> None:
    lens = [len(d.page_content) for d in docs]
    chunks = [d.page_content for d in docs]
    print(f"\n=== {label} ===")
    if not docs:
        print("  0 块（原文为空？）")
        return
    print(f"  原文 {len(original)} 字符 -> {len(docs)} 块")
    print(f"  块长：最短 {min(lens)}  最长 {max(lens)}  平均 {sum(lens) // len(lens)}")
    print(f"  超预算的块：{[l for l in lens if l > CHUNK_SIZE] or '无'}")
    print(f"  切分点落在标点上：{cut_at_punctuation(docs, original)}")
    print(f"  膨胀率 实测 {expansion_ratio(len(original), chunks):.1%}"
          f"   公式预测 {formula_expansion(CHUNK_SIZE, CHUNK_OVERLAP):.1%}")
    print("  --- 前 2 块预览 ---")
    for i, d in enumerate(docs[:2]):
        print(f"  #{i} ({len(d.page_content)}字) "
              f"{d.page_content[:60]}".replace("\n", " ") + "...")
    print(f"  metadata[0] = {docs[0].metadata}")
```

**报告函数不是"打印调试信息"，而是把关键指标固化成每次都能看到的体检单**：块数、块长分布、**超预算的块**、切分点命中率、膨胀率、每块 metadata。

**"超预算的块"这一行要单独列出来**——因为它是唯一一个"静默出错"的指标（3.1 的坑 1）。

三种模式的定位：

| 模式 | 用途 |
|---|---|
| `demo` | 自动造样例文档，**0 准备先证明代码是活的** |
| `real` | 读真实文件，逐文件打印加载结果与 metadata 检查 |
| `compare` | 同一份文本跑三组 `chunk_size`，看成本差异 |

**`demo` 模式是这份代码里我很推荐的一个习惯**：不依赖你手上有任何文档，先跑通再说。**没有 demo 模式的脚本，往往会卡在"我还没有 PDF"这一步。**

### 5.1 坑与可优化

**坑 11：`run_real` 一次把目录全读进内存。**

```python
all_docs: list[Document] = []
for p in files: ...
```

十来个文件没事，几百个就是内存问题。**可优化**：改成生成器逐文件处理、切完即丢。

**可优化**：目录遍历用 `Path.rglob` 收子目录；跳过隐藏文件与大文件；断点续跑时记录已处理文件的 hash。

---

## 六、入口：一行不能少

```python
if __name__ == "__main__":
    mode = sys.argv[1] if len(sys.argv) > 1 else "demo"
    if mode == "demo":
        ensure_sample_docs()
        run_demo()
    elif mode == "real":
        run_real()
    elif mode == "compare":
        run_compare()
    else:
        print(f"未知模式：{mode}（用法：python w8/day1_load_split.py [demo|real|compare]）")
```

**坑 12：漏了 `if __name__ == "__main__":` 就是"零输出"。**

这类脚本常见的事故——文件能跑、不报错、**什么都不打印**。原因就是入口没执行：模块被 import 时上面的函数只定义不调用，你没有入口就等于**只声明不执行**。

**坑 13：模式名不做兜底。**

传了个拼错的模式名（`comapre`），如果直接落到 `run_demo` 或静默退出，你会一脸疑惑"怎么没输出"。**未知输入要显式报错并打印用法**——错误要响，不要哑。

---

## 七、坑位汇总（对着自查）

| # | 坑 | 后果 | 防线 |
|---|---|---|---|
| 1 | `SEPARATORS` 结尾没留 `""` | 无分隔符内容整块超预算（**静默**） | 保留 `""`；报告里单列"超预算的块" |
| 2 | 中文标点没进 `separators` | 长句被按字符硬切，语义断裂 | `SEPARATORS` 含 `。！？；，` |
| 3 | `CHUNK_SIZE/OVERLAP` 抄经验值 | 与语料不匹配 | `compare` 模式算给你看 |
| 4 | `encoding` 不写 | 中文乱码（换机器才暴露） | 一律 `encoding="utf-8"` |
| 5 | 扫描件静默空内容 | 后续环节全在处理空气 | `check_text_layer` 探测 |
| 6 | `metadata` 只记文件名 | 重名文件撞 `source` | 记相对路径 + hash |
| 7 | `overlap >= chunk_size` | 报错或切不动 | 取 10~15%，并断言它真报错 |
| 8 | 用 `endswith("。")` 当判据 | **结构上不可达**，且指标反向 | 用 `start_index` 坐标判据 |
| 9 | 切出 1 字符碎块 | 进库就是噪声 | `drop_tiny_chunks` 过滤 |
| 10 | 两级切后 `start_index` 基准变了 | 引用定位错位 | 累加一级偏移 |
| 11 | `run_real` 全量读入内存 | 大目录吃光内存 | 生成器逐文件处理 |
| 12 | 漏 `__main__` 入口 | 零输出 | 入口 + 未知模式报错 |
| 13 | `split_one_text` 忘了传 metadata | 块没有 `source` | 优先用 `split_documents`（自动继承） |

---

## 八、这份代码里三个可复用的方法

**方法一：探测而不是假设。**

`check_text_layer` 只有三行，但思路值得抄——**凡是"外部输入可能悄悄为空/失效"的地方，都值得加一个显式探测**：扫描件、空文件、编码不对、被截断的下载。它们的共同点是**不报错**。

**方法二：把经验值变成度量。**

`300` / `50` / `0.15` 这些数，写死在任何项目里都会变成"祖传参数"。这份代码的做法是给它们配三样东西：**实测函数**（`measure_actual_overlap`）、**理论公式**（`expansion_ratio`）、**对照实验**（`compare` 模式）。参数还是那几个参数，但从此有依据。

**方法三：判据锚在稳定坐标上，别锚在实现细节上。**

`endswith("。")` 依赖"标点归哪一块"这个实现细节，而 `start_index` 回原文坐标不依赖它。**写判据时问一句：这个判据成立的前提，是"业务语义"还是"当前实现"？** 后者一变，判据就失效——更糟的是可能反向失效（坏数据得分更高）。

---

## 九、速查卡片

**加载**

| 目标 | 写法 |
|---|---|
| 读文本 | `Document(p.read_text(encoding="utf-8"), {"source": p.name})` |
| 读 PDF | `PyPDFLoader(str(p)).load()`（一页一 Doc，带 `page`） |
| 扫描件探测 | `[i for i,d in enumerate(pages) if len(d.page_content.strip()) < 20]` |

**分块**

| 目标 | 写法 |
|---|---|
| 递归切（批量） | `splitter.split_documents(docs)` ← **metadata 自动继承** |
| 递归切（单文本） | `splitter.create_documents([text], metadatas=[md])` |
| Markdown 两级切 | `MarkdownHeaderTextSplitter(...)` → 再 `split_documents` |
| 关键参数 | `chunk_size` 上限 / `chunk_overlap` 下限 / `add_start_index=True` |
| 兜底 | `separators` 结尾留 `""` |
| 过滤碎块 | `[d for d in docs if len(d.page_content.strip()) >= 10]` |

**度量**

| 目标 | 公式 / 方法 |
|---|---|
| 实测重叠 | 逐字符比对"前串后缀 == 后串前缀"的最大 k |
| 膨胀率 | `块字符总数 / 原文字符数 - 1` |
| 理论膨胀率 | `overlap / (chunk_size - overlap)` |
| 切分点质量 | **用 `start_index` 回原文坐标判断**，不查 `endswith` |

**三句口诀**

- **能按句子切，就绝不按字符切**——`separators` 含中文标点，结尾留 `""`。
- **metadata 第一天就要管**——`source` / `page` / `h1-h3` 缺一个，后面就得整库重跑。
- **判据锚在坐标上**——别让实现细节（标点归谁）决定你的指标。

---

## 十、一句话总结 + 下一篇预告

**今天这份代码只做两件事：把文档读成带 `metadata` 的 `Document`（`source`/`page` 第一天就得有、扫描件必须先探测），再把它切成 chunk（递归切分含中文标点并以 `""` 兜底、Markdown 两级切保住章节归属、碎块要过滤）。** 而它真正值得带走的是三个习惯：**探测而不是假设**、**把经验值配上一个度量**、**判据锚在稳定坐标上**。切分是 RAG 的地基——**向量库里存的就是你切出来的那些块，切得烂，检索永远救不回来。**

下一篇 **Day2 · 嵌入 + 入库**：把这些 chunk 变成向量存进 Chroma 并落盘，实现"一次入库、反复复用"，同时把嵌入模型选型与成本算清楚——**今天算出来的膨胀率，到那天就变成真金白银**。

**魔鬼代言人**：这份代码全部跑通，也只证明"结构对、配置生效"，**完全不证明"切得好"**。切得好不好只有你能判：打开输出，读前 5 块，问一句——**"这块单独拿出来，还看得懂它在讲什么吗？"** 如果一块被切成"……因此 Agent 需要"（后半句跑下一块去了），说明 size 或 separators 还得调。**这件事任何断言都测不出来，只能靠眼睛。**

**自查三问**：① `SEPARATORS` 结尾那个 `""` 删掉会发生什么？为什么它是"静默出错"？② `split_documents` 和 `split_one_text` 在 metadata 上有什么差别？这会导致什么常见疏漏？③ 为什么"块以句号结尾"不能当切分质量判据？换成什么判据就不受实现细节影响？