---
title: "打造 Zap 开箱即用日志组件"
date: 2019-12-15T22:49:01+08:00
lastmod: 2019-12-15T22:49:01+08:00
keywords: [golang, zap, log]
tags: [golang]
categories: [golang]
---

logrus 是 golang 一款非常优秀的日志框架, 其优点非常明显:

* 优雅的代码框架设计, 可以作为我们设计组件的参考. 具体请参见我前面文章(链接文末给出)
* 使用简单
* 组件化的开发思路
* 灵活的输出方式

但是, 性能终究是忍痛舍弃 logrus 的“阿喀琉斯之踵”, 前面的[文章](https://www.haohongfan.com/post/2019-10-05-logrus-life-cycle/)深入研究了 logrus 性能低的原因

目前 golang 日志库的大众选择主要集中在: logrus, zap, zerolog. zap 和 zerolog 的性能都是优秀的, 但是从用法习惯上我更倾向于 zap.

## 简单介绍 Zap 的使用

Zap 提供三种不同方式的输出(以 Info为 例)

```go
log.Info("hello zap") // {"level":"info","ts":1576423173.016333,"caller":"test_zap/main.go:28","msg":"hello zap"}
log.Infof("hello %s", "zap") // {"level":"info","ts":1576423203.056074,"caller":"test_zap/main.go:29","msg":"hello zap"}
log.Infow("hello zap", "field1", "value1") //{"level":"info","ts":1576423203.0560799,"caller":"test_zap/main.go:30","msg":"hello zap","field1":"value1"}
``` 

如果我们对 logrus 的 key-value 理论比较在意的话, 使用 zap infow 可以完美解决

## Zap 使用起来不便利的地方

1. Zap 使用上不能像 logrus 那样开箱即用
2. 使用者需要自己去组装相关函数
3. Zap 同样不提供日志切割的功能, 但是想添加上这个功能没有 logrus 那样便利

基于这些问题, 我封装了一套开箱即用的日志组件: https://github.com/georgehao/log

## 打造 Zap 开箱即用日志组件

提供的功能:

* 像 logrus 一样, 全局的 Debug, Info ... 函数
* 日志分割功能. 默认文件大小1024M，自动压缩, 最大有3个文件备份，备份保存时间7天, 不会打印日志被调用的文文件名和位置
* 日志默认会被分成五类文件：xxx.log.DEBUG，xxx.log.INFO, xxx.log.WARN, xxx.log.ERROR, xxx.log.Request. error, panic. fatal 都会打印在xxx.log.ERROR. xxx.log.Request输出 request log 的地方(如果有需要的话)


### 框架图

![log](https://images.haohongfan.com/log.png)

### 使用方法


```
go get github.com/georgehao/log
```

#### 例子

```go
package main

import "github.com/georgehao/log"

func main() {
	// init log
	// set absolute path, and level
	// set output level
	// don't need request log
	// set log's caller using logOption
	log.Init("./test.log", log.DebugLevel, false, log.SetCaller(true))
	log.Info("hello george log")
	// flush
	log.Sync()
	//output: {"level":"info","ts":"2019-12-16T10:37:11.364+0800","caller":"example/example.go:12","msg":"hello george log"}
}
```

## 参考

1. [Logrus源码阅读(1)--基本用法](https://www.haohongfan.com/post/2019-06-11-logurs-1/)
2. [Logrus源码阅读(2)--logrus生命周期](https://www.haohongfan.com/post/2019-10-05-logrus-life-cycle/)