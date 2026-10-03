---
description: ""
title: "长期记忆"
draft: false
date: "2026-08-16T01:03:01+08:00"
slug: "long-memory"
categories:
 - null
tags:
 - null
image: ""
---

# 长期记忆原理：语义嵌入 · 向量库 · RAG 心智模型 · 学习笔记

## 0. 一句话心智模型

```
用户事实 / 偏好
   ├─► 精确 KV（SQLite）  → "用户喜欢 DeepSeek"=确定事实, key/value 秒查
   └─► 语义向量（Chroma） → "聊过什么相关话题"=模糊语义, 相似度召回
            │
            ▼ 首次 add 时 Chroma 自动嵌入, 落盘 ./mem_store/
   跨会话: 程序重启 → 重新连 PersistentClient → 之前存的事实还在
```

长期记忆 = **把用户事实嵌入成向量 → 落盘到本地 → 查询时语义召回**。

---

## 1. 三个必须记住的点

1. **短期 vs 长期是两种"不忘"**：短期 = 本次对话不丢（在内存里）；长期 = 跨进程、跨天不丢（落盘到磁盘）。
2. **Chroma 首次 `add` 会联网下载嵌入模型**（默认 `all-MiniLM` 约 80MB）——需要联网，第一次跑会慢。
3. **KV 和向量互补**：确定事实（名字、默认模型）用 SQLite；语义相关历史（"之前吐槽过什么"）才走向量。不是谁替代谁。

---

## 2. 向量库和"写本地文件"到底差在哪

**文件靠"精确匹配"**（key 对上才有），**向量库靠"语义相似"**（意思接近就能找出来）。

向量库底层也是文件——Chroma 把"数字指纹"和原文一起写进 `./mem_store/` 文件夹。它并没有比"写本地文件"更高级，只是**额外多存了那份"数字指纹"，并把指纹组织成"能快速按相似度找"的索引结构**。所以"向量库" = 本地文件 + 指纹索引。

---

## 3. 长期记忆的完整过程（三步）

假设用户昨天说"我特别不喜欢用 GPT，觉得它回答太啰嗦"，你想让它今天还能"记得"。

### 第 1 步：嵌入（embedding）——把文字变成"指纹"

计算机看不懂文字，只看得懂数字。嵌入就是把一句话变成一串固定长度的数字（向量，比如 384 个）：

```
"我特别不喜欢用 GPT" → [0.21, -0.03, 0.88, ...]  (384 个数字)
```

关键：**意思相近的句子，数字也相近**。这是嵌入模型（如 `all-MiniLM`）训练出来的本领。

### 第 2 步：落盘——把"指纹 + 原文"一起存进本地文件夹

Chroma 把这个数字指纹和原文一起写进本地 `./mem_store/`（本质就是一些文件）。这就是"写本地文件"！只是多了那份指纹和索引。

### 第 3 步：召回（recall）——把新问题也变指纹，然后"找指纹最像的"

用户今天问"你觉得我用哪个模型好？"：

1. 先把这句话也变成指纹（同一个嵌入模型）；
2. 拿这个指纹去 `mem_store` 里找"指纹最像的几条"（数学上叫**算余弦相似度**）；
3. 找到"我特别不喜欢用 GPT" → 把它塞进这次的上下文里告诉模型。

**所以长期记忆 = 存的时候"文字→指纹→落盘"；查的时候"问题→指纹→找最像的→取回原文"。**

---

## 4. 三种存储方式对比（一张表记牢）

| 方式                 | 怎么查          | 适合           | 类比          |
| ------------------ | ------------ | ------------ | ----------- |
| **本地文件（JSON/txt）** | 按 key 精确读    | 少量确定数据       | 手写便签        |
| **SQLite（KV）**     | 按 key 秒查、可更新 | 用户画像：名字、默认模型 | 有编号的档案柜     |
| **Chroma（向量）**     | 按语义相似度召回     | 模糊历史、偏好、相关话题 | 按内容指纹找书的图书馆 |

向量库并没有"更高级"，它只是"换了一种查法"——用语义相似度代替精确 key。存"用户叫什么"用 SQLite/文件就够了；存"用户之前吐槽过什么"这种需要"意思接近就找到"的，才用向量库。**两者配合才完整。**

---

## 5. 和 RAG 的对应关系（同一个套路）

长期记忆，其实就是**单文档 RAG 把"文档"换成"用户事实"且持久化**：

| 你的方法                         | RAG 里的角色                              |
| ---------------------------- | ------------------------------------- |
| `add_fact`                   | 写入 / 索引（`chunk + embed + store`）      |
| `recall`                     | 检索（`query embed + similarity search`） |
| `PersistentClient(path=...)` | 让向量库落到磁盘，而不是内存版                       |

---

## 6. 源码逐段讲解（注释拆解）

### 6.1 `long_term.py`

```python
'''
长期记忆学习文档
长期记忆 = 关电脑也不丢
长期记忆 = 单文档 RAG + 落盘
    嵌入/相似度召回  → 只是把"文档"换成"用户事实"且持久化
pip install chromadb
'''
import chromadb
from pathlib import Path

BASE = Path(__file__).parent
```

- **`__file__`**：Python 内置变量，代表"当前这个文件自己的路径"（含文件名）。例如 `long_term.py` 在 `C:\Desktop\AgentLearn\memory\long_term.py`，那 `__file__` 就是这个完整字符串。
- **`Path(...)`**：把字符串"升级"成路径对象，才能用它做路径拼接、取上级目录（而不是手动拼字符串）。
- **`.parent`**：取这个文件的"爸爸"（上一级目录）。`C:\...\memory\long_term.py` 的 `.parent` = `C:\...\memory\`（去掉文件名，回到 memory 目录）。
- 合起来：`BASE = Path(__file__).parent` = "记下我这个代码文件所在的文件夹"。

```python
class LongTermMemory:
    def __init__(self, path=str(BASE/"mem_store"), collection="long_term"):
        self.client = chromadb.PersistentClient(path=path)        # 建客户端(连上本地存储)
        self.col = self.client.get_or_create_collection(collection) # 拿货架(没有就建)
```

- **`PersistentClient(path=path)`**：创建持久化客户端——向量库落到磁盘（不是内存），关掉程序数据还在。
- **`collection`（集合）** = Chroma 里的一个"货架/分档"，用来把不同类型的数据分开存。`collection="long_term"` = 指定"我要用/建一个叫 long_term 的货架"。

```python
    def add_fact(self, text: str, fid: str):
        # Chroma 的 add 要求每个参数都是列表（ids=[...]、documents=[...]）
        # ——因为 Chroma 的 add 支持批量加。就算你只存一条，也要写成 [fid]、[text]
        self.col.add(ids=[fid], documents=[text])

    def recall(self, query: str, k: int = 3) -> list[str]:
        res = self.col.query(query_texts=[query], n_results=k)
        return res["documents"][0] if res["documents"] else []

    def delete(self, fid: str):
        self.col.delete(ids=[fid])
```

- `add_fact`：存一条事实。Chroma 首次 `add` 会**自动调用嵌入模型**把 `text` 变成向量，连同原文存进 `mem_store/`。
- `recall`：把 `query` 也嵌入，找最像的 `k` 条，返回原文列表 `res["documents"][0]`（`[0]` 因为一次只问一个问题）。
- `delete`：按 `fid` 删除。

### 6.2 `profile_store.py`

```python
import sqlite3, json
from pathlib import Path
BASE = Path(__file__).parent

class ProfileStore:
    def __init__(self, db_path=str(BASE / "memory.db")):
        self.con = sqlite3.connect(db_path)   # 连接/创建库
        self.con.execute("CREATE TABLE IF NOT EXISTS profile (k TEXT PRIMARY KEY, v TEXT)")  # 建表
        self.con.commit()                      # 提交

    def set_profile(self, k, v):
        self.con.execute("INSERT OR REPLACE INTO profile VALUES(?,?)", (k, json.dumps(v)))  # 写

    def get_profile(self, k):
        row = self.con.execute("SELECT v FROM profile WHERE k=?", (k,)).fetchone()  # 查
        return json.loads(row[0]) if row else None   # 取第一列 / 防空 / 还原 dict
```

- `sqlite3.connect(db_path)`：连接（或创建）SQLite 库。
- `CREATE TABLE IF NOT EXISTS profile (k TEXT PRIMARY KEY, v TEXT)`：建一张表，`k` 主键、`v` 文本。
- `json.dumps(v)` / `json.loads(row[0])`：因为 SQLite 字段是文本，Python 的 dict 要先 `dumps` 成字符串再存，读出后 `loads` 还原。
- `?` 占位符：防注入，值通过第二个参数传。

---

## 7. 最终源码（干净附录）

### `long_term.py`

```python
'''
长期记忆学习文档
长期记忆 = 关电脑也不丢
长期记忆 = 单文档 RAG + 落盘
    嵌入/相似度召回  → 只是把"文档"换成"用户事实"且持久化
pip install chromadb
'''
import chromadb
from pathlib import Path

BASE = Path(__file__).parent

class LongTermMemory:
    def __init__(self, path=str(BASE/"mem_store"), collection="long_term"):
        self.client = chromadb.PersistentClient(path=path) #建客户端(连上本地存储)
        self.col = self.client.get_or_create_collection(collection)# 拿货架(没有就建)

    def add_fact(self, text: str, fid: str):
        #Chroma 的 add 要求每个参数都是列表（ids=[...]、documents=[...]）——因为 Chroma 的 add 支持批量加。就算你只存一条，也要写成 [fid]、[text]
        self.col.add(ids=[fid], documents=[text])

    def recall(self, query: str, k: int = 3) -> list[str]:
        res = self.col.query(query_texts=[query], n_results=k)
        return res["documents"][0] if res["documents"] else []

    def delete(self, fid: str):
        self.col.delete(ids=[fid])
```

### `profile_store.py`

```python
import sqlite3, json
from pathlib import Path
BASE = Path(__file__).parent

class ProfileStore:
    def __init__(self, db_path=str(BASE / "memory.db")):
        self.con = sqlite3.connect(db_path)
        self.con.execute("CREATE TABLE IF NOT EXISTS profile (k TEXT PRIMARY KEY, v TEXT)")
        self.con.commit()

    def set_profile(self, k, v):
        self.con.execute("INSERT OR REPLACE INTO profile VALUES(?,?)", (k, json.dumps(v)))

    def get_profile(self, k):
        row = self.con.execute("SELECT v FROM profile WHERE k=?", (k,)).fetchone()
        return json.loads(row[0]) if row else None
```

---

## 8. 验收 & 避坑

- 能口头说清"嵌入→落盘→召回"三步分别是干嘛的。
- 能区分：确定事实走 SQLite，模糊语义走 Chroma，不是替代关系。
- 能说清 `PersistentClient` 是"落盘版"、跨会话重启后数据还在。
-  理解 `res["documents"][0]` 里 `[0]` 是因为一次只查一个问题。

| 现象                          | 最可能原因                                                       |
| --------------------------- | ----------------------------------------------------------- |
| 首次 `add` 很慢 / 卡住            | 在联网下载嵌入模型（all-MiniLM ~80MB），正常现象，等一次即可                      |
| `recall` 返回空                | `mem_store/` 路径不对或库为空；检查 `PersistentClient(path=)` 是否指向同一目录 |
| `KeyError: 'documents'`     | 写成 `res["document"]`（单数）了，正确是复数 `res["documents"]`          |
| SQLite "database is locked" | 多进程同时写；学习期单进程即可                                             |

---

*把"长期记忆"从一句口号拆成了可运行的机制：文字→向量→落盘→相似度召回，本质上就是单文档 RAG 把"文档"换成"用户事实"。*
