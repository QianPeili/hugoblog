---
title: "Git Submodule的使用"
date: 2017-06-13T19:11:01+08:00
category: 
 - 笔记
tags: 
 - git
---

# Submodule介绍

前几天因为项目需求，接触了`Git`的`Submodule`功能。

客户端和服务端的项目各自独立开发，因为要用到一个共同的子库——SubA，SubA即需要独立保持版本信息，又需要同时给客户端和服务端的项目提供支持。`Submodule`正好可以帮助我们管理这个问题。

# Submodule语法
	
	git submodule [--quiet] add [<options>] [--] <repository[<path>]
	git submodule [--quiet] status [--cached] [--recursive] [--] [<path>…​]
	git submodule [--quiet] init [--] [<path>…​]
	git submodule [--quiet] deinit [-f|--force] (--all|[--] <path>…​)
	git submodule [--quiet] update [<options>] [--] [<path>…​]
	git submodule [--quiet] summary [<options>] [--] [<path>…​]
	git submodule [--quiet] foreach [--recursive] <command>
	git submodule [--quiet] sync [--recursive] [--] [<path>…​]
	git submodule [--quiet] absorbgitdirs [--] [<path>…​]

# Submodule实践

目前有3个库： Main， SubA, SubB

## 导出Main库

	git clone ssh://git@192.168.1.103:8080/qianpeili/Main.git

## 添加SubA作为子模块

	git submodule add ssh://git@192.168.1.103:8080/qianpeili/SubA.git

可以看到变化

	$ git status
	On branch master
	Your branch is up-to-date with 'origin/master'.
	Changes to be committed:
	  (use "git reset HEAD <file>..." to unstage)
	
	        new file:   .gitmodules
	        new file:   SubA

查看一下`.gitmodules`内容
	
	[submodule "SubA"]
	        path = SubA
	        url = ssh://git@192.168.1.103:8080/qianpeili/SubA.git

我们可以看到SubA被的信息被记录在了`.gitmodules`文件中

## 提交改动


	$ git add .
	
	$ git commit -m "add submodule SubA"
	[master ece1ec2] add submodule SubA
	 2 files changed, 4 insertions(+)
	 create mode 100644 .gitmodules
	 create mode 160000 SubA

看到提交信息，我们发现 SubA 的记录模式是 `16000`。这是Git的一种特殊模式，这个模式下的文件不被当作正常的版本控制下的文件。

## 添加另一个SubA

在前面的 .gitmodules 文件中，我们看到，`submodule`后面跟着的子项目的名称，这个名称决定了子模块在目录中的位置，即便是同一个远程库，不同的路径和名字都认为是不同的子模块。

比如我们再添加一个SubA作为子项目，但是放在 submodule/A 目录下。
	
	$ git submodule add ssh://git@192.168.1.103:8080/qianpeili/SubA.git submodule/A
	Cloning into 'F:/Main/submodule/A'...
	remote: Counting objects: 6, done.
	remote: Compressing objects: 100% (2/2), done.
	remote: Total 6 (delta 0), reused 0 (delta 0)
	Receiving objects: 100% (6/6), done.
	warning: LF will be replaced by CRLF in .gitmodules.
	The file will have its original line endings in your working directory.

可以看到主项目中多了一个 `submodule/` 目录，里面有一个名字为`A`的文件夹。.gitmodule目中的内容为：

	[submodule "SubA"]
	        path = SubA
	        url = ssh://git@192.168.1.103:8080/qianpeili/SubA.git
	[submodule "submodule/A"]
	        path = submodule/A
	        url = ssh://git@192.168.1.103:8080/qianpeili/SubA.git



## 导出项目Main

导出包含子项目的库。

	git clone ssh://git@192.168.1.103:8080/qianpeili/Main.git MainCopy

	$ ls
	README.md  SubA/  submodules/

目录下，已经包含了子项目的目录，但是文件夹下没有任何文件。

## 导出子项目

	$ git submodule init
	
	$ git submodule update
	Cloning into 'F:/MainCopy/SubA'...
	Cloning into 'F:/MainCopy/submodule/A'...

还有更简单的方发是，在 `git clone` 主项目时，后面加上 `--recursive`参数，那么 Git 会自动帮我们导出所有子项目。

另外，使用 `git submodule update` 会导出所有子项目的内容，如果想要单个子项目，只需要在命令后面跟上子项目的名称即可。

## 更新子模块


	$ git submodule update --remote SubA



# 参考文档

[Git - git-submodule Documentation](https://git-scm.com/docs/git-submodule)





