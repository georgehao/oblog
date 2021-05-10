---
title: 限流器系列(1) -- Leaky Bucket 漏斗桶
date: 2020-06-27T10:14:07+08:00
lastmod: 2020-06-27T10:14:07+08:00
keywords: [golang, bbr, leaky bucket, token bucket]
tags: [golang, kratos]
categories: [golang, kratos]
---


限流器(Rate Limiter)在微服务中的重要性不言而喻了. 下游服务的稳定性, 防止过载, 全靠这个组件来保证. 限流器的实现方式, 基本有下面几种方式

1. 计数器
2. 漏斗通 (Leaky Bucket)
3. 令牌桶 (Token Bucket)
4. 基于 BBR 算法的自适应限流
5. 基于 Nginx 的限流
6. 分布式限流

这个系列的文章会逐一介绍各种限流器. 本篇文章会结合比较成熟组件介绍: 漏斗桶

## 什么是限流器

> Web servers typically use a central in-memory key-value database, like Redis or Aerospike, for session management. A rate limiting algorithm is used to check if the user session (or IP address) has to be limited based on the information in the session cache.
>
> In case a client made too many requests within a given time frame, HTTP servers can respond with status code 429: Too Many Requests

这段话是摘自[维基百科](https://en.wikipedia.org/wiki/Rate_limiting). 简单来说`限流器`是基于 KV 内存数据库的一个限速判断, 在给定的时间内, 客户端请求次数过多, 服务器就会返回状态码 429: Too Many Request

## 计数器

计数器是一种最简单的限流器.

如果把 QPS 设置为100, 从第一个请求进来开始计时，在接下去的1s内，每来一个请求，就把计数加1，如果累加的数字达到了100，那么后续的请求就会被全部拒绝。等到1s结束后，把计数恢复成0，重新开始计数. 这种计数器一般称为`固定窗口计数器算法`.

可以看到计数器虽说有一定的缓冲空间, 但是需要一定的恢复空窗期, 在这个恢复时间内请求全部拒绝. 计数器还存在着另外一个问题, 特殊情况下会让请求的通过量为限制的两倍.

考虑如下情况：

限制 1 秒内最多通过 5 个请求，在第一个窗口的最后半秒内通过了 5 个请求，第二个窗口的前半秒内又通过了 5 个请求。这样看来就是在 1 秒内通过了 10 个请求

综合来看, 计数器方式的限流是比较简单粗暴的, 我们需要更加优雅的限流方式

## 漏斗桶

相对于`计数器`的粗鲁方式, 漏斗桶会更加优雅一些, 如下图

![leaky_bucket](https://images.haohongfan.com/leaky_bucket1.png?imageView2/2/w/500/h/500)

其实从字面就很好理解. 类似生活用到的漏斗, 当客户端请求进来时，相当于水倒入漏斗，然后从下端小口慢慢匀速的流出。不管上面流量多大，下面流出的速度始终保持不变.

当水流入速度过大时, 漏斗就会溢出, 同样会造成服务拒绝. 相对于`计数器`的在恢复期内全部拒绝请求, 因为漏斗桶会以一定的速率消费请求, 这样就能够让后续的请求有机会进入到漏斗桶里面.

### 漏斗桶的弊端

由于漏斗桶有点类似队列, 先进去才能被消费掉, 如果漏斗桶溢出了, 后续的请求都直接丢弃了, 也就是说漏斗桶是无法短时间应对突发流量的. 对于互联网行业来说, 面对突发流量, 不能一刀切将突发流量全部干掉, 这样会给产品带来口碑上影响. 因此漏斗桶也不是完美的方案.

不过漏斗桶能够限制数据的平均传输速率, 能够满足大部分的使用场景的. 如: 我们可以使用漏斗桶限制论坛发帖频率

## Uber Ratelimit

Uber Ratelimit 是漏斗桶的一个具体实现. 下面主要结合 [Uber Ratelimit](https://github.com/uber-go/ratelimit) 来介绍 Leaky Buckt(漏洞桶)

### 官方 demo

```go
func main() {
	rl := ratelimit.New(100) // per second
    prev := time.Now()
    for i := 0; i < 10; i++ {
        now := rl.Take()
        fmt.Println(i, now.Sub(prev))
        prev = now
    }
}
```

Output:

```
 0 0
 1 10ms
 2 10ms
 3 10ms
 4 10ms
 5 10ms
 6 10ms
 7 10ms
 8 10ms
 9 10ms
```

从这个例子的输出结果, 可以看出来有下面这些特点:

* 初始化时需要设置 bucket 大小
* 输出结果是间隔 10ms, 由此可以看出来 leaky bucket 一定是保证匀速率的从桶内取值
* 通过 `Take()` 函数与 ratelimiter 来交互, 但是 Take() 的返回值却是上一次拿到的请求时间

### gin 中间件

```go
import (
	"fmt"
	"github.com/gin-gonic/gin"
	"go.uber.org/ratelimit"
	"time"
)

var rl ratelimit.Limiter

func leakyBucketRateLimiter() gin.HandlerFunc {
	prev := time.Now()
	return func(c *gin.Context) {
		now := rl.Take()
		fmt.Println(now.Sub(prev)) // 这里不需要, 只是打印下多次请求之间的时间间隔
		prev = now // 这里不需要, 只是打印下多次请求之间的时间间隔
	}
}

func main() {
	engine := gin.Default()
	engine.GET("/test", leakyBucketRateLimiter(), func(context *gin.Context) {
		context.JSON(200, true)
	})
	engine.Run(":9191")
}

func init() {
	rl = ratelimit.New(10)
}
```

Output:

```
[GIN] 2020/06/29 - 23:21:22 | 200 |     166.119µs |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:22 | 200 |  116.954372ms |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:23 | 200 |  203.502985ms |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:23 | 200 |  303.266345ms |       127.0.0.1 | GET      /test

....

[GIN] 2020/06/29 - 23:21:57 | 200 | 24.899798034s |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:57 | 200 | 24.899258055s |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:57 | 200 | 24.899960588s |       127.0.0.1 | GET      /test
100ms
[GIN] 2020/06/29 - 23:21:57 | 200 | 24.899834294s |       127.0.0.1 | GET      /test
```

从这个例子的输出结果, 有下面这些特点:

* 输出结果是间隔仍然是 10ms
* 当漏斗桶溢出后, 请求处理耗时越来越长

### 疑问

1. Uber Ratelimiter 溢出后为什么请求耗时越来越长?
2. 为什么 Uber ratelimiter 不需要返回 429?

## 源码分析

### New 初始化函数

```go
func New(rate int, opts ...Option) Limiter {
	l := &limiter{
		perRequest: time.Second / time.Duration(rate),
		maxSlack:   -10 * time.Second / time.Duration(rate), // 最大松弛度
	}

	// ...

	return l
}
```

Uber Leaky Bucket 的设计有点取巧. `New(10)` 传入的 10 指的是 1s 内只有能有 10 个请求通过, 于是算出来每个请求之间应该间隔 100 ms. 如果两个请求之间间隔时间过短, 那么需要第二个请求 sleep 一段时间, 这样保证请求能够匀速从桶内流出. 如下图

![](https://images.haohongfan.com/uber_sleep_for.png?imageView2/2/w/500/h/500)

对比上面漏斗桶的概念, 我们发现当请求通过 Uber 限流器的时候, 如果溢出了, 就只能强行 sleep, 造成后续请求排队, 处理时长越来越长. 另外上游服务必须得有超时机制.

### Take()

```go
func (t *limiter) Take() time.Time {
	t.Lock()
	defer t.Unlock()

	now := t.clock.Now()

	// 如果是第一次请求, 直接让通过
	if t.last.IsZero() {
		t.last = now
		return t.last
	}

	// 这里有个最大松弛量的概念maxSlack
	t.sleepFor += t.perRequest - now.Sub(t.last)
	if t.sleepFor < t.maxSlack {
		t.sleepFor = t.maxSlack
	}

	// 判断是否桶溢出. 如果桶溢出了, 需要sleep一段时间
	if t.sleepFor > 0 {
		t.clock.Sleep(t.sleepFor)
		t.last = now.Add(t.sleepFor)
		t.sleepFor = 0
	} else {
		t.last = now
	}
	return t.last
}
```

### 最大松弛量

漏斗桶有个天然缺陷就是无法应对突发流量, 对于这种情况，uber-go 对 Leaky Bucket 做了一些改良，引入了最大松弛量 (maxSlack) 的概念

上面计算 sleepFor 的第 14 行代码如果按下面这样写:

```go
t.sleepFor = t.perRequest - now.Sub(t.last)
```

请求 1 完成后，15ms 后，请求 2 才到来，可以对请求 2 立即处理。请求 2 完成后，5ms 后，请求 3 到来，这个时候距离上次请求还不足 10ms，因此还需要等待 5ms, 但是，对于这种情况，实际上三个请求一共消耗了 25ms 才完成，并不是预期的 20ms

![](https://images.haohongfan.com/uber_max_slack1.png?imageView2/2/w/500/h/500)

Uber 实现的 ratelimit 中，可以把之前间隔比较长的请求的时间，匀给后面的使用，保证每秒请求数 (RPS). 对于以上 case，因为请求 2 相当于多等了 5ms，我们可以把这 5ms 移给请求 3 使用。加上请求 3 本身就是 5ms 之后过来的，一共刚好 10ms，所以请求 3 无需等待，直接可以处理。此时三个请求也恰好一共是 20ms

![](https://images.haohongfan.com/uber_max_slack2.png?imageView2/2/w/500/h/500)

但是也有特殊情况, 假设计算出来的间隔时间 100ms, 但是 `请求1` 和 `请求2` 之间的间隔时间 2h, 如果没有`t.sleepFor = t.maxSlack` 这段 `最大松弛量` 的代码, 那么 `请求2` 需要 sleep 2h 才能继续执行, 显然这不符合实际情况. 故引入了最大松弛量 (maxSlack), 表示允许抵消的最长时间

## 参考

1. [分布式服务限流实战，已经为你排好坑了](https://www.infoq.cn/article/Qg2tX8fyw5Vt-f3HH673)
2. [uber-go 漏桶限流器使用与原理分析](https://www.cyhone.com/articles/analysis-of-uber-go-ratelimit/)