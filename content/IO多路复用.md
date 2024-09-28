---
title: IO多路复用
     
categories: 日常学习
comments: true
---



# IO多路复用

同一时刻，一个线程(进程)监听多个IO请求

<!--more-->

## 非阻塞+忙轮询

```go
while true {
  for i in 流[] {
    if i has 数据 {
       读或其他处理
    }
   } 
}
```

## Select

```go
while true {
 select(流[]);//阻塞
  for i in 流[] {
    if i has 数据 {
       读或其他处理
    }
   } 
}
```

* 最多处理1024个连接
* 跨平台，标准库的一个内容
* 只知道有，但是不能确定是那个

## Epoll

```go
while true {
 可处理的流[] = epoll_wait(epoll_fd); //阻塞
 	for i in 可处理的流[] {
 		读 或 其他操作
 	}
}
```

* 只关注活跃连接
* 能够处理大量的连接请求(cat /proc/sys/fd/file-max查看)
* Linux独有



## Epoll的API

### 创建epoll

```c
// size 告诉内核监听句柄的数量
// returns 返回一个epoll句柄(一个文件描述符)
int epoll_create(int size)
```

<img src="https://img.ethanleo.top/uPic/image-20220820123812589.png" alt="image-20220820123812589" style="zoom:50%;" />

创建poll，其实就是在内核创建一个红黑树(平衡二叉树)的根节点，这个root对应的就是epfd

### 控制epoll

```c
// epfd 用epoll_create锁创建的epoll句柄
// op 表示对epoll监控描述符控制的动作
//  EPOLL_CTL_ADD 注册新的fd到epfd
//  EPOLL_CTL_MOD 修改已经注册的fd监听事件
//  EPOLL_CTL_EDL epfd删除一个fd
// fd 需要监听的文件描述符
// event 告诉内核需要监听的事件
// returns 成功返回0 失败返回-1 ，errno查看错误信息
int epoll_ctl(int epfd,int op,int fd,struct epoll_event *event);


struct epoll_event{
  _uint32_t events; // epoll事件 EPOLLIN(可输入),EPOLLOUT(可输出),EPOLLPRI,EPOLLHUP,EPOLLET,EPOLLONESHOT
  epoll_data_t data; // 用户传递的数据
}

typeof union epoll_data{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
}epoll_data_t;
```

在有一个新的fd来到的时候，会使用epoll_ctl方法，将fd以及需要监听的时间，假如到最开始的epfd的红黑树中

```c
struct epoll_event new_event;
new_event.events = EPOLLIN | EPOLLOUT;
new_event.data.fd = 5;
epoll_ctl(epfd,EPOLL_CTL_ADD,5,&new_event);
```

<img src="https://img.ethanleo.top/uPic/image-20220820161159608.png" alt="image-20220820161159608" style="zoom:50%;" />

### 等待epoll

```c
// epfd 用epoll_create所创建的句柄
// event 从内核中得到的事件集合
// maxevents 告诉内核这个events有多大，值不能大于epoll_create()创建时的size
// timeout 超时时间 -1:永久阻塞; 0:立即返回，非阻塞; >0:指定微秒
// returns 成功:有多少就绪的fd,时间到的时候返回0; 失败: -1,errno 查看错误
int epoll_wait(int epfd, struct epoll_event *event, int maxevents, int timeout);

struct epoll_event my_event[1000];
int epoll_cnt = epoll_wait(epfd,my_event,1000,-1);
```

<img src="https://img.ethanleo.top/uPic/image-20220820161324824.png" alt="image-20220820161324824" style="zoom:50%;" />

假设event，和event1中监听的事件有就绪的状态，那么内核就会把这两个节点上的fd加入到我们之前调用的epoll_wait传入的my_event中，那么我们后续只需要遍历my_event就可以很好的去处理就绪的fd了。

## epoll编程架构

```c
//创建epoll
int epfd = epoll_create(1000);

//将listen_fd 加入epfd中
epoll_ctl(epfd,EPOLL_CTL_ADD,listen_fd,&listen_event);


while(1){
   //阻塞等待epoll的fd触发
  int active_cnt = epoll_wait(epfd,events,1000,-1);
  
  for (i=0;i<active_cnt;i++){
    if (events[i].data.fd==listen_fd){
      //创建accpectfd，并且将创建的accept_fd加入到epoll中
    }else if(event[i].events & EPOLLIN){
      // 对此fd进行读操作
    }else if(event[i].events & EPOLLOUT){
      // 对此fd进行写操作
    }
  }
}
```

### 

## 并发模型

### 单线程Accept(无IO复用)

<img src="https://img.ethanleo.top/uPic/image-20220822190305020.png" alt="image-20220822190305020" style="zoom:50%;" />

1. 主线程main thread 执行阻塞Accept，等待客户端连接，假如有客户端连接进入，accept响应并建立连接
2. 创建连接成功，得到套接字ConnFd1，然后在main thread串行处理套接字读写，并处理业务
3. 在处理业务的过程中，如果还有新的客户端连接请求，Server无响应，直到当前套接字全部业务处理完毕
4. 客户端处理完当前套接字后，处理下一个客户端连接请求

**该模型并不是一个并发模型，是串行的服务器，最大的网络请求量为1，即并发量为1**

### 单线程Accept+多线程读写

<img src="https://img.ethanleo.top/uPic/image-20220822191428439.png" alt="image-20220822191428439" style="zoom:50%;" />

1. 主线程main thread 执行阻塞Accept，等待客户端连接，假如有客户端连接进入，accept响应并建立连接
2. 创建连接成功，得到connfd套接字后，创建一个新的线程来处理客户端的读写业务，main thread依然会到Accept阻塞等待新的客户端请求
3. thread 通过套接字connfd与客户端进行读写

**优点**

* 支持了并发的特性

* 一只会有一个线程与客户端做资源响应

**缺点**

* 由于一个客户端需要对应一个线程，对于高并发的情况下，线程数量会受到硬件的瓶颈
* 线程过多会增加CPU的切换成本
* 对于长连接，客户端无读写，只要不关闭，就会保持这个连接的状态，浪费资源

### 单线程+多路IO复用

<img src="https://img.ethanleo.top/uPic/image-20220822200213407.png" alt="image-20220822200213407" style="zoom:50%;" />

1. 主线程 main thread创建ListenFd之后，采用多路IO复用机制(select、epoll)进行IO状态阻塞，有客户端发送connect请求，IO复用机制检测到ListenFd触发读写事件，则建立Accept建立，并将connFd加入到IO监听集合中
2. 如果客户端再次进行正常读写业务请求，主线程多路IO复用机制阻塞返回，会触发该套接字的读/写事件
3. 对于出发了读/写业务的客户端，Server仍然是在main thread执行流程，如果此时有新的客户端发送链接请求，server不会响应
4. 等待server处理完读/写请求，则继续回到多路IO复用阻塞机制

**优点**

* 可以监听多个客户端的读写状态
* 多路IO复用，是一个阻塞的，不会浪费CPU资源

**缺点**

* 可以同时监听多个客户端的读写状态，但是同一时间只能处理一个客户端的读写操作
* 并发量大的时候，会出现排队的情况

### 单线程多路IO复用+多线程业务工作池

![image-20220822203122836](https://img.ethanleo.top/uPic/image-20220822203122836.png)

1. 主线程main thread创建了listenFd之后，采用多路IO复用机制(select、epoll)进行IO状态阻塞，有客户端发送connect请求，IO复用机制检测到ListenFd触发读写事件，则建立Accept建立，并将connFd加入到IO监听集合中
2. 如果客户端再次进行正常读写业务请求，主线程多路IO复用机制阻塞返回，会触发该套接字的读/写事件
3. main thread会把真正的业务逻辑的处理分配给其他的thread，这样就可以让主线程更早的返回多路IO复用的监听
4. 等工作thread把业务处理完，要进行写操作的时候，在让main thread进行write操作

**优点**

* 将业务的处理部分交给业务工作池，可以减少客户端的读写请求的等待时间
* 读写的并发为1，但是业务流程的并发为work pool的线程数量

**缺点**

* 读写的操作依然是main thread单独处理
* 虽然多个worker处理业务，但是写事件还是要排队

### 单线程多路IO复用+多线程多路IO复用(线程池)

![image-20220822204049857](https://img.ethanleo.top/uPic/image-20220822204049857.png)

1. server在启动监听之前，开辟固定数量N的线程池，用Thread Pool线程池管理
2. 主线程main thread创建了listenFd之后，采用多路IO复用机制(select、epoll)进行IO状态阻塞，有客户端发送connect请求，IO复用机制检测到ListenFd触发读写事件，则建立Accept建立，并将connFd分发给Thread Pool中的某个线程进行监听
3. 线程池中的IO多路复用去进行读写事件的监听，然后进行业务处理
4. 每一个线程里的业务处理也是串行进行的，最高的读写并行通道依然为N(线程池线程数量)