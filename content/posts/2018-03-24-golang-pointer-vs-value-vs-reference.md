---
title: Golang指针vs值
author: haohongfan
type: post
date: 2018-03-24T15:58:13+00:00
categories:
  - golang
---

Go语言允许通过`值`和`指针`的方式来传递参数.

严格的说Go只允许通过`值`的方式来传递参数. 当一个变量被当做函数的参数传递, 那么这个变量一定会被复制, 然后传入被调用的函数里. 当这个变量被复制, 那么一定会重新开辟内存地址.

对于参数为`值`的时候:
```
func test(p int) {
    fmt.Printf("%p\n", &p)  // 0xc42000e250
}

func main() {
    p := 10
    fmt.Printf("%p", &p) // 0xc42000e218
    test(p)
}
```    
从结果可以很容易看出来, 变量`p`和参数`p`是两个完全不一样地址, 就可以很容易理解变量`p`被复制了一份.

当参数为`指针`时如何理解呢? 当参数是以`指针`的方式传递时, 指向那个地址的指针就被复制了.

```
func test(p *int) {
    fmt.Printf("参数p的值: %p", p)
    fmt.Printf("参数p的地址: %p", &p)
}

func main() {
    p := 10
    fmt.Printf("变量p的地址: %p", &p)
    test(&amp;p)
}

结果:
变量p的地址: 0xc42006e178
参数p的值: 0xc42006e178
参数p的地址: 0xc42007a020
```  
可以从结果上明显看出来, 当参数p是指针的时候, p也是被复制了一份, 但是参数p指针指向的地址是不变的

## 参数通过值传递

参数通过值传递的时候, 在函数内改变参数的值, 因为是对值的一份拷贝, 故不会对变量改变
```
func test(p int) {
    p = 20
    fmt.Println("改变后的值:", p)
}

func main() {
    p := 10
    fmt.Println("调用test前:", p)
    test(p)
    fmt.Println("调用test后:", p)
}


结果:
调用test前: 10
改变后的值: 20
调用test后: 10
```

## 参数通过指针传递

参数通过指针传递的时候, 虽说参数`p`是变量`p`的指针的复制, 但是他们指向的地址没有发生改变. 在函数内改变了指针指向地址的内容, 那么变量`p`的值也就跟着发生了改变.
```
func test(p *int) {
    *p = 20
    fmt.Println("改变后的值:", *p)
}

func main() {
    p := 10
    fmt.Println("调用test前:", p)
    test(&amp;p)
    fmt.Println("调用test后:", p)
}
    
结果:
调用test前: 10
改变后的值: 20
调用test后: 20
```
## Receiver

### Receiver Name
1. `Receiver Name`是`Receiver Type`的一个反映, 通常是这个类型的一个或者两个字符的缩写, 如: 对于`Client`, `c`或者`cl`就足够了, 要足够简洁.
2. `Receiver Name` 不要使用`me`, `this`, `self`这些OOP语言通用的名字.
3. 同一个Type的Name, 要保持一致. 如果有一个Receiver叫`c`, 那么另外一个就不要叫`cl`了.

[参考链接](https://github.com/golang/go/wiki/CodeReviewComments#receiver-names)

### Receiver Type

选择Receiver是指针还是值, 这是一个纠结的事情. 有一个指导思想就是`If in doubt, use a pointer`, 简单来说, 就是Receiver尽量使用指针.

有以下指导思想:
**不需要使用指针**
1. 当`receiver`是`map`, `func`, `chan`时, 不要使用指针
2. 当`receiver`是`sclice`, 并且其方法不需要重新分配申请内存, 那么不要使用指针

**必须使用指针**
1. 当其方法需要改变`receiver`的值, 那么receiver必须是个指针
2. 当`receiver`是`struct`, 并且`Struct`包含`sync.Mutex`或者其他`同步`的字段, 那么必须使用指针避免复制
3. 当`receiver`是一个巨大的`struct`或者`array`, 为了保证效率, 最好使用指针. 

**同样这些准则对函数/方法的参数也是适用的.**

### 参考链接
[receiver-type](https://github.com/golang/go/wiki/CodeReviewComments#receiver-type)