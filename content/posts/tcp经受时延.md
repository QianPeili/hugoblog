---
title: "tcp经受时延"
date: 2017-05-20T16:15:51+08:00
categories: 
 - note
tags: 
 - tcp
---

学 `TCP/IP` 协议听到最多的就是三次握手和四次挥手协议了吧。由于最近在折腾 `tcpdump` ，就想着实战抓一下 `tcp` 协议的这个握手和挥手协议。于是我在本地向内网测试机器的服务上发起了一个
 `POST` 请求，抓到如下信息：

	15:05:03.904647 IP socket-client.56558 > socket-server.3000: Flags [S], seq 2090421259, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
	15:05:03.904699 IP socket-server.3000 > socket-client.56558: Flags [S.], seq 2303886283, ack 2090421260, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
	15:05:03.905263 IP socket-client.56558 > socket-server.3000: Flags [.], ack 1, win 2053, length 0
	15:05:03.905596 IP socket-client.56558 > socket-server.3000: Flags [P.], seq 1:230, ack 1, win 2053, length 229
	15:05:03.905611 IP socket-server.3000 > socket-client.56558: Flags [.], ack 230, win 237, length 0
	15:05:03.906212 IP socket-client.56558 > socket-server.3000: Flags [P.], seq 230:786, ack 1, win 2053, length 556
	15:05:03.906247 IP socket-server.3000 > socket-client.56558: Flags [.], ack 786, win 246, length 0
	15:05:03.912497 IP socket-server.3000 > socket-client.56558: Flags [P.], seq 1:336, ack 786, win 246, length 335
	15:05:03.913375 IP socket-client.56558 > socket-server.3000: Flags [F.], seq 786, ack 336, win 2051, length 0
	15:05:03.913807 IP socket-server.3000 > socket-client.56558: Flags [F.], seq 336, ack 787, win 246, length 0
	15:05:03.914068 IP socket-client.56558 > socket-server.3000: Flags [.], ack 337, win 2051, length 0


三次握手协议很明显，但是说好的四次挥手协议只有三条！。

难道我们抓的是假的`TCP`包？当然不是，在**《TCP/IP详解》**卷一第十九章中，有这么一段话：

> 把从bsdi发送到srv4的7个ACK标记为经受时延的ACK。通常TCP在接收到数据时并不立即发送ACK；相反，它推迟发送，以便将ACK与需要沿该方向发送的数据一起发送（有时称这种现象为数据捎带ACK）。绝大多数实现采用的时延为200 ms，也就是说，TCP将以最大200 ms的时延等待是否有数据一起发送。

这个行为叫做**经受时延**(`Delay ACK`)

> TCP delayed acknowledgment is a technique used by some implementations of the Transmission Control Protocol in an effort to improve network performance. In essence, several ACK responses may be combined together into a single response, reducing protocol overhead. However, in some circumstances, the technique can reduce application performance.

由于 `HTTP` 请求双发在发完消息后，双发几乎是同时关闭连接的，所以倒数第二条消息中，被动关闭方同时把 `FIN` 和 `ACK` 消息一同发送了，就变成了三次挥手。如果我们要看到四次挥手，只要让被动关闭方在主动关闭方发起 `FIN` 消息**>=200ms**之后再关闭连接，应该就可以看到四次挥手了。

所以写了段小代码测试。

服务端 `server.py`：

	import socket
	import time
	
	HOST, PORT = 'socket-server', 3005
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.bind((HOST, PORT))
	s.listen(1)
	conn, addr = s.accept()
	while 1:
	    data = conn.recv(1024)
	    time.sleep(1)
	    break
	conn.close()
	s.close()

客户端 `client.py`：

	import socket
	import time
	
	HOST, PORT = 'socket-server', 3005
	s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	s.connect((HOST, PORT))
	s.sendall("hello")
	s.close()

其中，客户端连接到服务器后，发送一条消息，并立即关闭连接。而服务端收到客户端的消息后，沉睡1秒后再关闭连接。

我再 `socket-server` 机器上跑 `server.py` 脚本，然后用 `tcpdump` 抓取 `3005` 端口的消息：

	sudo tcpdump -ttt -i eno1 tcp and host socket-server and port 3005

`-ttt` 参数可以帮我们显示出两条消息之间的间隔。

然后再我自己的电脑上跑 `client.py` 脚本。抓到如下数据：

	00:00:00.000000 IP socket-client.55683 > socket-server.3005: Flags [S], seq 3277094545, win 64240, options [mss 1460,nop,wscale 8,nop,nop,sackOK], length 0
	 00:00:00.000048 IP socket-server.3005 > socket-client.55683: Flags [S.], seq 450801491, ack 3277094546, win 29200, options [mss 1460,nop,nop,sackOK,nop,wscale 7], length 0
	 00:00:00.000528 IP socket-client.55683 > socket-server.3005: Flags [.], ack 1, win 2053, length 0
	 00:00:00.000042 IP socket-client.55683 > socket-server.3005: Flags [P.], seq 1:6, ack 1, win 2053, length 5
	 00:00:00.000013 IP socket-server.3005 > socket-client.55683: Flags [.], ack 6, win 229, length 0
	 00:00:00.000004 IP socket-client.55683 > socket-server.3005: Flags [F.], seq 6, ack 1, win 2053, length 0
	 00:00:00.036681 IP socket-server.3005 > socket-client.55683: Flags [.], ack 7, win 229, length 0
	 00:00:00.964009 IP socket-server.3005 > socket-client.55683: Flags [F.], seq 1, ack 7, win 229, length 0
	 00:00:00.000541 IP socket-client.55683 > socket-server.3005: Flags [.], ack 2, win 2053, length 0

在上面，我们已经可以看到四次挥手了，实验成功（就是他妈有点水- -）！
