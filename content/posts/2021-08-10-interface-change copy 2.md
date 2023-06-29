---
title: "interface 原理 - 类型转换"
date: 2021-08-10T23:27:21+08:00
authors: hhf
tags: [Go, golang]
categories: [Go,golang]
toc: true
---

hi, 大家好，我是 haohognfan。

可能你看过的 interface 剖析的文章比较多了，这些文章基本都是从汇编角度分析类型转换或者动态转发。不过随着 Go 版本升级，对应的 Go 汇编也发生了巨大的变化，如果单从汇编角度去分析 interface 变的非常有难度，本篇文章我会从**内度分配+汇编**角度切入 interface，去了解 interface 的原理。

限于篇幅 interface 有关动态转发和反射的内容，请关注后续的文章。本篇文章主要是关于类型转换，以及相关的容易出现错误的地方。


![eface](https://cdn.jsdelivr.net/gh/georgehao/img/eface.png)

![iface](https://cdn.jsdelivr.net/gh/georgehao/img/iface.png)

## eface 

```go
func main() {
	  var ti interface{}
	  var a int = 100
	  ti = a
	  fmt.Println(ti)
}
```

这段最常见的代码，现在提出一些问题：

* 如何查看 ti 是 eface 还是 iface ?
* 值 100 保存在哪里了 ？
* 如何看 ti 的真实的值的类型 ？

大部分源码分析都是从汇编入手来看的，这里也把对应的汇编贴出来

```s
0x0040 00064 (main.go:44)	MOVQ	$100, (SP)
0x0048 00072 (main.go:44)	CALL	runtime.convT64(SB)
0x004d 00077 (main.go:44)	MOVQ	8(SP), AX
0x0052 00082 (main.go:44)	MOVQ	AX, ""..autotmp_3+64(SP)
0x0057 00087 (main.go:44)	LEAQ	type.int(SB), CX
0x005e 00094 (main.go:44)	MOVQ	CX, "".ti+72(SP)
0x0063 00099 (main.go:44)	MOVQ	AX, "".ti+80(SP)
```

这段汇编有下面这些特点：

* CALL runtime.convT64(SB)：将 100 作为 runtime.convT64 的参数，该函数申请了一段内存，将 100 放入了这段内存里
* 将类型 type.int 放入到 SP+72 的位置
* 将包含 100 的那块内存的指针，放入到 SP + 80 的位置

这段汇编从直观上来说，interface 转换成 eface 是看不出来的。这个如何观察呢？这个就需要借助 gdb 了。

![](https://cdn.jsdelivr.net/gh/georgehao/img/eface_gdb.png)

再继续深究下，如何利用内存分布来验证是 eface 呢？需要另外再添加点代码。

```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}

type _type struct {
    size       uintptr
    ptrdata    uintptr // size of memory prefix holding all pointers
    hash       uint32
    tflag      tflag
    align      uint8
    fieldAlign uint8
    kind       uint8
    equal      func(unsafe.Pointer, unsafe.Pointer) bool
    gcdata     *byte
    str        nameOff
    ptrToThis  typeOff
}

func main() {
    var ti interface{}
    var a int = 100
    ti = a

    fmt.Println("type:", *(*eface)(unsafe.Pointer(&ti))._type)
    fmt.Println("data:", *(*int)((*eface)(unsafe.Pointer(&ti)).data))
    fmt.Println((*eface)(unsafe.Pointer(&ti)))
}
```

output:
```
type: {8 0 4149441018 15 8 8 2 0x10032e0 0x10e6b60 959 27232}
data: 100
&{0x10ade20 0x1155bc0}
```
从这个结果上能够看出来 
* eface.kind = 2, 对应着 runtime.kindInt
* eface.data = 100

从内存上分配上看，我们基本看出来了 eface 的内存布局及对应的最终的 eface 的类型转换结果。

## iface

```go
package main

type Person interface {
	  Say() string
}

type Man struct {
}

func (m *Man) Say() string {
	  return "Man"
}

func main() {
    var p Person

    m := &Man{}
    p = m
    println(p.Say())
}

```

iface 我们也看下汇编：

```s
0x0029 00041 (main.go:24)	LEAQ	runtime.zerobase(SB), AX
0x0030 00048 (main.go:24)	MOVQ	AX, ""..autotmp_6+48(SP)
0x0035 00053 (main.go:24)	MOVQ	AX, "".m+32(SP)
0x003a 00058 (main.go:25)	MOVQ	AX, ""..autotmp_3+64(SP)
0x003f 00063 (main.go:25)	LEAQ	go.itab.*"".Man,"".Person(SB), CX
0x0046 00070 (main.go:25)	MOVQ	CX, "".p+72(SP)
0x004b 00075 (main.go:25)	MOVQ	AX, "".p+80(SP)
```

这段汇编上，能够看出来是有 itab 的，但是是否真的是转成了 iface，汇编上仍然反应不出来。

同样，我们继续用 gdb 查看 Person interface 确实被转换成了 iface。
![](https://cdn.jsdelivr.net/gh/georgehao/img/iface_gdb.png)

关于 iface 内存布局，我们仍然加点代码来查看

```go
type itab struct {
    inter *interfacetype
    _type *_type
    hash  uint32
    _     [4]byte
    fun   [1]uintptr
}

type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type Person interface {
    Say() string
}

type Man struct {
    Name string
}

func (m *Man) Say() string {
    return "Man"
}

func main() {
    var p Person

    m := &Man{Name: "hhf"}
    p = m
    println(p.Say())

    fmt.Println("itab:", *(*iface)(unsafe.Pointer(&p)).tab)
    fmt.Println("data:", *(*Man)((*iface)(unsafe.Pointer(&p)).data))
}
```
output:
```
Man
itab: {0x10b3ba0 0x10b1900 1224794265 [0 0 0 0] [17445152]}
data: {hhf}
```

关于想继续探究 eface, iface 的内存布局的同学，可以基于上面的代码，利用 unsafe 的相关函数去看对应的内存位置上的值。

### 类型断言

```go
type Person interface {
	  Say() string
}

type Man struct {
	  Name string
}

func (m *Man) Say() string {
	  return "Man"
}

func main() {
	  var p Person

    m := &Man{Name: "hhf"}
    p = m

    if m1, ok := p.(*Man); ok {
      fmt.Println(m1.Name)
    }
}
```

我们仅关注类型断言那块内容，贴出对应的汇编

```s
0x0087 00135 (main.go:23)	MOVQ	"".p+104(SP), AX
0x008c 00140 (main.go:23)	MOVQ	"".p+112(SP), CX
0x0091 00145 (main.go:23)	LEAQ	go.itab.*"".Man,"".Person(SB), DX
0x0098 00152 (main.go:23)	CMPQ	DX, AX
```
能够看出来的是：将 iface.itab 放入了 AX，将 `go.itab.*"".Man,"".Person(SB)` 放入了 DX，比较两者是否相等，来判断 Person 的真实类型是否是 Man。

另外一个类型断言的方式就是 switch 了，其实两者本质上没啥区别。

### 坑

interface 最著名的坑的，应该就是下面这个了。

```go
func main() {
    var a interface{} = nil
    var b *int = nil
    
    isNil(a)
    isNil(b)
}

func isNil(x interface{}) {
    if x == nil {
      fmt.Println("empty interface")
      return
    }
    fmt.Println("non-empty interface")
}
```

output:

```
empty interface
non-empty interface
```

为什么会这样呢？这就涉及到 interface == nil 的判断方式了。一般情况只有 eface 的 type 和 data 都为 nil 时，interface == nil 才是 true。

当我们把 b 复制给 interface 时，x.\_type.Kind = kindPtr。虽说 x.data = nil，但是不符合 interface == nil 的判断条件了。

### 关于 interface 源码阅读的一点建议

关于 interface 源码阅读的一点建议，如果想利用汇编看源码的话，尽量选择 go1.14.x。

选择 Go 汇编来看 interface，基本上也是为了查看 interface 最终被转换成 eface 还是 iface，调用了 runtime 的哪些函数，以及对应的函数栈分布。如果 Go 版本选择的太高的话，go 汇编变化太大了，可能汇编上就看不到对应的内容了。

更多学习学习资料分享，关注公众号回复指令：

* 回复 0，获取 《Go 面经》
* 回复 1，获取 《Go 源码流程图》

![](https://cdn.jsdelivr.net/gh/georgehao/img/me.png)