---
title: "Bilibili Kratos 框架源码分析(1) -- 启动流程"
date: 2020-05-08T14:42:38+08:00
lastmod: 2020-05-08T14:42:38+08:00
keywords: [golang, kratos]
tags: [golang, kratos]
categories: [golang, kratos]
---

![](https://raw.githubusercontent.com/go-kratos/kratos/master/doc/img/kratos.png)

这里先吐槽一下 kratos 官方 wiki 写的实在不咋地, 一些很基本的使用方法, 一些很好的功能都没有体现出现, 同时也建议多去 github issue 里去找找答案, 那里面比 wiki 详细很多. 

这个系列的文章我会基于 v0.4.2 这个版本的源码进行. 现在正式进入这个系列源码的第一篇: Kratos 启动流程

## 安装 kratos

至于如何安装 kratos, 请参考 官方[wiki](https://github.com/go-kratos/kratos#quick-start), 

Kratos 官方推荐方式: `GO111MODULE=on && go get -u github.com/go-kratos/kratos/tool/kratos`, 不过有可能你依然无法安装成功. 

特别是 `bilibili/kratos` 迁移到 `go-kratos/kratos` 之后, 如果你之前安装过 kratos, 使用上面的方式基本就 gg 了, 会遇到下面问题 `module declares its path as: github.com/go-kratos/kratos but was required as: github.com/bilibili/kratos`

解决方案:

* 方案1: kratos 维护人员推荐的方案, 将整个代码拷贝下来, `cd kratos/tool && go install ./...`
* 方案2: 彻底清理以前安装的 kratos 相关的组件(kratos, kratos-gen-bts, kratos-gen-mc, kratos-gen-project, kratos-protoc, testgen, swagger, wire, testcli), 然后`GO111MODULE=on && go get -u github.com/go-kratos/kratos/tool/kratos` 应该就能安装成功了, 如果还不成功, `多试几次`或者尝试 方案1

## 初始化demo

使用 `kratos new kratos-demo` 生成一个demo项目

```
cd kratos-demo/cmd
go build
./cmd -conf ../configs
```

打开浏览器访问：`http://localhost:8000/kratos-demo/start`，你会看到输出了Golang 大法好 ！！！

## 整体流程

接下来看下程序的启动流程, 主函数在 cmd/main.go 中

```go
func main() {
	flag.Parse()
	log.Init(nil) // debug flag: log.dir={path} ---> 启动log
	defer log.Close()
	log.Info("kratos-demo start")
	paladin.Init()  // ----> 启动配置文件
	_, closeFunc, err := di.InitApp() // ----> 利用依赖注入, 初始化各个组件
	if err != nil {
		panic(err)
	}
	c := make(chan os.Signal, 1)  // ---> 优雅重启
	signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
	for {
		s := <-c
		log.Info("get a signal %s", s.String())
		switch s {
		case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
			closeFunc()
			log.Info("kratos-demo exit")
			time.Sleep(time.Second)
			return
		case syscall.SIGHUP:
		default:
			return
		}
	}
}
```

1. 启动 log
2. 启动启动配置文件 [config]
3. 利用依赖注入(wire), 初始化各个组件(redis, mc, db, dao, service, http server, grpc server), 并将整体串联起来
4. 优雅重启


### 1. 启动 log

这里先不对 log 做详细的分析, 后面连载章节介绍

> 官方介绍:
> 
> 基于zap的field方式实现的高性能log库，提供Info、Warn、Error日志级别；
> 并提供了context支持，方便打印环境信息以及日志的链路追踪，在框架中都通过field方式实现，避免format日志带来的性能消耗。

### 2. 启动启动配置文件

这里先不对 配置文件 做详细的分析, 后面连载章节介绍

> 官方介绍
> 
> 初看起来，配置管理可能很简单，但是这其实是不稳定的一个重要来源。
> 即变更管理导致的故障，我们目前基于配置中心（config-service）的部署方式，二进制文件的发布与配置文件的修改是异步进行的，每次变更配置，需要重新构建发布版。
> 
> 由此，我们整体对配置文件进行梳理，对配置进行模块化，以及方便易用的paladin config sdk

### 3. 利用依赖注入初始化各个组件

想要了解这个过程, 需要先简单了解 wire 的运作流程. 关于 wire 库是如何使用的, 我这里就不做 demo 过多的介绍, 推荐查看公众号: **GoUpUp**  [Go 每日一库之 wire](https://mp.weixin.qq.com/s/GFN6PWg6hfSFvPkWqHwNKw)

下面直接进入 kratos 的依赖注入过程


#### Provider(构造器)

`internal/dao/dao.go` 有下面一段代码: 

```go
var Provider = wire.NewSet(New, NewDB, NewRedis, NewMC)
```

 `internal/service/service.go` 也有差不多的一段代码:

 ```go
 var Provider = wire.NewSet(New, wire.Bind(new(pb.DemoServer), new(*Service)))
 ```

#### Injector(注入器)

```go
// +build wireinject ----> 尤其要注意这里
// The build tag makes sure the stub is not built in the final build.

package di

import (
	"kratos-demo/internal/dao"
	"kratos-demo/internal/service"
	"kratos-demo/internal/server/grpc"
	"kratos-demo/internal/server/http"

	"github.com/google/wire"
)

//go:generate kratos t wire
func InitApp() (*App, func(), error) {
	panic(wire.Build(dao.Provider, service.Provider, http.New, grpc.New, NewApp))
}
```

特别要注意: `// +build wireinject` , +build不是一个注释这么简单, 其实是 Go 语言的一个特性。类似 C/C++ 的条件编译，在执行go build时可传入一些选项，根据这个选项决定某些文件是否编译(引用Go 每日一库之 wire), 还要注意 **注释跟 package 之间要有空行(https://github.com/google/wire/issues/117)**


#### 几个非常重要的 New

```go
// NewApp 串联整个kraots的组件
func NewApp(svc *service.Service, h *bm.Engine, g *warden.Server) (app *App, closeFunc func(), err error){
	app = &App{
		svc: svc,
		http: h,
		grpc: g,
	}
	closeFunc = func() {
		ctx, cancel := context.WithTimeout(context.Background(), 35*time.Second)
		if err := g.Shutdown(ctx); err != nil {
			log.Error("grpcSrv.Shutdown error(%v)", err)
		}
		if err := h.Shutdown(ctx); err != nil {
			log.Error("httpSrv.Shutdown error(%v)", err)
		}
		cancel()
	}
	return
}

// http.New 初始化和启动 Http Server(gin)
func New(s pb.DemoServer) (engine *bm.Engine, err error) {
	var (
		cfg bm.ServerConfig
		ct paladin.TOML
	)
	if err = paladin.Get("http.toml").Unmarshal(&ct); err != nil {
		return
	}
	if err = ct.Get("Server").UnmarshalTOML(&cfg); err != nil {
		return
	}
	svc = s
	engine = bm.DefaultServer(&cfg)
	pb.RegisterDemoBMServer(engine, s)
	initRouter(engine)
	err = engine.Start()
	return
}

// grpc.New 初试化和启动 grpc server
func New(svc pb.DemoServer) (ws *warden.Server, err error) {
	var (
		cfg warden.ServerConfig
		ct paladin.TOML
	)
	if err = paladin.Get("grpc.toml").Unmarshal(&ct); err != nil {
		return
	}
	if err = ct.Get("Server").UnmarshalTOML(&cfg); err != nil {
		return
	}
	ws = warden.NewServer(&cfg)
	pb.RegisterDemoServer(ws.Server(), svc)
	ws, err = ws.Start()
	return
}
```

直接在 `internal/di` 下运行 wire 或者执行 `kratos tool wire`, 最终会在 `internal/di`下生成 `wire_gen.go`, 其生成的具体启动步骤:

1.  dao.NewRedis(): 生成 redis handler 及 cleanup 清理函数(给优雅重启使用)
2.  dao.NewMC(): 生成 memcache handler 及 cleanup 清理函数(给优雅重启使用)
3.  dao.NewDB(): 生成 db handler 及 cleanup 清理函数(给优雅重启使用)
4.  dao.New(redis, memcache, db): 将 redis, mc, db handler 传给 dao 的初始化函数, 初始化Dao
5.  service.New(daoDao): 将 dao 传给 service, 将 service 初始化
6.  http.New(serviceService): 将 service 传给 http, 启动 http server
7.  grpc.New(serviceService): 将 service 传给 grpc, 启动 grpc server
8.  NewApp(serviceService, engine, server): 将 http 和 grpc server 整体跟框架串联起来


### 4. 优雅重启

```go
c := make(chan os.Signal, 1)
signal.Notify(c, syscall.SIGHUP, syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT)
for {
	s := <-c
	log.Info("get a signal %s", s.String())
	switch s {
	case syscall.SIGQUIT, syscall.SIGTERM, syscall.SIGINT:
		closeFunc()
		log.Info("kratos-demo exit")
		time.Sleep(time.Second)
		return
	case syscall.SIGHUP:
	default:
		return
	}
}
```

这个就比较简单了, 主要利用各个组件的初始化函数返回的 cleanup 函数, 在程序收到 `syscall.SIGQUIT(3)`, `syscall.SIGTERM(15)`, `syscall.SIGINT(2)` 优雅关闭各个组件


## 总结

至此, 输出 `Golang 大法好 ！！！` 的整个流程大概介绍了一遍. 下面一篇会写一个简单的 demo, postman 通过 `/check_role` 接口 访问 biz 项目, biz 通过 grpc 和 http 两种方式访问 up 项目