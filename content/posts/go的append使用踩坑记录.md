---
title: "Go的append使用踩坑记录"
cate: 2017-12-03T18:23:21+08:00
category:
 - 笔记
tags: 
 - go
---

`go` 有个很好用的数据类型——`slice`，有点类似 `python` 里的 `list`。`go` 中给 `slice` 添加元素，用的是 `append` 函数，它跟`python`中的`list.append`方法有些不同，每次都会返回一个对象，所以我之前一直都把`go`中的`append`当做`python`中的 `+` 变形。

在我之前的理解中，某些情况下，两者其实是差不多的，比如：

`go` 代码

	package main
	
	import "log"
	
	func main() {
		a := []int{1, 2}
		a = append(a, []int{3, 4}...)
		log.Println(a)
	
	}


`python` 代码

	a = [1, 2]
    a = a + [3, 4]
    print(a)

输出结果 `a` 都是： **[1, 2, 3, 4]**，两者确实很相似

但是我今天在刷题目时，想要实现这么一个情况：

python 代码：

	a = [1, 2, 3, 4, 5]
    b = a[:2] + a[3:]

执行后的结果： a == [1, 2, 3, 4, 5], b == [1, 2, 4, 5]

然后我用 `go` 写了这么一段代码：

	a := []int{1, 2, 3, 4, 5}
	b := append(a[0:2], a[3:5]...)

然后我发现，`a` 的数据变成了 **[1, 2, 4, 5, 5]**。 太奇怪了，看来我对 `append` 的理解有些错误，它似乎和 `python` 中的列表的 `+` 操作并不一样。我看了 `append` 函数的注释：

*// The append built-in function appends elements to the end of a slice. If
// it has sufficient capacity, the destination is resliced to accommodate the
// new elements. If it does not, a new underlying array will be allocated.
// Append returns the updated slice. It is therefore necessary to store the
// result of append, often in the variable holding the slice itself:
//	slice = append(slice, elem1, elem2)
//	slice = append(slice, anotherSlice...)
// As a special case, it is legal to append a string to a byte slice, like this:
//	slice = append([]byte("hello "), "world"...)
func append(slice []Type, elems ...Type) []Type*


注释中 **If it has sufficient capacity, the destination is resliced to accommodate the
new elements.** 说明了，如果被传入的切片拥有足够容量，那么被添加的元素会直接放到该切片空闲的位置上去。也就是说，`append` 并不是一定返回一个新的 `slice` 对象，也有可能是改变了原先的对象并返回。

对 `go` 的切片进行选取时，如果位置是从 `0` 开始，那么这个新选取的切片会继承原有切片的容量，所以在上面的例子中, `a[0:2]` 的长度是2，容量是依旧是5。因此 `append` 操作会直接在原先的 `a` 上进行赋值操作。

`go` 的 `append` 其实既不是 `python` 中的 `+` 也不是 `list.append`，它们有很多不同。

*ps: 下面是对一些容易搞混的 `append` 操作的记录*


	package main
	
	import (
		"log"
	)
	
	func main() {
		a := []int{1, 2, 3, 4}
		b := append(a[0:2], []int{7, 8}...)
		log.Println(a, b)
		// a == [1, 2, 7, 8]
        // b == [1, 2, 7, 8]
	
		a = []int{1, 2, 3, 4}
		b = append(a[1:3], []int{7, 8}...)
		log.Println(a, b)
		// a == [1, 2, 3, 4]
        // b == [1, 2, 7, 8]
	
		a = []int{1, 2}
		b = append(a[:1], 3)
		log.Println(a, b)
		// a == [1, 3]
        // b == [1, 3]
	
		a = []int{1, 2}
		b = append(a, 3)
		log.Println(a, b)
		// a == [1, 2]
        // b == [1, 2, 3]
	}
