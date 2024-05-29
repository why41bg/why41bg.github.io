---
title: 学习Go中net/http框架笔记系列：函数转为结构体
date: 2024-05-30 00:03:15
tags:
- GoLang
- net/http
categories:
---

net/http 是 Go 中用于快速构建 Web 应用的标准库，使用 net/http 构建 server 的核心在于注册路由，即创建用户请求与具体的处理逻辑之间的映射关系（很像操作系统中系统调用和内核程序的关系）。下面是一个注册路由的简单的 demo。

```go
package main

import (
	"fmt"
	"net/http"
)

// indexHandler 就是具体的处理函数
func indexHandler(w http.ResponseWriter, r *http.Request) {
	_, err := fmt.Fprintf(w, "hello world")
	if err != nil {
		return
	}
}

func main() {
	http.HandleFunc("/", indexHandler)
	err := http.ListenAndServe(":8000", nil)
	if err != nil {
		return
	}
}

```



# http.HandleFunc

```go
http.HandleFunc("/", indexHandler)
```

上面这句代码绑定了一个路由关系，所有 `/` 请求都将路由到我们定义好的 `indexHandler` 函数进行处理。`HandleFunc` 源码如下：

```go
// HandleFunc registers the handler function for the given pattern in [DefaultServeMux].
// The documentation for [ServeMux] explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if use121 {
		DefaultServeMux.mux121.handleFunc(pattern, handler)
	} else {
		DefaultServeMux.register(pattern, HandlerFunc(handler))
	}
}
```

`HandleFunc` 函数接收两个参数，一个是请求路径，另一个是对应的处理函数。在 Go 语言中，函数是一等公民，可以被作为值来传递。这里的 `handler` 就是一个函数值，它可以被传递给 `HandleFunc` 函数，并在 HandleFunc 函数内部被调用。



# 继续跟进

跟进 `handlerFunc` 函数，源码如下：

```go
// Formerly ServeMux.HandleFunc.
func (mux *serveMux121) handleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.handle(pattern, HandlerFunc(handler))
}
```

继续跟进 `mux.handle` 函数，源码如下：

```go
// Formerly ServeMux.Handle.
func (mux *serveMux121) handle(pattern string, handler Handler) {
	// ...
}
```

最终调用了 `serveMux121` 结构体中的 `handle` 方法，该结构体保存了路由关系，该函数就用于注册路由关系，可以看到，函数接收两个参数，一个是请求 `pattern`，第二个是 `Handler` 对象。**关键点来了**，在整个注册流程中，我们传递的始终都是一个请求路径和对应的处理逻辑函数，可是最后需要的是一个对象。那么到底在哪一步开始，我们传递的函数变成了对象呢？



# 函数转化为对象

```go
mux.handle(pattern, HandlerFunc(handler))
```

这句代码实质上就是对 `handler` 进行了类型强转，将其从一个函数类型转变为了一个 `Handler` 类型。那么继续来看一下和 `HandlerFunc` 有关的源码吧。

```go
// The HandlerFunc type is an adapter to allow the use of
// ordinary functions as HTTP handlers. If f is a function
// with the appropriate signature, HandlerFunc(f) is a
// [Handler] that calls f.
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

对这部分源码的解释如下：`HandlerFunc` 被定义为一个函数类型，这个函数接受两个参数，一个是 `ResponseWriter` 类型，另一个是指向 `Request` 类型的指针，并且这个函数没有返回值。  这样的定义允许将函数作为值来使用，可以将它们存储在变量中，作为参数传递，或者从其他函数中返回。这为编写可以接受用户定义行为的通用代码提供了可能，因为用户可以通过定义和传递符合特定签名的函数来定制这些行为。

`(f HandlerFunc)` 是一个接收器，它将 `ServeHTTP` 方法绑定到 `HandlerFunc` 类型上。这样，任何 `HandlerFunc` 类型的变量就可以调用 `ServeHTTP` 方法。又因为 `HandlerFunc` 实现了 `ServeHTTP` 方法，因此它就成为了 `Handler` 的一个对象，`Handler` 定义的源码如下：

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```



# 总结

总的来说，`HandlerFunc` 是一个类型，只不过表示的是一个具有 `func(ResponseWriter, *Request)` 签名的函数类型，并且这种类型实现了 `ServeHTTP` 方法（在 `ServeHTTP` 方法中又调用了自身），也就是说这个类型的函数其实就是一个 `Handler` 类型的对象。利用这种类型转换，就可以将一个 `handler` 函数转换为一个 `Handler` 对象，而不需要定义一个结构体，再让这个结构实现 `ServeHTTP` 方法。

