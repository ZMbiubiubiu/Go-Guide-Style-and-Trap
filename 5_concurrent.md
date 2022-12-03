本节主要讨论`Go`的并发编程

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

# lock锁

## 🚩零值的锁结构都是开箱即用的
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

## 🚩当心Mutex的错误复制
todo
[Beware of copying mutexes in Go](https://eli.thegreenplace.net/2018/beware-of-copying-mutexes-in-go/)

# goroutine

## 关于goroutine应该知道的知识
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
## 关于特殊状态的channel，需要注意以下事实


|       | closed channel | nil channel |
|-------|----------------|-------------|
| ch <- | panic          | 阻塞          |
| <- ch | channel 中元素的零值 | 阻塞          |


## 🚩channel的大小要么是1，要么是0
TODO


# 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Beware of copying mutexes in Go](https://eli.thegreenplace.net/2018/beware-of-copying-mutexes-in-go/)