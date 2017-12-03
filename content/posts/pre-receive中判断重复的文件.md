---
title: "通过 pre-hook 判断是否推送重复的内容" 
date: 2017-07-14T18:20:59+08:00
category:
 - 笔记
tags: 
 - git
 - python
---

# 前言

昨天运维同事找我，要求实现一个 pre-receive 钩子，在里面判断推送的文件中是否有重复的文件，如果有就拒绝此次 push。问我该如何实现。

虽然觉得这个判断标准有点莫名其妙，但还是想办法实现了一下。途中遇到过很多问题，以为无法实现，不过最终还是找到了解决方法。

# 简单介绍

## pre-receive

> 参考连接： [Git - githooks Documentation](https://git-scm.com/docs/githooks#pre-receive)

`pre-receive` 是 Git 众多 Hooks 中的一种，在远程库中触发。当 server 端(比如：`gitlab`)收到用户的 push 内容后，在更改远程库内容前，如果 `pre-receive` 文件存在，就会执行这个文件。文件正常退出（exit 0）表示接受该次 push, 反之则拒绝。

server 端执行 pre-receive 时，没有参数，但会从标准输入写入一行数据：

	<old-value> SP <new-value> SP <ref-name> LF


## Git object 

> 参考连接：
> 
> - [Git Book - The Git Object Model](http://shafiulazam.com/gitbook/1_the_git_object_model.html)
> - [Git Magic - Chapter 8. Secrets Revealed](http://www-cs-students.stanford.edu/~blynn/gitmagic/ch08.html)

在使用 `git push` 命令时，终端可以看到类似下面的输出

 `Writing objects: 100% (3/3), 253 bytes | 0 bytes/s, done.` 

Git 是一个内容寻址系统，即核心是通过 `key-value` 方式存储内容。 这里的 objects 就是存储的内容—— `value`，它们都由一串 40 位长度的散列码(SHA-1)所代表，即 `key`。如：

	6ff87c4664981e4397625791c8ea3bbb5f2279a3


object 一共有四种类型： **blob**, **tree**, **commit**, **tag**。其中 `blob` 保存的是文件信息。
我们可以通过 `git ls-tree` 命令查看每个 `commit object` 中保存的文件信息：

	$ git ls-tree HEAD
	100644 blob 63c918c667fa005ff12ad89437f2fdc80926e21c    .gitignore
	100644 blob 5529b198e8d14decbe4ad99db3f7fb632de0439d    .mailmap
	100644 blob 6ff87c4664981e4397625791c8ea3bbb5f2279a3    COPYING
	040000 tree 2fb783e477100ce076f6bf57e4a6f026013dc745    Documentation
	100755 blob 3c0032cec592a765692234f1cba47dfdcc3a9200    GIT-VERSION-GEN
	100644 blob 289b046a443c0647624607d471289b2c7dcd470b    INSTALL
	100644 blob 4eb463797adc693dc168b926b6932ff53f17d0b1    Makefile
	100644 blob 548142c327a6790ff8821d67c2ee1eff7a656b52    README

# 实现

## 主要思路

文件 (blob) 的散列码由其内容决定，所以相同内容的文件的散列码是一样的，
因此我们可以通过比较新版本中所有 blob 散列码是否重复，来判断是否存在内容相同的文件。

## 步骤

1. 从标准输入获取到新的 commit object
2. 用 ls-tree 命令获取所有新版本的 blob 散列码
3. 判断是否存在相同的散列码，并进行对应的操作。

## python代码

版本为 python2.7。

	#!/usr/bin/python
	import sys
	import subprocess
	
	data = sys.stdin.read()
	_, new_obj, _ = data.split()
	
	
	def command_output(command):
	    output = subprocess.check_output(
	        command, shell=True).split("\n")
	    result = [l.strip() for l in output if l.strip()]
	    return result
	
	
	# get all blob-objects in new_obj
	cmd = "git ls-tree -r %s" % new_obj
	new_blobs = command_output(cmd)
	new_shas = [b.split(" ")[2].split("\t")[0] for b in new_blobs]
	
	# check if have same blob objects
	if len(set(new_shas)) < len(new_shas):
	    print "reject"
	    sys.exit(1)
	
	print "accept"
	sys.exit(0)

后续可以优化一下，输出重复的文件名称。

Gist 地址： [python pre-receive for check same blob object](https://gist.github.com/QianPeili/ce7c2f4acc3bcd15d876ebb44f93e736)

