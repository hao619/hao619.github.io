---
layout: post
title:  "学习tcpdump常用操作，这一篇就够了"
date:   2018-03-27 14:41:50
categories: 网络基础
tags: tcpdump
---


# 学习tcpdump常用操作，这一篇就够了

## tcpdump命令参数
```
tcpdump采用命令行方式，它的命令格式为：

　　tcpdump [ -adeflnNOpqStvx ] [ -c 数量 ] [ -F 文件名 ]

　　　　　　　　　　[ -i 网络接口 ] [ -r 文件名] [ -s snaplen ]

                                       [ -T 类型 ] [ -w 文件名 ] [表达式 ]

 tcpdump的选项介绍

　　　-a 　　　将网络地址和广播地址转变成名字；

　　　-d 　　　将匹配信息包的代码以人们能够理解的汇编格式给出；

　　　-dd 　　　将匹配信息包的代码以c语言程序段的格式给出；

　　　-ddd 　　　将匹配信息包的代码以十进制的形式给出；

　　　-e 　　　在输出行打印出数据链路层的头部信息，包括源mac和目的mac，以及网络层的协议；

　　　-f 　　　将外部的Internet地址以数字的形式打印出来；

　　　-l 　　　使标准输出变为缓冲行形式；

　　　-n 　　　指定将每个监听到数据包中的域名转换成IP地址后显示，不把网络地址转换成名字；

     -nn：    指定将每个监听到的数据包中的域名转换成IP、端口从应用名称转换成端口号后显示

　　　-t 　　　在输出的每一行不打印时间戳；

　　　-v 　　　输出一个稍微详细的信息，例如在ip包中可以包括ttl和服务类型的信息；

　　　-vv 　　　输出详细的报文信息；

　　　-c 　　　在收到指定的包的数目后，tcpdump就会停止；

　　　-F 　　　从指定的文件中读取表达式,忽略其它的表达式；

　　　-i 　　　指定监听的网络接口；

     -p：    将网卡设置为非混杂模式，不能与host或broadcast一起使用

　　　-r 　　　从指定的文件中读取包(这些包一般通过-w选项产生)；

　　　-w 　　　直接将包写入文件中，并不分析和打印出来；

            -s snaplen         snaplen表示从一个包中截取的字节数。0表示包不截断，抓完整的数据包。默认的话 tcpdump 只显示部分数据包,默认68字节。

　　　-T 　　　将监听到的包直接解释为指定的类型的报文，常见的类型有rpc （远程过程调用）和snmp（简单网络管理协议；）

     -X            告诉tcpdump命令，需要把协议头和包内容都原原本本的显示出来（tcpdump会以16进制和ASCII的形式显示），这在进行协议分析时是绝对的利器。
```


## TCP相关
### 输出信息含义
基本上tcpdump总的的输出格式为：系统时间 来源主机.端口 > 目标主机.端口 数据包参数

### TCP三次握手(创建 OPEN)
客户端发起一个和服务创建TCP链接的请求，这里是SYN(J)
服务端接受到客户端的创建请求后，返回两个信息： SYN(K) + ACK(J+1)
客户端在接受到服务端的ACK信息校验成功后(J与J+1)，返回一个信息：ACK(K+1)
服务端这时接受到客户端的ACK信息校验成功后(K与K+1)，不再返回信息，后面进入数据通讯阶段
数据通讯
客户端/服务端 read/write数据包
### TCP四次挥手(关闭 finish)
客户端发起关闭请求，发送一个信息：FIN(M)
服务端接受到信息后，首先返回ACK(M+1),表明自己已经收到消息。
服务端在准备好关闭之前，最后发送给客户端一个 FIN(N)消息，询问客户端是否准备好关闭了
客户端接受到服务端发送的消息后，返回一个确认信息: ACK(N+1)
最后，服务端和客户端在双方都得到确认时，各自关闭或者回收对应的TCP链接。

### TCP各种状态
1. SYN_SEND
客户端尝试链接服务端，通过open方法。也就是TCP三次握手中的第1步之后,注意是客户端状态
sysctl -w net.ipv4.tcp_syn_retries = 2 ,做为客户端可以设置SYN包的重试次数，默认5次(大约180s)引用校长的话：仅仅重试2次，现代网络够了
2. SYN_RECEIVED
服务接受创建请求的SYN后，也就是TCP三次握手中的第2步，发送ACK数据包之前
注意是服务端状态,一般15个左右正常，如果很大，怀疑遭受SYN_FLOOD攻击
sysctl -w net.ipv4.tcp_max_syn_backlog=4096 , 设置该状态的等待队列数，默认1024，调大后可适当防止syn-flood，可参见man 7 tcp
sysctl -w net.ipv4.tcp_syncookies=1 ,　打开syncookie，在syn backlog队列不足的时候，提供一种机制临时将syn链接换出
sysctl -w net.ipv4.tcp_synack_retries = 2 ,做为服务端返回ACK包的重试次数，默认5次(大约180s)引用校长的话：仅仅重试2次，现代网络够了
3. ESTABLISHED
客户端接受到服务端的ACK包后的状态，服务端在发出ACK在一定时间后即为ESTABLISHED
sysctl -w net.ipv4.tcp_keepalive_time = 1200 ，默认为7200秒(2小时)，系统针对空闲链接会进行心跳检查，如果超过net.ipv4.tcp_keepalive_probes * net.ipv4.tcp_keepalive_intvl = 默认11分，终止对应的tcp链接，可适当调整心跳检查频率
目前线上的监控 waring:600 , critial : 800
4. FIN_WAIT1
主动关闭的一方，在发出FIN请求之后，也就是在TCP四次握手的第1步
5. CLOSE_WAIT
被动关闭的一方，在接受到客户端的FIN后，也就是在TCP四次握手的第2步
6. FIN_WAIT2
主动关闭的一方，在接受到被动关闭一方的ACK后，也就是TCP四次握手的第2步
sysctl -w net.ipv4.tcp_fin_timeout=30, 可以设定被动关闭方返回FIN后的超时时间，有效回收链接，避免syn-flood.
7. LASK_ACK
被动关闭的一方，在发送ACK后一段时间后(确保客户端已收到)，再发起一个FIN请求。也就是TCP四次握手的第3步
8. TIME_WAIT
主动关闭的一方，在收到被动关闭的FIN包后，发送ACK。也就是TCP四次握手的第4步
sysctl -w net.ipv4.tcp_tw_recycle = 1 , 打开快速回收TIME_WAIT，Enabling this option is not recommended since this causes problems when working with NAT (Network Address Translation)
sysctl -w net.ipv4.tcp_tw_reuse =1, 快速回收并重用TIME_WAIT的链接, 貌似和tw_recycle有冲突，不能重用就回收?
net.ipv4.tcp_max_tw_buckets: 处于time_wait状态的最多链接数，默认为180000.

> 1. 主动关闭方在接收到被动关闭方的FIN请求后，发送成功给对方一个ACK后,将自己的状态由FIN_WAIT2修改为TIME_WAIT，而必须 再等2倍的MSL(Maximum Segment Lifetime,MSL是一个数据报在internetwork中能存在的时间)时间之后双方才能把状态 都改为CLOSED以关闭连接。目前RHEL里保持TIME_WAIT状态的时间为60秒
2. keepAlive策略可以有效的避免进行三次握手和四次关闭的动作

### TCP 数据包
通常tcpdump对tcp数据包的显示格式如下:
src > dst: flags data-seqno ack window urgent options

```
10:45:21.788156 IP 39.107.156.5.80 > 59.153.73.10.14900: Flags [S.], seq 3291927959, ack 2790204654, win 8192, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
10:45:21.788168 IP 59.153.73.10.14900 > 39.107.156.5.80: Flags [.], ack 1, win 58, length 0
10:45:21.790567 IP 59.153.73.10.14900 > 39.107.156.5.80: Flags [P.], seq 1:254, ack 1, win 58, length 253
10:45:21.810599 IP 39.107.156.5.80 > 59.153.73.10.14900: Flags [.], seq 1:1461, ack 254, win 513, length 1460
10:45:21.810610 IP 59.153.73.10.14900 > 39.107.156.5.80: Flags [.], ack 1461, win 63, length 0
10:45:21.810613 IP 39.107.156.5.80 > 59.153.73.10.14900: Flags [P.], seq 1461:1556, ack 254, win 513, length 95
10:45:21.810619 IP 59.153.73.10.14900 > 39.107.156.5.80: Flags [.], ack 1556, win 63, length 0
10:45:21.810622 IP 39.107.156.5.80 > 59.153.73.10.14900: Flags [F.], seq 1556, ack 254, win 513, length 0
10:45:21.810828 IP 59.153.73.10.14900 > 39.107.156.5.80: Flags [F.], seq 254, ack 1557, win 63, length 0
10:45:21.830608 IP 39.107.156.5.80 > 59.153.73.10.14900: Flags [.], ack 255, win 513, length 0
```

src 和 dst 是源和目的IP地址以及相应的端口. flags 标志由S(SYN), F(FIN), P(PUSH, R(RST),

W(ECN CWT(nt | rep:未知, 需补充))或者 E(ECN-Echo(nt | rep:未知,　需补充))组成,

单独一个'.'表示没有flags标识. 数据段顺序号(Data-seqno)描述了此包中数据所对应序列号空间中的一个位置(nt:整个数据被分段,

每段有一个顺序号, 所有的顺序号构成一个序列号空间)(可参考以下例子). Ack 描述的是同一个连接,同一个方向,下一个本端应该接收的

(对方应该发送的)数据片段的顺序号. Window是本端可用的数据接收缓冲区的大小(也是对方发送数据时需根据这个大小来组织数据).

Urg(urgent) 表示数据包中有紧急的数据. options 描述了tcp的一些选项, 这些选项都用尖括号来表示(如 <mss 1024>).

### HTTP包头位置

1. tcp[20:2]
>	可选值:
	0x4745 为"GET"前两个字母"GE"
	0x4854 为"HTTP"前两个字母"HT"
	0x504f PO POST
	0x4845 HE HEAD
	0x5055 PU PUT
	0x4445 DE DELETE
	0x4f50 OP OPTIONS
	0x5452 TR TRACE
	0x434f CO CONNECT

2. tcp[0:2] 源端口
3. tcp[2:2] 目的端口
4. tcp[13] tcp标记。
	可选值 syn:2  psh-ack:24 fin-ack:& 1 =1 
	rst:& 4 =4   syn-ack:& 2 =2   
5. 

### tcpdump常用字段偏移名称
1. icmptype (ICMP 类型字段)
2. icmpcode (ICMP 符号字段)
3. tcpflags (TCP 标记字段)
* ICMP 类型值有：
icmp-echoreply, icmp-unreach, icmp-sourcequench, icmp-redirect, icmp-echo,
icmp-routeradvert, icmp-routersolicit, icmp-timxceed, icmp-paramprob, icmp-tstamp,
icmp-tstampreply, icmp-ireq, icmp-ireqreply, icmp-maskreq, icmp-maskreply
* TCP 标记值：
tcp-fin, tcp-syn, tcp-rst, tcp-push, tcp-push, tcp-ack, tcp-urg

### 常用表达式
1. 关于类型的关键字，主要包括host，net，port
2. 传输方向的关键字，主要包括src , dst ,dst or src, dst and src
3. 协议的关键字，主要包括fddi,ip ,arp,rarp,tcp,udp等类型
4. 逻辑运算，取非运算是 'not ' '! ', 与运算是'and','&&';或运算 是'or' ,'||'
5. 其他重要的关键字如下：gateway, broadcast,less,greater

### 过滤数据包
libpcap利用BPF来过滤数据包。
过滤数据包需要完成3件事：

构造一个过滤表达式
编译这个表达式
应用这个过滤器
#### Lipcap已经把BPF语言封装成为了更高级更容易的语法了

```
src host 127.0.0.1
//选择只接受某个IP地址的数据包

dst port 8000
//选择只接受TCP/UDP的目的端口是80的数据包
 
not tcp
//不接受TCP数据包
 
tcp[13]==0x02 and (dst port ** or dst port **)
//只接受SYN标志位置（TCP首部开始的第13个字节）且目标端口号是22或23的数据包
 
icmp[icmptype]==icmp-echoreply or icmp[icmptype]==icmp-echo
//只接受icmp的ping请求和ping响应的数据包
 
ehter dst 00:00:00:00:00:00
//只接受以太网MAC地址为00：00：00：00：00：00的数据包
 
ip[8]==5
//只接受ip的ttl=5的数据包（ip首位第八的字节为ttl）
``` 
### 命令样例
* tcpdump  tcp[20:2]=0x4745 or tcp[20:2]=0x4854 and port 8000  -XvvennASs 0 -w abc.cap


	> 参数详解：
       -s : 默认只显示数据包的68字节，-s 0 表示不截断。
       -X: 告诉tcpdump命令，需要把协议头和包内容都原原本本的显示出来（tcpdump会以16进制和ASCII的形式显示），这在进行协议分析时是绝对的利器。
       -c: 收到指定数目的包后，tcpdump 就会停止
       -vv: 显示报文详细信息（从数据段没看出差异）
       -w: 将结果输出到指定文件中
 通过过滤表达式，只筛选出有价值的数据（类型、方向、协议）

* 仅抓取 GET 请求
tcpdump -i eth1  'tcp[(tcp[12]>>2):4] = 0x47455420 and port 8000' -s 0

* 仅抓取 POST 请求
tcpdump -i eth1  'tcp[(tcp[12]>>2):4] = 0x504F5354 and port 8000'  -vvs 0
> 注：
>>1. 0x47455420对应的 ascii 码为 GET， 0x504F5354对应 POST， 0x48545450对应 HTTP
>>2. tcp[12]表示 tcp 报文头的第13字节，也即4位首部长度+4位保留位。
     首部长度转换为字节的话：len*32/8 == len*4, len=tcp[12]>>4, 所以数据项的起始字节为 tcp[12]>>4再<<2，为 tcp[12]>>2 
>>3. http 是应用层协议，从TCP 报文的数据段开始


* http数据包抓取 (直接在终端输出package data)
> tcpdump tcp port 80 -n -X -s 0 指定80端口进行输出

* 抓取http包数据指定文件进行输出package
> tcpdump tcp port 80 -n -s 0 -w /tmp/tcp.cap

对应的/tmp/tcp.cap基本靠肉眼已经能看一下信息，比如http Header , content信息等

* 结合管道流
> tcpdump tcp port 80 -n -s 0 -X -l | grep xxxx
这样可以实时对数据包进行字符串匹配过滤

* mod_proxy反向代理抓包
>> 线上服务器apache+jetty，通过apache mod_proxy进行一个反向代理，80 apache端口, 7001 jetty端口
>> apache端口数据抓包：　tcpdump tcp port 80 -n -s 0 -X -i eth0 　　注意：指定eth0网络接口
>> jetty端口数据抓包：　tcpdump tcp port 7001 -n -s 0 -X -i lo 注意：指定Loopback网络接口

* 只监控特定的ip主机
>tcpdump tcp host 10.16.2.85 and port 2100 -s 0 -X　
>需要使用tcp表达式的组合，这里是host指示只监听该ip

## IP协议相关

### IP包头位置
1. ip[9] 协议版本 1,icmp 6,tcp 17,udp 2,igmp
2. ip[8] ttl
3. ip[12:4] src ip
4. ip[16:4] dst ip
5. ip[2:2] 总长度

## 引用一些资料

1. [tcpdump参数解析及使用详解](https://blog.csdn.net/hzhsan/article/details/43445787)
2. [ascii 码](http://ascii.911cha.com/)  
3. [tcpdump非常实用的抓包实例](https://blog.csdn.net/nanyun2010/article/details/23445223/)
4. [tcpdump抓取HTTP包](https://blog.csdn.net/kofandlizi/article/details/8106841)
5. [tcpdump高级过滤表达式](https://blog.csdn.net/lepton126/article/details/8162926)
6. [TCP报文格式](https://blog.csdn.net/a19881029/article/details/29557837)
7. [IP报文格式详解](https://blog.csdn.net/mary19920410/article/details/59035804)