---
title: "[每日一译]golang range与iteration之间的关系"
author: "HHF"
tags: ["golang", "每日一译"]
keywords: [golang]
categories: [golang, 每日一译]
date: 2019-06-04T11:11:54+08:00
---

> [原文地址](https://medium.com/golangspec/range-clause-and-the-address-of-iteration-variable-b1796885d2b7)

```go
package main
import (
    "fmt"
)
type T struct {
    id int
}
func main() {
    t1 := T{id: 1}
    t2 := T{id: 2}
    ts1 := []T{t1, t2}
    ts2 := []*T{}
    for _, t := range ts1 {
        ts2 = append(ts2, &t)
    }
    for _, t := range ts2 {
        fmt.Println((*t).id)
    }
}
```

先不要看结果, 自己想想输出是什么?

对于很多人(包括我自己), 结果可能会让人感到惊讶

```
2
2
```

我个人期待的结果, 但是这是一个错误结果

```
1
2
```

迭代变量`t`使用短变量声明的方式声明, 它的声明周期就是`for`代码块. 这个变量在第一次循环时是第一个元素的值, 在第二次循环时是第二个元素的值, 但是在内存的某个地方保存着slice被遍历结束时的值. `t`没有指向slice底层数组指向的值--这是一个临时的桶, 下一个元素会覆盖当前值. t是一个辅助变量来保存当前迭代的元素, 所以`&t`每次循环都是相同的值

*hhf: 这一段翻译好像不是特别准确, 等我后面查完资料再补充*

```go
t1 := T{id: 1}
t2 := T{id: 2}
ts1 := []T{t1, t2}
ts2 := []*T{}
for _, t := range ts1 {
    t.id = 3
    ts2 = append(ts2, &t)
}
for _, t := range ts2 {
    fmt.Println((*t).id)
}
fmt.Println(ts1)
fmt.Println(t1)
fmt.Println(t2)
```

output:
```
3
3
[{1} {2}]
{1}
{2}
```

可能的解决方案是使用`index`去获得slice元素的地址

```go
t1 := T{id: 1}
t2 := T{id: 2}
ts1 := []T{t1, t2}
ts2 := []*T{}
for i, _ := range ts1 {
    ts2 = append(ts2, &ts1[i])
}
for _, t := range ts2 {
    fmt.Println((*t).id)
}
```

output:
```
1
2
```

参考资料

https://golang.org/ref/spec#For_range