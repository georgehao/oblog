---
title: golang中defer, panic, recover用法
author: haohongfan
type: post
date: 2017-07-20T09:49:28+00:00
categories:
  - golang
---
昨天谢大在群里发了一个[golang面试题](https://zhuanlan.zhihu.com/p/26972862), 第一题就不会做了. 这题主要是考察`defer`, `panic`, 于是各种谷歌, 就写下了这篇文章, 由于本人水平有限, 有哪些理解不到的地方, 请在下面留言指出

## 一. defer 用法
为何会有defer这样的语法呢? 如果你之前是写C++的话这样的代码, 你会经常看到.
```
class Demo {
public:
	Demo() {
		p = new int(10);
	}
	
	~Demo() {
		if (p) { delete(p); }
	}
private:
	int *p = nullptr;
}
```
本来就是想要简单使用某个变量(比如, `new出来的变量`,`文件句柄`, `mutex`等等), 如果程序写的简单的话, 我们一般都会记得去释放这些变量, 但是程序会越写越复杂, 再加上各种函数之间的传递, 如果稍微不注意去释放这些变量, 内存泄漏就出来了(一般c/c++的BUG都是由这个引起的). 这个时候我们就会利用`c++`的析构函数, 去自动释放这些变量.  在`c++11`, `boost`专门为了这玩意制定出了`智能指针`, 说实话, 这玩意真心么有那么好用. 

在初接触到`go`时, 就被`defer`吸引住了, 要是`c++`也能这么写, 那就太爽了!

**`defer`的特性: 在函数返回之前, 调用defer函数的操作, 简化函数的清理工作.** 

使用`defer`关键字的时候, 有下面这些注意点:

**1. 在`defer`表达式确定的时候, `defer`修饰的函数(后面统称为defered函数)的参数也就确定了**
```
func argument() {
	i := 0
	defer fmt.Println(i)
	i++
	return
}
```

**2. 函数内可以有多个`defered`函数, 但是这些defered函数在函数返回时遵守`后进先出`的原则**
```
func LIFO() {
	for i := 0; i<4; i++ {
		defer fmt.Print(i)
	}
}
```
**3. 函数命名的返回值跟`defered`函数一起使用**
函数的返回值有可能被`defer`更改, 本质原因是`return xxx`语句并不是一条原子指令
```
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```
```
func g() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```
```
func h() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```
对于`defered`函数跟函数命名返回值一块使用的情况, 当无法判断返回值的时候, 需要对函数进行变形.
```
func f(result int) {
	result = 0
	func () {
		result++
	}()
	return
} 
```
结果: 1

```
func g() (r int) {
	t := 5
	r = t
	func () {
		t = t + 5
	}
	return
}
```
结果: 5

```
func h() (r int) {
	r = 1
	func (r int) {
	 	r = r + 5
	}(r)
	return
}
```
结果: 1, 在`func(r int) {...}`中, 由于`r`是以`值传递`的方式进行的, 所以r的值不会改变.

defer涉及到所有的代码, [点击这里查看](https://play.golang.org/p/VxSxBWyYnm)

关于`defer`实现原理, 留到后面出个专题

## 二. panic用法
> Panic is a built-in function that stops the ordinary flow of control and begins panicking. When the function F calls panic, execution of F stops, any deferred functions in F are executed normally, and then F returns to its caller. To the caller, F then behaves like a call to panic. The process continues up the stack until all functions in the current goroutine have returned, at which point the program crashes. Panics can be initiated by invoking panic directly. They can also be caused by runtime errors, such as out-of-bounds array accesses.

`panic`用法挺简单的, 上面这段引用是[golang](https://blog.golang.org/defer-panic-and-recover)的官方说法. `panic`其实就是`c++`中的`throw exception`

`panic `是内建函数.`panic`会中断函数`F`的正常执行流程, 从`F`函数中跳出来, 跳回到`F`函数的调用者. 对于调用者来说, `F`看起来就是一个`panic`, 所以调用者会继续向上跳出, 直到当前`goroutine`返回. 在跳出的过程中, 进程会保持这个函数栈. 当`goroutine`退出时, 程序会`crash`.

要注意的是, `F`函数中的`defered`函数会正常执行, 按照上面`defer`的规则.

同时引起`panic`除了我们主动调用`panic`之外, 其他的任何运行时错误, 例如数组越界都会造成`panic`

看下面一个例子
```
package main

import (
	"fmt"
)

func main() {
	defer_call()
}

func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()
	panic("触发异常")
	fmt.Println("test")
}
```
>结果:
```
打印后
打印中
打印前
panic: 触发异常
goroutine 1 [running]:
main.defer_call()
	/Users/wuling/go/src/github.com/georgehao/test/panic.go:15 +0xc0
main.main()
	/Users/wuling/go/src/github.com/georgehao/test/panic.go:8 +0x20
```

由这个例子, 我们看到程序没有打印`test`, 这就说明触发`panic`的函数`defer_call`从`panic("触发异常")`跳了出来. 先执行`defered`函数, 然后抛出`panic`异常信息. `defered`还是按照`FILO`的规则调用.  (使用`Gogland` IDE的同学要注意了, `Gogland`返回的`panic`异常信息跟`defered`函数的打印是无顺序的, 可能是`Gogland`的BUG)

下面对上面的例子做下稍微改动
```
package main

import (
	"fmt"
)

func main() {
	go defer_call()
}

func defer_call() {
	defer func() { fmt.Println("打印前") }()
	defer func() { fmt.Println("打印中") }()
	defer func() { fmt.Println("打印后") }()
	panic("触发异常")
	fmt.Println("test")
}
```
我们看程序结果: 没有任何输出, 而且程序也没有`crash`, 那是不是我们说了那么多, 是错了呢. 不, 我们再仔细想想, 是这为什么呢? 

是由于`main``goroutine`没有等`defer_call goroutine`返回就程序就结束了,程序当然不会`crash`. 在`go defer_call()`下再加一行`time.Sleep(time.Minute`)是不是效果就不一样了. 

其实想说的不是这个, 想说的是`panic`只会让当前的`goroutine`返回. 如果当前的`goroutine`没有去捕获这个`panic`的话, 那么程序就会`crash`.

## 三. recover 用法
>Recover is a built-in function that regains control of a panicking goroutine. Recover is only useful inside deferred functions. During normal execution, a call to recover will return nil and have no other effect. If the current goroutine is panicking, a call to recover will capture the value given to panic and resume normal execution.

`recover`也是一个内建函数. `recover`就是`c++`中的`catch`. 

不过需要注意的是:
1. `recover`如果想起作用的话, 必须在`defered`函数中使用.
2. 在正常函数执行过程中, 调用`recover`没有任何作用, 他会返回`nil`. 如这样:`fmt.Println(recover()) // nil`
3. 如果当前的`goroutine` `panic`了, 那么`recover`将会捕获这个`panic`的值, 并且让程序正常执行下去, 不会让程序`crash`.

[举个栗子](https://play.golang.org/p/F5Kfl9TvaW):
```
package main

import "fmt"

func recoverPanic() {
	func () {
		if r := recover(); r != nil {
			fmt.Println("recover value is", r)
		}
	}()

	panic("exception")
}

func main()  {
	recoverPanic()
}
```
这段代码有什么问题吗?
其实要是这么写的, `recover`没有任何作用, 因为`recover`必须在`defered`函数中才有作用.
[点击这里查看正确的代码](https://play.golang.org/p/1n2PBJOKEt)

## 四. 综合的例子

官方的一个例子, 基本就是`defer`, `panic`, `recover`的用法了

```
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```

>程序输出:
```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Recovered in f 4
Returned normally from f.
```


**这里有个问题, 为什么` fmt.Println("Returned normally from g.")`没有打印, 而`fmt.Println("Returned normally from f.")`打印了呢?**

先看下面这个例子, 只是对上面的例子, 移动了`defered recover`函数的位置:
```
package main

import "fmt"

func main() {
    f()
    fmt.Println("Returned normally from f.")
}

func f() {
    fmt.Println("Calling g.")
    g(0)
    fmt.Println("Returned normally from g.")
}

func g(i int) {
	defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered in f", r)
        }
    }()
    if i > 3 {
        fmt.Println("Panicking!")
        panic(fmt.Sprintf("%v", i))
    }
    defer fmt.Println("Defer in g", i)
    fmt.Println("Printing in g", i)
    g(i + 1)
}
```
是不是输出结果不一样了.
```
Calling g.
Printing in g 0
Printing in g 1
Printing in g 2
Printing in g 3
Panicking!
Recovered in f 4
Defer in g 3
Defer in g 2
Defer in g 1
Defer in g 0
Returned normally from g.
Returned normally from f.
```

由此, 我们可以看出, 在当前`goroutine`的中, `recover`会捕获`recover`所在的函数产生的的`panic`, 由于`panic`会让当前函数返回, 但是对于其调用者来说, 这个`panic`已经不存在了, 所以程序还是会按照正常的执行流程执行下去. 所以这个例子会打印出来` fmt.Println("Returned normally from g.")`, 而上面的那个却不会.

## 参考链接

1. [golang面试题](https://zhuanlan.zhihu.com/p/26972862)
2. [Defer, Panic, and Recover](https://blog.golang.org/defer-panic-and-recover)'
3. [defer关键字](https://tiancaiamao.gitbooks.io/go-internals/content/zh/03.4.html)
4. [Golang中defer、return、返回值之间执行顺序的坑](https://my.oschina.net/henrylee2cn/blog/505535)
5. [build-web-application-with-golang](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/02.3.md)