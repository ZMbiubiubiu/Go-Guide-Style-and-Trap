本节主要讨论`Go`的函数以及方法。


* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。
  
  
- [struct](#struct)
  - [🌵1、使用 var 声明零值的 struct](#🌵1、使用-var声明零值的-struct)
  - [🌵2、绑定到 struct 方法的排列顺序](#🌵2、绑定到-struct方法的排列顺序)
  - [🚩3、最好不要内嵌结构体](#🚩3、最好不要内嵌结构体)
    - [内嵌结构体](#内嵌结构体)
    - [内嵌结构体的缺点](#内嵌结构体的缺点)
- [method](#method)
  - [🚩method 的本质](#🚩method的本质)
  - [🚩明晰 values/pointer method 对 receiver 的影响](#🚩明晰valuespointer-method对-receiver的影响)
  - [🚩明晰 value vs pointer method](#🚩明晰value-vs-pointer-method)
  - [🚩value receiver 调用方法，是需要复制类型的](#🚩value-receiver调用方法，是需要复制类型的)
  - [🚩pointer receiver 调用方法可以修改状态值](#🚩pointer-receiver调用方法可以修改状态值)
  - [🚩可以绑定 method 的类型](#🚩可以绑定method的类型)
  - [🚩method定义的规范行为](#🚩method定义的规范行为)
    - [类型与其方法定义在同一个包](#类型与其方法定义在同一个包)
  - [method 的接受者类型最好统一](#method的接受者类型最好统一)
  - [参考](#参考)


# struct 

## 🌵1、使用 var 声明零值的 struct
看起来会更简洁，大家觉得呢？
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
s := structA{}
```

</td><td>

```go
var s structA
```

</td></tr>
</tbody></table>


## 🌵2、绑定到 struct 方法的排列顺序
* `struct`声明之后，如果有工厂函数，紧随着`struct`
* 向外暴露的方法排在前面
* 不向外暴露的方法排在最后面

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n []int) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}
```

</td><td>

```go
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

</td></tr>
</tbody></table>

## 🚩3、最好不要内嵌结构体
### 内嵌结构体
```go
type StructB struct {
    A // A is another struct
}
```
内嵌结构体是`Go`提供一种介于继承和组合之间的折中方案。

外层的结构体完全复制了里层的结构体的方法。

**内嵌结构体最大的特点以及缺点就是暴露了被潜入结构体的方法，哪怕这些方法并不想被外面的调用者使用。**

### 内嵌结构体的缺点
* 初始化不方便
```go
type Inner struct {
	in string
}

type Outer struct {
	Inner
}

func main() {
  // 如果想给内嵌结构体Inner的字段赋值，需要两步
	o := Outer{}
	fmt.Println(o.in) // 直接引用Inner的字段
	o.in = "555"      
	
	// 不可以在初始化的时候直接赋值in字段
	_ := Outer{in: "666"} // Unknown field 'in' in struct literal 
}
```

* 可能暴露出不想被外界调用的方法

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type SMap struct {
  sync.Mutex // 内嵌Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.Lock()
  defer m.Unlock()

  return m.data[k]
}
```

</td><td>

```go
type SMap struct {
  mu sync.Mutex

  data map[string]string
}

func NewSMap() *SMap {
  return &SMap{
    data: make(map[string]string),
  }
}

func (m *SMap) Get(k string) string {
  m.mu.Lock()
  defer m.mu.Unlock()

  return m.data[k]
}
```

</td></tr>

<tr><td>

内嵌的`Mutex`的`Lock`、`Unlock`方法被暴露给调用者。

可能会发生内部实现和外部使用者同时调用`Lock/Unlock`的情形。
</td><td>
标准的支持并发读写的map实现。
</td></tr>
</tbody></table>


* `json.Marshal`结果的不确定性
```go
package main

import (
	"encoding/json"
	"fmt"
	"github.com/davecgh/go-spew/spew"
	"time"
)

type Event struct {
	Id int64 `json:"id"`
	time.Time
}

func main() {
	e := Event{
		Id:   1,
		Time: time.Now(),
	}

	b, err := json.Marshal(&e)
	if err != nil {
		spew.Dump(err)
	}
	fmt.Println(string(b)) 
	
	// "2022-09-17T19:11:35.07228+08:00"
}
```


# method

## 🚩method 的本质

`method`就是以`receiver`作为第一个参数的**函数**。没错，`method`就是函数。

## 🚩明晰 values/pointer method 对 receiver 的影响

什么是`value method`,什么是`pointer method`，直接看示例

```go
type Person struct {
	Name string
}

// value method
func (p Person) GetName() string {
	return p.Name
}

// pointer receiver
func (p *Person) SetName(name string) {
	p.Name = name
}
```

明白了`value method`和`pointer method`，回到上面提到的「`method`本质上就是将`receiver`作为第一个参数的函数」。

那么既然是函数参数，就存在函数参数复制的问题。

下面通过一段示例代码来看下这个问题。
```go
package main

import "fmt"

type Container struct {
  i int
  s string
}

func (c Container) byValMethod() {
  fmt.Printf("byValMethod got &c=%p, &(c.s)=%p\n", &c, &(c.s))
}

func (c *Container) byPtrMethod() {
  fmt.Printf("byPtrMethod got &c=%p, &(c.s)=%p\n", c, &(c.s)) // 注意这里第一个c没有取地址符号
}

func main() {
  var c Container
  fmt.Printf("in main &c=%p, &(c.s)=%p\n", &c, &(c.s))

  c.byValMethod()
  c.byPtrMethod()
}

// 输出结果
in main &c=0xc00000a060, &(c.s)=0xc00000a068
byValMethod got &c=0xc00000a080, &(c.s)=0xc00000a088
byPtrMethod got &c=0xc00000a060, &(c.s)=0xc00000a068
```

* `byValMethod`中打印的地址与`main`中完全不同，这是因为它的`receiver`是通过`c`复制产生的。
* `byPtrMethod`中打印的地址与`main`中完全相同，这是因为它本身打印的就是`c`的地址。



## 🚩明晰 value vs pointer method

这一节主要讨论两者之间的区别。

其实`Go`的`value/pointer method`区别非常明显。如下：

> value methods can be invoked by value and pointer receiver, but pointer methods can be only invoked by pointer receiver.

翻译一下就是
 ```go
// value receiver
value := Person{}

// pointer receiver
pointer := &Person{}

// value methods can be invoked by value and pointer receiver
// GetName是一个value method
value.GetName()
pointer.GetName()

//	pointer methods can be only invoked by pointer receiver.
// SetName是一个pointer method
pointer.SetName("xxx")
 ```
如上所述，这很好理解。

但让人震惊的是，我们也可以这么用
 ```go
 value.SetName("why") // it's ok,不会报错
 ```

上面不是说`pointer methods can be only invoked by pointer receiver.`吗？

`SetName`作为一个`pointer method`，被`pointer receiver`调用，这没问题。为什么也能被`value receiver`调用？这究竟是为什么？

**根本原因是`Go`会默默的做隐式转换。**

```go
// value methods can be invoked by value and pointer receiver
// GetName是一个value method
value.GetName()
pointer.GetName()  // 相当于(*pointer).GetName()

//	pointer methods can be only invoked by pointer receiver.
// SetName是一个pointer method
pointer.SetName("xxx") 
value.SetName("why") // 相当于(&value).SetName("why")
```
看明白了吗。`Go`会帮助我们在适当的位置添加`&`、`*`操作符。

而我认为，**这个问题因为Go的隐式转换将其复杂化了**。

再看下面的例子：

```go
// 不管是val / pointer receiver,都可以通过语法糖的形式调用 pointer/val receiver method

package main

import "fmt"

type printer interface {
	print()
}

// method with value receiver
type S struct{}

func (s S) print() {
	fmt.Println("hello S")
}

// method with pointer receiver
type P struct{}

func (p *P) print() {
	fmt.Println("hello P")
}

func main() {
		// step 1
	s := S{}
	s.print()

	ps := &S{}
	ps.print() // (*ps).print()

	p := P{}
	p.print() // (&p).print()

	pp := &P{}
	pp.print()

	// step 2
	m := map[int]P{1: {}}
	m[1].print() // cannot take the address of m[1]

	// step 3
	var _ printer = S{}
	var _ printer = &S{}

	var _ printer = P{} // compile error
	var _ printer = &P{}
}
```

想必第一步的测试，我们已经懂得了。

第二步，`Go`本来也想帮我们取`P struct`实例的地址，以便符合`printer`接口，但是因为`Go map`的值是不能取地址，所以编译出错。

关于`Go map`键值对的值不能被取地址，可以参考本系列第一节「基础数据结构」中`map`部分的讲解。

总结来看：

**Methods with value receivers can be called on pointers as well as values. Methods with pointer receivers can only be called on pointers or addressable values.**

* 指针的`*`解引用，不会出错，是一定成立的。
* 值对象取地址操作就不是一定成立的了。

## 🚩value receiver 调用方法，是需要复制类型的

原因很简单，因为`method`就是`function`，`value receiver`作为函数的第一个参数，本质上就是复制。

```go
package main

import "fmt"

type Container struct {
  i int
  s string
}

func (c Container) byValMethod() {
  fmt.Printf("byValMethod got &c=%p, &(c.s)=%p\n", &c, &(c.s))
}

func (c *Container) byPtrMethod() {
  fmt.Printf("byPtrMethod got &c=%p, &(c.s)=%p\n", c, &(c.s))
}

func main() {
  var c Container
  fmt.Printf("in main &c=%p, &(c.s)=%p\n", &c, &(c.s))

  c.byValMethod()
  c.byPtrMethod()
}

// result 
in main &c=0xc00000a060, &(c.s)=0xc00000a068
byValMethod got &c=0xc00000a080, &(c.s)=0xc00000a088
byPtrMethod got &c=0xc00000a060, &(c.s)=0xc00000a068
```
## 🚩pointer receiver 调用方法可以修改状态值
因为传递的第一个参数，本来就是指针类型，内部当然可以修改其状态（字段值）。

```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Container struct {
  sync.Mutex                       // <-- Added a mutex
  counters map[string]int
}

func (c Container) inc(name string) {
  c.Lock()                         // <-- Added locking of the mutex
  defer c.Unlock()
  c.counters[name]++
}

func main() {
  c := Container{counters: map[string]int{"a": 0, "b": 0}}

  doIncrement := func(name string, n int) {
    for i := 0; i < n; i++ {
      c.inc(name)
    }
  }

  go doIncrement("a", 100000)
  go doIncrement("a", 100000)

  // Wait a bit for the goroutines to finish
  time.Sleep(300 * time.Millisecond)
  fmt.Println(c.counters)
}
```
这段程序会`panic`。但只要修改一个字符就可以完美运行。

## 🚩可以绑定 method 的类型
`receiver`参数的基类型本身不能是指针类型/接口类型

> methods can be defined for any named type (except a pointer or an interface); the receiver does not have to be a struct.
> -- < Effective Go >

```go
// pointer指针不能绑定method
type PtrInt *int

// error: Invalid receiver type 'PtrInt' ('PtrInt' is a pointer type)
func (p PtrInt) Hello() {
	fmt.Println("hello")
} 
```

## 🚩method定义的规范行为

### 类型与其方法定义在同一个包

别给他们搞分家。

## method 的接受者类型最好统一
**要么全是`method with value`，要么全是`method with pointer`**

当然是有例外的
todo


## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)