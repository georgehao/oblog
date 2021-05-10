---
title: "Redigo Pool 源码解析"
date: 2019-12-31T16:57:33+08:00
lastmod: 2019-12-31T16:57:33+08:00
keywords: [golang,redigo,pool]
---

## Redigo Pool 最重要的结构

```go
type Pool struct {
	// 真正获取跟redis-server连接的函数, 必填参数
	Dial func() (Conn, error)  

	// 这是个可选参数, 用于在从 pool 获取连接时, 检查这个连接是否正常使用. 所以这个参数一般是必填的
	TestOnBorrow func(c Conn, t time.Time) error 

	// 最多有多少个空闲连接保留, 一般必填
	MaxIdle int

	// 最多有多少活跃的连接数, 一般必填
	MaxActive int

	// 空闲连接最长空闲时间, 一般必填
	IdleTimeout time.Duration

	// Pool 的活跃的连接数达到 MaxActive, 如果 Wait 为 true, 
	// 那么 Get() 将要等待一个连接放到 Pool中, 才会返回一个连接给使用方
	Wait bool

	// 设置连接最大存活时间
	MaxConnLifetime time.Duration
	
	chInitialized uint32 // set to 1 when field ch is initialized
	mu     sync.Mutex    // mu protects the following fields
	closed bool          // 设置 Pool 是否关闭
	active int           // 当前 Pool 的活跃连接数
	ch     chan struct{} // 配合 Wait 为 true 使用
	idle   idleList      // 空闲队列
}
```

## Redigo 第二重要的结构: idleList

idleList 是个`双向链表`. 实现很简单. 只有三个方法: `pushFront`, `popFront`, `popBack`

### 初始化 idlelist

```go
type idleList struct {
	count       int
	front, back *poolConn
}
```

使用 Pool 不会明确的初始化 idle, 故当初始化`&Pool{....}`后, idle 就是默认值. 即: `count = 0, front = nil, back = nil.` 如下图:

![first_node](https://images.haohongfan.com/redigoidlelist01.png)

### 头部插入

idleList 只提供 pushFront 方法. 将 idleList 的 front 指针, 指向新的 Conn. 然后将新 Conn 与 之前节点连接即可, 如下图

![pushFront](https://images.haohongfan.com/redigoidlelist02.png)

### 从头部删除

![popFront](https://images.haohongfan.com/redigoidlelist03.png)

### 从尾部删除

![popBack](https://images.haohongfan.com/redigoidlelist04.png)

## Conn

整个 Pool 有两个 Conn: activeConn, errorConn, 他们都实现了 `redis.Conn` 接口. 

```go
type Conn interface {
	// Close closes the connection.
	Close() error
	// Err returns a non-nil value when the connection is not usable.
	Err() error
	// Do sends a command to the server and returns the received reply.
	Do(commandName string, args ...interface{}) (reply interface{}, err error)
	// Send writes the command to the client's output buffer.
	Send(commandName string, args ...interface{}) error
	// Flush flushes the output buffer to the Redis server.
	Flush() error
	// Receive receives a single reply from the Redis server
	Receive() (reply interface{}, err error)
}
```

不过 errorConn 实现的所有的函数都会返回 err. 所以我们在使用 Get() 获取链接时, 尽量要去判断拿到的连接是否可用(不过这也不是绝对, 我们也可以在使用 Conn 函数的时候去判断). 

activeConn 一般来说是可用的连接, 我们也可以通过 `activeConn.Err()` 来判断获取的连接是否可用.

activeConn 的函数都是调用 redis.Conn 的函数实现的. 唯一不同的地方在于, 所有的命令都会执行`LookupCommandInfo`这个函数. 

```go
func LookupCommandInfo(commandName string) CommandInfo {
	if ci, ok := commandInfos[commandName]; ok {
		return ci
	}
	return commandInfos[strings.ToUpper(commandName)]
}
```

```go
func (ac *activeConn) Do(commandName string, args ...interface{}) (reply interface{}, err error) {
	pc := ac.pc
	if pc == nil {
		return nil, errConnClosed
	}
	ci := internal.LookupCommandInfo(commandName)
	ac.state = (ac.state | ci.Set) &^ ci.Clear
	return pc.c.Do(commandName, args...)
}
```
以`Multi`, `EXEC`举例:

使用 Pool 获取到的连接, 发送 `Multi`完, 由于某种原因导致程序执行了 `defer activeConn.Close()`, 由于 redis-server 没有得到`Exec`, 那么接下来的命令 redis-server 都会把其当成事务的一部分. 所以通过 LookupCommandInfo 函数能够计算当前某条命令执行后 activeConn 的当前状态, 当执行到 `activeConn.Close()` 时发现还没有发送 `EXEC`, 那么就会发送 `DISCARD` 命令来将事务取消. 

LookupCommandInfo 支持的命令有: `WATCH`, `UNWATCH`, `MULTI`, `EXEC`, `DISCARD`, `PSUBSCRIBE`, `SUBSCRIBE`, `MONITOR`

## Redigo 灵魂函数 -- get() 

get() 主要提供给 Get() 函数调用, 是从 Pool 中获取连接, Get()是我们使用者调用的函数. 可以看到 Get() 由 `errorConn`, `activeConn` 组成. 

```go
func (p *Pool) Get() Conn {
	pc, err := p.get(nil)
	if err != nil {
		return errorConn{err}
	}
	return &activeConn{p: p, pc: pc}
}
```

#### Wait 配合 MaxActive 使用, 来保证 Get() 将要等待一个连接放到 Pool中, 才会返回一个连接给使用方

```go
if p.Wait && p.MaxActive > 0 {
	p.lazyInit()
	if ctx == nil {
		<-p.ch // <--- 这里会阻塞
	} else { 
		// 下面的代码不需要关于, Get() 传递下来的参数是 gnil
		select {
		case <-p.ch:
		case <-ctx.Done():
			return nil, ctx.Err()
		}
	}
}
```

设置 Wait 为 true 并且 MaxActive 设置有最大数时, 如果时第一次获取 Get(), 那么会调用`lazyInit()`进行初始化.

初始化一个 MaxActive 的 channel. 利用 channel 来保证当 Get() 获取到的连接数大于 MaxActive时, 阻塞 Get() 函数, 直到有连接使用完毕放入到 Pool 中.

#### 循环idleList, 关闭空闲队列中连接时长大于 IdleTimeout 的连接

```go
// Prune stale connections at the back of the idle list.
if p.IdleTimeout > 0 {
	n := p.idle.count
	for i := 0; i < n && p.idle.back != nil && p.idle.back.t.Add(p.IdleTimeout).Before(nowFunc()); i++ {
		pc := p.idle.back
		p.idle.popBack()
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}
}
```

这段代码会遍历整个 idleList, 从尾部拿出 `activeConn`,  activeConn.t 加上 IdleTimeout 时间, 跟当前时间比较. 如果比 time.Now() 小, 则从 idleList 的尾部 `pop` 这个conn. 同时关闭这个 activeConn, 让 Pool.active--

不过这段代码看上去比较奇怪, 需要耐心去看. 可以做个变形

```go
for i := 0; i < n ; i++ {
	if p.idle.back != nil && p.idle.back.t.Add(p.IdleTimeout).Before(nowFunc()) {
		pc := p.idle.back
		p.idle.popBack()
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}
}
```

#### 当 idleList 不为空时, 从头部获取 activeConn

```go
for p.idle.front != nil {
	pc := p.idle.front
	p.idle.popFront()
	p.mu.Unlock()
	if (p.TestOnBorrow == nil || p.TestOnBorrow(pc.c, pc.t) == nil) &&
		(p.MaxConnLifetime == 0 || nowFunc().Sub(pc.created) < p.MaxConnLifetime) {
		return pc, nil
	}
	pc.c.Close()
	p.mu.Lock()
	p.active--
}
```

这段代码比较简单.当 idleList 不为空时, 从头部 pop 出一个 activeConn, 还有另外两个功能: 

1. 通过 `TestOnBorrow()` 函数判断当前连接是否能正常使用
2. 通过 `MaxConnLifetime` 参数判断这个连接是否在 MaxConnLifetime 内

#### 判断 Pool 是否关闭

```go
if p.closed {
	p.mu.Unlock()
	return nil, errors.New("redigo: get on closed pool")
}
```

当发现 Pool 被调用 Pool.Close() 关闭了, 那么这里就会返回 `errors.New("redigo: get on closed pool")`错误

```
顺带说一句在 `pool.go` 里面总共有两个 Close() 函数:

1. func (p *Pool) Close() error {...}

这个函数是关闭 redigo 连接池的. 理论上可以不调用. 如果确实不放心, 需要在 main.go 里面 `defer pool.Close()` 
来调用

2. func (ac *activeConn) Close() error { ...}

这个函数是用来将从 Pool 中获取到的 activeConn 放回到 Pool 里面. 

这个函数是我们需要频繁调用的函数. 如果程序里Get() 之后没有 Close(), 那么就会造成 redis 连接泄漏.
更严重的情况, 如果Wait, MaxActive 都没有设置, 那么你的程序就会将 redis 搞瘫痪, 这是很危险的
```

#### 真正从 redis-server 获取连接

```go
// Handle limit for p.Wait == false.
if !p.Wait && p.MaxActive > 0 && p.active >= p.MaxActive {
	p.mu.Unlock()
	return nil, ErrPoolExhausted
}

p.active++
p.mu.Unlock()
c, err := p.Dial()
if err != nil {
	c = nil
	p.mu.Lock()
	p.active--
	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
}
return &poolConn{c: c, created: nowFunc()}, err
```

1. 判断 pool 的 active 是否达到 MaxActive
2. 通过参数 p.Dial() 去 redis-server 获取连接

## Redigo 灵魂函数 -- put() 

put 函数主要提供给 activeConn.Close() 调用

Close() 函数就不在详细说明,  主要根据 activeConn 的 stat, 判断在关闭连接之前是否发送过`WATCH`, `MULTI`, `PSUBSCRIBE`, `SUBSCRIBE`, `MONITOR` 这些命令. 如果发送过就会把这些命令结束(具体原因上面已经说过)

```go
func (p *Pool) put(pc *poolConn, forceClose bool) error {
	p.mu.Lock()
	// 判断 pool 是否关闭, 并且该命令是否需要强制关闭
	if !p.closed && !forceClose { 
		pc.t = nowFunc()
		// 将该 activeConn 压入 idleList 中
		p.idle.pushFront(pc)
		// 如果 idleList 的 count 已经大于 MaxIdle, 那么会将 idleList 的尾部的 activeConn pop 掉
		if p.idle.count > p.MaxIdle {
			pc = p.idle.back
			p.idle.popBack()
		} else {
			pc = nil
		}
	}

	// 如果是需要强制关闭或者是从尾部 pop 掉的 conn, 那么就会真正的关闭这个连接
	if pc != nil {
		p.mu.Unlock()
		pc.c.Close()
		p.mu.Lock()
		p.active--
	}

	// 如果开启了 Wait = true, 那么往 channel 里面发送一个struct{}{}, 代表等待的客户端可以获取连接了
	if p.ch != nil && !p.closed {
		p.ch <- struct{}{}
	}
	p.mu.Unlock()
	return nil
}
```

## 结尾

至此, redigo Pool 的源码基本都过了一遍. 为什么我会心血来潮把其源码读一遍呢? 

思考下面几个问题:

1. redigo 是否能够用于 codis?
2. codis 的 golang 客户端如何实现 ?
3. 如果不经过任何加工, 直接用 redigo 去访问 codis, 会出现什么样的问题?

这些问题就是促使我读redigo源码的原因, 后续文章我会一一解答这些问题

