---
title: golang面向对象分析
author: haohongfan
type: post
date: 2017-07-24T01:51:37+00:00
categories:
  - golang
---
说道面向对象(OOP)编程, 就不得不提到下面几个概念:

* 抽象
* 封装
* 继承
* 多态


其实有个问题`Is Go An Object Oriented Language?`, 随便谷歌了一下, 你就发现讨论这个的文章有很多:

1. [reddit](https://www.reddit.com/r/golang/comments/27p2bc/is_go_an_object_oriented_language_spf13com/?st=j5tjkwov&sh=87c16f01)
2. [google group](https://groups.google.com/forum/#!topic/Golang-Nuts/bSXry29pNo4)

那么问题来了

1. Golang是OOP吗?
2. 使用Golang如何实现OOP?

我入门教程基本就是`A Tour Of Go`以及`Go Web 编程`. 由于之前是写C++, 但是说到Go面向对象编程, 总是感觉怪怪的, 总感觉缺少点什么. 我搜集了一些资料和例子, 加上我的一些理解, 整理出这样一篇文章.

## 一. 抽象和封装
抽象和封装就放在一块说了. 这个其实挺简单. 看一个例子就行了.
```
type rect struct {
    width int
    height int
}

func (r *rect) area() int {
    return r.width * r.height
}

func main() {
    r := rect{width: 10, height: 5}
    fmt.Println("area: ", r.area())
}
```
[完整代码](https://play.golang.org/p/a3bV2vRBMQ)

要说明的几个地方:
1、Golang中的`struct`和其他语言的`class`是一样的.

2、可见性. 这个遵循Go语法的大小写的特性

3、上面例子中, 称`*rect`为`receiver`. 关于`receiver` 可以有两种方式的写法:
```
func (r *rect) area() int {
    return r.width * r.height
}
```
```
func (r rect) area() int {
    return r.width * r.height
}
```

这其中有什么区别和联系呢? 关于详细解释请查看[astaxie的解释](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/02.5.md), 写的非常清晰. 

简单来说, Receiver可以是值传递, 还是可以是指针, 两者的差别在于, 指针作为Receiver会对实例对象的内容发生操作,而普通类型作为Receiver仅仅是以副本作为操作对象,并不对原实例对象发生操作。

4、当`Receiver`为`*rect`指针的时候, 使用的是`r.width`, 而不是`(*r).width`, 是由于Go自动帮我转了,两种方式都是正确的.

5、任何类型都可以声明成新的类型, 因为任何类型都可以有方法.
```
type Interger int
func (i Interger) Add(interger Interger) Interger {
	return i + interger
}
```

6、虽然Interger是从int声明而来, 但是这样用是错误的.
```
var i Interger = 1
var a int = i //cannot use i (type Interger) as type int in assignment 
```

这是因为Go中没有`隐式转换`(写C++的同学都会特别讨厌这个, 因为编译器背着我们干的事情太多了). Golang中类型之间的相互赋值都必须`显式声明`.

上面的例子改成下面的方式就可以了.
```
var i Interger = 1
var a int = int(i)
```

## 二. ~~继承~~(Composition)
说道继承，其实在Golang中是没有继承(Extend)这个概念. 因为Golang舍弃掉了像C++, Java的这种传统的、类型驱动的子类。

> Go Effictive says:
> Go does not provide the typical, type-driven notion of subclassing, but it does have the ability to “borrow” pieces of an implementation by embedding types within a struct or interface.

换句话说, Golang中没有继承, 只有`Composition`. 

Golang中的`Compostion`有两种形式, `匿名组合(Pseudo is-a)`和`非匿名组合(has-a)`

注: 如果不了解OOP的`is-a`和`has-a`关系的话, 请自行google.

### 1. has-a
```
package main

import (
	"fmt"
)

type Human struct {
	name  string
	age   int
	phone string
}

type Student struct {
	h      Human //非匿名字段
	school string
}

func (h *Human) SayHi() {
	fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

func (s *Student) SayHi() {
	fmt.Printf("Hi student, I am %s you can call me on %s", s.h.name, s.h.phone)
}

func main() {
	mark := Student{Human{"Mark", 25, "222-222-YYYY"}, "MIT"}
	fmt.Println(mark.h.name, mark.h.age, mark.h.phone, mark.school)
	mark.h.SayHi()
	mark.SayHi()

}
```

> Output
```
Mark 25 222-222-YYYY MIT
Hi, I am Mark you can call me on 222-222-YYYY
Hi student, I am Mark you can call me on 222-222-YYYY
```

[完整代码](https://play.golang.org/p/LDjPNu92qH)

这种组合方式, 其实对于了解传统OOP的话, 很好理解, 就是把一个`struct`作为另一个`struct`的字段.

从上面例子可以, Human完全作为Student的一个字段使用. 所以也就谈不上继承的相关问题了.我们也不去重点讨论. 

### 2. is-a(Pseudo)----Embedding 
```
type Human struct {
	name string
	age int
	phone string
}

type Student struct {
	Human //匿名字段
	school string
}

func (h *Human) SayHi() {
	fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

func main() {
	mark := Student{Human{"Mark", 25, "222-222-YYYY"}, "MIT"}
	fmt.Println(mark.name, mark.age, mark.phone, mark.school)
	mark.SayHi()
}
```

> Output
```
Mark 25 222-222-YYYY MIT 
Hi, I am Mark you can call me on 222-222-YYYY
```
 [完整代码](https://play.golang.org/p/_hBrSQrsvq)

**这里要说的有几点:** 

**1、字段**
现在`Student`访问`Human`的字符, 就可以直接访问了, 感觉就是在访问自己的属性一样.  这样就实现了OOP的继承. 
```
fmt.Println("Student age:", mark.age) //输出: Student age: 25
```

但是, 我们也可以间接访问:
```
fmt.Println("Student age:", mark.Human.age) //输出: Student age: 25
```

这有个问题, 如果在`Student`也有个字段`name`, 那么当使用`mark.name`会以`Student`的`name`为准.
```
fmt.Println("Student name:", mark.name) //输出:Student Name: student name
```
[完整代码](https://play.golang.org/p/KIEk6yi1kp)

**2、方法**
`Student`也继承了`Human`的`SayHi()`方法
```
mark.SayHi() // 输出: Hi, I am Mark you can call me on 222-222-YYYY
```

当然, 我们也可以重写`SayHi()`方法:
```
type Human struct {
	name  string
	age   int
	phone string
}

type Student struct {
	Human  //匿名字段
	school string
	name   string
}

func (h *Human) SayHi() {
	fmt.Printf("Hi, I am %s you can call me on %s\n", h.name, h.phone)
}

func (h *Student) SayHi() {
	fmt.Println("Student Sayhi")
}

func main() {
	mark := Student{Human{"Mark", 25, "222-222-YYYY"}, "MIT", "student name"}	
	mark.SayHi()
}
```
> Output

```
Student Sayhi
```

[完整代码](https://play.golang.org/p/Ts673bwyV2)

**3、为什么称其为`Pseudo is-a`呢?**

因为`匿名组合`不提供`多态`的特性. 如下面的代码:
```
package main

type A struct{
}

type B struct {
	A  //B is-a A
}

func save(A) {
	//do something
}

func main() {
	b := new(B)
	save(*b);  
}
```
> Output
```
cannot use *b (type B) as type A in argument to save
```

[完整代码](https://play.golang.org/p/4TqVZf2_8g)

还有一个面试题的例子
```
type People struct{}

func (p *People) ShowA() {
	fmt.Println("showA")
	p.ShowB()
}
func (p *People) ShowB() {
	fmt.Println("showB")
}

type Teacher struct {
	People
}

func (t *Teacher) ShowB() {
	fmt.Println("teacher showB")
}

func main() {
	t := Teacher{}
	t.ShowA()
}
```
输出结果是什么呢? 
> Output
```
ShowA
ShowB
```

Effective Go Says:
> There's an important way in which embedding differs from subclassing. When we embed a type, the methods of that type become methods of the outer type, but when they are invoked the receiver of the method is the inner type, not the outer one

也就是说, `Teacher`由于组合了`People`, 所以`Teacher`也有了`ShowA()`方法, 但是在`ShowA()`方法里执行到`ShowB`时, 这个时候的`receiver`是`*People`而不是`*Teacher`, 主要原因还是因为`embedding`是一个`Pseudo is-a`, 没有多态的功能.

**4、 "多继承"的问题**
```
package main

import "fmt"

type School struct {
	address string
}

func (s *School) Address() {
	fmt.Println("School Address:", s.address)
}

type Home struct {
	address string
}

func (h *Home) Address() {
	fmt.Println("Home Address:", h.address)
}

type Student struct {
	School
	Home
	name string
}

func main() {
	mark := Student{School{"aaa"}, Home{"bbbb"}, "cccc"}
	fmt.Println(mark)
	mark.Address()
	fmt.Println(mark.address)

	mark.Home.Address()
	fmt.Println(mark.Home.address)
}
```
> 输出结果:
 
```
30: ambiguous selector mark.Address
31: ambiguous selector mark.address
```

[完整代码](https://play.golang.org/p/tImy0NTTiH)

由此可以看出, Golang中不管是方法还是属性都不存在类似C++那样的多继承的问题. 要访问`Embedding`相关的属性和方法, 需要在加那个相应的`匿名字段`, 如:
```
mark.Home.Address()
```

**5、`Embedding value`和 `Embedding pointer`的区别**

```
package main

import (
	"fmt"
)

type Person struct {
	name string
}

type Student struct {
	*Person
	age int
}

type Teacher struct {
	Person
	age int
}

func main()  {
	s := Student{&Person{"student"}, 10}
	t := Teacher{Person{"teacher"}, 40}
	fmt.Println(s, s.name)
	fmt.Println(t, t.name)
}
```

> Output
```
{0x1040c108 10} student
{{teacher} 40} teacher
```

[完整代码](https://play.golang.org/p/zoIXGfvadf)

I. 两者对于结果来说, 没有啥区别, 只是对传参的时候有影响
II. `Embedding value`是比较常规的写法
III. `Embedding  pointer`比较有优势一点, 不需要关注指针是什么时间被初始化的.


### 三. Interface 
Golang中`Composite`不提供多态的功能,  那是否`Golang`不提供多态呢? 答案肯定是否定. Golang依靠`Interface`实现多态的功能.

下面是我工程里面一段代码的简化:
```
package main

import (
	"fmt"
)

type Check interface {
	CheckOss()
}

type CheckAudio struct {
	//something
}

func (c *CheckAudio) CheckOss() {
	fmt.Println("CheckAudio do CheckOss")
}

func main() {
	checkAudio := CheckAudio{}

	var i Check

	i = &checkAudio //想一下这里为啥需要&

	i.CheckOss()
}

```
[完整代码](https://play.golang.org/p/wEv6e8vSOF)

**1、Interface 如何`Composite` ?**
```
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

type ReadWriter interface {
    Reader
    Writer
}
```
其实很简单, 就是把`Reader`, `Writer`嵌入到`ReadWriter`中, 这样`ReadWriter`就拥有了`Reader`和`Writer`的方法.
## 尾声
至此, 基本说完了Golang的面向对象.  有哪里我理解的不对的地方, 请给我留言.

参考资料
1. [Effective Go: Embedding](https://golang.org/doc/effective_go.html#Embedding)
2. [Go面试题](https://yushuangqi.com/blog/2017/golang-mian-shi-ti-da-an-yujie-xi.html)
3. [Is Go An Object Oriented Language?](http://spf13.com/post/is-go-object-oriented/)
4. [go web编程](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/02.4.md)
5. [object-oriented-programming-in-go](https://www.goinggo.net/2013/07/object-oriented-programming-in-go.html)