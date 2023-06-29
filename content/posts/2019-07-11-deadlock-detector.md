---
title: "一次golang deadlock的讨论"
date: 2019-07-11T10:44:07+08:00
lastmod: 2019-07-11T10:44:07+08:00
keywords: [golang, deadlock]
tags: [golang]
categories: [golang]
---

## 背景

在微信群一位同学抛出的一段代码, 各位看官猜想一下程序的执行结果

```go
// 程序1
func main() {
	fmt.Println("running, not deadlock")
	server, err := net.Listen("tcp", "127.0.0.1:9001")
	if err != nil {
		fmt.Println(err)
	}

	waitQueue := make(chan int)
	for {
		connection, err := server.Accept()
		if err != nil {
			panic("server")
		}
		fmt.Printf("Received connection from %s.\n", connection.RemoteAddr())
		waitQueue <- 1
	}
}
```

我猜想大部分同学都会说是: `fatal error: all goroutines are asleep - deadlock!`. 因为`waitQueue`是个没有缓冲的channel, `waitQueue <- 1`向里面send一个值, 理论上程序一运行就会报deadlock的错误

如下面这个例子

```go
// 程序1
func main() {
	waitQueue := make(chan int)
	for {
		waitQueue <- 1
	}
}
```

这个程序的结果毫无疑问是: `fatal error: all goroutines are asleep - deadlock!`

但是文章一开头的这个程序, 同学们在自己机器上运行一下, 我想99%的人应该得到的是下面的结果: `running, not deadlock`

我甚至把程序写的更极端些

```go
// 程序3
func main() {
    fmt.Println("running, not deadlock")
	waitQueue := make(chan int)
	waitQueue <- 1
	return

	server, err := net.Listen("tcp", "127.0.0.1:9001")
	if err != nil {
		fmt.Println(err)
	}

	for {
		connection, err := server.Accept()
		if err != nil {
			panic("server")
		}
		fmt.Printf("Received connection from %s.\n", connection.RemoteAddr())
	}
}
```

程序3和程序1不同的地方是: 我把整个channel放在了程序开始, 甚至直接`return`, 依然不会出现`deadlock`

**是不是陷入了深深的怀疑之中?**

我分别在`macOS`, `linux`, `ubuntu`上面验证了`1.10`, `1.12.2`, `1.12.5`, `1.12.6`都不会出现`deadlock`

于是我在golang的GitHub上提了这个issue, [bcmills](https://github.com/bcmills)很快回复了为什么不会出现`deadlock`的提示. 他的大概意思是golang built-in deadlock detector 在某些情况下是被禁用, 如通过`C`库进行系统调用. deadlock detector触发依赖于goroutine的调度和系统调用的具体实现. `deadlock detector`是一个有用的工具, 但是不能完全替代集成测试和负载测试

![deadlock2](http://images.haohongfan.com/deadlock2.png)

![deadlock1](http://images.haohongfan.com/deadlock1.png)

![deadlock](http://images.haohongfan.com/deadlock.png)

具体讨论内容可以查看: https://github.com/golang/go/issues/33004

也不是说所有的环境下都不会出现`deadlock`, 至少在[golang playground](https://play.golang.org/p/SBP_ccSoTlX)就出现了

![deadlock](http://images.haohongfan.com/deadlock4.png)

具体在往深处我就没有去追究built-in deadlock detector的实现了, 等后面再补充这个. 不过从这里也可以看出来, 我们不能还是要写好单元测试, 集成测试等避免`goroutine leaks`出现, 不能过分依赖与`deadlock detector`
