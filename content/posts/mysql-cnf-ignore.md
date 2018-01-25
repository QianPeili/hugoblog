---
title: "Mysql配置文件因为权限而未生效"
date: 2018-01-25T11:16:52+08:00
categories: 
 - note
tags: 
 - mysql
---

在项目组的机子上部署了一个 `mysql` 服务，上面跑了一个事件。

`mysql` 默认是关闭事件调度器的，我们需要通过手动运行命令打开它：

	SET GLOBAL event_scheduler = ON;  

但是这样一旦 `mysql` 重启，设置就失效了。因为我们的机子就是普通的台式机，公司有时候会断电，`mysql` 会时不时重启。于是我修改了 `/etc/my.cnf` 文件，设置 `event_scheduler=1`，重启 `mysql` 服务后发现配置并未生效。我怀疑是因为 `mysql` 并没有使用这份配置，于是我又修改了 `/etc/mysql/my.cnf`，重启后同样未生效。

偶然通过命令：

	mysql --print-defaults

发现打印出的内容有这样一条：

	mysql: [Warning] World-writable config file '/etc/mysql/my.cnf' is ignored.

网上查了一下，原因是因为配置文件可写权限太广（777），`mysql` 认为文件不安全，会忽略这份配置。

因此修改文件权限：

	chmod 644 /etc/mysql/my.cnf

即可解决问题。 再运行 `mysql --print-defaults` 就不会输出之前那条 `warning` 了。

继续重启 `mysql`，查看事件调度器已经自动开启。

### TODO

1. 没有找到对应的启动日志，是否无法根据启动日志获取这条warning。
2. service mysql status 命令可以看到一些日志，但也不包含这条warning。