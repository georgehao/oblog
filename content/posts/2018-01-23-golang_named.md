---
title: Golang命名规范
author: haohongfan
type: post
date: 2018-01-23T15:34:24+00:00
categories:
  - golang

---
## 文件名

  1. 没有明确的规定
  2. 根据`test`文件的命名规范推测, 还是使用`下划线`比较统一.
  3. 测试文件: path_test.go
  4. 版本区分: trap\_windows\_1.4.go
  5. 平台区分: file_windows.go
  6. CPU区分: vdso\_linux\_amd64.go

由于没有明确的文档说这个事情. 从源码来看, 我觉得<a href=\"https://studygolang.com/articles/8542\">这篇文档</a>归档的挺不错. 以后发现有别的规定再补充.

## 包名

尽量短, 尽量简洁, 容易记(short, concise, evocative). 因为别人会用你的package.

包名都是小写(lower case)

一个单词(single-word), 不能是下划线或者驼峰式(mixedCaps)的命名方式. 看源码发现有人是把两个单词合成一个在用, 如: `screen shot`会写成`screenshot`

在工程中`包名`并不一定需要唯一. 因为包名只是默认导入的名字. 当报名冲突的时候, 可以在引入的地方重新起一个别名

包名只是目录的`base name`. 例如: `src/encoding/base64`的引入方式是`import encoding/base64`,但包名为`base64`

由于我们使用包内的元素会带上这个包名, 因此对`struct`, `function`等命名的时候, 我们可以使用非常简洁的方式. 例如: `bufio`的`buffered reader type` 命名为`Reader`, 而不是`BufReader`; 还比如, 在`ring`包内, 有一个类`Ring`, 我们为这个`Ring`声明一个构造函数时, 可以直接把这个函数起名为`New`, 而不是`NewRing`

## 函数名/变量名

函数名遵循`Golang`包导出的原则, 首字符大写的是包外可见的; 否则包外不可见.

命名要遵循`MixedCaps`和`mixedCaps`的方式. 不能使用`下划线`的方式.

函数命名要避免使用一些特殊的名字, 如: Read, Write, Close, Flush, String等一些有特殊意义的名字. 防止和库函数混淆.

同样, 变量名也一样.

## Getters

`Go`不会自动提供`getters` 和 `setters`, 这两个方法通常需要自己提供.

通常不建议将`Get`放在方法名中. 如: 在获取字段`owner`的时候, 尽量使用`Owner()`, 而不是`GetOwner()`

`Set`方法的函数名是需要带上`Set`的. 如: `SetOwner()`

    owner := obj.Owner()
    if owner != user {
        obj.SetOwner(user)
    }
    

## 接口

如果接口只有一个方法, 那个这个接口的命名就是这个方法的名字 + 后缀(er)构成.

## MixedCaps

`Golang` 是推荐使用`MixedCaps`或者`mixedCaps`. 强烈不建议使用`下划线`方式.

## Receiver Name

`Receiver Name`是`Receiver Type`的一个反映, 通常是这个类型的一个或者两个字符的缩写, 如: 对于`Client`, `c`或者`cl`就足够了, 要足够简洁.

`Receiver Name` 不要使用`me`, `this`, `self`这些OOP语言通用的名字.

同一个Type的Name, 要保持一致. 如果有一个Receiver叫`c`, 那么另外一个就不要叫`cl`了.

参考:

1. [Effictive Go](https://golang.org/doc/effective_go.html)
2. [文件名规范](https://studygolang.com/articles/8542")
  
3. [receiver-names](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names)