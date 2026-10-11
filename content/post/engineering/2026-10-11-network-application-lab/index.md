---
description: ""
title: "从 Socket 发一句话到 HTTP、API 与实时通信"
draft: false
date: "2026-10-11T09:36:07+08:00"
slug: "network-application-lab"
categories:
 - 
tags:
 - Network
image: ""
---


# 计算机网络应用实战：从 Socket 发一句话到 HTTP、API 与实时通信

> **定位：个人学习与复盘，不是协议百科。** 我已经接触过 IP、ARP、ICMP、子网与路由，但过去的知识停留在“知道协议叫什么”。这次要真正让程序通信，并能在失败时沿链路找到证据。实验以 **Windows + Python 标准库 + PowerShell + 浏览器** 为主；无需第三方 Python 包。  
> **前置条件：** Windows 安装了 Python 3，可通过 `py -3 --version` 检查；如果用的是 `python` 命令，后文 `py -3` 可等价替换。实验全部先在本机 `127.0.0.1` 执行；局域网测试需可信网络及接收方防火墙允许。

## 0. 一张主图：一条消息从哪里走

```text
电脑 A（发送者）                              电脑 B（接收者）
网页 fetch / Python client                    HTTP API / Python server
         │                                             ▲
         ▼                                             │
[应用层] 组织消息：HTTP 方法、路径、正文       [应用层] 解析请求、处理业务
         │                                             ▲
[Socket API] 请求操作系统发送字节             [Socket API] 从操作系统读取字节
         │                                             ▲
[TCP] 端口、连接、可靠有序的字节流 ─────────────── TCP
         │                                             ▲
[IP] 寻址与路由 ─────────────────────────────────────── IP
```

**顺序不是先背名词，而是依次回答：** ① 怎样让另一个程序接收？② 怎样组织一条消息？③ 怎样把消息变成 API 调用？④ 网页为什么跟命令行表现不同？⑤ 怎样实现实时消息？⑥ 失败时检查到哪一层？

> **IP 找机器，端口找服务；Socket 是程序请求操作系统进行网络收发的接口；HTTP 约定应用消息怎样表达。** 不要把“程序的函数调用”和“网络发包”混为一谈。

## 1. 实验一：最小 Socket，一台电脑上的两个程序

**目标：** 不使用 HTTP，让客户端发一句话，服务端打印并回复。**完成判据：** 两个终端分别看到收到的消息。**你要观察：** `bind`、`listen`、`accept`、`recv` 出现在主调用链的哪里。

创建目录 `network-lab`，其中保存下列两个文件。

### `tcp_server.py` —— 接收者

```python
import socket

HOST = "127.0.0.1"  # 只接受本机连接
PORT = 8765

with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server:
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind((HOST, PORT))              # 监听哪一个本机地址和端口
    server.listen(1)                       # 开始接收连接请求
    print(f"[LISTEN] {HOST}:{PORT}", flush=True)
    conn, address = server.accept()        # 等待：新连接到来时返回
    with conn:
        print(f"[CONNECTED] {address}")
        data = conn.recv(4096)             # 等待：读取一批字节，不保证恰好一条消息
        print("[RECEIVED]", data.decode("utf-8"))
        conn.sendall("收到你的消息".encode("utf-8"))
```

### `tcp_client.py` —— 发送者

```python
import socket

with socket.create_connection(("127.0.0.1", 8765), timeout=5) as client:
    client.sendall("你好，来自电脑 A".encode("utf-8"))
    response = client.recv(4096)
    print("[REPLY]", response.decode("utf-8"))
```

**执行：** 先在第一个 PowerShell 窗口执行 `py -3 tcp_server.py`，保持窗口不关；第二个窗口执行 `py -3 tcp_client.py`。服务端打印接收到的文本，客户端显示 `[REPLY] 收到你的消息`。服务端只接受一次连接，然后退出；每次复测需要重新启动服务端。

### 调用链怎么读

```text
服务端：socket → bind → listen → accept → recv → sendall → close
客户端：create_connection(隐含创建与 connect) → sendall → recv → close
```

`accept()` 返回的是 **新连接的 Socket**，用于与这个客户端交流；最初的 `server` Socket 继续承担监听职责。这是以后理解并发服务器的起点。`recv(4096)` 代表最多读取 4096 字节，**不是“读取一条完整消息”**。

**思考并回答：** 如果 Windows 上同一台电脑同时运行浏览器、微信和 Python，数据凭什么交给 Python？因为目标端口由监听 Socket 占用；操作系统结合连接信息把收到的数据交给对应 Socket。

### 失败时怎么定位（不要猜）

| 表现 | 第一条证据 | 更可能的解释 |
|---|---|---|
| `ConnectionRefusedError` | 服务端终端是否打印 `[LISTEN]` | 没启动、已退出、端口不一致 |
| `WinError 10048` / Address in use | `Get-NetTCPConnection -LocalPort 8765` | 已有程序占用端口 |
| 客户端连接超时 | IP、端口、监听地址、网络/防火墙 | 连接未建立；本机实验先确认地址 |
| `UnicodeDecodeError` | 发送与接收编码是否一致 | 发的不是合法 UTF-8 字节 |

**验收：** 能解释为什么客户端不需要显式 `bind()`（操作系统通常自动选择本地临时端口）；能解释为什么 B 必须运行接收程序。

## 2. 实验二：TCP 传的是字节流，不是“聊天气泡”

这一步解释一个在工程中容易犯的错误：

```python
client.sendall(b"hello")
client.sendall(b"world")
```

接收方不能假设连续两次 `recv()` 必然分别得到 `hello` 和 `world`。一次 `recv()` 可能收到 `helloworld`，也可能只收到一部分。TCP 保证字节的顺序，不负责分隔业务消息。**如果把一次 `recv` 当成一条完整消息，输入变长就容易出错。**

### 两种消息边界设计

- **换行分隔：** 发送 `你好\n`，接收端缓冲数据直到遇到换行。适合文本且消息本身需要处理换行转义。
- **长度前缀：** 先发送一个固定长度的整数，表示随后正文的字节数。适合任意字节消息。接收端要循环读取，直到字节数满足。

下面是关键实现片段（用于理解消息边界，可放入后续完整程序）：

```python
# 把一条 UTF-8 文本封装为：4 字节长度 + 正文字节
import struct

message = "你好".encode("utf-8")
packet = struct.pack("!I", len(message)) + message
client.sendall(packet)

# 接收端必须循环读取，不能假设一次 recv 就够

def recv_exact(sock, size):
    parts = []
    while size > 0:
        chunk = sock.recv(size)
        if not chunk:
            raise ConnectionError("对端提前关闭连接")
        parts.append(chunk)
        size -= len(chunk)
    return b"".join(parts)

length = struct.unpack("!I", recv_exact(conn, 4))[0]
if length > 1024 * 1024:
    raise ValueError("消息过大")
message = recv_exact(conn, length).decode("utf-8")
```

**注意：** 片段中的 `client`、`conn` 需来自实验一的连接对象；不要把它当作独立可运行脚本。长度上限是防止对方声称要发送无限大正文的基本保护。

**验收：** 能说出为什么 `sendall()` 成功并不保证接收方业务已经处理成功；能说明编码、消息边界、超时分别解决什么问题。

## 3. 实验三：不用自己设计消息格式，改用 HTTP

原始 Socket 能收发字节，但要自己定义“请求什么、正文是什么、如何返回错误”。HTTP 已经提供了请求方法、路径、Header、Body、状态码。常用 **HTTP/1.1** 可用请求与响应文本理解；HTTP/2、HTTP/3 的线上编码方式不同，但应用语义基本一致。

```text
客户端 POST /messages（JSON） ──► HTTP 服务端
客户端 ◄── 201 Created（JSON） ────┘
```

### `http_server.py` —— 标准库最小消息 API

```python
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import json

messages = []  # 只存内存，重启服务就丢；先学习通信，不先做数据库

class Handler(BaseHTTPRequestHandler):
    def reply(self, status, obj):
        raw = json.dumps(obj, ensure_ascii=False).encode("utf-8")
        self.send_response(status)
        self.send_header("Content-Type", "application/json; charset=utf-8")
        self.send_header("Content-Length", str(len(raw)))
        self.end_headers()
        self.wfile.write(raw)

    def do_GET(self):
        if self.path != "/messages":
            return self.reply(404, {"error": "not found"})
        self.reply(200, {"messages": messages})

    def do_POST(self):
        if self.path != "/messages":
            return self.reply(404, {"error": "not found"})
        try:
            length = int(self.headers.get("Content-Length", "-1"))
            if not 0 <= length <= 4096:
                return self.reply(413, {"error": "body too large or missing"})
            payload = json.loads(self.rfile.read(length).decode("utf-8"))
            value = payload.get("message")
            if not isinstance(value, str) or not value.strip():
                return self.reply(400, {"error": "message required"})
        except (ValueError, UnicodeDecodeError, AttributeError):
            return self.reply(400, {"error": "invalid json"})
        messages.append(value)
        self.reply(201, {"saved": True, "count": len(messages)})

if __name__ == "__main__":
    with ThreadingHTTPServer(("127.0.0.1", 8766), Handler) as server:
        print("HTTP API: http://127.0.0.1:8766/messages", flush=True)
        server.serve_forever()
```

**终端 1：** `py -3 http_server.py`。**终端 2：** 使用 Windows 自带 `curl.exe`：

```powershell
curl.exe -i http://127.0.0.1:8766/messages
curl.exe -i -X POST http://127.0.0.1:8766/messages -H "Content-Type: application/json" --data-raw '{"message":"hello"}'
curl.exe -i http://127.0.0.1:8766/messages
```

PowerShell 对非 ASCII 参数的编码处理因版本和终端设置而异；首次实验先用 `hello` 确认协议流程。若请求体乱码，再单独排查 UTF-8 编码，不要把编码问题误判为网络问题。你应看到第一次 GET 返回空数组、POST 返回 `201`、第二次 GET 包含 `hello`。

**主动制造三个错误：** 用 GET 请求不存在的 `/wrong`（404）；POST `{}`（400）；让服务端关闭再请求（连接失败而不是 HTTP 状态码）。这三个结果分别说明请求已经到达 HTTP 服务、格式不合契约、根本没有建立 HTTP 通信。

### 一次 HTTP API 的主调用链

```text
curl/浏览器构造请求 → TCP 连接 → 服务端接收 HTTP 报文
  → 根据 Method + Path 路由到 Handler
  → 读取 Body → JSON 解码 → 参数校验
  → 执行 messages.append() → 构造状态码和 JSON → 返回
```

`JSON` 是数据表示格式，不是网络协议。`API` 是双方约定的操作接口，不是某个特殊端口。`REST` 是一种资源设计风格，不等于“所有 HTTP API 都是 REST”。

**验收：** 区分“TCP 通”“HTTP 有响应”“业务成功”这三种说法。

## 4. 实验四：让网页发送消息，以及为什么会遇到 CORS

建立 `send.html`：

```html
<!doctype html>
<html lang="zh-CN">
<meta charset="utf-8">
<title>发送消息实验</title>
<input id="msg" value="hello from browser">
<button id="send">发送</button>
<pre id="output"></pre>
<script>
  document.querySelector('#send').addEventListener('click', async () => {
    const message = document.querySelector('#msg').value;
    const output = document.querySelector('#output');
    try {
      const response = await fetch('http://127.0.0.1:8766/messages', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ message })
      });
      output.textContent = `${response.status}: ${await response.text()}`;
    } catch (error) {
      output.textContent = `请求失败：${error.message}`;
    }
  });
</script>
</html>
```

直接双击 HTML 时，**可能失败**：文件来源是 `file://`，浏览器对跨来源访问有额外限制。不是 `fetch()` 不会发 HTTP，而是浏览器实行安全策略。当前服务端也没有实现 CORS 预检（`OPTIONS`）与允许来源响应头，因此浏览器跨来源 JSON POST 通常会失败。

### 实际观察方式

1. 在浏览器按 F12，打开 **Network（网络）**。
2. 点击发送，寻找 `OPTIONS /messages` 或 `POST /messages`。
3. 查看 **Request Headers**：`Origin`、`Content-Type`。
4. 查看服务端是否返回 `Access-Control-Allow-Origin`，以及预检允许的方法/请求头。
5. 用 `curl.exe` 访问同一路径，对比它能否成功。

**现象解读：** curl 能请求而浏览器失败，优先看浏览器安全限制，而不是重写 IP 或 TCP 代码。`file://` 在不同浏览器和策略下表现可能不同；HTTPS 页面请求普通 HTTP、以及浏览器对本地网络的权限策略，还会带来额外限制。

### 更简单的实践解法：同源

开发时，让同一个 HTTP 服务同时提供 `send.html` 和 `/messages`，并使用相对路径 `fetch('/messages')`，那么网页与 API 来源相同，就不需要为这个请求跨域授权。当前示例服务**尚未实现静态文件服务**；这一点是你要理解的演进方向，而不是让上述代码立刻 magically 运行成功。

> **不要**为了让浏览器访问成功就直接在生产环境返回 `Access-Control-Allow-Origin: *`，尤其不要和带 Cookie 的接口混用。跨域授权范围要依据实际可信来源设置。

**验收：** 你能拿 F12 的 Network 面板证明请求是被预检阻止、被网络拒绝，还是 HTTP 返回了 400/404。

## 5. 实验五：从一台电脑搬到两台局域网电脑

之前的 `127.0.0.1` **只表示当前机器**。即使 B 知道 A 的回环地址，B 访问自己的 `127.0.0.1` 也不会到 A。

假设 B 的 LAN IP 是 `192.168.1.20`（仅示例，按 `ipconfig` 的实际 IPv4 地址替换）。让 B 的 HTTP 服务器监听 LAN 地址，或在可信网络实验中监听 `0.0.0.0`：

```python
# http_server.py 最后一处改为：
with ThreadingHTTPServer(("0.0.0.0", 8766), Handler) as server:
```

A 执行：

```powershell
Test-NetConnection 192.168.1.20 -Port 8766
curl.exe -i http://192.168.1.20:8766/messages
```

若 B 上的 HTTP API 可达，就可以向同一地址 POST。**局域网实验必须注意：** 演示服务无认证、数据存内存，只能在可信网络短时间使用；Windows 防火墙可能需要明确允许该端口的入站连接，实验结束恢复规则，不要关闭整个防火墙。

### 为什么 A 仍访问失败？按证据排序

| 测试 | 命令/观察 | 能说明什么，不能说明什么 |
|---|---|---|
| B 的 IP | `ipconfig` | 目标 IP；不证明服务已启动 |
| B 的监听 | `Get-NetTCPConnection -LocalPort 8766 -State Listen` | 存在监听；要检查 LocalAddress 是否仅 127.0.0.1 |
| A 的 TCP | `Test-NetConnection 192.168.1.20 -Port 8766` | TCP 连接能否建立；不保证 HTTP 路由和业务成功 |
| A 的 HTTP | `curl.exe -i http://192.168.1.20:8766/messages` | 看到状态码说明到达 HTTP 层；404 不等于断网 |
| 浏览器 fetch | F12 → Network | 与 curl 对照检查 CORS/浏览器限制 |

`ping` 属于 ICMP 测试，禁用 ICMP 的主机可能 ping 不通但 TCP 正常。IP、路由、代理、防火墙和目标服务都可能导致失败，**一次失败不能直接断定是哪一项**。

## 6. 实验六：从查询 API 到实时通信

如果每隔两秒调用一次 `GET /messages`，就能让另一个浏览器看到新消息。这叫 **Polling（轮询）**，非常适合当前的小实验：

```javascript
// 仅在网页与 API 同源、且有 /messages 接口时使用
async function refreshMessages() {
  const response = await fetch('/messages');
  if (!response.ok) throw new Error(`HTTP ${response.status}`);
  const data = await response.json();
  console.log(data.messages);
}
setInterval(() => refreshMessages().catch(console.error), 2000);
```

它有两个明显缺点：没有消息也反复请求；两次轮询之间存在延迟。然后再看其他方案：

| 方案 | 谁主动产生下一次网络动作 | 适合什么 |
|---|---|---|
| 轮询 | 客户端定时发请求 | 少量用户、小工具、简单状态刷新 |
| SSE | 客户端建立流式订阅，服务端持续写事件 | 日志流、Agent token 流、通知 |
| WebSocket | 双方在持久连接上互发消息 | 高频实时聊天、多人协作 |

**SSE 与 WebSocket 不是“必须升级”的目标。** 一段时间只有两个用户、每几秒刷新一次也能接受，轮询可能更简单。注意 SSE 原生 `EventSource` 不便于自由设置认证请求头，需要结合 Cookie 或选择其他客户端流式读取方式设计认证。

### 迁移到 AI Agent

```text
用户在网页提问
  → POST /agent/run 启动任务
  → 服务端调用模型、工具和数据库
  → SSE / HTTP 流式输出进度
  → 浏览器持续展示
```

这里的核心仍然是 HTTP 请求、数据流与应用协议；LLM 只是服务器内部的一种业务能力。MCP 又是在应用通信之上定义工具发现和调用的协议，其传输可采用本地 stdio 或 Streamable HTTP，并不等同于 TCP。

## 7. 真正可用于工作的排查流程：找出请求最远走到哪里

**原则：一次只验证一个假设；先收集证据再修改设置。** 原文《网络故障排查宝典》已经有 IP、路由、DNS、VPN、代理等广泛检查。这篇增加的是面向你自己写的应用程序的验证链。

```text
用户说：网页发不出去消息
       │
       ▼
① 服务端进程活着吗？在监听哪一个地址/端口？
       │ 是
       ▼
② 从客户端 Test-NetConnection 能建立 TCP 吗？
       │ 是
       ▼
③ 用 curl.exe 能得到 HTTP 状态码吗？
       │ 是
       ▼
④ Path、Method、Header、Body 合契约吗？
       │ 是
       ▼
⑤ 浏览器 Network：请求是否跨域、预检、证书或混合内容拦截？
       │ 已通过
       ▼
⑥ 服务端日志：是否收到请求、保存是否成功？
       │ 成功
       ▼
⑦ 浏览器渲染/数据更新逻辑是否正确？
```

### 现场排查对照表

| 看到什么 | 从哪一步查起 | 下一条最有价值的证据 |
|---|---|---|
| `Connection refused` | 服务监听 | 服务器 `Get-NetTCPConnection` 输出 |
| `Connection timed out` | 网络路径、防火墙、监听地址 | 客户端 `Test-NetConnection` + 服务端监听 |
| HTTP `404` | Path / 路由 | `curl.exe -i` 的完整路径和响应 |
| HTTP `400` | JSON / 参数校验 | 请求 Body 与服务端错误信息 |
| HTTP `401` / `403` | 身份认证 / 授权 | Token 使用方式、服务端权限日志（不要泄露令牌） |
| HTTP `500` | 服务端内部执行 | 服务端 traceback / 请求 ID |
| curl 成功、网页 CORS 失败 | 浏览器 Origin / OPTIONS | F12 → Network 的预检与响应头 |
| HTTP 200，界面空白 | 前端解析或渲染 | Response 内容、Console 报错 |
| 发送一次出现两条 | 重试、事件绑定、请求重复 | Network 请求数量、服务端日志/事件 ID |

### 一个很容易忽视的可靠性问题：超时不等于操作未执行

```text
客户端 POST 保存消息 → 服务器已写入 → 响应丢失/超时
                       客户端以为失败 → 重试 → 可能写两次
```

因此“网络重试”不能无脑应用到有副作用的请求。要根据语义决定是否重试，并考虑幂等键、去重或读取确认。这一点与 Agent 调用外部写工具完全一致。

## 8. 实际工程问题：如何选择，而不是只会用 API

| 需求 | 简单合理的起点 | 暂时不需要 |
|---|---|---|
| 同机发送一句话 | Socket 教学实验 | 框架、数据库 |
| 两台 LAN 设备互发消息 | 一个 HTTP 服务 + 两个客户端 | 分布式架构 |
| TaskFlow 本机保存数据 | IndexedDB | 服务器 |
| TaskFlow 连接自建 Memos | fetch + Memos API + 用户自己配置 Token | 重写 Memos 后端 |
| 展示 Agent 流式回复 | HTTP Streaming / SSE | 为流式输出强上 WebSocket |
| 远程调用工具 | HTTP API / RPC / MCP 适配 | 自造底层网络协议 |

最后要把**身份与信任边界**放进设计：私人局域网实验可以短暂使用无认证服务，但公开服务必须认证、限制权限、保护令牌与传输、合理处理跨域；浏览器本地保存 Token 需要明确存储风险。

