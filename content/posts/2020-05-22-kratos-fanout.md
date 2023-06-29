---
title: Bilibili Kratos 框架源码分析(3) -- fanout异步
date: 2020-05-22T10:14:07+08:00
lastmod: 2020-05-22T10:14:07+08:00
keywords: [golang, kratos]
tags: [golang, kratos]
categories: [golang, kratos]
---

在写项目代码时如果遇到需要异步处理时, 如异步更新 redis, 异步比对数据等等, 我们的常规处理一般是 MQ. 但有的时候我们的操作其实很简单, 写 MQ 显得又太重了, 那么该如何在程序里实现一个异步功能? 

本篇文章就介绍下 Kratos 官方 wiki 没有提到的功能 `Fanout`. 其实Fanout 在 生成的 kratos-demo 里面是有体现的, 只是没有使用 demo 而已

```go
type dao struct {
	...
	cache      *fanout.Fanout
	...
}

func newDao(r *redis.Redis, mc *memcache.Memcache, db *sql.DB) (d *dao, cf func(), err error) {
	...
	d = &dao{
		...
		cache:      fanout.New("cache"),
		...
	}
	cf = d.Close
	return
}
```

## 使用方式

实现功能: 程序处理完相关逻辑后, 同步更新mysql, 然后异步刷新redis

```go
func (s *Service) UpdateRole(ctx context.Context, req *pb.UpdateRoleReq) (resp *pb.UpdateRoleResp, err error) {
	resp = &pb.UpdateRoleResp{}
	_, err = s.dao.UpdateRole(ctx, req.Role)
	if err != nil {
		return resp, err
	}
	resp.Yes = true

	// 刷新redis
	err = s.dao.FanoutDo(ctx, func(c context.Context) {
		s.dao.UpdateRoleRedis(c, 100, req.Role)
	})
	return
}
```

全部的代码就不贴出来了. 会上传到[github](https://github.com/georgehao/kratos-demo)

## 实现原理

Fanout 的实现很简单, 其实就是一个 channel

```go
type Fanout struct {
	name    string  
	ch      chan item
	options *options
	waiter  sync.WaitGroup

	ctx    context.Context
	cancel func()
}
```

* name 对Fanout来说没有意义, 只是给 metrics 打点使用
* options Fanout 的参数: worker(有多少个协程在处理), buffer (channel大小)
* waiter 就是sync.WaitGroup, 用来管理 worker 
* cancel 关闭处理函数, 就是 context.WithCancel


### New() 创建 Fanout

```go
func New(name string, opts ...Option) *Fanout {
	if name == "" {
		name = "anonymous"
	}
	o := &options{
		worker: 1,
		buffer: 1024,
	}
	for _, op := range opts {
		op(o)
	}
	c := &Fanout{
		ch:      make(chan item, o.buffer),
		name:    name,
		options: o,
	}
	c.ctx, c.cancel = context.WithCancel(context.Background())
	c.waiter.Add(o.worker)
	for i := 0; i < o.worker; i++ {
		go c.proc()
	}
	return c
}
```

在 `internal/dao/dao.go` 里面, 通过 `fanout.New("cache")` 来初始化 Fanout. Fanout 默认一个 worker 来消费, channel 的默认大小1024. Fanout启动 `worker` 个 `c.proc` 来处理数据

### proc 

```go
func (c *Fanout) proc() {
	defer c.waiter.Done()
	for {
		select {
		case t := <-c.ch:
			wrapFunc(t.f)(t.ctx)
			_metricChanSize.Set(float64(len(c.ch)), c.name)
			_metricCount.Inc(c.name)
		case <-c.ctx.Done():
			return
		}
	}
}

func wrapFunc(f func(c context.Context)) (res func(context.Context)) {
	res = func(ctx context.Context) {
		defer func() {
			if r := recover(); r != nil {
				buf := make([]byte, 64*1024)
				buf = buf[:runtime.Stack(buf, false)]
				log.Error("panic in fanout proc, err: %s, stack: %s", r, buf)
			}
		}()
		f(ctx)
		if tr, ok := trace.FromContext(ctx); ok {
			tr.Finish(nil)
		}
	}
	return
}
```

`proc` 一直从 channel 获取数据. 真正的处理逻辑是 `t.f`. 这个是使用方传进来的

### Do

向 channel 里放数据的函数

```go
func (c *Fanout) Do(ctx context.Context, f func(ctx context.Context)) (err error) {
	if f == nil || c.ctx.Err() != nil {
		return c.ctx.Err()
	}
	nakeCtx := metadata.WithContext(ctx)
	if tr, ok := trace.FromContext(ctx); ok {
		tr = tr.Fork("", "Fanout:Do").SetTag(traceTags...)
		nakeCtx = trace.NewContext(nakeCtx, tr)
	}
	select {
	case c.ch <- item{f: f, ctx: nakeCtx}:
	default:
		err = ErrFull
	}
	_metricChanSize.Set(float64(len(c.ch)), c.name)
	return
}
```

注意的是: select 需要有默认的 `default`, 不然 channel 满的时候, 程序就阻塞了


## 总结

Fanout 实现很简单, 但是还是一个蛮实用的功能. 不过这个实现有个问题值得注意: 

所有的数据都是往一个 channel 里面 push, 如果 worker 数量设置不合理, 就会造成 channel 里面的数据挤压, 从而导致数据无法被处理而被 Do 函数舍弃掉或者消费的不及时导致出现数据不一致

所以在使用 Fanout 的时候, 要评估下数据量, 设置合理的 worker 及 buffer 大小