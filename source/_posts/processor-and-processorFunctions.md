---
title: 处理器和处理器函数理解
date: 2018-12-01 18:59:14
tags: [go-web]
categories: [后端]
mathjax: true
---

> 解释处理器和处理器函数的关系
>
> 剖析`Handle`、`HandleFunc`、`Hander`、`HanderFunc`的联系

<!--more-->

## 1 处理器是什么？

Web应用的工作流程如下：

+ 客户端将请求发送到服务器的一个URL上
+ 服务器的多路复用器将接收到的请求重定向到正确的**处理器**，然后由该**处理器**对请求进行处理
+ **处理器**处理请求并执行必要的动作。
+ 处理器调用模板引擎，生成相应的HTML并且将其返回给客户端

所以简单来说处理器就是处理**请求**和执行**动作**

## 2 处理器的工作原理

### 2.1 处理请求

先来看一段代码

```go
package main

import (
	"fmt"
	"net/http"
)

type MyHandler struct{}

func (h *MyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World!")
}

func main() {
	handler := MyHandler{}
	server := http.Server{
		Addr:    "127.0.0.1:8080",
		Handler: &handler,
	}
	server.ListenAndServe()
}
```

可以看到此代码是自己编写了一个处理器`MyHandler`，然后将其直接与服务器进行了绑定，以此代替原本在使用的**默认多路复用器**。这就意味着服务器不会再通过URL匹配来将请求路由至不同的处理器，而是直接使用同一个处理器处理所有的请求，因此无论浏览器访问什么地址，服务器返回的都是`Hello World!`。

从上面代码我们可以知道要实现一个处理器我们需要实现一个`Handler`接口，其源码如下：

```go
type Handler interface {
   ServeHTTP(ResponseWriter, *Request)
}
```

### 2.2 使用多个处理器

实际情况下我们不可能使用一个处理器去处理所有请求，所以我们最常用的就是不在`Server`里面去指定处理器，而是让服务器使用**默认多路复用器**，然后使用`http.Handle`函数来将处理器和**多路复用器**绑定。

下面代码就是一个两个处理器的样例程序

```go
package main

import (
   "fmt"
   "net/http"
)

type HelloHandler struct {}

func (h *HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   fmt.Fprintf(w, "Hello!")
}

type WorldHandler struct {}

func (h *WorldHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
   fmt.Fprintf(w, "Hello!")
}

func main() {
   server := http.Server{
      Addr: "127.0.0.1:8080",
   }
   hello := HelloHandler{}
   world := WorldHandler{}
   http.Handle("/hello", &hello)
   http.Handle("/world", &world)

   server.ListenAndServe()
}
```

### 2.3  处理器函数

从2.2和2.3的代码我们可以看到如果直接写处理器，那么我们需要额外定义一个结构体，这样写起来非常麻烦，所以我们有一种比较清爽的书写方式，其代码如下：

```go
package main

import (
	"fmt"
	"net/http"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello!")
}

func world(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "World!")
}

func main() {
	server := http.Server{
		Addr: "127.0.0.1:8080",
	}
	http.HandleFunc("/hello", hello)
	http.HandleFunc("/world", world)

	server.ListenAndServe()
}
```

可以看到上面代码的`hello`和`world`函数和之前代码的`ServeHTTP`方法有着相同的签名，然后在main函数中通过`http.HandleFunc`函数将`hello`和`world`转换成`Handler`并和多路复用器绑定。

要分析其实现原理，我们要先看`http.Handle`和`http.HandleFunc`的源码：

首先来看`http.Handle`的源码：

```go
func Handle(pattern string, handler Handler) { 
    DefaultServeMux.Handle(pattern, handler) 
}
```

可以看到他第二个参数就是处理器`Handler`，我们发现他其实调用的是默认多路复用器`DefaultServeMux`的`Handle`方法，后者的`Handle`只是多了一些处理细节这里不再赘述。

那么接下来看`http.HandleFunc`的源码：

```go
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

可以看到上述代码其实第二个参数是一个`Handler`处理器，但是其实实参是一个相同签名的其他函数，那么这个转换是通过什么进行的呢？

我们发现这里也是调用的是默认多路复用器`DefaultServeMux`的`HandleFunc`方法，其源码如下：

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	if handler == nil {
		panic("http: nil handler")
	}
	mux.Handle(pattern, HandlerFunc(handler))
}
```

通过观察源码答案就很清晰了，是`HandlerFunc`进行了转换工作。

我们来看一下`HandlerFunc`的源码：

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

看完上述代码我们可以明白其实将hello函数转换成处理器的实际代码如下：

```go
HelloHandler := HandlerFunc(hello)
```

看到这里想必各位已经明白，对于和`ServeHTTP`有相同签名的函数是如何转换成处理器`Handler`的了。

+ PS：虽然处理器函数的代码更加整洁，但是处理器函数不能完全替代处理器。这是因为在某些情况下，代码可能已经包含了某个接口或者某种类型，这时我们只需要为它们添加`ServeHTTP`方法就可以将它们转变成处理器。

### 2.4 串联多个处理器和处理器函数

有了上面的知识铺垫，现在我们来深入探讨一下串联多个处理器（处理器函数）的做法。

因为在GO语言中程序可以将一个函数传递给另一个函数，所以我们可以实现这个功能。

#### 2.4.1 串联多个处理器函数

直接用例子上手，假设现在要将`hello`、`world`串联起来，其代码如下：

```go
package main

import (
	"fmt"
	"net/http"
	"reflect"
	"runtime"
)

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello!\n")
}

func world(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "world!\n")
		h(w, r)
	}
}

func main() {
	server := http.Server{
		Addr: "127.0.0.1:8080",
	}
	http.HandleFunc("/hello", world(hello))
	server.ListenAndServe()
}
```

如果要再增加一个`funny`函数，其函数原型如下：

```go
func funny(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		...
		h(w, r)
	}
}
```

那么只需将`http.HandleFunc`修改为如下：

```go
http.HandleFunc("/hello", funny(world(hello)))
```

#### 2.4.2 串联多个处理器

同样是串联`hello`、`world`、`funny`，代码如下：

```go
package main

import (
	"fmt"
	"net/http"
)

type HelloHandler struct{}

func (h HelloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello!\n")
}

func world(h http.Handler) http.Handler {
	return http.HandlerFunc (func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "world!\n")
		h.ServeHTTP(w, r)
	})
}

func funny(h http.Handler) http.Handler {
	return http.HandlerFunc (func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprintf(w, "funny!\n")
		h.ServeHTTP(w, r)
	})
}

func main() {
	server := http.Server{
		Addr: "127.0.0.1:8080",
	}
	hello := HelloHandler{}
	http.Handle("/hello", funny(world(hello)))
	server.ListenAndServe()
}
```
参考资料：[Go Web Programming](https://livebook.manning.com/#!/book/go-web-programming/chapter-3/)

