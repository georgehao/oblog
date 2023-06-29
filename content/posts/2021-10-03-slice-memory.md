---
title: "透过内存看 Slice 和 Array的异同"
date: 2021-10-03T21:27:21+08:00
authors: hhf
tags: [Go, golang]
categories: [Go,golang]
toc: true
---

hi, 大家好，我是 hhf。

有这么一个 Go 面试题：请说出 slice 和 array 的区别？

这简直就是送分题。现在思考一下，你咋样回答才能让面试官满意呢？

我这里就不贴这道题的答案了。但是我想内存方面简单分析下 slice 和 array 的区别。

## Array 

```go
func main() {
  as := [4]int{10, 5, 8, 7}
  
  fmt.Println("as[0]：", as[0])
  fmt.Println("as[1]：", as[1])
  fmt.Println("as[2]：", as[2])
  fmt.Println("as[3]：", as[3])
}
```

这段很简单的代码，声明了一个 array。当然输出结果也足够简单。 

我们现在玩点花活，如何通过非正常的手段访问数组里面的元素呢？在做这个事情之前是需要先知道 array 的底层结构的。其实很简单，Go array 就是一块连续的内存空间。如下图所示

![](https://cdn.jsdelivr.net/gh/georgehao/img/array.png)


写一段简单的代码，我们不通过下标访问的方式去获取元素。通过移动指针的方式去获取对应位置的指针。

```go
func main() {
    as := [4]int{10, 5, 8, 7}

    p1 := *(*int)(unsafe.Pointer(&as))
    fmt.Println("as[0]:", p1)

    p2 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])))
    fmt.Println("as[1]:", p2)

    p3 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])*2))
    fmt.Println("as[2]:", p3)

    p4 := *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + unsafe.Sizeof(as[0])*3))
    fmt.Println("as[3]:", p4)
}
```

结果：

```
as[0]: 10
as[1]: 5
as[2]: 8
as[3]: 7
```

下图演示下获取对应位置的值的过程：

![](https://cdn.jsdelivr.net/gh/georgehao/img/go_array3.gif)

## Slice 

同样对于 slice 这段简单的代码：

```go
func main() {
  as := []int{10, 5, 8, 7}
  
  fmt.Println("as[0]：", as[0])
  fmt.Println("as[1]：", as[1])
  fmt.Println("as[2]：", as[2])
  fmt.Println("as[3]：", as[3])
}
```

想要通过移动指针的方式获取 slice 对应位置的值，仍然需要知道 slice 的底层结构。如图：
![](https://cdn.jsdelivr.net/gh/georgehao/img/go_slice1.png)


```go
func main() {
    as := []int{10, 5, 8, 7}

    p := *(*unsafe.Pointer)(unsafe.Pointer(&as))
    fmt.Println("as[0]:", *(*int)(unsafe.Pointer(uintptr(p))))
    fmt.Println("as[1]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0]))))
    fmt.Println("as[2]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0])*2)))
    fmt.Println("as[3]:", *(*int)(unsafe.Pointer(uintptr(p) + unsafe.Sizeof(&as[0])*3)))

    var Len = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + uintptr(8)))
    fmt.Println("len", Len) 

    var Cap = *(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(&as)) + uintptr(16)))
    fmt.Println("cap", Cap) 
}
```

结果：
```
as[0]: 10
as[1]: 5
as[2]: 8
as[3]: 7
len 4
cap 4
```

用指针取 slice 的底层 Data 里面的元素跟 array 稍微有点不同：

* 对 slice 变量 as 取地址后，拿到的是 SiceHeader 的地址，对这个指针进行移动，得到是 slice 的 Data, Len, Cap。
* 所以当拿到 Data 的值时，我们拿到的是 Data 所指向的 array 的首地址的值。
* 由于这个值是个指针，需要对这个值 *Data， 取到 array 真正的首地址的指针值
* 然后对这个值 &(*Data)，获取到真正的首地址，然后对这个值进行指针的移动，才能获取到 slice 的数组里的值


获取 slice cap 和 len:

![](https://cdn.jsdelivr.net/gh/georgehao/img/go_slice_caplen.gif)

获取 slice 的 Data：

![](https://cdn.jsdelivr.net/gh/georgehao/img/go_slice_array.gif)

欢迎关注公众号。更多学习学习资料分享，关注公众号回复指令：

- 回复 0，获取 《Go 面经》
- 回复 1，获取 《Go 源码流程图》

![](https://cdn.jsdelivr.net/gh/georgehao/img/me.png)