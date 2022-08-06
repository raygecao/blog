---
weight: 20
title: "Gin使用NoRoute实现默认路由踩坑总结"
subtitle: ""
date: 2022-08-06T12:34:25+08:00
lastmod: 2022-08-06T12:34:25+08:00
draft: false
author: "raygecao"
authorLink: ""
description: ""
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["go"]
categories: ["踩坑记录"]


toc:
  auto: false
lightgallery: true
license: ""
---
gin路由注册要求是互斥的，`engine.NoRoute`可以提供默认路由的能力，
使用gin NoRoute对grpc gateway进行路由时却发生了正常返回结果但伴有`404`status code的问题。
<!--more-->

## 背景

某存储服务实现了两组grpc-gateway分别对外提供artifact服务与package服务，并使用Gin框架进行路由。

```go
func main() {
	router := gin.Default()
  mux := runtime.NewServeMux()
  
  // 注册artifact与pkg ServiceHandler到mux上
  // ...
  
  // 匹配/v1/*的path路由到mux中
	router.Any("/v1/*path", gin.WrapH(mux))
	if err := router.Run(*flagListen); err != nil {
		panic(err)
	}
}
```

## 需求

现需要为此服务新增openapi路由，并且同样具有/v1 prefix，路由表如下所示
{{< image src="tree.png" caption="route tree" width=600 height=200 >}}

## 实现方案

最初的想法是在最开头加上相应的route

```go
// ...
router.GET("/v1/swagger/*path", swaggerHandler)
router.Any("/v1/*path", gin.WrapH(mux))
//...
```

运行时报如下错误

```shell
panic: catch-all conflicts with existing handle for the path segment root in path '/v1/*path'
```

gin的router在匹配路由时并不像三层路由表按定义的顺序一层一层匹配，而注册一组互斥的(method, pattern)用以全局匹配。而上述两个pattern是存在包含关系的，因此报错path conflict。由于mux内的二级路由比较多样，因此我们希望通过默认路由对mux进行路由改造。gin提供了`NoRoute`方法去处理匹配不上的路由，这正好可以满足此需求

```go
// ...
router.GET("/v1/swagger/*path", swaggerHandler)
router.NoRoute(gin.WrapH(mux))
//...
```

## 问题描述

经过本地测试，服务可以正常返回，但上线时UI并不能正常工作。进一步测试时发现虽然api返回了正确结果，但是status code返回`404`，所以client无法正常工作。

```shell
$ curl -v localhost:3001/v1/artifacts

# 省略Request输出

< HTTP/1.1 404 Not Found
< Content-Type: application/json
< Date: Fri, 05 Aug 2022 06:04:45 GMT
< Content-Length: 26
<
* Connection #0 to host localhost left intact
{"artifacts":[],"total":0}%
```

## 问题排查

上述请求返回了正确结果，因此可以确定其已经被artifact handler的逻辑正确处理，显然status code被gin置成了`404`。`NoRoute`的doc如此描述

```go
// NoRoute adds handlers for NoRoute. It return a 404 code by default.
func (engine *Engine) NoRoute(handlers ...HandlerFunc) {
	engine.noRoute = handlers
	engine.rebuild404Handlers()
}
```

一个合理的猜测是gin默认使用`200`作为response status code，`NoRoute`在处理匹配不上的路由同时，会将默认status code置为`404`。

为了更深入探索此问题，做了如下三个实验

- Case1：将`/v1/artifacts` 返回结果改造成internal error，观察status code
- Case2：给mux添加一个自定义路由`/v1/artifacts/success`（裸handler），总是返回`200`
- Case3：给mux添加一个自定义路由`/v1/artifacts/fail`，总是返回`500`

```go
// case1
func (m *Manager) ListArtifacts(ctx context.Context, request *v1.ListArtifactsRequest) (*v1.ListArtifactsResponse, error) {
  // /v1/artifacts handler
	return nil, fmt.Errorf("fail to list")
}

// case2
	mux.HandlePath("GET", "/v1/artifacts/success", func(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
		w.WriteHeader(http.StatusOK)
	})

// case3
	mux.HandlePath("GET", "/v1/artifacts/fail", func(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
		w.WriteHeader(http.StatusInternalServerError)
	})
```

上述请求的resp status code都是符合预期的

```shell
# case1
$ curl -v localhost:3001/v1/artifacts

< HTTP/1.1 500 Internal Server Error
< Content-Type: application/json
< Date: Fri, 05 Aug 2022 06:19:14 GMT
< Content-Length: 50
<
* Connection #0 to host localhost left intact
{"code":2, "message":"fail to list", "details":[]}%

# case2
$ curl -v localhost:3001/v1/artifacts/success

< HTTP/1.1 200 OK
< Date: Fri, 05 Aug 2022 06:20:55 GMT
< Content-Length: 0
<
* Connection #0 to host localhost left intact

# case3
$ curl -v localhost:3001/v1/artifacts/fail

< HTTP/1.1 500 Internal Server Error
< Date: Fri, 05 Aug 2022 06:21:24 GMT
< Content-Length: 0
<
* Connection #0 to host localhost left intact
```

上述现象基本表明在grpc ServiceHanlder中，如果处理正确，并不会显式向resp写`200`，但出错时会显式写特定的status code。

## 源码追溯

### gin侧

- 查看`NoRoute`配置的handler

  ```go
  func (engine *Engine) rebuild404Handlers() {
  	engine.allNoRoute = engine.combineHandlers(engine.noRoute)
  }
  ```

- 从名字来看，`allNoRoute`像是当匹配不到路由时会执行的handler，我们搜一下其引用位置

  ```go
  func (engine *Engine) handleHTTPRequest(c *Context) {
    // 省略匹配路由的逻辑
    // 省略HandleMethodNotAllowed处理逻辑
    // ...
  	c.handlers = engine.allNoRoute
  	serveError(c, http.StatusNotFound, default404Body)
  }
  
  // ServeHTTP conforms to the http.Handler interface.
  func (engine *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
  	c := engine.pool.Get().(*Context)
  	c.writermem.reset(w)
  	c.Request = req
  	c.reset() // 置默认status code
  
  	engine.handleHTTPRequest(c)
  
  	engine.pool.Put(c)
  }
  
  func (w *responseWriter) reset(writer http.ResponseWriter) {
  	w.ResponseWriter = writer
  	w.size = noWritten
  	w.status = defaultStatus // 200
  }
  
  const (
  	noWritten     = -1
  	defaultStatus = http.StatusOK
  )
  ```

- `ServeHTTP`调用`handleHTTPRequest`，并且将默认status code设置为`200`，路由匹配失败会进入到`serveError`函数里

  ```go
  func serveError(c *Context, code int, defaultMessage []byte) {
  	// 默认status改成了404
    c.writermem.status = code
    // 执行handler
  	c.Next()
  	if c.writermem.Written() {
  		return
  	}
  	// ...
  }
  
  // Next should be used only inside middleware.
  // It executes the pending handlers in the chain inside the calling handler.
  // See example in GitHub.
  func (c *Context) Next() {
  	c.index++
  	for c.index < int8(len(c.handlers)) {
      // 调用注册的handler
  		c.handlers[c.index](c)
  		c.index++
  	}
  }
  ```

- 很明显这里将默认status code改成`404`，并去执行相应的handler，正常情况下handler会覆写resp里的status code，而grpc ServiceHandler看上去在执行成功后没有写`200`

### Grpc-gateway侧

- 查看注册handler的逻辑，可以看到当handler返回错误后，会去调用`runtime.HTTPError`，而正确处理时调用`runtime.ForwardResponseMessage`

  ```go
  func RegisterArtifactServiceHandlerServer(ctx context.Context, mux *runtime.ServeMux, server ArtifactServiceServer) error {
  
  	mux.Handle("GET", pattern_ArtifactService_ListArtifacts_0, func(w http.ResponseWriter, req *http.Request, pathParams map[string]string) {
  
      // ...
  		if err != nil {
  			runtime.HTTPError(ctx, mux, outboundMarshaler, w, req, err)
  			return
  		}
      // 调用ListArtifactHandler
  		resp, md, err := local_request_ArtifactService_ListArtifacts_0(ctx, inboundMarshaler, server, req, pathParams)
      // 省略处理metadata逻辑
  		if err != nil {
  			runtime.HTTPError(ctx, mux, outboundMarshaler, w, req, err)
  			return
  		}
  		// 封装成http resp
  		forward_ArtifactService_ListArtifacts_0(ctx, mux, outboundMarshaler, w, req, resp, mux.GetForwardResponseOptions()...)
  
  	})
  }
  
  forward_ArtifactService_ListArtifacts_0 = runtime.ForwardResponseMessage
  ```

- `HTTPError`将grpc code与http status code做了转换后，直接写到resp里，这解释了为何错误的状态码可以正常返回。

  ```go
  // DefaultHTTPErrorHandler is the default error handler.
  // If "err" is a gRPC Status, the function replies with the status code mapped by HTTPStatusFromCode.
  // If "err" is a HTTPStatusError, the function replies with the status code provide by that struct. This is
  // intended to allow passing through of specific statuses via the function set via WithRoutingErrorHandler
  // for the ServeMux constructor to handle edge cases which the standard mappings in HTTPStatusFromCode
  // are insufficient for.
  // If otherwise, it replies with http.StatusInternalServerError.
  //
  // The response body written by this function is a Status message marshaled by the Marshaler.
  func DefaultHTTPErrorHandler(ctx context.Context, mux *ServeMux, marshaler Marshaler, w http.ResponseWriter, r *http.Request, err error) {
    // ...
    // grpc code向http code转换
   	st := HTTPStatusFromCode(s.Code())
  	if customStatus != nil {
  		st = customStatus.HTTPStatus
  	}
  	// 显式写入status code
  	w.WriteHeader(st)
  	if _, err := w.Write(buf); err != nil {
  		grpclog.Infof("Failed to write response: %v", err)
  	}
  // ...
  }
  ```

- 最后看一看handler正确处理后，封装resp相关逻辑，只将handler返回的结果序列化写到`resp.Body`中，并没有显式写status code。

  ```go
  // ForwardResponseMessage forwards the message "resp" from gRPC server to REST client.
  func ForwardResponseMessage(ctx context.Context, mux *ServeMux, marshaler Marshaler, w http.ResponseWriter, req *http.Request, resp proto.Message, opts ...func(context.Context, http.ResponseWriter, proto.Message) error) {
   // ...
  	if err := handleForwardResponseOptions(ctx, w, resp, opts); err != nil {
  		HTTPError(ctx, mux, marshaler, w, req, err)
  		return
  	}
  	var buf []byte
  	var err error
  	if rb, ok := resp.(responseBody); ok {
  		buf, err = marshaler.Marshal(rb.XXX_ResponseBody())
  	} else {
  		buf, err = marshaler.Marshal(resp)
  	}
  	if err != nil {
  		grpclog.Infof("Marshal error: %v", err)
  		HTTPError(ctx, mux, marshaler, w, req, err)
  		return
  	}
    // 只marshal了结果写入了resp body，没有显式写status code
  	if _, err = w.Write(buf); err != nil {
  		grpclog.Infof("Failed to write response: %v", err)
  	}
    // ... 
  }
  ```

- 这时需要考虑是否有地方可以通过options来做一些trick，从上述代码我们可以看到`handleForwardResponseOptions`传入了resp，猜测这里面有文章

  ```go
  func handleForwardResponseOptions(ctx context.Context, w http.ResponseWriter, resp proto.Message, opts []func(context.Context, http.ResponseWriter, proto.Message) error) error {
  	if len(opts) == 0 {
  		return nil
  	}
    // 这里的Opts实际上是mux.GetForwardResponseOptions()...
    // 可以在创建mux时通过runtime.WithForwardResponseOption自定义
  	for _, opt := range opts {
  		if err := opt(ctx, w, resp); err != nil {
  			grpclog.Infof("Error handling ForwardResponseOptions: %v", err)
  			return err
  		}
  	}
  	return nil
  }
  ```

- 因为执行到`ForwardResponseMessage`一定意味着service handler执行成功了，显然可以通过`ForwardResoponseOptions`将200写到resp status code来解决此问题

## 解决方法

创建mux时指定`WithForwardResponseOption`向resp中写入`200`status code

```go
	// ...
  //mux := runtime.NewServeMux()
	mux := runtime.NewServeMux(runtime.WithForwardResponseOption(func(ctx context.Context, writer http.ResponseWriter, message proto.Message) error {
		writer.WriteHeader(http.StatusOK)
		return nil
	}))
  // ... 
```

测试一切正常

## 后记

有小伙伴指出在`http.ResposeWriter`接口中的`Write`方法里有个重要说明：**If WriteHeader has not yet been called, Write calls WriteHeader(http.StatusOK) before writing the data**。这表明在没有显式写状态码时，调用`Write`前会写个200

```go
// A ResponseWriter interface is used by an HTTP handler to
// construct an HTTP response.
//
// A ResponseWriter may not be used after the Handler.ServeHTTP method
// has returned.
type ResponseWriter interface {
	// Header returns the header map that will be sent by
	// WriteHeader. The Header map also is the mechanism with which
	// Handlers can set HTTP trailers.
	//
	// Changing the header map after a call to WriteHeader (or
	// Write) has no effect unless the modified headers are
	// trailers.
	//
	// There are two ways to set Trailers. The preferred way is to
	// predeclare in the headers which trailers you will later
	// send by setting the "Trailer" header to the names of the
	// trailer keys which will come later. In this case, those
	// keys of the Header map are treated as if they were
	// trailers. See the example. The second way, for trailer
	// keys not known to the Handler until after the first Write,
	// is to prefix the Header map keys with the TrailerPrefix
	// constant value. See TrailerPrefix.
	//
	// To suppress automatic response headers (such as "Date"), set
	// their value to nil.
	Header() Header

	// Write writes the data to the connection as part of an HTTP reply.
	//
	// If WriteHeader has not yet been called, Write calls
	// WriteHeader(http.StatusOK) before writing the data. If the Header
	// does not contain a Content-Type line, Write adds a Content-Type set
	// to the result of passing the initial 512 bytes of written data to
	// DetectContentType. Additionally, if the total size of all written
	// data is under a few KB and there are no Flush calls, the
	// Content-Length header is added automatically.
	//
	// Depending on the HTTP protocol version and the client, calling
	// Write or WriteHeader may prevent future reads on the
	// Request.Body. For HTTP/1.x requests, handlers should read any
	// needed request body data before writing the response. Once the
	// headers have been flushed (due to either an explicit Flusher.Flush
	// call or writing enough data to trigger a flush), the request body
	// may be unavailable. For HTTP/2 requests, the Go HTTP server permits
	// handlers to continue to read the request body while concurrently
	// writing the response. However, such behavior may not be supported
	// by all HTTP/2 clients. Handlers should read before writing if
	// possible to maximize compatibility.
	Write([]byte) (int, error)

	// WriteHeader sends an HTTP response header with the provided
	// status code.
	//
	// If WriteHeader is not called explicitly, the first call to Write
	// will trigger an implicit WriteHeader(http.StatusOK).
	// Thus explicit calls to WriteHeader are mainly used to
	// send error codes.
	//
	// The provided code must be a valid HTTP 1xx-5xx status code.
	// Only one header may be written. Go does not currently
	// support sending user-defined 1xx informational headers,
	// with the exception of 100-continue response header that the
	// Server sends automatically when the Request.Body is read.
	WriteHeader(statusCode int)
}
```

这看似与上述排查结论相悖。但进一步看，这个结构是个nterface，在interface内声明这个规则并不能强制约束实现方去如此做。恰好gin `Context`中的相关接口实现恰好打破了这个规则

```go
func (w *responseWriter) Write(data []byte) (n int, err error) {
	// 装饰了一层写特定状态码的逻辑
  w.WriteHeaderNow()
	n, err = w.ResponseWriter.Write(data)
	w.size += n
	return
}

func (w *responseWriter) WriteHeaderNow() {
  // 如果之前没Write过，则将保存的状态码写到resp里
	if !w.Written() {
		w.size = 0
		w.ResponseWriter.WriteHeader(w.status)
	}
}

func (w *responseWriter) WriteHeader(code int) {
	if code > 0 && w.status != code {
		if w.Written() {
			debugPrint("[WARNING] Headers were already written. Wanted to override status code %d with %d", w.status, code)
		}
    // 仅更新内部状态码，待Write data时，将内部状态码直接写到resp里
		w.status = code
	}
}
```

gin 对`Write`装饰了一层写内部状态码的行为，用以对默认状态码的支持。
