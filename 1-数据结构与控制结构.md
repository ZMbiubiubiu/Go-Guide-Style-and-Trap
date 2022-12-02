
* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。
  
  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。
  
  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

## slice
### 🌵1、nil slice可以直接append

为什么单独拿出这一点？

是为了和`map`作比较。`nil map`是不能直接进行写操作的。
### 🌵2、合理的初始化
* 字面值初始化
```go
arr := []int{1,2,3}
```

* `make`初始化：已知元素个数
```go
arr := make([]int, 0, 20) // then append
```

* `nil`初始化：不知最终的元素个数
```go
var arr []int
```
项目中的代码示例
```go
// bad 
userIds := []uint32{}
sql := "..."
_, err := libs.GetMysqlDb(m.GetDbName()).Raw(sql, activityId).QueryRows(&userIds)

// good 
var userIds []uint32 
```

对于`nil slice`来说，可以直接`append`。

而`[]uint32{}`，相当于`make([]uint32, 0)`

### 🚩3、append时，使用原来的切片变量进行接收
* 承接上文，若已知`append`元素的个数，初始化设定底层数组的大小。

* 用原变量接收`append`之后的结果

```go
arr = append(arr, 3)
```

如果不这么做，很可能就会买入一个坑的世界。

### 🚩4、正确使用copy复制函数
<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
var dst []int
src := []int{1, 2, 3}

copy(dst, src)
fmt.Println(dst) // []
```
</td><td>

```go
src := []int{1, 2, 3}
var dst = make([]int, len(src))

copy(dst, src)
fmt.Println(dst) // [1 2 3]
```
</td></tr>
</tbody></table>

使用`copy`时，一定要注意：

* 复制的元素个数等于`src`、`dst`两个**切片长度的最小值**。

* 注意`copy`参数的顺序，目标切片在前，源切片在后。

### 🚩5、避免内存泄漏

> Golang will not leave unused slices

对于一个元素很多的切片，若我们只需要其很小的部分数据。那么我们应该采用`copy`的方式，而不是`slicing`的方式获取小部分数据。

<table>
<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func printAlloc() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d MB\n", m.Alloc/(1024*1024))
}

// slicing , bad
func getHead() []byte {
	var msg [100000000]byte
	printAlloc()
	return msg[:5]
}

func main() {
	printAlloc()       // 0 MB
	var head []byte
	head = getHead()   // 95 MB
	runtime.GC()
	printAlloc()       // 95 MB
	_ = head 
}
```
</td><td>

```go
// copy, good 
func getCopyHead() []byte {
	var msg [100000000]byte
	printAlloc()
	head := make([]byte, 5)
	copy(head, msg[:5])
	return head
}

func main() {
	printAlloc()           // 0 MB
	
	var head []byte
	head = getCopyHead()   // 95 MB
	
	runtime.GC()
	printAlloc()           // 0 MB
	_ = head
}
```
</td></tr>
</tbody></table>

### 🚩6、从切片中新建另一个切片的方式
大家想到的第一个方式可能就是`slicing`的方式，包括我也是。
```go
s2 := s1[:] // 从s1中新建切片s2
```
不过话说回来，这有什么问题吗？

`talk is cheap, show me the code.`
```go
a := []int{1, 2, 3}
b := a[:]
b[1] = 666

fmt.Println("a", a) // [1 666 3]
fmt.Println("b", b) // [1 666 3]
```

**问题在于两者有扯不清的关系**(当然某种意义上，两者还是能够扯清关系)

那有什么好的解决方式吗？有，而且有两种。

* `copy`的方式
```go
src := []int{1, 2, 3}
var dst = make([]int, len(src))
copy(dst, src)
```
思路虽然简单但是实现比较复杂：**居然需要三行代码**。

* `append`的方式

```go
// encoding/json/encode.go:165
buf := append([]byte(nil), e.Bytes()...)
```

相比上述`copy`的方式，这个更加的简单。

颇有`python`的风格，一行代码搞定。

## map
### 🌵1、合理的初始化

和切片一样，当我们初始化的时候，应尽可能设置容量大小。

采用`make`的方式，如果提前知道存入的键值对的数量，在`make`中指定
```go
const mapSize = 1000

func BenchmarkInitMapCap(b *testing.B) {
	b.Run("without cap", func(b *testing.B) {
		for i:=0;i<b.N;i++ {
		   // 未指定大小
			m := make(map[int]int)
			for k:=0;k<mapSize;k++ {
				m[k]=k
			}
		}
	})

	b.Run("with cap", func(b *testing.B) {
		for i:=0;i<b.N;i++ {
		   // 指定大小
			m := make(map[int]int, mapSize)
			for k:=0;k<mapSize;k++ {
				m[k]=k
			}
		}
	})
}

// 
🐂🍺  go test -benchmem -bench=. map_test.go
goos: darwin
goarch: amd64
cpu: Intel(R) Core(TM) i5-7267U CPU @ 3.10GHz
BenchmarkInitMapCap/without_cap-4                  13339             79271 ns/op           86552 B/op         64 allocs/op
BenchmarkInitMapCap/with_cap-4                     36530             31913 ns/op           41097 B/op          6 allocs/op
PASS
ok      command-line-arguments  3.703s

```

### 🌵2、实现某种类型的Set
```go
// struct{} or bool ？
var set = make(map[int]bool)  
```

* 使用`struct{}`的优缺点

优点：`struct{}`其本身不占用内存

缺点：赋值的时候比较怪异，`m[1] = struct{}{}`

* 使用`bool`的优缺点

优点：赋值的时候比较容易理解

缺点：值为`true`我们可以理解，但是如果值为`false`是个什么意思？

使用`bool`类型

### 🚩3、不要对map的遍历循序做任何假设
这一点大家都知道，列出来表示结构完整。

### 🚩4、不要在循环中的新增、删除，结果不确定
```go
// bad 
m := map[int]bool{
	1: false,
	2: true,
	3: true,
	4: false,
	5: true,
}

func main() {
	for k, v := range m {
		if v {
			m[k+10] = true
		}
	}
	fmt.Printf("map:%+v\n", m)
	// map:map[1:false 2:true 3:true 4:false 5:true 12:true 13:true 15:true 22:true 23:true 25:true]
	// map:map[1:false 2:true 3:true 4:false 5:true 12:true 13:true 15:true]
	// map:map[1:false 2:true 3:true 4:false 5:true 12:true 13:true 15:true 23:true 25:true 33:true]
}
```

### 🚩5、避免内存泄漏
一个`map`如何优雅的删除大量的`key`？

`map`的重建如同`redis`的`AOF`日志重写一样，不能简单的`del`。

先让我们谈谈`Redis`的`AOF`日志重写。

`AOF`是写后日志，运行完增删改的`Redis`命令之后，记录对应的日志。随着程序的运行，`AOF`日志会变得很大，尤其是类似下面这种情况。
```redis
set rediskey1 1
set rediskey1 4
set rediskey1 5
....
set rediskey1 1
```
而实际上`rediskey1`只需要一条记录即可，这种对`AOF`日志的大减肥，就叫做`AOF`日志重写。

`map`亦如是，当从大`map`变成小`map`，我们也需要重写。

我们首先看一下`Go map runtime`的形态
```go
// A header for a Go map.
type hmap struct {
    // Note: the format of the hmap is also encoded in cmd/compile/internal/gc/reflect.go.
    // Make sure this stays in sync with the compiler's definition.
    count     int // # live cells == size of map.  Must be first (used by len() builtin)
    flags     uint8
    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
    hash0     uint32 // hash seed

    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)

    extra *mapextra // optional fields
}
```
具体的键值对存储在`buckets`中，随着键值对的增多(`grows`)，`buckets`的数量会随着增多。

但问题在于，**随着键值对的减少(`shrinks`)，`buckets`数量不会随之减少，不会释放资源。**

下面让我们看一下示例
```go
func printAlloc() {
	var m runtime.MemStats
	runtime.ReadMemStats(&m)
	fmt.Printf("%d MB\n", m.Alloc/(1024*1024))
}


func main() {
	var n = 100_0000
	var m = make(map[int][128]byte) // init a zero length map

	for i := 0; i < n; i++ { // allocate 1 million k/v pairs
		m[i] = [128]byte{'h', 'e', 'l', 'l', 'o'}
	}

	printAlloc() // 461 MB

	for i := 0; i < n; i++ { // delete all map k/v pairs
		delete(m, i)
	}
	runtime.GC()
	printAlloc() // 293 MB
	runtime.KeepAlive(m)
}
```

> The reason is that the number of buckets in a map cannot shrink.

**那么如何解决map的内存泄漏呢？**

很简单，重启大法好

**重启的真的管用!**

另外就是定期重建`map`，重建`map`的时候，就会遇到`AOF`日志重写一模一样的问题。

### 🚩6、不支持并发读写

> 没那个能力，知道不。
```go
func main() {
	m := make(map[string]int)

	go func() {
		for {
		   // write
			m["a"] = 1
			time.Sleep(time.Microsecond)
		}
	}()

	go func() {
		for {
		   // read
			_ = m["b"]
			time.Sleep(time.Microsecond)
		}
	}()

	select {}
}
// fatal error: concurrent map read and map write
```

### 🚩7、键值对的值不支持取地址操作
因为随着键值对的增多，会增加`bucket`以及对应的调整，导致的结果就是位置会发生变更。
```go
type data struct {
	s string
}

func main() {
	m := map[string]data{
		"one": {"first"},
	}

	_ := &m["one"] // Cannot take the address of 'm["one"]'
}
```
### 🚩8、值为struct，不支持修改

```go
m := map[string]data{ // not ok
//m := map[string]*data{ // ok
	"one": {"first"},
}
m["one"].name = "one"

tmp := m["one"]
tmp.name = "one"
```
## string
### 🚩1、string是不可变类型
```go
s := "hello world"
s[0] = 56 // Cannot assign to s[0]
```

### 🚩2、明确字面量、Unicode码点(rune)、UTF-8编码之间的关系
* `Go`字符串的每个字符都是`Unicode`字符.
* `Go`的`Unicode`字符默认是`UTF-8`编码格式存储在内存当中的
```go
func main() {
	// 中文字符  Unicode CodePoint(码点rune)   UTF8编码
	//  中		    U+4E2D			       E4B8AD
	//  国		    U+56FD			       E59BBD
	//  欢		    U+6B22			       E6ACA2
	//  迎		    U+8FCE			       E8BF8E
	//  您		    U+60A8			       E682A8
	s := "中国欢迎您"
	rs := []rune(s)
	sl := []byte(s)
	for i, v := range rs {
		var utf8Bytes []byte
		// 因为上述中文在utf-8编码中，均为3字节长度
		for j := i * 3; j < (i+1)*3; j++ {
			utf8Bytes = append(utf8Bytes, sl[j])
		}
		fmt.Printf("%s => %X => %X\n", string(v), v, utf8Bytes)
	}
}
// output 
中 => 4E2D => E4B8AD
国 => 56FD => E59BBD
欢 => 6B22 => E6ACA2
迎 => 8FCE => E8BF8E
您 => 60A8 => E682A8
```

### 🌵3、恰当的字符串拼接
一些构造和拼接字符串的方式
```text
1、+=/+
2、fmt.Sprintf
3、strings.Join // 本质上是调用了strings.Builder
4、strings.Builder
5、bytes.Buffer
```

进行基准测试
```go
package main

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

var sl []string = []string{
	"Rob Pike ",
	"Robert Griesemer ",
	"Ken Thompson ",
}

func concatStringByOperator(sl []string) string {
	var s string
	for _, v := range sl {
		s += v
	}
	return s
}

func concatStringBySprintf(sl []string) string {
	var s string
	for _, v := range sl {
		s = fmt.Sprintf("%s%s", s, v)
	}
	return s
}

func concatStringByJoin(sl []string) string {
	return strings.Join(sl, "")
}

// 不指定builder的初始大小
func concatStringByStringsBuilder(sl []string) string {
	var b strings.Builder
	for _, v := range sl {
		b.WriteString(v)
	}
	return b.String()
}

func concatStringByStringsBuilderWithInitSize(sl []string) string {
	var b strings.Builder
	b.Grow(64)
	for _, v := range sl {
		b.WriteString(v)
	}
	return b.String()
}

func concatStringByBytesBuffer(sl []string) string {
	var b bytes.Buffer
	for _, v := range sl {
		b.WriteString(v)
	}
	return b.String()
}

func concatStringByBytesBufferWithInitSize(sl []string) string {
	buf := make([]byte, 0, 64)
	b := bytes.NewBuffer(buf)
	for _, v := range sl {
		b.WriteString(v)
	}
	return b.String()
}

func BenchmarkConcatStringByOperator(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByOperator(sl)
	}
}

func BenchmarkConcatStringBySprintf(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringBySprintf(sl)
	}
}

func BenchmarkConcatStringByJoin(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByJoin(sl)
	}
}

func BenchmarkConcatStringByStringsBuilder(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByStringsBuilder(sl)
	}
}

func BenchmarkConcatStringByStringsBuilderWithInitSize(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByStringsBuilderWithInitSize(sl)
	}
}

func BenchmarkConcatStringByBytesBuffer(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByBytesBuffer(sl)
	}
}

func BenchmarkConcatStringByBytesBufferWithInitSize(b *testing.B) {
	for n := 0; n < b.N; n++ {
		concatStringByBytesBufferWithInitSize(sl)
	}
}
// output 
🐂🍺 go test -benchmem -bench=. string_concat_benchmark_test.go 
goos: darwin
goarch: amd64
cpu: Intel(R) Core(TM) i5-7267U CPU @ 3.10GHz
BenchmarkConcatStringByOperator-4                       10869480               100.1 ns/op            80 B/op          2 allocs/op
BenchmarkConcatStringBySprintf-4                         2503515               478.0 ns/op           176 B/op          8 allocs/op
BenchmarkConcatStringByJoin-4                           21124852                57.97 ns/op           48 B/op          1 allocs/op
BenchmarkConcatStringByStringsBuilder-4                  9492667               124.0 ns/op           112 B/op          3 allocs/op
BenchmarkConcatStringByStringsBuilderWithInitSize-4     22255063                53.61 ns/op           64 B/op          1 allocs/op
BenchmarkConcatStringByBytesBuffer-4                    12520616                99.99 ns/op          112 B/op          2 allocs/op
BenchmarkConcatStringByBytesBufferWithInitSize-4        15130156                86.36 ns/op           48 B/op          1 allocs/op

```
* 如果拼接比较简单，直接使用 `+` 即可

* 如果拼接的过程比较复杂，建议使用`strings.Builder`，并最好提前设置大小

### 🚩4、区分 string/[]string/byte/[]byte/rune/[]rune
`string -> []rune`

`string -> []byte`

反过来也成转换

`[]rune -> string`
`[]byte -> string`

* 简单的转换需要重新分配内存的代价
```go
// []byte -> string 
func byteSliceToString() {
	sl := []byte{
		0xE4, 0xB8, 0xAD,
		0xE5, 0x9B, 0xBD,
		0xE6, 0xAC, 0xA2,
		0xE8, 0xBF, 0x8E,
		0xE6, 0x82, 0xA8,
		0xEF, 0xBC, 0x8C,
		0xE5, 0x8C, 0x97,
		0xE4, 0xBA, 0xAC,
		0xE6, 0xAC, 0xA2,
		0xE8, 0xBF, 0x8E,
		0xE6, 0x82, 0xA8,
	}

	_ = string(sl)
}

// string -> []byte 
func stringToByteSlice() {
	s := "中国欢迎您，北京换欢您"
	_ = []byte(s)
}

func main() {
	fmt.Println(testing.AllocsPerRun(1, byteSliceToString)) // 1
	fmt.Println(testing.AllocsPerRun(1, stringToByteSlice)) // 1
}
```
输出`1`，表示每个转换过程都需要`1`次内存分配操作。

下面来看进行转换时，不需要进行内存分配的情况。

### 🌵5、编译器的优化之`[]byte->string`
要义在于`[]byte`是可变。

如果转换成`string`且没有内存分配，对此时`string`的使用场景是由严格要求的
* `map`中`key`
* 字符串比较
* 字符串拼接
```go
var (
	s  = "world"
	bs = []byte("hello")
	m  = make(map[string]int, 1)
)

func main() {
	//
	m["hello"] = 1

	fmt.Println(testing.AllocsPerRun(1, func() {
		_ = m[string(bs)]
	})) // 0

	fmt.Println(testing.AllocsPerRun(1, func() {
		if "world" > string(bs) {
		}
	})) // 0

	fmt.Println(testing.AllocsPerRun(1, func() {
		s = string(bs) + s
	})) // 1

	fmt.Println(testing.AllocsPerRun(1, func() {
		_ = string(bs) + ""
	})) // 0
}
```

总结一下就是：`[]byte->string`，转换之后的`string`是临时的，`string`只是一个步骤的中间过程，一闪而过。

### 🌵6、编译器优化之`string->[]byte`
主要就是`for`循环中
```go
// 其中s是上述代码所示的 "world"
fmt.Println(testing.AllocsPerRun(1, func() {
	for range []byte(s) {
	}
})) // 0
```
### 🚩7、理解子字符串
* 什么是子字符串(`substring`)？
```go
s := "hello world"
sub := s[:5]
```

`sub`就是所谓的子字符串

* `substring`的切分单位
  以字节为单位，数值代表字节的大小，而不是`Unicode`字符
```go
s := "中国"
s := "中国"
sub := s[:3]

fmt.Println(sub) // 中
```

如过我们想要截取固定个`Unicode`呢？

`substring`是不行了，因为它只认字节。不过解决的办法也很简单，我们就利用切片，不过首先转换为`[]rune`。

```go
s := "我是中国人，我说中国话"

sub := []rune(s)[:5]

fmt.Printf("sub:%T\t%s", sub, string(sub))
```

* `substring`不会新分配内存
```go
s := "this is my car, that is your car. ok ? fool you"

fmt.Println(testing.AllocsPerRun(1, func() {
	sub := s[:3]
	_ = sub
})) // 0 表示进行了0次内存分配
```
类似切片的`slicing`
```go
arr := []int{1, 2, 3}
arr2 := arr[:2]

fmt.Printf("s:%p\tsub:%p\n", arr, arr2)
// s:0xc0000b8020  sub:0xc0000b8020
```
`substring`不重新分配内存的理由甚至强于切片的`slicing`。何出此言？

**因为`substring`和`string`都是不可变量！都是只读的，当然可以引用同一份底层数据！**

不过正因为`substring`不会分配新的内存，所以和切片的`slicing`一样，可能一不小心，就会导致内存泄漏。

欲知后事如何，请看下文分解。
### 🚩8、避免内存泄漏
考虑一个日志系统，日志开头是固定的`36`字节长度的`UUID`。假如我们需要存储最近的`n`个`UUID`。
```go
func (s store) handleLog(log string) error {
	if len(log) < 36 {
		return errors.New("log is not correctly formatted")
	}
	uuid := log[:36]
	s.store(uuid) // Do something 
}
```
那么，上述的实现有什么问题？

如何修正？

```go
func (s store) handleLog(log string) error {
	if len(log) < 36 {
		return errors.New("log is not correctly formatted")
	}
	uuid := string([]byte(log[:36])) // Goland 提示：Redundant type conversion
	s.store(uuid) // Do something 
}
```

`uuid := string([]byte(log[:36]))`不得不说，这个操作好骚啊。

利用`[]byte(log[:36])`分配新的内存，存储`36`个字节，作为字节数组

然后再将上述的字节数组转换为字符串

整体上就是字符串转换为字符串

骚，实在是骚！
