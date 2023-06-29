---
title: "你真的懂 golang reslice 吗"
date: 2020-10-20T10:17:05+08:00
lastmod: 2020-10-20T10:17:05+08:00
keywords: [golang, slice]
tags: [golang, slice]
categories: [golang]
---

```go
package main

func a() []int {
	a1 := []int{3}
	a2 := a1[1:]
	return a2
}

func main() {
	a()
}
```

看到这个题, 你的第一反应是啥? 

```
(A) 编译失败
(B) panic: runtime error: index out of range [1] with length 1
(C) []
(D) 其他
```

第一感觉: 肯定能编译过, 但是运行时一定会panic的. 但事与愿违竟然能够正常运行, **结果是:[]**

## 疑问

```golang
a1 := []int{3}
a2 := a1[1:]
fmt.Println("a[1:]", a2)
```

a1 和 a2 共享同样的底层数组, len(a1) = 1, a1[1]绝对会panic, 但是a[1:]却能正常输出, 这是为何?

## 从表面入手

整体上看下整体的情况

```golang
a1 := []int{3}
fmt.Printf("len:%d, cap:%d", len(a1), cap(a1))
fmt.Println("a[0:]", a1[0:])
fmt.Println("a[1:]", a1[1:])
fmt.Println("a[2:]", a1[2:])
```

结果:

```
len:1, cap:1
a[0:]: [1]
a[1:] []
panic: runtime error: slice bounds out of range [2:1]
```

从表面来看, 从a[2:]才开始panic, 到底是谁一手造成这样的结果呢?

## 汇编上看一目了然

```asm
"".a STEXT size=87 args=0x18 locals=0x18
	// 省略...
	0x0028 00040 (main.go:6)	CALL	runtime.newobject(SB)
	0x002d 00045 (main.go:6)	MOVQ	8(SP), AX  // 将slice的数据首地址加载到AX寄存器
	0x0032 00050 (main.go:6)	MOVQ	$3, (AX)   // 把3放入到AX寄存器中, 也就是a1[0]
	0x0039 00057 (main.go:8)	MOVQ	AX, "".~r0+32(SP)
	0x003e 00062 (main.go:8)	XORPS	X0, X0     // 初始化 X0 寄存器
	0x0041 00065 (main.go:8)	MOVUPS	X0, "".~r0+40(SP) // 将X0放入返回值
	0x0046 00070 (main.go:8)	MOVQ	16(SP), BP
	0x004b 00075 (main.go:8)	ADDQ	$24, SP
	0x004f 00079 (main.go:8)	RET
	// 省略....
```

其实主要关心这两行即可. 

```
0x003e 00062 (main.go:8)	XORPS	X0, X0     // 初始化 X0 寄存器
0x0041 00065 (main.go:8)	MOVUPS	X0, "".~r0+40(SP) // 将X0放入返回值
```

是不是很神奇, a[1:] 没有调用`runtime.panicSliceB(SB)`, 而是返回的是一个空的slice. 这是为何呢? 

持着怀疑态度, 去 github 提上一枚 issue. https://github.com/golang/go/issues/42069 

![reslice](https://images.haohongfan.com/reslice.png)

结论: 这是故意的, 单纯为了保持reslice对称而已. 这也就解释了返回一个空的slice的原因. 

## reslice 原理

上面的问题已经解释清楚了, 回过头来看正常 reslice 的例子

```go
func a() []int {
	a1 := []int{3, 4, 5, 6, 7, 8}
	a2 := a1[2:]
	return a2
}
```

用简单的图来描述这段代码里, a1 和 a2 之间的reslice 关系. 可以看到 a1 和 a2 是共享底层数组的.

![reslice](https://images.haohongfan.com/reslice-arry1.png)


如果你知道这些, 那么 slice 的使用基本上不会出现问题. 

下面这些问题你考虑过吗 ?

1. a1, a2 是如何共享底层数组的?
2. a1[low:high]是如何实现的?

继续来看这段代码的汇编:

```asm
"".a STEXT size=117 args=0x18 locals=0x18
	// 省略...
	0x0028 00040 (main.go:4)	CALL	runtime.newobject(SB)
	0x002d 00045 (main.go:4)	MOVQ	8(SP), AX
	0x0032 00050 (main.go:4)	MOVQ	$3, (AX)
	0x0039 00057 (main.go:4)	MOVQ	$4, 8(AX)
	0x0041 00065 (main.go:4)	MOVQ	$5, 16(AX)
	0x0049 00073 (main.go:4)	MOVQ	$6, 24(AX)
	0x0051 00081 (main.go:4)	MOVQ	$7, 32(AX)
	0x0059 00089 (main.go:4)	MOVQ	$8, 40(AX)
	0x0061 00097 (main.go:5)	ADDQ	$16, AX
	0x0065 00101 (main.go:6)	MOVQ	AX, "".~r0+32(SP)
	0x006a 00106 (main.go:6)	MOVQ	$4, "".~r0+40(SP)
	0x0073 00115 (main.go:6)	MOVQ	$4, "".~r0+48(SP)
	0x007c 00124 (main.go:6)	MOVQ	16(SP), BP
	0x0081 00129 (main.go:6)	ADDQ	$24, SP
	0x0085 00133 (main.go:6)	RET
	// 省略....
```
* 第4行: 将 AX 栈顶指针下移 8 字节, 指向了 a1 的 data 指向的地址空间里
* 第5-10行: 将 [3,4,5,6,7,8] 放入到 a1 的 data 指向的地址空间里
* 第11行: AX 指针后移 16 个字节. 也就是指向元素 5 的位置
* 第12行: 将 SP 指针下移 32 字节指向即将返回的 slice (其实就是 a2 ), 同时将 AX 放入到 SP. 注意 AX 放入 SP 里的是一个指针, 也就造成了a1, a2是共享同一块内存空间的
* 第13行: 将 SP 指针下移 40 字节指向了 a2 的 len,  同时 把 4 放入到 SP, 也就是 len(a2) = 4 
* 第14行: 将 SP 指针下移 48 字节指向了 a2 的 cap,  同时 把 4 放入到 SP, 也就是 cap(a2) = 4 

下图是 slice 的 栈图, 可以配合着上面的汇编一块看.

![slice stack](https://images.haohongfan.com/slice_stack1.png?imageView2/2/h/500)

看到这里是不是一目了然了. 于是有了下面的这些结论:

1. reslice 完全是利用汇编实现的
2. reslice 时, slice 的 data 通过指针的移动完成, 造成了共享相同的底层数据, 同时将新的 len, cap 放入对应的位置

至此, golang reslice的原理基本已经阐述清楚了. 

## 参考资料

1. [深入Go的底层，带你走近一群有追求的人](https://qcrao.com/2019/03/20/dive-into-go-asm/)
2. [汇编角度看 Slice，一个新的世界](https://qcrao.com/2019/04/02/dive-into-go-slice/)
3. [Why slice not painc](https://github.com/golang/go/issues/42069)
4. [Slice expressions](https://golang.org/ref/spec#Slice_expressions)
5. [A Quick Guide to Go's Assembler](https://golang.org/doc/asm)
6. [plan9 assembly 完全解析](https://github.com/cch123/golang-notes/blob/master/assembly.md)