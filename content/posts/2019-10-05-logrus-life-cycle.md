---
title: "Logrus源码阅读(2)--logrus生命周期"
date: 2019-10-05T11:21:24+08:00
lastmod: 2019-10-05T11:21:24+08:00
keywords: [logrus]
tags: [golang, logrus]
categories: [golang]
---

<!--more-->

[上一篇](https://www.haohongfan.com/post/2019-06-11-logurs-1/)介绍logrus的基本用法, 本篇文章介绍logrus的整个生命周期

```golang
func main() {
	log.Info("hello logrus")
}
```

从上面这个简单的例子, 追踪logrus的整个生命周期

## 初始化

```golang
// exported.go:L108
func Info(args ...interface{}) {
	std.Info(args...)
}
```

Info函数的参数是一个可变参数, 接收任意类型的参数

```golang
// exported.go:L11
var (
	// std is the name of the standard logger in stdlib `log`
	std = New()
)

func StandardLogger() *Logger {
	return std
}
```

std是一个全局变量, 是一个`logrus.Logger`类型. 由于std是包外面无法访问的, 所以还提供StandardLogger()函数获取到std

logrus就是初始化一个全局变量`std`, 所有的使用方式都是围绕着这个std来的

上一篇关于logrus的三种使用方式: `logrus.Info`, `logrus.WithField`, `Entry(ctx *gin.Context) *logrus.Entry`本质就是调用全局变量std

**这里留个思考题, 您是否知道golang的初始化流程呢? std全局变量是什么时候被初始化完成的?**

这里还要引申出另外一个问题: 由于我们维护一个全局变量, 但是我们程序是多goroutine的, 当程序多个地方打印日志或者写入文件时, 如何保证日志顺序的正确性, 也就是并发是如何实现的?

### New

从`初始化`那里可以看到std是由`New`函数创建出来的

```go
func New() *Logger {
	return &Logger{
		Out:          os.Stderr,
		Formatter:    new(TextFormatter),
		Hooks:        make(LevelHooks),
		Level:        InfoLevel,
		ExitFunc:     os.Exit,
		ReportCaller: false,
	}
}
```

logrus由`Out`, `Formatter`, `Hooks`, `Level`, `ExitFunc`, `ReportCaller`组成. 关于组件的详细作用, 下面再具体介绍剖析

其实还有两个重要的字段

* MutexWrap: 用来解决并发. // Used to sync writing to the log. Locking is enabled by Default
* entryPool: 用来解决Entry gc压力. // Reusable empty entry

## 调用流程

回到`log.Info("hello logrus")`这个最简单的使用的例子, 追踪下具体的调用过程

![](https://images.haohongfan.com/logrus-liucheng2.png)

```go
// exported.go:L107
// Info logs a message at level Info on the standard logger.
func Info(args ...interface{}) {
	std.Info(args...) // <-- 看这里
}

// logger.go:L205
func (logger *Logger) Info(args ...interface{}) {
	logger.Log(InfoLevel, args...) // <-- 看这里
}

// logger.go:L189
func (logger *Logger) Log(level Level, args ...interface{}) {
	if logger.IsLevelEnabled(level) {
		entry := logger.newEntry()
		entry.Log(level, args...) // <-- 看这里
		logger.releaseEntry(entry)
	}
}

// entry.go:L266
func (entry *Entry) Log(level Level, args ...interface{}) {
	if entry.Logger.IsLevelEnabled(level) {
		entry.log(level, fmt.Sprint(args...)) // <-- 看这里
	}
}

// entry.go:L206
func (entry Entry) log(level Level, msg string) {
	...

	buffer = bufferPool.Get().(*bytes.Buffer)
	buffer.Reset()
	defer bufferPool.Put(buffer)
	entry.Buffer = buffer

	entry.write() // <-- 看这里

	....
}

// entry.go:L252
func (entry *Entry) write() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	serialized, err := entry.Logger.Formatter.Format(entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to obtain reader, %v\n", err)
	} else {
		_, err = entry.Logger.Out.Write(serialized)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Failed to write to log, %v\n", err)
		}
	}
}
```

虽说初始化的时候, New出来的是`Logger`类型, 但logrus真正执行者却是`Entry`. `Logger`有两个比较重要的函数`newEntry`, `releaseEntry`, logrus所有的log函数, 比如: `Info`, `Error`.... 最终都会调用这两个函数

### newEntry && releaseEntry

```go
// logger.go:L90-101
func (logger *Logger) newEntry() *Entry {
	entry, ok := logger.entryPool.Get().(*Entry)
	if ok {
		return entry
	}
	return NewEntry(logger)
}

func (logger *Logger) releaseEntry(entry *Entry) {
	entry.Data = map[string]interface{}{}
	logger.entryPool.Put(entry)
}
```

Logger使用到了`sync.Pool`, 用来解决频繁创建/释放Entry对象造成的gc的压力. 具体位置就是`logger.go:L189`

当我们使用logrus log相关函数时, 必定会调用到`logger.Log()`函数, 该函数会调用`newEntry()`来申请Pool内存, 调用完成后会再调用`releaseEntry()`返还给Pool

注意点:

1. 初始化Logger时, `New`函数没有初始化entryPool, 所以entryPool默认返回的是nil
2. entry, ok := logger.entryPool.Get().(*Entry), 这段代码是先从Pool获取内存, 然后判断获取到的值是否是`*Entry`类型
3. 在第一次调用时由于获取到的值肯定是nil, 故调用了`NewEntry`函数获取了一块内存
4. 不过在调用releaseEntry函数时将这块内存Put到entryPool里, 后面所有调用都是从Pool里面获取

newEntry, releaseEntry这是`Sync.Pool`的另外一种用法, 可以看这里一个具体的简单的[例子](https://play.golang.org/p/74gWnH37-zr)

## Entry

```go
type Entry struct {
	Logger *Logger // 其实就是std指针, 后面再说具体的作用
	Data Fields    // 就是各种WithXXX所带的参数
	Time time.Time // 提供给logrus.WithTime, logrus.WithContext使用
	Level Level    // 日志级别
	Caller *runtime.Frame  // 当设置SetReportCaller时使用, 具体后面再说
	Message string // 真正打印的日志内容
	Buffer *bytes.Buffer // 提供给各种Formatter使用, 其实就是真正要打印的日志的内存地址
	Context context.Context // 提供给logrus.WithTime, logrus.WithContext使用
	err string // 提供一个能够包含错误信息的字段
}
```

```go
// entry.go:L80-86
func NewEntry(logger *Logger) *Entry {
	return &Entry{
		Logger: logger,
		// Default is three fields, plus one optional.  Give a little extra room.
		Data: make(Fields, 6),
	}
}
```

注意到到Data其实就是`map[string]interface{}`, 其预先分配了6个空间(预先给make函数⼀一个合理元素数量参数，有助于提升性能。因为事先申请⼀一⼤大块内存， 可避免后续操作时频繁扩张 -- Go 学习笔记 第四版. 引申: map是否能用cap函数计算其容量? 为什么?)

### WithXXX

几个比较重要的With函数

* WithContext
* WithField
* WithFields
* WithTime
* WithError

考虑到篇幅过长, 这个几个函数具体实现, 下篇介绍logurs高级用法再说

### log

不管程序是否调用WithXXX函数, 最终都会调用Entry.log函数. 这是logrus最重要的函数, Hook机制也就是在这里实现的

```go
func (entry Entry) log(level Level, msg string) {
	var buffer *bytes.Buffer

	// 判断时间是否为空, 如果是空的话, 就设置entry.Time为当前时间
	if entry.Time.IsZero() {
		entry.Time = time.Now()
	}

	entry.Level = level
	entry.Message = msg
	// 设置调用者
	if entry.Logger.ReportCaller {
		entry.Caller = getCaller()
	}

	// 调用Hook
	entry.fireHooks()

	buffer = bufferPool.Get().(*bytes.Buffer)
	buffer.Reset()
	defer bufferPool.Put(buffer)
	entry.Buffer = buffer

	entry.write()

	entry.Buffer = nil

	// 当日志级别是PanicLevel时, 让程序直接panic
	if level <= PanicLevel {
		panic(&entry)
	}
}
```

## Hook

logrus提供了一个很方便的插件功能就是Hook, 其实现原理很简单. 流程调用的地方就是`entry.fireHooks()`

```go
func (entry *Entry) fireHooks() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	err := entry.Logger.Hooks.Fire(entry.Level, entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to fire hook: %v\n", err)
	}
}
```

我们可以根据自己的需求自定义Hook, 但需要实现Hook的interface

```go
type Hook interface {
	Levels() []Level // 用来确定哪些级别的日志, 去调用Hook
	Fire(*Entry) error // 真正执行自定义Hook
}
```

**Hook的使用方法**

1. 实现Levels, Fire函数
2. 调用全局AddHook, 将Hook注册, ok

以`github.com/rifflock/lfshook`举例

```go
package main

import (
	rotatelogs "github.com/lestrrat-go/file-rotatelogs"
	"github.com/rifflock/lfshook"
)

func newLfsHook(fileName string, maxRemainCnt uint) log.Hook {
	writer, err := rotatelogs.New(
		fileName+".%Y%m%d%H%M",
		rotatelogs.WithLinkName("ling_nest_log"),
		rotatelogs.WithRotationTime(time.Hour*time.Duration(config.Config.GetInt("log.time"))),
		rotatelogs.WithRotationCount(maxRemainCnt),
	)

	if err != nil {
		log.Errorf("config local file system for logger error: %v", err)
	}

	lfsHook := lfshook.NewHook(lfshook.WriterMap{
		log.DebugLevel: writer,
		log.InfoLevel:  writer,
		log.WarnLevel:  writer,
		log.ErrorLevel: writer,
		log.FatalLevel: writer,
		log.PanicLevel: writer,
	}, &log.JSONFormatter{})

	return lfsHook
}

func main() {
	fileName := "log.txt"
	logrus.AddHook(newLfsHook(fileName, 100))
	logrus.Info("xxxx")
}
```

值得注意的是: **由于logrus本身并不提供写文件, 并且按照日期自动分割, 删除过期日志文件的功能. 一般情况下大家都是使用`github.com/rifflock/lfshook`配合`github.com/lestrrat-go/file-rotatelogs`来实现相关的功能**

原理:

```go
type Logger struct {
	...

	Hooks LevelHooks

	...
}
```

```go
type LevelHooks map[Level][]Hook
func (hooks LevelHooks) Add(hook Hook) {
	for _, level := range hook.Levels() {
		hooks[level] = append(hooks[level], hook)
	}
}

func (hooks LevelHooks) Fire(level Level, entry *Entry) error {
	for _, hook := range hooks[level] {
		if err := hook.Fire(entry); err != nil {
			return err
		}
	}

	return nil
}
```

1. 调用AddHook时, 将Hook加入到LevelHooks map中
2. 程序打印log, 会最终执行到`Entry.log()`
3. Entry.log()会调用`fireHooks()`
4. fireHooks又会调用LevelHooks Fire()函数, 该函数会遍历所有的Hook, 从而执行相应的Hook

## ReportCaller

```go
import (
	log "github.com/sirupsen/logrus"
)

func main() {
	log.SetReportCaller(true)
	log.Info("hello logrus")
}
```

输出:

`INFO[0000]/Users/haohongfan/goproject/test/logrus_test/main.go:37 main.main() hello logrus`

对比不开启`ReportCaller`的日志, 多了下面字段:

1. 日志打印的文件名字
2. 日志打印的行号
3. 日志打印的函数名字

关于如何实现的, 推荐先看鸟窝博客- <<如何在Go的函数中得到调用者函数名?>>

其实ReportCaller主要是提供给Formatter使用的, 例如: JSONFormatter

```go
// json_formatter.go:L91-103
if entry.HasCaller() {
	funcVal := entry.Caller.Function
	fileVal := fmt.Sprintf("%s:%d", entry.Caller.File, entry.Caller.Line)
	if f.CallerPrettyfier != nil {
		funcVal, fileVal = f.CallerPrettyfier(entry.Caller)
	}
	if funcVal != "" {
		data[f.FieldMap.resolve(FieldKeyFunc)] = funcVal
	}
	if fileVal != "" {
		data[f.FieldMap.resolve(FieldKeyFile)] = fileVal
	}
}
```

由于打算结合logrus的实现, 出一篇介绍golang如何获取调用者的文件名/函数名/行号等等, 及其实现的原理的文章, 就不在继续扩展了

这里简单说下logrus的实现过程.

规则:

1. 当设置SetReportCaller(true)时, 会最终在Entry.log()函数调用`entry.Caller = getCaller()`
2. `getCaller()`函数有个`callerInitOnce` sync.Once变量, 在第一次被调用时会获取logrus的包名字是`github.com/sirupsen/logrus`
3. 紧接着调用`runtime.CallersFrames`获取到所有函数调用栈
4. 然后比对函数栈的package名字, 与`github.com/sirupsen/logrus`相比, 如果不相等, 则是去掉logrus包的第一个调用者; 否则continue

比如:

```go
func main() {
	log.SetReportCaller(true)
	log.Info("hello logrus")
}
```

函数调用栈的顺序是:

1. github.com/sirupsen/logrus.(*Logger).Log
2. github.com/sirupsen/logrus.(*Logger).Info
3. github.com/sirupsen/logrus.Info
4. main.main

按照上面的规则, 由于1,2,3获取到的package包名都是github.com/sirupsen/logrus, 故continue, 最终获取到的第一个函数是main.main的`*runtime.Frame`. Frame包含着文件名, 函数名, 行号等等

我们回过头看logrus获取调用者的这个实现. 是靠着完全遍历匹配package名来获取调用者的. 先抛去runtime.Caller等相关的函数是否慢的问题, 单说这个完全匹配的过程已经浪费了大量时间处理这个事情. **所以我们日志在release版本下还是尽量不要开启这个选项, logrus也不建议使用这开启这个选项**

## write

logrus另外一个非常重要的函数

```go
// entry.go:L252-264
func (entry *Entry) write() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	serialized, err := entry.Logger.Formatter.Format(entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to obtain reader, %v\n", err)
	} else {
		_, err = entry.Logger.Out.Write(serialized)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Failed to write to log, %v\n", err)
		}
	}
}
```

看着很简单, 其实包含的内容还是挺多的: `Formatter`, `Out`

### Formatter

```go
type Formatter interface {
	Format(*Entry) ([]byte, error)
}
```

由于Formatter是个接口类型, 故可以根据自己的需求, 实现自己的Formatter, 只需要实现对应的Format函数即可

继续查看Formatter的具体调用过程(暂且不管Mutex的问题)

```go
// 执行Formatter的地方
func (entry *Entry) write() {
	...

	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	serialized, err := entry.Logger.Formatter.Format(entry)

	...
}
```

```go
// 设置Formatter的地方
// SetFormatter sets the standard logger formatter.
func SetFormatter(formatter Formatter) {
	std.SetFormatter(formatter)
}
```

在调用logrus.SetFormatter()函数后, `Logger`的`Formatter`字段就被设置为你想使用的`XXXFormatter`了, 如果没有设置那么就是默认的`TextFormatter`

程序执行到`entry.Logger.Formatter.Format(entry)`时, 就会执行具体的XXXFormatter的Format函数, 从而执行具体的序列化过程

这里由于篇幅限制只解析比较简单的`JSONFormatter`, 这个其实经常被用到的Formatter.

#### JSONFormatter

```go
// json_formatter.go:L24-54
type JSONFormatter struct {
	TimestampFormat string // 设置Formatter时间格式
	DisableTimestamp bool  // 控制序列化时是否显示时间
	DataKey string // 配合主要是配合WithFields使用
	FieldMap FieldMap // 其实用处很小, 就是让用户自定义序列化字段的名字
	CallerPrettyfier func(*runtime.Frame) (function string, file string) // 配合SetReportCaller, 不需要太关注
	PrettyPrint bool // 让Json格式化输出
}
```

**主要字段介绍**

##### 1.TimestampFormat

Time的时间格式, 设置JSONFormatter TimestampFormat字段时就可以选择下面这些常量. 默认值:time.RFC3339
```
ANSIC       = "Mon Jan _2 15:04:05 2006"
UnixDate    = "Mon Jan _2 15:04:05 MST 2006"
RubyDate    = "Mon Jan 02 15:04:05 -0700 2006"
RFC822      = "02 Jan 06 15:04 MST"
RFC822Z     = "02 Jan 06 15:04 -0700" // RFC822 with numeric zone
RFC850      = "Monday, 02-Jan-06 15:04:05 MST"
RFC1123     = "Mon, 02 Jan 2006 15:04:05 MST"
RFC1123Z    = "Mon, 02 Jan 2006 15:04:05 -0700" // RFC1123 with numeric zone
RFC3339     = "2006-01-02T15:04:05Z07:00"
RFC3339Nano = "2006-01-02T15:04:05.999999999Z07:00"
Kitchen     = "3:04PM"
// Handy time stamps.
Stamp      = "Jan _2 15:04:05"
StampMilli = "Jan _2 15:04:05.000"
StampMicro = "Jan _2 15:04:05.000000"
StampNano  = "Jan _2 15:04:05.000000000
```

##### 2.DataKey

```go
func main() {
	log.SetFormatter(&log.JSONFormatter{
		DataKey: "hhf",
	})
	log.WithFields(log.Fields{"k1": "v1"}).Info("hello logrus")
}
```

输出:

`{"hhf":{"k1":"v1"},"level":"info","msg":"hello logrus","time":"2019-10-09T13:31:05+08:00"}`

当没有注释掉`DataKey: "hhf"`时, 输出就会变成下面

`{"k1":"v1","level":"info","msg":"hello logrus","time":"2019-10-09T13:32:26+08:00"}`

其实就是用`DataKey`来包装一下WithFields的k-v字段

##### 3.FieldMap

```go
func main() {
	log.SetFormatter(&log.JSONFormatter{
		FieldMap: log.FieldMap{
			log.FieldKeyTime:  "@timestamphhf",
			log.FieldKeyLevel: "@levelhhf",
			log.FieldKeyMsg:   "@messagehhf",
			log.FieldKeyFunc:  "@callerhhf",
		},
	})
	log.WithFields(log.Fields{"k1": "v1"}).Info("hello logrus"
}
```

输出:

```
{"@levelhhf":"info","@messagehhf":"hello logrus","@timestamphhf":"2019-10-09T13:42:09+08:00","k1":"v1"}
```

主要的key有下面这几种类型

```
FieldKeyMsg            = "msg"
FieldKeyLevel          = "level"
FieldKeyTime           = "time"
FieldKeyLogrusError    = "logrus_error"
FieldKeyFunc           = "func"
FieldKeyFile           = "file
```

##### 4.PrettyPrint

```go
func main() {
	log.SetFormatter(&log.JSONFormatter{PrettyPrint: true})
	log.WithFields(log.Fields{"k1": "v1"}).Info("hello logrus")
}
```

输出结果:

```
{
  "k1": "v1",
  "level": "info",
  "msg": "hello logrus",
  "time": "2019-10-09T13:52:18+08:00"
}
```

#### Format

```go
// json_formatter.go:L57-121
func (f *JSONFormatter) Format(entry *Entry) ([]byte, error) {
	// 将Entry的WithFields的kv值遍历放入到map[string]interface{}类型的data中
	data := make(Fields, len(entry.Data)+4)
	for k, v := range entry.Data {
		switch v := v.(type) {
		case error:
			// Otherwise errors are ignored by `encoding/json`
			// https://github.com/sirupsen/logrus/issues/137
			data[k] = v.Error()
		default:
			data[k] = v
		}
	}

	// 判断是否存在DataKey, 如果是就用DataKey包装一下data
	if f.DataKey != "" {
		newData := make(Fields, 4)
		newData[f.DataKey] = data
		data = newData
	}

	// 该函数判断调用WithFields时, 当用户自定义的Key与logrus内置的key相同时, 
	// 用户自定义的key会转换成fields.xx. 例如:logrus.WithField("level", 1).Info("hello"), 
	// 由于level跟内置的FieldKeyLevel冲突了, 那么输出就会变成
	// {"level": "info", "fields.level": 1, "msg": "hello", "time": "..."}
	prefixFieldClashes(data, f.FieldMap, entry.HasCaller())

	// 设置时间的序列化方式
	timestampFormat := f.TimestampFormat
	if timestampFormat == "" {
		timestampFormat = defaultTimestampFormat
	}

	// 判断entry的error是否有值, 进行相关的序列化
	if entry.err != "" {
		data[f.FieldMap.resolve(FieldKeyLogrusError)] = entry.err
	}

	// 判断是否禁用Timestamp, 如果不禁用, 就将时间戳按照相应的格式序列化. entry.Time在entry.log()函数里进行了初始化
	// if entry.Time.IsZero() {
	// 	entry.Time = time.Now()
	// }
	if !f.DisableTimestamp {
		data[f.FieldMap.resolve(FieldKeyTime)] = entry.Time.Format(timestampFormat)
	}

	// 设置日志的具体内容
	data[f.FieldMap.resolve(FieldKeyMsg)] = entry.Message
	// 设置日志级别
	data[f.FieldMap.resolve(FieldKeyLevel)] = entry.Level.String()
	// 序列化调用位置
	if entry.HasCaller() {
		funcVal := entry.Caller.Function
		fileVal := fmt.Sprintf("%s:%d", entry.Caller.File, entry.Caller.Line)
		if f.CallerPrettyfier != nil {
			funcVal, fileVal = f.CallerPrettyfier(entry.Caller)
		}
		if funcVal != "" {
			data[f.FieldMap.resolve(FieldKeyFunc)] = funcVal
		}
		if fileVal != "" {
			data[f.FieldMap.resolve(FieldKeyFile)] = fileVal
		}
	}

	// entry.Buffer是在entry.log()函数里(entry.go:L226-229)从sync.Pool里获取到一块内容空间
	// 目的是: 防止JSONFormatter每次调用都会去申请空间, 减小GC压力
	var b *bytes.Buffer
	if entry.Buffer != nil {
		b = entry.Buffer
	} else {
		b = &bytes.Buffer{}
	}

	// 将Buffer提供给json encoder使用
	encoder := json.NewEncoder(b)
	if f.PrettyPrint {
		encoder.SetIndent("", "  ")
	}
	if err := encoder.Encode(data); err != nil {
		return nil, fmt.Errorf("failed to marshal fields to JSON, %v", err)
	}

	// 序列化完成, 将序列化的内容返回
	return b.Bytes(), nil
}
```

### Out

Formatter介绍完了, 回到上面write()函数继续剖析Out相关

```go
// entry.go:L252-264
func (entry *Entry) write() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	serialized, err := entry.Logger.Formatter.Format(entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to obtain reader, %v\n", err)
	} else {
		_, err = entry.Logger.Out.Write(serialized)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Failed to write to log, %v\n", err)
		}
	}
}
```

在没有调用SetOutput时, 默认的Out是`os.Stderr`, 所以默认情况下基本都打印到终端里, 没有存入文件

即使logrus可以提供io.Writter, 但是还是不建议在这里将日志落盘, 还是使用`lfsHook`来做这个事情

在这里也回答上面一个问题, **为什么在`Entry`结构体里面会有`Logger`指针存在?**

答: 以Out举例, 由于我们在调用logrus.SetOutput()函数时, Out是设置给Logger的, 但是真正的使用者却是`Entry`. 故需要将Logger传给Entry一份

## logrus如何保证并发的正确性

logrus的并发控制的正确性是靠着Logger.Mutex来实现的. 程序中调用Logger.Mutex地方有几处:

1. fireHooks() entry.go:L243
2. write() entry.go:L252
3. AddHook() logger.go:L313
4. ReplaceHook() logger.go:L345
5. SetFormatter() logger.go:L235
6. SetNoLock() logger.go:L294
7. SetOutput() logger.go:L332
8. SetReportCaller() logger.go:338

最重要的两处就是fireHooks(), write()

```go
func (entry *Entry) fireHooks() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	err := entry.Logger.Hooks.Fire(entry.Level, entry)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to fire hook: %v\n", err)
	}
}
```

```go
func (entry *Entry) write() {
	entry.Logger.mu.Lock()
	defer entry.Logger.mu.Unlock()
	serialized, err := entry.Logger.Formatter.Format(entry)
	....
}
```

可以观察到, 不管有多少goroutine在调用logrus, 都是靠着资源竞争来保证顺序的正确性

查看整个logrus的源码, logrus只有一个goroutine顺序处理日志数据, 并且没有相关的buffer来保存日志信息, 这就造成logrus的整体效率是不高的

后面一篇文章会专门对比`zap`之间的差别, 敬请期待

## 总结

至此, logrus的主体源码已经解析完毕

一句话总结其生命周期就是: logrus是在编译期就确定的一个全局变量, 伴随着我们程序的整个生命周期而存在. 最重要的组件是: Formatter, Hook. 良好的序列化机制, 方便的插件开发是我们选择logrus的原因

## 参考文档

1. 鸟窝博客 - 如何在Go的函数中得到调用者函数名? https://colobu.com/2018/11/03/get-function-name-in-go/
2. logrus https://github.com/sirupsen/logrus
3. file-rotatelogs https://github.com/lestrrat-go/file-rotatelogs
4. lfshook https://github.com/rifflock/lfshook
5. dingrus https://github.com/dandans-dan/dingrus