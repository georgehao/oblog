---
title: go 跨平台编译
author: haohongfan
type: post
date: 2018-05-04T05:30:16+00:00
categories:
  - golang
---

## Go编译支持的平台类型

1. `amd64` (also known as `x86-64`)
2. `386` (`x86` or `x86-32`)
3. `arm` (`ARM`)
4. `arm64` (`AArch64`)  version >= 1.5 
5. `ppc64, ppc64le`    version >= 1.5
6. `mips, mipsle` (32-bit MIPS big- and little-endian)  version >= 1.8
7. `mips64, mips64le` (64-bit MIPS big- and little-endian) version >= 1.6
8. `s390x` (IBM System z) version >= 1.7

| $GOOS     | $GOARCH  |
|-----------|----------|
| android   | arm      |
| darwin    | 386      |
| darwin    | amd64    |
| darwin    | arm      |
| darwin    | arm64    |
| dragonfly | amd64    |
| freebsd   | 386      |
| freebsd   | amd64    |
| freebsd   | arm      |
| linux     | 386      |
| linux     | amd64    |
| linux     | arm      |
| linux     | arm64    |
| linux     | ppc64    |
| linux     | ppc64le  |
| linux     | mips     |
| linux     | mipsle   |
| linux     | mips64   |
| linux     | mips64le |
| linux     | s390x    |
| netbsd    | 386      |
| netbsd    | amd64    |
| netbsd    | arm      |
| openbsd   | 386      |
| openbsd   | amd64    |
| openbsd   | arm      |
| plan9     | 386      |
| plan9     | amd64    |
| solaris   | amd64    |
| windows   | 386      |
| windows   | amd64    |

## 源码安装go

#### 需要安装软件

```
apt-get install gcc git bzip2
```

#### note

>  想要go支持`cgo`,  `c`编译器(比如: gcc, clang)必须先安装. 如果不安装`cgo`, 必须设置`CGO_ENABLED=0`.


#### 安装go1.4

1. wget https://dl.google.com/go/go1.4-bootstrap-20170518.tar.gz 
2. 解压到go1.4 
3. export GOROOT_BOOTSTRAP=/go/go1.4 
4. ./all.bash 
5. export PATH="$PATH:/go/go1.4/bin"



#### note

>  需要首先配置`GOROOT_BOOTSTRAP`.  `GOROOT_BOOTSTRAP`默认值是`$HOME/go1.4`

>  `bootstrap toolchain`有很多选项. 设置`GOROOT_BOOTSTRAP`为解压的目录. `$GOROOT_BOOTSTRAP/bin/go`必须是可执行文件`go`

#### 安装最新版本

1. wget https://dl.google.com/go/go1.10.2.src.tar.gz 
2. 解压到g1.10 
3. cd src 
4. GOOS=linux GOARCH=mipsle ./bootstrap.bash
5. 会在`../../go-${GOOS}-${GOARCH}-bootstrap`形成这个文件夹, 将这个文件夹拷贝到目标环境中就可以了.



#### 安装go

```
$ cd src
$ ./all.bash
```



如果安装成功, 会显示:

```
ALL TESTS PASSED

---
Installed Go for linux/amd64 in /home/you/go.
Installed commands in /home/you/go/bin.
*** You need to add /home/you/go/bin to your $PATH. ***
```



#### 测试安装成功

```
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```

Then run it with the `go` tool:

```
$ go run hello.go
hello, world
```
## 使用官方编译好的

> 下载地址: https://golang.google.cn/dl/

#### linux安装方式:

1. wget https://dl.google.com/go/go1.10.2.linux-amd64.tar.gz 
2. `tar zxvf go1.10.2.linux-amd64.tar.gz  -C  $HOME`
3. `export GOROOT=$HOME/go `
4. `export GOPATH=$HOME/gopath`
5. `export PATH=$PATH:$GOROOT/bin:$GOPATH/bin`

>  其他安装方式参见 [官方](https://golang.google.cn/doc/install)

#### 测试安装成功

```
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```

Then run it with the `go` tool:

```
$ go run hello.go
hello, world
```
#### 编译跨平台程序

```
GOARCH=mipsle GOOS=linux go build  test.go  // 具体支持的查看Go编译支持的平台类型
```
#### Note

> 1. 如何查看编译的程序是否是对应的平台的程序 ?

利用file, xxd命令

```
file test

test: ELF 32-bit LSB executable, MIPS, MIPS32 version 1 (SYSV), statically linked, with debug_info, not stripped
```



## 结论

一般情况下, 没有必要从源码编译go, 直接使用官方编译好的就可以. 确定一下几个条件:

1. 目标平台
2. 选用正确的`GOARCH`, `GOOS`
3. 正确的go版本

### 参考文章

1. [Installing Go from source](https://golang.google.cn//doc/install/source)
2. [go-mips32 交叉编译go程序 编译kcptun例子](https://github.com/xtaci/kcptun/issues/79)
3. [Ubuntu下交叉编译kcptun go语言源码 for openwrt](http://iytc.net/wordpress/?p=1564)
4. [搭建go-mips32的docker镜像](https://www.jianshu.com/p/23fa1d177a20)



