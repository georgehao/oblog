---
title: "最清晰易懂的 Go WaitGroup 源码剖析"
date: 2021-04-01T09:57:16+08:00
lastmod: 2021-04-01T09:57:16+08:00
authors: hhf
tags: [Go, golang]
categories: [Go,golang]
series: Go源码分析与实战
toc: true
---

hi，大家好，我是haohongfan。


本篇主要介绍 WaitGroup 的一些特性，让我们从本质上去了解 WaitGroup。关于 WaitGroup 的基本用法这里就不做过多介绍了。相对于《这可能是最容易理解的 Go Mutex 源码剖析》来说，WaitGroup 就简单的太多了。
## 源码剖析
![add](https://images.haohongfan.com/waitgroup_add.png)

![wait](https://images.haohongfan.com/waitgroup_wait.png)

```go
type WaitGroup struct {
	noCopy noCopy
	state1 [3]uint32
}
```
WaitGroup 底层结构看起来简单，但 WaitGroup.state1 其实代表三个字段：counter，waiter，sema。


counter ：可以理解为一个计数器，计算经过 wg.Add(N), wg.Done() 后的值。
waiter ：当前等待 WaitGroup 任务结束的等待者数量。其实就是调用 wg.Wait() 的次数，所以通常这个值是 1 。
sema ： 信号量，用来唤醒 Wait() 函数。


### 为什么要将 counter 和 waiter 放在一起 ？
其实是为了保证 WaitGroup 状态的完整性。举个例子，看下面的一段源码
```go
// sync/waitgroup.go:L79 --> Add()
if v > 0 || w == 0 { // v => counter, w => waiter
    return
}
// ...
*statep = 0
for ; w != 0; w-- {
    runtime_Semrelease(semap, false, 0)
}
```
当同时发现 wg.counter <= 0 && wg.waiter != 0 时，才会去唤醒等待的 waiters，让等待的协程继续运行。但是使用 WaitGroup 的调用方一般都是并发操作，如果不同时获取的 counter 和 waiter 的话，就会造成获取到的 counter 和 waiter 可能不匹配，造成程序 deadlock 或者程序提前结束等待。
### 如何获取 counter 和 waiter ?
对于 wg.state 的状态变更，WaitGroup 的 Add()，Wait() 是使用 atomic 来做原子计算的(为了避免锁竞争)。但是由于 atomic 需要使用者保证其 64 位对齐，所以将 counter 和 waiter 都设置成 uint32，同时作为一个变量，即满足了 atomic 的要求，同时也保证了获取 waiter 和 counter 的状态完整性。但这也就导致了 32位，64位机器上获取 state 的方式并不相同。如下图：
![waitgroup state](https://images.haohongfan.com/waitgroup_state1.png)
简单解释下：


因为 64 位机器上本身就能保证 64 位对齐，所以按照 64 位对齐来取数据，拿到 state1[0], state1[1] 本身就是64 位对齐的。但是 32 位机器上并不能保证 64 位对齐，因为 32 位机器是 4 字节对齐，如果也按照 64 位机器取 state[0]，state[1] 就有可能会造成 atmoic 的使用错误。


于是 32 位机器上空出第一个 32 位，也就使后面 64 位天然满足 64 位对齐，第一个 32 位放入 sema 刚好合适。早期 WaitGroup 的实现 sema 是和 state1 分开的，也就造成了使用 WaitGroup 就会造成 4 个字节浪费，不过 go1.11 之后就是现在的结构了。

### 为什么流程图里缺少了 Done ?
其实并不是，是因为 Done 的实现就是 Add. 只不过我们常规用法 wg.Add(1) 是加 1 ，wg.Done() 是减 1，即 wg.Done() 可以用 wg.Add(-1) 来代替。 尽管我们知道 wg.Add 可以传递负数当 wg.Done  使用，但是还是别这么用。

### 退出waitgroup的条件
其实就一个条件， WaitGroup.counter 等于 0
## 日常开发中特殊需求
### 1. 控制超时/错误控制
虽说 WaitGroup 能够让主 Goroutine 等待子 Goroutine 退出，但是 WaitGroup 遇到一些特殊的需求，如：超时，错误控制，并不能很好的满足，需要做一些特殊的处理。
**
**真实场景：**


用户在电商平台中购买某个货物，为了计算用户能优惠的金额，需要去获取 A 系统（权益系统），B 系统（角色系统），C 系统（商品系统），D 系统（xx系统）。为了提高程序性能，可能会同时发起多个 Goroutine 去访问这些系统，必然会使用 WaitGroup 等待数据的返回，但是存在一些问题：


1. 当某个系统发生错误，等待的 Goroutine 如何感知这些错误？
1. 当某个系统响应过慢，等待的 Goroutine 如何控制访问超时？



这些问题都是直接使用 WaitGroup 没法处理的。如果直接使用 channel 配合 WaitGroup 来控制超时和错误返回的话，封装起来并不简单，而且还容易出错。我们可以采用 ErrGroup 来代替 WaitGroup。


有关 ErrGroup 的用法这里就不再阐述。golang.org/x/sync/errgroup 
```go
package main

import (
	"context"
	"fmt"
	"golang.org/x/sync/errgroup"
	"time"
)

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), time.Second*5)
	defer cancel()
	errGroup, newCtx := errgroup.WithContext(ctx)

	done := make(chan struct{})
	go func() {
		for i := 0; i < 10; i++ {
			errGroup.Go(func() error {
				time.Sleep(time.Second * 10)
				return nil
			})
		}
		if err := errGroup.Wait(); err != nil {
			fmt.Printf("do err:%v\n", err)
			return
		}
		done <- struct{}{}
	}()

	select {
	case <-newCtx.Done():
		fmt.Printf("err:%v ", newCtx.Err())
		return
	case <-done:
	}
	fmt.Println("success")
}
```
### 2. 控制 Goroutine 数量
场景模拟：
大概有 2000 - 3000 万个数据需要处理，根据对服务器的测试，当启动 200 个 Goroutine 处理时性能最佳。如何控制？


遇到诸如此类的问题时，单纯使用 WaitGroup 是不行的。既要保证所有的数据都能被处理，同时也要保证同时最多只有 200 个 Goroutine。这种问题需要 WaitGroup 配合 Channel 一块使用。


```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func main() {
	var wg = sync.WaitGroup{}
	manyDataList := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
	ch := make(chan bool, 3)
	for _, v := range manyDataList {
		wg.Add(1)
		go func(data int) {
			defer wg.Done()

			ch <- true
			fmt.Printf("go func: %d, time: %d\n", data, time.Now().Unix())
			time.Sleep(time.Second)
			<-ch
		}(v)
	}
	wg.Wait()
}
```
## 使用注意点


使用 WaitGroup 同样不能被复制。具体例子就不再分析了。具体分析过程可以参见《这可能是最容易理解的 Go Mutex 源码剖析》


WaitGroup 的剖析到这里基本就结束了。有什么想跟我交流的，欢迎评论区留言。

![gzh](https://images.haohongfan.com/gzh1.png)

## 版权

以上内容均不得复制用于商业用途或发行

© 2020-2021 haohongfan. Licensed under CC-BY-NC-ND 4.0