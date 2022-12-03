本节主要讨论`Go`的标准库，一些常见的使用问题。


* 一些约定

  🌵：表示「能知道最好」，如果不知道也不会导致错误。

  🚩：表示「最起码要知道」，如果不知道很可能写出不好找的`bug`、性能问题。

  🈲：表示「这个就别做到了」，如果不知道就非常可能出问题。


- [time](#time)
  - [🌵使用time来处理时间](#🌵使用time来处理时间)
  - [🌵使用time.Duration处理时间段](#🌵使用time-duration处理时间段)
  - [🌵谨慎使用time.After](#🌵谨慎使用time-after)
- [json](#json)
  - [🌵marshal 一个map的顺序](#🌵marshal一个-map的顺序)
  - [🌵JavaScript parses integers as floats and your int64 might overflow](#🌵javascript-parses-integers-as-floats-and-your-int64-might-overflow)
- [http](#http)
  - [🌵Prefer http.HandlerFunc over http.Handler](#🌵prefer-http-handlerfunc-over-http-handler)
  - [🚩always close http body aka defer r.Body.Close()](#🚩always-close-http-body-aka-defer-r-body-close)
  - [🚩should read http body regardless the use of http body](#🚩should-read-http-body-regardless-the-use-of-http-body)
  - [想要复用TCP连接](#想要复用tcp连接)
  - [不想复用TCP连接](#不想复用tcp连接)
- [参考](#参考)


## time
### 🌵使用time来处理时间

If you are comparing timestamps, use time.Before or time.After. Don't use time.Sub to get a duration and then check its value.

### 🌵使用time.Duration处理时间段
```go
// BAD
delay := time.Second * 60 * 24 * 60

// VERY BAD
delay := 60 * time.Second * 60 * 24

// GOOD
delay := 24 * 60 * 60 * time.Second

// EVEN BETTER
delay := 24 * time.Hour
```

<thead><tr><th>Bad</th><th>Good</th></tr></thead>
<tbody>
<tr><td>

```go
func poll(delay int) {
  for {
    // ...
    time.Sleep(time.Duration(delay) * time.Millisecond)
  }
}

poll(10) // was it seconds or milliseconds?
```

</td><td>

### 🌵谨慎使用time.After
> Remember that the resources created will only be released when the timer expires.

所以不建议在`for`循环、`HTTP`请求等重复执行的代码逻辑中，使用`time.After`

要点就是不要忘记停止`ticker`
```go
ticker := time.NewTicker(1 * time.Second)
defer ticker.Stop()
```

## json

### 🌵marshal 一个map的顺序
虽然`map`本身是没有顺序可言的，但是`json.Marshal map`，会按照`map`的key作为排序规则
```go
english := map[string]int{
	"b": 1,
	"z": 3,
	"y": 2,
	"a": 1,
}

chinese := map[string]int{
	"我": 1,
	"是": 2,
	"名": 1,
	"跑": 1,
	"卡": 1,
	"丁": 1,
	"车": 1,
}

func main() {
	raw, _ := json.Marshal(english)
	fmt.Println(string(raw)) // {"a":1,"b":1,"y":2,"z":3}

	raw, _ = json.Marshal(chinese)
	fmt.Println(string(raw)) // {"丁":1,"卡":1,"名":1,"我":1,"是":2,"跑":1}
}
```

### 🌵JavaScript parses integers as floats and your int64 might overflow
```go
type Request struct {
	ID int64 `json:"id,string"`
}
```

## http
### 🌵Prefer http.HandlerFunc over http.Handler

### 🚩always close http body aka defer r.Body.Close()

### 🚩should read http body regardless the use of http body
> Otherwise, HTTP client's Transport will not reuse connections unless the body is read to completion and closed.
```go
io.Copy(ioutil.Discard, resp.Body) // if you don't use http body
```

### 想要复用TCP连接
读取`response.Body`的内容 并且 读取之后调用`r.Body.Close()`

### 不想复用TCP连接
* request.Closed = true
* client.Transport.KeepAlive=false


## 参考
* `白明《Go语言精进之路》(📚)`
* `[100 go mistakes](📚)`
* [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md)
* [Effective Go](https://go.dev/doc/effective_go)
