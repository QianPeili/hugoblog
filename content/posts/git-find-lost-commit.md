---
title: "GIT 找回丢失的 commit"
date: 2018-04-26T20:01:32+08:00
categories: 
 - note
tags: 
 - git
---


之前为了修一个BUG，脑子一抽把当前写了很久的功能 hard reset 掉了，其它分支也没有记录。

不过好在虽然项目的节点图里看不到那些节点了，但 git 还是保留了这些提交记录，我们可以使用 git 重新找回丢失的节点。

### Step 1 查看最近提交

	git log --reflog 

上面的命令可以显示最近的 commit 数据。如：
	
	commit adcf5a5a2b3533102f761ff94493d0f6a4a35257
	Author: 爪子 <zhuazi@example.com>
	Date:   Thu Apr 26 17:48:13 2018 +0800
	
	    modify something
	
	commit 78ac5cf1ca16c90946d40769299ec91b55c495b2
	Author: 爪子 <zhuazi@example.com>
	Date:   Thu Apr 26 17:42:12 2018 +0800
	
	    fix something.
	
	commit 2661e4b9859aefe261dc81c8f3e66d172edcc755
	Author: 爪子 <zhuazi@example.com>
	Date:   Thu Apr 26 17:37:47 2018 +0800
	
	    refactor something


### Step 2 回到需要的节点

我们可以看到 `modify something` 这个节点的哈希值是`adcf5a5a2b3533102f761ff94493d0f6a4a35257`，假设我们希望回到这个节点，我们就可以用这个命令：

	git reset --hard adcf5a5a2b3533102f761ff94493d0f6a4a35257

当前分支的 `HEAD` 就会跳到这个节点了。