---
title: "Bilibili Kratos 框架源码分析(2) -- Kratos 一些简单例子"
date: 2020-05-16T10:14:07+08:00
lastmod: 2020-05-16T10:14:07+08:00
keywords: [golang, kratos]
tags: [golang, kratos]
categories: [golang, kratos]
---

本篇主要介绍四种使用 kratos 的例子. 前情透漏, 这一篇的篇幅比较长, 如果已经会用 Kratos 的可以跳过这一节

## http 服务

http 服务其实就比较简单, 开篇的`Golang 大法好 ！！`直接就能对外提供 http 服务. http server默认有两个函数`howToStart`, `ping`.  关于ping 函数后面再具体看其作用

如何添加一个新的 api 接口?

```go
// internal/server/http/server.go
func initRouter(e *bm.Engine) {
	e.Ping(ping)
	g := e.Group("/kratos-demo")
	{
		g.GET("/start", howToStart)
		g.GET("/sayHi", sayHi)
		g.GET("/sayHello", func(context *bm.Context) {
			svc.SayHello(context, &pb.HelloReq{
				Name: "test",
			})
			context.JSON("success", nil)
		})
	}

	v2Group := e.Group("/kratos-demo/v2")
	{
		v2Group.GET("/sayHi", sayHi)
	}
}

func sayHi(c *bm.Context) {
	c.JSON("hi", nil)
}
```

这里你会不会想所有的 api controller 全部都放在一个 server.go, 后面会不会变得不可维护? 答案肯定的

解决方案: 在 `internal/server/http` 目录下创建代表不同功能的文件

1. 升级不同版本了, 可以v1.go, v2.go, 
2. 负责评论的叫 comment.go
3. 负责收藏的叫 favorite.go

## grpc 服务

kratos-demo 默认是是支持 grpc 服务的, gprc 是需要 protobuf 支持的. 工程所有的 protobuf 应该都放在 api 目录下. 

通过`kratos tool protoc --grpc --bm api.proto` 生成两个文件 `api.pb.go`, `api.bm.go`

protobuf 的内容:

```protobuf
service Demo {
  rpc Ping(.google.protobuf.Empty) returns (.google.protobuf.Empty);
  rpc SayHello(HelloReq) returns (.google.protobuf.Empty);
  rpc SayHelloURL(HelloReq) returns (HelloResp) {
    option (google.api.http) = {
      get: "/kratos-demo/say_hello"
    };
  };
}
```

当然要想使用 grpc 的话需要实现上面三个函数. kratos-demo 默认是实现这三个方法的, 不过在`internal/service/service.go`里面

```go
// SayHello grpc demo func.
func (s *Service) SayHello(ctx context.Context, req *pb.HelloReq) (reply *empty.Empty, err error) {
	reply = new(empty.Empty)
	fmt.Printf("hello %s", req.Name)
	return
}

// SayHelloURL bm demo func.
func (s *Service) SayHelloURL(ctx context.Context, req *pb.HelloReq) (reply *pb.HelloResp, err error) {
	reply = &pb.HelloResp{
		Content: "hello " + req.Name,
	}
	fmt.Printf("hello url %s", req.Name)
	return
}

// Ping ping the resource.
func (s *Service) Ping(ctx context.Context, e *empty.Empty) (*empty.Empty, error) {
	return &empty.Empty{}, s.dao.Ping(ctx)
}
```

我们要注意的是, `internal/server/grpc/server.go`并没有调用这些方法的地方, grpc是如何调用起来的呢?

我们上一文章说到, kratos 使用 wire 依赖注入将各种组件组合到一块

```go
// wire_gen.go
server, err := grpc.New(serviceService)
if err != nil {
	...
	return nil, nil, err
}
```

这里的 `serviceService` 就是 `internal/service/service.go`的 `Service`

这里推荐使用 [bloomrpc](https://github.com/uw-labs/bloomrpc) 来验证 grpc 接口

![](https://images.haohongfan.com/kratos-demo-bloomrpc.png)

## http 调用 http 服务

不管 http 调用 http, 还是 http 调用 grpc, 官方 wiki 并没有过多的提及. 就看到这么[一句话](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-quickstart.md#client%E8%B0%83%E7%94%A8), 更加坑的地方在于找遍所有的地方竟然没有 http client 的配置文件的格式

> 请进入internal/dao方法内，一般对资源的处理都会在这一层封装。对于client端，前提必须有对应proto文件生成的代码，那么有两种选择：
> 1. 拷贝proto文件到自己项目下并且执行代码生成
> 2. 直接import服务端的api package

下面看 http 如何调用 http. 假设有一个策略服务(biz)通过角色服务(up)的`check/role`判断某个用户的角色是否满足

```
kratos new biz // 创建策略服务
kratos new up  // 创建角色服务
```

##### up

proto文件, `kratos tool protoc --grpc --bm api.proto` 生成对应的 go 文件

```proto
// api/api.proto
syntax = "proto3";

import "google/protobuf/empty.proto";

package up.service.v1;

option go_package = "api";

service Up {
    rpc Ping(.google.protobuf.Empty) returns (.google.protobuf.Empty);
    rpc CheckRole(CheckUpReq) returns (CheckUpResp);
}

message CheckUpReq {
    int32 Role = 1;
}

message CheckUpResp {
    bool Yes = 1;
}
```

在 `internal/service/service.go`里面添加下面代码

```go
func (s *Service) CheckRole(ctx context.Context, req *pb.CheckUpReq) (resp *pb.CheckUpResp, err error) {
	log.Infoc(ctx, "check role:%d", req.Role)
	if req.Role == 10 {
		resp = &pb.CheckUpResp{Yes: true}
	} else {
		resp = &pb.CheckUpResp{Yes: false}
	}
	return
}

// Ping ping the resource.
func (s *Service) Ping(ctx context.Context, e *empty.Empty) (*empty.Empty, error) {
	return &empty.Empty{}, s.dao.Ping(ctx)
}
```

在 `internal/server/http/server.go`里面添加如下代码

```go
type roleParam struct {
	RoleId int32 `form:"roleId"`
}

type roleResult struct {
	Yes bool `json:"yes"`
}

func checkRole(context *bm.Context) {
	var input roleParam
	if err := context.Bind(&input); err != nil {
		context.JSON(nil, errors.New("client param error"))
		return
	}

	resp, err := svc.CheckRole(context, &pb.CheckUpReq{Role: input.RoleId})
	if err != nil {
		context.JSON(nil, err)
		return
	}
	res := roleResult{Yes: resp.Yes}
	context.JSON(res, nil)
}
```

up服务已经可以了, 接下来看 biz 如何调用 up

##### biz

在`configs/http.html`里面添加下面配置

```
[Server]
    addr = "0.0.0.0:8020"
    timeout = "1s"
[Client]
    dial = "2s"
    timeout = "100s"
    keepAlive = "60s"
    timer = 1000
    [httpClient.breaker]
    window  = "10s"
    sleep   = "2000ms"
    bucket  = 10
    ratio   = 0.5
    request = 100
```

在 `internal/dao/http.go`下添加下面代码

```go
package dao

import (
	"github.com/go-kratos/kratos/pkg/conf/paladin"
	"github.com/go-kratos/kratos/pkg/net/http/blademaster"
)

func NewHttpClient() *blademaster.Client {
	var cfg blademaster.ClientConfig
	var ct paladin.TOML
	if err := paladin.Get("http.toml").Unmarshal(&ct); err != nil {
		return nil
	}

	if err := ct.Get("Client").UnmarshalTOML(&cfg); err != nil {
		return nil
	}
	return blademaster.NewClient(&cfg)
}
```

在 `internal/dao/dao.go`里面添加下面代码

```go
type dao struct {
	db         *sql.DB
	redis      *redis.Redis
	mc         *memcache.Memcache
	cache      *fanout.Fanout
	demoExpire int32
	rpcClient  api.UpClient
	httpClient blademaster.Client
}

// ....

var grpcClient api.UpClient
grpcClient, err = NewGrpcClient()
if err != nil {
	return
}

d = &dao{
	db:         db,
	redis:      r,
	mc:         mc,
	cache:      fanout.New("cache"),
	demoExpire: int32(time.Duration(cfg.DemoExpire) / time.Second),
	rpcClient:  grpcClient,
	httpClient: *NewHttpClient(),
}

func (d *dao) CheckRole(ctx context.Context, req *api.CheckUpReq) (resp *api.CheckUpResp, err error) {
	params := url.Values{}
	params.Set("roleId", fmt.Sprintf("%d", req.Role))

	var resp1 struct {
		Code int        `json:"code"`
		Data roleResult `json:"data"`
	}

	upUrl := "http://127.0.0.1:8000/up/check_role"
	if err = d.httpClient.Get(context.Background(), upUrl, "", params, &resp1); err != nil {
		return nil, err
	}

	if resp1.Code != 0 {
		err = errors.Errorf("up url(%s) res(%+v) err(%+v)", upUrl+"?"+params.Encode(), resp, ecode.Int(resp1.Code))
		return
	}

	resp = &api.CheckUpResp{
		Yes: resp1.Data.Yes,
	}
	return
}
```

在 `internal/service/service.go`下添加

```go
//...
func (s *Service) CheckRole(ctx context.Context, req *pb.CheckUpReq) (resp *pb.CheckUpResp, err error) {
	log.Infoc(ctx, "check role:%d", req.Role)
	resp, err = s.dao.CheckRole(ctx, req)
	return
}
```

在 `internal/server/http/server.go`里面添加

```go
func initRouter(e *bm.Engine) {
	e.Ping(ping)
	g := e.Group("/zbbiz")
	{
		g.GET("/check_role", checkRole)
	}
}

func ping(ctx *bm.Context) {
}

type roleParam struct {
	RoleId int32 `form:"roleId"`
}

func checkRole(context *bm.Context) {
	var input roleParam
	if err := context.Bind(&input); err != nil {
		context.JSON(nil, ecode.RequestErr)
		return
	}

	resp, err := svc.CheckRole(context, &pb.CheckUpReq{Role: input.RoleId})
	if err != nil {
		context.JSON(nil, err)
		return
	}

	if resp == nil {
		context.JSON(nil, errors.New("resp is nil"))
		return
	}

	cr := model.CheckRole{
		Yes: resp.Yes,
	}
	context.JSON(cr, nil)
}
```

至此, http 调用 http 已经大功告成 !

![](https://images.haohongfan.com/http-http.png)
![](https://images.haohongfan.com/http-up.png)

## http 调用 grpc 服务

Grpc 的话, 其实 [wiki](https://github.com/go-kratos/kratos/blob/master/doc/wiki-cn/warden-resolver.md)基本上说明白了怎么用了. 唯一要注意的地方 **使用discovery** 来做服务发现时, 切记要先去安装[discovery](https://github.com/bilibili/discovery), 看[issue](https://github.com/go-kratos/kratos/issues/506)问为何启动报错, 多半是这个问题


我这里就先使用 etcd 来做服务发现. 如果安装 etcd 就略过了

### up 注册服务

在 `cmd/main.go` 里面添加下面代码, 其他的保持跟 http 一样即可

```go
etcdV3Conf := clientv3.Config{
	Endpoints:   []string{"127.0.0.1:2380"},
	DialTimeout: time.Second * time.Duration(30),
	DialOptions: []grpc.DialOption{grpc.WithBlock()},
}

etcdBuilder, err := etcd.New(&etcdV3Conf)
if err != nil {
	panic(err)
}
ip := "0.0.0.0"
port := "9002"
hn, _ := os.Hostname()
ins := &naming.Instance{
	Zone:     "test",
	Env:      "dev",
	AppID:    "demo.service.up",
	Hostname: hn,
	Addrs: []string{
		"grpc://" + ip + ":" + port,
	},
}

cancel, err := etcdBuilder.Register(context.Background(), ins)
if err != nil {
	panic(err)
}
defer cancel()
```

### 客户端调用

在 `configs/grpc.toml` 添加下面内容

```
[Server]
    addr = "0.0.0.0:9005"
    timeout = "1s"

[Client]
     timeout = "1s"
     timer = 1000
```

在 `internal/dao` 下添加 grpc.go

```go
//const target = "direct://default/127.0.0.1:9002"

const AppID = "demo.service.up" // NOTE: example

func init() {
	etcdV3Conf := clientv3.Config{
		Endpoints:   []string{"127.0.0.1:2380"},
		DialTimeout: time.Second * time.Duration(30),
		DialOptions: []grpc.DialOption{grpc.WithBlock()},
	}
	resolver.Register(etcd.Builder(&etcdV3Conf))
}

func NewGrpcClient() (grpcClient api.UpClient, err error) {
	cfg := &warden.ClientConfig{}
	var ct paladin.TOML
	if err := paladin.Get("grpc.toml").Unmarshal(&ct); err != nil {
		return nil, err
	}

	if err := ct.Get("Client").UnmarshalTOML(&cfg); err != nil {
		return nil, err
	}

	client := warden.NewClient(cfg)
	//cc, err := client.Dial(context.Background(), target)
	//cc, err := client.Dial(context.Background(), "discovery://default/"+AppID)
	cc, err := client.Dial(context.Background(), "etcd://default/"+AppID)
	if err != nil {
		return nil, err
	}
	return api.NewUpClient(cc), nil
}
```

在 `internal/dao/dao.go`添加下面内容

```go
type dao struct {
	db         *sql.DB
	redis      *redis.Redis
	mc         *memcache.Memcache
	cache      *fanout.Fanout
	demoExpire int32
	rpcClient  api.UpClient
	httpClient blademaster.Client
}

// ...

func (d *dao) CheckRole(ctx context.Context, req *api.CheckUpReq) (resp *api.CheckUpResp, err error) {
	resp, err = d.rpcClient.CheckRole(ctx, req)
	return
}
```

其他的跟 http 的调用保持一致, 至此 http 调用 grpc 基本完事

![](https://images.haohongfan.com/http-grpc.png)


## 总结

本篇文章以 demo 为主, 所有的代码都会传至 github: https://github.com/georgehao/kratos-demo 