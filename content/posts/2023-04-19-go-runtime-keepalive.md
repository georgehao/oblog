---
title: "Go runtime.KeepAlive"
date: 2023-04-18T21:27:21+08:00
authors: hhf
tags: [Go] 
categories: [Go]
toc: true
---

在 Go 编程中，你是否注意过临时变量是什么时候被回收的？比如下面这段程序, 变量 a 会在什么时间被 GC 回收掉的？

```go
func f2() {
    // do something
}

func f1() int {
    var a int = 10
    a += f2()
    b := a
    
    // do something
    return b
}
```

* A. f1() 函数结束后的 GC 周期内
* B. 当 a 复制给 b 后的 GC 周期内

很多同学稍微不注意，会觉得是 A 是正确的。其实正确答案应该是 B，因为 a 复制给 b 后，a 已经没有被引用了，如果 GC 发生时 a 变量是白色对象，就会被清除掉。

## runtime.SetFinalizer 会导致的一些问题

有些同学喜欢利用 runtime.SetFinalizer 模拟析构函数，当变量被回收时，执行一些回收操作，加速一些资源的释放。在做性能优化的时候这样做确实有一定的效果，不过这样做是有一定的风险的。

比如下面这段代码，初始化一个文件描述符，当 GC 发生时释放掉无效的文件描述符。

```go

type File struct { d int }

func main() {
    p := openFile("t.txt")
    content := readFile(p.d)

    println("Here is the content: "+content)
}

func openFile(path string) *File {
    d, err := syscall.Open(path, syscall.O_RDONLY, 0)
    if err != nil {
      panic(err)
    }

    p := &File{d}
    runtime.SetFinalizer(p, func(p *File) {
      syscall.Close(p.d)
    })

    return p
}

func readFile(descriptor int) string {
    doSomeAllocation()

    var buf [1000]byte
    _, err := syscall.Read(descriptor, buf[:])
    if err != nil {
      panic(err)
    }

    return string(buf[:])
}

func doSomeAllocation() {
    var a *int

    // memory increase to force the GC
    for i:= 0; i < 10000000; i++ {
      i := 1
      a = &i
    }

    _ = a
}
```

上面这段代码是对 [go 官方文档](https://pkg.go.dev/runtime#KeepAlive "go 官方文档")的一个延伸，doSomeAllocation 会强制执行 GC，当我们执行这段代码时会出现下面的错误。

```
panic: no such file or directory

goroutine 1 [running]:
main.openFile(0x107a65e, 0x5, 0x10d9220)
        main.go:20 +0xe5
main.main()
        main.go:11 +0x3a
```

这是因为 syscall.Open 产生的文件描述符比较特殊，是个 int 类型，当以值拷贝的方式在函数间传递时，并不会让 File.d 产生引用关系，于是 GC 发生时就会调用 `runtime.SetFinalizer(p, func(p *File)` 导致文件描述符被 close 掉。

## 什么是 runtime.KeepAlive ?

如上面的例子，我们如果才能让文件描述符不被 gc 给释放掉呢？其实很简单，只需要调用 runtime.KeepAlive 即可。

```go
func main() {
    p := openFile("t.txt")
    content := readFile(p.d)
    
    runtime.KeepAlive(p)

    println("Here is the content: "+content)
}
```

runtime.KeepAlive 能阻止 runtime.SetFinalizer 延迟发生，保证我们的变量不被 GC 所回收。

## 结论

正常情况下，runtime.KeepAlive，runtime.SetFinalizer 不应该被滥用，当我们真的需要使用时候，要注意使用是否合理。

《性能优化 | Go Ballast 让内存控制更加丝滑》我们说到如何让整个程序的声明周期内维护一个 slice 不被 gc 给回收掉，这里就用到了 runtime.KeepAlive。

这里还有有一篇关于 runtime.KeepAlive 的[文档](https://medium.com/a-journey-with-go/go-keeping-a-variable-alive-c28e3633673a "文档")，有兴趣的可以看一下。