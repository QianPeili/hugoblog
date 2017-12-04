---
title: "升级 GCC 来解决 heapdump build Error"
date: 2017-08-16T21:10:24+08:00
categories: 
 - note
tags: 
 - gcc
 - node
 - linux
---

今天要部署一个`node.js`的项目，第一次玩`node.js`有点崩溃。安装依赖的时候一个`heapdump`库出现了`build error`。

在`github`上找到了类似的`issue`： [[error] heapdump@0.3.7 Build Error · Issue #72 · bnoordhuis/node-heapdump](https://github.com/bnoordhuis/node-heapdump/issues/72)

里面提到的解决方式是升级`GCC`：

> Well, I have fixed it!
> 
> My gcc version is 4.6.3 when I update to 4.9.1 and retry it work!

我查看了一下机器的`gcc`版本，是 4.4.7 的，问题应该就在这了。所以安装一个新版本的`gcc`应该就行了。

国内为了速度可以选择中科院的镜像： [Index of /gnu/gcc/](http://mirrors.opencas.org/gnu/gcc/)

我选择了 6.1.0 版本，下载源码后简单地根据官方教程 [InstallingGCC - GCC Wiki](https://gcc.gnu.org/wiki/InstallingGCC)
进行安装。

    tar xzf gcc-6.1.0.tar.gz
    cd gcc-6.1.0
    ./contrib/download_prerequisites
    cd ..
    mkdir objdir
    cd objdir
    $PWD/../gcc-6.1.0/configure --prefix=$HOME/gcc-6.1.0 --disable-multilib

因为`gcc`编译真的要好久好久，所以我用上了全部 cpu 核心。

    make -j 16
    make -j 16 install

替换旧版的`gcc`

	mv /usr/bin/gcc /usr/bin/gcc-4.4.7
	ln -s $HOME/gcc-6.1.0/bin/gcc /usr/bin/gcc

	mv /usr/bin/g++ /usr/bin/++-4.4.7
	ln -s $HOME/gcc-6.1.0/bin/g++ /usr/bin/g++

装完后再执行`npm install`就一切顺利了~

升级`gcc`后可能会引起动态库不一致的问题，可以参考这篇文章进行处理。

[解决类似 /usr/lib64/libstdc++.so.6: version `GLIBCXX_3.4.21' not found 的问题 - IT笔录](https://itbilu.com/linux/management/NymXRUieg.html)







