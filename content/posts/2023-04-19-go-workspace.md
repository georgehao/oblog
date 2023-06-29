---
title: "Go1.18 Go workspace 初体验"
date: 2023-04-17T21:27:21+08:00
authors: hhf
tags: [Go] 
categories: [Go]
toc: true
---

hi，大家好，我是好久没有更新的 haohongfan。

Go 1.18 终于正式发布了，本次版本更新中 Go mod 有个很实用的功能 “multi-module workspaces”. 本篇文章简单介绍下 workspace 的使用方式以及使用场景。

更新 go 1.18 版本，推荐使用 [goup](github.com/owenthereal/goup "goup")，做多版本管理很方便。

## Go work 使用方式

### 1. 创建一个工作空间 

```
mkdir workspace
cd  workspace
```

### 2. 初始化一个项目

```
mkdir hello_work
cd hello
go mod init example.com/hello_work
```

output：go: creating new go.mod: module example.com/hello_work

接着执行：`go get github.com/georgehao/gomodtestc`

在 workspace/hello_work 文件夹下，创建 hello_work.go

```go
package main

import (
    "fmt"

    "github.com/georgehao/gomodtestc"
)

func main() {
    fmt.Println(gomodtestc.PrintStr("Hello", 100))
}
```
output:
```
go run example.com/hello

project C Hello_100
```

### 3. 本地更改依赖包做测试

因为需求的变更，依赖的 gomodtestc 的功能更改了，但是不确定更改的是否正常，我们需要用 hello_work 项目来测试。

比如：`git clone git@github.com:georgehao/gomodtestc.git`, gomodtestc 的更改如下，

```go
package gomodtestc

import "fmt"

func PrintStr(str string, num int) string {
	return fmt.Sprintf("project C %s_%d", str, num)
}

// TestWork 新增功能。。。
func TestWork()  string {
	return "hello workespace"
}
```

如何使用 hello_work 项目来测试呢？

在没有 go1.18 之前，只能使用 replace，如下：

```
module example.com/hello

go 1.17

require github.com/georgehao/gomodtestc v1.0.1

replace (
	github.com/georgehao/gomodtestc =>  "/Users/haohongfan1/project/workspace/gomodtestc"
)
```

这样做程序能正常测试，但是带来一个问题，我们要修改 go.mod，一不留神这个 replace 过的 go.mod 就上传到 gitlab 去了。实在很烦。

福音来了，go workspaces 真的好爽。(**读者朋友们，如果上面跟着做了replace操作，go.mod记得要恢复原状**)

```
cd workspace
go work init ./hello_work
```
会在 workspace 下生成一个 go.work文件

```
go 1.18

use ./hello_work
```

这个时候，我们把 gomodtestc 项目移动到 workspace 目录下，然后执行：

```
go work use ./gomodtestc
```

go.work的更改如下：

```
go 1.18

use (
	./gomodtestc
	./hello_work
)
```

然后执行：go run example.com/hello
output:
```
project C Hello_100
hello workespace
```

使用 go work，我们不需要对 hello_work 项目的 go.mod 做任何操作就能正常测试项目的修改。真的非常方便。


## Go work 使用场景及注意点

### 使用场景
* 本地调试。

比如，公司内部开发时，一般都会沉淀出一个 common 的组件项目，这个组件项目可能会有很多人修改，这个项目修改后本地测试，使用 go work 会很方便快捷。

### 注意点

* 测试的项目和被测试项目必须位于同一个文件夹下。如本文中的 hello_work 与 gomodtestc 必须都在 workspace 文件夹下
* go.work 不需要提交

关于 go work 可以看官方文档

* [go blog](https://go.dev/doc/tutorial/workspaces "go blog")
* [go ref](https://go.dev/ref/mod#workspaces "go ref")
