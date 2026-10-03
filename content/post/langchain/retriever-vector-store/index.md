---
description: ""
title: "Retriever / Vector Store"
draft: true
date: "2026-08-28T07:25:20+08:00"
slug: "Retriever-Vector Store"
categories:
 - null
tags:
 - null
image: ""
---


## 翻车前的自白

W3 你写的 `MemoryManager`，长期记忆那一截其实是把事实塞进 Chroma 向量库，再靠 `recall(query, k=3)` 按语义捞回来。那东西能跑、能召回，但它是**你自研 API**——LangChain 的链不认它。

D1 你学了 `|` 传送带，D2 学了 `RunnableParallel / Passthrough / Lambda`，D3 把「对话记忆」接进了链首。今天 D4 要把**另一半资产**也认领进来：**知识召回**，在 LangChain 里它叫 `Retriever`。

一句话：**Vector Store 负责「存」，Retriever 负责「取」，两者接进 LCEL 链，就做出最朴素的 RAG（检索增强生成）。** 你 W3 手写的向量召回，今天变成 `chain` 里的一环——决策逻辑一行不改，只在外面套层标准接口。

---

## 一、大局：今天学什么？

> **Vector Store（仓库）负责"存"，Retriever（取货员）负责"取"，把两者接进 LCEL 链，就做出最朴素的 RAG。**

类比成套了：

- **Vector Store = 超市仓库**：货架上摆满带条码的货物（每条知识 = 一个 `Document`）。
- **Retriever = 取货员**：顾客问"年假政策"，他去仓库按相关度把最像的 2 件货捞出来。
- **RAG = 取货 → 摆桌 → 做菜**：取货员取货（检索）→ 把货摆上桌（拼进 prompt）→ 厨师做菜（生成答案）。

你 W3 的 `LongTermMemory` 就是那个仓库，`recall` 就是那个取货员。今天不是重写，是给它发一张 LangChain 的「工牌」。

---

## 二、原理：五个概念，每个都对应你已经会的东西

### ① Document = 档案袋（内容 + 标签）

Chroma 里存的不是裸字符串，是 `Document(page_content, metadata)`：

```python
Document(page_content="公司年假政策:入职满1年享5天",
         metadata={"部门": "HR", "更新时间": "2026"})
```

**metadata 就是档案袋上的标签**——它对应你 W3 的 `private` 标记！`add_fact(text, fid, private=True)` 里的 `private`，放到 Document 里就是 `metadata={"private": True}`。**概念完全同源**。

### ② Vector Store 的三个动作（这就是"仓库"的完整用法）

| 动作 | 方法 | 类比 |
|---|---|---|
| 存 | `store.add_documents([...])` | 进货入库 |
| 查 | `store.similarity_search(query, k=2)` | 按相关度捞货 |
| 转 | `store.as_retriever(...)` | 给取货员发工作证（变成 LangChain 标准接口） |

### ③ Retriever 在 LCEL 里的接法（D2 知识全用上）

```python
chain = (
    {"context": retriever | _fmt, "question": RunnablePassthrough()}   # ← D2 学的并行+透传+加工
    | prompt
    | model
    | parser
)
```

retriever 的输出**自动塞进 prompt 的 `{context}` 占位符**——你 D2 学的 `RunnableParallel`（并行拿 context 和 question）、`RunnablePassthrough`（原样传问题）、`RunnableLambda`（`_fmt` 加工文档），**今天全用上了**。

### ④ 检索参数：k 和 score_threshold

- `k=2`：召回几条（**Top-K**，对应今天算法题 215）。
- `score_threshold=0.3`：相似度低于这个值的不要（**阈值筛选**，对应算法题 704）。

### ⑤ 最重要：和你 W3 的记忆同源

> **RAG 的"知识库"本质就是 W3 的"长期向量记忆（Chroma）"——今天练的 retriever，之后可以直接复用 W3 的 `LongTermMemory`，不必另起炉灶。**

你 W3 的 `LongTermMemory.recall(query, k=3)` 就是"取货员"，只是它返回 `list[str]`，LangChain 要的是 `list[Document]`——**包一层就行**。

---

## 三、一个坑先摆平：DeepSeek 没有 embedding API

任务表骨架用了 `OpenAIEmbeddings()`（要 `OPENAI_API_KEY`）——**你没有这个 key，只有 DeepSeek**。而 DeepSeek 官方**不提供 embedding 接口**（确认过只有 `deepseek-v4-flash` / `pro` 这类生成模型）。

所以有两个走法，**主走路径 A（零新依赖、立刻能跑）**：

| 路径 | 做法 | 依赖 |
|---|---|---|
| **A. 包装 W3（推荐）** | 用 `RunnableLambda` / `BaseRetriever` 把 `mm.recall` 包成 retriever，返回 `Document` | 无新增！W3 已配好 Chroma |
| **B. 独立 langchain_chroma** | `Chroma.from_texts` + embedding 函数 | 需要本地 embedding 模型（all-MiniLM ~80MB）或另一家 embedding API |

> 为什么选 A：你 W3 已经把 Chroma 落盘、召回跑通了，再为 embedding 另装 80MB 模型纯属重复造轮子。路径 B 适合你想单独练"仓库+取货员分开"的标准写法时才上。

---

## 四、实操 · 步骤 A/B：跑通最小检索 + 接进 LCEL 做 RAG

建 `W4/lc_retriever_w3.py`（路径 A，复用 W3，立即跑）：

```python
# W4/lc_retriever_w3.py —— 路径A：把 W3 长期记忆包装成 Retriever（复用已跑通的代码）
import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))  # 回到项目根

from langchain_core.documents import Document
from langchain_core.runnables import RunnableLambda
from memory.memory_manager import MemoryManager   # W3 的门面

mm = MemoryManager(window=8, max_tokens=1500, keep=4)
# 先存两条事实（persist=True 落长期 Chroma）
mm.add({"role": "user", "content": "用户喜欢用 DeepSeek，预算 2000 元"}, persist=True)
mm.add({"role": "user", "content": "用户叫小明，喜欢蓝色"}, persist=True)

def make_retriever(mm, k=3):
    """把 mm.recall 包成 LangChain 标准 Retriever：query -> list[Document]"""
    def recall_docs(query: str) -> list[Document]:
        facts = mm.recall(query, k=k)              # W3 已跑通的语义召回
        return [Document(page_content=f,
                         metadata={"source": "long_term"}) for f in facts]
    return RunnableLambda(recall_docs)

retriever = make_retriever(mm, k=3)

# 验收A：肉眼确认"相似片段被正确召回"
docs = retriever.invoke("用户喜欢什么模型？")
for d in docs:
    print("召回:", d.page_content)
```

**验收 A 标准**：打印出含"DeepSeek"的召回结果——相似片段被正确召回，步骤 A 完成。

> 实际跑时模型那部分（步骤 B）需要 `DEEPSEEK_API_KEY`。下面把步骤 B 接上：

```python
# 步骤B：接进 LCEL 链做最朴素 RAG
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_openai import ChatOpenAI
from dotenv import load_dotenv

load_dotenv()
model = ChatOpenAI(model="deepseek-v4-flash",
                   api_key=os.getenv("DEEPSEEK_API_KEY"),
                   base_url="https://api.deepseek.com")

prompt = ChatPromptTemplate.from_template(
    "根据已知信息回答。\n已知:{context}\n问题:{question}\n回答:"
)

def _fmt(docs):
    return "\n".join(d.page_content for d in docs)

chain = (
    {"context": retriever | _fmt, "question": RunnablePassthrough()}   # 并行取数
    | prompt
    | model
    | StrOutputParser()
)

q = "用户喜欢什么模型？预算多少？"
print("问:", q)
print("答:", chain.invoke(q))
```

**验收 B 标准**：模型基于召回内容答出"DeepSeek / 2000"——RAG 链能基于召回内容回答。

### （可选）步骤 C：独立 langchain_chroma 版（练"标准任务骨架"）

如果你想练任务表标准版（仓库 + 取货员分开），用这个骨架——**注意 embedding 的坑**（需 `pip install langchain-community sentence-transformers`，首次下载模型会卡一会）：

```python
# W4/lc_retriever_demo.py —— 路径B：独立最小 Chroma
from langchain_chroma import Chroma
from langchain_community.embeddings import HuggingFaceEmbeddings  # 本地模型，免 API key
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
import os
from dotenv import load_dotenv

load_dotenv()

docs = [
    "公司年假政策:入职满 1 年享 5 天,满 3 年享 10 天。",
    "报销流程:发票上传财务系统,直属领导审批后 3 个工作日到账。",
    "IT 工单:内网提交,紧急故障可拨打 8000 转 1。",
]

embeddings = HuggingFaceEmbeddings(model_name="all-MiniLM-L6-v2")  # 首次下载~80MB
store = Chroma.from_texts(
    docs, embeddings,
    collection_name="w4_demo",
    persist_directory=os.path.join(os.path.dirname(__file__), "mem_store"),
)
retriever = store.as_retriever(search_kwargs={"k": 2})

model = ChatOpenAI(model="deepseek-v4-flash",
                   api_key=os.getenv("DEEPSEEK_API_KEY"),
                   base_url="https://api.deepseek.com")
prompt = ChatPromptTemplate.from_template("根据已知信息回答。\n已知:{context}\n问题:{question}\n回答:")

def _fmt(docs):
    return "\n".join(d.page_content for d in docs)

chain = ({"context": retriever | _fmt, "question": RunnablePassthrough()}
         | prompt | model | StrOutputParser())

print(chain.invoke("我入职两年,年假有几天?"))   # 期望答出"5天"
```

---

## 五、验收清单（对照任务表）

| 验收项 | 你的证据 |
|---|---|
| 相似片段被 retriever 正确召回 | 步骤A：`retriever.invoke("喜欢什么模型")` 打出含 DeepSeek 的 Document |
| RAG 链能基于召回内容回答 | 步骤B：答出 DeepSeek / 2000 |
| 调 k / score_threshold 后行为符合预期 | 把 `k` 从 3 改 1 观察召回变少；加 `score_threshold=0.3` 观察过滤 |
| (可选)复用 W3 LongTermMemory | 路径A 就是——零新增依赖直接复用 |

---

## 六、临时工 → 正式工：把 retriever 升级成 BaseRetriever 子类

之前 D4 我们为了快速跑通，用 `RunnableLambda` 手动包了一个"假 retriever"。能用，但那是**临时工**——LangChain 生态不认它。

**剩下的部分 = 把它升级成官方认证的 `BaseRetriever` 子类**——活一样干，但"持证上岗"，整个框架都认识你。

> 类比：临时工和正式工都干活，但正式工有工牌（`BaseRetriever` 这个父类）——门禁（`invoke/stream/batch`）、培训体系（LangGraph 节点、工具封装）都对他开放。

### 为什么要升级（3 个理由，面试能讲）

| 理由 | 说明 |
|---|---|
| **接口白送** | D1 学的 `invoke / stream / batch` 三接口免费获得 |
| **生态识别** | 能被 `create_retriever_tool`、LangGraph 节点、`RetrievalQA` 等标准组件直接使用 |
| **类型安全** | 字段用 pydantic 声明（`mm`、`k`），有类型检查、能序列化保存 |

### MemoryRetriever 代码（正式版）

```python
# w4/retriever.py —— W4-D4 正式版：BaseRetriever 子类
from typing import Any
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from pydantic import Field, ConfigDict


class MemoryRetriever(BaseRetriever):
    """把 W3 的 MemoryManager.recall 包装成官方 Retriever。

    BaseRetriever 是个 pydantic 模型，所以字段要声明类型 + Field。
    """
    model_config = ConfigDict(arbitrary_types_allowed=True)  # 允许装非 pydantic 对象

    mm: Any = Field(description="带 recall(query, k) 方法的记忆管理器")
    k: int = Field(default=3, description="召回条数")

    def _get_relevant_documents(self, query: str, **kwargs) -> list[Document]:
        """唯一必须实现的抽象方法：query → Document 列表。

        **kwargs 兼容新旧版本签名（旧版无 run_manager，新版有）。
        """
        facts = self.mm.recall(query, k=self.k)          # W3 的召回
        return [
            Document(page_content=f, metadata={"source": "长期记忆"})
            for f in facts
        ]
```

### 三个关键点讲透（面试能讲）

**关键点 1：`_get_relevant_documents` 是唯一必须实现的抽象方法**
它回答一句话："给我一句 query，还你相关文档列表。" LangChain 内部所有调用（`invoke` / `stream` / `batch`）最终都会走到这个方法——你只写这一个，剩下的框架替你管。

> 类比：你只负责"**按关键词翻档案柜**"这一个动作（`_get_relevant_documents`），至于"客人怎么下单（invoke）、怎么批量查（batch）、怎么跟踪调用记录"——那是工牌（`BaseRetriever`）自带的，不用你写。

注意签名里的 `**kwargs`——这是**版本兼容保险**：老版本签名是 `_get_relevant_documents(self, query)`，新版本多了 `run_manager` 参数。写 `**kwargs` 两种都能跑，不会踩你 D1 那种 `TextAccessor` 版本坑。

**关键点 2：invoke / stream / batch 白送**
`BaseRetriever` 本身就是一个 Runnable（和 `prompt | model | parser` 里的每个零件一样）——所以 D1 学的三接口直接能用：

```python
docs = retriever.invoke("用户喜欢什么颜色")     # list[Document]
docs_list = retriever.batch(["问题1", "问题2"])  # 批量
```

而且**能直接进 LCEL 链**（D2 刚学的 `RunnableLambda` 用上了）：

```python
from langchain_core.runnables import RunnableLambda

chain = retriever | RunnableLambda(
    lambda docs: "；".join(d.page_content for d in docs)
)
result = chain.invoke("用户喜欢什么模型？")   # 返回纯字符串
```

> 类比：正式工（BaseRetriever）**自带门禁卡**（Runnable 协议），D1 你学的 `invoke/stream/batch` 三孔插头直接能插；临时工（RunnableLambda 手工包）得你每次自己手动接电。

**关键点 3：为什么字段要 `Field` + `ConfigDict`**
`BaseRetriever` 是 pydantic 模型（不是普通类）——所以 `mm` 和 `k` 必须**声明类型 + Field**，否则实例化报错；`ConfigDict(arbitrary_types_allowed=True)` 是告诉 pydantic："`mm` 里装的 MemoryManager 不是 pydantic 对象，但请放行"。

> 类比：正式工的工牌信息表（pydantic）要求"姓名、工号"必须填标准格式；`arbitrary_types_allowed` 就是备注栏"允许填非标准信息"（比如你的 MemoryManager 实例）。

### 怎么替换你之前的 make_retriever（无缝升级）

你之前 D4 前半段用的是临时工版（`RunnableLambda` 手动包）。替换成正式版就三步：

```python
# 之前（临时工）:
# make_retriever = RunnableLambda(lambda q: [Document(page_content=f) for f in mm.recall(q)])

# 现在（正式工）:
from w4.retriever import MemoryRetriever

mm = MemoryManager(window=8, max_tokens=1500, keep=4)   # 不需要 llm，纯召回
retriever = MemoryRetriever(mm=mm, k=3)

# 用法完全一样，甚至更简单：
docs = retriever.invoke("用户喜欢什么颜色")
```

**你已有的链一行不用改**——因为 `retriever` 本身就是 Runnable，之前链里怎么用 `make_retriever` 就怎么用它。

---

## 七、验证 & 进阶：不需要 API key + 接成 Agent 工具

### 验证（用假记忆管理器，不联网）

```python
# test_retriever.py
from w4.retriever import MemoryRetriever

class FakeMemory:
    """模拟 MemoryManager.recall —— 不联网也能测"""
    def recall(self, query, k=3):
        return ["用户叫小明", "喜欢蓝色", "预算2000"]

r = MemoryRetriever(mm=FakeMemory(), k=3)

# 测试1：invoke 返回 list[Document]
docs = r.invoke("用户喜欢什么")
assert isinstance(docs, list) and len(docs) == 3
print(f"✓ invoke 返回 {len(docs)} 条 Document")

# 测试2：page_content 和 metadata 都在
print(f"✓ 内容: {docs[1].page_content} | 来源: {docs[1].metadata['source']}")

# 测试3：能直接进 LCEL 链
from langchain_core.runnables import RunnableLambda
chain = r | RunnableLambda(lambda ds: "；".join(d.page_content for d in ds))
print(f"✓ 链输出: {chain.invoke('用户喜欢什么')}")
```

预期输出：

```
✓ invoke 返回 3 条 Document
✓ 内容: 喜欢蓝色 | 来源: 长期记忆
✓ 链输出: 用户叫小明；喜欢蓝色；预算2000
```

### 进阶：接成 Agent 工具（把 D4 和 W2/D5 串起来）

正式版 Retriever 的**最大红利**在这里——LangChain 官方有标准工具封装：

```python
from langchain.tools.retriever import create_retriever_tool

memory_tool = create_retriever_tool(
    retriever,
    name="memory_search",
    description="从长期记忆检索用户偏好、事实。当用户问起以前说过的事时调用",
)
# memory_tool 就是一个标准工具，D5 学 @tool 时直接复用
```

> 类比：临时工版**没工牌**，官方工具封装（`create_retriever_tool`）不认它；正式版**有工牌**，官方接口直接认——这就是"生态识别"的价值。

---

## 八、读源码实战：三个疑问一次说清

你本地建好 `retriever.py` 后，最容易卡在"这字段/这方法哪来的"。逐个拆：

### Q1：`mm` 和 `k` 是给 W3 MemoryManager 的参数吗？

**是，而且分工明确：**

| 字段 | 装什么 | 对应 W3 的什么 |
|---|---|---|
| `mm` | **MemoryManager 的实例**（一个对象） | W3 的 `MemoryManager`——它身上带着 `recall` 方法 |
| `k` | **一个整数**（召回几条） | 传给 `recall(query, k=3)` 的 `k` 参数 |

> 类比：`mm` 是"档案室管理员本人"（他认识所有档案、知道怎么查），`k` 是"这次让他查几份"（3 份）。你雇了管理员（传实例），再告诉他查几份（传数字）——两件事，两个字段。

### Q2：`Field` 是什么类型？

**`Field` 不是数据类型，它是 pydantic 库的一个"字段说明书函数"。** 你之前没见过很正常——它是 pydantic（数据验证库）的语法，而 `BaseRetriever` 是 pydantic 模型，所以**字段必须用 pydantic 的写法声明**：

```python
from pydantic import Field

k: int = Field(default=3, description="召回条数")
#  ↑类型   ↑默认值      ↑给开发者的说明
```

- 普通类：`k = 3` 直接赋值就行
- pydantic 模型：`k: int = Field(default=3, ...)` —— 多了**类型标注 + 默认值 + 描述**，好处是自动类型检查、报错信息清晰、还能序列化保存

> 类比：**入职登记表的格子**。普通类是"随便写张纸"，pydantic 是"正规表格"——每个格子规定填什么类型（int）、没填时默认填什么（3）、这格是干嘛的（说明）。`Field` 就是"填表说明"。

### Q3：`facts = self.mm.recall(query, k=self.k)` 里的 recall 是哪来的？

**`recall` 是你 W3 自己写的！** 就在 `memory_manager.py` 里——只是它不在 `retriever.py` 这个文件里，所以你没找到。追根溯源：

```
retriever.py（现在）:
    self.mm.recall(query, k=self.k)
        ↓  self.mm 是你传入的 MemoryManager 实例
memory_manager.py（W3 你写的）:
    def recall(self, query: str, k: int = 3) -> list[str]:
        return self.ltm.recall(query, k=k)     # ← 找到了！在这
        ↓
long_term.py（W2/W3 你写的）:
    def recall(self, query, k=3, include_private=False):
        res = self.col.query(query_texts=[query], n_results=k)   # ← 最终在这
        ... 遍历 documents[0] 拿事实
```

**调用链：`retriever._get_relevant_documents` → `MemoryManager.recall` → `LongTermMemory.recall` → Chroma 向量检索**

> 类比：`recall` 像**老家的钥匙**——它在"家里"（memory_manager.py / long_term.py），你只是把管理员（mm 实例）带到了"新公司"（retriever.py），他用随身带的钥匙开门。钥匙不是新公司造的，是老家配的。

**怎么快速找到它（别再翻半天）**：把光标放在 `recall` 上 → `Ctrl+点击` 直接跳转定义；或者按 `Shift+F12` 查所有引用。你之前问过"上千行代码找不到函数"，这就是标准解法——**跳转，不搜索**。

### 最容易忽略的点（面试可能问）

`MemoryManager.recall` 返回的是 **`list[str]`（字符串列表）**，而 Retriever 要求返回 **`list[Document]`（文档列表）**——所以 `_get_relevant_documents` 里做了一次"包装"：

```python
facts = self.mm.recall(query, k=self.k)        # ["用户叫小明", "喜欢蓝色", "预算2000"]
return [
    Document(page_content=f, metadata={"source": "长期记忆"})  # 字符串 → 文档
    for f in facts
]
```

**类型不一样，这就是"适配"**：你的记忆系统说"字符串"，LangChain 说"Document"——`MemoryRetriever` 就是那个翻译官。

---

## 九、完整代码（可直接抄）

### `W4/lc_retriever_w3.py`（路径 A：RunnableLambda 包 W3）

```python
# W4/lc_retriever_w3.py —— 路径A：把 W3 长期记忆包装成 Retriever（复用已跑通的代码）
import sys, os
sys.path.insert(0, os.path.dirname(os.path.dirname(os.path.abspath(__file__))))  # 回到项目根

from langchain_core.documents import Document
from langchain_core.runnables import RunnableLambda
from memory.memory_manager import MemoryManager   # W3 的门面

mm = MemoryManager(window=8, max_tokens=1500, keep=4)
# 先存两条事实（persist=True 落长期 Chroma）
mm.add({"role": "user", "content": "用户喜欢用 DeepSeek，预算 2000 元"}, persist=True)
mm.add({"role": "user", "content": "用户叫小明，喜欢蓝色"}, persist=True)

def make_retriever(mm, k=3):
    """把 mm.recall 包成 LangChain 标准 Retriever：query -> list[Document]"""
    def recall_docs(query: str) -> list[Document]:
        facts = mm.recall(query, k=k)              # W3 已跑通的语义召回
        return [Document(page_content=f,
                         metadata={"source": "long_term"}) for f in facts]
    return RunnableLambda(recall_docs)

retriever = make_retriever(mm, k=3)

# 验收A：肉眼确认"相似片段被正确召回"
docs = retriever.invoke("用户喜欢什么模型？")
for d in docs:
    print("召回:", d.page_content)
```

（步骤 B 接 LCEL 链的完整版见第四节代码块，此处不重复。）

### `W4/retriever.py`（MemoryRetriever 正式版）

```python
# w4/retriever.py —— W4-D4 正式版：BaseRetriever 子类
from typing import Any
from langchain_core.retrievers import BaseRetriever
from langchain_core.documents import Document
from pydantic import Field, ConfigDict


class MemoryRetriever(BaseRetriever):
    """把 W3 的 MemoryManager.recall 包装成官方 Retriever。

    BaseRetriever 是个 pydantic 模型，所以字段要声明类型 + Field。
    """
    model_config = ConfigDict(arbitrary_types_allowed=True)  # 允许装非 pydantic 对象

    mm: Any = Field(description="带 recall(query, k) 方法的记忆管理器")
    k: int = Field(default=3, description="召回条数")

    def _get_relevant_documents(self, query: str, **kwargs) -> list[Document]:
        """唯一必须实现的抽象方法：query → Document 列表。

        **kwargs 兼容新旧版本签名（旧版无 run_manager，新版有）。
        """
        facts = self.mm.recall(query, k=self.k)          # W3 的召回
        return [
            Document(page_content=f, metadata={"source": "长期记忆"})
            for f in facts
        ]
```

### `W4/test_retriever.py`（验证，不需要 API key）

```python
# test_retriever.py
from w4.retriever import MemoryRetriever

class FakeMemory:
    def recall(self, query, k=3):
        return ["用户叫小明", "喜欢蓝色", "预算2000"]

r = MemoryRetriever(mm=FakeMemory(), k=3)
docs = r.invoke("用户喜欢什么")
assert isinstance(docs, list) and len(docs) == 3
print(f"✓ invoke 返回 {len(docs)} 条 Document")
print(f"✓ 内容: {docs[1].page_content} | 来源: {docs[1].metadata['source']}")

from langchain_core.runnables import RunnableLambda
chain = r | RunnableLambda(lambda ds: "；".join(d.page_content for d in ds))
print(f"✓ 链输出: {chain.invoke('用户喜欢什么')}")
```

---

## 十、踩坑记 & 知识点

**① DeepSeek 没有 embedding API** 这是今天最实战的一个坑。任务表骨架用 `OpenAIEmbeddings()` 要 `OPENAI_API_KEY`，而你只有 DeepSeek key——DeepSeek 官方不提供 embedding 接口。解法：路径 A 复用 W3 已落盘的 Chroma，零新增依赖。想练标准 `Chroma.from_texts` 再走路径 B（本地 `HuggingFaceEmbeddings`，首次下载 ~80MB）。

**② metadata = W3 的 `private` 标记** `Document.metadata` 和 W3 `add_fact(..., private=True)` 里的 `private` 是同源概念。之后做权限过滤（比如 `include_private`）时，直接读 `metadata["private"]` 即可，不用另建字段。

**③ 类型适配是核心：`list[str]` → `list[Document]`** `MemoryManager.recall` 返回字符串列表，Retriever 要求文档列表。`MemoryRetriever._get_relevant_documents` 里那层列表推导就是"翻译官"。忘写这层会报类型不匹配——这是接自研模块进 LangChain 最常见的坑。

**④ 临时工 vs 正式工，别混用** `RunnableLambda` 手工包能用、但没工牌——`create_retriever_tool` 等官方组件不认。要接 Agent 工具就必须是 `BaseRetriever` 子类。你已有的链一行不用改，因为两者都是 Runnable。

**⑤ 你提交的 `W4` 文件里有几处 typo，博客版已帮你归一（建议你顺手改掉原文件）：**
- `lc_retriever_w3.py`：`"用户喜欢Deekseek"` → `DeepSeek`；`metadata={"source":"lang_term"}` → `"long_term"`；函数名 `_ftm` → `_fmt`（且 prompt 模板里 `"：问题{question}"` 多了个全角冒号，应为 `问题:{question}`）。
- `retriever.py`：文件头注释 `BaseRetriver` → `BaseRetriever`；`padantic` → `pydantic`；`_get_relevant_documents` 的 docstring 缩进需顶格（当前缩进错位会导致语法怪异）。博客里的代码是归一后的干净版。

**⑥ `**kwargs` 是版本兼容保险** 老版本 `_get_relevant_documents(self, query)` 无 `run_manager`，新版有。`**kwargs` 两种都能跑，避免踩 D1 那种 `TextAccessor` 版本坑。

---

## 十一、总结

今天把 W3 的 Chroma 长期向量记忆，认领进了 LangChain 的 `Retriever` 标准接口——**决策逻辑一行没改，只是外面套了层适配器**：

- **Vector Store = 仓库**（`add_documents` 存、`similarity_search` 查），**Retriever = 取货员**（`as_retriever` 转成 LangChain 标准接口）。
- 接进 LCEL 用 `{"context": retriever | _fmt, "question": RunnablePassthrough()}` 并行取数，retriever 输出自动进 `{context}` 占位符。
- 检索参数 `k` 控 Top-K 召回数、`score_threshold` 控相似度下限。
- 临时工（`RunnableLambda`）能用但没工牌；正式工（`BaseRetriever` 子类 `_get_relevant_documents`）白送 `invoke/stream/batch`、能被 `create_retriever_tool` 识别、pydantic 字段类型安全。
- 核心认知：**RAG 的知识库 = W3 的长期向量记忆**，复用零新增依赖。

---
