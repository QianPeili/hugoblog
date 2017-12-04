---
title: "tcpdump的使用实例"
date: 2017-05-20T13:46:45+08:00
categories: 
 - note
tags: 
 - linux
 - tcpdump
---

## 介绍

`tcpdump` 是 Linux 系统下进行网络分析的重要工具，我也因为对抓包这一块有些好奇，就找了些资料学习，然后整理成这篇文章。


## 使用说明

命令模式：

    tcpdump [ -AbdDefhHIJKlLnNOpqStuUvxX# ] [ -B buffer_size ]     
	     [ -c count ] 	    
	     [ -C file_size ] [ -G rotate_seconds ] [ -F file ] 	    
	     [ -i interface ] [ -j tstamp_type ] [ -m module ] [ -M secret ]
	     [ --number ] [ -Q in|out|inout ] 
	     [ -r file ] [ -V file ] [ -s snaplen ] [ -T type ] [ -w file ] 
	     [ -W filecount ] 
	     [ -E spi@ipaddr algo:secret,... ] 
	     [ -y datalinktype ] [ -z postrotate-command ] [ -Z user ] 
	     [ --time-stamp-precision=tstamp_precision ] 
	     [ --immediate-mode ] [ --version ] 
	     [ expression ] 
 

### 参数说明

`tcpdump` 的可选参数很多，我这里只列举几个常用的，更详细的欢迎点击 **[Manpage of TCPDUMP](http://www.tcpdump.org/tcpdump_man.html)** 学习。

- `-i`: 监听指定的网络接口，比如我的测试机器是：`-i eno1`
- `-n`: 不解析主机名，用纯 `ip` 显示。
- `-t`: 每一行数据前面不显示时间戳。
- `-c`: 指定抓取的条数，流量大的时候适用。
- `-w`: 将结果输出到指定文件中。
- `-r`: 从指定文件中读取数据信息。

### 条件语句

-   **且**: 用 `and` 或者 `&&` 表示；
-  **或**: 用 `or` 或者 `||` 表示；
-  **非**: 用 `not` 或者 `!` 表示。

多个条件语句连接示例（引号可加可不加）： 
    
	sudo tcpdump -i eno1 'tcp and port 3000 or port 3001'

## Example

### 指定主机或端口

指定抓取的 `ip` 地址

	sudo tcpdump -i eno1 host 192.168.1.2

指定源 `ip` 和目的 `ip`

	sudo tcpdump -i eno1 src 192.168.1.2
 	sudo tcpdump -i eno1 dst 192.168.1.3

指定端口，以下两条协议是一样的作用
	
	sudo tcpdump -i eno1 port 21
	sudo tcpdump -i eno1 port ftp

指定源端口和目的端口

	sudo tcpdump -i eno1 src port 3000 
	sudo tcpdump -i eno1 dst port 5000

同时指定协议、 `ip` 和端口

	sudo tcpdump -i eno1 'tcp and host 192.168.1.2 and port 3000'

### 抓取 HTTP 协议

> 待补充：这里我还不知道如何显示http协议发生的时间

抓取 `POST` 请求：

    sudo tcpdump -n -A -i eno1 | grep -e 'POST'

得到输出为：

    . '.L..?......MP...E...POST /daily_quests/finish_quest HTTP/1.1
    . '.M...`./....P.......POST /getachievementaward HTTP/1.1
    . '.N...."..=;xP.......POST /buylottery/1.9.0 HTTP/1.1
    . '.O....8Q...AP...M...POST /daily_quests/finish_quest HTTP/1.1
    . '.P.."x+m&...P...y...POST /getDailyQuestAward HTTP/1.1

其它 `methods` 类似， 不一一列举。

## 参考资料

- [Manpage of TCPDUMP](http://www.tcpdump.org/tcpdump_man.html)
- [tcpdump - Linux Wiki](http://linuxwiki.github.io/NetTools/tcpdump.html)
- [Tcpdump tutorial – Sniffing and analysing packets from the commandline](http://www.binarytides.com/tcpdump-tutorial-sniffing-analysing-packets/)


