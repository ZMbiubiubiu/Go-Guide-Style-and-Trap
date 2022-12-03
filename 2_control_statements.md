本节主要讨论`Go`的控制语句，比如`for`、`select`、`switch`等。

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。
  
- [for](#for)
  - [🚩1、迭代变量只会初始化一次](#🚩1、迭代变量只会初始化一次)
  - [🈲2、goroutine 对迭代变量的引用（闭包）](#🈲2、goroutine对迭代变量的引用（闭包）)
  - [🈲3、不要在循环中使用 defer](#🈲3、不要在循环中使用-defer)
  - [🚩4、range 后接的表达式只会被求值一次](#🚩4、range后接的表达式只会被求值一次)
- [select](#select)
  - [🈲1、不要假定和依赖 select 的执行顺序](#🈲1、不要假定和依赖-select的执行顺序)
- [其他](#其他)
  - [🚩1、必须处理类型断言](#🚩1、必须处理类型断言)
  - [🚩2、嵌套不要太深，最多2层](#🚩2、嵌套不要太深，最多2层)
  - [🚩3、注意 break 的使用](#🚩3、注意-break的使用)
- [参考](#参考)



## for 
### 🚩1、迭代变量只会初始化一次
**只不过会不断的赋值**
```go
func main() {
	arr := []string{"one", "two", "three"}
	var wg sync.WaitGroup

	for _, v := range arr {
		wg.Add(1)
		go func() {
			defer wg.Done()
			fmt.Println(v) // 闭包
		}()
	}
	wg.Wait()
}
// three
// three 
// three 
```

不要被代码中的`v:= range arr`中的`:=`给迷惑住，`v`只会初始化一次！

下面是另一个例子。输出是什么？如果没有按照预想的结果输出，如何修改？
```go
package main

import (
	"fmt"
)

type Custom struct {
	Id   uint32 `json:"id"`
	Name string `json:"name"`
}

func main() {
	customs := []Custom{
		{1, "one"},
		{2, "two"},
		{3, "three"},
	}
	var m = make(map[int]*Custom, len(customs))

	for i, cm := range customs {
		m[i] = &cm
		fmt.Printf("%p\n", &cm)
	}
	for k, v := range m {
		fmt.Printf("key:%d, val:%+v\n", k, v)
	}
}

```
### 🈲2、goroutine 对迭代变量的引用（闭包）

```go
// 打印的val值是不确定的
for _, val := range values {
	go func() {
		fmt.Println(val)
	}()
}

// 打印的val值是不确定的
for _, val := range values {
	go func(val int) {
		fmt.Println(val)
	}(val)
}
```

### 🈲3、不要在循环中使用 defer
> Defers will grow your stack 

下面的问题是什么？
```go
fileNames := []string{}

for _, file := range {
    f, err := read(file)
    defer f.Close()
}
```

### 🚩4、range 后接的表达式只会被求值一次
比如 `for i:v := range sliceInt`

`sliceInt`就是一个表达式。当然表达式可以为字符串、切片、数组、`channel`、`map`等，但不论是什么表达式，都只会被求值一次，且将求值后的表达式复制给临时变量。

比如下面这个`for range`会无限循环吗？
```go
s := []int{0, 1, 2}
for range s {
    s = append(s, 10)
}
```

实际的执行
```go
s := []int{0, 1, 2}
copyS := s // 复制
for i := 0; i < len(copyS); i++ {
	s = append(s, 10)
}

fmt.Println(s)
fmt.Println(copyS)
```

> the range loop evaluates the provided expression only once, before the beginning of the loop, by doing a copy (regardless of the type).

* * *
为了比较学习，下面这个原始`for`循环呢？会无限循环吗？
```go
s := []int{0, 1, 2}
for i:=0;i<len(s); i++ {
    s = append(s, 10)
}
```

> the len(s) expression is evaluated during each iteration.

## select 
### 🈲1、不要假定和依赖 select 的执行顺序
```go
select {
case <-ch1:
	// balabala
case <-ch2:
	// balabala
case <-ch3:
	// balabala
case <-ch4:
	// balabala
case <-ch5:
	// balabala
}
```
`go`的`select`，并不是依次判断`ch1/ch2/ch3/ch4/ch5...`，而是随机的。

为什么是随机的呢？
因为怕饿死。

## 其他

### 🚩1、必须处理类型断言
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
t := i.(string)
```

</td><td>

```go
t, ok := i.(string)
if !ok {
  // handle the error gracefully
}
```

</td></tr>
</tbody></table>

### 🚩2、嵌套不要太深，最多2层
如果大于上面规定的层数，说明你应该写个函数了。

### 🚩3、注意 break 的使用
当遇到`for`、`switch`、`select`时，要清楚使用`break`产生的影响。

**普通的`break`只会跳出最里层的`for`、`select`、`switch`结构**

当你有疑惑的时候，一定是你嵌套太多层了 ！

减少嵌套，生活更美好。


## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)