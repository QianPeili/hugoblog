---
title: "【翻译】Go语言中的反射机制"
date: 2017-07-28T20:48:11+08:00
category: 
 - 翻译
tags: 
 - go
---

原文链接： [The Laws of Reflection - The Go Blog](https://blog.golang.org/laws-of-reflection)

作者： Rob Pike   

翻译： 钱佩丽

# 介绍

反射( `reflection` )在计算机中指的是一个程序审查自身结构的能力，尤其是类型( `Types` )。
反射是元编程的一种形式，也是让人觉得头疼的一个点。

在这篇文章中，我们将对反射在 Go 中的工作原理做一些解释说明。
各种计算机语言的反射机制不尽相同（有的甚至没有反射），
但这篇文章，我们只关心 Go 语言中的反射，
文中所有反射，都是在 Go 语言的背景下。

# 类型和接口

由于反射建立在类型机制上，所以我们先来介绍一下 Go 的类型。

Go 是静态类型语言，每个变量都有一个静态的类型，就是说，在编译的时候，变量的类型必须是固定且唯一的。
比如： `int`, `float32`, `*MyType`, `[]byte` 等等。如果我们如下申明：

    type MyInt int
    
    var i int
    var j MyInt

那么，变量 `i` 就是 `int` 类型，变量 `j` 就是 `MyInt` 类型。 尽管 `i` 和 `j` 拥有相同的底层类型，
但他们本身是完全不同的类型，他们不能直接互相赋值，必须通过类型转换。

接口( `interface` )是 Go 类型中非常重要的一种类型，它代表一些方法的固定集合。
一个接口变量可以存放任意的（非接口）具体值，只要这个值实现了接口包含的所有方法。
比较著名的一对例子就是 `io.Reader` 和 `io.Writer`，
`Reader` 和 `Writer` 两个类型都来自包 [io-package](http://golang.org/pkg/io/)。


    // Reader is the interface that wraps the basic Read method.
    type Reader interface {
    Read(p []byte) (n int, err error)
    }
    
    // Writer is the interface that wraps the basic Write method.
    type Writer interface {
    Write(p []byte) (n int, err error)
    }

任何类型，只要实现了上述的 `Read` （或者 `Write`）方法，我们就说它实现了 `io.Reader` （或者 `io.Writer`）接口。
也就是说，`io.Reader` 对象可以存放任何实现了 `Read` 方法的值。

    var r io.Reader
    r = os.Stdin
    r = bufio.NewReader(r)
    r = new(bytes.Buffer)
    // and so on
    
但必须清楚一点，不管 `r` 的具体值是什么，`r` 的类型只会是 `io.Reader`： Go 是静态类型，而 `r` 的静态类型就是 `io.Reader`。


接口的一个极端例子是空接口：

	interface{}

它代表一个空的方法集合并且适用于任何值，因为任务值都有至少零个或者更多个方法。

有些人可能会误解 Go 的接口是动态类型，
但它其实是静态类型：
一个值的接口类型永远只会是同一个静态类型，
即使在程序运行时，
接口的值类型发生了变化，
这个值也永远都会满足这个接口所定义的方法。

我们需要明确了解以上这些信息，因为反射和接口密切相关。

# 接口的表示

Russ Cox 曾写过一篇 [文章](http://research.swtch.com/2009/12/go-data-structures-interfaces.html) 详细讲述了 Go 语言中接口的表示。
这里就不再长篇大论了，只做一个简短总结。

一个接口变量存储的内容是成对的：
变量的具体值和值的类型描述。
更精确的说，值就是实现接口的底层的数据，
而类型就是底层数据的原始类型。
举个例子，如果做如下申明：

	var r io.Reader
	tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
	if err != nil {
	    return nil, err
	}
	r = tty

`r` 包含的 (值、类型) 对大致是： (`tty`, `*os.File`)。
注意尽管接口只需要我们提供 `Read` 方法，
但是 `*os.File` 类型实现了 `Read` 以外的方法，
内部的值包含了值的类型信息，因此我们也可以这么做：

	var w io.Writer
	w = r.(io.Writer)

上面的表达式完成了一次类型断言。
断言的内容是 `r` 内部的值也实现了 `io.Writer` 的接口，
因此我们可以把这个值赋值给 `w`。赋值后，
`w` 保存了跟 `r` 一样的一对信息(`tty`, `*os.File`)。
接口的静态类型决定了接口变量可以调用哪些方法，
而不用去管这个具体值是否拥有更多其它的方法。

接下来，我们来实现以下代码：

	var empty interface{}
	empty = w

我们的接口变量 `empty` 包含了同样的信息对 (`tty`, `*os.File`)。
这很好理解：一个空接口可以存放任何值，
并包含了所有我们需要的关于这个值的信息。

（在这里我们不需要进行类型断言，
因为我们知道 `w` 肯定满足成为一个空接口的要求。
而在将值从一个 `Writer` 接口类型赋值到 `Reader` 接口类型的例子中，
我们必须要通过类型断言明确知道这个值是否符合新的接口类型，
因为 `Writer` 和 `Reader` 接口所需要实现的方法是完全不同的。）

一个重要细节是，
一个接口变量的信息对永远只会是（值，具体类型）,
而不会是（值，接口类型）。
接口不存放接口值。

# 反射定律

## 第一定律：反射可以将接口值转换成反射对象


在基础层面，反射仅仅是一种用来检查接口变量值-类型对的机制。
首先，我们需要了解 [reflect - package](https://golang.org/pkg/reflect/) 的:  
[Type](https://golang.org/pkg/reflect/#Type) 和 
[Value](https://golang.org/pkg/reflect/#Value)，
这两个类型可以让我们访问一个接口变量的内容。另外两个简单的函数， 
`reflect.TypeOf` 和 `reflect.ValueOf` 可以帮我们获取接口值的 `reflect.Value` 和 `reflect.Type`。
当然通过 `reflect.Value` 可以很容易得到 `reflect.Type`，但现在我们先把这两者的概念分离。

现在我们开始使用 `TypeOf` 吧：

	package main
	
	import (
	    "fmt"
	    "reflect"
	)
	
	func main() {
	    var x float64 = 3.4
	    fmt.Println("type:", reflect.TypeOf(x))
	}

程序输出：

	type: float64

你也许会好奇这段代码跟接口有什么关系，
这个程序看起来只是单纯地将一个 `float64` 的值赋值给 `x`，
而不是一个接口值，然后调用了 `reflect.TypeOf` 函数。
但是这里确实用到了接口， 
在文档 [godoc reports](https://golang.org/pkg/reflect/#TypeOf) 里可以看到，`reflect.TypeOf` 的参数是一个空接口。

	// TypeOf returns the reflection Type of the value in the interface{}.
	func TypeOf(i interface{}) Type

当我们调用 `reflect.TypeOf(x)`时，
`x` 是被放在空接口中作为参数传递给函数的，
然后 `reflect.TypeOf` 通过解析空接口变量来复原类型信息。

而 `reflect.ValueOf` 函数自然是复原它的值信息
（现在开始，让我们忽略样板，只关注代码的执行）。

	var x float64 = 3.4
	fmt.Println("value:", reflect.ValueOf(x).String())

输出：

	value: <float64 Value>

`reflect.Type` 和 `reflect.Value` 还有许多其它好用的方法。
一个显著的例子是 `Value` 有一个 `Type` 方法，
它会返回一个 `reflect.Value` 的 `Type` 。 
还有一个例子是，`Type` 和 `Value` 都有一个 `Kind` 方法，
返回表示底层数据存储的类型常量，
比如: `Uint`, `Float64`, `Slice` 等等。 
`Value` 还有 `Int` 和 `Float` 方法来帮我们挖掘底层数据的值 (比如 `int64` 和 `float64`):

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("type:", v.Type())
	fmt.Println("kind is float64:", v.Kind() == reflect.Float64)
	fmt.Println("value:", v.Float())

输出：

	type: float64
	kind is float64: true
	value: 3.4

另外还有一些方法如 `SetInt` 和 `SetFloat`，
但想要使用它们我们必须要懂 "可设值性（`settability`）", 
这是反射第三定律的主题，我们将会在下面讨论。

反射库有很多值得单独讨论的内容。 
首先，为了保持 API 的简洁，`Value` 的 `getter` 和 `setter` 方法会使用范围最大的类型来存放数据，
比如： 用 `int64` 来存放所有含符号整数。 
也就是说 `Value` 的 `Int` 方法返回的是数据类型 `int64`，`SetInt` 方法接收一个 `int64` 数据。 有时候我们就不得不转换类型来应对这个情况:

	var x uint8 = 'x'
	v := reflect.ValueOf(x)
	fmt.Println("type:", v.Type())                            // uint8.
	fmt.Println("kind is uint8: ", v.Kind() == reflect.Uint8) // true.
	x = uint8(v.Uint())                                       // v.Uint returns a uint64.

反射对象的 `Kind` 属性是用来描述底层的类型，
而不是静态类型。 
如果一个反射对象拥有一个自定义整数类型，比如： 

	type MyInt int
	var x MyInt = 7
	v := reflect.ValueOf(x)

尽管 `x` 的静态类型是 `MyInt`, 而不是 `int`， 
但 `v` 的 `Kind` 依旧是 reflect.Int。换句话说就是, 
`Kind` 无法区分一个 `int` 类型和 MyInt类型，
但是 `Type` 可以。

## 第二定律：反射可以将反射对象转换成接口值

跟物理反射类似，Go 的反射也可以生成自己的逆像。

对一个 reflect.Value，
我们可以通过 `Interface` 获得其接口值，
实际上这个方法重修打包成用于表示接口的（`Value`，`Type`）对并返回。

	// Interface returns v's value as an interface{}.
	func (v Value) Interface() interface{}

因此我们可以用以下方式输出反射对象 `v` 对应的 `float64` 值：

	y := v.Interface().(float64) // y will have type float64.
	fmt.Println(y)

We can do even better, though. 
正如之前的例子所展示的，
fmt.Printf 等函数都是将参数放在空接口中传入，然后解包进行操作。
所以，如果想要正确输出 `reflect.Value` 的内容，
我们需要先调用 `Interface` 方法，
再把获得的结果传给格式化输出程序:

	fmt.Println(v.Interface())

(为什么不直接 fmt.Println(v)? 
因为 `v` 是一个 reflect.Value， 
但我们需要的是它所存放的具体值。)
因为 `v` 是一个 `float64` 数字，我们也可以格式化输出浮点类型:

	fmt.Printf("value is %7.1e\n", v.Interface())

得到结果：

	3.4e+00

再强调一遍， 我们并不需要对 `v.Interface()` 的结果做 `float64` 的类型断言， 
空接口值内部已经包含了对应的信息，在输出函数中会对这些信息进行恢复。

总而言之，除了那些只能是空接口的值，
`Interface` 方法可以将 `ValueOf` 函数的结果进行逆转。

重申： 反射可以将接口值转换成反射对象，也可以逆向转回接口值。

## 第三定律：只有具有可设置性的反射对象才可以修改

反射的第三条定律是最微妙也是最令人迷惑的，
但如果我们从第一条定律开始推，那么就很好理解了。

这里有一段无法正常运行，但很值得学习的代码：

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	v.SetFloat(7.1) // Error: will panic.

如果你运行这段代码，就会看到这样的报错信息：

	panic: reflect.Value.SetFloat using unaddressable value

这里的问题并不是说 `7.1` 这个数据无法访问，
而是因为 `v` 是不可设置（`settable`）的。
可设置性（`settability`）是 `reflect.Value` 的一个属性，
并不是所有 `reflect.Value` 都有这个属性。

`Value` 有一个 `CanSet` 方法判断其是不是可设置的。比如：

	var x float64 = 3.4
	v := reflect.ValueOf(x)
	fmt.Println("settability of v:", v.CanSet())

输出：

	settability of v: false

对一个不可设置的 `Value` 进行 `Set` 操作，就会报错。那么，什么是可设置性？

可设置性有点类似可寻址性（`addressability`）， 
但是比可寻址性更严格。 可设置性表示的是，
反射对象是否可以修改组成其自身的实际存储信息，
它由反射对象是否存有原始对象决定。当我们运行：

	var x float64 = 3.4
	v := reflect.ValueOf(x)

我们调用 `reflect.ValueOf` 时，传进去的是一个 `x` 的拷贝对象生成的接口，
而不是 `x` 本身。如果以下语句：

	v.SetFloat(7.1)

我们假设它成功了，也不会改变 `x` 本身，尽管 `v` 的内容来自 `x` 。
这个语句改变的只是包含在接口值中的 `x` 的拷贝对象，
对 `x` 本身来说并不受任何影响。这样的结果会造成误解，
也不产生实际作用，所以这是非法的，
这也是可设置性属性用来避免的情况。

虽然看起来很离奇，但它其实只是用一个奇怪的形式，
实现了一个我们很熟悉的方法。
假设我们要把 `x` 传给一个函数:

	f(x)

我们并不能期望 `f` 能够改变 `x` 的内容，
因为我们传进的是一个拷贝值，而不是 `x` 本身。 
如果我们想要 `f` 能够修改 `x`， 
我们必须传递 `x` 的地址给函数(也就是 `x` 的指针):

	f(&x)

现在看起来就比较直观和熟悉了，
反射的原理其实也是同样的。
如果我们想通过反射修改 `x`，
我们必须将 `x` 的指针传给反射工具。

我们来试试吧。首先我们像往常一下初始化 `x` ，
然后创建一个指向它的指针的反射 `p`。

	var x float64 = 3.4
	p := reflect.ValueOf(&x) // Note: take the address of x.
	fmt.Println("type of p:", p.Type())
	fmt.Println("settability of p:", p.CanSet())

输出：

	type of p: *float64
	settability of p: false

反射对象 `p` 并没有可设置性，但 `p` 并不是我们想要修改的，`*p` 才是。 
我们可以使用 `Value` 的 `Elem` 方法，
间接通过指针来获取它所指向的对象，我们把结果赋值给 `v`:

	v := p.Elem()
	fmt.Println("settability of v:", v.CanSet())

现在，`v` 就是一个可设置的反射对象了：

	settability of v: true

这里的 `v` 代表的就是 `x`，我们终于可以通过 `v.SetFloat` 来修改 `x` 的值了：

	v.SetFloat(7.1)
	fmt.Println(v.Interface())
	fmt.Println(x)

如我们所料，输出变成了：

	7.1
	7.1

反射有些难以理解，但它所做的都是一个编程语言该做的，
只不过用反射的 `Types` 和 `Values` 进行了封装。 
记住一点，反射值需要知道变量的地址才能修改他们所代表的对象。

## 结构体

在我们前面的例子中， v 并不是指针，它只是从指针获取的一个对象。 我们通常用这个方式来通过反射修改结构体。只要我们拥有这个结构体的地址，我们就可以修改结构体的字段。

这里有一个处理结构体 `t` 的小例子。我们使用结构体的地址来创建一个反射对象，稍后我们会用它来修改这个结构体。然后我们将 `s` 的类型信息记录在 `typeOfT`，然后通过它来迭代获取字段信息 (详细内容请参考 `reflect` 库)。我们从结构体类型中提取出了字段的名字，但这些字段本身通常也是 `reflect.Value` 对象。

	type T struct {
	    A int
	    B string
	}
	t := T{23, "skidoo"}
	s := reflect.ValueOf(&t).Elem()
	typeOfT := s.Type()
	for i := 0; i < s.NumField(); i++ {
	    f := s.Field(i)
	    fmt.Printf("%d: %s %s = %v\n", i,
	        typeOfT.Field(i).Name, f.Type(), f.Interface())
	}

程序的输出结果是：
	
	0: A int = 23
	1: B string = skidoo

关于可设置性的介绍，这里还有一点需要注意：`T` 的字段名称都是大写开头的（公有的，可被外部访问的）， 因为只有结构体的公有字段才是可设值的。

因为 `s` 包含的是一个可设值的反射对象，因此我们可以修改结构体的字段值。

	s.Field(0).SetInt(77)
	s.Field(1).SetString("Sunset Strip")
	fmt.Println("t is now", t)

输出结果:

	t is now {77 Sunset Strip}

我们我们的程序中，`s` 并不是由 `&t` 而是 `t` 创建而来，那么 `SetInt` 和 `SetString` 的调用都会因为 `s` 的不可设置性而失败。

# 总结

这里再次总结下反射的三个定律:

- 反射可以将接口值转换成反射对象。

- 反射可以将反射对象转换成接口值。
 
- 只有具有可设置性的反射对象才可以修改。

一旦你理解了 Go 中反射的机制，尽管依旧有点微妙，丹妮你已经可以很容易地使用它们了。 这是一个很强大的工具，所以你使用的时候必须严谨和小心。

还有很多其它的反射内容本文没有涉及——在 `channels` 中的收发，内存分配，切片和映射的使用，方法和函数调用——但是这篇文章已经足够丰富了，我们将会在后续的文章中讨论这些内容。
