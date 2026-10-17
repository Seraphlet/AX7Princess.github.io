---
description: ""
title: "Linux 运维排障：从网络异常到内存问题的完整定位思路"
draft: false
date: "2026-10-17T11:59:30+08:00"
slug: "LinuxOpsCommand"
categories:
 - 
tags:
 - Linxu
image: ""
---

# Linux 运维排障：从网络异常到内存问题的完整定位思路

上一篇最重要的骨架是：

```text
现象
 ↓
系统
 ↓
资源
 ↓
连接
 ↓
进程
 ↓
应用
 ↓
日志
 ↓
根因
```

但实际操作还有一个问题：

> 我知道应该“检查内存”，可我不知道检查内存应该输入什么。

因此还需要建立第二层知识：

```text
我想知道什么？
      ↓
应该输入什么？
      ↓
输出看什么？
      ↓
异常意味着什么？
      ↓
下一步输入什么？
```

这部分就是 Linux 排障的实际操作层。

---

# 一、先记住一个原则：不要背命令，记“问题 → 命令”

例如：

```text
我想知道机器整体忙不忙
→ uptime

我想知道 CPU / 内存谁有问题
→ top

我想知道还有多少内存
→ free -h

我想知道谁最吃内存
→ ps aux --sort=-%mem

我想知道 8000 端口有没有启动
→ ss -lntp | grep :8000

我想知道哪个程序占着 8000
→ ss -lntp | grep :8000

我想知道一个 PID 到底是什么
→ ps -fp PID

我想知道服务器有没有 OOM
→ journalctl -k | grep -i oom

我想知道目标 IP 通不通
→ ping

我想知道目标端口通不通
→ nc

我想知道 HTTP 到底哪里失败
→ curl -v
```

这样记会比记：

```text
top
ps
ss
free
vmstat
ip
```

容易得多。

因为现实中我们永远是：

> **先有问题，再找工具。**

---

# 二、场景 1：服务器突然很卡，我第一步敲什么？

先不要猜。

执行：

```bash
uptime
```

例如：

```text
19:32:10 up 32 days, 4:12, 2 users,
load average: 0.50, 0.42, 0.39
```

主要看：

```text
load average
```

如果明显异常，继续：

```bash
top
```

现在不要试图把 `top` 全看懂。

只看：

```text
load average

%Cpu(s)

MiB Mem

MiB Swap
```

以及下面：

```text
PID
%CPU
%MEM
COMMAND
```

形成第一轮判断：

```text
服务器卡
 │
 ├─ CPU 高？
 │    ↓
 │   查高 CPU 进程
 │
 ├─ Memory 紧？
 │    ↓
 │   查高内存进程
 │
 └─ wa 高？
      ↓
     查 IO / Swap
```

---

# 三、CPU 很高：到底是谁？

直接：

```bash
ps aux --sort=-%cpu | head
```

意思不用死记参数，可以直接记成：

> 给我看看 CPU 用得最多的进程。

例如：

```text
USER   PID    %CPU   %MEM   COMMAND
root   4211   95.2   12.1   python app.py
mysql  1123   20.1   15.3   mysqld
```

马上得到：

```text
CPU 高
 ↓
PID 4211
 ↓
python app.py
```

继续调查这个 PID：

```bash
ps -fp 4211
```

如果想持续观察：

```bash
top -p 4211
```

于是操作链就是：

```text
CPU 高
 ↓
ps aux --sort=-%cpu | head
 ↓
找到 PID
 ↓
ps -fp PID
 ↓
top -p PID
```

---

# 四、内存很高：先别杀进程

第一条：

```bash
free -h
```

例如：

```text
               total    used    free   shared   buff/cache   available
Mem:            16Gi     13Gi    300Mi    200Mi      2.7Gi       2.3Gi
Swap:            4Gi      1Gi      3Gi
```

重点不是：

```text
free
```

而是：

```text
available
```

简单记：

```text
free
= 完全没人用的

available
= Linux 估计还能给应用使用的
```

所以：

```text
free 很低
available 还很多
```

先不用慌。

真正需要继续调查的是：

```text
available 很低
```

---

# 五、谁最吃内存？

执行：

```bash
ps aux --sort=-%mem | head
```

例如：

```text
USER   PID    %CPU   %MEM   COMMAND
root   4211   10.2   40.1   python app.py
mysql  1123    5.2   15.3   mysqld
```

发现：

```text
python
PID 4211
```

继续：

```bash
ps -p 4211 -o pid,ppid,%cpu,%mem,rss,vsz,cmd
```

重点：

```text
RSS
```

RSS 可以先理解为：

> 这个进程现在真正驻留在物理内存里的量。

如果：

```text
RSS = 5242880 KB
```

大约就是：

```text
5 GB
```

于是已经从：

```text
服务器内存很高
```

缩小成：

```text
PID 4211 的 Python
实际占用了大约 5 GB RAM
```

---

# 六、怀疑内存泄漏：不要只看一次

内存泄漏是：

```text
越来越大
```

而不是：

```text
现在很大
```

所以持续观察：

```bash
watch -n 2 'ps -p 4211 -o pid,%cpu,%mem,rss,vsz,cmd'
```

意思：

```text
每 2 秒
重新执行一次
```

你可能看到：

```text
20:00 RSS 1.2G
20:01 RSS 1.3G
20:02 RSS 1.5G
20:03 RSS 1.8G
20:04 RSS 2.1G
```

这时候才值得怀疑：

```text
持续申请
   ↓
没有释放
   ↓
内存不断上涨
```

然后进入 Python / Java / Node 等应用内部排查。

---

# 七、服务器越来越卡：是不是 Swap？

先：

```bash
free -h
```

发现：

```text
Swap:
4G total
3G used
```

不要仅凭 `Swap used` 一个数字下结论。

继续：

```bash
vmstat 1
```

这里的 `1` 表示：

```text
每秒刷新一次
```

输出大概：

```text
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa
 1  0 100000 300000 10000 500000    0    0    10    20  100  200 20  5 70  5
```

第一次看很恐怖。

现在只盯：

```text
si
so
```

可以先记：

```text
si = swap in
so = swap out
```

如果持续：

```text
si 很高
so 很高
```

说明：

```text
RAM
 ↕
Swap
 ↕
磁盘
```

系统正在频繁搬内存页。

这可能就是：

```text
CPU 看着不高
但机器特别卡
```

的原因。

---

# 八、程序突然消失：是不是 OOM？

这是服务器排障非常值得记住的一条。

例如：

```text
昨天程序还好好的
今天进程没了

Python 日志最后也没有 traceback
```

不要只想着：

```text
程序崩了
```

先检查：

```bash
journalctl -k | grep -i -E 'oom|killed process|out of memory'
```

或者：

```bash
dmesg | grep -i -E 'oom|killed process|out of memory'
```

如果看到：

```text
Out of memory
Killed process 4211 (python)
```

那证据已经很强：

```text
内存耗尽
 ↓
Kernel OOM
 ↓
Linux 杀掉 Python
 ↓
服务消失
```

这时候重启 Python 只能恢复服务。

真正还要解决的是：

```text
为什么内存耗尽？
```

---

# 九、我想知道服务器自己的 IP

执行：

```bash
ip addr
```

简写：

```bash
ip a
```

可能看到：

```text
2: eth0:
    inet 192.168.1.20/24
```

那么：

```text
网卡：eth0
IP：192.168.1.20
```

如果只想快速看：

```bash
hostname -I
```

可能得到：

```text
192.168.1.20
```

---

# 十、我想知道服务器默认从哪个网卡出去

执行：

```bash
ip route
```

例如：

```text
default via 192.168.1.1 dev eth0

192.168.1.0/24 dev eth0
```

可以理解：

```text
default
 ↓
不知道目标在哪时
 ↓
交给 192.168.1.1
 ↓
从 eth0 出去
```

也就是：

```text
服务器
   │
 eth0
   │
   ▼
192.168.1.1
   │
   ▼
外部网络
```

---

# 十一、我想知道“访问这个 IP 会走哪条路”

这个命令非常实用：

```bash
ip route get 8.8.8.8
```

例如：

```text
8.8.8.8 via 192.168.1.1 dev eth0 src 192.168.1.20
```

翻译：

```text
我要访问：

8.8.8.8

↓

下一跳：

192.168.1.1

↓

使用网卡：

eth0

↓

使用本机 IP：

192.168.1.20
```

所以以后你怀疑：

```text
是不是走错网卡？
是不是路由有问题？
```

可以直接：

```bash
ip route get 目标IP
```

---

# 十二、我想知道“服务器有没有网”

不要只 ping 一个域名。

第一步：

```bash
ping -c 4 1.1.1.1
```

这里：

```text
-c 4
```

表示只发 4 次。

如果：

```text
IP 能 ping
```

再：

```bash
ping -c 4 example.com
```

于是有一个非常实用的判断：

```text
1.1.1.1 成功
example.com 成功

→ 基本网络正常


1.1.1.1 成功
example.com 失败

→ 优先怀疑 DNS


1.1.1.1 失败

→ 网络 / 路由 / 防火墙等继续调查
```

注意：

```text
ping 失败
≠
目标一定挂了
```

因为 ICMP 可以被禁用。

所以 ping 只是一个证据。

---

# 十三、我怀疑 DNS

执行：

```bash
dig example.com
```

如果没有 `dig`，有些机器可以：

```bash
nslookup example.com
```

我们真正问的是：

```text
example.com
     ↓
DNS
     ↓
到底解析成什么 IP？
```

例如：

```text
example.com
 ↓
93.184.216.34
```

如果 DNS 根本解析失败：

```text
域名访问失败
```

就不一定是 HTTP 服务的问题。

---

# 十四、我想知道某个端口到底开没开

这是以后开发 Agent、FastAPI、数据库服务经常会用的。

例如：

```text
我启动 FastAPI 8000
```

检查：

```bash
ss -lntp | grep :8000
```

如果出现：

```text
LISTEN 0 2048 0.0.0.0:8000 ...
```

说明：

```text
8000
 ↓
有人监听
```

如果什么都没有：

```text
8000
 ↓
没有 TCP 服务监听
```

---

# 十五、我想知道是谁占了这个端口

还是：

```bash
ss -lntp | grep :8000
```

例如：

```text
LISTEN 0 2048 0.0.0.0:8000
users:(("python",pid=4211,fd=6))
```

直接得到：

```text
Port 8000
 ↓
python
 ↓
PID 4211
```

继续：

```bash
ps -fp 4211
```

就可以知道：

```text
这个 Python 到底是什么程序。
```

这条链非常值得记：

```text
Port
 ↓
ss
 ↓
PID
 ↓
ps
 ↓
Program
```

---

# 十六、我的程序明明启动了，为什么其他电脑访问不了？

先：

```bash
ss -lntp | grep :8000
```

如果：

```text
127.0.0.1:8000
```

这是一个非常强的线索。

意味着：

```text
只监听 localhost
```

本机：

```bash
curl http://127.0.0.1:8000
```

可能成功。

但是：

```text
其他电脑
 ↓
服务器IP:8000
 ↓
失败
```

如果服务本来就应该对外开放，再检查应用监听配置是否应该是：

```text
0.0.0.0:8000
```

FastAPI/Uvicorn 常见：

```bash
uvicorn main:app --host 0.0.0.0 --port 8000
```

注意：

```text
0.0.0.0
```

不是一个让浏览器访问的目标地址。

它表示：

> 服务监听本机所有适用网络接口。

---

# 十七、我想知道另一台服务器的 8000 端口能不能连

例如：

```text
目标：
192.168.1.20:8000
```

执行：

```bash
nc -vz 192.168.1.20 8000
```

如果成功：

```text
Connection succeeded
```

说明：

```text
本机
 ↓
TCP
 ↓
192.168.1.20:8000
```

至少连接建立成功。

如果：

```text
Connection refused
```

通常优先调查：

```text
目标到了
 ↓
但是端口没有接受连接
```

如果：

```text
timed out
```

则继续考虑：

```text
防火墙
安全组
路由
网络路径
```

---

# 十八、HTTP 服务有问题，优先学会 curl -v

例如：

```bash
curl -v http://192.168.1.20:8000
```

它非常适合回答：

> HTTP 请求到底走到哪一步了？

可能看到：

```text
Trying 192.168.1.20:8000...
Connected to 192.168.1.20
```

这已经证明：

```text
TCP Connect
✓
```

然后：

```text
> GET / HTTP/1.1
```

说明 HTTP 请求已经发出。

如果返回：

```text
< HTTP/1.1 500 Internal Server Error
```

那么：

```text
网络
✓

TCP
✓

HTTP
✓

Application
×
```

这时候就不要继续折腾：

```text
ping
防火墙
路由
```

而应该：

```text
查应用日志。
```

---

# 十九、我想知道数据包到底走到哪里断了

可以：

```bash
tracepath example.com
```

或者系统安装了 traceroute：

```bash
traceroute example.com
```

可以把它理解成：

```text
我的服务器
 ↓
Router A
 ↓
Router B
 ↓
Router C
 ↓
目标服务器
```

如果路径问题需要进一步定位，它可以帮助判断：

```text
到底在哪一段开始没有响应。
```

不过中间设备可能不响应探测，因此某一跳出现 `*` 并不能单独证明那个路由器故障。

---

# 二十、我想看服务器现在有多少 TCP 连接

执行：

```bash
ss -ant
```

如果只想看已建立连接：

```bash
ss -ant state established
```

如果想看 TCP 整体统计：

```bash
ss -s
```

这个非常适合先快速看看：

```text
TCP 到底忙不忙。
```

---

# 二十一、我怀疑 CLOSE-WAIT 堆积

直接：

```bash
ss -ant state close-wait
```

统计数量：

```bash
ss -ant state close-wait | wc -l
```

如果数量持续增加：

```text
10
 ↓
50
 ↓
200
 ↓
1000
```

就值得进一步调查：

```text
哪个应用没有正确处理连接关闭？
```

如果需要进程信息，可以继续：

```bash
ss -antp state close-wait
```

然后：

```text
Socket
 ↓
PID
 ↓
Process
 ↓
Application
```

---

# 二十二、我怀疑连接特别多

查看连接来源：

```bash
ss -ant
```

如果只是某个端口，例如 8000：

```bash
ss -ant | grep ':8000'
```

如果想简单统计：

```bash
ss -ant | grep ':8000' | wc -l
```

于是：

```text
感觉连接很多
```

变成：

```text
当前匹配连接约 238 个
```

这就是：

```text
感觉
 ↓
数据
```

---

# 二十三、服务启动失败，我怀疑端口被占了

例如 FastAPI 报：

```text
Address already in use
```

马上：

```bash
ss -lntp | grep :8000
```

得到：

```text
python
PID 4211
```

然后：

```bash
ps -fp 4211
```

先确认：

```text
它是谁？
该不该存在？
```

不要看到 PID 就直接：

```bash
kill -9 4211
```

因为你可能杀的是另一个正常服务。

---

# 二十四、我要停止一个确定有问题的进程

先确认：

```bash
ps -fp 4211
```

确定以后优先：

```bash
kill 4211
```

这默认发送 `SIGTERM`，给程序正常退出的机会。

然后检查：

```bash
ps -p 4211
```

如果确实无法退出、并且你已经确认可以强制终止，最后才考虑：

```bash
kill -9 4211
```

可以理解：

```text
kill PID
 ↓
“请你正常退出”

kill -9 PID
 ↓
“内核直接终止你”
```

所以：

> `kill -9` 不应该成为第一反应。

---

# 二十五、服务挂了，我想看 systemd 日志

假设服务叫：

```text
myapp.service
```

先：

```bash
systemctl status myapp
```

看看：

```text
running？
failed？
什么时候退出？
exit code？
```

再：

```bash
journalctl -u myapp
```

看这个服务的日志。

只看最近：

```bash
journalctl -u myapp -n 100
```

持续跟踪：

```bash
journalctl -u myapp -f
```

于是：

```text
服务挂了
 ↓
systemctl status
 ↓
确认状态
 ↓
journalctl
 ↓
找错误
```

---

# 二十六、我不知道系统刚才发生了什么

看系统日志：

```bash
journalctl
```

内核：

```bash
journalctl -k
```

最近的：

```bash
journalctl -n 100
```

从今天开始：

```bash
journalctl --since today
```

最近 10 分钟：

```bash
journalctl --since "10 minutes ago"
```

于是如果：

```text
刚才 19:20 服务突然挂了
```

就不用翻一天日志。

直接围绕时间找：

```bash
journalctl --since "19:15" --until "19:25"
```

这是非常重要的排障习惯：

```text
知道故障时间
 ↓
缩小日志时间窗口
```

---

# 二十七、磁盘满了也可能表现成“程序异常”

虽然本文主要是网络和内存，但这个必须会。

先：

```bash
df -h
```

例如：

```text
Filesystem      Size  Used Avail Use%
/dev/sda1       100G   98G   2G   98%
```

马上发现：

```text
磁盘 98%
```

如果想知道哪个目录大：

```bash
du -sh /*
```

但生产机器根目录这样扫可能较慢，所以更稳妥地逐步缩小，例如：

```bash
du -sh /var/*
```

发现：

```text
/var/log 40G
```

继续：

```bash
du -sh /var/log/*
```

形成：

```text
Disk Full
 ↓
df -h
 ↓
哪个文件系统
 ↓
du
 ↓
哪个目录
 ↓
继续 du
 ↓
哪个文件
```

---

# 二十八、我不知道一个命令怎么用怎么办？

Linux 自己就有说明书。

例如不知道 `ss`：

```bash
man ss
```

不知道 `free`：

```bash
man free
```

不知道 `ps`：

```bash
man ps
```

快速看帮助：

```bash
ss --help
```

或者：

```bash
ip help
```

所以以后遇到：

```text
我知道应该查 TCP
但忘记 ss 参数
```

完全可以：

```bash
man ss
```

你不需要记住 Linux 的所有参数。

---

# 二十九、最终：我的 Linux 排障命令地图

以后可以直接从这张表开始。

| 我现在想知道什么 | 命令 |
|---|---|
| 服务器整体忙不忙 | `uptime` |
| CPU / 内存整体状态 | `top` |
| 内存还有多少 | `free -h` |
| 内存 / Swap / IO 动态变化 | `vmstat 1` |
| 谁最吃 CPU | `ps aux --sort=-%cpu \| head` |
| 谁最吃内存 | `ps aux --sort=-%mem \| head` |
| 这个 PID 是谁 | `ps -fp PID` |
| PID 实际内存情况 | `ps -p PID -o pid,ppid,%cpu,%mem,rss,vsz,cmd` |
| 持续观察某个进程 | `top -p PID` |
| 是否发生 OOM | `journalctl -k \| grep -i oom` |
| 本机 IP | `ip addr` |
| 默认路由 | `ip route` |
| 到目标 IP 怎么走 | `ip route get IP` |
| IP 能不能到 | `ping -c 4 IP` |
| DNS 是否正常 | `dig DOMAIN` |
| 网络经过哪些节点 | `tracepath HOST` |
| 哪些 TCP 端口在监听 | `ss -lntp` |
| 8000 是否监听 | `ss -lntp \| grep :8000` |
| 谁占了 8000 | `ss -lntp \| grep :8000` |
| 当前 TCP 连接 | `ss -ant` |
| TCP 总体统计 | `ss -s` |
| CLOSE-WAIT | `ss -ant state close-wait` |
| 对方端口能不能连 | `nc -vz HOST PORT` |
| HTTP 请求哪里失败 | `curl -v URL` |
| 服务状态 | `systemctl status SERVICE` |
| 服务日志 | `journalctl -u SERVICE` |
| 最近系统日志 | `journalctl -n 100` |
| 内核日志 | `journalctl -k` |
| 磁盘空间 | `df -h` |
| 哪个目录占空间 | `du -sh PATH/*` |

但这张表真正应该这样使用：

```text
不要：

“ss 是干嘛的？”

而是：

“我想知道 8000 有没有程序监听。”
            ↓
       ss -lntp
```

这才是我更需要建立的 Linux 能力。

---

# 三十、最后形成真正可执行的排障 SOP

以后服务器出现：

```text
“访问不了了，而且感觉服务器很卡。”
```

我甚至可以机械地开始：

```bash
uptime
```

↓

```bash
top
```

↓

```bash
free -h
```

↓

```bash
vmstat 1
```

如果内存异常：

```bash
ps aux --sort=-%mem | head
```

找到 PID：

```bash
ps -fp PID
```

同时检查 OOM：

```bash
journalctl -k | grep -i -E 'oom|killed process'
```

如果系统资源正常，转网络：

```bash
ip addr
```

↓

```bash
ip route
```

↓

```bash
ping -c 4 目标IP
```

↓

```bash
nc -vz 目标IP 目标端口
```

如果检查本机服务：

```bash
ss -lntp | grep :端口
```

↓

```bash
ps -fp PID
```

↓

```bash
curl -v http://127.0.0.1:端口
```

↓

```bash
systemctl status 服务名
```

↓

```bash
journalctl -u 服务名 -n 100
```

这样，一个原本完全不知道原因的：

```text
“服务器好像坏了”
```

就会被一步一步压缩：

```text
服务器坏了
 ↓
系统资源正常
 ↓
网络有问题
 ↓
IP 可达
 ↓
8000 不可达
 ↓
服务器没有 LISTEN 8000
 ↓
对应服务已经退出
 ↓
日志发现 OOM
 ↓
Python 进程之前持续增长
 ↓
进入应用内存问题调查
```

到这里，我不需要一开始就知道答案。

我只需要知道：

> **当前这一步，我想证明什么？**

然后找到能够回答这个问题的命令。

Linux 排障也就从：

```text
不会
 ↓
Google / 问 AI
 ↓
复制一条不知道什么意思的命令
```

逐渐变成：

```text
我想验证 X
 ↓
所以执行 Y
 ↓
结果说明 Z
 ↓
因此下一步验证 W
```

这才是真正能迁移到以后 Agent 服务、FastAPI、数据库、Docker、云服务器上的运维能力。