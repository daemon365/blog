---
title: "golang nethttp包"
date: "2020-01-11T00:00:00+08:00"
tags: 
- go
showToc: true
---


## http协议

超文本传输协议（HTTP，HyperText Transfer Protocol)是互联网上应用最为广泛的一种网络传输协议，所有的WWW文件都必须遵守这个标准。设计HTTP最初的目的是为了提供一种发布和接收HTML页面的方法。

关于http(https)协议: [https://www.cnblogs.com/yuemoxi/p/15162601.html](https://www.cnblogs.com/yuemoxi/p/15162601.html)

## http包中重要的类型和接口

*   server：HTTP服务器，定义监听的地址、端口，处理器等信息。
*   conn：用户每次请求的链接。
*   response：返回给用户的信息。
*   request：用户请求的信息。
*   Handler: 处理器，用户请求到来时，服务器的处理逻辑，它是一个包含ServeHTTP方法的接口。

## http包的运行流程

![](/images/8f5c944f-7915-4f74-8efe-e9253fea0c5d.png)

### http包如何实现高并发

http包中，server每接收一个用户请求，都会生成一个conn链接，并生成一个goroutines来处理对应的conn。所以每个请求都是独立的，相互不阻塞的。

```go
c := srv.newConn(rw)
c.setState(c.rwc, StateNew) 
go c.serve(ctx)
```

### 处理器(Handler)和多路复用器(ServeMux)

```go
func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
    handler := sh.srv.Handler
    if handler == nil {
        handler = DefaultServeMux
    }
    if req.RequestURI == "*" && req.Method == "OPTIONS" {
        handler = globalOptionsHandler{}
    }
    handler.ServeHTTP(rw, req)
}
```

所以当我们的处理器为空时，http包会默认帮我们生成一个DefaultServeMux处理器。 不是说第二个参数是处理器吗，但是DefaultServeMux是一个ServeMux结构的实例, 是一个多路复用器。这是怎么回事？ 其实处理器是一个拥有ServeHTTP方法的接口，只要实现了这个方法，它就是一个处理器。所以DefaultServeMux不仅是一个多路复用器，而且还是一个处理器。只是这个处理器比较特殊，它只是根据请求的URL将请求分配到对应的处理器。

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
```

我们可以自定义自己的处理器

```go
package main
import (
    "fmt"
    "net/http"
)
// 定一个自定义的Handler的结构
type MyHanlder struct{}
// 为MyHanlder结构实现Hanlder接口的ServeHTTP的方法，此时MyHandler将是一个处理器(Handler)
func (h MyHanlder) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This my handler")
}
func main() {
    // 实例化MyHandler
    hanlder := MyHanlder{}
    //启动服务器并监听8000/tcp端口
    err := http.ListenAndServe(":8000", hanlder)
    if err != nil {
        fmt.Println(err)
    }
}
```

### ServeMux和DefaultServeMux

前面我们说DefaultServeMux是一个ServeMux的实例，它不仅是一个多路复用器，而且是一个处理器。多路复用器具有路由功能，可以根据请求的URL将请求传递给对应的处理器处理。看下http包中默认的多路复用器是怎么实现的：

```go
type ServeMux struct {
    mu    sync.RWMutex  //锁，应为请求是并发处理的，所以这里需要锁机制
    m     map[string]muxEntry //路由规则的map，一个string对应一个muxEntry
    es    []muxEntry 
    hosts bool       
}
type muxEntry struct {
    h       Handler    //string对应的处理器
    pattern string     //匹配的字符串
}
```

默认的多路复用器是根据请求的URL与存储的map做匹配，匹配到了就返回对应的handler，然后调用该handler的ServeHTTP方法来处理请求。

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
    if r.RequestURI == "*" {
        if r.ProtoAtLeast(1, 1) {
            w.Header().Set("Connection", "close")
        }
        w.WriteHeader(StatusBadRequest)
        return
    }
    h, _ := mux.Handler(r)
    h.ServeHTTP(w, r)
}
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
    if r.Method == "CONNECT" {
        if u, ok := mux.redirectToPathSlash(r.URL.Host, r.URL.Path, r.URL); ok {
            return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
        }
        return mux.handler(r.Host, r.URL.Path)
    }
    host := stripHostPort(r.Host)
    path := cleanPath(r.URL.Path)
    if u, ok := mux.redirectToPathSlash(host, path, r.URL); ok {
        return RedirectHandler(u.String(), StatusMovedPermanently), u.Path
    }
    if path != r.URL.Path {
        _, pattern = mux.handler(host, path)
        url := *r.URL
        url.Path = path
        return RedirectHandler(url.String(), StatusMovedPermanently), pattern
    }
    return mux.handler(host, r.URL.Path)
}
func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
    mux.mu.RLock()
    defer mux.mu.RUnlock()
    if mux.hosts {
        h, pattern = mux.match(host + path)
    }
    if h == nil {
        h, pattern = mux.match(path)
    }
    if h == nil {
        h, pattern = NotFoundHandler(), ""
    }
    return
}
```

我们也可以根据接口定义实现自己简单的路由器

```go
package main
import (
    "fmt"
    "net/http"
)
// 自定义一个多路复用器的结构
type MyMux struct{}
// 实现ServeHTTP方法
func (m MyMux) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 判断URL并转到对应的handler处理
    if r.URL.Path == "/index" {
        IndexHandler(w, r)
        return
    }
    http.NotFound(w, r)
    return
}
func IndexHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "This is index page")
}
func main() {
    // 实例化一个自定义的路由器
    mux := MyMux{}
    err := http.ListenAndServe(":8000", mux)
    if err != nil {
        fmt.Println(err)
    }
}
```

## HTTP客户端

### 基本的HTTP/HTTPS请求

Get、Head、Post和PostForm函数发出HTTP/HTTPS请求。

```go
resp, err := http.Get("http://example.com/")
...
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
...
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})
```

程序在使用完response后必须关闭回复的主体。

```go
resp, err := http.Get("http://example.com/")
if err != nil {
	// handle error
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
// ...
```

### GET请求示例

使用net/http包编写一个简单的发送HTTP请求的Client端，代码如下：

```go
package main
import (
	"fmt"
	"io/ioutil"
	"net/http"
)
func main() {
	resp, err := http.Get("https://www.liwenzhou.com/")
	if err != nil {
		fmt.Printf("get failed, err:%v\n", err)
		return
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("read from resp.Body failed, err:%v\n", err)
		return
	}
	fmt.Print(string(body))
}
```

将上面的代码保存之后编译成可执行文件，执行之后就能在终端打印liwenzhou.com网站首页的内容了，我们的浏览器其实就是一个发送和接收HTTP协议数据的客户端，我们平时通过浏览器访问网页其实就是从网站的服务器接收HTTP数据，然后浏览器会按照HTML、CSS等规则将网页渲染展示出来。

### 带参数的GET请求示例

关于GET请求的参数需要使用Go语言内置的net/url这个标准库来处理。

```go
func main() {
	apiUrl := "http://127.0.0.1:9090/get"
	// URL param
	data := url.Values{}
	data.Set("name", "小王子")
	data.Set("age", "18")
	u, err := url.ParseRequestURI(apiUrl)
	if err != nil {
		fmt.Printf("parse url requestUrl failed, err:%v\n", err)
	}
	u.RawQuery = data.Encode() // URL encode
	fmt.Println(u.String())
	resp, err := http.Get(u.String())
	if err != nil {
		fmt.Printf("post failed, err:%v\n", err)
		return
	}
	defer resp.Body.Close()
	b, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("get resp failed, err:%v\n", err)
		return
	}
	fmt.Println(string(b))
}
```

对应的Server端HandlerFunc如下：

```go
func getHandler(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()
	data := r.URL.Query()
	fmt.Println(data.Get("name"))
	fmt.Println(data.Get("age"))
	answer := `{"status": "ok"}`
	w.Write([]byte(answer))
}
```

### Post请求

上面演示了使用net/http包发送GET请求的示例，发送POST请求的示例代码如下：

```go
package main
import (
	"fmt"
	"io/ioutil"
	"net/http"
	"strings"
)
// net/http post demo
func main() {
	url := "http://127.0.0.1:9090/post"
	// 表单数据
	//contentType := "application/x-www-form-urlencoded"
	//data := "name=小王子&age=18"
	// json
	contentType := "application/json"
	data := `{"name":"小王子","age":18}`
	resp, err := http.Post(url, contentType, strings.NewReader(data))
	if err != nil {
		fmt.Printf("post failed, err:%v\n", err)
		return
	}
	defer resp.Body.Close()
	b, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		fmt.Printf("get resp failed, err:%v\n", err)
		return
	}
	fmt.Println(string(b))
}
```

对应的Server端HandlerFunc如下：

```go
func postHandler(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()
	// 1\. 请求类型是application/x-www-form-urlencoded时解析form数据
	r.ParseForm()
	fmt.Println(r.PostForm) // 打印form数据
	fmt.Println(r.PostForm.Get("name"), r.PostForm.Get("age"))
	// 2\. 请求类型是application/json时从r.Body读取数据
	b, err := ioutil.ReadAll(r.Body)
	if err != nil {
		fmt.Printf("read request.Body failed, err:%v\n", err)
		return
	}
	fmt.Println(string(b))
	answer := `{"status": "ok"}`
	w.Write([]byte(answer))
}
```

### Client

要管理HTTP客户端的头域、重定向策略和其他设置，创建一个Client：

```go
client := &http.Client{
	CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http://example.com")
// ...
req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)
resp, err := client.Do(req)
// ...
```

#### 自定义Transport

要管理代理、TLS配置、keep-alive、压缩和其他设置，创建一个Transport：

```go
tr := &http.Transport{
	TLSClientConfig:    &tls.Config{RootCAs: pool},
	DisableCompression: true,
}
client := &http.Client{Transport: tr}
resp, err := client.Get("https://example.com")
```

Client和Transport类型都可以安全的被多个goroutine同时使用。出于效率考虑，应该一次建立、尽量重用。

## 服务端

### 默认的Server

ListenAndServe使用指定的监听地址和处理器启动一个HTTP服务端。处理器参数通常是nil，这表示采用包变量DefaultServeMux作为处理器。

Handle和HandleFunc函数可以向DefaultServeMux添加处理器。

```go
http.Handle("/foo", fooHandler)
http.HandleFunc("/bar", func(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %q", html.EscapeString(r.URL.Path))
})
log.Fatal(http.ListenAndServe(":8080", nil))
```

### 默认的Server示例

使用Go语言中的net/http包来编写一个简单的接收HTTP请求的Server端示例，net/http包是对net包的进一步封装，专门用来处理HTTP协议的数据。具体的代码如下：

```go
// http server
func sayHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello 沙河！")
}
func main() {
	http.HandleFunc("/", sayHello)
	err := http.ListenAndServe(":9090", nil)
	if err != nil {
		fmt.Printf("http server failed, err:%v\n", err)
		return
	}
}
```

将上面的代码编译之后执行，打开你电脑上的浏览器在地址栏输入127.0.0.1:9090回车，此时就能够看到如下页面了。![](/images/c35e630d-731a-4c19-af21-01ab6838d42f.png)

### 自定义Server

要管理服务端的行为，可以创建一个自定义的Server：

```go
s := &http.Server{
	Addr:           ":8080",
	Handler:        myHandler,
	ReadTimeout:    10 * time.Second,
	WriteTimeout:   10 * time.Second,
	MaxHeaderBytes: 1 << 20,
}
log.Fatal(s.ListenAndServe())
```

## 参考文章

*   [https://www.jianshu.com/p/c90ebdd5d1a1](https://www.jianshu.com/p/c90ebdd5d1a1)
*   [https://www.liwenzhou.com/posts/Go/go_http/](https://www.liwenzhou.com/posts/Go/go_http/)