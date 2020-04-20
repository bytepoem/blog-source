---
title: 记录一次服务器出现大量 CLOSE_WAIT 的原因排查过程以及解决办法
date: 2020-04-20 23:28:54
permalink: to-many-close-wait
category:
  - linux
tags:
  - linux
  - tcp
  - python
toc: true
---

最近在服务器上发现有大量的 CLOSE_WAIT 存在，排查发现是 Python 爬虫中 requests 导致的。

### 1. CLOSE WAIT

简单来说，TCP 连接断开时需要进行“四次挥手”（建立连接是“三次握手”），TCP 连接的两端都可以发起关闭连接的请求，若其中一端发起了关闭连接，但另外一端没有关闭连接，那么该连接就会处于 CLOSE_WAIT 状态。

<!-- more -->

![TCP四次挥手](https://i.loli.net/2020/04/20/tevTiJ9cWzSmQdV.png)

四次握手的流程：

1. Client 发送一个 FIN，用来关闭客户端到服务器的数据传送（报文段 1）
2. Server 收到这个 FIN，发回一个 ACK，确认需要为收到的序号加 1（报文段 5）。和 SYN 一样，一个 FIN 占用一个序号
3. Server 关闭 Client 的连接，发送一个 FIN 给客户端（报文段 6）
4. Client 发回 ACK 报文确认，并将确认序号设为收到的序号加 1（报文段 4）

### 2. 原因

通常来说，CLOSE_WAIT 在服务器停留的时间很短，且只会发生在被动关闭连接的一端。除非 Kill 掉进程，否则它是不会消失的，意味着一直占用资源。如果发现有大量的 CLOSE_WAIT，那就是被动关闭的一方没有及时发送 FIN，一般来说有以下几种可能：

- 代码问题：请求的时候没有显式关闭 Socket 连接，或者死循环导致关闭连接的代码没有执行到，即 FIN 包没有发出，导致 CLOSE_WAIT 不断累积
- 响应过慢 / 超时设置过小：双方连接不稳定，一方 Timeout，另外一方还在处理逻辑，导致 Close 被延后

### 3. 排查

#### 查看网络连接

首先我们到服务器上看一下网络连接情况。

查看 TCP 连接中各个状态数量可以使用以下命令：

- `netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'`
  - ESTABLISHED 表示正在通信
  - TIME_WAIT 表示主动关闭
  - CLOSE_WAIT 表示被动关闭

![netstat1](https://i.loli.net/2020/04/20/s4St7ZGcPlpmy5E.png)

查看 TCP 连接中 CLOSE_WAIT 的数量可以使用以下命令：

- `netstat -antop | grep CLOSE_WAIT | wc -l`
  - netstat 命令用于显示网络状态
    - -a：显示所有连线中的 Socket
    - -n：直接使用 IP 地址，而不通过域名服务器
    - -t：显示 TCP 传输协议的连线状况
    - -o：显示计时器
    - -p：显示正在使用 Socket 的程序识别码和程序名称
  - grep 命令用于查找文件里符合条件的字符串
  - wc 命令用于计算字数
    - -l：只显示行数

![netstat2](https://i.loli.net/2020/04/20/iftzA3VnX9DyOwx.png)

通过以上命令可以看到服务器上的 CLOSE_WAIT 将近 1w2，这还是我重启了部分运行时间较长的爬虫后的数字，重启前高达 4w，这是个不容小觑的潜在问题。

#### 梳理 TCP 连接流向

接着我们可以从上图梳理出 CLOSE_WAIT 的连接流向，命令返回里中的 Foreign Address（第 5 栏）代表对方的 IP 地址，即和我们连接着但是却主动关闭了连接的机器，在我这里即是代理池里边的代理。

#### 根据项目数据请求流向还原可能场景

然后我们可以根据项目数据请求流向，还原出可能的场景，在我这里即是 CLOSE_WAIT 都发生在本机爬虫、代理以及目标网站的连接上。毕竟 Program name（最后一栏）都写着 Python 了，且这台服务器上只有爬虫用的 Python。

#### 解决思路

问题出在爬虫就简单了，唯一会发生网络请求的地方就是使用 requests 模块。

- requests 模块会自行处理连接池的问题，且访问完成后出现 CLOSE_WAIT 是正常的，后续会直接复用这些连接。但是如果 CLOSE_WAIT 数量太多且一直下不去，有可能是高并发或者没有关闭的连接一直累积
- 如果你直接 get 或者 post 的话会每次都创建一个连接池，且这个连接池只会用一次。所以需要通过全局 session 来使用 requests 模块的连接池功能
- 不是任何时候使用连接池都是有效的，仅当你使用了多线程之类的并发才会有效果。毕竟你是单线程的话，里面是串行，一个线程对应一个连接。此外，我们还可以构造好 HttpAdapter 实例用 session.mount 上。HttpAdapter 的常用参数有以下几种：
  - timeout：超时时间
  - maxretries：最大尝试次数
  - pool_connections：最多连接多少个不同的主机，针对每个 Host 都是独立的连接池
  - pool_maxsize：针对每个主机能创建的最大连接数（底层 TCP），由于一个线程对应一个连接，所以一般有多少个线程指定 pool_maxsize 为多少
- 使用 requests 模块的时候最好将请求放在 try 中，把可能发生的异常用 except 获取并处理

---

参考链接：

[浅谈 CLOSE_WAIT](http://blog.huoding.com/2016/01/19/488)

[又见 CLOSE_WAIT](https://mp.weixin.qq.com/s?__biz=MzI4MjA4ODU0Ng==&mid=402163560&idx=1&sn=5269044286ce1d142cca1b5fed3efab1&3rd=MzA3MDU4NTYzMw==&scene=6#rd)

[【原创】python requests 库底层 Sockets 处于 close_wait 状态](https://www.cnblogs.com/pengyusong/p/5805704.html)
