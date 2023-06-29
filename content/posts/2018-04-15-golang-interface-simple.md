---
title: Golang interface入门
author: haohongfan
type: post
date: 2018-04-15T01:50:43+00:00
categories:
  - golang

---
### 定义
`Interface`是定义一组方法的集合

### 如何实现接口
任何`type`只要实现了`interface`的所有方法, 即可实现接口. 不需要像`java`需要使用`implements`关键字来显式声明.

所有的类型都实现了`empty interfa1ce`

```
type I interface {
	M()
}

type T struct {
	S string
}

// T就实现了interface I, 不需要显式声明
func (t T) M() {
	fmt.Println(t.S)
}

func main() {
	var i I = T{"hello"}
	i.M()
}
```
### Interface本质

interface本质是一个(value, type)元组. interface的value是有一个潜在的类型. 


```
type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	fmt.Println(t.S)
}

type F float64

func (f F) M() {
	fmt.Println(f)
}

func main() {
	var i I

	i = &T{"Hello"}
	describe(i)
	i.M()

	i = F(math.Pi)
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}
```

output:
```
(&{Hello}, *main.T)
Hello
(3.141592653589793, main.F)
3.141592653589793
```

### interface值是nil

如果interface的值是`nil`, 那么实现这个接口的类型的`receiver`也是`nil`

在其他语言中, 用空指针或者空对象调用对象的函数, 这将会抛出异常. 但是在go中这是很普遍的, 可以被完美的解决, 例如像下面的`M`一样. 

```
type I interface {
	M()
}

type T struct {
	S string
}

func (t *T) M() {
	if t == nil {
		fmt.Println("<nil>")
		return
	}
	fmt.Println(t.S)
}

func main() {
	var i I
	var t *T
	i = t
	describe(i)
	i.M()
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}

```
output:
```
(<nil>, *main.T)
<nil>
```

### nil interface
nil interface既没有值也没有类型

```
type I interface {
	M()
}

func main() {
	var i I
	describe(i)
	//i.M()   panic: runtime error
}

func describe(i I) {
	fmt.Printf("(%v, %T)\n", i, i)
}

```
output
```
(<nil>, <nil>)
```

**注意:**
和 **interface值是nil** 相比, **nil interface**的`i.M()`直接就崩掉了, 而**interface值是nil**的`i.M()`能够正常执行. 这是因为**interface值是nil**, 但是inteface本身却不是nil. 

### 空interface

interface没有方法的叫空interface. 空interface可以表示任何类型的值. 

```
func main() {
	var i interface{}
	describe(i) // 这里i是个nil interface 同时也是一个empty interface

	i = 42
	describe(i)

	i = "hello"
	describe(i)
}

func describe(i interface{}) {
	fmt.Printf("(%v, %T)\n", i, i)
}

```

### interface 类型断言

```
t := i.(T)
```

断言表达式的意思: `i`interface的值的具体类型是`T`, 并且把类型为T的值赋值给变量`t`. 如果interface `i`不是`T`这个类型, 那么这个表达式会`panic`

```
t, ok := i.(T)
```
判断interface是否是指定的类型, 类型断言返回两个结果: 值 和 boolean. 

如果`i`是`T`类型, 那么`t`是具体的值, `ok`是`true`

如果`i`不是`T`类型, 那么`t`将会是类型`T`的`0`值, `ok`是`false`, 不会有panic发生.

```
func main() {
	var i interface{} = "hello"

	s := i.(string)
	fmt.Println(s)

	s, ok := i.(string)
	fmt.Println(s, ok)

	f, ok := i.(float64)
	fmt.Println(f, ok)

	f = i.(float64) // panic
	fmt.Println(f)
}
```

### 类型switch

类型switch跟switch一样, 不同在于类型switch的case是指定的type(不是值).

```
switch v := i.(type) {
case T:
    // here v has type T
case S:
    // here v has type S
default:
    // no match; here v has the same type as i
}
```

### Stringers

Stringer是`fmt`定义的非常实用的interface
```
type Stringer interface {
    String() string
}
```

任何类型只要实现`Stringer`接口, 那么就可以使用`fmt`来打印值.
```
type Person struct {
	Name string
	Age  int
}

func (p Person) String() string {
	return fmt.Sprintf("%v (%v years)", p.Name, p.Age)
}

func main() {
	a := Person{"Arthur Dent", 42}
	z := Person{"Zaphod Beeblebrox", 9001}
	fmt.Println(a, z)
}

```

### Errors

```
type error interface {
    Error() string
}
```
error是内置的interface. 

```
type ErrNegativeSqrt float64

func (e ErrNegativeSqrt) Error() string {
	return fmt.Sprintf("cannot Sqrt negative number: %f", e)
}

func Sqrt(x float64) (float64, error) {
	if x < 0 {
		return x, ErrNegativeSqrt(x)
	}
	return x, nil
}

func main() {
	fmt.Println(Sqrt(2))
	fmt.Println(Sqrt(-2))
}

```