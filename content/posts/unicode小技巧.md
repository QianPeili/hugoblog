---
title: "unicode小技巧"
date: 2017-04-07T10:20:30+08:00
categories: 
 - tips
tags: 
 - linux
 - tcpdump
---

今天遇到一个问题

在mysql查询日志时，发现一些unicode字符串存储变成了

	u7528u6237u53d6u6d88u8ba2u5355u8bf7u6c42

这个形式，手动加`\`虽然可行，但毕竟不是长久的办法

所以最好的办法是将字符串中的内容，根据unicode格式转换成unicode

代码如下:

	import re	
	a = "u7528u6237u53d6u6d88u8ba2u5355u8bf7u6c42"	
	pattern = re.compile("\u(.{4})")	
	maps = pattern.findall(a)	
	s = ""	
	for i in maps:
		tmp = unichr(int(i, 16))
		s += tmp
	print s

就会输出原先代表的文字意思了

注： 以上是代码是python2版本， python3需要把`unichr`改成`chr`

