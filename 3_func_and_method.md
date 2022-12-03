本节主要讨论`Go`的函数以及方法。


* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

# function函数
`func`是`Go`中的一等公民。

所谓一等公民，也就是`func`类型可以赋值给变量、可以作为函数的参数、可以作为函数的返回值、可以自定义类型，如`net/http`包中
```go
type HandlerFunc func(ResponseWriter, *Request)
```

## 🌵函数定义的规范行为
### 若参数有context.Context类型，必做第一个参数
```go
func getContent(ctx content.Context, ...) error
```

### 若返回err，必将最后一个返回值
```go
func foo()(int, string, error)
```

### 同一类型的参数可以合并
**合并同类项**

> few params of the same type can be defined in a short way
```go
// lat and log 都是float64类型
func getCoordinate(lat, log float64) (string, error)
```

### 函数名中禁止出现`filter`？

`filter`意思着“过滤”，是一种常见的操作。为何要禁止使用？

![](media/16633077919315/16643488487757.jpg)


因为`filter`过滤本身可以有两种含义：你是想要留在筛子里面的东西呢？还是想要被筛下去的东西呢？

在我们的项目中，一般指的是：给定集合，给定筛选条件，我们需要得到符合条件的集合

可以用`match`、`qualify`等词替代

## 🚩具名返回值：谨慎使用
### 明确具名返回值本身不是函数签名的一部分

如何证明？

```go
type locator interface {
	getCoordinate(address string) (lat, lon float64, err error)
}

type Compass struct{}

func (c Compass) getCoordinate(address string) (float64, float64, error) {
	return 0, 0, nil
}

func main() {
	var l locator
	l = Compass{}

	fmt.Printf("compass is type of locator, %T", l)
}
```
证毕

### 存在的目的：为了表达清晰
```go
// bad 
func getAddress(addr string) (float64, float64, error)

// good 
// 如此一来，更加清晰
func getAddress(addr string) (lat float64, log float64, err error) 
```

### 函数体返回值的风格要统一

* 如果函数体本身比较长，就不要省略`return`的参数

> if you're going to omit returning params, do it explicitly

因为具名返回值的函数，可以不写具体的返回值

```go
// 关于如何写出不可维护的代码，我”具名返回值函数“有话说
func getLocation(address string) (lat, lng float64, err error) {
	if address == "" {
		return
	}

	if address == "666" {
		return 1, 1, nil
	} else {
		// do something 
		return lat, lng, nil
	}
}
```



## 🚩init函数：能不用就不用
首先看下`init`内置函数的特点
### `init`函数的特点

* `init`和`main`一样，均为`func()` 函数类型： 无参数、无返回值

* 在`Go`中初始化一个`package`的流程：首先初始化`package`中的常量、变量，然后就是`init`函数

* 一个`package`中可以有多个`init`函数，甚至一个文件中都可以定义多个`init`函数。按照”先到先得“的顺序执行

### `init`函数的缺点

* 无法很好的处理`init`内部的`error`，因为无法返回值

* 不能方便的测试`init`中的逻辑

* 如果在`init`中初始化资源，需要提前声明全局变量，然后赋值给它。而全局变量的使用迈向了另外的充满坑的世界。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
type Foo struct {
    // ...
}

var _defaultFoo Foo

func init() {
    _defaultFoo = Foo{
        // ...
    }
}
```

</td><td>

```go
var _defaultFoo = Foo{
    // ...
}

// or, better, for testability:

var _defaultFoo = defaultFoo()

func defaultFoo() Foo {
    return Foo{
        // ...
    }
}
```

</td></tr>
<tr><td>

```go
type Config struct {
    // ...
}

var _config Config

func init() {
    // Bad: based on current directory
    cwd, _ := os.Getwd()

    // Bad: I/O
    raw, _ := os.ReadFile(
        path.Join(cwd, "config", "config.yaml"),
    )

    yaml.Unmarshal(raw, &_config)
}
```
</td><td>

```go
type Config struct {
    // ...
}

func loadConfig() Config {
    cwd, err := os.Getwd()
    // handle err

    raw, err := os.ReadFile(
        path.Join(cwd, "config", "config.yaml"),
    )
    // handle err

    var config Config
    yaml.Unmarshal(raw, &config)

    return config
}
```
</td></tr>
</tbody></table>

### `init`万一使用，需要满足
1. Be completely deterministic, regardless of program environment or invocation.

2. Avoid depending on the ordering or side-effects of other `init()` functions.
   While `init()` ordering is well-known, code can change, and thus
   relationships between `init()` functions can make code brittle and
   error-prone.

3. Avoid accessing or manipulating global or environment state, such as machine
   information, environment variables, working directory, program
   arguments/inputs, etc.

4. Avoid I/O, including both filesystem, network, and system calls.

### 使用`init`函数的成功案例

todo Go sql
[design-patterns-in-gos-databasesql-package](https://eli.thegreenplace.net/2019/design-patterns-in-gos-databasesql-package/)

## 🚩只在`main`函数中退出
使用`os.Exit`或者`log.Fatal`退出

`log.Fatal`内部使用`os.Exit`

## 🚩尽量不要使用`new`
因为`new`能做到的，其他的方式都能做到，而且做得更好。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
sval := T{Name: "foo"}

// inconsistent
sptr := new(T)
sptr.Name = "bar"
```

</td><td>

```go
sval := T{Name: "foo"}

sptr := &T{Name: "bar"}
```

</td></tr>
</tbody></table>

## 🚩defer函数：合理使用
### defer的定义
> A defer statement defers the execution of a function until the surrounding function returns.

一句话总结：`defer`就是用来延迟执行函数的技术手段。

虽然只有短短一句话，但是我们也可以获得很多。

来来来，我们先来总结一下，（更多的内容将在下文展开论述）。

1、`defer`后面跟函数:`defers the execution of a `**`function`**。

2、`defer`的作用就是激活后接的函数:`the execution`.

3、运行时机:`until the surrounding function returns`.

> A "defer" statement invokes a function whose execution is deferred to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because the corresponding goroutine is panicking.

### 为什么要有defer
再次重申，**`defer`的本质就是延迟自动执行函数**。

它的出现就是为了更加便利的进行资源管理，减轻程序员的心智负担，提高代码的可读性、可维护性和可扩展性。

假如`Go`没有`defer`机制，为了实现一个受保护的文件操作。我们得这么写

```Go
func writeToFile(fname string, data []byte, mu *sync.Mutex) error {
    mu.Lock()
    f, err := os.OpenFile(fname, os.O_RDWR, 0666)
    if err != nil {
    mu.Unlock()
    return err
    }

    _, err = f.Seek(0, 2)
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    _, err = f.Write(data)
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    err = f.Sync()
    if err != nil {
        f.Close()
        mu.Unlock()
        return err
    }

    err = f.Close()
    if err != nil {
        mu.Unlock()
        return err
    }

    mu.Unlock()
    return nil
}
```

就像老奶奶的裹脚布——又臭又长。

程序员每次操作，都得处理资源的释放：锁的释放、文件句柄的关闭。仅仅两个资源的释放操作，就得『如临深渊，如履薄冰』。增加资源的参与数，那简直不堪设想。

幸好，我们的`Go`有`defer`机制，我们可以改写如下

```go
func writeToFile(fname string, data []byte, mu *sync.Mutex) error {

    mu.Lock()
    defer mu.Unlock()

    f, err := os.OpenFile(fname, os.O_RDWR, 0666)
    if err != nil {
        return err
    }
    defer f.Close()
    
    _, err = f.Seek(0, 2)
    if err != nil {
        return err
    }

    _, err = f.Write(data)
    if err != nil {
        return err
    }

    return f.Sync()
}
```

我们所需要做的就是申请资源之后，马上`defer+`释放资源的逻辑。当`writeToFile`运行结束时，会自动释放锁和关闭文件句柄。

### defer的特性

上面对`defer`的描述，只是非常粗糙的讲解。下面讲述为了释放资源，延迟执行，`defer`的具体规定细节。

比如

`defer`后接函数，那么函数的参数什么时候确定？（要知道同样的变量，在函数退出时与刚进入函数时，内容可能是不一样的）可以后接方法吗？

`defer`后的函数真正执行时机是：声明`defer`表达式的函数执行结束时。那么这个所谓的函数执行结束时，是返回后，还是返回前？还是其他什么时间点？

如果一个函数之中，声明了多个`defer`表达式，虽然都是函数退出时调用，但是它们执行的顺序如何？随机的？（像`select channel`一样）？先进先出？（越先被`defer`的函数越早调用）？先进后出？

来吧，让我们回答上面的问题。

不过在此之前，学习两个英文单词。

`execution`：中文是执行的意思。比如函数的执行，就是运行函数的意思。

`evaluate`：中文是评价、估计的意思。在编程里面，比如函数参数，当我们说`evaluate function parameters`，就是确定函数参数值的意思。嗯，`evaluate`可以翻译成确定。

下面进入正题

1、defered function evaluates its parameter when is occur

被`defer`的函数会立马确定后接函数参数

```go
func print(num int) {
    fmt.Println(num)
}

func foo() {
    var i = 666
    defer print(i)
    
    i = 888
    return
}

func main() {

    foo()
}
```

会打印`666`。这就叫做当出现`defer`时，后接函数的参数立马被确定，即`i=666`。虽然`defer`在后面才会执行，但是参数的确定是在一开始就确定了。

2、defered function will execute before real return

被`defer`的函数会在调用它的函数返回前执行

这句话不好理解。重点在于什么叫做`real return`。

休息一下，下面的例子需要仔细看

```go
func sara() (result int) {

    result = 1

    defer func() {
        result = 666
    }()

    return result

}

func lisa() int {

    var result = 1
    
    defer func() {
        result = 666
    }()

    return result
}

func main() {

    r := sara()
    fmt.Printf("sara finially result, %d\n", r) // 666

    r = lisa()
    fmt.Printf("lisa finially result, %d\n", r) // 1
}
```

可以观察到，`sara`和`lisa`基本逻辑是一样的，唯一的不同就是`sara`是具名返回值函数，而`lisa`是匿名返回值函数。这就是大大的不同了。

好好理解一下上面的例子，下面给出`go`函数的返回值模型

其实`go`的`return`语句是分成两部分的。比如对于匿名返回值函数`lisa`来说

```go
return result
相当于

1、var ret = result // ret 是真返回值
2、return ret
```

对于具名返回值函数sara来说

```go
return result
相当于

1、var result = result
2、return result
```

注意两者的区别。

重点来啦，`defer`的运行时机就是步骤`1`和步骤`2`之间！

所以对于`lisa`来说，我要返回的是`ret`，就算你在`defer`中修改了`result`，也不会影响到最终返回值哦。但是`sara`就不一样了，她返回的就是`result`，在`defer`中修改了`result`，最终的结果就不一样啦。

咳咳，具名返回值和`defer`的联合使用，要注意哦😑。别写这样的代码。

3、多个`defer`的执行的顺序是：后进先出，类似栈

```golang
func print(num int) {
    fmt.Println(num)
}

func bob() {
    defer print(1)
    defer print(2)
    defer print(3)

    return

}

func main() {
    bob()
}
```

会打印

```text
3
2
1
```

### 如何记住defer的特性

`defer`出现就是为了资源的管理。

比如如果有以下这个过程：首先获取资源`A`，然后获取资源`B`，最后获取资源`C`才能执行某个操作，那么`defer`的顺序，肯定是先放回`C`资源，然后`B`资源，最后是`A`资源。

这个过程让我想起来的举证的乘法和逆运算——逆矩阵。

比如`AB`，表示矩阵`A`乘以矩阵`B`，那么它俩的逆运算呢？

$(AB)^{-1} = B^{-1}A^{-1}$

当时为了记住这个特性，我是记住了下面的这个类比：

先穿袜子，然后穿裤子；如果这一过程反过来，那就是先拖鞋，再脱袜子。

`nice`。

如今看来，不论是穿衣服、矩阵乘法、资源的管理，都是一种行为、一种运动、一种变化。他们是分先后的，所以他们的“反运动”要与正向运动相反。

### 不要在循环中滥用defer
详见Go编程指引与陷阱1中的for循环章节


## 🚩对函数进行显示类型转换
像对整型变量那样`int64(13240366)`
```go
func greeting(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome, Gopher!\n")
}

func main() {
    // 直接传递greeting是会出错的!

	//http.ListenAndServe(":8080", http.HandlerFunc(greeting))
	http.ListenAndServe(":8080", greeting) 
}
```
`ps`：经过`http.HandlerFunc`转换之后就可以用了哈。

(如果对此感兴趣，可参考本人拙作[Go语言系列之HTTP服务器](https://zhuanlan.zhihu.com/p/588192056?))

本质上相同的行为
```go
type BinaryAdder interface {
	Add(int, int) int
}

// 实现BinaryAdder接口
type MyAdderFunc func(int, int) int

func (f MyAdderFunc) Add(x, y int) int {
	return f(x, y)
}

func MyAdd(x, y int) int {
	return x + y
}

func main() {
	var i BinaryAdder = MyAdderFunc(MyAdd)
	fmt.Println(i.Add(5, 6))
}
```

## 🚩了解变长参数的妙用
### 基本特点-常见
如同空气一般，常见常用但往往没有引起人们的注意。
比如
```go
func Printf(format string, a ...interface{}) (n int, err error) 

func Sprintf(format string, a ...interface{}) string 

func append(slice []Type, elems ...Type) []Type
```
### 基本使用
* 函数内使用-如同切片一般
```go
func sum(arr ...int) int {
	var t int
	for _, n := range arr {
		t += n
	}
	return t
}
```

* 函数外使用

1、可`t...`使用，其中`t`是`[]T`切片类型

2、可多个`T`使用
```go
// ok
sum(1,2,3)
sum([]int{1,2,3}...)
	
// not ok 
sum([3]int{1,2,3}...) // Cannot use '[3]int{1,2,3}' (type [3]int) as the type []int
```

### 实参与形参不匹配的问题
```go
func dump(args ...interface{}) {
	for _, v := range args {
		fmt.Println(v)
	}
}

func main() {
	//s := []string{"Tony", "John", "Jim"} // not ok ，类型不匹配
	s := []interface{}{"Tony", "John", "Jim"}
	dump(s...)
}
```
`ps`:都怪标准库里面的`printf`😂

### 变长参数+类型验证=重载函数
* 一个我们需要的拼接函数
```go

// SliceJoin 重载之
func SliceJoin(sep string, nums ...interface{}) string {
	if len(nums) == 0 {
		return ""
	}

	var arr = make([]string, 0, len(nums))
	for _, num := range nums {
		switch num.(type) {
		case int64, uint32, int:
			arr = append(arr, fmt.Sprintf("%d", num))
		case string:
			arr = append(arr, fmt.Sprintf("%s", num))

		case []string:
			arr = append(arr, num.([]string)...)
		case []uint32:
			for _, e := range num.([]uint32) {
				arr = append(arr, fmt.Sprintf("%d", e))
			}
		case []int64:
			for _, e := range num.([]int64) {
				arr = append(arr, fmt.Sprintf("%d", e))
			}
		case []int:
			for _, e := range num.([]int) {
				arr = append(arr, fmt.Sprintf("%d", e))
			}
		}
	}
	return strings.Join(arr, sep)
}

func main() {
	fmt.Println("int", SliceJoin(",", []int{1, 2, 3}))                 // int 1,2,3
	fmt.Println("int64", SliceJoin(",", []int64{1, 2, 3}))             // int64 1,2,3
	fmt.Println("uint32", SliceJoin(",", []uint32{1, 2, 3}))           // uint32 1,2,3
	fmt.Println(SliceJoin("-", 1, 2, "sr", "z7", "x", []int{1, 2, 3})) // 1-2-sr-z7-x-1-2-3
}
```

### 功能选项模式
如何构建一个合理的构建函数？

尤其是当构建的`struct`有多个字段，不同的调用方需要不同的配置。我们怎么做才能提供一个合理的构建函数呢？

这就是本节需要讨论的问题。
```go
// 房子装修风格
type FinishedHouse struct {
	style                  int    // 0: Chinese, 1: American, 2: European
	centralAirConditioning bool   // true or false
	floorMaterial          string // "ground-tile" or ”wood"
	wallMaterial           string // "latex" or "paper" or "diatom-mud"
}

type Option func(*FinishedHouse) // 无返回值

func NewFinishedHouse(options ...Option) *FinishedHouse {
	h := &FinishedHouse{
		// 默认选项
		style:                  0,
		centralAirConditioning: true,
		floorMaterial:          "wood",
		wallMaterial:           "paper",
	}

	for _, option := range options {
		option(h)
	}

	return h
}

func WithStyle(style int) Option {
	return func(h *FinishedHouse) {
		h.style = style
	}
}

func WithFloorMaterial(material string) Option {
	return func(h *FinishedHouse) {
		h.floorMaterial = material
	}
}

func WithWallMaterial(material string) Option {
	return func(h *FinishedHouse) {
		h.wallMaterial = material
	}
}

func WithCentralAirConditioning(centralAirConditioning bool) Option {
	return func(h *FinishedHouse) {
		h.centralAirConditioning = centralAirConditioning
	}
}

func main() {
	fmt.Printf("%+v\n", NewFinishedHouse()) // use default options
	fmt.Printf("%+v\n", NewFinishedHouse(WithStyle(1),
		WithFloorMaterial("ground-tile"),
		WithCentralAirConditioning(false)))
}
```

# struct 

## 🌵1、使用var声明零值的struct
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


## 🌵2、绑定到struct方法的排列顺序
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

**内嵌结构体最大的特点就是暴露了里层的方法，哪怕这些方法并不想被外面的调用者使用。**

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
	o := Outer{}

	fmt.Println(o.in) // 直接引用Inner的字段
	o.in = "555"      // 直接设置Inner的字段

	// 可以在初始化的时候赋值in字段吗？
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

内嵌的`Mutex`的`Lock`、`Unlock`方法被暴露给调用者

The `Mutex` field, and the `Lock` and `Unlock` methods are unintentionally part
of the exported API of `SMap`.

</td><td>

The mutex and its methods are implementation details of `SMap` hidden from its
callers.

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
`Go`方法的本质：就是以`receiver`类型的实例作为第一个参数的**函数**。

## 🚩类型与其方法定义在同一个包

## 🚩可以绑定method的类型
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

## 🚩明晰value & pointer receiver

* 什么是`value receiver`,什么是`pointer receiver`？
```go
type Person struct {
	Name string
}

// value receiver
func (p Person) GetName() string {
	return p.Name
}

// pointer receiver
func (p *Person) SetName(name string) {
	p.Name = name
}
```

下面来讨论两者之间的区别。

其实`Go`的`value/pointer receiver`区别非常明显、简单。总结来说，就是如下：

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

根本原因是Go会默默的做隐式转换。而我认为，**这个问题因为Go的隐式转换将其复杂化了**。

* `Go`会帮我们做默认的转化。

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

**Methods with value receivers can be called on pointers as well as values. Methods with pointer receivers can only be called on pointers or addressable values.**
为什么要有以上的规定？

## 🚩value receiver是需要复制类型的
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
## 🚩修改类型状态要用pointer receiver
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
这段程序会panic。但只要修改一个字符就可以完美运行。

## 🌵method的接受者类型最好统一
**要么全是`method with value`，要么全是`method with pointer`**

当然是有例外的
todo

# empty interface 空接口

我觉得在Go中，空接口与非空接口完全表现为两种不同的事物，所以我打算分开来讨论。

## 🚩尽量避免使用空接口作为函数的参数类型

> 空接口不提供任何信息。   —— Rob Pike, Go语言之父

# interface 非空接口

**中心思想：接口中的方法越多，这个接口越不通用**

> The bigger the interface, the weaker the abstraction.

## 🌵name
通常来说，如果`interface`只有一个`method`，那么该`interface`的名字：`method`的名称`+er`，比如`Reader`、`Writer` 等等

> By convention, one-method interfaces are named by the method name plus an -er suffix or

## 🌵显式验证：某个类型实现指定的接口
```go
// 验证structA是否实现了interfaceA
var _ interfaceA = (*structA)(nil)
```
体现了`interface`类型的静态性。

既然是静态性，也就是编译期间的检查，一些运行时的出错就在所难免。

所谓的运行时错误，那就是内嵌`interface`的时候。
```go
type MyStruct struct{}

// 内嵌error interface
type MyError struct {
	error
}

func main() {
	var _ error = (*MyStruct)(nil) // *MyStruct does not implement error (missing Error method)
	
	var _ error = (*MyError)(nil) // ok 
	
	var e error = &MyError{}  // ok

	fmt.Println(e.Error()) // panic: runtime error: invalid memory address or nil pointer dereference
}
```

## 🚩内嵌interface

内嵌`interface`一共分为两种形式

* `interface`内嵌`interface`
* `struct`内嵌`interface`

第一种，`interface`内嵌`interface`很常见，也很容易理解。比如
```go
// io/io.go
type Reader interface {
	Read(p []byte) (n int, err error)
}

type Writer interface {
	Write(p []byte) (n int, err error)
}

// 内嵌interface，扩大interface的方法
type ReadWriter interface {
	Reader
	Writer
}
```

关键在于第二种，`struct`内嵌`interface`，这是什么组合？这有什么道理？
todo:[文章阅读](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs)/


## 🚩当心直接返回interface的情况
```go
type MyError struct {
	error
}

var ErrBad = MyError{
	error: errors.New("bad error"),
}

func bad() bool {
	return false
}

func returnsError() error {
	var p *MyError = nil // 📢
	if bad() {
		p = &ErrBad
	}
	return p
}

func main() {
	e := returnsError()
	if e != nil {
		fmt.Printf("error: %+v\n", e) // error: <nil>
		return
	}
	fmt.Println("ok")
}
```
体现了`interface`的动态性

其动态性包含两部分：动态类型、动态类型的值。

此例子就是因为返回的`error`接口类型的动态类型不为`nil`，`main`里面的`e`就不为`nil`。

## 🚩尽量定义小接口

> 接口越大，抽象程度越低。（一个人的能力越大，他的责任也就越大）

但是我们不得不说，在项目初期，很难分辨和抽象出实用的小接口。也就是说初期可不在意接口的大小，因为对问题的理解是循序渐进的，毕竟标准库的`io.Reader/io.Writer`也不是一开始就确定的。




## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)
