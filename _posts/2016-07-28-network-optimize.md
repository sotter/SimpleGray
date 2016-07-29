---
layout: post
title: 高性能网络服务器-协议层优化
category: default
---

## 0. TCP连接过程

> 主动建立方：CLOSED->SYN_SEND->ESTABLISHED

> 被动建立方：CLOSED->LISTEN->SYN_RECV->ESTABLISHED

设置监听状态队列的最大值： 高性能的服务端可能用到的，net.core.somaxconn = 100000


## 1. TCP连接关闭过程 - 主动关闭方

> 状态迁移图：ESTABLISHED->FIN_WAIT_1->FIN_WAIT_2->TIME_WAIT->CLOSED

- **为什么存在TIME_WAIT状态，还要等2MSL？**

作为主动关闭方，收到对方的FIN后，自己回复ACK，按道理来讲整个过程就结束了。存在一个异常情况：如果最后这个ACK丢失了，那么FIN发送方会认为FIN可能没有收到，会重传这个FIN，此时如果已经CLOSED，那么就不会处理这个FIN了。
总而言之：是为了最后一个ACK，对方能够收到；

- **为什么超时时间是2MSL?**

MSL指的是报文段的最大生存时间，如果报文段在网络活动了MSL时间，还没有被接收，那么会被丢弃，（RFC建议是2分钟，Linux一般为30s）。

等待2MSL后，这个时间里对方即便没有收到ACK，那么也应该收到对方的FIN了，如果没有收到对方的FIN，那么就默认对方收到了这个ACK。（当然，如果对方的这个FIN也丢失了，也就没有办法了，应该会走TCP的超时机制）

- **如何解决TIME_WAIT过多的问题？**

方法1： 在连接的主动发起方设置tcp_tw_reuse，那么TIME_WAIT socket 1fd占用的端口号，可以被重复利用；但是从上面的分析可以知道，如果这个被复用的socket上，收到上一个连接一个重传的FIN，如何解决这个问题？

方法2： 设置tcp_tw_recycle， time_wait状态设置快速回收

注意这个地方是有坑的：
[tcp_tw_recycle和tcp_timestamps导致connect失败问题 ](http://blog.sina.com.cn/s/blog_781b0c850100znjd.html)

看下面的总结一下：[原文](http://blog.csdn.net/yunhua_lee/article/details/8146845)

1）快速回收到底有多快？     
	局域网环境下，700ms就回收；     
2）有的资料说只要打开tcp_tw_recycle即可，有的又说要tcp_timestamps同时打开，具体是哪个正确？     
	需要同时打开，但默认情况下tcp_timestamps就是打开的，所以会有人说只要打开tcp_tw_recycle即可；     
3）为什么从虚拟机发起客户端连接时选项无效，非虚拟机连接就有效？     
	和网络组网有关系，无法获取对端信息时就不进行快速回收；    

方法3： 设置tcp_max_tw_buckets参数，TIME_WAIT的总数，可以防止一些简单的DDos攻击；

## 2. TCP连接关闭过程 - 被动关闭方

>状态迁移图： ESTABLISHED->CLOSE_WAIT->LAST_ACK->CLOSED

- **为什么关闭方采用的四次握手而不是三次握手**

主动关闭方A给被动关闭方B发送了一个FIN，然后B给A回复了一个ACK；这表示的含义是，A告诉B我已经没有数据再给你了，我把本端的写端给关闭了；
这个时候B中可能还有没有发送完的数据给A，所以不能里面就回给A一个FIN，需要把自己要发给A的数据发完，所以FIN和ACK是分开发的，不能合并发送；

- **对于应用层的程序来讲，先发送一大堆数据，然后不确定这些数据是否发送给对方的情况下，在发送一个FIN，是否所有的数据都已经传递给对方了？**

在不设置SO_LINGER的情况下，是讲所有的数据写端数据全部发送完，再给远端发送Fin, 所以不存在上述问题；

[参见这里](http://unliminet.blog.51cto.com/380895/346686)

## 3. TCP数据传输

**TCP快速重传**
TODO： 关于TCP的快速重传和快速回复，后续专题研究一下

这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段);
对于广域网的服务端通信来说， 这个选项应该启用，但是这会增加对 CPU 的占用。

问题： 对于客户端的APP来说，服务端给客户端传送的数据比较多， 开启此功能是否导致耗电量增加？

**TCP包活定时器 keepAlived**

> net.ipv4.tcp_keepalive_time = 1800 #表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为30分钟。
> net.ipv4.tcp_keepalive_probes = 3  #如果对方不予应答，探测包的发送次数
> net.ipv4.tcp_keepalive_intvl = 15  #eepalive探测包的发送间隔

问题：对于keepalive的探测包，服务端是否可以在EPOLL触发读事件？

18:08:06.789076 IP 127.0.0.1.41734 > 127.0.0.1.5000: Flags [.], ack 1, win 65495, options [nop,nop,TS val 1435097775 ecr 1435087775], length 0
	0x0000:  4500 0034 fb20 4000 4006 41a1 7f00 0001  E..4..@.@.A.....
	0x0010:  7f00 0001 a306 1388 9789 af18 e491 0dd0  ................
	0x0020:  8010 ffd7 4fee 0000 0101 080a 5589 daaf  ....O.......U...
	0x0030:  5589 b39f                                U...
18:08:06.789089 IP 127.0.0.1.5000 > 127.0.0.1.41734: Flags [.], ack 1, win 65483, options [nop,nop,TS val 1435097776 ecr 1434997774], length 0
	0x0000:  4500 0034 59e7 4000 4006 e2da 7f00 0001  E..4Y.@.@.......
	0x0010:  7f00 0001 1388 a306 e491 0dd0 9789 af19  ................
	0x0020:  8010 ffcb af8a 0000 0101 080a 5589 dab0  ............U...
	0x0030:  5588 540e

结论：不会触发Epoll的读事件，当然也不会有数据了；

client端不发keepalived时，
[从keep-alive原理 分析TCP游戏服务端心跳包的实用功能](http://www.cocoachina.com/cms/wap.php?action=article&id=13501)

**TCP的发送缓冲区**

> $ cat /proc/sys/net/ipv4/tcp_rmem
> 4096	87380	4194304
> $ cat /proc/sys/net/ipv4/tcp_wmem
> 4096	16384	4194304

	第一个值是为 socket 的发送缓冲区分配的最少字节数。
	第二个值是默认值（该值会被 wmem_default 覆盖），缓冲区在系统负载不重的情况下可以增长到这个值。
	第三个值是发送缓冲区空间的最大字节数（该值会被 wmem_max 覆盖）。

**TCP套接字占用的内存**
net.ipv4.tcp_wmem = 30000000 30000000 30000000

在超高并发的环境需要对这个参数进行调整。
> $ cat /proc/sys/net/ipv4/tcp_mem
> 6179328	8239104	12358656

上面3个值分别对应了low, pressure, high的状态，注意：单位是内存页，默认值是4096字节，可通过getpagesize函数查询当前系统的内存大小；

|名称|大小|含义|
|---|---|---|
|low| 23G  |当TCP使用了低于该值的内存页面数时，TCP不会考虑释放内存;|
|pressure|31G|当TCP使用了超过该值的内存页面数量时，TCP试图稳定其内存使用，进入pressure模式，当内存消耗低于low值时则退出pressure状态;|
|high| 47G | 允许所有tcp sockets用于排队缓冲数据报的页面量，当内存占用超过此值，系统拒绝分配socket，后台日志输出“TCP: too many of orphaned sockets”;|

## 4. TCP调优

```
net.core.netdev_max_backlog = 400000
#该参数决定了，网络设备接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

net.core.optmem_max = 10000000
#该参数指定了每个套接字所允许的最大缓冲区的大小

net.core.rmem_default = 10000000
#指定了接收套接字缓冲区大小的缺省值（以字节为单位）。

net.core.rmem_max = 10000000
#指定了接收套接字缓冲区大小的最大值（以字节为单位）。

net.core.somaxconn = 100000
#Linux kernel参数，表示socket监听的backlog(监听队列)上限

net.core.wmem_default = 11059200
#定义默认的发送窗口大小；对于更大的 BDP 来说，这个大小也应该更大。

net.core.wmem_max = 11059200
#定义发送窗口的最大大小；对于更大的 BDP 来说，这个大小也应该更大。

net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
#严谨模式 1 (推荐)
#松散模式 0

net.ipv4.tcp_congestion_control = bic
#默认推荐设置是 htcp

net.ipv4.tcp_window_scaling = 0
#关闭tcp_window_scaling
#启用 RFC 1323 定义的 window scaling；要支持超过 64KB 的窗口，必须启用该值。

net.ipv4.tcp_ecn = 0
#把TCP的直接拥塞通告(tcp_ecn)关掉

net.ipv4.tcp_sack = 1
#关闭tcp_sack
#启用有选择的应答（Selective Acknowledgment），
#这可以通过有选择地应答乱序接收到的报文来提高性能（这样可以让发送者只发送丢失的报文段）；
#（对于广域网通信来说）这个选项应该启用，但是这会增加对 CPU 的占用。

net.ipv4.tcp_max_tw_buckets = 10000
#表示系统同时保持TIME_WAIT套接字的最大数量

net.ipv4.tcp_max_syn_backlog = 8192
#表示SYN队列长度，默认1024，改成8192，可以容纳更多等待连接的网络连接数。

net.ipv4.tcp_syncookies = 1
#表示开启SYN Cookies。当出现SYN等待队列溢出时，启用cookies来处理，可防范少量SYN攻击，默认为0，表示关闭；

net.ipv4.tcp_timestamps = 1
#开启TCP时间戳
#以一种比重发超时更精确的方法（请参阅 RFC 1323）来启用对 RTT 的计算；为了实现更好的性能应该启用这个选项。

net.ipv4.tcp_tw_reuse = 1
#表示开启重用。允许将TIME-WAIT sockets重新用于新的TCP连接，默认为0，表示关闭；

net.ipv4.tcp_tw_recycle = 1
#表示开启TCP连接中TIME-WAIT sockets的快速回收，默认为0，表示关闭。

net.ipv4.tcp_fin_timeout = 10
#表示如果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2状态的时间。

net.ipv4.tcp_keepalive_time = 1800
#表示当keepalive起用的时候，TCP发送keepalive消息的频度。缺省是2小时，改为30分钟。

net.ipv4.tcp_keepalive_probes = 3
#如果对方不予应答，探测包的发送次数

net.ipv4.tcp_keepalive_intvl = 15
#keepalive探测包的发送间隔

net.ipv4.tcp_mem
#确定 TCP 栈应该如何反映内存使用；每个值的单位都是内存页（通常是 4KB）。
#第一个值是内存使用的下限。
#第二个值是内存压力模式开始对缓冲区使用应用压力的上限。
#第三个值是内存上限。在这个层次上可以将报文丢弃，从而减少对内存的使用。对于较大的 BDP 可以增大这些值（但是要记住，其单位是内存页，而不是字节）。

net.ipv4.tcp_rmem
#与 tcp_wmem 类似，不过它表示的是为自动调优所使用的接收缓冲区的值。

net.ipv4.tcp_wmem = 30000000 30000000 30000000
#为自动调优定义每个 socket 使用的内存。
#第一个值是为 socket 的发送缓冲区分配的最少字节数。
#第二个值是默认值（该值会被 wmem_default 覆盖），缓冲区在系统负载不重的情况下可以增长到这个值。
#第三个值是发送缓冲区空间的最大字节数（该值会被 wmem_max 覆盖）。

net.ipv4.ip_local_port_range = 1024 65000
#表示用于向外连接的端口范围。缺省情况下很小：32768到61000，改为1024到65000。

net.ipv4.netfilter.ip_conntrack_max=204800
#设置系统对最大跟踪的TCP连接数的限制

net.ipv4.tcp_slow_start_after_idle = 0
#关闭tcp的连接传输的慢启动，即先休止一段时间，再初始化拥塞窗口。

net.ipv4.route.gc_timeout = 100
#路由缓存刷新频率，当一个路由失败后多长时间跳到另一个路由，默认是300。

net.ipv4.tcp_syn_retries = 1
#在内核放弃建立连接之前发送SYN包的数量。

net.ipv4.icmp_echo_ignore_broadcasts = 1
# 避免放大攻击

net.ipv4.icmp_ignore_bogus_error_responses = 1
# 开启恶意icmp错误消息保护

net.inet.udp.checksum=1
#防止不正确的udp包的攻击

net.ipv4.conf.default.accept_source_route = 0
#是否接受含有源路由信息的ip包。参数值为布尔值，1表示接受，0表示不接受。
#在充当网关的linux主机上缺省值为1，在一般的linux主机上缺省值为0。
#从安全性角度出发，建议你关闭该功能。
```
- 文件系统调优
<!-- 4. 数据库调优
数据库锁的方式：
数据库的存储机制：
数据库的分布式策略： -->