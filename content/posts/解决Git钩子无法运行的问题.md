---
title: "解决 Git Hooks 无法运行的问题"
date: 2017-07-15T14:46:17+08:00
categories: 
 - tips
tags: 
 - git
---

这两天在研究 [Git Hooks](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks) 的过程中，遇到一个问题。

在触发钩子时，终端一直输出：

	remote: error: cannot run hooks/post-receive: No such file or directory

其实这不是文件不存在的问题，反而是文件存在，但是有问题才会出现这个提示。

经过网上查找，主要有两个原因

**一、 脚本解释器异常**
	
比如不小心把 `/bin/bash` 写成了 `/bin/bsah`, 这个情况修改成正确可用的解释器即可。

**二、 错误的换行符**
	
在 windows 系统里，采用 `CRLF` 作换行符，而在 linux 下采用 `LF` 作换行符。

因为我的脚本是在 windows 系统下编写完成后，直接放到 gitlab 服务器上的。所以代码里的换行符是 `CRLF`，这造成了 git 运行文件时的失败。

修复方法，去掉脚本文件中的 `/r` 字符：

	sed -i -e 's/\r//g' .git/hooks/pre-commit


参考：

[Git - remote: error: cannot run hooks/post-receive: No such file or directory - Stack Overflow](https://stackoverflow.com/questions/11630433/git-remote-error-cannot-run-hooks-post-receive-no-such-file-or-directory/26767706)

另外推荐一篇讲 GIT 换行符的

[GitHub 第一坑：换行符自动转换 · Issue #22 · cssmagic/blog](https://github.com/cssmagic/blog/issues/22)