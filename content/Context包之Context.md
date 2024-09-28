---
title: Context包之Context
categories: golang
comments: true
---

# Context包之Context

# 简介

Context意思为"上下文",主要的作用是用来在goroutine之间传递上下文的信息。上下文包含的功能有：

* k-v
* 取消信号
* 超时时间
* 截止时间

上下文是可以进行传播的，也就是说通过调用一些方法可以由父级派生出子级

<!--more-->

## 方法

### func Background() Context

```go
ctx:=context.Background()
```

该方法返回一个非零的空上下文。它永远不会被取消，没有价值，也没有截止日期。它通常由 main 函数、初始化和测试使用，并作为传入请求的顶级上下文。

### func TODO() Context

```go
ctx:=context.TODO()
```

该方法返回一个非零的空上下文。如果实在不清楚需要传递什么，可以使用该方法

### func WithValue(parent Context, key, val any) Context

**func WithValue(parent Context, key, val interface{}) Context 1.18以前的版本函数签名是这样的** 

该方法会返回一个context的副本，返回的ctx中会携带key，val的信息

<u>The provided key must be comparable and should not be of type string or any other built-in type to avoid collisions between packages using context. Users of WithValue should define their own types for keys. To avoid allocating when assigning to an interface{}, context keys often have concrete type struct{}. Alternatively, exported context key variables' static type should be a pointer or interface.</u>

```go
type contextKey string
ctx:=context.Background()
k:=contextKey("Ethan")
ctx = context.WithValue(ctx,k,"Leo")
```

#### 示例

```go
package main

import (
   "context"
   "fmt"
)

type contextKey string

func main() {
   k := contextKey("Ethan")
   ctx := context.Background()
   f(ctx)
   ctx = context.WithValue(ctx, k, "Leo")
   f(ctx)
}

func f(ctx context.Context) {
   if value := ctx.Value(contextKey("Ethan")); value != nil {
      fmt.Printf("value is %v\n", value)
      return
   }

   fmt.Println("no value")

}
```

### func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)

该方法会返回一个context副本，和一个取消函数，参数timeout的设置，会让ctx在指定的时间段时候，接收到ctx.Done()的信号，用来关闭context

```go
package main

import (
   "context"
   "fmt"
   "time"
)

type contextKey string

var c chan struct{}

func main() {
   c = make(chan struct{})
   ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
   defer cancel()
   go f(ctx)
   time.Sleep(1 * time.Second)
   close(c)
   time.Sleep(1 * time.Second)

}

func f(ctx context.Context) {
   select {
   case <-c:
      fmt.Println("main goroutine is done")
   case <-ctx.Done():
      fmt.Println("context is time out")
   }

}
```

### func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)

WithDeadline 返回父上下文的副本，截止日期调整为不晚于 d。如果父级的截止日期已经早于 d，则 WithDeadline(parent, d) 在语义上等同于父级。当截止日期到期、调用返回的取消函数或父上下文的 Done 通道关闭时，返回的上下文的 Done 通道将关闭，以先发生者为准



## 示例

### listening_cancel

```go
package main

import (
   "fmt"
   "net/http"
   "os"
   "time"
)

func main() {
   http.ListenAndServe(":8081", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
      ctx := r.Context()

      select {
      case <-time.After(3 * time.Second):
         // 2秒后要是收到信号
         // 说明请求被执行
         w.Write([]byte("请求被执行"))
      case <-ctx.Done():
         // 如果请求被取消 输出请求被取消
         fmt.Fprint(os.Stderr, "请求被取消\n")
         fmt.Println(ctx.Err())
      }
   }))
}
```

### withCancel

```go
package main

import (
   "context"
   "fmt"
   "sync"
   "time"
)

func main() {
   ctx, cancel := context.WithCancel(context.Background())
   wg := sync.WaitGroup{}
   wg.Add(1)
   go process(ctx, &wg)

   time.Sleep(3 * time.Second)
   cancel()
   wg.Wait()

}

func process(ctx context.Context, wg *sync.WaitGroup) {
   defer wg.Done()
loop:
   for {
      select {
      case <-ctx.Done():
         fmt.Println("main goroutine done")
         break loop
      default:
         fmt.Println("do some work")
         time.Sleep(1 * time.Second)
      }
   }
}
```

### withDeadLine

```go
package main

import (
   "context"
   "net/http"
   "time"
)

type ContextKey string

func main() {

   http.ListenAndServe(":8083", http.HandlerFunc(handler))

}

func handler(w http.ResponseWriter, r *http.Request) {
   deadline := time.Now().Add(10 * time.Second)
   ctx, cancel := context.WithDeadline(r.Context(), deadline)
   defer cancel()
   c := make(chan string)

   go readFile(c)
   go getDb(c)

   select {
   case <-ctx.Done():
      w.Write([]byte(ctx.Err().Error()))
   case data := <-c:
      w.Write([]byte(data))

   }

}

func readFile(c chan<- string) {
   time.Sleep(11 * time.Second)
   c <- "data from file"

}

func getDb(c chan<- string) {
   time.Sleep(13 * time.Second)
   c <- "data from db"
}
```

### withTimeout

```go
package main

import (
   "context"
   "net/http"
   "time"
)

type ContextKey string

func main() {

   http.ListenAndServe(":8082", http.HandlerFunc(handler))

}

func handler(w http.ResponseWriter, r *http.Request) {
   ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
   ctx = context.WithValue(ctx, ContextKey("Ethan"), "Hi Leo")
   defer cancel()
   c := make(chan string)
   go func() {
      getFromDb(ctx, c)
   }()

   select {
   case <-ctx.Done():
      w.Write([]byte(ctx.Err().Error()))
   case data := <-c:
      w.Write([]byte(data))

   }

}

func getFromDb(ctx context.Context, c chan<- string) {
   time.Sleep(6 * time.Second)
   value := ctx.Value(ContextKey("Ethan"))
   v, ok := value.(string)
   if !ok {
      c <- "get data error"
      return
   }
   c <- v
}
```

### memory

```go
package main

import (
   "context"
   "fmt"
   "time"
)

func main() {
   ctx, cancel := context.WithCancel(context.Background())

   go process(ctx)

   time.Sleep(5 * time.Second)
   cancel()
   time.Sleep(2 * time.Second)
}

func process(ctx context.Context) {

   for {
      select {
      case <-ctx.Done():
         fmt.Println("main goroutine done ")
         return
      default:
         fmt.Println("do some work...")
         time.Sleep(time.Second)
      }
   }
}
```