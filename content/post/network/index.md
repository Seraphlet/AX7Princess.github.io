---
description: ""
title: "网络故障排查宝典 "
draft: false
date: "2026-09-11T00:30:25+08:00"
slug: "network"
categories:
 - 
tags:
 - Network
image: ""
---

# 网络故障排查宝典

> **定位：Windows 网络故障的“小白可执行 + IT 可继续深入”排障手册**
>
> 本文以原《网络排障总纲：OSI 七层逐层排查（详版）》为基础进行重构和扩充。原稿的核心框架——“OSI 七层 → 现象 → 判断 → 命令 → 修复”——全部保留，并增加故障决策树、命令结果解释、Windows 图形界面路径、典型故障案例、企业办公网络场景、排障记录模板和安全边界。
>
> **重要说明：** OSI 七层是排障思维框架。现实中的 Windows、TCP/IP、DNS、TLS、HTTP、VPN、代理、防火墙等协议与组件并不会严格“一层只归一个层”。为了让排障更容易执行，本文采用“**最适合定位的层**”来归类，而不是把所有协议机械地塞进某一层。

---

# 目录

1. 新手先看：网络到底是怎么工作的
2. 遇到网络问题先做什么
3. 10 秒定位法：应该从哪一层开始查
4. 一张万能排障决策树
5. 第 1 层：物理层
6. 第 2 层：数据链路层
7. 第 3 层：网络层
8. 第 4 层：传输层
9. 第 5 层：会话层
10. 第 6 层：表示层
11. 第 7 层：应用层
12. “浏览器正常，某软件不正常”的专门排查路线
13. DNS 专项排查
14. 代理专项排查
15. VPN / 隧道专项排查
16. 防火墙与安全软件专项排查
17. 共享文件夹 / 打印机专项排查
18. 企业办公网络常见场景
19. 常用命令逐个讲解
20. 端口 ↔ 进程专项工具
21. Wireshark 进阶排查思路
22. 修复动作的安全边界
23. 常见故障案例库
24. 排障记录模板
25. 一页速查表
26. 最终心法

---

# 一、新手先看：网络到底是怎么工作的

## 1.1 把网络想成“寄快递”

如果你完全不懂网络，可以先不要背 TCP、UDP、ARP、DNS 这些名词。

把一次网络访问想成寄快递：

```text
你要访问一个网站
        ↓
电脑先确认“网线/Wi-Fi 有没有通”
        ↓
确认“我是谁，我在这个局域网哪里”
        ↓
确认“去互联网应该走哪个出口”
        ↓
确认“我要连接哪个门牌号（端口）”
        ↓
建立并保持通信
        ↓
确认加密和证书
        ↓
最后才是“登录、查询、上传、下载”等具体业务
```

对应到本文的七层：

```text
L7 应用层    = 你真正使用的软件/网页/业务
L6 表示层    = 加密、证书、数据表示
L5 会话层    = 会话建立、保持、重连
L4 传输层    = TCP/UDP、端口
L3 网络层    = IP、路由、网关
L2 链路层    = MAC、ARP、交换
L1 物理层    = 网线、网卡、Wi-Fi、接口、信号
```

口诀：

> **物、链、网、传、会、表、应。**

---

# 二、遇到网络问题先做什么

## 2.1 第一原则：不要一上来就“修”

很多网络故障越修越乱，是因为没有先确定问题。

错误做法：

```text
网页打不开
↓
先改 DNS
↓
还是不行
↓
关防火墙
↓
还是不行
↓
删路由
↓
重装网卡
↓
最终不知道原来是什么问题
```

正确做法：

```text
发现问题
↓
记录现象
↓
确定影响范围
↓
做最小测试
↓
定位 OSI 层
↓
只验证一个假设
↓
只修改一个变量
↓
重新测试
↓
确认修复
```

---

## 2.2 先问自己 6 个问题

普通用户遇到问题时，不需要知道一堆网络术语，先回答这 6 个问题即可：

### 问题 1：是“所有东西都不能上网”，还是“只有一个软件有问题”？

- 所有软件都不行 → 从 L1 开始
- 只有一个软件不行 → 优先 L4~L7

### 问题 2：Wi-Fi / 网线显示连接了吗？

- 未连接 → 先看 L1
- 已连接 → 继续

### 问题 3：电脑有没有 IP 地址？

执行：

```cmd
ipconfig
```

重点看：

- IPv4 地址
- 默认网关

### 问题 4：电脑能不能找到路由器？

执行：

```cmd
ping 默认网关
```

例如：

```cmd
ping 192.168.1.1
```

- 能通 → L1/L2 大体正常
- 不通 → 从 L1/L2 查

### 问题 5：能不能访问公网 IP？

执行：

```cmd
ping 8.8.8.8
```

- 能通 → 公网 IP 通，继续查 DNS / L4~L7
- 不通 → 继续查 L3

### 问题 6：域名能不能解析？

执行：

```cmd
nslookup baidu.com
```

- 能解析 → DNS 大体正常
- 解析失败 → 重点查 DNS

---

# 三、10 秒定位法：应该从哪一层开始查

## 3.1 最重要的快速定位表

| 现象 | 首先查 | 为什么 |
|---|---|---|
| 网卡显示“媒体已断开” | L1 | 物理连接本身没建立 |
| Wi-Fi 搜不到网络 | L1 | 无法建立无线链路 |
| 有 IP，但网关 ping 不通 | L2 | 局域网邻居通信异常 |
| 网关能通，公网 IP 不通 | L3 | 路由/出口异常 |
| 公网 IP 通，域名打不开 | DNS / L3→L7 | 域名解析异常 |
| ping 通，TCP 端口不通 | L4 | 端口/防火墙/服务监听 |
| TCP 能建立，但连接反复断 | L5 | 会话维持异常 |
| TLS/SSL/证书错误 | L6 | 加密/证书校验异常 |
| HTTP 返回 401/403/404/5xx | L7 | 业务层异常 |
| 只有某个软件不能联网 | L4~L7 | 上层局部配置更可疑 |
| 同一网站所有电脑都打不开 | L3~L7 | 可能是网络出口或服务端 |
| 只有这一台电脑打不开 | L1~L7 | 先查本机 |

---

## 3.2 一个非常实用的判断

### “好软件”与“坏软件”对照

例如：

```text
Chrome         √
Edge           √
微信           √
某业务软件     ×
```

此时不要再反复检查网线。

因为已经有多个软件证明：

```text
L1 物理层     大概率正常
L2 链路层     大概率正常
L3 网络层     大概率正常
```

优先检查：

```text
L4 端口
L5 会话
L6 TLS
L7 软件配置 / 账号 / API
以及：
DNS / 代理 / VPN / 防火墙
```

---

# 四、一张万能排障决策树

下面这张流程可以直接给普通用户照着做。

```text
                    网络有问题
                        │
          ┌─────────────┴─────────────┐
          │                           │
       全部异常                    只有一个软件异常
          │                           │
       从 L1 开始                 L4-L7 优先
          │                           │
   Wi-Fi/网线已连接？                 │
      │          │                   │
     否          是                  │
      │          │                   │
     L1        检查 IP                │
                 │                    │
          IPv4 是 169.254.x.x？       │
             │        │              │
            是        否              │
             │        │              │
          L1/L2     ping 网关        │
                       │              │
                ┌──────┴──────┐       │
                │             │       │
               不通          通       │
                │             │       │
              L1/L2        ping 8.8.8.8
                              │
                       ┌──────┴──────┐
                       │             │
                      不通          通
                       │             │
                      L3          nslookup
                                    │
                              ┌──────┴──────┐
                              │             │
                            失败           成功
                              │             │
                            DNS         测端口
                                            │
                                      ┌─────┴─────┐
                                      │           │
                                     不通        通
                                      │           │
                                     L4       TLS/应用
```

---

# 五、第 1 层 · 物理层（Physical）

## 5.1 这一层普通人怎么理解

这一层就是：

> **“电脑和网络设备之间有没有真的连上。”**

包括：

- 网线
- 水晶头
- 网卡
- 网卡接口
- 交换机接口
- 路由器 LAN 口
- 光猫接口
- Wi-Fi 无线信号
- 无线网卡

如果这一层没有建立，后面的层没有意义。

---

## 5.2 典型症状

- 网卡显示“媒体已断开”
- 网线拔掉后没有反应
- Wi-Fi 搜不到指定无线网络
- Wi-Fi 信号极弱
- 网络图标红叉
- 网络连接一会儿有、一会儿没有
- 网卡在设备管理器里消失
- USB 网卡经常断开

---

## 5.3 第一检查动作

### Windows 图形界面

进入：

```text
设置
→ 网络和 Internet
→ 高级网络设置
→ 更多网络适配器选项
```

观察：

- 以太网是否已启用
- Wi-Fi 是否已启用
- 是否显示“已断开连接”

### 设备管理器

执行：

```text
devmgmt.msc
```

进入：

```text
网络适配器
```

查看有没有：

- 黄色感叹号
- 红色叉号
- 网卡完全消失

---

## 5.4 命令检查

```cmd
ipconfig /all
netsh interface show interface
netsh wlan show interfaces
netsh wlan show networks mode=bssid
```

### `ipconfig /all` 看什么

重点找：

```text
Ethernet adapter / Wireless LAN adapter
IPv4 Address
Subnet Mask
Default Gateway
DNS Servers
Physical Address
```

### 常见异常

#### A：Media disconnected

```text
Media disconnected
```

解释：

> 当前这个网卡没有建立有效物理/无线连接。

处理：

- 检查网线
- 换网线
- 换路由器接口
- 重新连接 Wi-Fi
- 启用网卡
- 检查驱动

#### B：169.254.x.x

这表示电脑没有拿到正常的 DHCP 地址。它不是“正常办公网络 IP”。

注意：

> 看到 169.254.x.x 时，不要只盯着 DNS。首先查“为什么没拿到 IP”。

---

## 5.5 修复顺序

推荐从最简单到最复杂：

### 第 1 步：检查硬件

```text
网线重新插拔
→ 换网线
→ 换 LAN 口
→ 换 USB 接口
→ 确认路由器/交换机有电
```

### 第 2 步：禁用再启用网卡

```cmd
netsh interface set interface "以太网" disable
netsh interface set interface "以太网" enable
```

如果你的网卡名称不是“以太网”，先查看：

```cmd
netsh interface show interface
```

### 第 3 步：重新获取地址

```cmd
ipconfig /release
ipconfig /renew
```

### 第 4 步：网络协议栈重置

```cmd
netsh winsock reset
netsh int ip reset
```

执行后通常需要重启。

### 第 5 步：驱动

图形界面：

```text
设备管理器
→ 网络适配器
→ 对应网卡
→ 属性
→ 驱动
```

可以考虑：

- 更新驱动
- 回滚驱动
- 卸载设备后重新扫描硬件

---

## 5.6 L1 不要误判

### “Wi-Fi 图标存在”不等于互联网正常

Wi-Fi 图标只能说明：

> 无线链路可能建立了。

它不能证明：

- DHCP 正常
- 网关正常
- DNS 正常
- 外网正常
- 软件服务器正常

所以 Wi-Fi 显示“已连接”后，仍然要继续查 L2/L3。

---

# 六、第 2 层 · 数据链路层（Data Link）

## 6.1 这一层普通人怎么理解

这一层负责：

> **“电脑和同一个局域网里的设备，能不能互相找到。”**

常见概念：

- MAC 地址
- ARP
- 二层交换
- 交换机端口
- VLAN

---

## 6.2 最典型症状

```text
电脑已经拿到 IP
但是：

ping 网关失败
```

例如：

```text
IPv4：192.168.1.100
网关：192.168.1.1

ping 192.168.1.1
Request timed out
```

此时优先检查 L1/L2，而不是 DNS。

---

## 6.3 为什么先看网关

电脑访问互联网时，第一站通常是默认网关。

如果：

```text
电脑 → 网关
```

这一跳都不通，访问：

```text
电脑 → 互联网
```

自然也很难成立。

---

## 6.4 命令

```cmd
ipconfig /all
arp -a
getmac /v
ping 你的网关
```

例如：

```cmd
ping 192.168.1.1
```

---

## 6.5 `arp -a` 怎么看

典型格式：

```text
Interface: 192.168.1.100
Internet Address      Physical Address      Type
192.168.1.1           aa-bb-cc-dd-ee-ff     dynamic
```

重点看：

```text
网关 IP → 是否出现 MAC
```

如果网关始终无法解析到 MAC，可能是：

- 二层通信异常
- VLAN 不一致
- 交换机端口异常
- ARP 异常
- 网关设备没有响应

注意：

> `arp -a` 没看到目标条目，并不能单独证明“一定是 L2 故障”。需要结合 ping、网络拓扑和设备状态判断。

---

## 6.6 修复

### 清 ARP 缓存

```cmd
arp -d *
```

需要管理员权限。

然后重新测试：

```cmd
ping 网关
arp -a
```

### 检查 IP 冲突

如果同一个 IP 被两台设备使用，可能出现：

- 网络时好时坏
- 某些设备访问正常，某些不正常
- ARP 表频繁变化

企业环境应进一步在交换机、DHCP、IPAM 等系统中核对。

### 检查 VLAN

企业网络常见问题：

```text
同一办公室
看起来都插在“网络口”
但实际被分到了不同 VLAN
```

普通用户通常无法自己修复，应提供：

- 电脑 MAC
- IP
- 端口位置
- 故障时间

给 IT/网络管理员。

---

# 七、第 3 层 · 网络层（Network）

## 7.1 这一层普通人怎么理解

这一层负责：

> **“数据应该往哪里走。”**

核心：

- IP 地址
- 子网掩码
- 默认网关
- 路由
- 静态路由

DNS 严格来说属于应用层协议，但为了实用排障，本文继续把“域名→IP”放在 L3→L7 的联合检查中处理。

---

## 7.2 L3 最关键的三步测试

按这个顺序：

### 测试 1：网关

```cmd
ping 你的网关
```

例如：

```cmd
ping 192.168.1.1
```

### 测试 2：公网 IP

```cmd
ping 8.8.8.8
```

### 测试 3：域名

```cmd
nslookup baidu.com
```

---

## 7.3 三个测试的意义

### 情况 A

```text
网关       √
8.8.8.8    ×
```

优先查：

```text
L3
```

包括：

- 默认路由
- WAN/出口
- VPN
- 路由器
- 运营商
- 企业防火墙

### 情况 B

```text
网关       √
8.8.8.8    √
域名解析   ×
```

优先查：

```text
DNS
```

### 情况 C

```text
网关       √
8.8.8.8    √
域名解析   √
```

说明：

> 基础 IP 连通和 DNS 基本都正常，问题继续向 L4~L7 排查。

---

## 7.4 查看本机网络参数

```cmd
ipconfig /all
```

必须重点看：

```text
IPv4 Address
Subnet Mask
Default Gateway
DNS Servers
```

### 正常思路

```text
IPv4：应该属于当前网络的合法地址范围
网关：应该是实际的网络出口地址
DNS：应该指向企业/路由器/运营商或明确配置的 DNS
```

---

## 7.5 查看路由表

```cmd
route print -4
```

重点寻找：

```text
0.0.0.0          0.0.0.0          默认网关
```

它表示默认路由。

简单理解：

> “不知道去哪的数据包，默认发给谁。”

### 如果看到多个默认路由

可能出现：

- VPN 抢路由
- 虚拟网卡优先级异常
- 多网络同时连接
- 某个软件创建了隧道

这时不要盲目删除路由，先记录：

```text
目标
网络掩码
网关
接口
跃点数
```

---

## 7.6 `tracert` 怎么用

```cmd
tracert -d 8.8.8.8
```

用途：

> 看数据包大致经过了哪些路由节点，以及在哪一段开始异常。

但要注意：

> `tracert` 某一跳出现 `*`，不等于这一跳一定“坏了”。很多网络设备会主动限制或不响应 TTL/探测报文。

正确理解是：

> 看“从哪里开始持续异常”，并结合最终目的地是否能到达判断。

---

## 7.7 L3 修复

### 重新获取 IP

```cmd
ipconfig /release
ipconfig /renew
```

### 清 DNS 缓存

```cmd
ipconfig /flushdns
```

### 检查 VPN

先断开 VPN，再重新测试：

```text
ping 网关
ping 8.8.8.8
nslookup baidu.com
```

如果断开 VPN 后恢复，说明重点转向 VPN / 路由 / DNS / 代理。

### 删除路由：谨慎

原稿提供：

```cmd
route delete 0.0.0.0
```

**v2.0 增加安全提醒：**不要把这条命令当作普通修复命令直接执行。删除默认路由可能导致整台机器失去互联网访问。

正确方式：

1. 先记录 `route print -4`
2. 确认哪条路由异常
3. 确定由哪个 VPN/虚拟网卡创建
4. 优先关闭/退出对应 VPN
5. 只有明确知道后果时再修改路由

---

# 八、第 4 层 · 传输层（Transport）

## 8.1 这一层普通人怎么理解

可以把“IP 地址”想成一栋楼的地址，端口就是：

> **“楼里的具体房间/服务入口。”**

例如：

```text
IP       = 哪台机器
Port 443 = 这台机器上的某个 HTTPS 服务入口
```

常见：

- TCP
- UDP
- 端口
- Socket
- 防火墙端口规则

---

## 8.2 为什么“某软件不能联网”经常查这里

因为不同软件可能使用：

- 不同域名
- 不同服务器
- 不同 TCP/UDP 端口
- 不同代理
- 不同协议

所以：

```text
浏览器正常
某软件异常
```

完全有可能只有该软件使用的端口被阻断。

---

## 8.3 最重要的测试

PowerShell：

```powershell
Test-NetConnection example.com -Port 443
```

重点看：

```text
TcpTestSucceeded : True
```

### True

说明：

> 从当前机器到目标的 TCP 端口测试成功。

继续往 L5/L6/L7 查。

### False

说明：

> 当前 TCP 端口测试没有建立成功。

重点查：

- DNS 是否解析到正确目标
- 目标服务是否监听
- 防火墙
- 代理
- VPN
- 上游 ACL
- 目标服务器

注意：不能简单地把 `False` 等价成“本机防火墙拦了”。

---

## 8.4 `netstat` 看什么

```cmd
netstat -ano
```

重点字段：

```text
Local Address
Foreign Address
State
PID
```

常见状态：

### LISTENING

某进程正在监听端口。

### ESTABLISHED

TCP 连接已经建立。

### SYN_SENT

本机已经发起连接请求，但还没建立成功。

### TIME_WAIT

连接已经关闭，系统暂时保留状态。

> `TIME_WAIT` 本身不是故障。大量 TIME_WAIT 需要结合连接模式、端口耗尽等现象判断。

---

## 8.5 端口→进程

```cmd
netstat -ano | findstr :8080
```

假设看到：

```text
TCP  0.0.0.0:8080  0.0.0.0:0  LISTENING  17400
```

最后一个：

```text
17400
```

就是 PID。

继续：

```cmd
tasklist /fi "PID eq 17400"
```

得到具体进程。

---

## 8.6 进程→端口

```cmd
tasklist | findstr /i 进程名
netstat -ano | findstr PID
```

或者 PowerShell：

```powershell
Get-NetTCPConnection -State Listen -OwningProcess (Get-Process 进程名).Id
```

---

## 8.7 修复端口冲突

如果一个程序必须监听：

```text
8080
```

但是另一个程序已经占用：

```text
8080
```

就会出现：

```text
Address already in use
bind failed
port is occupied
```

处理思路：

```text
先确认 PID
↓
确认是什么程序
↓
判断能不能停止
↓
能停 → 停止占用程序
不能停 → 改业务程序端口
```

不要看到陌生 PID 就直接：

```cmd
taskkill /f /pid xxx
```

---

# 九、第 5 层 · 会话层（Session）

## 9.1 这一层普通人怎么理解

重点不是：

> “能不能连上。”

而是：

> **“连上以后能不能保持。”**

例如：

```text
登录
↓
正在连接
↓
连接成功
↓
几秒后掉线
↓
自动重连
↓
再次掉线
```

这是典型的会话稳定性问题。

---

## 9.2 常见症状

- 刚连接就掉
- 每隔几十秒断一次
- VPN 连接成功但很快断开
- 远程桌面频繁重连
- 长时间空闲后连接失效
- 长连接应用经常重连

---

## 9.3 检查方法

```cmd
netstat -ano | findstr :443
```

可以连续执行几次，观察：

```text
ESTABLISHED
→ 消失
→ 再次 ESTABLISHED
```

说明连接可能在重复建立。

进一步可用：

```cmd
curl.exe -v https://example.com
```

观察连接建立后是否马上被关闭。

---

## 9.4 常见原因

- VPN/隧道不稳定
- 中间代理
- 防火墙状态超时
- 设备 NAT 会话超时
- MTU/分片相关问题
- 服务端主动断开
- 应用自身 keep-alive 配置

注意：

> “连上就断”不一定是 L5 独有的问题。L4/L6/L7 同样可能造成相似现象，因此要结合抓包和应用日志进一步确认。

---

# 十、第 6 层 · 表示层（Presentation）

## 10.1 这一层普通人怎么理解

可以简单理解为：

> **“双方能不能按相同的规则加密、解码、验证数据。”**

最常见的是：

- TLS/SSL
- 证书
- 加密算法协商
- 编码/压缩

---

## 10.2 最典型的症状

- Certificate expired
- Certificate verify failed
- SSL handshake failed
- TLS handshake error
- 系统时间不正确
- 某软件报证书不受信任
- 浏览器正常，但某软件证书错误

---

## 10.3 第一检查：系统时间

执行：

```cmd
w32tm /query /status
w32tm /query /source
```

因为证书有有效期。

如果系统时间明显错误，可能直接导致：

```text
证书还没生效
```

或者：

```text
证书已经过期
```

---

## 10.4 强制同步时间

管理员命令行：

```cmd
w32tm /resync
```

图形界面：

```text
设置
→ 时间和语言
→ 日期和时间
→ 自动设置时间
```

---

## 10.5 用 curl 看 TLS

```cmd
curl.exe -v https://example.com
```

重点观察：

- 是否成功建立 TCP
- TLS ClientHello/ServerHello
- 证书信息
- 是否出现 verify failed
- 最终是否拿到 HTTP 响应

如果：

```text
TCP 成功
TLS 失败
```

就不要继续折腾网卡，应优先查 L6。

---

## 10.6 企业代理 / MITM

企业网络可能使用 HTTPS 检查、代理或安全网关。

表现可能是：

```text
浏览器正常
某软件不正常
```

原因可能是：

- 浏览器信任企业根证书
- 某软件使用自己的证书库
- 某软件进行更严格的证书校验
- 安全软件重新签发了证书

处理：

> 不要为了“能访问”而直接关闭证书验证。应该确认企业代理的合法根证书是否正确部署，或由管理员确认安全设备策略。

---

# 十一、第 7 层 · 应用层（Application）

## 11.1 这一层普通人怎么理解

前面六层解决的是：

> **“能不能把数据送过去。”**

L7 解决的是：

> **“送过去以后，业务到底同不同意。”**

例如：

- 登录
- 密码验证
- API
- HTTP
- 文件上传
- 文件下载
- 软件服务器地址
- 软件账号权限

---

## 11.2 典型症状

- 401 Unauthorized
- 403 Forbidden
- 404 Not Found
- 500 Internal Server Error
- 登录失败
- 账号被禁用
- 接口返回错误
- 软件显示“服务器连接成功，但业务失败”

---

## 11.3 `curl -I` 看 HTTP

```cmd
curl.exe -I https://example.com
```

常见状态码：

| 状态码 | 通常表示 | 优先考虑 |
|---|---|---|
| 200 | 请求成功 | 网络和基本业务正常 |
| 301/302 | 重定向 | 继续观察目标地址 |
| 400 | 请求有问题 | 请求格式/参数 |
| 401 | 未认证 | 登录/凭证 |
| 403 | 被拒绝 | 权限/策略/WAF |
| 404 | 资源不存在 | URL/路径 |
| 429 | 请求太多 | 限流 |
| 500 | 服务端异常 | 服务端应用 |
| 502 | 网关错误 | 反向代理/上游 |
| 503 | 服务不可用 | 服务端/负载 |
| 504 | 网关超时 | 上游响应太慢/不可达 |

重要：

> HTTP 状态码只是定位线索，不应机械地把每个 4xx/5xx 都归结为同一个原因。

---

# 十二、“浏览器正常，某软件不正常”的专门排查路线

这是实际工作中非常常见的一类故障。

## 12.1 先不要怀疑网线

因为浏览器已经证明：

```text
网络基础链路基本可用
```

直接进入：

```text
1. 软件内置代理
2. 系统代理
3. WinHTTP
4. DNS
5. 目标端口
6. VPN
7. 防火墙/安全软件
8. TLS/证书
9. 软件服务器地址
10. 账号/业务权限
```

---

## 12.2 对照测试

假设：

```text
Chrome → example.com       √
业务软件 → example.com     ×
```

先做：

```cmd
nslookup example.com
```

再测：

```powershell
Test-NetConnection example.com -Port 443
```

再测：

```cmd
curl.exe -v https://example.com
```

如果：

```text
DNS √
TCP 443 √
curl √
业务软件 ×
```

那么问题高度集中在：

```text
L5 会话
L6 TLS/证书
L7 软件自身
代理
安全软件
```

---

# 十三、DNS 专项排查

## 13.1 DNS 是干什么的

把：

```text
example.com
```

变成：

```text
目标 IP
```

所以：

```text
IP 通
但域名不通
```

第一嫌疑就是 DNS。

---

## 13.2 基础检查

```cmd
nslookup example.com
```

看：

```text
Server
Address
Name
Address
```

### 正常

能得到明确的 IP 地址。

### 异常

例如：

```text
DNS request timed out
Non-existent domain
```

或者返回明显不符合预期的地址。

---

## 13.3 检查本机 DNS 配置

```cmd
ipconfig /all
```

看：

```text
DNS Servers
```

如果电脑连的是公司网络，却突然出现一个来源不明的 DNS 地址，应进一步确认配置来源。

---

## 13.4 清 DNS 缓存

```cmd
ipconfig /flushdns
```

然后重新：

```cmd
nslookup example.com
```

---

## 13.5 hosts 文件

Windows hosts：

```cmd
type %SystemRoot%\System32\drivers\etc\hosts
```

图形界面可以直接打开：

```text
C:\Windows\System32\drivers\etc\hosts
```

检查有没有针对目标域名的异常映射。

注意：

> 不要看到 hosts 文件里存在任何内容就认为是病毒。很多开发、企业内部系统确实会使用 hosts。应结合域名、IP、用途判断。

---

# 十四、代理专项排查

## 14.1 为什么代理会造成“有的软件能上，有的软件不能上”

因为 Windows 上常见的不只有一种代理机制。

至少要考虑：

```text
系统代理（WinINET）
WinHTTP 代理
环境变量代理
软件自己配置的代理
PAC 自动代理
VPN 内置代理
```

---

## 14.2 查 WinHTTP

```cmd
netsh winhttp show proxy
```

如果不需要代理而显示了代理服务器，需要结合企业环境确认是否为合理配置。

恢复 WinHTTP 直连：

```cmd
netsh winhttp reset proxy
```

---

## 14.3 查环境变量代理

```cmd
set | findstr /i proxy
```

重点关注：

```text
HTTP_PROXY
HTTPS_PROXY
ALL_PROXY
NO_PROXY
```

这对：

- Git
- Python
- Node.js
- Docker
- CLI 工具

非常重要。

---

## 14.4 查 Windows 用户代理

注册表：

```cmd
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer
```

如果确认应该直连，可以关闭代理：

```powershell
Set-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings' -Name ProxyEnable -Value 0
```

**企业环境注意：**如果公司统一代理是正常工作所必需的，不要擅自关闭。

---

# 十五、VPN / 隧道专项排查

## 15.1 VPN 为什么会“让原来正常的网络变坏”

因为 VPN 可能改变：

- 默认路由
- 特定目标路由
- DNS
- 代理
- 虚拟网卡
- MTU
- 防火墙策略

所以：

```text
VPN 断开 → 正常
VPN 连接 → 异常
```

是非常有价值的对照实验。

---

## 15.2 第一步：断开 VPN 再测

依次：

```cmd
ping 默认网关
ping 8.8.8.8
nslookup example.com
```

如果断开后恢复，重点看：

```text
route print -4
ipconfig /all
```

特别检查新增的虚拟网卡和默认路由。

---

## 15.3 VPN 场景常见现象

| 现象 | 优先怀疑 |
|---|---|
| VPN 一连接就无法上外网 | 默认路由/DNS/全隧道路由 |
| 内网能访问，公网不能 | Split Tunnel/企业策略 |
| 公网能访问，内网不能 | 企业路由/DNS/ACL |
| 只有某个内网域名打不开 | 企业 DNS / DNS split |
| VPN 能连接但几分钟后掉线 | 会话/网络稳定性/服务端 |

---

# 十六、防火墙与安全软件专项排查

## 16.1 防火墙并不等于“网络层坏了”

防火墙可能按照：

- 端口
- 程序
- 地址
- 网络配置文件
- 入站/出站

进行过滤。

因此出现：

```text
浏览器正常
某软件不能联网
```

也可能是安全策略只拦了某个程序。

---

## 16.2 查看 Windows 防火墙规则

```cmd
netsh advfirewall firewall show rule name=all
```

可以进一步按关键词筛选：

```cmd
netsh advfirewall firewall show rule name=all | findstr /i "程序名"
```

---

## 16.3 临时关闭防火墙进行验证？

原稿提供：

```cmd
netsh advfirewall set allprofiles state off
```

v2.0 强调：

> 这不是“修复命令”，只是一个高风险验证手段。办公电脑、生产电脑、服务器不建议普通用户自行关闭防火墙。

更安全的优先级：

```text
查看日志
↓
确认具体程序/端口
↓
检查现有规则
↓
添加最小范围允许规则
↓
重新测试
```

测试结束后，如果确实临时关闭过：

```cmd
netsh advfirewall set allprofiles state on
```

---

## 16.4 第三方安全软件

企业常见：

- EDR
- DLP
- 零信任客户端
- 上网行为管理
- HTTPS 检查
- 应用控制

表现可能是：

```text
只有某软件不能联网
同样账号换一台电脑正常
```

这时不要无限修改 Windows 网络配置，应检查安全软件日志或交给 IT/安全团队。

---

# 十七、共享文件夹 / 打印机专项排查

## 17.1 为什么共享和上网不是一回事

电脑可以：

```text
上互联网 √
```

但：

```text
访问局域网共享 ×
```

因为访问共享涉及：

- 局域网路径
- DNS/名称解析
- SMB
- TCP 445
- 防火墙
- 身份认证
- 权限

---

## 17.2 共享文件夹排查

假设共享服务器：

```text
FILESERVER
```

先测试名称：

```cmd
ping FILESERVER
```

再查看解析：

```cmd
nslookup FILESERVER
```

然后测试 SMB 端口：

```powershell
Test-NetConnection FILESERVER -Port 445
```

### 三种结果

```text
名称解析失败
→ DNS/NetBIOS/名称解析方向

445 不通
→ 防火墙/网络 ACL/服务方向

445 通，但访问拒绝
→ L7/身份/权限
```

---

## 17.3 打印机排查

先确认：

```text
打印机 IP
```

测试：

```cmd
ping 打印机IP
```

如果是网页管理：

```text
http://打印机IP
```

如果网络通但无法打印：

继续查：

- 打印机端口
- Windows 打印队列
- 驱动
- 打印服务器
- 权限

---

# 十八、企业办公网络常见场景

## 场景 1：同办公室别人都正常，只有你不能上网

优先：

```text
L1/L2
↓
IP
↓
网关
↓
DNS
↓
本机代理/VPN
↓
安全软件
```

因为企业出口通常不是第一嫌疑。

---

## 场景 2：办公室所有人都不能访问某系统

优先：

```text
企业出口
↓
DNS
↓
防火墙/ACL
↓
VPN
↓
目标服务器
```

不要让每个员工各自重装网卡。

---

## 场景 3：换一台电脑就正常

说明：

> 服务端和基础网络大概率正常，问题集中到原电脑。

重点比较：

```text
IP
DNS
代理
VPN
证书
安全软件
hosts
软件版本
```

---

## 场景 4：同一台电脑换手机热点就正常

这是非常有价值的对照实验。

说明原网络路径和热点路径存在差异。

优先比较：

```text
DNS
代理
VPN
路由
企业防火墙
出口策略
```

---

## 场景 5：同一网络，手机正常，电脑不正常

重点查：

```text
电脑自己的：
L1
L2
IP
DNS
代理
VPN
防火墙
软件
```

---

# 十九、常用命令逐个讲解

## 19.1 ipconfig

```cmd
ipconfig
```

作用：快速看 IPv4、网关。

完整版：

```cmd
ipconfig /all
```

适用：

```text
L1/L2/L3/DNS
```

---

## 19.2 ping

```cmd
ping 目标
```

用于判断 IP 层面的基本可达性。

常见用法：

```cmd
ping 127.0.0.1
ping 你的IP
ping 网关
ping 8.8.8.8
ping 域名
```

### 这五个测试分别可以帮助判断什么

```text
127.0.0.1
↓
本机 TCP/IP 栈基本测试

自己的 IP
↓
本机接口/协议栈相关测试

网关
↓
局域网到网关

8.8.8.8
↓
公网 IP 路径测试

域名
↓
DNS + IP 路径综合测试
```

注意：

> 某些服务器会禁止 ICMP，因此“ping 不通”不能直接证明服务器的 TCP/HTTPS 一定不可达。

---

## 19.3 nslookup

```cmd
nslookup example.com
```

作用：检查 DNS 解析。

---

## 19.4 tracert

```cmd
tracert -d 8.8.8.8
```

作用：查看路径上的路由跃点。

---

## 19.5 route

```cmd
route print -4
```

作用：查看 IPv4 路由表。

重点：

```text
0.0.0.0
```

---

## 19.6 arp

```cmd
arp -a
```

作用：查看 ARP 缓存。

---

## 19.7 netstat

```cmd
netstat -ano
```

作用：

```text
端口
连接
状态
PID
```

---

## 19.8 curl

```cmd
curl.exe -v https://example.com
```

这是一个非常重要的“跨层测试工具”。

它可以帮助你观察：

```text
DNS
↓
TCP
↓
TLS
↓
HTTP
```

因此非常适合“浏览器/软件到底卡在哪”的问题。

---

# 二十、端口 ↔ 进程专项工具

## 20.1 端口 → PID → 程序

```cmd
netstat -ano | findstr :8080
tasklist /fi "PID eq 17400"
```

---

## 20.2 程序 → PID → 端口

```cmd
tasklist | findstr /i 程序名
netstat -ano | findstr PID
```

---

## 20.3 PowerShell

```powershell
Get-NetTCPConnection -State Listen -LocalPort 8080 | Select LocalAddress,LocalPort,OwningProcess
```

---

## 20.4 杀进程前必须回答 3 个问题

```text
这个 PID 是什么程序？
↓
这个程序为什么运行？
↓
杀掉它会不会影响别人/系统？
```

如果不知道：

> 不要直接 `taskkill /f`。

---

# 二十一、Wireshark 进阶排查思路

Wireshark 属于进阶工具，普通用户可以先学会“知道什么时候该用”。

## 21.1 什么时候需要抓包

当你已经知道：

```text
IP 正常
端口疑似正常
但连接就是反复失败
```

或者：

```text
软件报错信息非常模糊
```

可以考虑抓包。

---

## 21.2 常见过滤思路

例如只看某个 IP：

```text
ip.addr == 192.168.1.100
```

看 HTTPS：

```text
tcp.port == 443
```

看 DNS：

```text
dns
```

看 TCP 重传：

```text
tcp.analysis.retransmission
```

看 TCP reset：

```text
tcp.flags.reset == 1
```

---

## 21.3 抓包最重要的问题不是“看到多少包”

而是回答：

> **“是谁先出问题。”**

例如：

```text
客户端发 SYN
↓
服务器没有回应
```

与：

```text
客户端发 SYN
↓
服务器 SYN/ACK
↓
客户端建立连接
↓
服务器立即 RST
```

这是两类不同的问题。

---

# 二十二、修复动作的安全边界

## 22.1 低风险：普通用户可以优先做

```text
重新连接 Wi-Fi
重新插拔网线
重新获取 IP
flushdns
关闭并重新打开软件
重启电脑
重新连接 VPN
检查系统时间
```

---

## 22.2 中风险：应记录修改前状态

```text
修改代理
修改 DNS
修改 hosts
修改网卡配置
修改应用端口
调整 MTU
```

要求：

> **改之前先记录原值。**

---

## 22.3 高风险：没有明确原因不要执行

```text
route delete
大范围修改防火墙
关闭所有防火墙
删除系统网络配置
删除未知服务
批量杀进程
修改企业安全客户端策略
```

---

# 二十三、常见故障案例库

## 案例 1：Wi-Fi 显示已连接，但是网页打不开

### 现象

```text
Wi-Fi √
网页 ×
```

### 排查

```cmd
ipconfig
ping 网关
ping 8.8.8.8
nslookup baidu.com
```

### 判断

```text
网关 × → L1/L2
网关 √ + 8.8.8.8 × → L3
公网 IP √ + DNS × → DNS
三者都 √ → L4~L7
```

---

## 案例 2：浏览器正常，某客户端打不开

### 排查

```cmd
nslookup 软件服务器域名
```

```powershell
Test-NetConnection 软件服务器域名 -Port 443
```

```cmd
curl.exe -v https://软件服务器域名
```

然后检查：

```text
软件代理
VPN
证书
安全软件
账号
```

---

## 案例 3：只有一个网站打不开

先测试：

```cmd
nslookup 网站域名
```

再：

```powershell
Test-NetConnection 网站域名 -Port 443
```

再：

```cmd
curl.exe -v https://网站域名
```

判断：

```text
DNS × → DNS
TCP × → L4/路径/服务端
TCP √ TLS × → L6
HTTP 返回 4xx/5xx → L7/服务端
```

---

## 案例 4：某软件提示“服务器连接失败”，但浏览器能访问同服务器

优先比较：

```text
软件 DNS
软件代理
软件端口
软件证书
软件是否使用 IPv4/IPv6
软件是否有独立网络配置
```

还要考虑：

```text
EDR/安全软件是否只拦这个 EXE
```

---

## 案例 5：连接 VPN 后所有网站都打不开

### 立即做对照实验

```text
VPN 断开 → 测试
VPN 连接 → 测试
```

查看：

```cmd
ipconfig /all
route print -4
netsh winhttp show proxy
```

重点寻找：

- 新增虚拟网卡
- 默认路由变化
- DNS 变化
- 代理变化

---

## 案例 6：共享文件夹打不开，但网页正常

测试：

```cmd
ping 文件服务器
nslookup 文件服务器
```

然后：

```powershell
Test-NetConnection 文件服务器 -Port 445
```

如果 445 通但“拒绝访问”：

> 网络路径大概率已经能到，继续查账号、共享权限、NTFS 权限、SMB 配置等。

---

## 案例 7：别人能打印，我不能打印

先测试打印机 IP：

```cmd
ping 打印机IP
```

再检查：

- 打印机端口
- 驱动
- 打印队列
- Windows Print Spooler
- 本机防火墙
- 打印服务器

---

## 案例 8：网页提示证书错误

先查：

```cmd
w32tm /query /status
```

再看：

```text
电脑时间
时区
证书有效期
证书签发者
```

如果是公司网络：

> 再确认是否存在 HTTPS 检查代理和企业根证书。

---

## 案例 9：某网站打开特别慢，但不是完全打不开

不要只看“能不能 ping”。

建议：

```cmd
ping 域名
tracert -d 域名
curl.exe -v https://域名
```

进一步可以用：

```text
Wireshark
浏览器开发者工具 Network
```

区分：

- DNS 慢
- TCP 建连慢
- TLS 握手慢
- 服务端响应慢
- 页面资源慢

---

## 案例 10：网络时好时坏

优先记录：

```text
发生时间
Wi-Fi 信号
IP 是否变化
ping 网关是否丢包
ping 公网是否丢包
VPN 是否连接
是否只有某软件
```

如果网关都丢包：

```text
L1/L2 优先
```

如果网关稳定、公网丢包：

```text
L3/出口优先
```

如果网络稳定但某应用断：

```text
L4~L7
```

---

# 二十四、排障记录模板

发生故障时，直接复制下面模板。

```text
================ 网络故障记录 ================

用户：
电脑：
系统：
时间：
网络：Wi-Fi / 网线 / VPN / 手机热点

【一、故障现象】
现象：
影响的软件/网站：
是否所有软件都受影响：

【二、影响范围】
其他电脑：正常 / 异常
其他手机：正常 / 异常
换网络：正常 / 异常

【三、L1/L2】
ipconfig：
IPv4：
网关：
Wi-Fi/网线状态：

ping 网关：
结果：

arp -a：
结果：

【四、L3】
ping 8.8.8.8：
结果：

route print -4：
是否存在异常默认路由：

nslookup 目标域名：
结果：

【五、L4】
目标域名：
目标端口：
Test-NetConnection：
结果：

netstat：
结果：

【六、L5/L6】
curl -v：
TLS/SSL错误：
系统时间：

【七、L7】
HTTP状态码：
软件报错：
账号状态：
软件代理：

【八、修改记录】
修改前：
修改内容：
修改后：

【九、最终定位】
OSI层：
根因：
解决方式：
================================================
```

---

# 二十五、一页速查表

## 25.1 现象 → 层

| 看到什么 | 先查 |
|---|---|
| 网线未连接 | L1 |
| Wi-Fi 搜不到 | L1 |
| 169.254.x.x | L1/L2 |
| IP 有，但网关不通 | L2 |
| 网关通，公网 IP 不通 | L3 |
| 公网 IP 通，域名不通 | DNS |
| ping 通，443 不通 | L4 |
| TCP 通，连接频繁掉 | L5 |
| TLS/证书错误 | L6 |
| 401/403/业务错误 | L7 |
| 只有一个软件异常 | L4~L7 |
| 换手机热点恢复 | 原网络路径/DNS/代理/VPN 优先 |
| 换电脑恢复 | 原电脑配置优先 |

---

## 25.2 命令 → 用途

| 命令 | 用途 |
|---|---|
| `ipconfig /all` | IP、网关、DNS、MAC |
| `netsh interface show interface` | 网卡状态 |
| `netsh wlan show interfaces` | Wi-Fi 状态 |
| `ping 网关` | 局域网/网关可达性 |
| `ping 8.8.8.8` | 公网 IP 路径 |
| `ping 域名` | DNS + IP 综合 |
| `arp -a` | IP→MAC |
| `route print -4` | 路由表 |
| `tracert -d` | 路径定位 |
| `nslookup` | DNS |
| `ipconfig /flushdns` | 清 DNS 缓存 |
| `ipconfig /release` | 释放 DHCP 地址 |
| `ipconfig /renew` | 重新获取 DHCP 地址 |
| `Test-NetConnection` | 测 TCP 端口 |
| `netstat -ano` | 连接/监听/PID |
| `tasklist` | PID→程序 |
| `curl.exe -v` | DNS/TCP/TLS/HTTP 综合检查 |
| `w32tm` | 系统时间 |
| `netsh winhttp show proxy` | WinHTTP 代理 |
| `set | findstr /i proxy` | 环境变量代理 |
| `type ...hosts` | hosts |
| `netsh advfirewall ...` | Windows 防火墙 |

---

# 二十六、最终心法

## 26.1 第一原则：先确定影响范围

```text
一个软件
一个网站
一台电脑
一个办公室
全公司
```

影响范围越大，越应该向下/向基础设施方向考虑。

---

## 26.2 第二原则：从低层到高层，但不是死板地从 L1 一直查到 L7

### 全断

```text
L1 → L2 → L3 → L4 → L5 → L6 → L7
```

### 个别软件

```text
L4 → L5 → L6 → L7
```

同时穿插：

```text
DNS / 代理 / VPN / 防火墙
```

---

## 26.3 第三原则：每一次测试都要有目的

不要：

```text
“我先 ping 一下。”
```

应该是：

```text
“我 ping 网关，是为了判断电脑到局域网出口是否正常。”
```

```text
“我 ping 8.8.8.8，是为了绕开 DNS 测公网 IP 连通性。”
```

```text
“我 nslookup，是为了确认域名有没有被正确解析。”
```

```text
“我 Test-NetConnection 443，是为了确认 TCP 端口能不能建立连接。”
```

```text
“我 curl -v，是为了继续区分 TCP、TLS 还是 HTTP 层。”
```

---

## 26.4 第四原则：一次只改一个变量

例如：

```text
先只断开 VPN
测试
↓
恢复 VPN
↓
只改 DNS
测试
↓
恢复 DNS
↓
再测试代理
```

这样才能知道真正原因。

---

## 26.5 第五原则：先记录，后修改

尤其是：

```text
代理
DNS
路由
hosts
防火墙
VPN
MTU
```

建议保存：

```cmd
ipconfig /all
route print -4
netstat -ano
netsh winhttp show proxy
```

---

## 26.6 第六原则：最有价值的不是“修好了”，而是“知道为什么”

真正优秀的排障结果应该是：

```text
现象
↓
测试
↓
证据
↓
定位
↓
修改
↓
复测
↓
确认根因
```

例如：

```text
只有业务软件不能登录
↓
nslookup 正常
↓
443 正常
↓
curl 正常
↓
软件独有连接失败
↓
发现软件代理指向已失效地址
↓
关闭软件代理
↓
重新登录成功
```

这才是一条完整的排障链。

---

# 附录 A：给完全不懂网络的人看的“最短路线”

当你什么都不会时，只做下面这些。

```text
① 看 Wi-Fi/网线有没有连接
        ↓
② 打开 CMD
        ↓
③ ipconfig
        ↓
④ 找 Default Gateway / 默认网关
        ↓
⑤ ping 网关
        ↓
⑥ ping 8.8.8.8
        ↓
⑦ nslookup 你要访问的域名
        ↓
⑧ 如果某软件仍不行：
   Test-NetConnection 域名 -Port 443
        ↓
⑨ curl.exe -v https://域名
        ↓
⑩ 把这些结果发给 IT
```

---

# 附录 B：交给 IT 时，至少提供什么

不要只说：

> “网络坏了。”

至少提供：

```text
1. 什么软件/网站打不开
2. 从什么时候开始
3. 是只有你还是所有人
4. Wi-Fi/网线
5. IPv4 地址
6. 默认网关
7. ping 网关结果
8. ping 8.8.8.8 结果
9. nslookup 结果
10. Test-NetConnection 结果
11. 软件完整报错文字
12. 最近是否装过 VPN/代理/安全软件
13. 是否换手机热点后恢复
```

这样 IT 可以大幅减少“先重启电脑试试”的无效来回。

---

# 附录 C：原版核心口诀保留

> **物链网传会话表应。**
>
> **全断从 L1 往上查；个别软件连不上，优先查 L4~L7，同时检查 DNS、代理、VPN、防火墙。**
>
> **网关不通 → 局域网优先；公网 IP 不通 → 路由/出口优先；公网 IP 通但域名不通 → DNS 优先；TCP 通但 TLS 失败 → 证书/TLS；HTTP 通但业务失败 → 应用层。**

---

# 版本说明

**v2.0 相比原稿新增：**

- 面向零基础用户的解释方式
- 10 秒定位法
- 万能排障决策树
- 每一层“先看什么、再看什么、什么时候停止”的操作顺序
- Windows 图形界面路径
- 命令逐字段解释
- DNS 专项
- 代理专项
- VPN/隧道专项
- 防火墙/安全软件专项
- 共享文件夹/打印机专项
- 企业办公网络场景
- Wireshark 进阶思路
- 10 个典型故障案例
- 排障记录模板
- 风险分级与回滚原则
- 一页速查表

**本文没有把 OSI 七层当成现实网络组件的绝对边界，而是把它作为排障顺序和定位语言。**
