---
title: kratos http原理
date: '2023-06-29T00:00:00+08:00'
tags:
- go
- kratos
- 源码分析
showToc: true
categories:
- kratos
---



## 概念

[kratos](https://github.com/go-kratos/kratos) 为了使http协议的逻辑代码和grpc的逻辑代码使用同一份，选择了基于protobuf的IDL文件使用proto插件生成辅助代码的方式。

protoc http插件的地址为：[https://github.com/go-kratos/kratos/tree/main/cmd/protoc-gen-go-http](https://github.com/go-kratos/kratos/tree/main/cmd/protoc-gen-go-http)

## 示例

```protobuf
syntax = "proto3";

package helloworld;

option go_package = "test/helloworld;helloworld";
option java_multiple_files = true;
option java_package = "helloworld";
import "google/api/annotations.proto";

service Greeter {
	rpc SayHello (HelloRequest) returns (HelloReply)  {
		  option (google.api.http) = {
			  post: "/helloworld", // 声明路由
			  body: "*"
		  };
	}
  }
  
message HelloRequest {
	string name = 1;
}
  
message HelloReply {
	string msg = 1;
}
```

使用`kratos proto client xxx` 生成的代码为：

```go
// Code generated by protoc-gen-go-http. DO NOT EDIT.
// versions:
// - protoc-gen-go-http v2.4.0
// - protoc             v3.19.4
// source: helloworld/helloworld.proto

package helloworld

import (
	context "context"
	http "github.com/go-kratos/kratos/v2/transport/http"
	binding "github.com/go-kratos/kratos/v2/transport/http/binding"
)

// This is a compile-time assertion to ensure that this generated file
// is compatible with the kratos package it is being compiled against.
var _ = new(context.Context)
var _ = binding.EncodeURL

const _ = http.SupportPackageIsVersion1

const OperationGreeterSayHello = "/helloworld.Greeter/SayHello"

type GreeterHTTPServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
}

func RegisterGreeterHTTPServer(s *http.Server, srv GreeterHTTPServer) {
	r := s.Route("/")
	r.POST("/helloworld", _Greeter_SayHello0_HTTP_Handler(srv))
}

func _Greeter_SayHello0_HTTP_Handler(srv GreeterHTTPServer) func(ctx http.Context) error {
	return func(ctx http.Context) error {
		var in HelloRequest
		if err := ctx.Bind(&in); err != nil {
			return err
		}
		http.SetOperation(ctx, OperationGreeterSayHello)
		h := ctx.Middleware(func(ctx context.Context, req interface{}) (interface{}, error) {
			return srv.SayHello(ctx, req.(*HelloRequest))
		})
		out, err := h(ctx, &in)
		if err != nil {
			return err
		}
		reply := out.(*HelloReply)
		return ctx.Result(200, reply)
	}
}

type GreeterHTTPClient interface {
	SayHello(ctx context.Context, req *HelloRequest, opts ...http.CallOption) (rsp *HelloReply, err error)
}

type GreeterHTTPClientImpl struct {
	cc *http.Client
}

func NewGreeterHTTPClient(client *http.Client) GreeterHTTPClient {
	return &GreeterHTTPClientImpl{client}
}

func (c *GreeterHTTPClientImpl) SayHello(ctx context.Context, in *HelloRequest, opts ...http.CallOption) (*HelloReply, error) {
	var out HelloReply
	pattern := "/helloworld"
	path := binding.EncodeURL(pattern, in, false)
	opts = append(opts, http.Operation(OperationGreeterSayHello))
	opts = append(opts, http.PathTemplate(pattern))
	err := c.cc.Invoke(ctx, "POST", path, in, &out, opts...)
	if err != nil {
		return nil, err
	}
	return &out, err
}
```

开启一个grpc及http服务:

```go
package main

import (
	"context"
	"fmt"
	"log"
	"test/helloworld"

	"github.com/go-kratos/kratos/v2"
	"github.com/go-kratos/kratos/v2/middleware/recovery"
	"github.com/go-kratos/kratos/v2/transport/grpc"
	"github.com/go-kratos/kratos/v2/transport/http"
)

type server struct {
	helloworld.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *helloworld.HelloRequest) (*helloworld.HelloReply, error) {
	return &helloworld.HelloReply{Msg: fmt.Sprintf("Hello %+v", in.Name)}, nil
}

func main() {
	s := &server{}
	httpSrv := http.NewServer(
		http.Address(":8000"),
		http.Middleware(
			recovery.Recovery(),
		),
	)
	grpcSrv := grpc.NewServer(
		grpc.Address(":9000"),
		grpc.Middleware(
			recovery.Recovery(),
		),
	)
    
	helloworld.RegisterGreeterServer(grpcSrv, s)
	helloworld.RegisterGreeterHTTPServer(httpSrv, s)

	app := kratos.New(
		kratos.Name("test"),
		kratos.Server(
			httpSrv,
			grpcSrv,
		),
	)

	if err := app.Run(); err != nil {
		log.Fatal(err)
	}
}
```

http client：

```go
package main

import (
	"context"
	"log"
	"test/helloworld"

	"github.com/go-kratos/kratos/v2/middleware/recovery"
	transhttp "github.com/go-kratos/kratos/v2/transport/http"
)

func main() {
	callHTTP()
}

func callHTTP() {
	conn, err := transhttp.NewClient(
		context.Background(),
		transhttp.WithMiddleware(
			recovery.Recovery(),
		),
		transhttp.WithEndpoint("127.0.0.1:8000"),
	)
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	client := helloworld.NewGreeterHTTPClient(conn)
	reply, err := client.SayHello(context.Background(), &helloworld.HelloRequest{Name: "kratos"})
	if err != nil {
		log.Fatal(err)
	}
	log.Printf("[http] SayHello %s\n", reply.Msg)
}
```

## http server端实现原理

核心流程为下图 ：

![](/images/4fe2f5ff-fbaf-462f-8fd1-1fcced329791.jpg)

首先新建一个struct 并实现 http_pb.go种 GreeterHTTPServer interface  的方法，GreeterHTTPServer的命名方式为protobuf文件中的 `service`+`HTTPServer`，interface的方法为`protobuf`中使用`google.api.http`生命http路由所有的method。

然后使用RegisterGreeterHTTPServer方法把服务注册进去。大体的流程如下：

```go
const OperationGreeterSayHello = "/helloworld.Greeter/SayHello"

func RegisterGreeterHTTPServer(s *http.Server, srv GreeterHTTPServer) {
	r := s.Route("/")
	r.POST("/helloworld", _Greeter_SayHello0_HTTP_Handler(srv)) // 注册路由
}

func _Greeter_SayHello0_HTTP_Handler(srv GreeterHTTPServer) func(ctx http.Context) error {
	return func(ctx http.Context) error {
		var in HelloRequest // protobuf 中声明的request
 		if err := ctx.Bind(&in); err != nil { // 把http的参数绑定到 in
			return err
		}
		http.SetOperation(ctx, OperationGreeterSayHello) // 设置Operation 和grpc一值，用于middleware select 等
		h := ctx.Middleware(func(ctx context.Context, req interface{}) (interface{}, error) {
			return srv.SayHello(ctx, req.(*HelloRequest)) // 这个方法也就是上文提到的GreeterHTTPServer接口的方法，也就是我们自己实现的struct server里的SayHello方法
		}) // 使用责任链模式middleware 这里没有任何中间件
		out, err := h(ctx, &in) // 执行
		if err != nil {
			return err
		}
		reply := out.(*HelloReply) 
		return ctx.Result(200, reply) 
	}
}
```

> 什么事责任链模式？
>  
> [https://haiyux.cc/post/designmode/behavioral/#责任链模式](https://haiyux.cc/post/designmode/behavioral/#%E8%B4%A3%E4%BB%BB%E9%93%BE%E6%A8%A1%E5%BC%8F)


上段代码中的POST方法为：

代码在https://github.com/go-kratos/kratos/blob/main/transport/http/router.go#L76

```go
func (r *Router) POST(path string, h HandlerFunc, m ...FilterFunc) {
	r.Handle(http.MethodPost, path, h, m...) // MethodPost = POST net/http下的常量
}

// h 为上段xxx_http_pb.go代码中_Greeter_SayHello0_HTTP_Handler的返回值
func (r *Router) Handle(method, relativePath string, h HandlerFunc, filters ...FilterFunc) {
	next := http.Handler(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		ctx := r.pool.Get().(Context)
		ctx.Reset(res, req) // 把 net/http的http.ResponseWriter 和*http.Request 设置ctx中
		if err := h(ctx); err != nil { // 执行h 
            r.srv.ene(res, req, err) // 如果出错了 执行 ene(EncodeErrorFunc)
		}
		ctx.Reset(nil, nil)
		r.pool.Put(ctx)
	}))
	next = FilterChain(filters...)(next)
	next = FilterChain(r.filters...)(next) // 添加filter 责任链模式
	r.srv.router.Handle(path.Join(r.prefix, relativePath), next).Methods(method) // router 为 mux的router 把方法注册到路由中
}
```

当我们访问 `path.Join(r.prefix, relativePath)`也就是`/helloworld` 时，会执行上段代码中的`next`方法，next是一个责任链。

核心为会执行_Greeter_SayHello0_HTTP_Handler方法，

如果没发生错误，执行`ctx.Result(200, reply)`

```go
type wrapper struct {
	router *Router
	req    *http.Request
	res    http.ResponseWriter
	w      responseWriter
}

func (c *wrapper) Result(code int, v interface{}) error {
	c.w.WriteHeader(code)
	return c.router.srv.enc(&c.w, c.req, v)
}
```

enc也就是`EncodeResponseFunc`, 为kratos预留的返回值函数

```
type EncodeResponseFunc func(http.ResponseWriter, *http.Request, interface{}) error
```

kratos提供了默认的EncodeResponseFunc

```go
func DefaultResponseEncoder(w http.ResponseWriter, r *http.Request, v interface{}) error {
	if v == nil {
		return nil
	}
	if rd, ok := v.(Redirector); ok { // 检查有无Redirect方法，如果实现了interface 为跳转路由 也就是http的301 302等
		url, code := rd.Redirect()
		http.Redirect(w, r, url, code) // 跳转
		return nil
	}
	codec, _ := CodecForRequest(r, "Accept") // 查看需要返回的参数类型 比如json
	data, err := codec.Marshal(v) // 把数据Marshal成[]byte
	if err != nil {
		return err
	}
	w.Header().Set("Content-Type", httputil.ContentType(codec.Name())) // 设置header
	_, err = w.Write(data) // 写数据
	if err != nil {
		return err
	}
	return nil
}
```

如果没发生错误，执行`ene`,也就是`EncodeErrorFunc`, 为kratos预留的错误返回值删除

```go
type EncodeErrorFunc func(http.ResponseWriter, *http.Request, error)
```

kratos提供了默认的EncodeErrorFunc

```go
func DefaultErrorEncoder(w http.ResponseWriter, r *http.Request, err error) {
	se := errors.FromError(err) // 把error变成自定义的实现error的结构体
	codec, _ := CodecForRequest(r, "Accept") // 查看需要返回的参数类型 比如json
	body, err := codec.Marshal(se)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		return
	}
	w.Header().Set("Content-Type", httputil.ContentType(codec.Name()))
	w.WriteHeader(int(se.Code)) // 写入 error中的code
	_, _ = w.Write(body) // 返回错误信息
}
```

## http client端实现原理

在上传的代码中http client的部分为

```go
type GreeterHTTPClient interface {
	SayHello(ctx context.Context, req *HelloRequest, opts ...http.CallOption) (rsp *HelloReply, err error)
}

type GreeterHTTPClientImpl struct { // 实现 GreeterHTTPClient 接口
	cc *http.Client
}

func NewGreeterHTTPClient(client *http.Client) GreeterHTTPClient {
	return &GreeterHTTPClientImpl{client}
}

func (c *GreeterHTTPClientImpl) SayHello(ctx context.Context, in *HelloRequest, opts ...http.CallOption) (*HelloReply, error) {
	var out HelloReply // 返回值
	pattern := "/helloworld" 
	path := binding.EncodeURL(pattern, in, false) // 整理path 传入in 是由于可能有path参数或者query
	opts = append(opts, http.Operation(OperationGreeterSayHello))
	opts = append(opts, http.PathTemplate(pattern))
	err := c.cc.Invoke(ctx, "POST", path, in, &out, opts...) // 访问接口
	if err != nil {
		return nil, err
	}
	return &out, err
}
```

上段代码中的Invoke方法为：

代码在https://github.com/go-kratos/kratos/blob/main/transport/http/client.go#L192

```go
func (client *Client) Invoke(ctx context.Context, method, path string, args interface{}, reply interface{}, opts ...CallOption) error {
	var (
		contentType string
		body        io.Reader
	)
	c := defaultCallInfo(path)
	for _, o := range opts {
		if err := o.before(&c); err != nil {
			return err
		}
	}
	if args != nil {
		data, err := client.opts.encoder(ctx, c.contentType, args)
		if err != nil {
			return err
		}
		contentType = c.contentType
		body = bytes.NewReader(data)
	}
	url := fmt.Sprintf("%s://%s%s", client.target.Scheme, client.target.Authority, path)
	req, err := http.NewRequest(method, url, body)
	if err != nil {
		return err
	}
	if contentType != "" {
		req.Header.Set("Content-Type", c.contentType)
	}
	if client.opts.userAgent != "" {
		req.Header.Set("User-Agent", client.opts.userAgent)
	}
	ctx = transport.NewClientContext(ctx, &Transport{
		endpoint:     client.opts.endpoint,
		reqHeader:    headerCarrier(req.Header),
		operation:    c.operation,
		request:      req,
		pathTemplate: c.pathTemplate,
	})
	return client.invoke(ctx, req, args, reply, c, opts...)
}

func (client *Client) invoke(ctx context.Context, req *http.Request, args interface{}, reply interface{}, c callInfo, opts ...CallOption) error {
	h := func(ctx context.Context, in interface{}) (interface{}, error) {
		res, err := client.do(req.WithContext(ctx))
		if res != nil {
			cs := csAttempt{res: res}
			for _, o := range opts {
				o.after(&c, &cs)
			}
		}
		if err != nil {
			return nil, err
		}
		defer res.Body.Close()
		if err := client.opts.decoder(ctx, res, reply); err != nil {
			return nil, err
		}
		return reply, nil
	}
	var p selector.Peer
	ctx = selector.NewPeerContext(ctx, &p)
	if len(client.opts.middleware) > 0 {
		h = middleware.Chain(client.opts.middleware...)(h)
	}
	_, err := h(ctx, args)
	return err
}
```
