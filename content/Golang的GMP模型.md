---
title: GMP模型
 
categories: 日常学习
comments: true
---



# Golang的GMP模型

### Golang对协程的处理

![image-20220824200321748](https://img.ethanleo.top/uPic/image-20220824200321748.png)

<!--more-->

### Golang早期调度器的处理

## GMP

|      |                 |
| :--: | :-------------: |
|  G   |  goroutine协程  |
|  P   | processor处理器 |
|  M   |   thread线程    |



* **全局队列** ：用来存放等待运行的G

* **本地队列** : 
  * 存放等待运行的G
  * 数量限制，不超过256G
  * 优先将创建的G放在P的本地队列中，如果满了会放在全局队列
* **P列表**
  * 程序启动时创建
  * 最多有GOMAXPROCS个(可配置)
* **M列表**
  * 当前操作系统分配到当前Go程序的内核线程数
* **P和M的数量**
  * P的数量问题
    * 设置环境变量$GOMAXPROCS
    * 在程序中通过runtime.GOMAXPROCS()来设置
  * M的数量问题
    * GO本身限定M的最大是10000
    * runtime/debug包中的SetMaxThreads函数来设置
    * 有一个M阻塞，会创建一个新的M
    * 如果有M空闲，那么会回收或者睡眠



### 调度器设计策略

#### 复用线程

**work stealing机制**

<img src="https://img.ethanleo.top/uPic/image-20220824203648126.png" alt="image-20220824203648126" style="zoom:50%;" />



当本地线程M2无可运行的G时，尝试从其他线程绑定的P中偷取G，而不是销毁线程M2

<img src="https://img.ethanleo.top/uPic/image-20220824203709562.png" alt="image-20220824203709562" style="zoom: 50%;" />

**hand off机制**

<img src="https://img.ethanleo.top/uPic/image-20220824203925458.png" alt="image-20220824203925458" style="zoom:50%;" />

本地线程M1因G1进行系统调用阻塞时，线程释绑定的P，把P转移给其他的空闲线程执行，在运行阻塞G1的M1中的G1重新运行后，如果G1还在运行，那么M1继续执行，如果G1运行完成，那么M1随即进入睡眠或被销毁

<img src="https://img.ethanleo.top/uPic/image-20220824204045795.png" alt="image-20220824204045795" style="zoom:50%;" />



#### 利用并行

**GOMAXPROCS限定P的个数=CPU核数/2**

设置P的个数，最多有GOMAXPROCS个线程分布在多个CPU上同时运行

#### 抢占

<img src="https://img.ethanleo.top/uPic/image-20220824204635413.png" alt="image-20220824204635413" style="zoom:50%;" />



在coroutine中要等待一个协程主动让出CPU才执行下一个协程



<img src="https://img.ethanleo.top/uPic/image-20220824204746906.png" alt="image-20220824204746906" style="zoom:50%;" />

在Go中，一个goroutine最多占用CPU10ms，防止其他goroutine被饿死

#### 全局G队列

**work stealing机制，从全局偷取**

<img src="https://img.ethanleo.top/uPic/image-20220824205016842.png" alt="image-20220824205016842" style="zoom:50%;" />

当M2执行work stealing从其他P偷不到G时，他会从全局G队列中获取G，这个获取的过程时需要加锁的

<img src="https://img.ethanleo.top/uPic/image-20220824205045123.png" alt="image-20220824205045123" style="zoom:50%;" />

## GO指令调度流程

1. 通过Go func() 创建一个goroutine
2. 有两个存储G的队列，一个是局部的调度器P的本地队列、一个是全局G队列。新创建的G会优先保存在本地队列中，如果本地队列满了，就会存放在全局队列中。
3. G只能运行在M中，一个M必须持有一个P，M与P的关系是1:1。M会从本P的本地队列中弹出一个可执行状态的G来执行，如果P的本地队列为空，那么就会从其他的组合中偷取一个可执行的G来执行，又或者去全局队列中获取一个可执行的G
4. 假设M在执行G的时候，G发生了systemcall或者其他阻塞操作，M会阻塞。如果当前有一些G正在执行，runtime会把这个线程M从P中摘除，然后创建一个新的M(如果有空闲的线程就会复用空闲的线程)来服务于摘除的P。这个时候被摘除的M就和阻塞的G绑定在一起了，在执行完G之后，M可能会被休眠或销毁。又或者重新接管P(P没有被其他的M给绑定)

<img src="https://img.ethanleo.top/uPic/image-20220825194900010.png" alt="image-20220825194900010" style="zoom:50%;" />





## 调度器生命周期

* **M0** 
  * 启动程序后编号为0的主线程
  * 在全局runtime.m0中，不需要在heap上分配
  * 负责执行初始化操作和启动第一个G
  * 启动第一个G后，M0和其他的M一样
* **G0**
  * 每次启动一个M，都会第一个创建的gorutine，就是G0
  * G0仅用于负责调度G
  * G0不指向任何可执行的函数
  * 每个M都会有一个自己的G0
  * 在调度或系统调用时会使用M切换到G0,来调度M0的G0会放在全局空间



## GMP可视化编程

1. 创建trace文件 **os.Create("trace.out")**
2. 启动trace **trace.Start(f)**
3. 停止trace **trace.Stop()**
4. go build 并且运行后可以得到一个trace.out文件 
5. 通过go tool trace 打开tarce文件  go tool trace trace.out

```go
package main

import (
	"fmt"
	"os"
	"runtime/trace"
)

func main() {
	// 1.创建一个trace文件
	f, err := os.Create("trace.out")
	if err != nil {
		panic(err)
	}

	defer f.Close()

	// 2.启动trace
	err = trace.Start(f)
	if err != nil {
		panic(err)
	}

	fmt.Println("Hello GMP")

	// 3.停止trace
	trace.Stop()

}

```



![image-20220825201131366](https://img.ethanleo.top/uPic/image-20220825201131366.png)

打开View trace

![image-20220825201327121](https://img.ethanleo.top/uPic/image-20220825201327121.png)

<img src="https://img.ethanleo.top/uPic/image-20220825201951963.png" alt="image-20220825201951963" style="zoom:50%;" />

* Gotoutines  G协程信息
  * Goroutines Value G0
  * Goroutines Value G1
* Heap 堆栈信息
* Threads M线程信息
  * Threads Value 0 M0
* PROCS P调度器信息



### 通过Debug trace查看GMP信息

GODEBUG=schedtrace=1000 ./可执行程序

```go
package main

import (
	"fmt"
	"time"
)

func main()  {
	for i := 0; i < 5; i++ {
		time.Sleep(time.Second)
		fmt.Println("Hello World")
	}
}


 	go build trace.go 

	GODEBUG=schedtrace=1000 ./trace
```



```bash
SCHED 0ms: gomaxprocs=8 idleprocs=7 threads=2 spinningthreads=0 idlethreads=0 runqueue=0 [0 0 0 0 0 0 0 0]
Hello World
SCHED 1010ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello World
SCHED 2011ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello World
SCHED 3018ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello World
SCHED 4022ms: gomaxprocs=8 idleprocs=8 threads=5 spinningthreads=0 idlethreads=3 runqueue=0 [0 0 0 0 0 0 0 0]
Hello World

```

* SCHED 调试的信息
* 0ms 从程序启动到输出的时间
* gomaxprocs P的数量 一般默认是和CPU的核心数是一致的
* idleprocs 处理idle状态P的数量，gomaxprocs - idleprocs= 目前正在执行的P的数量
* threads 线程数量(包括M0,也包括当前GODEBUG调试线程)
* Spinningthreads 处于自旋状态thread的数量
* Idlethread 处理idle状态thread
* Runqueue 全局G队列中的G的数量
* [0 0 0 0 0 0 0 0] 每个P的local queue本地队列中，目前存在的G的数量



## GMP调度分析

### G1创建G3

<img src="https://img.ethanleo.top/uPic/image-20220825204942399.png" alt="image-20220825204942399" style="zoom:50%;" />

P拥有G1，M1获取P后开始运行G1，G1使用go func()创建了G3，为了局部性G3优先加入到P的本地队列中

### G1执行完毕

<img src="https://img.ethanleo.top/uPic/image-20220825205448270.png" alt="image-20220825205448270" style="zoom:50%;" />

G1运行完成之后，M上运行的Goroutine切换为G0，G0负责调度时协程切换（schedule）从P本地队列取出G2，从G0切换到G2，并且开始运行G2，实现线程M的复用

### 开辟过多的G

<img src="https://img.ethanleo.top/uPic/image-20220825210210240.png" alt="image-20220825210210240" style="zoom:50%;" />



1. 如果连续开辟过多的G，会先把本地G存满

2. 如果本地已经满了，还需要继续创建，那么就会把本地队列前一半的G打乱顺序，和新创建的G放到全局队列中
3. 继续创建G，这个时候本地队列没满，那么就会新创建的G存放在本地队列

### 唤醒正在休眠的M

![image-20220826191558727](https://img.ethanleo.top/uPic/image-20220826191558727.png)

在创建G时，运行的G3会尝试唤醒其他空闲的P和M组合去执行，假设G3唤醒了M1，M1绑定了P1，首先要运行G0，但是由于P1本地队列中不存在G。那么M2此时就会进入自旋状态，变成自旋线程(没有G但是为运行状态的线程，不断寻找G)

### 被唤醒的M从全局队列中取批量的G

<img src="https://img.ethanleo.top/uPic/image-20220826192048534.png" alt="image-20220826192048534" style="zoom:50%;" />

从全局队列中取P的过程称之为负载均衡

### 偷取G

<img src="https://img.ethanleo.top/uPic/image-20220826192709291.png" alt="image-20220826192709291" style="zoom:50%;" />

假设全局队列中已经没有G了，那么M1会进行work stealing,会从其他有G的P中偷取一半的G，放在自己的本地P队列中，然后由自身的G0进行调度，并且运行G。图中M绑定的P一半剩下后只有G9被偷取

### 自旋线程的最大限制

<img src="https://img.ethanleo.top/uPic/image-20220826193220466.png" alt="image-20220826193220466" style="zoom:50%;" />



自旋线程+执行线程 <= **GOMAXPROCS**

### G发生系统调用/阻塞

<img src="https://img.ethanleo.top/uPic/image-20220826193920364.png" alt="image-20220826193920364" style="zoom:50%;" />

当G4发生了系统调用或者阻塞的时候，会尝试从休眠线程队列中唤醒一个M，去和P1绑定，并且让唤醒的M去执行G5，自旋线程只存在偷取G的情况，所以不会去绑定其他的P，因为自身已经绑定了P

### G系统调用结束/非阻塞

<img src="https://img.ethanleo.top/uPic/image-20220826194413853.png" alt="image-20220826194413853" style="zoom:50%;" />

加入G4完成了系统调用或者非阻塞了，与之绑定的M会去寻在之前的P1，查看是否可以绑定，加入不能绑定，那么回去空闲P队列中查看，如果空闲P队列中也没有P，那么G4会被放回全局队列等待调度