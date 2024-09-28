---
title: 并发编程之Select(二)
categories: golang
comments: true
---

# 并发编程之Select(二)

# 简介

处理一个管道的数据我们可以通过`<-`取值，或者通过`for...range`来进行数据的获取，但是当我们需要同时多个管道数据的发送或者接收的操作的时候，我们可以通过`select`关键字来进行操作。`select`代码块和`switch`有些相似，也包含了多个case分支，但是case的执行并不会按照顺序，当有一个case满足条件的时候，那么就会执行满足条件的case，如果case都不满足，那么就会执行default下的内容，倘若没有default，那么select就会阻塞，直到有case可以满足

<!--more-->

## 方法

### 基本使用

```go
package main

import "fmt"

func main() {

	c := make(chan int)

	go func() {
		c <- 1
	}()
	select {
	case i := <-c:
		fmt.Println(i)
	}
	fmt.Println("done")
}
```

### for-select

```go
package main

import (
   "fmt"
   "time"
)

func main() {

   c := make(chan int)

   go func() {
      // defer close(c)
      for i := 0; i < 10; i++ {
         c <- i
      }
   }()

   for {
      select {
      case i := <-c:
         fmt.Println(i)
         time.Sleep(1 * time.Second)
      default:
         fmt.Println("no message from channel")
         time.Sleep(1 * time.Second)

      }
   }

}
```

for-select语句可以让我们持续对channel进行监听

**退出循环**

为了退出for-select的循环，我们可以使用`goto`语句来进行

```go
package main

import (
   "fmt"
   "time"
)

func main() {

   c := make(chan int)
   done := make(chan struct{})
   go func() {
      defer close(c)
      for i := 0; i < 10; i++ {
         c <- i
      }
   }()

   go func() {
      defer close(done)
   loop:
      for {
         select {
         case i, ok := <-c:
            if !ok {
               break loop
            }
            fmt.Println(i)
            time.Sleep(1 * time.Second)
         default:
            fmt.Println("no message from channel")
            time.Sleep(1 * time.Second)

         }
      }
   }()

   <-done

   fmt.Println("done")

}
```