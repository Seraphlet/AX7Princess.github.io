---
description: ""
title: "ShotMemory"
draft: false
date: "2026-08-15T00:04:07+08:00"
slug: "shotmemory"
categories:
 - null
tags:
 - null
image: ""
---


![1](1.webp)

## Agent短期记忆
大模型是没有记忆的，它不会记住上次聊了什么，每次聊天都是一个新的任务。
短期记忆 = "给大模型的对话历史设一个上限"，因为模型每次只能"看"有限的上下文（token 窗口），你不可能把无限长的聊天记录全塞给它。主要包括四个概念， 全量缓冲 / 滑动窗口 / token 裁剪 / 保留策略

 ### 全量缓冲（conversation buffer）

把所有对话一条不落全存着，像记账一样，代价就是每次给大模型输入的历史逐渐增加，Token消耗也逐渐增加直至最大上下文窗口。

```
buffer = [
    {"role": "user", "content": "你好"},
    {"role": "assistant", "content": "你好，有什么可以帮你？"},
    {"role": "user", "content": "我叫小王"},
    # ... 越攒越多,永不丢弃
]

```

### 滑动窗口（sliding window）
```
# 从 buffer 里取:系统提示 + 最近 N 条
def context(self):
    return sys_msgs + buffer[-N:]   # 只看最近 N 条
```
给模型看的时候，只取最近 N 条——像手机相册"最近 10 张，滑动窗口解决的是"给模型的输入太长"，不是"存储"太长。

存储 = 你程序里"攒着"的对话记录（buffer）
输入 = 你每次请求发给模型的那段消息

窗口只给模型看最近 10 条（输入可控），但 buffer 里存了 100 轮的原文 → 内存/文件越来越大，重启时加载越来越慢。

### token 裁剪（trim）

抽屉真的装不下了，扔最旧的东西。这是"真删"——和窗口的"只是不给你看"完全不同。裁剪直接改 buffer。

### 保留策略（retention policy）

哪些消息是"红线"，永远不能删——就是 system_roles（系统提示）。

```
system_roles = ("system",)   # 系统提示是红线
# 窗口和裁剪都先把它单独拎出来,永远保留
sys_msgs = [m for m in buffer if m["role"] in system_roles]
```
## 三者关系：由粗到细的三级防线

全量缓冲(全留,最费token)
   ↓ 取上下文时
滑动窗口(只看最近N条,不删)      ← 管"输入"长短
   ↓ buffer 真的超了
token 裁剪(从头部真删,超了删)    ← 管"存储"大小
   ↓ 但无论怎么删
保留策略(系统提示永远不动)       ← 红线,兜底

```
class ShortTermMemory:
    def __init__(self,window=10,maxtokens:int=500,system_roles=("system",)):
        self.window=window #聊天窗口大小
        self.maxtokens=maxtokens #最大tokens
        self.system_roles=system_roles 
        self.buffer=[] #消息缓存

    def _tokens(self,msgs) ->int: # 估算token数
        return sum(max(1,len(m.get("content",""))//3)for m in msgs) #1 个 token ≈ 3~4 个英文字符（约 0.75 个英文单词）

    def add(self,msg:dict):#消息缓存
        self.buffer.append(msg)

    def context(self)->list[dict]: # 返回最近n条消息
        sys_msgs=[m for m in self.buffer if m.get("role") in self.system_roles]
        others=[m for m in self.buffer if m.get("role") not in self.system_roles]
        return sys_msgs+others[-self.window:]
        
    def trim(self): #消息裁剪
        sys_msgs= [m for m in self.buffer if m.get("role") in self.system_roles]
        others =[m for m in self.buffer if m.get("role") not in self.system_roles]
        while self._tokens(others) >self.maxtokens and len(others)>1:
            others.pop(0)
        self.buffer=sys_msgs+others
        return self.buffer
        

```
总结：短期记忆 = buffer 存 + context 取窗口 + trim 真删 + system 永留