---
description: ""
title: "记忆本地化"
draft: false
date: "2026-08-17T07:22:41+08:00"
slug: "local-memory"
categories:
 - 
tags:
 - 
image: ""
---

# 给 AI Agent 的记忆找个"保险柜"：三层持久化与隐私不出网的设计复盘

## 一、先给记忆分个类：三层持久化，不是替代而是分层

新手最容易踩的坑，是以为"记忆存一个地方就够了"。其实数据形态不同，落盘介质就得不同。我们用了三种，它们是**分层分工**，不是谁替代谁：

| 持久化方式 | 介质 | 适合存什么 | 典型场景 |
|---|---|---|---|
| **纯本地文件** | `facts.json` | 配置 / 简单开关 / 零依赖小事实 | 手改也行、不需要语义的键值 |
| **SQLite** | `memory.db` | 用户画像 KV / 结构化表 | 精确查询、按 key 取用户资料 |
| **Chroma on-disk** | `mem_store/` | 向量嵌入 / 语义召回 | "用户喜欢什么"这种相关度检索 |

一句话记住：**纯文件管配置，SQLite 管画像，Chroma 管语义。** 别把用户画像塞进向量库，也别把聊天嵌入塞进 JSON。

---

## 二、存得稳：给"账本"找保险柜

记忆最先死在哪？**路径漂移**。

很多人写 `PersistentClient(path="./mem_store")`——这个 `./` 不是"项目目录"，而是**你敲命令那一刻所在的目录（cwd）**。在 A 目录跑建了库、在 B 目录跑就找不到，重启后记忆仿佛被偷了。

解决办法只有一条铁律：**用绝对路径**。

```python
import os
BASE = os.path.dirname(os.path.abspath(__file__))
MEM_STORE  = os.path.join(BASE, "mem_store")   # Chroma 向量库
PROFILE_DB = os.path.join(BASE, "memory.db")   # SQLite 用户画像
FACTS_JSON = os.path.join(BASE, "facts.json")  # 纯本地简单事实
```

`__file__` 是"这个文件自己在哪里"，所以无论从哪个目录启动，所有落盘点都稳稳指向同一处。这就像给账本找了一个**固定不动的保险柜**，而不是随手塞进今天的裤兜。

---

## 三、存得安全：隐私默认不泄露

这是今天最核心的设计动作：**隐私分流**。它不会自动发生，是你必须主动写进代码里的。

大模型 API 调用，会把你传给它的 prompt 内容发出去。所以得想清楚：

- 🔒 **永远留本地、绝不进 prompt**：用户真实姓名、手机号、密钥、明文敏感偏好。
- 🌐 **会发到云端 LLM API**：你主动塞进 system/user 消息的"召回结果"（比如"已知用户长期事实：喜欢 DeepSeek"）。
- ⚠️ **敏感事实**：存本地，但召回时**默认不自动进 prompt**，需用户显式确认才带上。

我们用一个极简机制实现"默认不泄露"：

```python
# 存的时候贴标签
lt.add_fact("用户的手机号是 13800001234", fid="f_phone", private=True)

# 默认不泄露：recall 默认 include_private=False → 召回不到敏感事实
lt.recall("用户手机号")                    # -> []

# 要带才带：显式传 include_private=True 才召回
lt.recall("用户手机号", include_private=True)  # -> ["用户的手机号是 13800001234"]
```

背后的逻辑就三行，却把"隐私底线"钉死了：

```python
def recall(self, query, k=3, include_private=False):
    res = self.col.query(query_texts=[query], n_results=k)
    facts = []
    for doc, meta in zip(res["documents"][0], res["metadatas"][0]):
        if meta and meta.get("private") and not include_private:
            continue   # 敏感事实：没授权就不进结果
        facts.append(doc)
    return facts
```

**心智模型**：存的时候贴标签（`private=True`），召回时默认"看都不看"，只有你明确说"这次可以带"才放行。你传给 LLM 的，永远只是召回出的非敏感摘要；原始记忆文件只在本地磁盘。

---

## 四、代码长什么样

下面是三个核心文件的完整源码，注释里把设计约定也写进去了。

### `paths.py` —— 集中管理所有落盘路径

```python
"""集中管理所有落盘路径, 避免各处硬编码 './xxx' (相对路径会随 cwd 漂移)."""
# ============ 隐私分流（设计约定, 不是代码自动给的） ============
# 🔒 永远留本地、绝不进 prompt: 用户真实姓名 / 联系方式 / 密钥 / 明文敏感偏好
# 🌐 会发到云端 LLM API: 你主动塞进 system/user 消息的"召回结果"
#     (如"已知用户长期事实: 喜欢 DeepSeek")
# ⚠️ 敏感事实: 存本地, 但召回时不自动进 prompt, 需用户显式确认才带上
# ==============================================================

import os
BASE = os.path.dirname(os.path.abspath(__file__))
MEM_STORE   = os.path.join(BASE, "mem_store")    # Chroma 向量库
PROFILE_DB  = os.path.join(BASE, "memory.db")    # SQLite 用户画像
FACTS_JSON  = os.path.join(BASE, "facts.json")   # 纯本地简单事实
if __name__=="__main__":
    print(BASE,"\n",MEM_STORE,"\n",PROFILE_DB,"\n",FACTS_JSON)
```

### `long_term.py` —— Chroma 长期记忆（语义召回 + 隐私开关）

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
import chromadb, time
from memory.paths import MEM_STORE 
#BASE=Path(__file__).parent

class LongTermMemory:
    def __init__(self, path=MEM_STORE, collection_name="long_term"):
        self.client = chromadb.PersistentClient(path=path)  # 绝对路径
        self.col = self.client.get_or_create_collection(collection_name)

    def add_fact(self, text: str, fid: str, private: bool = False):
        """存一条长期事实。private=True 表示敏感, 召回时默认不出网。"""
        self.col.add(ids=[fid], documents=[text],
                     metadatas=[{"private": private, "ts": time.time()}])

        #Chroma 的 add 要求每个参数都是列表（ids=[...]、documents=[...]）——因为 Chroma 的 add 支持批量加。就算你只存一条，也要写成 [fid]、[text]

    def recall(self,query:str,k:int=3,include_private=False)->list[str]:
        """召回相关事实。默认跳过 private=True 的敏感事实(不进 prompt)。"""
        res=self.col.query(query_texts=[query],n_results=k)
        facts=[]
        for doc,meta in zip(res["documents"][0],res["metadatas"][0]):
            if  meta and meta.get("private") and not include_private:
                continue
            facts.append(doc)
        return facts
    def delete(self,fid:str):
        self.col.delete(ids=[fid])
```

### `profile_store.py` —— SQLite 用户画像 KV

```python
import sqlite3, json
from pathlib import Path
from memory.paths import PROFILE_DB
#BASE = Path(__file__).parent

class ProfileStore:
    def __init__(self, db_path=PROFILE_DB):
        self.con = sqlite3.connect(db_path)   # 绝对路径
        self.con.execute("CREATE TABLE IF NOT EXISTS profile (k TEXT PRIMARY KEY, v TEXT)")
        self.con.commit()

    def set_profile(self, k, v):
        self.con.execute("INSERT OR REPLACE INTO profile VALUES(?,?)",(k,json.dumps(v)))

    def get_profile(self,k):
        row=self.con.execute("SELECT v FROM profile WHERE k=?",(k,)).fetchone()
        return json.loads(row[0]) if row else None
```

---

## 五、踩坑记：Python 包导入的"隐形墙"

顺手记一个今天踩到的坑，凡是把代码组织成"包"的人都会遇到。

我们的 `memory/` 是一个**包**（有 `__init__.py`），`long_term.py` 里写的是 `from memory.paths import MEM_STORE`——这是**包导入**，要求 `memory` 的**父目录**在 `sys.path` 里。

如果你在 `memory/` 目录里面直接 `python test_c.py`，而 `test_c.py` 又把 `memory/` 自己（而不是它的父目录）加进 `sys.path`，就会炸：

```
ModuleNotFoundError: No module named 'memory'
```

原因：此时 Python 的搜索路径里只有 `memory/` 自己，根本没有名为 `memory` 的包。

**铁律**：包名加不加，全工程必须统一。内部选了 `from memory.xxx`，外部调用方就**必须夹包名**。文件"在同一个文件夹"≠"可以不加包名"——`import` 永远是基于 `sys.path` 找顶层模块，不是基于"当前文件旁边有什么"。

最干净的运行方式，是从父目录用 `-m` 启动：

```powershell
cd AgentLearn
python -m memory.test_c
```

---

## 六、总结

1. **三层持久化是分层不是替代**——文件管配置、SQLite 管画像、Chroma 管语义。
2. **绝对路径是记忆不漂移的底线**——`os.path.dirname(os.path.abspath(__file__))` 用起来。
3. **隐私是设计出来的，不是自动的**——`add_fact(private=True)` 贴标签，`recall()` 默认不泄露，要带才带。

给账本找好保险柜、贴好标签、分好类，你的 Agent 才算真正"记得住、守得住"。下一步（Day4）我们做上下文压缩：把超长的老历史浓缩成一条系统摘要——压是浓缩，不是丢掉。
