本节主要讨论`Go`的函数以及方法。

* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。

- [前言](#前言)
- [🌵function定义的规范行为](#🌵function定义的规范行为)
  - [若参数有 context.Context 类型，必做第一个参数](#若参数有context-context类型，必做第一个参数)
  - [若返回 error，必将最后一个返回值](#若返回error，必将最后一个返回值)
  - [同一类型的参数可以合并](#同一类型的参数可以合并)
  - [函数名中禁止出现 filter](#函数名中禁止出现filter)
- [🚩具名返回值：谨慎使用](#🚩具名返回值：谨慎使用)
  - [明确具名返回值不是函数签名的一部分](#明确具名返回值不是函数签名的一部分)
  - [存在的目的，为了表达清晰](#存在的目的，为了表达清晰)
  - [函数返回方式的风格要统一](#函数返回方式的风格要统一)
- [🚩init 函数：能不用就不用](#🚩init函数：能不用就不用)
  - [init 函数的特点](#init函数的特点)
  - [init 函数的缺点](#init函数的缺点)
  - [init 万一使用，需要满足](#init万一使用，需要满足)
  - [使用 init 函数的经典案例](#使用init函数的经典案例)
- [🚩defer 函数：合理使用](#🚩defer函数：合理使用)
  - [defer 的定义](#defer的定义)
  - [为什么要有 defer](#为什么要有defer)
  - [defer 的特性](#defer的特性)
  - [如何记住 defer 的特性](#如何记住defer的特性)
  - [不要在循环中滥用 defer](#不要在循环中滥用defer)
- [🚩只在 main 函数中退出](#🚩只在main函数中退出)
- [🌵尽量不要使用 new](#🌵尽量不要使用new)
- [🌵灵活对函数进行类型转换](#🌵灵活对函数进行类型转换)
- [🌵了解变长参数的妙用](#🌵了解变长参数的妙用)
  - [基本特点-常见](#基本特点常见)
  - [基本使用](#基本使用)
  - [实参与形参不匹配的问题](#实参与形参不匹配的问题)
  - [变长参数+类型验证=重载函数](#变长参数类型验证重载函数)
  - [功能选项模式](#功能选项模式)
- [参考](#参考)

## 前言
`func`是`Go`中的一等公民。

所谓一等公民，表现为`func`类型：

* 可以赋值给变量
* 可以作为函数的参数
* 可以作为函数的返回值、
* 可以自定义类型，如`net/http`包中
```go
type HandlerFunc func(ResponseWriter, *Request)
```

在下文中，会说明为何`http.HandlerFunc`这个`net/http`包自定义的函数类型是如何的强大。

后面也会探讨在函数的定义和使用中容易出错的方式和场景，不过在此之前，让我们先对函数的使用做一些规范，以便达成共识。

## 🌵function定义的规范行为
### 若参数有 context.Context 类型，必做第一个参数
```go
func getContent(ctx content.Context, ...) error
```

### 若返回 error，必将最后一个返回值
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

### 函数名中禁止出现 filter

`filter`意思着“过滤”，是一种常见的操作。为何要禁止使用？

![](media/16633077919315/16643488487757.jpg)


因为`filter`过滤本身可以有两种含义：你是想要留在筛子里面的东西呢？还是想要被筛下去的东西呢？

在我们的项目中，一般指的是：给定集合，给定筛选条件，我们需要得到符合条件的集合

可以用`match`、`qualify`等词替代

## 🚩具名返回值：谨慎使用
所谓具名返回值，顾名思义，就是给返回值命名一下，如

```go
// 普通函数定义
func add(a,b int) (int) {}

// 使用具名返回值的函数定义
func add(a,b int) (result int) {}
```

### 明确具名返回值不是函数签名的一部分

如何证明？反证法。

定义一个接口，接口中方法使用具名返回值的形式。

然后我们自定义一个类型，实现这个方法，不过我们不采用具名返回值。

看看自定义类型在编译器的检查下，是否实现了该接口。

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
运行不会报错，说明自定义类型`Compass`实现了`locator`接口。

证毕：具名返回值不是函数签名的一部分。

### 存在的目的，为了表达清晰
```go
// bad 
func getAddress(addr string) (float64, float64, error)

// good 
// 如此一来，更加清晰
func getAddress(addr string) (lat float64, log float64, err error) 
```

### 函数返回方式的风格要统一

比如下面就是一个反面的示例

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
另外，关于具名返回值，有一个编程建议

* 如果函数本身比较长，就不要省略`return`的参数


## 🚩init 函数：能不用就不用
首先看下`init`内置函数的特点
### init 函数的特点

* `init`和`main`一样，均为`func()` 函数类型： 无参数、无返回值。

* 在`Go`中初始化一个`package`的流程：首先初始化`package`中的常量、变量，然后就是`init`函数。

* 一个`package`中可以有多个`init`函数，甚至一个文件中都可以定义多个`init`函数，按照”先到先得“的顺序执行。

### init 函数的缺点

* 无法很好的处理`init`内部的`error`，因为无法返回值。

* 不能方便的测试`init`中的逻辑。

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
// 直接干掉init函数
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

### init 万一使用，需要满足
1. Be completely deterministic, regardless of program environment or invocation.

2. Avoid depending on the ordering or side-effects of other `init()` functions.
   While `init()` ordering is well-known, code can change, and thus
   relationships between `init()` functions can make code brittle and
   error-prone.

3. Avoid accessing or manipulating global or environment state, such as machine
   information, environment variables, working directory, program
   arguments/inputs, etc.

4. Avoid I/O, including both filesystem, network, and system calls.

### 使用 init 函数的经典案例

虽然`init`函数最好别用，但是并非不可用。下面就让我们从`database/sql`标准库欣赏`init`函数的妙用。

使用`sql databases`的通用步骤
```go
// Step 1: import the main SQL package
import "database/sql"

// Step 2: import a driver package to use a specific SQL database
import _ "github.com/mattn/go-sqlite3"

// 或者像我们项目中的mysql
// _ "github.com/go-sql-driver/mysql"

// Step 3: open a database using a registered driver name
func main() {
  // ...
  db, err := sql.Open("sqlite3", "database.db")
  // ...
}
```

下面细究一下上述的三个步骤。

* 第一步
```go
// database/sql/sql.go
var (
	driversMu sync.RWMutex
	drivers   = make(map[string]driver.Driver) 
)
```
导入`database/sql`，便会初始化`drivers`。

正如文档所说：`The sql package must be used in conjunction with a database driver.`

我们需要将具体的`sql driver`注册到`drivers map`上。

为此`database/sql`包提供了注册方法。
```go
// database/sql/sql.go

// Register makes a database driver available by the provided name.
// If Register is called twice with the same name or if driver is nil,
// it panics.
func Register(name string, driver driver.Driver) {
	driversMu.Lock()
	defer driversMu.Unlock()
	if driver == nil {
		panic("sql: Register driver is nil")
	}
	if _, dup := drivers[name]; dup {
		panic("sql: Register called twice for driver " + name)
	}
	drivers[name] = driver
}
```

* 第二步

`import _ "github.com/go-sql-driver/mysql"`不会产生任何影响，**除了运行`mysql`包中的`init`**。

根据上面所述，猜的没错的话，`mysql`的`init`应该是将`mysql`的`driver`注册到`database/sql`包中的`drivers map`上。

```go
// github.com/go-sql-driver/mysql/driver.go

func init() {
	sql.Register("mysql", &MySQLDriver{})
}
```

* 第三步

既然`mysql driver`已经注册上了，那么就开始使用吧~

```go
// database/sql/sql.go 

func Open(driverName, dataSourceName string) (*DB, error) {
	driversMu.RLock()
	driveri, ok := drivers[driverName]
	driversMu.RUnlock()
	if !ok {
		return nil, fmt.Errorf("sql: unknown driver %q (forgotten import?)", driverName)
	}


	return ...
}
```


如果你对这个话题有更深的兴趣，推荐查看参考中的`<design-patterns-in-gos-databasesql-package>`文章。

## 🚩defer 函数：合理使用
### defer 的定义
> A defer statement defers the execution of a function until the surrounding function returns.

一句话总结：`defer`就是用来延迟执行函数的技术手段。

虽然只有短短一句话，但是我们也可以从中获取很多信息：

1、`defer`后面跟函数:`defers the execution of a `**`function`**（和方法`method`，后文会讲到方法`method`本质上就是函数`function`）。

2、`defer`的作用就是在特定时机激活后接的函数:`the execution`。

3、所谓的特定时机:`until the surrounding function returns`。

我们会发现，下面这句话就是对上述三条总结的再次说明。

> A "defer" statement invokes a function whose execution is deferred to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because the corresponding goroutine is panicking.

### 为什么要有 defer
再次重申，**`defer`的本质就是延迟自动执行函数**。

它的出现就是为了更加便利的进行资源管理，减轻程序员的心智负担，提高代码的可读性、可维护性和可扩展性。

假如`Go`没有`defer`机制，为了实现一个锁保护的文件操作。我们得这么写：

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

我们每次操作，都得处理资源的释放：锁的释放、文件句柄的关闭。

仅仅两个资源的释放操作，我们在编写代码就需要『如临深渊，如履薄冰』。增加资源的参与数，那简直不堪设想。

幸好，`Go`有`defer`机制，我们可以改写如下：

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

### defer 的特性

上面对`defer`的描述，只是非常粗糙的讲解。下面让我们沉浸到`defer`的具体技术细节。

为了讨论一个技术，提出好的问题是理解其要领的因素之一。比如：

* `defer`后接函数，那么函数的参数什么时候确定？（要知道同样的变量，在函数退出时与刚进入函数时，内容可能是不一样的）
* 可以后接方法吗？

* `defer`后的函数真正执行时机是：声明`defer`表达式的函数执行结束时。那么这个所谓的函数执行结束时，是返回后，还是返回前？还是其他什么时间点？
* 如果一个函数之中，声明了多个`defer`表达式，虽然都是函数退出时调用，但是它们执行的顺序如何？随机的？（像`select channel`一样）？先进先出？（越先被`defer`的函数越早调用）？还是先进后出？

没错，有很多问题会萦绕心头。来吧，让我们回答上面的问题。

不过在此之前，学习两个英文单词。

`execution`：中文是执行的意思。比如函数的执行，就是运行函数的意思。

`evaluate`：中文是评价、估计的意思。在编程里面，比如函数参数，当我们说`evaluate function parameters`，就是确定函数参数值的意思。嗯，`evaluate`可以翻译成「求值」。

下面进入正题：

**1、defered function evaluates its parameter when is occur**

被`defer`的函数会立马求值函数的参数

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

会打印`666`而不是`888`。

这就叫做当出现`defer`时，后接函数的参数立马被求值，继而确定，即`i=666`。

虽然`deferred function`在后面才会执行，但是参数从一开始就确定了。

**2、defered function will execute before real return**
被`defer`的函数会在调用它的函数返回前执行

这句话不好理解。重点在于什么叫做`real return`。

休息一下，下面的例子需要仔细看：

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

可以观察到，`sara`和`lisa`基本逻辑是一样的，唯一的不同就是`sara`是具名返回值函数，而`lisa`是匿名返回值函数。而这一点区别可太重要了，直接导致不同的处理逻辑。

好好理解一下上面的例子，下面给出`Go`函数的返回值模型

其实`go`的`return`语句是分成两部分的。比如对于匿名返回值函数`lisa`来说

```go
return result
相当于

1、var ret = result // ret 是真返回值
2、return ret
```

对于具名返回值函数`sara`来说

```go
return result
相当于

1、var result = result
2、return result
```

注意两者的区别。

重点来啦，`defer`的运行时机就是步骤`1`和步骤`2`之间！

* 对于普通函数`lisa`来说，我要返回的是`ret`，就算你在`defer`中修改了`result`，也不会影响到最终返回值哦。
* 对于具名返回值函数`sara`就不一样了，她返回的就是`result`，在`defer`中修改了`result`，最终的结果就不一样啦。

咳咳，具名返回值和`defer`的联合使用，要注意哦😑。别写这样的代码。

**3、多个`defer`的执行的顺序是：后进先出，类似栈**
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

### 如何记住 defer 的特性

`defer`出现就是为了资源的管理。

比如有以下这个过程：首先执行某个操作，需要首先获取资源`A`，然后获取资源`B`，最后获取资源`C`。那么`defer`的顺序，肯定是先放回`C`资源，然后`B`资源，最后是`A`资源。

这个过程让我想起线性代数中矩阵的乘法和逆运算——逆矩阵。

比如`AB`，表示矩阵`A`乘以矩阵`B`，那么它俩的逆运算呢？

$(AB)^{-1} = B^{-1}A^{-1}$

当时为了记住这个特性，我是记住了下面的这个类比：

先穿袜子，然后穿鞋子；如果这一过程反过来，那就是先拖鞋，再脱袜子。

`nice`。

你不能先脱袜子，再拖鞋啊！

如今看来，不论是穿衣服、矩阵乘法、资源的管理，都是一种行为、一种运动、一种变化。他们是分先后的，所以他们的“反运动”要与正向运动相反。

### 不要在循环中滥用 defer
详见`Go`编程指引与陷阱`2`控制结构中的`for`循环章节。

## 🚩只在 main 函数中退出
使用`os.Exit`或者`log.Fatal`退出

`log.Fatal`内部使用`os.Exit`

## 🌵尽量不要使用 new
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

使用`new`徒增代码行数。

## 🌵灵活对函数进行类型转换
像对整型变量那样`int64(13240366)`
```go
func greeting(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Welcome, Gopher!\n")
}

func main() {
    // 直接传递greeting是会编译出错的
    // 因为ListenAndServe第二参数需要Handler类型

	//http.ListenAndServe(":8080", http.HandlerFunc(greeting))
	http.ListenAndServe(":8080", greeting) 
}
```
为了满足Handler接口，只需要简单的进行一次转换。堪称魔法。
```go
// net/http/server.go 
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}

// 魔法！
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```
本质上就是一种强制类型转换，然后转换后的类型实现了`Handler`接口。

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

如果对该主题感兴趣，可参考本人拙作[Go语言系列之HTTP服务器](https://zhuanlan.zhihu.com/p/588192056?)

## 🌵了解变长参数的妙用
### 基本特点-常见

接收变长参数的函数如同空气一般：常见常用但往往没有引起人们的注意。比如：
```go
func Printf(format string, a ...interface{}) (n int, err error) 

func Sprintf(format string, a ...interface{}) string 

func append(slice []Type, elems ...Type) []Type
```
### 基本使用
* 函数内使用

**对待变长参数如同切片一般。**

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

1、可`ts...`使用，其中`ts`是`[]T`切片类型。

2、可多个`T`使用。

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

* 一个我们经常用到的拼接函数

将传入的参数通过指定的连接符串联起来，返回一个字符串。

```go
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

* 为什么`Go`不支持函数重载

我们直接引用`Go FAQ`吧

> Experience with other languages told us that having a variety of methods with the same name but different signature was occasionally useful but that it could be confusing and fragile in practice.
> Regarding operator overloading, it seems more a convenience than an absolute requirement. Again, things are simpler without it.

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


## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
* [Embedding in Go: Part 3 - interfaces in structs](https://eli.thegreenplace.net/2020/embedding-in-go-part-3-interfaces-in-structs/)

