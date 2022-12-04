本节主要讨论`Go`的并发编程

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。
  
- [lock锁](#lock锁)
  - [🚩零值的锁开箱即用](#🚩零值的锁开箱即用)
  - [🚩Mutex 的错误复制](#🚩mutex的错误复制)
- [goroutine](#goroutine)
  - [🚩关于 goroutine 应该知道的知识](#🚩关于goroutine应该知道的知识)
  - [🚩Before you launch a goroutine, know when it will stop](#🚩before-you-launch-a-goroutine-know-when-it-will-stop)
- [channel](#channel)
  - [🌵To pass a signal prefer chan struct{} instead of chan bool.](#🌵to-pass-a-signal-prefer-chan-struct-instead-of-chan-bool)
  - [🚩channel 特殊状态的危险操作](#🚩channel特殊状态的危险操作)
  - [🚩channel 的大小要么是 1 ，要么是 0](#🚩channel的大小要么是-1，要么是-0)
- [参考](#参考)

# lock锁

## 🚩零值的锁开箱即用
**zero-valus of lock is valid**
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
mu := new(sync.Mutex)
mu.Lock()
```
</td><td>

```go
var mu sync.Mutex
mu.Lock()
```
</td></tr>
</tbody></table>

虽然`Lock`、`Unlock`是接受者为`pointer`的方法，但是仍然可以直接用`var mu = sync.Mutex`
```go
func (m *Mutex) Lock()
func (m *Mutex) Unlock() 
```
理由见`method`章节的`value receiver & pointer receiver`

## 🚩Mutex 的错误复制

编写此节的灵感来源一篇技术博客，名为『`Beware of copying mutexes in Go`』，链接🔗见参考部分。

实现了一个支持并发读写的`map`。
```go
package main

import (
  "fmt"
  "sync"
  "time"
)

type Container struct {
  sync.Mutex                       // 通过内嵌的方式加入锁
  counters map[string]int
}

func (c Container) inc(name string) {
  c.Lock()                         // 加锁
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

直接调用上述程序是会报错的：

`fatal error: concurrent map writes`

根据报错的提示，`map`依然发生了并发读写，我们不是已经加了锁保护吗？

`ps`：但是只需要修改一个字符，就能完美运行。

根本原因在于，`Container`上`inc`方法为`value method`。每次调用`inc`都会复制一个`Container`，包括锁，相当于每次调用`inc`都会生成一个新的锁，这就不能保护`map`在并发读写下的安全性。

> The problem with the code is that whenever inc is called, our container c is copied into it, because inc is defined on Container, not *Container

锁是值类型，不能复制使用。
```go
// A Mutex is a mutual exclusion lock.
// The zero value for a Mutex is an unlocked mutex.
//
// A Mutex must not be copied after first use.
type Mutex struct {
	state int32
	sema  uint32
}
```

而实际上，当你这么写的时候，`Goland`编辑器已经发现了问题：

`'inc' passes a lock by the value: type 'Container' is 'sync.Locker' `

我们的改动也非常简单，将`inc`从一个`value method`改变为`pointer method`即可。
```go
func (c *Container) inc(name string) {
	c.Lock() // <-- Added locking of the mutex
	defer c.Unlock()
	c.counters[name]++
}
```

这段代码真的是一个非常棒的示例：
* 清楚的揭示了`value receiver`和`pointer receiver`之间的区别。
* 如果方法需要修改类型状态，就一定要定义成`pointer method`。
* 锁是值类型，不要复制它。

# goroutine

## 🚩关于 goroutine 应该知道的知识
* 只要`main goroutine`退出，不管其余`children goroutine`是否运行完毕，直接退出
* 如果某个`goroutine`在函数/方法的调用时出现`panic`，一个被称之为`panicking`的过程被激活，一直向上，运行上层每一层的`defer`（如果有的话），如果没有使用`recover`捕获该`panic`，那么整个程序会异常退出。


## 🚩Before you launch a goroutine, know when it will stop


# channel
## 🌵To pass a signal prefer chan struct{} instead of chan bool.
```go
type Service struct {
	deleteCh chan bool // what does this bool mean? 
}

type Service struct {
	deleteCh chan struct{} // ok, if event than delete something.
}
```
## 🚩channel 特殊状态的危险操作

|       | closed channel | nil channel |
|-------|----------------|-------------|
| ch <- | panic          | 阻塞          |
| <- ch | channel 中元素的零值 | 阻塞          |


## 🚩channel 的大小要么是 1 ，要么是 0
TODO


# 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Beware of copying mutexes in Go](https://eli.thegreenplace.net/2018/beware-of-copying-mutexes-in-go/)