---
title: golang之泛型
 
categories: golang
comments: true
---

# golang之泛型

## 简介

泛型（Generics）是指在定义函数、接口或类的时候，不预先指定具体的类型，而在使用的时候再指定类型的一种特性

<!--more-->

golang在1.18实现了泛型的语言特性

* 函数和类型的类型参数
* 将接口定义为类型集，不包括任何的方法类型
* 类型推断，允许调用函数的时候省略类型参数

## 示例

```go
package main

import "fmt"

func ExistInt(s []int, i int) bool {
   for _, v := range s {
      if v == i {
         return true
      }
   }
   return false
}

func ExistString(ss []string, s string) bool {
   for _, v := range ss {
      if v == s {
         return true
      }
   }
   return false
}

func Exist[T int | string](s []T, i T) bool {
   for _, v := range s {
      if v == i {
         return true
      }
   }
   return false
}

func main() {

   // fmt.Printf("exist int: %v\nexist string: %v", ExistInt([]int{1, 3, 4, 5, 6}, 7), ExistString([]string{"a", "b", "C"}, "a"))
   fmt.Printf("exist int: %v\nexist string: %v", Exist([]int{1, 3, 4, 5, 6}, 7), Exist([]string{"a", "b", "C"}, "a"))

}
```

## 类型声明

```go
// 类型参数
[T []int] = [T interface{[]int}]

// 泛型类型
type T interface{
  int | string
}

//
type Node[T string|int]{
  Data T
}
```

## 函数声明

```go
func min[T ~int|~float64](x,y T)T{
  if x < y{
    return x
  }
  return y
}
```

## ~符号

```go
type T interface {~string}
```

这里的～符号的意思就是对所有的字符串都可以支持，而不是单纯的string

例如:

```go
type Name string
```

如果我们定一个Name类型，他也是会被支持的，但是如果我们没有加上～号，那么就是不会被支持的

[泛型官方教程](https://go.dev/doc/tutorial/generics)

[An Introduction To Generics](https://go.dev/blog/intro-generics)





