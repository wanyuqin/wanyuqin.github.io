---
title: Go之Unsafe包
 
categories: golang
comments: true
---



# Go之Unsafe包

## 概述

在Go中，我们常见的指针是类型安全的，但是存在很多的限制。比如:

* Go的指针不能进行数学运算
* 不同类型的指针不能进行相互转换
* 不同类型的指针不能用 `==`或者`!=`比较
* 不同类型的指针不能相互赋值

但是在Go中也存在非类型安全的指针，比如说unsafe.Pointer。

<!--more-->

unsafe包中包含绕过 Go 程序的类型安全的操作。导入不安全的包可能是不可移植的，并且不受 [Go 1兼容性指南](https://go.dev/doc/go1compat)的保护。

此文章是基于go1.20版本进行介绍的，不同的go版本，该包中的方法会有些不同

在go1.20的版本中，unsafe包的结构是

```go
func Alignof(x ArbitraryType) uintptr
func Offsetof(x ArbitraryType) uintptr
func Sizeof(x ArbitraryType) uintptr
func String(ptr *byte, len IntegerType) string （Add in go 1.20)
func StringData(str string) *byte （Add in go 1.20)
type ArbitraryType
	func Slice(ptr *ArbitraryType, len IntegerType) []ArbitraryType （Add in go 1.17)
	func SliceData(slice []ArbitraryType) *ArbitraryType （Add in go 1.20)
type IntegerType
type Pointer
	func Add(ptr Pointer, len IntegerType) Pointer （Add in go 1.17)
```

## unsafe.Pointer

```go
type Pointer *ArbitraryType
type ArbitraryType int
```

这里的`ArbitraryType`它表示Go用来表示任意类型的一个简写，它并不表示任何一个类型

`Pointer` 非类型安全指针

* 任何类型的指针都可以与`Pointer`相互转换
* uintptr和`Pointer`可以相互转换

## 函数

* **func** **Sizeof**(x ArbitraryType) **uintptr**

​	返回类型x所占的字节数，但不包含x所指向的内容大小，对于一个slice来说返回的就是slice header的大小

```go
func demo1() {
	a := int(1)
	b := float32(2.0)
	s := "Ethan"
	sl := make([]string, 16)

	println(unsafe.Sizeof(a))  // 8
	println(unsafe.Sizeof(b))  // 4
	println(unsafe.Sizeof(s))  // 16
	println(unsafe.Sizeof(sl)) // 24
}
```

* **func** **Offsetof**(x ArbitraryType) **uintptr**

​	返回结构体成员在内存中位置离结构体起始处的字节数，所传递的参数必须是结构体的成员

```go
	
type User struct {
	Name     string
	Nickname string
	Age      int
}

func demo2() {
	u := User{
		Age:      10,
		Name:     "Ethan",
		Nickname: "Leo",
	}
	println(unsafe.Offsetof(u.Name))     // 0
	println(unsafe.Offsetof(u.Nickname)) // 16
	println(unsafe.Offsetof(u.Age))      // 32
}

```

* **func** **Alignof**(x ArbitraryType) **uintptr**

​	返回的值是指当类型进行内存对齐时，它分配到的内存地址可以整除该函数的返回值	

```go

func demo3() {
	a := 1
	b := int8(8)
	s := "1"
	println(unsafe.Alignof(a)) // 8
	println(unsafe.Alignof(b)) // 1
	println(unsafe.Alignof(s)) // 8
}
```

以上三个函数返回的结果都是`uintptr`类型，这个类型可以和`unsafe.Pointer`类型相互转换。由于三个函数都在编译期间执行，所以它们的结果可以直接赋值给`const`变量

* **func String**(ptr *byte, len IntegerType) **string**

该函数返回一个字符串，底层字节从ptr开始，长度为len。len参数必须是证书类型或者是无类型常量，常量len参数必须是非负的并且可以用int类型的值来表示。如果在运行时len为负，或者ptr为nil且len不为零，那么会发生panic

  ```go
  func demo4() {
  	b := []byte{'h', 'e', 'l', 'l', 'o'}
  	println(unsafe.String(&b[0], len(b))) // hello
  
  }
  ```

* **func StringData**(str string) ***byte**

StringData返回一个指向str的底层字节的指针。对于空字符串，返回值未指定，可能为nil。由于Go字符串是不可变的，因此不能修改StringData返回的字节。

```go
func demo5() {
	println(unsafe.StringData("hello")) // 0x100b2bfc5
}


```

* **func Slice**(ptr *ArbitraryType, len IntegerType) **[]ArbitraryType**

函数Slice返回一个切片，其底层数组以ptr开头，长度和容量为len，如果ptr为nil，len为零，Slice将返回nil。len参数必须是整数类型或非类型常量。常量len参数必须是非负的，并且可以由int类型的值表示；如果它是一个非类型化常量，则它被赋予int类型。在运行时，如果len为负，或者如果ptr为nil且len不为零，则会发生panic。

* **func Add**(ptr Point, len IntegerType) **Pointer**

此函数在一个(非安全)指针表示的地址上添加一个偏移量，然后返回一个新地址的指针‘

```go

func demo6() {

	a := [16]int{3: 3, 9: 9, 11: 11}
	fmt.Println(a)
	eleSize := int(unsafe.Sizeof(a[0]))
	p9 := &a[9]
	up9 := unsafe.Pointer(p9)
	p3 := (*int)(unsafe.Add(up9, -6*eleSize))
	fmt.Println(*p3) // 3
	s := unsafe.Slice(p9, 5)[:3]
	fmt.Println(s)              // [9 0 11]
	fmt.Println(len(s), cap(s)) // 3 5

	t := unsafe.Slice((*int)(nil), 0)
	fmt.Println(t == nil) // true

	// 下面是两个不正确的调用。因为它们
	// 的返回结果引用了未知的内存块。
	_ = unsafe.Add(up9, 7*eleSize)
	_ = unsafe.Slice(p9, 8)
}

```

## 使用

### String 和 []byte相互转换

```go
func ByteSlice2String(b []byte) string {
	return unsafe.String(&b[0], len(b))
}

func String2ByteSlice(s string) []byte {
	return unsafe.Slice(unsafe.StringData(s), len(s))
}
```

### 修改未导出的成员变量

 ```go
 package order
 
 type User struct {
 	Id   int
 	name string
 }
 
 func (u *User) Name() string {
 	return u.name
 }
 
 ```

```go
func demo7() {
	u := order.User{
		Id: 1,
	}
	println(u.Name())
	n := (*string)(unsafe.Add(unsafe.Pointer(&u), unsafe.Sizeof(u.Id)))
	*n = "Modify by unsafe"
	println(u.Name())
}
```

## 获取Slice和Map的长度

获取slice的长度

```go
// runtime/slice.go
// type slice struct {
// 	array unsafe.Pointer
// 	len   int
// 	cap   int
// }

// 获取slice的长度
func demo8() {
	s := make([]int, 9, 20)
	l := (*int)(unsafe.Add(unsafe.Pointer(&s), uintptr(8)))
	println(*l)
}
```

获取map的长度

```go
// runtime/map.go
//
// // A header for a Go map.
//
// type hmap struct {
//    // Note: the format of the hmap is also encoded in cmd/compile/internal/reflectdata/reflect.go.
//    // Make sure this stays in sync with the compiler's definition.
//    count     int // # live cells == size of map.  Must be first (used by len() builtin)
//    flags     uint8
//    B         uint8  // log_2 of # of buckets (can hold up to loadFactor * 2^B items)
//    noverflow uint16 // approximate number of overflow buckets; see incrnoverflow for details
//    hash0     uint32 // hash seed
//
//    buckets    unsafe.Pointer // array of 2^B Buckets. may be nil if count==0.
//    oldbuckets unsafe.Pointer // previous bucket array of half the size, non-nil only when growing
//    nevacuate  uintptr        // progress counter for evacuation (buckets less than this have been evacuated)
//
//    extra *mapextra // optional fields
// }
func demo9() {
   m := make(map[int]string)
   l := (**int)(unsafe.Pointer(&m))
   fmt.Println(**l)
   fmt.Println(len(m))
}
```





## 参考资料

[深度解密Go语言之Unsafe](https://qcrao.com/post/dive-into-go-unsafe/)

[Type-Unsafe Pointers](https://go101.org/article/unsafe.html)

[官方文档](https://go.dev/ref/spec#Package_unsafe)
