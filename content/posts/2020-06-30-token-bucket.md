---
title: "限流器系列(2) -- Token Bucket 令牌桶"
date: 2020-06-30T18:38:23+08:00
lastmod: 2020-06-30T18:38:23+08:00
keywords: [golang, bbr, leaky bucket, token bucket]
description: ""
tags: [golang, kratos]
categories: [golang, kratos]
author: ""
---

[上一篇](https://www.haohongfan.com/post/2020-06-27-leaky-bucket/)说到 Leaky Bucket 能限制客户端的访问速率, 但是无法应对突发流量, 本质原因就是漏斗桶只是为了保证固定时间内通过的流量是一样的. 面对这种情况, 本篇文章继续介绍另外一种限流器: Token Bucket -- 令牌桶

## 什么是 Token Bucket

漏斗桶的桶空间就那么大, 其只能保证桶里的请求是匀速流出, 并不关心流入的速度, 只要桶溢出了就服务拒绝, 这可能并不符合互联网行业的使用场景.

试想这样的场景, 尽管服务器的 QPS 已经达到限速阈值了, 但是并不想将所有的流量都拒之门外, 仍然让部分流量能够正常通过限流器. 这样我们的服务器在面对突发流量时能够有一定的伸缩空间, 而不是一直处于不可用状态.

基于上面的场景需求, 令牌桶采用跟漏斗桶截然不同的做法.

![令牌桶](https://images.haohongfan.com/token_bucket.png?imageView2/2/w/500/h/500)

令牌桶也有自己的固定大小, 我们设置 QPS 为 100, 在初始化令牌桶的时候, 就会立即生成 100 个令牌放到桶里面, 同时还按照一定的速率, 每隔一定的时间产生固定数量的令牌放入到桶中. 如果桶溢出了, 则舍弃生成的令牌.

只要有请求能够拿到令牌, 就能保证其通过限流器. 当然拿不到令牌的请求只能被无情拒绝了(或者等待令牌产生), 这个请求的命不好~

面对突然爆发的流量, 可能大部分流量都被限流器给挡住了, 但是也有部分流量刚好拿到了刚刚生成的 Token, 就这样在千军万马中通过了限流器. 相对于漏斗桶来说, 令牌桶更适合现在互联网行业的需要, 是被广泛使用的限流算法

**如何设置令牌桶的大小和产生令牌的速率?**

答: 多进行生产环境的压测, 根据集群的实际承受能力设置相应桶的大小和产生令牌的速率. 血的经验告诉我, 周期性的线上压测是一件很重要的事情(使用local cache的程序, 压测的时候一定要记得先临时关闭它)

## juju/ratelimit

[juju/ratelimit](https://github.com/juju/ratelimit) 是大部分项目都在使用的 golang 令牌桶的实现方案. 下面会结合其用法, 源码剖析令牌桶的实现的方案.

### gin 中间件

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
	"github.com/juju/ratelimit"
)

var limiter = ratelimit.NewBucketWithQuantum(time.Second, 10, 10)

func tokenRateLimiter() gin.HandlerFunc {
	fmt.Println("token create rate:", limiter.Rate())
	fmt.Println("available token :", limiter.Available())
	return func(context *gin.Context) {
		if limiter.TakeAvailable(1) == 0 {
			log.Printf("available token :%d", limiter.Available())
			context.AbortWithStatusJSON(http.StatusTooManyRequests, "Too Many Request")
		} else {
			context.Writer.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", limiter.Available()))
			context.Writer.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", limiter.Capacity()))
			context.Next()
		}
	}
}

func main() {
	e := gin.Default()
	e.GET("/test", tokenRateLimiter(), func(context *gin.Context) {
		context.JSON(200, true)
	})
	e.Run(":9091")
}

```

Output:

```
token create rate: 100
available token : 100
[GIN] 2020/07/03 - 17:34:37 | 200 |     157.505µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |     310.898µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |       61.64µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |       8.677µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |       6.145µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |      23.576µs |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:34:37 | 200 |       5.617µs |       127.0.0.1 | GET      /test
.....
[GIN] 2020/07/03 - 17:35:03 | 429 |    6.193792ms |       127.0.0.1 | GET      /test
[GIN] 2020/07/03 - 17:35:03 | 200 |       8.509µs |       127.0.0.1 | GET      /test <----- 看这里
[GIN] 2020/07/03 - 17:35:03 | 429 |      10.324µs |       127.0.0.1 | GET      /test
....
```

有下面特点:

1. 令牌桶初始化后里面就有 100 个令牌
2. 每秒钟会产生 100 个令牌, 保证每秒最多有 100 个请求通过限流器, 也就是说 QPS 的上限是 100
3. 流量过大时能够启动限流, 在限流过程中, 仍然能让部分流量通过

### 源码分析

#### 初始化

建议使用初始化函数有下面三种:

* NewBucket(fillInterval time.Duration, capacity int64): 默认的初始化函数, 每一个周期内生成 1 个令牌, 默认 quantum = 1
* NewBucketWithQuantum(fillInterval time.Duration, capacity, quantum int64) : 跟 NewBucket 类似, 每个周期内生成 quantum 个令牌
* NewBucketWithRate(rate float64, capacity int64): 每秒中产生 rate 速率的令牌

其实初始化函数还有很多种, 但基本上大同小异, 最后都是调用 `NewBucketWithQuantumAndClock`.

```go
func NewBucketWithQuantumAndClock(fillInterval time.Duration, capacity, quantum int64, clock Clock) *Bucket {
	// ....
	return &Bucket{
		clock:           clock,
		startTime:       clock.Now(),
		latestTick:      0,            // 上一次产生token的记录点
		fillInterval:    fillInterval, // 产生token的间隔
		capacity:        capacity,     // 桶的大小
		quantum:         quantum,      // 每秒产生token的速率
		availableTokens: capacity,     // 桶内可用的令牌的个数
	}
}
```

#### Rate

```go
func (tb *Bucket) Rate() float64 {
	return 1e9 * float64(tb.quantum) / float64(tb.fillInterval)
}
```

`time.Duration` 实际的是以 `nanosecond` 试试呈现的, 就是 `1e9`, 1e9 / float64(tb.fillInterval) 的结果就是 1/tb.fillInterval 秒.

于是令牌桶产生令牌的速率是: 每秒内产生 float64(tb.quantum) / float64(tb.fillInterval)

#### TakeAvailable

```go
func (tb *Bucket) currentTick(now time.Time) int64 {
	// 由于 tb.quantum 是每秒产生的token的数量. 于是计算从bucket初始化的startTime到现在now的时间间隔 t,
	// t/tb.fillInterval * tb.quantum 计算的是从开始到现在应该产生的 token 数量
	return int64(now.Sub(tb.startTime) / tb.fillInterval)
}

func (tb *Bucket) adjustavailableTokens(tick int64) {
	if tb.availableTokens >= tb.capacity { // 如果令牌的可用数量已经达到桶的容量, 直接返回
		return
	}
	
	// tick * tb.quantum 是从bucket初始化到本次请求应该产生的 token的数量
	// tb.latestTick 是从bucket初始化到上次请求应该产生的 token的数量
	// tick * tb.quantum - tb.latestTick 计算出两次请求间应该产生的token数量
	// tb.availableTokens += (tick - tb.latestTick) * tb.quantum: 桶内剩余的token数量 + 新产生的token数量
	tb.availableTokens += (tick - tb.latestTick) * tb.quantum
	if tb.availableTokens > tb.capacity {
		tb.availableTokens = tb.capacity // 如果产生的令牌数量超过了桶的容量, 则桶内剩余的令牌数量等于桶的size
	}
	tb.latestTick = tick
	return
}

func (tb *Bucket) takeAvailable(now time.Time, count int64) int64 {
	if count <= 0 {
		return 0
	}
	tb.adjustavailableTokens(tb.currentTick(now))
	if tb.availableTokens <= 0 { // 如果桶内剩余token数量小于等于0, 直接返回0
		return 0
	}
	if count > tb.availableTokens {
		count = tb.availableTokens
	}
	tb.availableTokens -= count
	return count
}

// 如果返回值是0, 代表桶内已经没有令牌了
func (tb *Bucket) TakeAvailable(count int64) int64 {
	tb.mu.Lock()
	defer tb.mu.Unlock()
	return tb.takeAvailable(tb.clock.Now(), count)
}
```

TakeAvailable 是 Token Bucket的核心函数. 从这个实现我们能看到

1. jujue/ratelimit 计算出请求间隔中应该产生的token的数量, 并不是另外启动一个 goroutine 专门定时产生固定数量的token
2. 桶内令牌在产生过程中是累加的, 同时减去每次调用消耗的数量
3. 初始化后桶内的令牌数量就是桶的大小

![示意图](https://images.haohongfan.com/token_bucket_rate.png?imageView2/2/w/700/h/700)

## Token Bucket的缺陷

令牌桶算法能满足绝大部分服务器限流的需要, 是被广泛使用的限流算法, 不过其也有一些缺点:

1. 令牌桶是没有优先级的，无法让重要的请求先通过
2. OP可能因为硬件故障去调整资源, 系统负载也会随着在变化, 如果对服务限流进行缩容和扩容，需要人为手动去修改，运维成本比较大
3. 令牌桶只能对局部服务端的限流, 无法掌控全局资源

下一篇我们看`alibaba/Sentinel`, kratos 的 bbr 算法是如何做到系统自适应限流

## 参考

[1] [分布式服务限流实战，已经为你排好坑了](https://www.infoq.cn/article/Qg2tX8fyw5Vt-f3HH673)

[2] [维基百科--Token_bucket](https://en.wikipedia.org/wiki/Token_bucket)

[3] [juju/ratelimit](https://github.com/juju/ratelimit)

[4] [B 站在微服务治理中的探索与实践](https://www.infoq.cn/article/zRuGHM_SsQ0lk7gWyBgG)