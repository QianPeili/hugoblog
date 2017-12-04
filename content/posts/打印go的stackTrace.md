---
title: "打印 Go 的Stack Trace"
date: 2017-11-04T17:42:02+08:00
categories: 
 - tips
tags: 
 - go
---


算是写了第一个比较正式的 `Go` 项目吧，然后现在在考虑如何设计日志系统。

服务采用 `TCP` 长连接，每个 `session` 都是单独启动一个 `goroutine` 处理事务（其实发消息也有一个协程，不过发消息部分逻辑简单），所以考虑在最顶层写一个 `recover` 函数，收集上报的错误信息并输出。大致是这么个东西：

    package main
    
    import (
    	"log"
    	"errors"
    )
    
    func main() {
    
    	defer func() {
    		if r := recover(); r != nil {
    			log.Println(r)
    		}
    	}()
    
    	// make a panic
    	panic(errors.New("ha! A bug"))
    }
    
输出： `2017/11/04 17:19:16 ha! A bug`

但如果只是这样，其实并不能帮助我们很好的排查问题，我们知道有了一个 `BUG`，但不知道是哪里除了问题。因此我们打印出捕获错误时的 Stack Trace。内建库 `runtime` 就有这个功能：

**func Stack(buf []byte, all bool) int**

	Stack formats a stack trace of the calling goroutine into buf and returns the number of bytes written to buf. If all is true, Stack formats stack traces of all other goroutines into buf after the trace for the current goroutine.

不过更方便的是使用 `runtime/debug` 的 `Stack()` 功能：

	package main
	
	import (
		"log"
		"errors"
		"runtime/debug"
	)
	
	func main() {
	
		defer func() {
			if r := recover(); r != nil {
				log.Printf("%s\n%s\n", r, debug.Stack())
			}
		}()
	
		// make a panic
		panic(errors.New("ha! A bug"))
	}

现在我们可以看到更明确的错误情况：

	2017/11/04 17:33:31 ha! A bug
	goroutine 1 [running]:
	runtime/debug.Stack(0xc04205be80, 0x4baa00, 0xc0420301c0)
		C:/Go/src/runtime/debug/stack.go:24 +0xae
	main.main.func1()
		D:/Projects/mygo/src/Test/example/main.go:13 +0x75
	panic(0x4baa00, 0xc0420301c0)
		C:/Go/src/runtime/panic.go:491 +0x291
	main.main()
		D:/Projects/mygo/src/Test//example/main.go:19 +0xc9