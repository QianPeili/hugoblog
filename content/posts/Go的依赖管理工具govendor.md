---
title: "Go的依赖管理工具govendor"
date: 2017-05-22T18:11:05+08:00
category: 
 - 笔记
tags: 
 - go
---

依赖管理是我接触 go 以来，觉得比较麻烦的事情。
不过在 go-1.5 版本中，出现了一个 `vendor` 的概念，并且 `vendor` 作为一个默认功能存在于1.7之后的版本中。

vendor的概念就类似于python中的 `virtualenv` 建立的 `env`，它是一个文件夹，里面存放了项目所用的依赖。如果项目中有个文件 import 了第三方的 package，而这个 package 又存在项目的 vendor 目录中，这个文件编译的时候会从 vendor 中 import 这个 package

比如：

	$GOPATH
	|	src/
	|	|	project/
	|	|	|	main.go
	|	|	|	vendor/
	|	|	|	|	github.com/pkg/sftp/
	|	|	|	|	golang.org/x/crypto/ssh/
	|	|	|	|	|	agent/

其中 `project/main.go` 中:

	import (
		...
		"golang.org/x/crypto/ssh"
	)

因为项目 project 中有 vendor 目录，并且存在 `golang.org/x/crypto/ssh`，所以这一行:

	import "golang.org/x/crypto/ssh"

在编译的时候会变成:

	import "project/vendor/golang.org/x/crypto/ssh"


[Go 1.5 Vendor Experiment - Google 文档](https://docs.google.com/document/d/1Bz5-UB7g2uPBdOx-rw5t9MxJwkfpx90cqG9AFL0JAYo/edit) 这里可以看到 vendor 的详细说明。


## Govendor

`govendor` 是一个管理 vendor 的工具

代码地址： [kardianos/govendor: Go vendor tool that works with the standard vendor file.](https://github.com/kardianos/govendor)


### govendor安装

	go get -u -v github.com/kardianos/govendor

### govendor使用

首先对项目进行初始化

	govendor init

然后列出项目所需的依赖
	
	govendor list

下面是我自己的项目所需的依赖，其中 `Bomber` 是当前项目的名称

	 e  github.com/golang/protobuf/proto
	 e  golang.org/x/crypto/bcrypt
	 e  golang.org/x/crypto/blowfish
	 e  gopkg.in/mgo.v2
	 e  gopkg.in/mgo.v2/bson
	 e  gopkg.in/mgo.v2/internal/json
	 e  gopkg.in/mgo.v2/internal/sasl
	 e  gopkg.in/mgo.v2/internal/scram
	pl  Bomber
	 l  Bomber/db
	 l  Bomber/gate
	 l  Bomber/handlers
	 l  Bomber/models
	 l  Bomber/network
	 l  Bomber/protodata
	pl  Bomber/test
	 l  Bomber/tools

每个 package 前有个 flag 标记，分别代表：

	+local    (l) packages in your project
	+external (e) referenced packages in GOPATH but not in current project
	+program  (p) package is a main package
	+all      +all packages

现在我们把第三方包加入vendor目录中

	govendor add +e

或者单独添加某个包
	
	govendor add {package_name}

当然还可以添加当前项目到 `vendor` 中
	
	govendor add +l

*添加当前项目似乎没有太大作用，但是在制作 `docker image` 的时候将当前项目加入 vendor 会省事不少。*
	
然后你会看到项目中的 vendor 目录中，已经把依赖的源码添加进去了。其中还有个 `vendor.json` 文件，里面记录了 vendor 目录中包含的 package 信息。

	{
		"comment": "",
		"ignore": "",
		"package": [
				...
				{
					"checksumSHA1": "aiUlobdvi4uVkijyZ856jdyvXng=",
					"path": "github.com/golang/protobuf/ptypes",
					"revision": "c9c7427a2a70d2eb3bafa0ab2dc163e45f143317",
					"revisionTime": "2017-03-07T00:15:33Z"
				},
                ...
        ]
		"rootPath": "Bomber"
	}


### 将vendor加入版本控制

什么！要把依赖加入版本控制中？！不会很臃肿吗？！

当然会臃肿……

如果你不把依赖源码加入版本控制中，
你可以直接通过`go get ./.. `安装所需依赖，但是不能指定版本

或者只保留 `vendor.json` 文件，然后用 `govendor sync` 命令来下载对应版本的依赖。但是由于 go 的很多第三方包都来自 github ，可靠性和稳定性都会比较差。更可怕的是，由于 GFW 的存在，会导致一些官方包无法下载。所以，把依赖的源码加入版本控制，是个比较好的选择。
