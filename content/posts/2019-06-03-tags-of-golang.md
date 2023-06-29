---
title: "[每日一译] Tags in Golang"
date: 2019-06-03T22:22:15+08:00
lastmod: 2019-06-03T22:22:15+08:00
tags: [golang, 每日一译]
categories: [golang, 每日一译]
---

> 原文地址: [Tags in Golang](https://medium.com/golangspec/tags-in-golang-3e5db0b8ef3e)

我们声明golang struct时可以在struct字段后面添加一些字符串来丰富这个字段, 这些字符串称为`tag`. Tags可以被当前package或者包外使用. 让我们首先看看struct是如何声明的, 然后深入研究下tag本身, 最后用几个例子结束

## Struct type

struct是一系列的字段的组合. 每一个字段都由一个`optional`名字和`required`type组成

```go
package main

import "fmt"

type T1 struct {
    f1 string
}

type T2 struct {
    T1
    f2     int64
    f3, f4 float64
}

func main() {
    t := T2{T1{"foo"}, 1, 2, 3}
    fmt.Println(t.f1)    // foo
    fmt.Println(t.T1.f1) // foo
    fmt.Println(t.f2)    // 1
}
```

> T1 被称为embedded field, 因为它只有类型没有名字 (hhf: 关于这个可以看我另外一篇文章: [golang面向对象分析](http://www.haohongfan.com/post/2017-07-24-golang_oop/))

字段可以用一个类型声明多个标识符, 就像T2的f3, f4一样

golang语言声明每个字段声明后面跟着分号, 但是我们在写的时候可以忽略分号(hhf:https://golang.org/doc/effective_go.html#semicolons, if the newline comes after a token that could end a statement, insert a semicolon). 当多个字段放在同一行时, 需要用分号分割

```go
package main

import "fmt"

type T struct {
    f1 int64; f2 float64
}

func main() {
    t := T{1, 2}
    fmt.Println(t.f1, t.f2)  // 1 2
}

```

## Tag

Struct的字段声明时, 如果后面跟着可选的`tag`时, 那么`tag`将相对应的那些字段的属性

```go
type T struct {
    f1     string "f one" // 解释字符串
    f2     string
    f3     string `f three` // 原始字符串
    f4, f5 int64  `f four and five`
}
```

> tag可以使用原始字符串或者解释字符串, 下面描述的传统格式的例子需要使用原始字符串. 原始字符串和解释字符串
> 在[spec](https://golang.org/ref/spec#String_literals)有详细解释

struct字段同一个type声明多个标识符(例如上面的f4,f5), 那么tag对f4,f5都是有效的

## Reflection

Tags是可以通过`reflect`被访问到的.

```go
package main

import (
    "fmt"
    "reflect"
)

type T struct {
    f1     string "f one"
    f2     string
    f3     string `f three`
    f4, f5 int64  `f four and five`
}

func main() {
    t := reflect.TypeOf(T{})
    f1, _ := t.FieldByName("f1")
    fmt.Println(f1.Tag) // f one
    f4, _ := t.FieldByName("f4")
    fmt.Println(f4.Tag) // f four and five
    f5, _ := t.FieldByName("f5")
    fmt.Println(f5.Tag) // f four and five
}
```

设置空tag和不使用tag的效果时一样的

```go
type T struct {
    f1 string ``
    f2 string
}
func main() {
    t := reflect.TypeOf(T{})
    f1, _ := t.FieldByName("f1")
    fmt.Printf("%q\n", f1.Tag) // ""
    f2, _ := t.FieldByName("f2")
    fmt.Printf("%q\n", f2.Tag) // ""
}
```

## Conventional format

[reflect: support for struct tag use by multiple packages](https://github.com/golang/go/commit/25733a94fde4f4af4b6ac5ba01b7212a3ef0f013)的介绍中允许为每个package设置元信息. 这提供了简单的命名空间. Tags被格式化成key-value键值对, Key可以是像`json`包这样的名字. 键值对可以被空格分开(这是可选的)--`key1:"value1" key2:"value2" key3:"value3"`. 如果使用传统格式，那么我们可以使用两种struct tag方法--`Get or Lookup`, 他们返回key对应的value

`Lookup`返回两个值: 与key关联的值, bool值(是否找到那个key)

```go
type T struct {
    f string `one:"1" two:"2"blank:""`
}
func main() {
    t := reflect.TypeOf(T{})
    f, _ := t.FieldByName("f")
    fmt.Println(f.Tag) // one:"1" two:"2"blank:""
    v, ok := f.Tag.Lookup("one")
    fmt.Printf("%s, %t\n", v, ok) // 1, true
    v, ok = f.Tag.Lookup("blank")
    fmt.Printf("%s, %t\n", v, ok) // , true
    v, ok = f.Tag.Lookup("five")
    fmt.Printf("%s, %t\n", v, ok) // , false
}
```

`Get`是对`LookUp`的简单封装, 但是他舍弃掉了bool值

```go
func (tag StructTag) Get(key string) string {
    v, _ := tag.Lookup(key)
    return v
}
```

> 当Tag不是传统格式时, 那么`Lookup`和`Get`不会返回指定的值 (hhf:这句话是什么意思呢 Return value of Get or Lookup is unspecified if tag doesn’t have conventional format.)

不管tag是任何字符串(原始字符串或者解释字符串), 只有value包含在双引号之间, 那么`LookUp`和`Get`会返回key对应的值.

```go
type T struct {
    f string "one:`1`"
}
func main() {
    t := reflect.TypeOf(T{})
    f, _ := t.FieldByName("f")
    fmt.Println(f.Tag) // one:`1`
    v, ok := f.Tag.Lookup("one")
    fmt.Printf("%s, %t\n", v, ok) // , false
}
```

**hhf: 从这里可以看出来, 只有value包含在""之中才能得到相关的值**

可以在双引号中使用转义的解释字符串, 但是这个可读性要差很多

```go
type T struct {
    f string "one:\"1\""
}
func main() {
    t := reflect.TypeOf(T{})
    f, _ := t.FieldByName("f")
    fmt.Println(f.Tag) // one:"1"
    v, ok := f.Tag.Lookup("one")
    fmt.Printf("%s, %t\n", v, ok) // 1, true
}
```

## Conversion

当一个struct向另外一个转换时, 需要对应字段的类型是相同的. 但是这些字段的tag是被忽略的

```go
type T1 struct {
     f int `json:"foo"`
 }
 type T2 struct {
     f int `json:"bar"`
 }
 t1 := T1{10}
 var t2 T2
 t2 = T2(t1)
 fmt.Println(t2) // {10}
```

## 使用struct tag的例子

### (Un)marshaling

golang使用tag最多的地方可能就是[marshalling](https://en.wikipedia.org/wiki/Marshalling_%28computer_science%29)
让我们看一下来自json包的函数Marshal是如何使用的

```go
import (
    "encoding/json"
    "fmt"
)
func main() {
    type T struct {
       F1 int `json:"f_1"`
       F2 int `json:"f_2,omitempty"`
       F3 int `json:"f_3,omitempty"`
       F4 int `json:"-"`
    }
    t := T{1, 0, 2, 3}
    b, err := json.Marshal(t)
    if err != nil {
        panic(err)
    }
    fmt.Printf("%s\n", b) // {"f_1":1,"f_3":2}
}
```

`xml`包也使用了tag的特性—https://golang.org/pkg/encoding/xml/#MarshalIndent.

### ORM

像gorm广泛使用tag [example](https://github.com/jinzhu/gorm/blob/58e34726dfc069b558038efbaa25555f182d1f7a/multi_primary_keys_test.go#L10)

*hhf: 我觉得最好用的orm

### Other

其他潜在的用法可能就是`配置管理`, `struct默认值`, `validation`, `命令行参数`, 例如https://github.com/golang/go/wiki/Well-known-struct-tags

### go vet

Go编译器不强制执行结构标记的传统格式，但是去看看它是否值得使用它，例如作为CI管道的一部分

```go
package main
type T struct {
    f string "one two three"
}
func main() {}
> go vet tags.go
tags.go:4: struct field tag `one two three` not compatible with reflect.StructTag.Get: bad syntax for struct tag pair
```

*hhf: 一般IDE都直接提示:  bad syntax for struct tag*