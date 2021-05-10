---
title: "如何用 Redigo 访问 Codis"
date: 2020-01-06T13:58:36+08:00
lastmod: 2020-01-06T13:58:36+08:00
keywords: [golang,redis,redigo,codis]
---

开篇依然是那三个问题: 

1. redigo 是否能够用于 codis ?
2. 如果不经过任何加工, 直接用 redigo 去访问 codis, 会出现什么样的问题 ?
3. codis 的 golang 客户端如何实现 ?

先贴出来, 我之前直接用 Redigo 接入 codis 的代码

```go
// Redis global redis connection pool
var Redis *redis.Pool
var RedisInitErr = errors.New("init redis error")

Redis = &redis.Pool{
	MaxIdle: 10,
	Dial: func() (conn redis.Conn, e error) {
		addrs, err := getHosts() 
		if err != nil {
			panic("init redis panic")
		}

		rand.Seed(time.Now().UnixNano())
		rand.Shuffle(len(addrs), func(i, j int) {
			addrs[i], addrs[j] = addrs[j], addrs[i]
		})

		var handler redis.Conn
		for _, v := range addrs {
			var err error
			handler, err = redis.Dial("tcp", v)
			if err != nil || handler == nil {
				continue
			}

			res, err := handler.Do("PING")
			if pong, err := redis.String(res, err); err != nil && pong != "PONG" {
				_ = handler.Close()
				continue
			}
		}

		if handler != nil && handler.Err() == nil {
			return handler, nil
		}

		return nil, RedisInitErr
	},
	Wait: true,
}
```

这代码里 `getHosts()` 函数是从服务发现里面取到 Codis Proxy 的所有的 IP + Port. 还使用`洗牌算法`, 保证是随机从所有的 Proxy 中拿到一个 IP + Port.

```go
func TestRedis(t *testing.T) {
	res, err := Redis.Get().Do("ping")
	pong, err := redis.String(res, err)
	assert.NoError(t, err)
	assert.Equal(t, pong, "PONG")
}
```
我还写了一系列的单元测试, 表面上看也能设置/获取到数据, 似乎一切都很完美, Perfect!

但是上完线后, OP 告诉我访问 Codis 访问不均匀. 我当时就纳闷了, 啥叫访问不均匀 (咱啥也不知道, 啥也不敢问呀!)

![codis-uneven](https://images.haohongfan.com/codis-uneven1.png)

说到这里, 我们不得不说说 Codis 的架构图

![codis-architecture](https://images.haohongfan.com/codis-architecture.png)

请注意, Zookeeper, 这里也就是告诉客户端想要获取到 codis-proxy, 是需要通过服务注册发现的方式的. 但是我的程序里面也有这个呀, 为啥就访问不均匀了呢?

**想这样一个问题, 假如有 10 个 codis-proxy, 如果因为是随机从 codis-proxy 中取值的, 如果说刚好从 4 个 proxy 中取到的连接数就能满足所有的请求数. 由于这 4 个连接一直在 redigo pool 中保持活跃, 而且 pool 参数里面没有设置 `MaxConnLifetime` , 最终的结果就导致了所有的请求全都分配到了这 4 个 proxy 上.** 

这个现象依然适用于很大的 QPS 时, 当很大的 QPS 请求时取到的连接, 很可能大部分集中在某几个 codis-proxy上, 也就出现了上面访问不均匀的截图, 真相就是这样

**结论: 直接使用 redigo 访问 codis 是有问题的.** 当然, 如果你的服务 QPS 很低, 这个问题倒不是很大, 但也要特别注意这个问题

#### 那么问题来了, Go 如何访问 Codis 呢?

Google 了一下也没有发现开源的 golang codis 客户端. 我们还是去看看官方爸爸的 codis java 版本客户端 [jodis](https://github.com/CodisLabs/jodis) 是如何实现的? 

使用起来很简单, 如下:

```java
JedisResourcePool jedisPool = RoundRobinJedisPool.create()
        .curatorClient("zkserver:2181", 30000).zkProxyDir("/jodis/xxx").build();
try (Jedis jedis = jedisPool.getResource()) {
    jedis.set("foo", "bar");
    String value = jedis.get("foo");
    System.out.println(value);
}
```

主要关注`create()`, `build()`, `getResource()` 是如何实现的即可


1.create()

```java
public static Builder create() {
	return new Builder();
}
```

2.build()

```java
public RoundRobinJedisPool build() {
	validate();
	return new RoundRobinJedisPool(curatorClient, closeCurator, zkProxyDir, poolConfig,
			connectionTimeoutMs, soTimeoutMs, password, database, clientName);
}
```

build() 返回一个 `RoundRobinJedisPool` 对象. 从名字也能看出来, Codis Pool 是个轮询池

3.getResource()

```java
@Override
public Jedis getResource() {
	ImmutableList<PooledObject> pools = this.pools;
	if (pools.isEmpty()) {
		throw new JedisException("Proxy list empty");
	}
	for (;;) {
		int current = nextIdx.get();
		int next = current >= pools.size() - 1 ? 0 : current + 1;
		if (nextIdx.compareAndSet(current, next)) {
			return pools.get(next).getResource();
		}
	}
}
```

要先弄明白 pools 是什么? 

```java
private void resetPools() {
	ImmutableList<PooledObject> pools = this.pools;
	Map<String, PooledObject> addr2Pool = Maps.newHashMapWithExpectedSize(pools.size());
	for (PooledObject pool: pools) {
		addr2Pool.put(pool.addr, pool);
	}
	ImmutableList.Builder<PooledObject> builder = ImmutableList.builder();
	for (ChildData childData : watcher.getCurrentData()) {
		try {
			CodisProxyInfo proxyInfo = MAPPER.readValue(childData.getData(), CodisProxyInfo.class);
			if (!CODIS_PROXY_STATE_ONLINE.equals(proxyInfo.getState())) {
				continue;
			}
			String addr = proxyInfo.getAddr();
			PooledObject pool = addr2Pool.remove(addr);
			if (pool == null) {
				String[] hostAndPort = addr.split(":");
				String host = hostAndPort[0];
				int port = Integer.parseInt(hostAndPort[1]);
				pool = new PooledObject(addr,
						new JedisPool(poolConfig, host, port, connectionTimeoutMs, soTimeoutMs,
								password, database, clientName, false, null, null, null));
				LOG.info("Add new proxy: " + addr);
			}
			builder.add(pool);
		} catch (Exception e) {
			LOG.warn("parse " + childData.getPath() + " failed", e);
		}
	}
	this.pools = builder.build();
	...
}
```

这个 pools 其实是 redis pool 的集合, 具体的操作流程:

1. 从 Zookeeper 中获取到 codis proxy 的信息. 这个其实不重要, 我们可以把这个换成 etcd 
2. 为所有的 codis proxy 都建立一个 redis pool, 当客户端从某个 codis proxy上取连接的时候, 其实是中这个 codis proxy 的 redis pool 中去取连接
3. 查看 codis pools 中是否有所有的 proxy 的 redis pool, 如果没有的话, 就创建一个放到 codis pools 中

再回到 `getResource()` 函数

1. 通过 RoundRobinJedisPool 类的原子变量 nextIdx 获取上一次从哪个 codis-proxy 的 redis pool 中获取的redis 连接
2. 按照轮询的方式, 计算获取下一次应该去哪个 codis-proxy 的 redis pool 中去获取连接. 这里使用的 compareAndSet lockfree 方法来处理的

#### 看完 Jodis 的实现, 我们如何使用 Redigo 来实现一个 Golang 版本的 codis pool 呢 ?

看到这里其实我们心里应该有个大致的思路了, 总结一下:

**1. 设计一个类 CodisPool**

```go
type CodisPool struct {
	pools 	[]*redigo.Pool
	mux 	sync.Mutex
	index 	int
}
```

**2. 从 Zk 或者 etcd 中获取所有的 codis proxys 信息**

**3. 为每一个 codis-proxy 都建立一个 redis.Pool 放入 pools 中**

```go
for _, proxy := range proxys {
	p, err := NewRedisPool(
		url.Host,
		url.Port,
		url.MaxIdle,
		url.IdleTimeout,
		url.ConnectTimeout,
		url.ReadTimeout,
		url.WriteTimeout,
		url.PoolSize)
	if err != nil {
		log.Printf("%s %d connected failed %s", url.Host, url.Port, err.Error())
		continue
	}
	c.pools = append(c.pools, p)
}
```
**4. 从 codis pool 获取连接时, 通过轮询计算该从哪个 codis-proxy Pool 上获取连接**

```go
func (c *CodisPool) Pool() {
	...
	c.mux.Lock()
	defer c.mux.Lock()

	c.index++
	if  c.LastPoolIdx >= len(c.pools) {
		c.index = 0
	}
	p := c.pools[c.index]

	...
}
```

具体代码目前还不能放出来, 所以提供一个大致思路.