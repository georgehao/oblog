---
title: golang遇到的问题总结
author: haohongfan
type: post
date: 2017-07-02T03:41:10+00:00
categories:
  - golang
draft: true
---

### 问题1
1. golang中开辟的内存在栈上还是堆上?
2. golang中临时变量的返回

摘抄[深入解析go: escape analyze](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.6.html)的一段话:

```
func f() *Cursor {
    var c Cursor
    c.X = 500
    noinline()
    return &c
}
```
Cursor是一个结构体，这种写法在C语言中是不允许的，因为变量c是在栈上分配的，当函数f返回后c的空间就失效了。但是，在Go语言规范中有说明，这种写法在Go语言中合法的。语言会自动地识别出这种情况并在堆上分配c的内存，而不是函数f的栈上。

识别出变量需要在堆上分配，是由编译器的一种叫escape analyze的技术实现的。如果输入命令
`go build --gcflags=-m cluster.go`

可以看到输出：
> 
cluster.go:13, &c escapes to heap
cluster.go:11, moved to heap: c

表示c逃逸了，被移到堆中。escape analyze可以分析出变量的作用范围，这是对垃圾回收很重要的一项技术

更多资料:
1. [Go的变量到底在堆还是栈中分配](http://www.haohongfan.com/index.php/2017/07/02/golang_questions/)
3. [Golang escape analysis](http://www.agardner.me/golang/garbage/collection/gc/escape/analysis/2015/10/18/go-escape-analysis.html?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
4. [How do I know whether a variable is allocated on the heap or the stack?](https://golang.org/doc/faq#stack_or_heap)

综上来看, 其实写`golang`不需要像写`c++`一样关注这些变量内存的开辟位置.

### 问题2 golang闭包和goroutine一起使用的坑
一道面试题, 结果是多少?
```
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func() {
			fmt.Println("i: ", i)
			wg.Done()
		}()
	}
}
```
1. [Golang 中关于闭包的坑](http://www.jianshu.com/p/fa21e6fada70), 这个文章基本总结了所有的坑, 值得一看
2. [A Go Gotcha: When Closures and Goroutines Collide](https://blog.cloudflare.com/a-go-gotcha-when-closures-and-goroutines-collide/) 这个文章解释了, 这个面试题为啥结果输出是那样的.

对于上面的面试题的解决办法:
```
func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}
	wg.Add(10)
	for i := 0; i < 10; i++ {
		go func(i int) {
			fmt.Println("i: ", i)
			wg.Done()
		}(i)
	}
}
```

#### 4. golang中 struct tag用法

#### 5. golang panic recover用法

#### 6. golang中常用的字符串操作
1. [golang中常用的字符串操作](http://www.golangprograms.com/golang/string-functions/)