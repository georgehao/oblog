---
title: Bilibili Kratos 框架源码分析(4) -- Kratos Log
date: 2020-06-09T10:14:07+08:00
lastmod: 2020-06-09T10:14:07+08:00
keywords: [golang, kratos]
tags: [golang, kratos]
categories: [golang, kratos]
---

## 用法

| flag       | env        | type   | remark                                                     |
| :--------- | :--------- | :----- | :--------------------------------------------------------- |
| log.v      | LOG_V      | int    | 日志级别：DEBUG:0 INFO:1 WARN:2 ERROR:3 FATAL:4            |
| log.stdout | LOG_STDOUT | bool   | 是否标准输出：true、false                                  |
| log.dir    | LOG_DIR    | string | 日志文件目录，如果配置会输出日志到文件，否则不输出日志文件 |
| log.module | LOG_MODULE | string | 可单独配置每个文件的verbose级别：file=1,file2=2            |
| log.filter | LOG_FILTER | string | 过虑敏感信息 format: field1,field2.                        |

被阉割的功能: log.agent

## 配置文件

```toml
[log]
	family = "xxx-service"
	dir = "/data/log/xxx-service/"
	stdout = true
	v = 3
	filter = ["fileld1", "field2"]
[log.module]
	"dao_user" = 2
	"servic*" = 1
```

对应的加载方法

```go
// cmd/main.go
func main() {
	paladin.Init()
	var cfg struct {
		Log *log.Config
	}
	if err := paladin.Get("application.toml").UnmarshalTOML(&cfg); err != nil {
		return
	}
	// init log
	log.Init(cfg.Log) // debug flag: log.dir={path}
	defer log.Close()
}
```

### 最简单用法

Kratos Log 有 4 种级别的打印方式: log.Info(), log.Warn(), log.Error(), 以及一个log.V(4).Info(). 如果开启文件打印后, 会分别对应info.log, warn.log, error.log

```go
func main() {
  flag.Parse()
  log.Init(nil)
  log.Info("hi:%s", "kratos")
  log.Infoc(Context.TODO(), "hi:%s", "kratos")
  log.Infov(Context.TODO(), log.KVInt("key1", 100), log.KVString("key2", "test value")
  // Warn, Error 这里就不写了, 跟 Info() 一样
  log.V(5).Info("xxxx")
}
```

### log.v

kratos log 日志级别(level)跟其他我们熟知的日志框架(logrus, zap)不太一样.

以 logrus 为例:

```go
package main
import (
  log "github.com/sirupsen/logrus"
)

func main() {
	log.SetLevel(log.WarnLevel)
  	log.WithFields(log.Fields{
    	"animal": "walrus",
    	"size":   10,
	}).Info("A group of walrus emerges from the ocean")
}
```

当 `SetLevel` 后只会打印这个 Level 高的日志. 但是 Kraots 使用 log.Info(), log.Warn(), log.Error() 却没有这个功能. 也就是说只要程序里面使用就会输出.

当然 Kratos 也提供了类似 logrus 的这种对应用法, 但是感觉有点另类. 需要配置文件设置`log.v` 配合 log.V(4).Info()使用. 如:

```go
log.V(5).Info("xxx")
```

如果 `log.v` 设置是 5, 那么能打印`xxx`. 如果 `log.v` 小于 5, 则不能打印. 也就是说 log.V 只能打印比 log.v 小的日志

#### log.module

这个功能我觉得没啥用处. 具体的功能就是:

配置文件 `log.v` 设置是 3. 按照上面说的, log.V(5).Info("xx") 是肯定打印不出来的. 但是可以通过 `log.module` 来另外定义某些文件特殊打印级别

```
[log]
	v = 3
[log.module]
	"dao_user" = 5
	"servic*" = 1
```

在 dao_user.go 里面, log.V(5).Info("dao_user"), 日志照样会打印 "dao_user". `dao_user`, `servic*`是对应的文件的名字

## log.stdout

这个简单, 就是控制日志是否打印在命令行里面. 生产环境需要关闭改选项(默认关闭), 在开发时打开这个是很有用的

## log.dir

设置写日志文件的路径

## log.filter

这个还是蛮有意思的功能. 比如打印日志的时候, `密码`等敏感信息一般是不能打印在日志文件里. 该功能就能将这些信息隐藏掉

```toml
[log]
filter = ["password"]
```

```go
log.Infov(context.Background(), log.KV("password", "123456"))
```

打印出来就会变成`[2020/06/08 23:19:55.490] [INFO] [/demo1/cmd/main.go:21] password=***`

整体上 kratos log 就提供了这么多功能, 下面进入源码分析

## 源码分析

### 加载 Init

其实加载没啥说的, 主要就是调用Init()函数

```go
func Init(conf *Config) {
	var isNil bool
	if conf == nil {
		isNil = true
		conf = &Config{
			Stdout: _stdout,
			Dir:    _dir,
			V:      int32(_v),
			Module: _module,
			Filter: _filter,
		}
	}
	if len(env.AppID) != 0 {
		conf.Family = env.AppID // for caster
	}
	conf.Host = env.Hostname
	if len(conf.Host) == 0 {
		host, _ := os.Hostname()
		conf.Host = host
	}
	var hs []Handler
	// when env is dev
	if conf.Stdout || (isNil && (env.DeployEnv == "" || env.DeployEnv == env.DeployEnvDev)) || _noagent {
		hs = append(hs, NewStdout())
	}
	if conf.Dir != "" {
		hs = append(hs, NewFile(conf.Dir, conf.FileBufferSize, conf.RotateSize, conf.MaxLogFile))
	}
	h = newHandlers(conf.Filter, hs...)
	c = conf
}
```

作用: 

1. 初始化文章一开头提到那五个功能: log.v, log.stdout, log.dir, log.module, log.filter
2. 初始化 stdout, file handler

### Field

> 基于zap的field方式实现的高性能log库，提供Info、Warn、Error日志级别；
> 并提供了context支持，方便打印环境信息以及日志的链路追踪，在框架中都通过field方式实现，避免format日志带来的性能消耗

上面这段摘自官方wiki, 但是看完源码之后, 发现跟 zap 毛线关系没有, 就用了 zap 的一个结构体

```go
type Field struct {
	Key       string
	Value     interface{}
	Type      FieldType
	StringVal string
	Int64Val  int64
}
```

其实想不太明白, 就用了一个结构体(也没有用任何 zap 为该结构体提供的函数), 为何将 zap 那么多代码复制了过来, 差评

Kartos log 没有采用类似 logrus 或者 Zap json 或者 text 序列化的方案, 就以上面的 Field 为基础扩展一个 KVString, KVInt, .... KVDuration 结构. 在写日志时候, 我们就可以像下面这样打印多个kv

```
log.Infov(Context.TODO(), log.KVInt("key1", 100), log.KVString("key2", "test value")
```

### Handler

上面的log.InfoV(xxx) 将会调用下面的 Log() 函数

```go
func (hs Handlers) Log(ctx context.Context, lv Level, d ...D) {
	hasSource := false
	for i := range d {
		if _, ok := hs.filters[d[i].Key]; ok {
			d[i].Value = "***" // 实现log.filter
		}
		if d[i].Key == _source {
			hasSource = true
		}
	}
	if !hasSource {
		fn := funcName(3)
		errIncr(lv, fn)
		d = append(d, KVString(_source, fn))
	}
	d = append(d, KV(_time, time.Now()), KVInt64(_levelValue, int64(lv)), KVString(_level, lv.String()))
	for _, h := range hs.handlers {
		h.Log(ctx, lv, d...)
	}
}
```

Kratos log 提供的所有Info(), Warn(), Error() 系列函数最终都会调用到这个函数上面来. 

这个函数的主要作用就是:

1. 实现 log.filter 功能
2. 增加一些额外的字段, 如: time, level, source Field
3. 根据初始化 log 的时候是否启动 stdout 或者 file handler, 来调用对应的handler 的 Log 函数做最终处理

### handlers (stdout, file)

上面的Handler Log() 里面有这么句代码. 就是具体调用 stdout或者file handlers

```go
for _, h := range hs.handlers {
	h.Log(ctx, lv, d...)
}
```

不过在调用具体的 handler 来打印的时候, 需要对相关的 Field 来做处理, 分别会调用下面 toMap, addExtraField 函数, 将处理结果都放在一个 map[string]interface{}里

#### toMap

```go
func toMap(args ...D) map[string]interface{} {
	d := make(map[string]interface{}, 10+len(args))
	for _, arg := range args {
		switch arg.Type {
		case core.UintType, core.Uint64Type, core.IntTpye, core.Int64Type:
			d[arg.Key] = arg.Int64Val
		case core.StringType:
			d[arg.Key] = arg.StringVal
		case core.Float32Type:
			d[arg.Key] = math.Float32frombits(uint32(arg.Int64Val))
		case core.Float64Type:
			d[arg.Key] = math.Float64frombits(uint64(arg.Int64Val))
		case core.DurationType:
			d[arg.Key] = time.Duration(arg.Int64Val)
		default:
			d[arg.Key] = arg.Value
		}
	}
	return d
}
```

其实这段代码没啥意思, 就是将 KVInt(), KVString()系列函数对应的Val从对应的字段上取出来

#### addExtraField

```go
func addExtraField(ctx context.Context, fields map[string]interface{}) {
	if t, ok := trace.FromContext(ctx); ok {
		fields[_tid] = t.TraceID()
	}
	if caller := metadata.String(ctx, metadata.Caller); caller != "" {
		fields[_caller] = caller
	}
	if color := metadata.String(ctx, metadata.Color); color != "" {
		fields[_color] = color
	}
	if env.Color != "" {
		fields[_envColor] = env.Color
	}
	if cluster := metadata.String(ctx, metadata.Cluster); cluster != "" {
		fields[_cluster] = cluster
	}
	fields[_deplyEnv] = env.DeployEnv
	fields[_zone] = env.Zone
	fields[_appID] = c.Family
	fields[_instanceID] = c.Host
	if metadata.String(ctx, metadata.Mirror) != "" {
		fields[_mirror] = true
	}
}
```

这段代码就是另外再加几个字段, 如果 env, zone, appid, instance_id, caller, tid

### Pattern

经过上面的操作基本上操作, map[string]interface{} 里面大概有这些内容

```
test1 -> test1
source -> /learn_kratos/demo1/cmd/main.go:21
time -> 2020-06-09T19:07:36.3063
level_value -> 1
level -> INFO
app_id -> xxx-service
instance_id -> localhost
env -> dev
zone -> zone01
```

接着要做的就是将他们写成字符串(对, kratos 只提供了TextFormater方式)


```go
var patternMap = map[string]func(map[string]interface{}) string{
	"T": longTime,
	"t": shortTime,
	"D": longDate,
	"d": shortDate,
	"L": keyFactory(_level),
	"f": keyFactory(_source),
	"i": keyFactory(_instanceID),
	"e": keyFactory(_deplyEnv),
	"z": keyFactory(_zone),
	"S": longSource,
	"s": shortSource,
	"M": message,
}

....

for _, f := range p.funcs {
	builder.WriteString(f(d))
}
```

这里的实现方式还是蛮巧妙的, 可以看看

### stdout, filewriter

经过上面的处理, 就进入了最后的输出阶段了, 分别对应 stdout, filewirter

鉴于篇幅有限, filewriter 本篇就不再介绍, 另外再出一篇吧

## 总结

优点: kratos log整体采用 Kv 结构来处理, 结构比较清晰, 功能简单, 没有序列化的压力, 追踪日志需要的字段全都有.

缺点: 只支持TextFormater, 日志级别控制跟当前主流 Log 框架使用习惯不太一样