---
title: Sync包之Sync.Mutex
 
categories: golang
comments: true
---

# Sync包之Sync.Mutex

## 简介

简单来说Sync.Mutex就是一个互斥锁

<!--more-->



## 方法

### func (m *Mutex) Lock()

对m的锁进行加锁，如果m已经被锁定，那么goroutine就会被阻塞，直到成功获取到锁

### func (m *Mutex) TryLock() bool

尝试去获取m的锁，并且返回是否成功获取到锁，如果获取锁失败，那么就会返回false，如果成功那么就对m进行加锁

### func(m *Mutext)UnLock()

对m锁进行解锁

### 示例

```go
package main

import (
   "fmt"
   "math/rand"
   "sync"
   "time"
)

type Toilet struct {
   m sync.Mutex
}

type Person struct {
   Name string
}

var DateTime = "2006-01-02 15:04:05"

var otherThings = []string{"唱歌", "跳舞", "打坐"}

func main() {

   t := &Toilet{}
   wg := sync.WaitGroup{}
   wg.Add(5)
   for i := 0; i < 5; i++ {

      go func(i int, t *Toilet, wg *sync.WaitGroup) {
         p := Person{
            Name: fmt.Sprintf("%d号", i),
         }
         for !t.m.TryLock() {
            p.DoOtherThing()
         }
         p.InToiletLock(t, wg)
         // InToiletUnLock(wg)
      }(i, t, &wg)
   }
   wg.Wait()
}

func (p *Person) InToiletLock(t *Toilet, wg *sync.WaitGroup) {
   defer wg.Done()
   // t.m.Lock()
   i := rand.Intn(5)

   fmt.Printf("%s,上厕所时间 %v\n", p.Name, time.Now().Format(DateTime))
   time.Sleep(time.Duration(i) * time.Second)
   t.m.Unlock()
}

func (p *Person) InToiletUnLock(wg *sync.WaitGroup) {
   defer wg.Done()
   i := rand.Intn(5)
   fmt.Printf("%s,上厕所时间 %v\n", p.Name, time.Now().Format(DateTime))
   time.Sleep(time.Duration(i) * time.Second)
}

func (p *Person) DoOtherThing() {
   i := rand.Intn(2)
   fmt.Printf("%s想上厕所,厕所有人,做些其他的事情 %s\n", p.Name, otherThings[i])
   time.Sleep(3 * time.Second)
}
```