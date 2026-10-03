---
description: ""
title: "RAG ：检索、生成与两道拒答闸"
draft: false
date: "2026-10-03T09:08:39+08:00"
slug: "rag-retrieve-generate"
categories:
 - Rag
tags:
 - 
image: ""
---


# W8-D3 · RAG ：检索、生成与两道拒答闸

## 零、今天往前跨了两段

```fallback
Day1: 手册 → 200 字一块                    （切）
Day2: 每块 → 向量 → 落盘                    （记）
Day3: 提问 → 向量 → 找最近的块 → 照着书答     （查 + 答）← 今天
Day4-5: 换检索策略 / 优化
Day6: 把它挂成 Agent 的一个工具
```

今天这一段要立住四个能力：

| 能力 | 判据 |
|---|---|
| **可交互问答** | 输入问题 → 输出答案 |
| **检索独立可测** | `retrieve()` 只做 query → chunks，**不碰 LLM**，能脱离生成层单测 |
| **带引用回答** | 答案里标出 [1][2] 对应哪条资料 |
| **拒答机制** | 资料里没有 → 明确说"资料未提及"，而不是编一个 |

**第二行那四个字是关键：独立可测。** 检索和生成必须能分开测——整条链路一起测，出了问题你分不清是"没搜到"还是"搜到了但模型答歪了"。这两个病，诊断方式和修法完全不同。

> 类比：检索是**开卷考试的"翻书"**。你不用背，但必须**翻对页**——翻错页，你抄的答案再工整也是错的。**这就是"检索质量决定回答质量"的直译：生成层救不了检索层。** 模型再强，喂进去的资料是错的，它只会把错的资料组织得更通顺。

---

## 一、三个类比，把三段钉死

**类比 1：检索 = 翻书。** 翻错页 → 抄得再工整也是错的。

**类比 2：生成 = "只许照着书答"。** 不做约束，模型会用**自己脑子里的知识**答——然后你的 RAG 就白做了：它看着像在引用资料，其实在自由发挥。

**类比 3：拒答 = 开卷考但书里没有 → 写"本卷未涉及"。** 考生最常见的失分方式是"书里没有就自己编"，模型的默认倾向也是这样。**所以拒答不是"能力不足"，是必须显式实现的机制。**

---

## 二、开工前的三个预判

### 2.1 生成也需要模型，而且要选"数据出不出门"

Day2 把嵌入放到本地了，今天生成同样要一个模型。两条路：

| | 云端 API（如 DeepSeek） | 本地（Ollama） |
|---|---|---|
| 成本 | 按 token（很便宜） | 0 |
| 数据 | **文档内容要发出去** | 全本地 |
| 上手 | 立刻（OpenAI 兼容接口） | 要装 Ollama + 拉模型 |
| 速度 | 快 | 看机器 |

**这里有一个必须摆出来的自相矛盾**：如果 Day2 选本地嵌入的理由之一是"数据不出门"，今天生成却把资料发到云端——**同一份文档，一半留在本地、一半发出去，"数据不出门"就只对了一半。**

这是真的矛盾，不是小事。**文档不敏感，就明确接受它；敏感，今天就该上 Ollama。别糊弄过去。**

### 2.2 检索分数是"距离"：越小越像，方向别搞反

```python
hits = vs.similarity_search_with_score(query, k=4)
# hits = [(Document, distance), ...]      ← distance 越小越像
```

**函数名里写着 `similarity`，返回值却是距离**——这是 Chroma 的历史包袱。方向搞反是**和 Day1 那个 `endswith` 判据同一类的错**：指标方向搞反，然后为了"让测试变绿"去改本来正确的代码。

**所以第一件事是把它换算成"像的程度"**，让方向和直觉一致：

```python
def to_similarity(distance: float) -> float:
    """hnsw:space=cosine 时 cosine_distance = 1 - cos_sim，所以 sim = 1 - distance"""
    return 1.0 - float(distance)
```

### 2.3 query 前缀必须和建库时一致

如果嵌入模型（比如 BGE 系列）对 query 有专用前缀，检索时**忘了加**——**不报错，只是召回变差**。典型的静默劣化：测试全绿，答案慢慢变烂。

---

## 三、段④ 检索：`retrieve()` 为什么必须"独立可测"

```python
import re
from pathlib import Path

import build_index as bi            # 复用 Day2 的嵌入 / 落盘 / 防线
from langchain_chroma import Chroma

DEFAULT_K = 4
DEFAULT_MIN_SIM = 0.35              # ★ 临时值，必须用标定脚本换成实测值（见第七节）


def to_similarity(distance: float) -> float:
    """Chroma 返回【距离】，越小越像；换算成【相似度】，越大越像。

    hnsw:space=cosine 时 cosine_distance = 1 - cos_sim，
    所以 sim = 1 - distance（大致落在 [0, 1]，异常时可能到 2）。
    换库时若 space 不是 cosine，这个换算不成立——靠自检里
    "自己搜自己 distance≈0" 来验证方向没反。
    """
    return 1.0 - float(distance)


def retrieve(query: str, k: int = DEFAULT_K, vs: Chroma | None = None):
    """段④：query → [(Document, similarity)]

    ★ 这个函数【不碰 LLM】，所以能脱离生成层单独测——
      "独立可测"靠的就是这条边界。
    """
    if vs is None:
        raise ValueError("retrieve() 需要显式传入 Chroma 实例（便于测试时替换）")
    raw = vs.similarity_search_with_score(query, k=k)
    return [(doc, to_similarity(dist)) for doc, dist in raw]


def filter_by_score(hits, min_sim: float):
    return [(d, s) for d, s in hits if s >= min_sim]
```

**`retrieve()` 的签名里那个 `vs` 不是多余的**：把向量库变成参数，测试时就能塞假实现、也能在不启动 LLM 的情况下反复调用。

### 3.1 一条断言同时验两件事

```python
def self_test(vs: Chroma):
    """检索层自检——不花钱，每次启动都跑"""
    snap = vs.get(limit=1)
    if not snap["documents"]:
        raise SystemExit("库是空的")
    text = snap["documents"][0]
    probe = text[:30]

    hits = retrieve(probe, k=3, vs=vs)
    for i, (d, s) in enumerate(hits, 1):
        hit = " ← 就是它" if d.page_content[:30] == probe else ""
        print(f"  top{i} sim={s:.4f}{hit}")

    assert hits[0][1] > 0.95, (
        f"自己搜自己相似度只有 {hits[0][1]:.3f}（期望 ≈1.0）\n"
        "  → 要么 hnsw:space 不是 cosine，要么 to_similarity 方向反了"
    )
```

**"拿库里第一条文本的前 30 字去搜它自己"**，理论上相似度应接近 1.0。一个断言覆盖两种病：

- 报 **0.05** → 方向反了（1 - 0.95 的镜像）
- 报 **-0.x** → 空间不是 cosine

**这是最省事的自检**——不需要标注数据，也不需要花一分钱。

---

## 四、段⑤ 生成：两道拒答闸

### 4.1 系统提示词

```python
SYSTEM_PROMPT = """你是知识库问答助手。严格遵守以下规则：

1) 只能依据【资料】回答。不得使用你自己的知识，不得推测、不得补充。
2) 每个事实后面必须标注来源编号，格式如 [1] 或 [1][2]。
3) 如果【资料】不足以回答问题，只回答这一句：资料未提及。不要解释，不要猜测。
4) 回答简洁直接，不要复述这些规则。
"""
```

第 1 条治"自由发挥"，第 2 条让答案可溯源，第 3 条是模型侧的兜底拒答，第 4 条防它把规则念一遍。

### 4.2 编号上下文：引用的载体

```python
def build_context(hits) -> str:
    """把 chunks 编号成 [1][2][3]——编号是引用溯源的载体"""
    parts = []
    for i, (doc, sim) in enumerate(hits, 1):
        src = Path(str(doc.metadata.get("source", "?"))).name
        off = doc.metadata.get("start_index", "-")
        parts.append(f"[{i}] 来源: {src} 偏移: {off} 相似度: {sim:.3f}\n{doc.page_content}")
    return "\n\n".join(parts)
```

**编号必须显式写在资料前面**——否则模型没有可引用的对象，第 2 条规则就落空了。

### 4.3 两道闸：便宜的前置 + 兜底的后置

```python
def answer(query: str, k: int = DEFAULT_K, min_sim: float = DEFAULT_MIN_SIM,
           vs: Chroma | None = None, llm=None) -> dict:
    """两道闸的完整问答。

    闸1（前置，便宜）：最像的那块都没过阈值 → 直接拒答，【不调模型】
    闸2（后置，兜底）：让模型看资料够不够 → 它自己说"资料未提及"
    """
    raw = retrieve(query, k=k, vs=vs)
    best = raw[0][1] if raw else -1.0

    # ---- 闸1：分数不过线 → 拒答，一次 LLM 都不调 ----
    if not raw or best < min_sim:
        return {
            "status": "refused_by_score",
            "text": "资料未提及。",
            "best": best, "hits": raw, "cites": None,
            "note": f"最高相似度 {best:.3f} < 阈值 {min_sim}",
        }

    hits = filter_by_score(raw, min_sim)
    resp = llm.invoke(build_messages(query, hits))
    text = resp.content if hasattr(resp, "content") else str(resp)
    cites = check_citations(text, len(hits))

    refused = "资料未提及" in text
    return {
        "status": "refused_by_model" if refused else "ok",
        "text": text, "best": best, "hits": hits, "cites": cites,
        "note": ("模型判定资料不足" if refused else
                 (f"无效引用 {cites['invalid']}" if cites["invalid"] else "")),
    }
```

**为什么两道闸都要**：前置阈值只能拦**"完全无关"**，拦不住**"沾边但答不了"**——后者要靠模型看着资料说"不够"。

**为什么闸1 必须在模型之前**：

- **便宜**：没必要为一个注定拒答的问题付 token 和延迟；
- **可测**：阈值是个数字，能写断言；"模型判断得准不准"没法写断言；
- **安全**：**模型在"没资料"时恰恰是它最容易编的时候**——你把资料递到它面前说"资料不足就说未提及"，它仍可能觉得"我其实知道"。**前置阈值不给它这个机会。**

### 4.4 引用校验：能查出什么，查不出什么

```python
CITE_RE = re.compile(r"\[(\d+)\]")


def check_citations(text: str, n_hits: int) -> dict:
    """引用校验：模型会编出不存在的编号（比如只有 3 条资料却引用 [5]）。

    ★ 只能查出【编号越界】，查不出【编号张冠李戴】——后者只能人工看。
      这是验证缺口，不是已解决。
    """
    nums = [int(x) for x in CITE_RE.findall(text)]
    invalid = sorted({x for x in nums if x < 1 or x > n_hits})
    return {"cited": sorted(set(nums)), "invalid": invalid, "has_cite": bool(nums)}
```

**把"这个校验测不到什么"直接写进 docstring**——比事后被它骗一次强。

---

## 五、完整代码 1：`w8/rag_qa.py`

```python
"""
W8-Day3  RAG 第④⑤段：检索 + 生成（带引用 + 拒答）

运行：
    python w8/rag_qa.py                          # 交互式
    python w8/rag_qa.py -q "手册里XX是什么"        # 单问
    python w8/rag_qa.py --k-only -q "XX"          # ★ 只看检索，不调模型
    python w8/rag_qa.py --topk 6 --min-sim 0.45
    set OLLAMA=1 && python w8/rag_qa.py           # 全本地（Ollama），文档不出机器
"""
from __future__ import annotations

import argparse
import os
import re
import sys
from pathlib import Path

os.environ.setdefault("HF_ENDPOINT", "https://hf-mirror.com")   # 必须在 import 前

HERE = Path(__file__).resolve().parent
sys.path.insert(0, str(HERE))

from langchain_chroma import Chroma

import build_index as bi          # 复用 Day2 的嵌入 / 落盘 / 防线

# ---------------------------------------------------------------- 参数
DEFAULT_K = 4
DEFAULT_MIN_SIM = 0.35            # ★ 临时值，用 calibrate_threshold.py 换成实测值
GENERATION_BACKEND = "deepseek"   # deepseek | ollama


# ================== 段④ 检索（独立可测） ==================
def to_similarity(distance: float) -> float:
    """Chroma 返回【距离】，越小越像；换算成【相似度】，越大越像。

    hnsw:space=cosine 时 cosine_distance = 1 - cos_sim，所以 sim = 1 - distance。
    换库时若 space 不是 cosine，这个换算不成立——靠自检里
    "自己搜自己 distance≈0" 来验证方向没反。
    """
    return 1.0 - float(distance)


def retrieve(query: str, k: int = DEFAULT_K, vs: Chroma | None = None):
    """段④：query → [(Document, similarity)]

    ★ 不碰 LLM，所以能脱离生成层单独测。
    """
    if vs is None:
        raise ValueError("retrieve() 需要显式传入 Chroma 实例（便于测试时替换）")
    raw = vs.similarity_search_with_score(query, k=k)
    return [(doc, to_similarity(dist)) for doc, dist in raw]


def filter_by_score(hits, min_sim: float):
    return [(d, s) for d, s in hits if s >= min_sim]


# ================== 段⑤ 生成（带引用 + 拒答） ==================
SYSTEM_PROMPT = """你是知识库问答助手。严格遵守以下规则：

1) 只能依据【资料】回答。不得使用你自己的知识，不得推测、不得补充。
2) 每个事实后面必须标注来源编号，格式如 [1] 或 [1][2]。
3) 如果【资料】不足以回答问题，只回答这一句：资料未提及。不要解释，不要猜测。
4) 回答简洁直接，不要复述这些规则。
"""


def build_context(hits) -> str:
    """把 chunks 编号成 [1][2][3]——编号是引用溯源的载体"""
    parts = []
    for i, (doc, sim) in enumerate(hits, 1):
        src = Path(str(doc.metadata.get("source", "?"))).name
        off = doc.metadata.get("start_index", "-")
        parts.append(f"[{i}] 来源: {src} 偏移: {off} 相似度: {sim:.3f}\n{doc.page_content}")
    return "\n\n".join(parts)


def build_messages(query: str, hits):
    return [
        ("system", SYSTEM_PROMPT),
        ("human", f"【资料】\n{build_context(hits)}\n\n【问题】{query}"),
    ]


CITE_RE = re.compile(r"\[(\d+)\]")


def check_citations(text: str, n_hits: int) -> dict:
    """引用校验：能查【编号越界】，查不出【编号张冠李戴】（只能人工看）。"""
    nums = [int(x) for x in CITE_RE.findall(text)]
    invalid = sorted({x for x in nums if x < 1 or x > n_hits})
    return {"cited": sorted(set(nums)), "invalid": invalid, "has_cite": bool(nums)}


def get_llm():
    if GENERATION_BACKEND == "ollama" or os.getenv("OLLAMA"):
        from langchain_ollama import ChatOllama
        return ChatOllama(model=os.getenv("OLLAMA_MODEL", "qwen2.5:7b"), temperature=0)

    from langchain_openai import ChatOpenAI
    key = os.getenv("DEEPSEEK_API_KEY")
    if not key:
        raise SystemExit(
            "\n没找到 DEEPSEEK_API_KEY。\n"
            "  用云端：  set DEEPSEEK_API_KEY=sk-xxxx\n"
            "  用本地：  set OLLAMA=1（需先装 Ollama 并 ollama pull qwen2.5:7b）\n"
            "  只看检索：加 --k-only，不调模型\n"
        )
    return ChatOpenAI(model="deepseek-chat", api_key=key,
                      base_url="https://api.deepseek.com/v1",
                      temperature=0)      # 0 提高确定性，但 LLM 输出仍非完全确定


def answer(query: str, k: int = DEFAULT_K, min_sim: float = DEFAULT_MIN_SIM,
           vs: Chroma | None = None, llm=None) -> dict:
    """两道闸的完整问答：闸1 前置拒答（不调模型），闸2 模型兜底拒答"""
    raw = retrieve(query, k=k, vs=vs)
    best = raw[0][1] if raw else -1.0

    if not raw or best < min_sim:                 # ---- 闸1 ----
        return {
            "status": "refused_by_score",
            "text": "资料未提及。",
            "best": best, "hits": raw, "cites": None,
            "note": f"最高相似度 {best:.3f} < 阈值 {min_sim}",
        }

    hits = filter_by_score(raw, min_sim)
    resp = llm.invoke(build_messages(query, hits))
    text = resp.content if hasattr(resp, "content") else str(resp)
    cites = check_citations(text, len(hits))

    refused = "资料未提及" in text
    return {
        "status": "refused_by_model" if refused else "ok",
        "text": text, "best": best, "hits": hits, "cites": cites,
        "note": ("模型判定资料不足" if refused else
                 (f"无效引用 {cites['invalid']}" if cites["invalid"] else "")),
    }


# ================== 展示 ==================
def show_hits(hits, title="检索结果"):
    print(f"\n--- {title} ---")
    for i, (doc, sim) in enumerate(hits, 1):
        src = Path(str(doc.metadata.get("source", "?"))).name
        snippet = doc.page_content.replace("\n", " ")[:60]
        print(f"  [{i}] sim={sim:.3f}  {src}  {snippet}...")


def show_result(r):
    show_hits(r["hits"])
    print(f"\n--- 回答 ---\n{r['text']}")
    print(f"\n状态: {r['status']}   最高相似度: {r['best']:.3f}"
          + (f"   {r['note']}" if r["note"] else ""))
    if r.get("cites"):
        c = r["cites"]
        print(f"引用编号: {c['cited'] if c['cited'] else '（没有标引用！）'}")
        if c["invalid"]:
            print(f"⚠️ 无效引用（资料里没有这些编号）: {c['invalid']}  ← 模型编的")


def load_vs(kind: str) -> Chroma:
    info = bi.read_info()
    if not info:
        raise SystemExit("没找到索引信息文件 → 先跑 python w8/build_index.py --reset")
    bi.guard_backend(kind)        # ★ 模型对不上就不让启动（Day2 的防线，今天继续用）
    emb = bi.get_embeddings(kind)
    vs = Chroma(collection_name=bi.COLLECTION,
                embedding_function=emb,
                persist_directory=bi.PERSIST_DIR)
    print(f"[加载] 模型={info.get('model_name')} dim={info.get('dim')} 条数={info.get('count')}")
    return vs


# ================== 自检 ==================
def self_test(vs: Chroma):
    """检索层自检——不花钱，每次启动都跑"""
    print("\n" + "=" * 60 + "\n检索层自检\n" + "=" * 60)
    snap = vs.get(limit=1)
    if not snap["documents"]:
        raise SystemExit("库是空的")
    text = snap["documents"][0]
    probe = text[:30]

    hits = retrieve(probe, k=3, vs=vs)
    for i, (d, s) in enumerate(hits, 1):
        hit = " ← 就是它" if d.page_content[:30] == probe else ""
        print(f"  top{i} sim={s:.4f}{hit}")

    assert hits[0][1] > 0.95, (
        f"自己搜自己相似度只有 {hits[0][1]:.3f}（期望 ≈1.0）\n"
        "  → 要么 hnsw:space 不是 cosine，要么 to_similarity 方向反了"
    )
    print("  → 自己搜自己 ≈ 1.0 ✅（方向没反 + cosine 空间成立）")


# ================== 主流程 ==================
def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--embed", default="bge-small")
    ap.add_argument("--topk", type=int, default=DEFAULT_K)
    ap.add_argument("--min-sim", type=float, default=DEFAULT_MIN_SIM)
    ap.add_argument("-q", "--question")
    ap.add_argument("--k-only", action="store_true",
                    help="只跑检索不调模型（= 检索层的独立可测接口）")
    args = ap.parse_args()

    vs = load_vs(args.embed)
    self_test(vs)

    k, min_sim = args.topk, args.min_sim
    llm = None if args.k_only else get_llm()

    def once(q):
        if args.k_only:
            hits = retrieve(q, k=k, vs=vs)
            show_hits(hits, f"检索结果 (top-{k}, 不调模型)")
            ok = [(d, s) for d, s in hits if s >= min_sim]
            print(f"\n超过阈值 {min_sim} 的: {len(ok)}/{len(hits)} 块")
            print("→ 若过线为 0，正式流程会走【拒答】")
            return
        show_result(answer(q, k=k, min_sim=min_sim, vs=vs, llm=llm))

    if args.question:
        once(args.question)
        return

    print("\n输入问题; :k N 改 topk, :sim X 改阈值, :q 退出\n")
    while True:
        try:
            q = input("问> ").strip()
        except (EOFError, KeyboardInterrupt):
            print()
            break
        if not q:
            continue
        if q in (":q", "quit", "exit"):
            break
        if q.startswith(":k "):
            k = int(q[3:]); print(f"topk = {k}"); continue
        if q.startswith(":sim "):
            min_sim = float(q[5:]); print(f"min_sim = {min_sim}"); continue
        once(q)


if __name__ == "__main__":
    main()
```

### 5.1 这段复用了 Day2 的哪些接口

`rag_qa.py` 不重建索引，它只**读**Day2 留下的东西。依赖的接口一共五个：

| 接口 | 作用 |
|---|---|
| `bi.COLLECTION` | collection 名（读写必须一致，否则读的是另一个库） |
| `bi.PERSIST_DIR` | 落盘目录 |
| `bi.get_embeddings(kind)` | 造嵌入器（**必须和建库时同一个类**，否则 query 前缀对不上） |
| `bi.guard_backend(kind)` | 校验"当前要用的模型"和"建库时用的模型"是否一致，不一致直接拒绝启动 |
| `bi.read_info()` | 读索引信息（模型名、维度、条数），启动时打印出来对账 |

**`guard_backend` 这一条值得单独说**：它是 Day2 留下的防线——如果检索时用的嵌入模型和建库时不是同一个，**它不让程序启动**。因为这种情况**不会报错，只会让召回悄悄变差**（第二节说的"静默劣化"）。

> 把"模型一致性"做成启动检查，而不是靠人记得——**这类检查成本极低，收益却是一次事故。**

---

---

## 六、完整代码 2：`w8/test_rag.py`

```python
"""
W8-Day3 检索层断言 —— 全部离线，不花一分钱，不需要 API Key。
生成层【无法自动断言】，见文件末尾的说明。
运行：python w8/test_rag.py
"""
import sys
from pathlib import Path

HERE = Path(__file__).resolve().parent
sys.path.insert(0, str(HERE))

import build_index as bi
import rag_qa as rq
from langchain_chroma import Chroma

CASES = []


def case(fn):
    CASES.append(fn)
    return fn


VS = None


def vs():
    global VS
    if VS is None:
        VS = Chroma(collection_name=bi.COLLECTION,
                    embedding_function=bi.get_embeddings("bge-small"),
                    persist_directory=bi.PERSIST_DIR)
    return VS


# ---------- 假的 LLM：记录调用次数，用来证明"拒答时不调模型" ----------
class FakeLLM:
    def __init__(self, body="这是资料里的答案 [1]"):
        self.body, self.calls = body, []

    def invoke(self, messages):
        self.calls.append(messages)

        class M:
            pass

        m = M()
        m.content = self.body
        return m


class BoomLLM:
    """一被调用就炸——用来证明某条路径【不该走到模型】"""

    def invoke(self, messages):
        raise AssertionError("这条分支不该调用模型！")


@case
def t_to_similarity_direction():
    """方向：距离越小 → 相似度越大。搞反了后面全是错的。"""
    assert rq.to_similarity(0.0) == 1.0
    assert rq.to_similarity(0.2) > rq.to_similarity(0.8)
    print("      0 距离→1.0，0.2 距离比 0.8 距离更相似 ✅")


@case
def t_self_search_tops():
    """自己搜自己：必须排第一，且相似度≈1 —— 同时验方向 + 空间"""
    text = vs().get(limit=1)["documents"][0]
    probe = text[:30]
    hits = rq.retrieve(probe, k=3, vs=vs())
    print(f"      top1 sim={hits[0][1]:.4f}（期望≈1.0）")
    assert hits[0][1] > 0.95, f"self-search 只到 {hits[0][1]:.3f} → 方向或空间有问题"
    assert any(d.page_content[:30] == probe for d, _ in hits), "自己的块没进 top3"


@case
def t_topk_grows_monotonically():
    """k 变大 → 结果不减少，且 top-1 不变。
    防的是"调 k 把最好那条挤掉了"这种直觉错误。"""
    text = vs().get(limit=1)["documents"][0]
    q = text[:30]
    h2 = rq.retrieve(q, k=2, vs=vs())
    h6 = rq.retrieve(q, k=6, vs=vs())
    print(f"      k=2→{len(h2)}条  k=6→{len(h6)}条  top1: {h2[0][1]:.4f} vs {h6[0][1]:.4f}")
    assert len(h6) >= len(h2), "k 变大结果反而变少 → 实现有问题"
    assert abs(h2[0][1] - h6[0][1]) < 1e-6, "k 变了 top-1 也变了 → 排序不稳定"


@case
def t_irrelevant_query_scores_lower():
    """★ 对照实验：无关问题的最高分，必须低于相关问题的最高分。
    没有这条，阈值就无从谈起——你得先证明"分数能区分好坏"。"""
    text = vs().get(limit=1)["documents"][0]
    good = rq.retrieve(text[:30], k=1, vs=vs())[0][1]
    bad = rq.retrieve("红烧肉怎么炖才软烂", k=1, vs=vs())[0][1]
    print(f"      相关问题 top1 sim={good:.4f}   无关问题 top1 sim={bad:.4f}   "
          f"差距={good - bad:.4f}")
    assert good > bad, "无关问题分更高 → 阈值无法区分，检索层是坏的"


@case
def t_refusal_is_before_llm():
    """★★ 拒答的核心：阈值卡死 → 拒答，且【一次模型都不调】。
    用 BoomLLM（一调就炸）证明这个分支真的前置。"""
    llm = BoomLLM()
    r = rq.answer("红烧肉怎么炖才软烂", k=4, min_sim=1.1, vs=vs(), llm=llm)
    print(f"      status={r['status']}  text={r['text']!r}  模型调用=0")
    assert r["status"] == "refused_by_score", f"该拒答却走了 {r['status']}"
    assert "资料未提及" in r["text"]


@case
def t_model_refusal_path():
    """闸2：分数过线但模型说资料不足 → 状态也是拒答，别和闸1 混成一类"""
    r = rq.answer("红烧肉怎么炖才软烂", k=4, min_sim=-1.0, vs=vs(),
                  llm=FakeLLM("资料未提及。"))
    print(f"      status={r['status']}")
    assert r["status"] == "refused_by_model"


@case
def t_citation_check_catches_invented():
    """引用校验：只有 3 条资料却引用 [5] → 必须被认出来"""
    c = rq.check_citations("答案见 [1] 和 [5]", n_hits=3)
    print(f"      cited={c['cited']} invalid={c['invalid']}")
    assert c["invalid"] == [5], "编造的引用没被识别"
    assert rq.check_citations("没有任何引用", 3)["has_cite"] is False
    # ★ 已知缺口：张冠李戴查不出来
    assert rq.check_citations("咖啡偏好见 [2]", 3)["invalid"] == []


@case
def t_fake_llm_gets_numbered_context():
    """塞给模型的资料必须带 [1][2] 编号——否则模型没法引用"""
    hits = rq.retrieve(vs().get(limit=1)["documents"][0][:30], k=3, vs=vs())
    msgs = rq.build_messages("测试问题", hits)
    body = msgs[1][1]
    for i in range(1, len(hits) + 1):
        assert f"[{i}]" in body, f"资料里缺编号 [{i}]"
    print(f"      资料编号 [1]..[{len(hits)}] 齐全 ✅")


def main():
    print(f"共 {len(CASES)} 个断言\n")
    passed, failed = 0, []
    for fn in CASES:
        try:
            fn()
            print(f"PASS {fn.__name__}")
            passed += 1
        except Exception as e:
            print(f"FAIL {fn.__name__}\n     {type(e).__name__}: {e}")
            failed.append(fn.__name__)
    print(f"\n{passed}/{len(CASES)} 通过")
    if failed:
        print("失败: " + ", ".join(failed))

    print("\n--- 本文件【测不到】的 ---")
    print("  · 引用编号是否【张冠李戴】（只查越界）")
    print("  · 答案是否忠实于资料（模型可能答得通顺但偏离）")
    print("  · 拒答的【比例】是否合理（全靠阈值标定）")
    print("  → 这三项必须人工看 3~5 个真实问题。别当成已通过。")
    sys.exit(1 if failed else 0)


if __name__ == "__main__":
    main()
```

### 6.1 为什么要造一个"一调就炸"的假模型

`BoomLLM` 的 `invoke()` 直接抛异常。如果拒答逻辑写在模型**之后**，这条测试必炸。**它证明的是"前置"这个性质，而不是"结果看着对"。**

这是"验证设计"里很实用的一招：**当一个性质无法从输出上直接观察时，就构造一个"违反它就会爆炸"的输入。**

---

## 七、完整代码 3：`w8/calibrate_threshold.py`

`DEFAULT_MIN_SIM = 0.35` 是**临时值**。不标定它，拒答机制就是**靠猜在工作**。

```python
"""
把【拍】的阈值换成【算】的。

做法：拿"该答的"和"该拒的"各若干问题，各跑一遍检索，看两组分数的分布，
      找一个能分开两堆的切点。
运行：python w8/calibrate_threshold.py
"""
import statistics as st
import sys
from pathlib import Path

HERE = Path(__file__).resolve().parent
sys.path.insert(0, str(HERE))

import build_index as bi
import rag_qa as rq
from langchain_chroma import Chroma

# ★★ 照着 w8/docs/ 里的文档，手抄 4~5 个"文档里能答出来"的问题
GOOD_Q = [
    "把这里换成手册里真有的问题1",
    "手册里真有的问题2",
    "手册里真有的问题3",
    "手册里真有的问题4",
]

# 这些是"文档肯定答不出"的——拒答的对照组
BAD_Q = [
    "红烧肉怎么炖才软烂",
    "今天北京天气怎么样",
    "Python 的 GIL 是什么",
    "如何申请信用卡",
    "周杰伦最新专辑叫什么",
]


def main():
    vs = Chroma(collection_name=bi.COLLECTION,
                embedding_function=bi.get_embeddings("bge-small"),
                persist_directory=bi.PERSIST_DIR)

    def best_scores(qs):
        out = []
        for q in qs:
            top = rq.retrieve(q, k=4, vs=vs)
            out.append(top[0][1] if top else -1.0)
        return out

    g = best_scores(GOOD_Q)
    b = best_scores(BAD_Q)

    print("【该答的】问题 → top1 相似度")
    for q, s in zip(GOOD_Q, g):
        print(f"  {s:.4f}  {q}")
    print("\n【该拒的】问题 → top1 相似度")
    for q, s in zip(BAD_Q, b):
        print(f"  {s:.4f}  {q}")

    print(f"\n该答组: min={min(g):.4f}  median={st.median(g):.4f}  max={max(g):.4f}")
    print(f"该拒组: min={min(b):.4f}  median={st.median(b):.4f}  max={max(b):.4f}")

    gap_lo, gap_hi = max(b), min(g)
    print(f"\n该拒组的最高分 = {gap_lo:.4f}")
    print(f"该答组的最低分 = {gap_hi:.4f}")

    if gap_lo < gap_hi:
        mid = (gap_lo + gap_hi) / 2
        print(f"\n✅ 两组可分。切点取中位 → min_sim ≈ {mid:.3f}")
        print(f"   理由：该拒组全在 {gap_lo:.3f} 以下，该答组全在 {gap_hi:.3f} 以上。")
        print(f"   ★ 把 rag_qa.py 的 DEFAULT_MIN_SIM 改成 {mid:.2f}")
    else:
        print(f"\n❌ 两组【不可分】：该拒组最高分 {gap_lo:.3f} 反而 >= "
              f"该答组最低分 {gap_hi:.3f}")
        print("   含义：单靠绝对阈值分不开。可能原因：")
        print("     · 该答组里有问题文档里其实答不了（数据错）")
        print("     · 该拒组里有问题和文档内容意外沾边（选了不合适的对照）")
        print("     · 阈值这一招不够 → 需要 rerank 或混合检索（W8 后段）")
        print("   ★ 别硬凑一个数字让它'看起来能分'——那是自欺。")


if __name__ == "__main__":
    main()
```

**这个脚本最关键的地方是最后的 `else` 分支**：如果两组**分不开**，它**明确说"分不开"**，而不是硬挤一个数字。

因为最容易发生的事是：你调半天，终于找到个让 8/9 都对的阈值，然后以为搞定了——**实际上那可能只是过拟合到你自己挑的那 9 个问题上。**

---

## 八、跑的顺序：先测确定性，再测智能性

```fallback
# ① 先只跑检索，不花一分钱
python w8/rag_qa.py --k-only -q "手册里真有的问题"

# ② 离线断言（不花钱、不用 key）
python w8/test_rag.py

# ③ 标定阈值（把拍的 0.35 换成算的）
python w8/calibrate_threshold.py
#   → 按输出改 rag_qa.py 里的 DEFAULT_MIN_SIM，重跑 ②

# ④ 设 key，跑生成
set DEEPSEEK_API_KEY=sk-xxxx
python w8/rag_qa.py -q "手册里真有的问题"
python w8/rag_qa.py -q "红烧肉怎么炖才软烂"     # 应该拒答
```

**第 ① 步是今天最重要的设计**：`--k-only` 让你**在花任何 token 之前**先确认检索是对的。顺序永远是——**先测确定性，再测智能性。**

> ⚠️ 注意 `--k-only` 和 `test_rag.py` 不是一回事：前者只是"打印过线数量"，后者才是真正的独立断言集。**别把两个当一回事。**

---

## 九、三个值得想清楚的问题

### 9.1 top-k 调大调小分别有什么影响

| k | 后果 |
|---|---|
| 太小（1~2） | **漏召回**：答案在某块里，但没进 top-k → 模型看不到 → 拒答 |
| 太大（10+） | **噪声进 prompt**：弱相关的块挤进上下文，模型容易被带偏、甚至引用错块；token 和延迟一起涨 |

还有一个副作用是**"中间塌陷"**：上下文太长时，**夹在中间的资料容易被模型忽略**（这一点属于经验判断，可以用 `:k 2` vs `:k 12` 实测对比）。

**一句话：k 是在"漏"和"杂"之间权衡，不是越大越好。** 而且它不该是拍的——把 k 换成 1/2/4/8 各跑一遍，看不同的召回与噪声。

### 9.2 引用溯源为什么是 RAG 的关键优势

三层价值，一层比一层硬：

1. **可核验**：用户能自己翻回原文——**这是对抗幻觉唯一有效的手段**（没有出处的陈述，你没法验证）；
2. **可定位故障**：答错了，看引用就知道是**检索错**（引用了不相关的块）还是**生成错**（引用的块是对的，但答案歪了）。**没有引用，这两种病分不出来**；
3. **暴露边界**：引用少 = 这个答案根基浅，用户自己会打折扣。

**第 2 层最容易被忽略**——它把"答错了"从一团模糊变成两个可分别诊断的问题。

### 9.3 拒答为什么要"没检索到就不调模型"

- **便宜**：没必要为一个注定拒答的问题付 token 和延迟；
- **可测**：阈值是个**数字**，能写断言；"模型判断得准不准"**没法写断言**；
- **安全**：模型在"没资料"时恰恰最容易编——递资料给它说"不足就说未提及"，它仍可能觉得"我其实知道"。**前置阈值不给它这个机会。**

**但要说清楚**：前置阈值只能拦**"完全无关"**，拦不住**"沾边但答不了"**。所以两道闸都要——**闸1 是确定性的门，闸2 是智能性的门。**
