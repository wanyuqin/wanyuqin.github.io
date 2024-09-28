---
title: Sync包之Sync.Cond
 
categories: golang
comments: true
---

# Sync包之Sync.Cond

## 简介

简单来说，sync.Cond实现了一个广播和单个通知的功能，主要还是作用于有多个goroutine的情况下，通过一个变量来对两种不同类型的事件进行控制。但是在go语言中，我们更加推荐通过channel来进行多个协程的通信

<!--more-->



## 方法

### fun NewCond(l Locker)*Cond

该方法会返回一个新的cond，在创建cond的时候，我们需要传入一把锁

```go
func main(){
 m :=&sync.Mutex{}
 c := sync.NewCond(m)
}
```

## func (*Cond) Broadcast()

广播，通过广播的方式唤醒所有的goroutine

```go
c.Broadcast()
```

### func (c *Cond) Signal()

唤醒任意一个等待的goroutine

```go
c.Signal()
```

### func (c *Cond) Wait()

```go
c.Wait()
```

调用c.wait()会自动释放c.L，并且挂起调用的goroutine,所以goroutine会阻塞在 Wait 方法调用的地方。如果其他协程调用了 Signal 或 Broadcast 唤醒了该协程，那么 Wait 方法在结束阻塞时，会重新给 c.L 加锁，并且继续执行 Wait 后面的代码。

### 示例

```go
package main

import (
   "fmt"
   "sync"
   "time"
)

var start = false
var done = false

func main() {
   wg := sync.WaitGroup{}
   wg.Add(10)
   m := &sync.Mutex{}
   c := sync.NewCond(m)
   for i := 0; i < 10; i++ {
      go Soldiers(i, c, &wg)
   }
   go Waiter(c)
   Officer(c)
   wg.Wait()
   fmt.Println("所有大兵干完饭")

   CleanUp(c)
   time.Sleep(3 * time.Second)
   fmt.Println("打扫结束")

}

func CleanUp(c *sync.Cond) {
   c.L.Lock()
   done = true
   c.L.Unlock()
   c.Signal()
}

func Officer(c *sync.Cond) {
   fmt.Printf("长官准备中....\n")
   time.Sleep(time.Second * 5)
   c.L.Lock()
   start = true
   c.L.Unlock()
   // 唤醒所有的等待线程
   c.Broadcast()

}

func Soldiers(i int, c *sync.Cond, wg *sync.WaitGroup) {
   defer wg.Done()
   c.L.Lock()
   fmt.Printf("大兵%d号等待干饭....\n", i)
   for !start {
      // wait 解锁lock 同时挂起goroutine
      // 当wait的goroutine被唤醒的时候，会重新将锁加上
      c.Wait()
   }
   fmt.Printf("大兵%d号开始干饭....\n", i)
   c.L.Unlock()

}

func Waiter(c *sync.Cond) {
   c.L.Lock()
   for !done {
      c.Wait()
   }
   fmt.Println("用餐结束，开始打扫....")
   c.L.Unlock()
   return
}
```

