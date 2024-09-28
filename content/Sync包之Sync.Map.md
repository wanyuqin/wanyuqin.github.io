---
title: Sync包之Sync.Map
 
categories: golang
comments: true
---

# Sync包之Sync.Map

## 简介

Sync包下的map很像是Go的map[interface{}]interface{}，但是对于普通的map来说，sync.Map可以保证在多个goroutine的并发使用是安全的，无需额外添加锁来进行控制

<!--more-->



## 方法

### func( * Map)Store(key,value any)

将一个key,value存入map中

```go
func main(){
	var smm sync.Map
	smm.Store("Ethan","Leo")
}
```

### func(* Map)Delete(key any)

将一个key从map中删除

```go
func main(){
	var smm sync.Map
	smm.Delete("Ethan")
}
```

### func(* Map)Load(key any) (value any, ok bool)

从map中获取一个Key

```go
func main(){
var smm sync.Map
	smm.Store("Ethan", "Leo")
	value, ok := smm.Load("Ethan")
	if !ok {
		fmt.Println("The key not exist")
		return
	}
	fmt.Println(value)
}
```

### func(* Map)LoadAndDelete(key any) (value any, loaded bool)

从map中获取一个key，并且将key删除，如果key存在那么loaded会返回true，并且value就是map中存的值，如果key不存在那么loaded会返回一个false，并且value为nil

```go
func main(){
var smm sync.Map
	smm.Store("Ethan", "Leo")
	value, loaded := smm.LoadAndDelete("Ethan")
	fmt.Printf("values is %v,is loaded %v \n", value, loaded)
	value, loaded = smm.LoadAndDelete("Ethan")
	fmt.Printf("values is %v,is loaded %v \n", value, loaded)
}
```

### func (m *Map) LoadOrStore(key, value any) (actual any, loaded bool)

从map中加载一个key，如果key不存在，那么就会将value存放在key中，返回value，loaded为false，如果key存在则返回value，loaded为true

```go
func main(){
var smm sync.Map
	actual, loaded := smm.LoadOrStore("Ethan", "Leo")
	fmt.Printf("actual %v,is loaded %v \n", actual, loaded)
	actual, loaded = smm.LoadOrStore("Ethan", "Ethan")
	fmt.Printf("actual %v,is loaded %v \n", actual, loaded)
	value, ok := smm.Load("Ethan")
}
```

### func (m *Map) Range(f func(key, value any) bool)

迭代出map中的key，value,如果回调函数中返回false，那么就会停止迭代，如果返回true，那么就会继续迭代

```go
func main(){
var smm sync.Map
	smm.Store("Ethan", "Leo")
	smm.Store("Jack", "Chen")
	smm.Range(func(key, value any) bool {
		fmt.Printf("Key: %v - Value: %v\n", key, value)
		return true
	})
}
```

