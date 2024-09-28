---
title: Sync包之Sync.RWMutex
 
categories: golang
comments: true
---

# Sync包之Sync.RWMutex

## 简介

RWMutex就是一个读写锁，可以被多个读者或单个写持有，假如有一个goroutine持有读锁，而另一个goroutine想进行Lock,那么调用Lock的goroutine会被一直阻塞，直到读锁被释放

<!--more-->



## 方法

### func (rw *RWMutex) Lock()

加写锁

### func (rw *RWMutex) RLock()

加读锁

### func (rw *RWMutex) RUnlock()

读锁释放，如果在释放的时候没有加锁，那么是一个运行时错误

### func (rw *RWMutex) TryLock() bool

尝试加写锁，如果加锁失败返回false，如果加锁成功返回true，同时已经拿上了写锁

### func (rw *RWMutex) TryRLock() bool

尝试加读锁，如果加锁失败返回false，如果加锁成功返回true，同时已经拿上了读锁

### func (rw *RWMutex) Unlock()

写锁释放，如果在释放的时候没有加锁，那么是一个运行时错误

## 示例



```go
package main

import (
   "fmt"
   "math/rand"
   "sync"
   "time"
)

type Cinema struct {
   rw *sync.RWMutex
}

type Person struct {
   Name string
}

type Admin struct {
   Name string
}

func main() {
   // example1()
   example2()
}

func (p *Person) SitDown(wg *sync.WaitGroup) {
   defer wg.Done()
   fmt.Printf("%s 观众坐下\n", p.Name)
}

func (p *Person) StartWatch() {
   fmt.Printf("%s 开始观看\n", p.Name)
   i := time.Duration(rand.Intn(5) + 1)
   time.Sleep(i * time.Second)
   fmt.Printf("%s 观看结束\n", p.Name)

}

func (p *Person) WatchMovie(c *Cinema, wg *sync.WaitGroup, b chan int) {
   c.rw.RLock()
   // 先坐下
   p.SitDown(wg)
   <-b
   // 看电影
   p.StartWatch()
   c.rw.RUnlock()

}

func (a *Admin) ChangeMovie(c *Cinema, wg *sync.WaitGroup) {
   defer wg.Done()
   c.rw.Lock()
   // for !c.rw.TryLock() {
   //     fmt.Printf("还有人看电影\n")
   //     time.Sleep(1 * time.Second)
   // }
   fmt.Printf("所有观众看完电影，%s管理员切换影片\n", a.Name)
   c.rw.Unlock()
}

// 电影院
func example1() {
   c := &Cinema{
      rw: &sync.RWMutex{},
   }
   a := &Admin{
      Name: "强人",
   }

   b := make(chan int)

   wg := sync.WaitGroup{}
   wg.Add(10)
   for i := 0; i < 10; i++ {
      go func(i int, c *Cinema, wg *sync.WaitGroup, b chan int) {
         p := Person{
            Name: fmt.Sprintf("%d 号观众", i),
         }
         p.WatchMovie(c, wg, b)

      }(i, c, &wg, b)
   }
   wg.Wait()
   fmt.Printf("所有观众入座，开始播放电影\n")
   close(b)

   wg.Add(1)
   go func(wg *sync.WaitGroup) {
      a.ChangeMovie(c, wg)
   }(&wg)
   wg.Wait()
}

type Desk struct {
   rw     *sync.RWMutex
   Snacks int
}

func example2() {
   d := Desk{
      rw: &sync.RWMutex{},
   }
   wg := sync.WaitGroup{}
   wg.Add(12)

   go d.PlaceSnacks(&wg)
   for i := 0; i < 11; i++ {
      p := Person{
         Name: fmt.Sprintf("%d 号猿猴", i),
      }
      go p.GetSnacks(&d, &wg)
   }

   wg.Wait()
}

func (d *Desk) PlaceSnacks(wg *sync.WaitGroup) {
   defer wg.Done()
   d.rw.Lock()
   d.Snacks = 10
   time.Sleep(3 * time.Second)
   fmt.Printf("零食准备完毕...\n")
   d.rw.Unlock()
}

func (p *Person) GetSnacks(d *Desk, wg *sync.WaitGroup) {

   for !d.rw.TryRLock() {
      fmt.Printf("桌上没零食，%s望眼欲穿\n", p.Name)
      time.Sleep(2 * time.Second)
   }

   d.rw.RUnlock()
   defer wg.Done()
   defer d.rw.Unlock()
   d.rw.Lock()
   if d.Snacks > 0 {
      fmt.Printf("%s抢到零食，开心！\n", p.Name)
      d.Snacks--
      return
   }

   fmt.Printf("%s没抢到零食，难受！\n", p.Name)

}
```