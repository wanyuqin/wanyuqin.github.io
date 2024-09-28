---
title: 并发编程之Channel(一)
 
categories: golang
comments: true
---

# 并发编程之Channel(一)

**不要通过共享内存通信，要通过通信共享内存**

# 简介

channel(通道),golang并发编程中的一种重要的数据结构，用于多个goroutine之间进行通信。

<!--more-->

我们通常可以把channel想象成一个传送带，将goroutine想象成为传送带周边的人，一个人送传送带的上游放上物品，通过传送带的传输下游的人可以将物品取走

<img src="https://img.ethanleo.top/uPic/image-20221126105842690.png" alt="image-20221126105842690" style="zoom:50%;" />

<!--more-->

## channel的类型

对于channel来说，我们可以创建两种类型的channel，一种是**无缓冲区的channel**,一种是**有缓冲区的channel**

### 无缓冲区的channel

对于无缓冲区的channel来说，channel本身是不存放数据的，整个channel的发送和接收都会被阻塞的。

<img src="https://img.ethanleo.top/uPic/image-20221126145330161.png" alt="image-20221126145330161" style="zoom:50%;" />

#### 创建

```go
var c chan interface{}
c = make(chan interface{})
```

或

```go
c:=make(chan interface{})
```

### 带缓冲区的channel

带缓冲区的channel是具有暂存数据的功能，当channel的缓冲区满了或者空的时候，channel的发送和接收会被阻塞

<img src="https://img.ethanleo.top/uPic/image-20221126145800931.png" alt="image-20221126145800931" style="zoom:50%;" />

#### 创建

```go
var c chan interface{}
c = make(chan interface{},1)
```

或

```go
c:=make(chan interface{},1)
```

我们这里可以通过内置函数make去示例化一个channel,与无缓冲区的channel不同的是，创建带有缓冲区的channel，可以在make函数后添加一个参数，来指定缓冲区的大小

#### 对比

```go
package main

import (
   "fmt"
   "time"
)

func main() {

   c := make(chan string)
   s := []string{"A", "B", "C", "D"}
   go func() {
      defer close(c)
      for _, v := range s {
         fmt.Printf("send %s\n", v)
         c <- v
         time.Sleep(1 * time.Second)
      }
   }()

   for v := range c {
      fmt.Printf("received: %s\n", v)
   }

}



--------




package main

import (
	"fmt"
	"time"
)

func main() {

	c := make(chan string, 2)
	s := []string{"A", "B", "C", "D"}
	go func() {
		defer close(c)
		for _, v := range s {
			fmt.Printf("send %s\n", v)
			c <- v
			time.Sleep(1 * time.Second)
		}
	}()
	time.Sleep(5 * time.Second)
	for v := range c {
		fmt.Printf("received: %s\n", v)
	}

}

```



## 使用

### 发送和接收

channel是一个双向的管道，具有发送和接收的功能

#### **发送**

```go
c <- data
```

#### 接收

```go
data <- c
```

对于接收来说，除了一般的`data <- c`,channel的接收还给我们提供了一个返回值

```go
data,ok:= <-c
```

这里的ok可以用来表示接受到的值是上游的发送收到的值，还是channel被关闭之后收到的值

**示例**

正常channel接收到的值

```go

	c := make(chan int)
	go func(c chan int) {
		c <- 1
	}(c)
	value, ok := <-c
	// value 1,ok true
	fmt.Printf("value %d,ok %v\n", value, ok)
```

channel关闭后接受到的值

```go
c := make(chan int)

	go func(c chan int) {
		close(c)
	}(c)
	value, ok := <-c
	// value 0,ok false
	fmt.Printf("value %d,ok %v\n", value, ok)

```

### 关闭

channel的关闭可以直接使用内置close方法

```go
close(chan)
```

如果channel已经关闭，那么继续向channel发送数据后会panic

`panic: send on closed channel`

但是我们可以一直去读取一个已经关闭的channel,但是返回的只有零值和false

```go
package main

import "fmt"

func main() {

   c := make(chan int)

   close(c)
   for {
      value, ok := <-c
      fmt.Printf("v %d,ok %v\n", value, ok)
   }

}
```

### 单向channel

#### 单向发送

```go
var c chan<- interface{}
c = make(chan<- interface{})
```

或

```go
c:=make(chan<- interface{})
```

#### 单向接收

```go
var c <-chan interface{}
c = make(<-chan interface{})
```

或

```go
c:=make(<-chan interface{})
```

单向的channel虽然存在，但是我们一般不会直接去声明一个单项的channel,而是会通过先声明一个双向的channel,然后将其隐式转换成一个单向的channel来进行使用

**示例**

```go
	c := make(chan int)
	// c chan<- int
	// 将双向的channel 转换成单向只输入的channel
	go func(c chan<- int) {
		c <- 1
	}(c)

	fmt.Println(<-c)
```







## 警告

对于一个已经阻塞的channel来说，如果继续发送，或接收会出现死锁的运行时错误



<img src="https://img.ethanleo.top/uPic/image-20221126123316398.png" alt="image-20221126123316398" style="zoom:50%;" />

**示例**

非缓冲channel只有

```go
package main

func main() {

	c := make(chan int)
	<-c
}

```

非缓冲channel只有发送

```go
package main

func main() {

	c := make(chan int)
	c <- 1
}

```

空的缓冲channel只有接收

```go
package main

func main() {

   c := make(chan int, 1)
   <-c
}
```

缓冲channel满了之后只有发送没有接收

```go
package main

func main() {

   c := make(chan int, 1)
   c <- 1
   c <- 2
}
```

## channel关闭问题示例

### 非缓冲channel

```go
package main

import (
   "fmt"
   "time"
)

func main() {

   ch := make(chan int)
   go handle(ch)

   ch <- 1
   ch <- 2
   close(ch)

}

func handle(c chan int) {
   for v := range c {
      fmt.Printf("start to deal %s \n", time.Now().Format("2006-01-02 15:04:05"))
      fmt.Println(v)
      time.Sleep(5 * time.Second)
      fmt.Printf("done %s \n", time.Now().Format("2006-01-02 15:04:05"))
   }
}
```

### 缓冲channel

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	ch := make(chan int, 3)
	go product(ch)
	time.Sleep(3 * time.Second)
	for i := range ch {
		fmt.Println(i)
		time.Sleep(1 * time.Second)
	}
}

func product(ch chan int) {

	for i := 0; i < 3; i++ {
		ch <- i + 1
	}
	close(ch)
	fmt.Println("channel close")
}

```

![image-20230109134235407](https://img.ethanleo.top/uPic/image-20230109134235407.png)**提问**

```
对于 channel 的关闭还有点疑惑，如果发送端关闭了 channel，但是接收端来不及处理，或者说处理的慢，岂不是会因为 channel close 而读不到了么?

```

**回答**

​	根据上面的两个示例，其实我们可以发现，这个问题得分开讨论，一种是带有不带缓冲的channel，一种是带有缓冲的channel。

* 不带缓冲的channel

对于不带缓冲的channel来说，发送接收都是阻塞的，也就是说，假如发送端需要连续发送两个值进去，但是消费端消费速度过慢，那么发送端在发送的时候就会被阻塞，也就是一定得让消费端消费完第一个，发送端才会继续发送下一个值，接下来再去关闭channel

* 带缓冲的channel

对于带缓冲的channel来说，由于具有缓冲，所以在缓冲区满之前都是非阻塞的，也就是说，生产端可以提前向channel里面放值，然后将channel关闭。消费端处理的慢也没事，就算channel关闭了他依然可以去迭代channel中的数据