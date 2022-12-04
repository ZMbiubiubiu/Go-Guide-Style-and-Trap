本节主要讨论`Go`的`interface`，分为`empty interface`和`normal interface`。

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

- [interface](#interface)
  - [🌵类型满足 interface 接口不需要显示定义](#🌵类型满足interface接口不需要显示定义)
  - [🌵显式的验证类型实例是否满足指定接口](#🌵显式的验证类型实例是否满足指定接口)
  - [🚩内嵌 interface](#🚩内嵌interface)
  - [🚩interface 变量的内部表示](#🚩interface变量的内部表示)
  - [🚩当心直接返回 interface 的情况](#🚩当心直接返回interface的情况)
  - [🚩尽量避免使用空接口作为函数的参数类型](#🚩尽量避免使用空接口作为函数的参数类型)
  - [🌵interface 命名的惯例](#🌵interface命名的惯例)
  - [🌵尽量定义小接口](#🌵尽量定义小接口)
  - [🚩使用接口作为代码水平组合的连接点](#🚩使用接口作为代码水平组合的连接点)
- [参考](#参考)


## interface

`Go`语言推崇面向组合编程，而`interface`是`Go`语言中实践组合编程的重要手段。

`Go`的`interface`真的有太多东西要说，请听我慢慢道来~

### 🌵类型满足 interface 接口不需要显示定义

> A type satisfies an interface if it possesses all the methods the interface requires.

`Go`的类型只要实现了指定`interface`中要求的方法集合，那么就默认了该类型满足指定`interface`。

直接来看我们平时用的最多的函数家族——`fmt`包下的格式化函数们。
```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error)

func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```
我们可以看到`Printf`函数内部直接调用`Fprintf`，并且第一个参数传递`os.Stdout`。

之所以能这么用，需要：`os.Stdout`满足`io.Writer`接口。

首先看下`io.Writer`接口的定义。
```go
// io/io.go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

其次看下`os.Stdout`是何物。
```go
// os/file.go
Stdout = NewFile(uintptr(syscall.Stdout), "/dev/stdout")

func NewFile(fd uintptr, name string) *File
```
`os.Stdout`是`*os.File`类型，那么它是否真的实现了`io.Writer`接口中的`Write`方法呢？

我们继续跟踪。
```go
// os/file.go

// Write writes len(b) bytes to the File.
// It returns the number of bytes written and an error, if any.
// Write returns a non-nil error when n != len(b).
func (f *File) Write(b []byte) (n int, err error)
```

正如大家所见，`*os.File`类型只是定义了`Write`方法，并没有显式的声明它实现了`io.Writer`接口哈。

这就是所谓的`satified implicitly`。

> What makes Go's interfaces so distinctive is that they are satisfied implicitly.  
> -- 《The Go Programming Language》

行吧，我们已经接受在`Go`中，接口的满足与否都是默默工作的。那如果我们想要显式判断，该如何做呢？

### 🌵显式的验证类型实例是否满足指定接口
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

为了阐明这个代码中的内嵌`interface`而产生的运行时空指针的问题，我们需要讨论两个问题：

* 什么是内嵌`interface`？
* `interface`变量在运行时如何表示？

### 🚩内嵌 interface

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

关键在于第二种，`struct`内嵌`interface`，这有什么道理？
todo:[文章阅读](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs)/



### 🚩interface 变量的内部表示

`interface`变量一共分为两种：

* `empty interface`空接口变量

如`var i interface{} = 1`，`i`为空接口变量。

* `normal interface`非空接口变量，所谓非空接口就是作为一个`interface`，内部的方法集的个数不为`0`

如`var err error = nil`，`err`为非空接口变量。

首先看一下，`Go`的`runtime`时`interface`变量的内部表示。
```go
// runtime/tuntime2.go

// empty interface
type eface struct {
	_type *_type
	data  unsafe.Pointer
}

// normal interface
type iface struct {
	tab  *itab
	data unsafe.Pointer
}

type itab struct {
	inter *interfacetype
	_type *_type // eface中的_type也是*_type类型
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
我们简单的做一下总结：
* 1、`eface`表示空接口类型，`iface`表示非空接口类型。
* 2、`eface`、`iface`均由两个指针字段表示。
* 3、`eface`、`iface`内部实现相同的部分是`data`字段，表示一个指针，均指向**当前赋值给接口类型变量的动态类型变量的值**。

这句话虽然比较绕，但是很好理解。比如`var i interface{} = 1`，`data`作为一个指针，指向了存储`1`这个被赋值给接口变量的值。

* 4、`eface`、`iface`的第一个指针字段都是表示动态类型的不同。但是字段类型并不同，具体来说`iface`的结构更复杂。

对于`eface`来说，是一个超集的存在。为何？

因为`eface`没有方法，所以`_type`字段不需要存储接口的方法集合、对应方法的具体实现。

下面让我们仔细观察一下非空接口变量的内部实现`iface`结构：
```go
type itab struct {
	inter *interfacetype 
	_type *_type // eface中的_type也是*_type类型，存储动态类型信息
	hash  uint32 // copy of _type.hash. Used for type switches.
	_     [4]byte
	fun   [1]uintptr // variable sized. fun[0]==0 means _type does not implement inter.
}
```
* `inter *interfacetype` ：存储接口本身的信息，包括类型、包路径、方法集合切片。
* `_type *_type` ：`eface`中的`_type`也是`*_type`类型，存储动态类型信息。
* `fun`：被赋值的动态类型已实现的接口方法的调用地址数组。

下面让我们通过具体的代码来展现一下上述的描述。

不过在此之前，我们需要引用一下`println`函数。为何？

因为`eface`和`iface`是`runtime`包中的非导出(均为小写嘛)结构体定义，我们不能直接在包外使用。

`println`会在编译阶段根据要输出的参数类型转换为特定的函数，我们来看下`Go`转为`eface`和`iface`准备的函数
```go
// runtime/print.go
func printeface(e eface) {
	print("(", e._type, ",", e.data, ")")
}

func printiface(i iface) {
	print("(", i.tab, ",", i.data, ")")
}
```

其实就是直接打印`eface`、`iface`两个结构体中的两个字段嘛（而且还是地址值哈）。

下面真的让我们通过代码来展示上面关于`interface`的内部实现吧

```go
type printer interface {
	Print()
}

type Screen struct {
	s string
	n int
}

func (s Screen) Print() {
	fmt.Println("hello. Screen is open.")
}

func main() {
	var (
		ep interface{} = Screen{s: "empty", n: 1}  // ep空接口变量
		ip printer     = Screen{s: "normal", n: 2} // ip非空接口变量
	)
	println(ep) // (0x10b9ec0,0xc000098f70)
	// 0x10b9ec0指向Screen这个类型信息(_type)
	// 0xc000098f70指向Screen值信息(s: "empty", n: 1)

	println(ip) // (0x10ec9e0,0xc000098f68)
	// 0x10ec9e0指向Screen这个类型信息(_tab)
	// 0xc000098f68指向Screen值信息(s: "normal", n: 2)
}
```

下面给出图解：

![interface变量的内部表示-示例](https://us3tmp.laiye.com/interface_variable_internal_display_1670151144_4576.png)


好了，我们讨论完了`struct`内嵌`interface`以及`interface`变量的内部版表示，现在让我们来回答之前的运行时错误的案例。
```go
type MyStruct struct{}

// 内嵌error interface
type MyError struct {
	error
}

func main() {
	var e error = &MyError{}  // ok

	fmt.Println(e.Error()) // panic: runtime error: invalid memory address or nil pointer dereference
}
```
之所以报运行时空指针`panic`，是因为`MyError`虽然静态编译通过，但是并没有实现`error`接口需要的`Error() string`方法啊。

下面我们将继续讨论，因`interface`内部的具体实现，而容易导致`bug`的情形和案例。

### 🚩函数直接返回 interface

这样干，可能会出现违反直觉且难以排查的`bug`。

让我们直接来看下面的这个例子：

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
		fmt.Printf("error: %+v\n", e)
		return
	}
	fmt.Println("ok")
}
```
注意力集中在`returnsError`函数的实现上，初始化`p`为`nil`，返回`p`，所以我们期待返回一个`nil`。

当我们运行`main`的时候，我们会惊奇的发现，其输出结果为

`error: <nil>`

难道`returnsError`函数内部的变量`p`不是`nil`吗？我们来验证一下，修改代码如下：
```go
func returnsError() error {
	var p *MyErr = nil // 📢
	if bad() {
		p = &ErrBad
	}

    // 函数返回前判断p到底是否为nil
	if p == nil {
		fmt.Println("in func, p is nil")
	}

	return p
}
```

运行代码，打印如下：
```text
in func, p is nil
error: <nil>
```

函数内部打印为`nil`，函数外部继续叫嚣着我不是`nil`。

这究竟是什么原因？

根本原因在于，我命名为：`Go`的值类型返回。

我来解释一下：

当我们在代码中，`return p`的时候，实际上什么情况？

```go
return p
// 相当于

var ret error 
ret = p
return  
```
你说说看，此时的`ret`能是`nil`吗？

不是的。

体现了`interface`的动态性：如同在`interface`变量的内部表示那一节中所说，其动态性包含两部分：动态类型、动态类型的值。

动态类型的值虽然为`nil`，但是动态类型并不是为`nil`，而是`*MyErr`类型啊！

如何验证我们的想法？
```go
func main() {
	e := returnsError()
	
	// 类型验证
	_, ok := e.(*MyErr)
	fmt.Println("e is MyErr 类型", ok)
	
	if e != nil {
		fmt.Printf("error: %+v\n", e)
		return
	}
	fmt.Println("ok")
}
```

输出结果如下：
```text
in func, p is nil
e is MyErr 类型 true
error: <nil>
```

### 🚩使用空接口作为函数的参数类型

最好不要这样干

> 空接口不提供任何信息。   —— Rob Pike, Go语言之父


### 🌵interface 命名的惯例
通常来说，如果`interface`只有一个`method`，那么该`interface`的名字：`method`的名称`+er`，比如`Reader`、`Writer` 等等

> By convention, one-method interfaces are named by the method name plus an -er suffix or




### 🌵尽量定义小接口

> 接口越大，抽象程度越低。（一个人的能力越大，他的责任也就越大）


**中心思想：接口中的方法越多，这个接口越不通用**

> The bigger the interface, the weaker the abstraction.

但是我们不得不说，在项目初期，很难分辨和抽象出实用的小接口。也就是说初期可不在意接口的大小，因为对问题的理解是循序渐进的，毕竟标准库的`io.Reader/io.Writer`也不是一开始就确定的。

### 🚩使用接口作为代码水平组合的连接点

## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* `[The Go Programming Language](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)