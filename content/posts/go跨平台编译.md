---
title: "Go跨平台编译"
date: 2017-11-01T19:22:02+08:00
category: 
 - 笔记
tags: 
 - go
---


公司开发系统是 Windows，服务器部署环境是 Linux。一开始测试的时候选的是 git pull 带生产机上，编译后启动。

但这个方法太笨，一是代码容易泄露，二是生产机要同时同步库文件，不管是 go get 方式还是 govendor 方式都显得太麻烦。

不过好在 Go 的跨平台编译非常方便，所以我们完全可以在开发环境下编译好可执行文件，再放入生产机部署。

总结一下 windows环境下 go 跨平台编译的方式。

## GOOS 和 GOARCH

关于这两个环境变量的含义，我们可以从官方文档： [Installing Go from source - The Go Programming Language](https://golang.org/doc/install/source#environment)中了解，分别是目标系统的 `操作系统` 和 `编译架构`。只要我们设置好这两个参数的内容，就可以轻松进行跨平台编译了。

支持的列表

    $GOOS	$GOARCH
    android	arm
    darwin	386
    darwin	amd64
    darwin	arm
    darwin	arm64
    dragonfly	amd64
    freebsd	386
    freebsd	amd64
    freebsd	arm
    linux	386
    linux	amd64
    linux	arm
    linux	arm64
    linux	ppc64
    linux	ppc64le
    linux	mips
    linux	mipsle
    linux	mips64
    linux	mips64le
    netbsd	386
    netbsd	amd64
    netbsd	arm
    openbsd	386
    openbsd	amd64
    openbsd	arm
    plan9	386
    plan9	amd64
    solaris	amd64
    windows	386
    windows	amd64

## Windows cmd

最直接的了，需要修改环境变量

    set GOOS=linux
    set GOARCH=amd64
    go build -o main

## Git Bash

通过安装 Git bash 或者其它 bash 工具

	$ GOOS=linux GOARCH=amd64 go build -o main

## PowerShell

说起来，我一直在 PowerShell 下使用 cmd 的命令方式编译，结果发现编译的文件都是 windwos 平台下的。

后来发现 PowerShell 与 cmd 有很大不同，所以用 set 命令设置环境变量是完全无效的。正确方法如下：

    $env:GOOS = "linux"
    $env:GOARCH = "amd64"
    go build -o main


