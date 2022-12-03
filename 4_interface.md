本节主要讨论`Go`的`interface`，分为`empty interface`和`normal interface`。

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

- [empty interface 空接口](#empty-interface空接口)
  - [🚩尽量避免使用空接口作为函数的参数类型](#🚩尽量避免使用空接口作为函数的参数类型)
- [normal interface 非空接口](#normal-interface非空接口)
  - [🌵name](#🌵name)
  - [🌵显式验证：某个类型实现指定的接口](#🌵显式验证：某个类型实现指定的接口)
  - [🚩内嵌 interface](#🚩内嵌interface)
  - [🚩当心直接返回 interface 的情况](#🚩当心直接返回interface的情况)
  - [🚩尽量定义小接口](#🚩尽量定义小接口)
- [参考](#参考)


## empty interface 空接口

我觉得在Go中，空接口与非空接口完全表现为两种不同的事物，所以我打算分开来讨论。

### 🚩尽量避免使用空接口作为函数的参数类型

> 空接口不提供任何信息。   —— Rob Pike, Go语言之父

## normal interface 非空接口

**中心思想：接口中的方法越多，这个接口越不通用**

> The bigger the interface, the weaker the abstraction.

### 🌵name
通常来说，如果`interface`只有一个`method`，那么该`interface`的名字：`method`的名称`+er`，比如`Reader`、`Writer` 等等

> By convention, one-method interfaces are named by the method name plus an -er suffix or

### 🌵显式验证：某个类型实现指定的接口
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

关键在于第二种，`struct`内嵌`interface`，这是什么组合？这有什么道理？
todo:[文章阅读](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs)/


### 🚩当心直接返回 interface 的情况
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

### 🚩尽量定义小接口

> 接口越大，抽象程度越低。（一个人的能力越大，他的责任也就越大）

但是我们不得不说，在项目初期，很难分辨和抽象出实用的小接口。也就是说初期可不在意接口的大小，因为对问题的理解是循序渐进的，毕竟标准库的`io.Reader/io.Writer`也不是一开始就确定的。

## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)