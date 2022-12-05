- [我的疑问](#我的疑问)
- [HTTP请求的生命周期](#http请求的生命周期)
- [解决我的疑问](#解决我的疑问)
- [说到底，ServeMux是个啥啊](#说到底，servemux是个啥啊)
- [让我们继续向更多的handler前进](#让我们继续向更多的handler前进)
- [总结](#总结)
- [参考](#参考)


# 我的疑问

让我们开门见山吧

```go
package main

import (
    "fmt"
    "net/http"
)

func hello(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "hello\n")
}
func headers(w http.ResponseWriter, req *http.Request) {
    for name, headers := range req.Header {
        for _, h := range headers {
            fmt.Fprintf(w, "%v: %v\n", name, h)
        }
    }
}

func main() {
    http.HandleFunc("/hello", hello)
    http.HandleFunc("/headers", headers)
    http.ListenAndServe(":8090", nil)
}
```

上述的代码很简答，运行之后的结果就是开启了一个端口为`8090`的`HTTP`服务器。

我初次见到类似的代码，我最大的疑问就是`ListenAndServe`第二个参数为啥是`nil`啊！

```go
func ListenAndServe(addr string, handler Handler) error
```

ListenAndServe第二个参数需要一个实现`Handler`的实例

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

但是在上面的代码中，居然传递了`nil`。

然而我们使用`curl`访问之，却发现`hello`、`headers`两个`handler`已经注册上了，也就是说绑定上了。

```shell

✘ 🐂🍺  curl -v http://127.0.0.1:8080/hello

* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8080 (#0)
> GET /hello HTTP/1.1
> Host: 127.0.0.1:8080
> User-Agent: curl/7.54.0
> Accept: */*
>

< HTTP/1.1 200 OK
< Date: Thu, 27 Oct 2022 06:54:09 GMT
< Content-Length: 12
< Content-Type: text/plain; charset=utf-8
<

* Connection #0 to host 127.0.0.1 left intact
hello world.%
```

答案当然是很简单的，代码的世界没有魔法，一定是`net/http`具体的实现细节帮助我们进行了连接。

与其单纯的探究这个问题，我们不妨提出一个更好的问题：`Go`语言的`HTTP`服务器的生命周期是什么样的。具体来说，当一个`request`请求进入，`HTTP`服务器是如何将之交给与对应的`handler`处理，如何进行的匹配？如何返回的`response`？

# HTTP请求的生命周期

经过深入代码的阅读，整理一下`http`请求如何与`http server`交互的


![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4f53a0456d0741338a4fbfc742e92e30~tplv-k3u1fbpfcp-watermark.image?)

* `http.ListenAndServe`首先在某端口上开始监听

* `http server`收到`http`请求之前，先建立两端的`tcp`连接

* `http server go`一个`goroutine`，专门处理这个请求

1、在`goroutine`内部解析出`request`

2、根据请求路径匹配对应的`handler`

3、调用`handler`，处理这个请求，返回`response`

# 解决我的疑问

之前我提到的自定义的业务`handler`是如何绑定到`HTTP server`上的，现在根据上一节的「`HTTP`请求的生命周期」，我们很容易就能定位将这二者串通起来的连接点——读取`req`之后的`ServeHTTP`方法。

重点就是这个『调用`handler`』的步骤

```go
// net/http/server.go
// 找到咯，就是在这里
// 这里就是调用handler的入口
serverHandler{c.server}.ServeHTTP(w, w.req)
```

下面进入这个调用的内部实现

```go
type serverHandler struct {
    srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    // 重点
    if handler == nil {
        handler = DefaultServeMux
    }

    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }

    handler.ServeHTTP(rw, req)
}
```

啊哈！因为我们调用`http.ListenAndServe(":8090", nil)`时第二个参数为`nil`，所以`HTTP`服务器默认启用`DefaultServeMux`作为自己的`handler`。

```go
if handler == nil {
    handler = DefaultServeMux
}
```

而我们自定义的`hello/headers`两个`handler`就是注册在`DefaultServeMux`上的。

```go
// 将hello/headers 绑定
http.HandleFunc("/hello", hello)
http.HandleFunc("/headers", headers)

// http.HandleFunc的实现
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}
```

这样一切就明了啊，优咔哒，优咔哒！

# 说到底，ServeMux是个啥啊

所以关`ServeMux`什么关系？

不要急，是这样的。让我们回顾一下。

```go
http.HandleFunc("/hello", hello)

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
    DefaultServeMux.HandleFunc(pattern, handler)
}

var DefaultServeMux = &defaultServeMux

var defaultServeMux ServeMux
```

没错，DefaultServeMux就是`*ServeMux`.当我们调用`http.ListenAndServe(":8080", nil)`启动`http`服务器，且第二个参数为`nil`时，从本质上来说，就是把`DefaultServeMux`作为第二个参数传递进去了啊。

好吧好吧，你说了这么多，不是第一节就已经说明的事情嘛。干嘛还翻来覆去的说个不停呢？

好，下面进入重点。

`http.ListenAndServe`的第二个参数是个啥类型？其实上面已经说过了，是一个实现`http.Handler`接口的实例

```go
type Handler interface {
    ServeHTTP(ResponseWriter, *Request)
}
```

所以从本质上来说，`DefaultServeMux`就是一个`Handler`实例而已。

所以为了实现`hello handler`的逻辑，即用户访问`http`服务器，返回用户`hello`字符串信息。

我们完全可以这么写。

```go
type HelloHandler struct {
}

func (*HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "hello world.")
}

func main() {
    helloHandler := &HelloHandler{}
    http.ListenAndServe(":8080", helloHandler)
}
```

可以看到，我们自定义了`HelloHandler`类型，其指针类型实现了`Handler`接口，所以直接传入`http.ListenAndServe`，即可发挥作用。此时我们`curl`下，成功返回`“hello world."`字符串。

但是问题有两个

1、提供一个`handler`处理函数，就需要我们自定义一个类型，然后该类型实现`ServeHTTP`方法以便实现`Handler`接口。这也太麻烦了吧

啊，对对对对，说的没错。所以构建`net/http`包的大佬，为我们提供了一个`http.HandlerFunc`类型。

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

`HandlerFunc`本质上是一个自定义类型，实现了`ServeHTTP`方法。

所以我们可以这么用`http.HandlerFunc(hello)`，就可以将类似`hello`这样平平无奇的函数，变成实现`http.Handler`接口的实例.

`HandlerFunc(hello)`类似整数类型的强制转换，比如`int32(666)`.如此一来，`func hello(w http.ResponseWriter, r *http.Request)`被转换成了`HandlerFunc`类型，即`handler`实例。当调用`ServeHTTP`的时候，其实就是调用了`hello`本身嘛。

```go
func hello(w http.ResponseWriter, r *http.Request) {
    fmt.Fprint(w, "hello world.")
}

func main() {
    http.ListenAndServe("8080", http.HandlerFunc(hello))
}
```

真是精彩

2、虽然一行代码就启动了`http`服务器。可是这个服务器，好像只有一个`handler`方法？

```go
http.ListenAndServe("8080", http.HandlerFunc(hello))
```

不论我们

```shell
curl -v http://127.0.0.1:8080/h
curl -v http://127.0.0.1:8080/
curl -v http://127.0.0.1:8080/hello
curl -v http://127.0.0.1:8080/shit
```

都会一丝不苟的返回`hello world`字符串。

咋搞哒!这能叫`http`服务器？这不就是一个小玩具吗？

所以让我们把眼睛放回到最开始

```go
func main() {
    http.HandleFunc("/hello", hello) // 注册到DefaultServeMux上，A ServeMux is just a Handler
    http.HandleFunc("/headers", headers)
    //
    http.ListenAndServe(":8080", nil)
}
```

`http.HandleFunc`分明就是把路由注册到了`DefaultServeMux`上！然后给每个`tcp`连接开启的`goroutine`上，通过适配器`serverHandler`会匹配请求的路由，调用相应的`handler`。

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }

    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }

    // 调用DefaultServeMux的ServeHTTP方法
    // ServeHTTP方法实现路由匹配机制，匹配之后调用我们自定义的handler处理方法
    handler.ServeHTTP(rw, req)
}
```

`DefaultServeMux`的`ServeHTTP`逻辑

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }

        w.WriteHeader(StatusBadRequest)
            return
    }

    h, _ := mux.Handler(r) // 根据r的请求路由，返回相应的下一级handler
    h.ServeHTTP(w, r) // 调用我们自定义的handler，比如hello
}
```

哦，一切明了。

`DefaultServeMux`本质上就是一个实现`http.Handler`接口的`Handler`实例。

`DefaultServeMux`里面也注册了很多`Handler`，根据路由选择对应的`Handler`。

所以我们完全可以将最开始的`HTTP`服务器代码改造如下

```go

func main() {
    //http.HandleFunc("/hello", hello)
    //http.HandleFunc("/headers", headers)
    smx := http.NewServeMux()
    smx.HandleFunc("/hello", hello)
    smx.HandleFunc("/headers", headers)

    http.ListenAndServe(":8090", smx)
}
```

我们主动创建了一个`ServeMux`，然后将两个自定义的`handler`绑定，最后调用`ListenAndServe`的时候主动将`ServeMux`作为第二个参数传递。这样可比传递`nil`，内部使用`DefaultServeMux`容易让人理解的多呀。

看到这里，你是否已经晕了？就算晕了也不要紧，在你回看之前。重点记住我下面的论述：

* `Q:`当我说`handler`的时候，我指的是什么？

* `A:handler`就是实现了`http.Handler`接口的实例。

* `Q:handler`重要吗？

* `A：`哪里都是`handler`，到处都是`handler`，我们可以自定义`handler`，包内也有自己实现好的`handler`，`handler`是`HTTP`服务器非常重要的一部分。

好了，建议你在重新看一遍上面的章节。

# 让我们继续向更多的handler前进

上面说到到处都是handler，下面继续深化这个概念，并且狂野一点，让我们开启链式handler模式。

`DefaultServeMux`本身是个`handler`，然后上面绑定了我们自定义的`handler`（`hello`、`headers`这两个`handler`），本质上是外层`handler`包裹了内层`handler`。但实际上，内层的`handler`可以继续绑定`handler`，一直组合一直组合，搞一个链式调用。

通过不断的组合，我们可以增加额外的处理逻辑，这就是传说中的前置处理器和后置处理器

`DefaultServeMux`就是有自己的前置处理器。没错，根据路由选择具体的下一级的`Handler`就是`DefaultServeMux`的前置处理器。

为了复习和验证上面的知识，我们搞一个多条链路组合的`handler`

加什么好呢？

搞一个计时日志吧！

干！

```go
package main

import (
    "fmt"
    "net/http"
    "time"
)

type LogHandler struct {
    handler http.Handler
}

func (l LogHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    now := time.Now()
    l.handler.ServeHTTP(w, r)
    fmt.Printf("serve this handler cost %v\n", time.Since(now))
}

// 最内层的handler
// 当目前hello只是一个普通的函数
func helloHandler(w http.ResponseWriter, req *http.Request) {
    fmt.Fprintf(w, "hello\n")
}

func main() {
    smx := http.NewServeMux()
    logHandler := LogHandler{handler: http.HandlerFunc(helloHandler)}
    smx.Handle("/hello", logHandler)
    http.ListenAndServe(":8000", smx)
}
```

当我们发送http请求后

```shell
🐂🍺  curl -v http://127.0.0.1:8000/hello

* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8000 (#0)
> GET /hello HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/7.64.1
> Accept: */*

>
< HTTP/1.1 200 OK
< Date: Wed, 30 Nov 2022 13:57:22 GMT
< Content-Length: 6
< Content-Type: text/plain; charset=utf-8
<

hello
* Connection #0 to host 127.0.0.1 left intact
* Closing connection 0
```

果然打印了

```text
serve this handler cost 56.305µs
```

这次套了一层，我们还可以继续套娃呢!

```go
func main() {
    smx := http.NewServeMux()
    timeoutHello := http.TimeoutHandler(http.HandlerFunc(helloHandler), 1*time.Second, "you are time out")
    logHandler := LogHandler{handler: timeoutHello}
    // defaultServerMux(Handler) > LogHandler > Timeout Handler > Hello

    smx.Handle("/hello", logHandler)

    http.ListenAndServe(":8000", smx)
}
```

当我们调用时

```shell
🐂🍺  curl -v http://127.0.0.1:8000/hello

* Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to 127.0.0.1 (127.0.0.1) port 8000 (#0)
> GET /hello HTTP/1.1
> Host: 127.0.0.1:8000
> User-Agent: curl/7.64.1
> Accept: */*
>

< HTTP/1.1 503 Service Unavailable
< Date: Wed, 30 Nov 2022 14:02:31 GMT
< Content-Length: 16
< Content-Type: text/plain; charset=utf-8
<

* Connection #0 to host 127.0.0.1 left intact
you are time out* Closing connection 0
```

返回了`503`的错误信息哈

`http`服务器打印的日志如下

```text
serve this handler cost 1.005334466s
```

`ps`：有没有觉得这种`handler`的组合内嵌的使用方法，类似于`Python`里面的装饰器呢！


# 总结

`Go`原生的`net/http`包非常强大，默认就提供了并发请求的支持。

`http.Handler`接口是如此的方便和强大，实现一个优秀的`HTTP`服务器，我们要做的事情就是构建各种handler。

为此，我们可以

1、自定义函数，然后通过`http.HandlerFunc`强制转换为`handler`

2、自定义类型，然后通过实现`ServeHTTP`方法成为`handler`实例

3、使用默认的`DefaultServeMux`或者新建`NewServeMux`

4、调用内置的`Handler`函数，比如`TimeoutHandler`

然后综合上述提高的实现`handler`实例的方式，进行组合吧。

我们组合的这个行为，就是一个函数或者方法，接收一个`handler`，返回一个新的`handler`。

**这种方式叫做适配器模式**。

**通过适配器模式组合的`handler`，在`Go`中，称之为中间件(`middleware`)**

`Go`的`net/http`包可真有意思。

# 参考

* [Life of an HTTP request in a Go server](https://eli.thegreenplace.net/2021/life-of-an-http-request-in-a-go-server/)