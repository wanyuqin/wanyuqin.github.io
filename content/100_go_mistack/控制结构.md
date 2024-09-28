

# 控制结构

​	Go中的控制结构类似于C或Java中的控制结构，但是在其他方面他们不同。例如：Go中没有do或while循环，只有一个广义的for。本章深入探讨与控制结构相关的最常见错误。重点关注`range`循环。这是错误最常见的来源

<!--more-->

## 忽略元素在range循环中被复制的事实

​	`range`循环是一种遍历各种数据结构的便捷方式。我们不必处理索引和终止状态。Go程序员可能会忘记，`range`循环如何分配值，导致超贱的错误。首先，让我们看看如何使用`range`循环，然后我们在看看在range中值是如何赋值的

### 简介

​	`range`支持对不同的数据结构进行循环

* String
* Array
* Pointer to an Array
* Slice
* Map
* Receiving channel

​	

​	与传统的for循环相比较，`range`循环更加的方便，也不太容易出错，因为我们不需要去处理条件表达式和迭代的变量，下面是一个对于字符串切片进行迭代的示例：

```go
s := []string{"a", "b", "c"}
for i , v :=range s {
  fmt.Printf("index=%d, value=%s\n", i, v)
}
```

​	此代码遍历了切片的每个元素，在每次迭代的时候，`range`会产生一对值：一个索引和一个元素值，分别分配给i和v。一般来说，range迭代美中数据结构的时候都会产生两个值（channel除外），channel的迭代只会产生一个元素

​	在某些情况下，我们可能只对元素值感兴趣，而不是索引。但是由于不使用局部变量会导致编译错误，所以我们可以使用空白标识符来代替索引变量，例如：

```go
s := []string("a", "b", "c")
for _, v := range s {
  fmt.Printf("index=%d, value=%s\n", i, v)
}
```

​	通过空白标识符，我们可以忽略索引，并且仅将元素值分配个v来迭代每个元素，如果我们对值不感兴趣，我们可以省略第二个元素

```go
for i := range s {}
```

### 值的复制

​	了解在每次迭代的过程中如何处理值至关重要，让我们通过一个具体的示例来看看他是如何工作的。

​	首先我们创建一个account的结构体，该结构体中包含了banlance的字段

```go
type account struct {
  balance float32
}
```

​	接下来，我们创建一个account类型的切片，然后通过range去对其进行迭代，在迭代的过程中，我们对切片中的每一个元素的balance进行累加

```go
func main() {
   accounts := []Account{
      {balance: 100.},
      {balance: 200.},
      {balance: 300.},
   }
   for _, a := range accounts {
      a.balance += 1000.
   }
}
```

​	上面的代码运行的结果是[{100},{200},{300}]，而并不是我们预期的[{1100},{1200},{1300}]，这是为什么呢？这是因为，在Go中，我们分配的所有内容都是一个副本

* 如果我们给一个函数的返回值设置为一个struct，那么他就会将其副本进行返回
* 如果我们函数的返回值是一个指针，那么函数返回的其实是内存地址的副本

​	牢记这一点对于避免常见的错误（包括与range循环相关的错误）至关重要。实际上，当我们使用range进行遍历数据结构时候，它会将每个元素的值复制到第二个变量上

​	那么在上面的示例中，迭代的过程中会将accounts里的每个元素的值复制给`a`，因此，在我们执行`a.balance+=1000.`的时候，它改变的是a的值，而不是切片中的元素。

​	那么，如果我们想更新切片的元素该怎么办？有两个方式，第一是选择通过索引访问切片里的元素，第二种方式是使用经典的for循环来实现

```go
for i := range accounts {
  accounts[i].balance+=1000.
}

for i:=0 i<len(accounts);i++{
  accounts[i].balance += 1000
}
```

​	两种迭代都具有相同的效果，都可以更新accounts切片中的元素。那么我们应该使用哪一种呢？这取决上下文，如果我们想遍历每个元素，第一个的写入和读取的时间会更短，但是如果我们需要控制迭代更新具体的某个元素或者某几个连续的元素，那么第二种就会更加适合。

​	所以，我们应该记住range循环中的值元素是一个副本。因此，如果值是我们需要改变的结构，我们只会更新副本而不是元素的本身，除非我们修改的值或字段是指针。

## 忽略在range循环中的参数计算方式

​	rang循环语法需要一个表达式，比如`for i,v := range exp `这里的exp就是表达式。正如我们所见，它可以是字符串、数组、指向数组的指针、切片、map或者channel。现在，我们来了解一下，这个表达式是如何求值的？

​	让我们看看下面的示例，你相信会终止循环吗？

```go
package main

import "fmt"

func main() {
   s := []int{0, 1, 2}
   for range s {
      s = append(s, 10)
   }

   fmt.Println(s)
}
```

​	要理解这个问题，我们需要知道在使用range循环时，所提供的表达式仅在循环开始之前计算一次。在这个上下文中，计算意味着提供的表达式会复制给一个临时变量，然后range会去遍历这个临时变量。

![image-20230105201419547](https://img.ethanleo.top/uPic/image-20230105201419547.png)

​	在这个例子中，当对s进行表达式求值的时候，其实是对切片的副本进行计算，同时range在进行迭代的时候也是使用这个临时变量。而原始切片s在每次迭代的时候也会进行更新，因此在三次迭代之后会就变成

![image-20230105201632018](https://img.ethanleo.top/uPic/image-20230105201632018.png)

​	每一次的迭代都会添加一个新的元素。然而，在三次循环之后，我们已经遍历了所有的元素。因为，range遍历使用的临时切片仍然是一个长度为三的切片，因此循环在三次迭代之后完成。这样的情况和传统的for循环不同

```go
s := []int{0,1,2}
for i :=0 i < len(s); i++{
  s = append(s, 10)
}
```

​	在上面的示例中，循环永远不会结束，因为len(s)的表达式在每次迭代的时候都会重新被计算，并且因为我们不断的添加元素，所以迭代永远不会被终止。牢记这种差异对准确使用Go的循环非常重要。

​	回到range循环，我们应该知道我们描述的行为（表达式只计算一次）适用于提供的所有数据类型，我们来看看这样的行为对于通道和数组的影响

### channels

​	我们来看看基于range循环遍历通道的具体示例。我们创建了两个goroutine，它们都将元素发送到不同的通道。然后，在父goroutine中，我们使用range循环实现一个消费者，该range循环尝试在迭代期间切换到另一个通道

```go

func main() {
	ch1 := make(chan int, 3)
	go func() {
		ch1 <- 1
		ch1 <- 2
		ch1 <- 3
		close(ch1)

	}()

	ch2 := make(chan int, 3)

	go func() {
		ch2 <- 11
		ch2 <- 12
		ch2 <- 13
		close(ch2)
	}()

	ch := ch1

	for v := range ch {
		fmt.Println(v)
		ch = ch2

	}

}
```

​	在这个示例中，提供给range的是指向ch1的ch通道。因此，range是计算ch，并且将其复制到临时变量，并迭代来自该通道内的元素。尽管有`ch=ch2`，但是range迭代的还是ch1，而不是ch2

```go
0
1
2
```

​	但是ch=ch2语句并非没有效果。由于我们将ch分配给第二个变量，如果我们在这段代码之后调用close(ch)，他将关闭第二个通道，而不是第一个

​	现在让我们看看range的计算在数组上的使用影响

### 数组

​	range循环的计算对数组会有什么影响，因为range的表达式是在循环开始前进行计算的，所以分配的临时循环变量是数组的副本，让我们通过下面的示例来更详细的了解

```go
package main

import "fmt"

func main() {
	a := [3]int{0, 1, 2}
	for i, v := range a {
		a[2] = 10
		if i == 2 {
			fmt.Println(v)
		}
	}
}

```

​	上面的代码将数组的最后一个元素值更新为10。然而，在我们运行代码之后，我们可以看到打印的结果并不是10，仍然是2。具体的原因我们可以通过下面的图示来了解。

![image-20230106153206671](https://img.ethanleo.top/uPic/image-20230106153206671.png)

​	

​	正如我们所看到的那样，range循环会将数组进行一个复制。然而，循环并没有更新复制的数组，循环中更新的是原数组a。因此，v在最后的输出中依然是2，而不是10

​	如果我们还是想修改最后一个元素，我们可以有其他的方式

* 使用数组指针

```go
a := [3]int{0,1,2}
for i, v := range &a {
  a[2] = 10
  if i == 2 {
    fmt.Println(v)
  }
}
```

​	使用指针，那么在range迭代的过程中，虽然会进行复制，但是指针指向的都是相同的底层数组，所以我们进行修改的时候底层的数组元素值也会更新

* 通过索引访问元素

```go
a := [3]int{0,1,2}
for i, v := range &a {
  a[2] = 10
  if i == 2 {
    fmt.Println(a[2])
  }
}
```

因为我们访问的是原始数组，所以打印出来的是10而不是2

以上的两种选项都是有效的，然而，第一个选择并不会复制整个数组，如果数组非常大，需要注意这一点

## 忽略在range中指针元素的影响

​	本节着眼于使用带指针元素的范围循环时的一个特定错误。 如果我们不够谨慎，可能会导致引用错误元素的问题。 让我们检查一下这个问题以及如何解决它。
在我们开始之前，让我们澄清一下使用指针元素的切片或映射的基本原理。 主要有以下三种情况：

*  在语义方面，使用指针语义存储数据意味着共享元素。 例如，以下方法包含将元素插入缓存的逻辑：

```go
type Store struct {
  m map[string]*Foo
}

func (s Store)Put(id string, foo *Foo){
  s.m[id] = foo
}
```

* 有时我们已经在操纵指针。 因此，直接在我们的集合中存储指针而不是值会很方便。
* 如果我们存储大型结构，并且这些结构经常发生变化，我们可以使用指针避免每次变化时候的复制和插入

```go
func updateMapValue(mapValue map[string]LargeStruct, id string){
  value:= mapValue[id]
  value.foo = "bar"
  mapValue[id] = value
}

// 存储使用指针作为value
func updateMapPointer(mapPointer map[string]*LargeStruct, id string){
  mapPoint[id].foo = "bar"
}
```

因为updateMapPointer接收的是值为指针的映射，所以可以一步完成对foo字段的修改

现在，我们可以讨论在range循环中，指针元素常见的错误，我们考虑以下两个结构：

* 表示客户的Customer结构
* 保存客户指针的Store结构

```go
type Customer struct{
  ID string
  Balance float64
}

type Store struct{
  
  m map[string]*Customer
}
```

接下来Store中会有一个方法将一个Consumer的切片，进行存储

```go
func (s *Store)storeCustomer(customers []Customer){
  for _,customer :=range customers{
    s.m[customer.ID] = &customer
  }
}
```

在此示例中，我们使用range将传入的Customer切片中每个元素的指针存在map中。但是这样的做法是否符合我们的预期。

让我们创建三个不同的Customer并且调用以下该方法

```go
s.storeCustomers([]Customer{
  	{ID: "1", Balance: 10},
    {ID: "2", Balance: -10},
    {ID: "3", Balance: 0},
})
```

将store中的m打印出来的结果是:

```go
func main() {
	s := Store{}
	s.m = make(map[string]*Customer)
	s.storeCustomer([]Customer{{ID: "1", Balance: 10},
		{ID: "2", Balance: -10},
		{ID: "3", Balance: 0}})

	for k, v := range s.m {
		fmt.Printf("key %s,value %v \n", k, *v)
	}
}

```

```sh
key 3,value {3 0} 
key 1,value {3 0} 
key 2,value {3 0} 

```

正如我们所见，所有存入的map的元素都是相通的Cutsomer struct:3 ，而不是不同的三个Customer结构体，为什么会这样的呢？

使用range循环遍历customers切片，不管元素的数量有多少，都会创建一个具有固定地址的customer变量，我们可以通过在迭代期间打印指针地址来验证这一点

```go

func (s *Store) storeCustomer(customers []Customer) {
	for _, customer := range customers {
		fmt.Printf("%p \n", &customer) // 打印customer地址
		s.m[customer.ID] = &customer
	}
}

```

```sh
0x1400011a018 
0x1400011a018 
0x1400011a018 

```

为什么这个如此重要？让我们来检查每一次的迭代：

* 在第一次迭代时，customer指向第一个元素,Customer 1. 我们将其存入customer结构体的指针

* 第二次迭代的时候，customer现在指向了另一个元素：Customer 2，我们还是将customer的指针存入了map

* 最后，当我们在进行最后一次迭代的时候，customer指向了最后一个元素，Customer 3，最后一次，情况还是一样的，我们将customer的指针存入了map

  

<img src="/Users/lizi/Library/Application Support/typora-user-images/image-20230209203848296.png" alt="image-20230209203848296"  />

​	(其实理解起来就是，每次存储的都是customer的地址，而不是迭代过程中的每个元素的地址。然而在range循环中customer是固定的)

那么我们应该如何解决这个问题呢？这里有两个结局方案。第一个解决方案有点像第一章中的“变量阴影”。需要我们创建一个本地的变量

```go
func (s *Store)storeCustomers(customers []Customer){
  
  for _,customer:=range customers {
    current := customer
    s.m[current.ID] = &current
  }
}
```

​	在这个示例中，我们不存储customer的指针引用，我们存储的是current的指针引用。current在每次的迭代中都是一个指向唯一的Customer的变量。因此，根据上面的解决方案中，我们就可以将不同的Customer结构体指针存入map中。另一个解决方案就是在我们迭代的过程中使用切片索引

```go
func (s *Store)storeCustomers(customers []Customer){
  for i := range customers{
    customer := &customers[i]
    s.m[customer.ID] = customer
  }
}
```

​	在这个解决方案中，customer是一个指针。因为在每次迭代中都会进行初始化，所以这个也是一个唯一的地址，因此我们可以将不同的指针存入map

​	当我们在使用range进行对数据进行迭代的时候，我们必须记住所有的值都会指向一个地址固定的变量。因此，如果我们需要在每次迭代的过程中存储迭代中value元素的指针，我们最终都只会存储相同的指针，并且改指针指向的是切片中最后一个元素

## 在map的迭代中做出错误的假设

​	在进行map的迭代过程中会出现很多的误解和错误，主要是因为开发人员对迭代做出了错误的假设。

* 排序
* 迭代过程中的更新

### 排序

​	关于排序，我们需要了解map数据结构中的一些基本行为

* map不会对key进行排序（map不是基于二叉树）
* map也不会按照添加的顺序进行数据的保留，比如，我们在B之前插入A，数据保存的时候不一定B就排在A之前

​	此外，我们在遍历map的时候，我们不应该做出任何排序的假设。

<img src="/Users/lizi/Library/Application Support/typora-user-images/image-20230220212512034.png" alt="image-20230220212512034" style="zoom:50%;" />

​	现在，让我们来使用`range`来对map进行遍历

```go
for k:= range m {
  fmt.Print(k)
}
```

​	我们提到map中的数据不是按键排序的，因此我们不能期望代码打印出来的是`acdeyz`，同时，map也不是按照插入的顺序进行排序的，所以我们也不能期望代码打印出来的是`ayzcde`

​	但是我们可以期望代码按照当前键存储的顺序被输出吗？答案是不行，在Go中，并没有指定map的迭代顺序，也不能保证从一个迭代到下一个迭代的顺序是相同的，所以我们不能对迭代做基于键排序的假设

​	那为什么Go会选择以这样的方式来遍历map呢？这是语言的设计者有意为之，它们像增减某种形式的随机性，以确保开发人员在使用map的时候不会依赖任何顺序的假设(http://mng.bz/M2JW)

​	因此，作为Go开发人员，我们永远不应该在map遍历的时候就顺序做出任何假设。但是，请注意，但是，请注意，使用标准库或外部库中的包可能会导致不同的行为。例如，当 encoding/json 包将映射编组为 JSON 时，它会按键的字母顺序对数据重新排序，而不管插入顺序如何。但这不是 Go 地图本身的属性。 如果需要排序，我们应该依赖其他数据结构，例如二进制堆（位于 https://github.com/emirpasic/gods 的 GoDS 库包含有用的数据结构实现）。

### 迭代期间map的插入

​	在 Go 中，允许在迭代期间更新map（插入或删除元素）； 它不会导致编译错误或运行时错误。 然而，在迭代期间在map中添加条目时，我们应该考虑另一个方面，以避免不确定的结果。
​	让我们检查以下迭代 map[int]bool 的示例。 如果 pair 值为真，我们添加另一个元素。 你能猜出这段代码的输出是什么吗？

```go
m:=map[int]bool{
  0: true,
  1: false,
  2: true,
}


for k,v:=range m {
  if v {
    m[10+k] = true
  }
}

fmt.Println(m)
```

​	此代码的运行结果是不可预测的，如果我们多次运行这段代码，可能会得到下面的结果

```she
map[0:true 1:false 2:true 10:true 12:true]
map[0:true 1:false 2:true 10:true 12:true 22:true 32:true 42:true 52:true 62:true]
map[0:true 1:false 2:true 10:true 12:true 22:true 32:true 42:true 52:true 62:true]

```

​	要理解原因，我们必须阅读Go规范中关于迭代期间向map中添加新的k-v的内容

*If a map entry is created during iteration, it may be produced during the iteration or skipped. The choice may vary for each entry created and from one iteration to the next.*

​	因此，当一个元素在迭代期间被添加到map时，他可能会存放在还没迭代到的元素后边，也可能不会。作为Go的开发人员，我们没有任何方法强制让新添加的K-V存放在已经迭代完的元素前面，所以，在运行上面的代码的时候，我们才会得到不同的结果。

​	我们必须牢记这样的情况，以确保我们的代码不会产生任何不可预测的输出。如果我们想在迭代map的时候，同时更新map中的k-v并确保添加的k-v不会是迭代的一部分，我们可以通过创建map的副本来解决，比如：

```go
m:=map[int]bool{
  0: true,
  1: false,
  2: true,
}

m2:=copyMap(m)

for k,v := range m {
  m2[k] = v 
  if v {
    m2[10+k] = true
  }
}

fmt.Println(m2)
```

​	在此示例中，我们取消了正在读取的map与需要更新的map的关联。我们不断的迭代m，但是更新是在m2上完成的

```sh
map[0:true 1:false 2:true 10:true 12:true]
```

​	总而言之，当我们使用map的时候，我们不应该依赖一下内容：

* 按照key的顺序进行排序
* 按照插入顺序排序
* 迭代顺序固定
* 在迭代map的过程中对迭代的map进行插入

​	记住这些行为有助于我们避免一些基于错误假设的常见错误

## 忽略break语句的工作原理

​	break 语句通常用于终止循环的执行。 当循环与 switch 或 select 一起使用时，开发人员经常会犯下break语句的错误。

​	让我们看看下面的例子。 我们在 for 循环中实现一个开关。 如果循环索引的值为 2，我们要中断循环：

```go
for i := 0; i < 5; i++ {
		fmt.Printf("%d ", i)
		switch i {
		default:
		case 2:
			break
		}
}
```

​	这段代码乍一看可能没有问题，然而，它并没有达到我们的预期，break语句不会终止for循环，它会终止switch语句。因此，这段代码不会是从0迭代到2，而是从0迭代到4。需要注意的是，break语句终止最内层的for、switch或select语句的执行。在前面的示例中，他终止了switch语句。那么我们该如何编写打破循环的代码，而不是switch语句呢？最常用的方法是使用标签：

```go
loop:
	for i := 0; i < 5; i++ {
		fmt.Printf("%d ", i)
		switch i {
		default:
		case 2:
			break loop
		}
	}
```

​	在这里，我们将循环标签与 for 循环相关联。 然后，因为我们为 break 语句提供了循环标签，它会中断循环，而不是开关。 因此，正如我们预期的那样，这个新版本将打印 0 1 2

​	循环内的 select 也可能导致错误的语句中断。 在下面的代码中，我们希望在两种情况下使用 select 并在上下文取消时中断循环：

```go
for {
		select {
		case <-ch:
			// Do something
		case <-ctx.Done():
			break
		}
	}
```

 	这里出现的问题和之前其实是一样的，break语句终止的是select，并不是最外层的for，同样，如果要终止最外层的for，我们一样可以使用label来进行操作

```go
loop:
  for {
      select {
      case <-ch:
        // Do something
      case <-ctx.Done():
        break loop
      }
    }
```

*我们在使用`continue`的时候使用label标签来进行下一次迭代*

​	在循环中使用 switch 或 select 语句时，我们应该保持谨慎。 使用 break 时，我们应该始终确保我们知道它会影响哪条语句。 正如我们所见，使用标签是强制破坏特定语句的惯用解决方案。
​	在本章的最后一节中，我们继续讨论循环，但这次是结合 defer 关键字。

## 在循环中使用defer

​	defer 语句延迟会在函数返回时执行。 它主要用于减少样板代码。 例如，如果资源最终必须关闭，我们可以使用 defer 来避免在每次返回之前重复关闭调用。 然而，一个常见的错误是没有意识到在循环内使用 defer 的后果。 让我们来看看这个问题。
​	我们将实现一个打开一组文件的函数，其中文件路径是通过通道接收的。 因此，我们必须遍历此通道、打开文件并处理关闭。 这是我们的第一个版本

```go
func readFiles(ch <-chan string)error{
  for path := range ch {
    
    file, err:= os.Open(path)
    if err!=nil{
      return err
    }
    defer file.Close()
    // Do something with file
  }
  
  return nil
}
```

​	此实现存在重大问题。我们需要知道defer语句是在整个函数返回的时候才被调用。在这种情况下，延迟调用不是在每次的迭代过程中执行，而是在readFiles函数返回的时候执行。如果readFiles没有返回，文件描述符将永远保持打开的状态，从而导致泄漏。解决此问题的方式有哪些？一种是通过手动处理文件关闭，而不是使用延迟语句。但如果我们这样做，我们将不得不放弃Go工具集的一个方便的功能，因为我们处于循环中。那么，如果我们想继续使用defer的话，有什么其他的选择呢？我们必须围绕defer创建另一个在每次迭代期间调用的函数。例如，我们可以实现一个readFile函数，其中包含接收到的每个新文件路径的逻辑：

```go
func readFiles(ch <-chan string)error{
  for path := range ch {
    if err:=readFile(path);err!=nil{
      return err
    }
  }
  return nil
}


func readFile(path string) error {
   file, err:= os.Open(path)
    if err!=nil{
      return err
    }
    defer file.Close()
   // Do something with file
  return nil 
}
```

​	在此实现中，延迟函数在readFile返回时被调用，这意味着在每次迭代结束时。因此，在父readFiles函数返回之前，我们不会保持文件描述符打开。

​	另一种方法是使得readFile函数成为一个闭包：

```go
func readFiles(ch <-chan string) error {
	for path := range ch {
		err := func() error {
			// ....
			defer file.close()
			// ....
		}()
		if err != nil {
			return err
		}
	}
	return nil
}
```

​	但从本质上来讲，这依然是相同的解决方案：添加另一个外围函数在每次迭代期间执行延迟调用，第一种的解决方案看起来更加清晰一些，我们还可以为它编写特定的单元测试。

​	使用defer时，我们必须他会在包裹的函数返回的时候被调用。因此，在循环中调用defer只会将defer叠加起来，并不会在每次迭代期间执行。如果循环没有终止，看你会造成内存泄漏。
