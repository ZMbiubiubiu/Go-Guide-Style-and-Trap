- [前言](#前言)
  - [约定](#约定)
  - [总览](#总览)
- [一、interface in interface](#一、interface-in-interface)
- [二、interface in struct](#二、interface-in-struct)
  - [什么是 interface in struct](#什么是interface-in-struct)
  - [为什么需要这种内嵌方式](#为什么需要这种内嵌方式)
  - [案例：sort.Reverse](#案例：sort-reverse)
  - [案例：context.WithValue](#案例：context-withvalue)
- [三、struct in struct](#三、struct-in-struct)
  - [什么是 struct in struct](#什么是struct-in-struct)
  - [方法与字段的引用](#方法与字段的引用)
  - [案例：crypto.lruSessionCache](#案例：crypto-lrusessioncache)
  - [案例：elf.File](#案例：elf-file)
- [参考](#参考)


# 前言

## 约定

内嵌：`embedding`

提升：`promoted`

* 关键词

`Go`、`embedding`、内嵌、标准库

## 总览

在学习`Go`语法以及观摩标准库代码时，我们时常会遇到”内嵌“这种代码组织方式。比如下面的`reverse`就内嵌了一个名为`Interface`的东西。
```go
// sort/sort.go

type reverse struct {
	Interface
}
```

搞懂内嵌结构是写出优雅`Go`代码的必要条件。

下面列出内嵌的种类：

1、`structs in structs`

2、`interface in interface`

3、`interface in struct` 

`Go`的内嵌结构一种有三种情况，分为内嵌`interface`的两种情形以及内嵌`struct`的一种情形。

不论是上面的哪种情况，本质上，在`Go`中，`embedding = promoted`。

具体来说，**就是把属于里层（被嵌套的对象）的所有方法、属性提升(`promoted`)到外层**。

下文我们会详细论证内嵌的三种类型，并通过实际案例来描述与证明。

最后，希望读完本文后我们能够接受`Go`的内嵌，并在日常工作中使用这种优雅的用法，以便写出更好的代码~


# 一、interface in interface

这种是十分常见的。标准库中有非常多这样的用法：

```go
// io/io.go

type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// method 1
type ReadWriter interface {
    Read(p []byte) (n int, err error)
    Write(p []byte) (n int, err error)
}

// method 2 
type ReadWriter interface {
  Reader
  Writer
}
```

正如代码中所暗示的，`method1`和`method2`将达到同样的效果。不过使用`interface in interface`更加方便，而且代码不重复，容易维护。

总结，`interface in interface`就是将被包裹的`interface`的方法集提升到外层`interface`，使外层`interface`成为方法”超集“。

# 二、interface in struct

## 什么是 interface in struct

乍一看，这种内嵌相比`interface in interface`，实在有些匪夷所思，不太能看懂到底发挥了什么作用。

不过让我们慢慢揭开它神秘的面纱吧。

当我们在代码中写明
```go
// 内嵌error接口
type MyError struct {
	error 
}
```

相当于默认实现了下述方法
```go
func (m MyError) Error() string {
    return m.error.Error()
}
```

没错，`interface in struct`和`interface in interface`从本质上来说没有任何不同。均是达到同一目的：**把属于里层的所有方法提升(`promoted`)到外层**。

* 里层：指的就是被嵌套的`interface`。
* 外层：指的就是包裹层，可以是`interface`和`struct`。

而我们应该如何使用这种`interface in struct`的内嵌方式呢？

来，随我吟唱一下：

**需要给`struct`包裹的`interface`赋值一个实现接口方法集的`interface`实例**。
```go
var ErrBad = MyError{
	error: errors.New("bad error"),
}
```

若未给包裹的`interface`赋值满足接口要求的实例，而直接调用会发生什么呢？

和你想的没错，也和上面实例展示的一样，会报错。
```go
var e error = &MyError{}  // ok

fmt.Println(e.Error()) // panic: runtime error: invalid memory address or nil pointer dereference
```

因为被嵌套的`interface`未被赋值，所以保持它的零值状态——`nil`。直接调用肯定报错呀~

```go
func (m MyError) Error() string {
    return m.error.Error() // 相当于 m.error是一个nil啊，空指针。
}
```

好了道理我们都懂，也明白`interface in struct`类似`interface in interface`。

问题是：

**我们为什么需要这种`interface in struct`的内嵌方式呢？它能给我们带来什么好处吗？**

## 为什么需要这种内嵌方式

一句话总结：**`interface in struct`允许我们`DIY`定制化接口方法集中的特定功能**。

下面我们来详细说说。

在一个成熟的系统中，同时正如我们在大多数函数或方法中看到的那样，传递的、返回的是`interface`类型，并且通常来说系统中已经定义了符合这种`interface`的实例。但我们有时会有特定的需求，仅仅稍微修改`interface`中的一个方法，这一点有时候非常困难，难道我们需要从头到尾捯饬一个满足`interface`的实例吗？

说起来太抽象了，我们举例
```go
// net/dial.go

// Dial connects to the address on the named network.
func Dial(network, address string) (Conn, error) {
	var d Dialer
	return d.Dial(network, address)
}
```
其中返回的`net.Conn`便是一个`interface`接口，表示一个控制层的连接。它可以用在很多地方。`net`包中很多地方可以看见它的身影，比如从连接上读数据、写数据、关闭连接等地方。

那么好了，假如现在我们接手一个特定的需求：

我们需要统计`TCP`连接上读取到的所有字符。

我们来分析一下这个需求：

1、实现的新示例`X`，满足`net.Conn`接口，这意味着只要`net.Conn`能够使用的地方，`X`也能使用。

2、每次读取数据的时候，累计读到的字节数，也就是重新定义`net.Conn`方法集中的`Read`。

没错，这就叫做定制化接口功能，可怎么办？

为了实现这个需求，我们可以定义一个结构体，然后实现`net.Conn`接口的所有方法。

不过等等，`net.Conn`接口可是有`8`个方法啊！
```go
type Conn interface {
	Read(b []byte) (n int, err error)

	Write(b []byte) (n int, err error)

	Close() error

	LocalAddr() Addr

	RemoteAddr() Addr

	SetDeadline(t time.Time) error

	SetReadDeadline(t time.Time) error

	SetWriteDeadline(t time.Time) error
}
```
而且这`8`个方法，可不是打印`hello world`那么简单！

啊哈，这就体现`interface in struct`内嵌方式的重要性和必要性。

我们直接来看解决方案
```go
type StatsConn struct {
	net.Conn

	BytesRead int64
}

// 重新定义Read
func (s *StatsConn) Read(p []byte) (int, error) {
	n, err := s.Conn.Read(p)
	s.BytesRead += int64(n)

	return n, err
}

func main() {

	conn, err := net.Dial("tcp", ":8080")
	if err != nil {
		log.Fatalln(err)
	}

	// 使用的方式
	sconn := &StatsConn{
		Conn:      conn,
		BytesRead: 0,
	}

	// 你想怎么用就怎么用吧
	_ = sconn
}
```

所谓的「统计读到的总字节数」就是`DIY net.Conn`的`Read`方法嘛。

鬼斧神工！如果看不懂就再看一遍。

下面请欣赏，`Go`的标准库是如何将`interface in struct`玩的神乎其技，骚气蓬勃的。

## 案例：sort.Reverse

我们知道`sort.Sort`的作用是给传进来的集合进行排序。当然需要满足一定条件，即满足下面的接口要求
```go
// sort/sort.go

func Sort(data Interface) {
	n := data.Len()
	quickSort(data, 0, n, maxDepth(n))
}

type Interface interface {
	Len() int
	Less(i, j int) bool
	Swap(i, j int)
}
```

所谓的“反”排序，不过就是`DIY sort.Interface`的`Less`方法嘛。

查看具体实现之前，大家可以思考一下如何实现。

```go
// sort/sort.go

// Reverse returns the reverse order for data.
func Reverse(data Interface) Interface {
	return &reverse{data}
}

type reverse struct {
	// This embedded Interface permits Reverse to use the methods of
	// another Interface implementation.
	Interface
}

// Less returns the opposite of the embedded implementation's Less method.
func (r reverse) Less(i, j int) bool {
	return r.Interface.Less(j, i)
}
```

如何使用呢？

```go
arr := []int{5, 2, 9, 1, 6, 3}

sort.Sort(sort.Reverse(sort.IntSlice(arr)))

fmt.Println(arr) // [9 6 5 3 2 1]
```

> reverse implements sort.Interface by means of embedding it (as long as it's initialized with a value implementing the interface), and it intercepts a single method from that interface - Less.

很简单吧，大家看懂了吗？

## 案例：context.WithValue

```go
// context/context.go

func WithValue(parent Context, key, val interface{}) Context {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	if key == nil {
		panic("nil key")
	}
	if !reflectlite.TypeOf(key).Comparable() {
		panic("key is not comparable")
	}
    
    // 忽略上面的错误情况检查，直击目标
	return &valueCtx{parent, key, val}
}
```

你或许已经猜到了，重点就在于这个`*valueCtx`。
```go
// A valueCtx carries a key-value pair. It implements Value for that key and
// delegates all other calls to the embedded Context.
type valueCtx struct {
	Context
	key, val interface{}
}
```
没错，内嵌`interface`的情形又出现了。

如此一来，`*valueCtx`可以定制化Context的四个方法了。我们来看看

```go
func (c *valueCtx) Value(key interface{}) interface{} {
	if c.key == key {
		return c.val
	}
	return c.Context.Value(key)
}
```

> valueCtx now implements the Context interface and is free to intercept any of Context's 4 methods. It intercepts Value


# 三、struct in struct

## 什么是 struct in struct
```go
type Base struct {
	b int
}

type Container struct {
	Base
	c int
}

func main() {
	// 初始化方式：1
	container := Container{}
	container.b = 1 // 果然，Base的b如同提升到Container中
	container.c = 2

	// 初始化方式：2 字面量值初始化
	con := Container{
		Base: Base{b: 666},
		c:    777,
	}

	fmt.Printf("con1:%+v, con2:%+v\n", container, con)
}
```
在字面值初始化的时候，只能像上面那样，全部展开进行赋值。

可以将`container.b`当做一种语法糖。
```go
container.b == container.Base.b
```

## 方法与字段的引用

`embedding structs`调用方法和引用字段值时，总体上符合“就近原则”

我们先来学习几个英文单词

* `promoted field`：直译就是提升字段。

拿上面的例子来说，就是`container`可以直接引用`container.b`，字段`b`是`Base`下的字段，对于`container`来说，字段`b`就是被提升了

* `shadowed filed`：直译就是被遮盖。

```go
type Base struct {
	b int
}

type Container struct {
	Base
	b int
	c int
}

func main() {
	con := Container{
		Base: Base{b: 0},
		b:    666,
		c:    777,
	}

	fmt.Println(con.b) // 666
}
```
`con.b`为`666`，说明`container`取到了自身的字段`b`，而不是内嵌`Base`的字段`b`的值。

上面讲的是`promoted/shadowed field`，其实对于`method`也有同样的问题啦。

* `protomed method`
```go
type Base struct {
	b int
}

func (b Base) Describe() {
	fmt.Printf("Base b is %d\n", b.b)
}

type Container struct {
	Base
	c int
}

func main() {
	con := Container{
		Base: Base{b: 0},
		c:    777,
	}

	con.Describe()
}
```

`shadowed method`我想不用说了，道理是一脉相传的。

## 案例：crypto.lruSessionCache

```go
// crypto/common.go

type lruSessionCache struct {
  sync.Mutex
  m        map[string]*list.Element
  q        *list.List
  capacity int
}
```
评价：如此一来。`lruSessionCache`实例上就可以调用`Lock/Unlock`方法了。

我知道，这里想用锁机制保护并发情况下的更新。

不过这真的是你想要的吗？你完全可以如此操作，使用组合的方式。
```go
type lruSessionCache struct {
  mu sync.Mutex
  m        map[string]*list.Element
  q        *list.List
  capacity int
}
```


## 案例：elf.File

```go
// debug/elf/file.go

// A FileHeader represents an ELF file header.
type FileHeader struct {
  Class      Class
  Data       Data
  Version    Version
  OSABI      OSABI
  ABIVersion uint8
  ByteOrder  binary.ByteOrder
  Type       Type
  Machine    Machine
  Entry      uint64
}

// A File represents an open ELF file.
type File struct {
  FileHeader
  Sections  []*Section
  Progs     []*Prog
  closer    io.Closer
  gnuNeed   []verneed
  gnuVersym []byte
}
```
评价：完全可以在`File`将所有字段平铺开来，但是`elf package`的作者没有这么做。

`FileHeader`与其他信息分割开来，这是有意义的且自说明的。

# 参考
* [Embedding in Go: Part 1 - structs in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-1-structs-in-structs/)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)