
# 代码和项目结构

本章包含

* 确定代码风格
* 有效的使用接口和泛型
* 构建项目的最佳实践

<!--more-->

构建一个干净的，可维护的go代码库并不是简单的事情。需要我们拥有一定的经验甚至经历一些错误才能去理解所有代码或者项目的最佳实践。那么有哪些坑需要我们去避免（例如：变量遮蔽、代码随意的嵌套）？我们如何去组织包结构？我们应该在什么时候，什么地方使用接口和泛型，初始化函数以及工具包？在这一章节，我们会介绍一些常见的错误

## 意外的变量遮蔽

​	变量的作用域是指变量可以被应用的地方。换句话说.... 在go中，一个变量可以在不同的代码块中重复声明，这就是**变量遮蔽**。这也是一个常见的错误

​	下面的例子就简单的演示了变量遮蔽的情况

```go
var client *http.Client // 声明了一个client变量

	if tracing {
		client, err := createClientWithTracing() // client 变量在这个代码块中被遮蔽了
		if err != nil {
			return err 
		}
		log.Println(client)
	} else {
		client, err := createDefaultClient()  // client 变量在这个代码块中被遮蔽了
		if err!=nil{
			return err
		}
		log.Println(client)
	}
	
	// Use client
```

​	在这个示例中，我们首先定义了一个client的变量，然后我们通过(:=)在if-else代码块中将变量声明，并且将结果给到了内部的client变量，这就导致了外部的client变量将一直都是nil。

​	我们如何才能确认外部声明的client变量能够正确被分配到值呢？这里有两种不同的方式

​	第一种方式是通过在内部的代码块中声明零时变量

```go
var client *http.Client
if tracing{
  client, err := createClientWithTracing()
  if err != nil {
			return err 
		}
  client = c
} else {
  // 一样的逻辑
}
```

​	我们将结果分配给一个临时变量`c`,`c`的作用域只在if这个代码块内。然后，我们将返回的值分配给`client`。同时，我们在else代码块中，进行同样的逻辑

​	第二种方式是在内部的代码块中使用赋值运算符(=)，将函数返回的值分配给`client`。但是，使用这样的方式需要我们额外创建一个`error`的变量，因为赋值运算符只允许我们将值分配给已经声明好的变量

```go
// 声明变量
var client *http.Client
var err error 
if tracing{
  client,err = createClientWithTracing()
  if err != nil{
    return err
  }
} else {
   // 一样的逻辑
}
```

​	这样的方式可以让我们直接将返回值给到`client`，而不需要额外声明临时变量

​	以上的两种方式都是十分有效的。在以上两种方式的选择上，第二种方式的做法让我们只需要进行一次变量的赋值，这样可以提高代码的可读性。此外，我们可以在if-else代码块之外，对错误(`error`)进行处理，例如

```GO
if tracing {
  client,err = createClientWithTracing()
} else {
  client, err = createDefaultClient()
}
if err != nil {
  // 对错误进行处理
}
```

​	当变量名在内部的一些代码块中被重复声明的时候，就会发生**变量遮蔽**，但是我们可以看到这样的做法很容易出错。是否禁止这样的编码方式取决于个人的习惯。例如：有时候重用现有的变量名如`err`会很方便，但是总的来说，我们对这样的写法应该持有谨慎的态度。因为对于重复的变量声明是可以通过编译的，但是我们变量接收到的值是不可预期的

## 不必要的代码嵌套

​	在编程时，我们需要维护心智模型（例如，关于整体代码交互和功能实现）。基于命名、一致性、格式等多个标准，代码被限定为可读的。可读的代码需要较少的认知努力来维持心智模型；因此，它更容易阅读和维护。

<!--**心理模型**是对某人关于现实世界中某事如何运作的[思考](https://en.wikipedia.org/wiki/Thought)过程的解释。它代表了周围的世界、世界各个部分之间的关系以及一个人对自己的行为及其后果的直觉感知。心智模型可以帮助塑造[行为](https://en.wikipedia.org/wiki/Behaviour)并设定解决问题（类似于个人[算法](https://en.wikipedia.org/wiki/Algorithm)）和完成任务的方法。-->

​	代码可读性的关键在于嵌套的数量，让我们来看看，假如我们处理一个新的项目，并且需要了解`join`函数的作用

```go
 func join(s1, s2 string, max int) (string, error) {
   if s1 == "" {
      return "", errors.New("s1 is empty")
   }else {
     if s2 == "" {
        return "", errors.New("s2 is empty")
     }else {
       concat, err := concatenate(s1, s2) // 调用concatenate函数，执行一些逻辑，可能会返回错误
       if err != nil{
         return "" , err
       } else {
         if len(concat) > max {
        	return concat[:max], nil
         } else {
           return concat, nil
         } 
       }
     }
   }
 }


func concatenate(s1 string, s2 string) (string, error) {
    // ...
}
 
```

​	`join`函数接收了两个string类型的参数，并且将其传递给`concatenate`,如果`concatenate`返回string长度大于`max`,那么就将该返回值进行切割。但是在`join`函数中不仅校验了s1，s2，同时还校验了`concatenate`返回的错误信息

​	从代码实现的层面来说，上面的实现的功能是正确的，但是创建一个涵盖各种可能性的函数并不是一个容易的事情，因为会嵌套的等级会过多

​	下面，我们尝试将这个案例来进行一个不同的实现

```go
func join(s1, s2 string, max int) (string, error) {
   if s1 == "" {
      return "", errors.New("s1 is empty")
   }
  if s2 == "" {
     return "", errors.New("s2 is empty")
  }
  concat, err := concatenate(s1, s2)
  if err!=nil {
     return "",err
  }
  
  if len(concat) > max {
    return concat[:max] , nil
  }
  return concat , nil
  
 }


func concatenate(s1 string, s2 string) (string, error) {
    // ...
}
 
```

​	你或许注意到了，在新的代码逻辑实现中。我们实现了和之前一样的功能，但是在代码阅读上我们承受了更少的阅读压力。我们只维护了两层的一个代码嵌套。

​	由于嵌套的if-else语句，我们很难在第一个示例中查看整个的代码逻辑。相反，在第二个示例中，我们可以很轻松的查看不同情况的代码预期

​	通常来说，越多的代码嵌套，会让代码的阅读变得难以理解。让我们通过下面两个不同的示例来理解如何去提高我们代码的可读性

​	当if代码块进行了函数的返回时，我们应该省略else代码块的编写

```go
// bad
if foo() {
  // ....
  return true 
} else {
  // ...
}

// good
if foo() {
  return true
}
// ...
```

我们可以通过判断不满足的情况，来省略else代码块的编写

```go
 // bad
if s != ""{
  // ...
} else {
  return errors.New("empty string")
}

// good
if s == "" {
  return errors.New("empty string")
}

// ....

```

​	编写出阅读性高的代码，对于每个开发人员来说都是需要面临的重要挑战。尽可能的去减少代码嵌套的层级，同时提前进行函数返回是提高可读性的重要方式。

## 错误的使用init函数

​	有时，在go的应用中我们会错误的使用init函数。这样会导致我们对错误的处理不够妥善，同时会让代码的阅读难度提高，接下来我们来重新认识一下init函数，并且我们将了解到何时推荐使用init函数，何时不推荐使用

### 概念

​	init函数是一个在程序初始化状态下被执行的函数。该函数不需要任何参数，也不会有任何的返回值。当一个包被加载的时候，首先会将包内的常量和全部变量进行初始化，然后init函数就会被执行。下面是一个`main`包初始化的示例

```go
package main

import "fmt"

var a  = func() init {  // 第一个被执行
  fmt.Printlb("var")  
  return 0
}()

func init() {
  
  fmt.Println("init")  // 第二个被执行
  
}


func main() {
  fmt.Println("main") // 最后被执行
}
```

​	以上的示例输出的顺序是

```shell
var
init
main
```

​	当一个包被初始化的时候`init`函数将会被执行。下面的示例中，我们定义了两个包，`main`和`redis`,在`main`包中导入`redis`包

```go
package main

import (
	"fmt"
  
  "redis"
)

func init(){
  // ...
}

func main(){
  err := redis.Store("foo","bar") // 这里依赖了redis包
}

```

```go
package redis

// imports


func init() {
   // ...
}

func Store(key , value string ) error {
  // ...
 
}

```

<img src="https://img.ethanleo.top/uPic/image-20221208220927402.png" alt="image-20221208220927402" style="zoom:30%;" />

​	因为`main`包中依赖了`redis`包，所以`redis`包会先被加载，同时,`redis`包里的`init`函数会被先执行，紧接着`main`包里的`init`函数才会被执行

​	我们可以在每个包里定义`init`函数，每个包里的`init`函数执行顺序会根据源文件的字母顺序。例如：一个包内包含了a.go和一个b.go，并且这两个go文件都有一个`init`函数，那么a.go内的`init`函数会被先执行

​	我们不应该依赖包内的`init`函数的执行顺序，因为源文件可以重命名，这样的话就会影响执行顺序

​	我们也可以在一个源文件中定义多个`init`函数，例如：

```go
package main

import "fmt"

func init() {
  fmt.Println("init 1")  // 第一个init函数
}

func init() {
  fmt.Println("init 2") // 第二个init函数
}


func main() {
  
}

```

​	这个时候，`init`函数执行顺序会根据源文件中的`init`函数定义顺序来执行，以下是输出:

```go
init 1
init 2
```

​	当我们只想让某个包中的`init`函数执行时，我们可以通过在一个包中对另一个包非强依赖，来使得另一个包的`init`函数被执行，例如：

```go
package main

import (
	"fmt"
  
  _ "foo" // 导入但并不适用
)

func init() {
  fmt.Println("main init")
}

func main() {
  // ...
}
```

```go
package foo

func init() {
  fmt.Println("foo init")
}
```

​	通过上面的方法，我们可以让`foo`包在`main`包之前被初始化,下面是输出：

```shell
foo init
main init
```

​	另外，`init`函数是不能被显式调用的,例如:

```go
package main

func init() {}

func main() {
  init()
}
```

​	上面的代码,会导致编译错误

```shell
$ go build .
./main.go:13:2: undefined: init
```

​	现在，我们对`init`函数有了新的认识，并且也了解了`init`函数的工作方式。接下来，让我们来看看我们什么时候应该使用，什么时候不应该使用。

### 何时使用init函数

​	首先，我们通过一个不适合使用`init`函数的示例：示例中持有一个数据库连接池。在示例中的 init 函数中，我们使用 sql.Open 打开一个数据库。我们使这个数据库成为其他函数以后可以使用的全局变量：

```go
var db *sql.DB


func init() {
  dataSourceName := os.Getenv("MYSQL_DATA_SOURCE_NAME")
  d,err := sql.Open("mysql",dataSourceName) 
  if err !=nil{
    log.Panic(err)
  }
  
  err = d.Ping()
  if err!=nil{
    log.Panic(err)
  }
  
  db = d // 将DB的链接信息赋值给全局的db变量
  
}
```

​	上面的示例中，我们创立了一个数据的链接，并且检查是否可以ping通。然后，我们将获取到的链接赋值给全局的变量。我们应该如何去看待这个逻辑？让我们来看看以上的方式有哪三个主要的缺点。

​	首先，对于错误的处理在`init`函数中是受限的。实际上，`init`函数并不会给我们返回任何的错误信息。我们能收到错误信息唯一的方式就是进行程序的panic，在我们上面的示例中，当我们打开数据库出现错误的时候，程序的停止是可以被接受的。然而我们并不会很希望程序自己去决定何时停止应用。函数的调用者可能更喜欢实现一些重试机制或者使用回退机制。在这种情况下，`init`函数会组织客户端包实现他们对错误逻辑的处理。

​	另一个重要的缺点与测试有关。如果我们在这里添加测试文件，那么`init`函数就会在测试代码执行前被执行，这样的做法并不是必需的（比如说我， 添加的单元测试的方法并不依赖数据库连接的创立）,因此`init`函数在这个示例中会让单元测试变得复杂

​	最后一个缺点就是在这个示例中需要将数据库连接池分配给全局变量，然而全局变量是会有一些严重的缺点

* 任何函数都可以改变包内的全局变量
* 单元测试可能会变的更复杂，因为一个函数都会因为依赖全局变量而不再是孤立的，耦合度变高

​	在大多数的情况下，我们应该将变量进行封装，而不是将其保存在全局

​	根据以上的原因，对于上述的初始化操作，我们应该将其放在一个普通的函数中去实现

```go
func createClient(dsb string) (*sql.DB , error) {
  db, err :=sql.Open(*mysql,dsn)
  if err != nil {
    return nil , err
  }
  if err = db.Ping(); err != nil {
    return nil , err
  }
  return db , err
}
```

​	通过这样的方式，我们解决了之前所提到的缺点。

* 上面的方式可以让错误的处理留给调用者
* 我们可以创建一个集成测试来验证上面的功能是否有效
* 连接池被封装在函数内部

​	我们是否应该所有的`init`函数的调用呢？其实并不是这样的，在某些情况下，`init`函数的使用还是很有帮助的，例如在([官方博客](http://mng.bz/PW6w))中使用了`init`函数去设置静态资源服务器

```go

func init() {
	// Redirect "/blog/" to "/", because the menu bar link is to "/blog/"
	// but we're serving from the root.
	redirect := func(w http.ResponseWriter, r *http.Request) {
		http.Redirect(w, r, "/", http.StatusFound)
	}
	http.HandleFunc("/blog", redirect)
	http.HandleFunc("/blog/", redirect)

	// Keep these static file handlers in sync with app.yaml.
	static := http.FileServer(http.Dir("static"))
	http.Handle("/favicon.ico", static)
	http.Handle("/fonts.css", static)
	http.Handle("/fonts/", static)

	http.Handle("/lib/godoc/", http.StripPrefix("/lib/godoc/", http.HandlerFunc(staticHandler)))
}
```

​	在上面的示例中，`init`函数不会失败(http.HandleFunc 会panic，但是只有handle为nil的时候才会触发，但是这样的情况在上面的示例中并不存在)，也就是说，我们不需要创建任何的全局变量。这个函数也不会影响单元测试。因此，这段代码给我们很好的展示了什么时候使用`init`函数是会比较有帮助的。总而言之，我们知道了初始化函数会导致的一些问题

* 对于错误的处理会收到限制
* 可能会让单元测试变得复杂
* 如果初始化某个东西去设置一个状态，通过初始化函数来做，必须依靠一个全局变量才能解决

​	我们应该谨慎使用初始化函数。虽然在某些情况下初始化函数很有用（例如定义一些静态的配置），但是大多数情况下，我们应该通过一些特定的函数来处理初始化

## 过度使用 Getters 和 Setters

​	在很多程序中，有一些变量会被封装在对象当中。Getters和Setters方法可以让被封装的对象字段暴露出来。在Go中，并没有向其他语言一样原生支持getters和setters，因为使用getter和setter来访问结构体中的字段并不被认为是强制性的也不是惯用的。例如，在标准库中实现了可以直接访问的某些字段，就像`time.Time`中

```go
time := time.NewTimer(time.Second)
<- time.C.   // 这里的C是 Time结构体中的一个只读通道
```

​	即使这样的做法并不推荐，因为我们可以直接修改C这个字段（如果修改了那么就不会在收到事件了）。但是这个示例说明了即使针对我们不应该修改的字段，在标准库中也并没有强制使用getter或setter方法

​	另一方面，使用getter/setter方法也会有一些优势，例如：

* 它们封装了获取或设置字段的相关行为，允许在后期正对字段添加新的功能(例如，验证字段，返回计算值或将字段进行上锁)
* 它们隐藏了内部的行为，使得我们在公开内容的时候可以更具有灵活性
* 它们为属性的修改提供了调试的断点，运行时，使得调试更加的容易

​	如果我们在保证向前兼容的情况下陷入这些情况或者预见到一个可能的用例，那么使用 getters 和 setters 可以带来一些价值。例如，如果我们将它们与一个名为 balance 的字段一起使用，我们应该遵循以下命名约定

* getter方法应该被称为Balance而不是(GetBalance)
* setter方法应该被称为SetBalnace

```go
currentBalance := customer.Balance()  // Getter 
if currentBalance < 0 {
	customer.SetBalance(0)  // Setter
}

```

​	总而言之，如果getter/setter不能带来任何价值，那么我们不应该使用过多的getter/setter。我们更加应该务实，在编程效率和遵循其他编程范式中找平衡

​	请记住，Go是一个独特的语言，专门为许多特性而设计，包括简单性。但是，如果我们发现需要使用getter或setter的时候，可以保证向前兼容的同时能够预见未来的需求，那么使用它们并没有错

​	接下来，我们将会讨论过度适应接口的问题

## 接口污染

​	在设计和构建我们的代码时，接口是Go语言的基石之一。然而，像许多工具或者概念一样，滥用它们并不是一个很好的注意。接口污染其实就是在我们的代码过度不必要的抽象，导致我们的代码难以理解。这是具有不用习惯的另一种语言的开发人员常犯的错误。根据我们之前章节的习惯，让我们重新刷新对Go中接口的认识，并且了解我们应该如何更好的使用接口，知道什么时候会产生接口污染

### 概念

​	接口提供了一种指定对象行为的方法。我们使用接口去定义多种提供给多个对象实现的公共抽象。与其他语言不同的是Go的接口是隐式实现的。它并没有提供一些关键词（像 implements）去标记某个对象实现了某个接口

​	为了理解为什么接口具有如此强大的作用，我们将深入两个标准库中去查看接口的设计：`io.Reader`和`io.Writer`。在`io`包中提供了对于I/O操作的抽象。在这些抽象中，`io.Reader`定义了一些从数据源读取数据的方法,而`io.Writer`则是定义了一些将数据写入指定位置的方法

![image-20221210145719795](https://img.ethanleo.top/uPic/image-20221210145719795.png)

​	`io.Reader`中只包含了一个Read方法

```go
type Reader interface {
  Read(p []byte) (n int, err error)
}
```

​	实现`io.Reader`接口需要接受一个byte的切片，将数据写入切片中，并且返回写入的字节数或错误

​	另一方面`io.Writer`也定义了一个单独的Write方法

```go
type Writer interface {
  Write(p []byte) (n int, err error)
}
```

​	`io.Writer`的自定义实现应该将来自切片的数据写入目标并返回写入的字节数或错误。因此，这两个接口都提供了基本的抽象：

* io.Reader从数据源读取数据
* io.Writer将数据写入目标

​	那么为什么要定义这两个接口？创建这些抽象有什么意义？

​	假设我们需要实现一个函数，将一个文件的内容复制到另一个文件，我们可以创建一个将两个 *os.Files 作为输入的特定函数。或者，我们可以选择使用 io.Reader 和 io.Writer 抽象来创建更通用的函数

```go
func copySourceToDest(source io.Reader, dest io.Writer) error { 
  // ...
}
```

​	此函数将使用 *os.File 参数（因为 *os.File 实现了 io.Reader 和 io.Writer）以及任何其他将实现这些接口的类型。例如，我们可以创建自己的 io.Writer 来写入数据库，代码将保持不变。它增加了功能的通用性，它的可重用性。此外，为这个函数编写单元测试更容易，因为我们不必处理文件，而是可以使用提供有用实现的字符串和字节包

```go
func TestCopySourceToDest(t *testing.T) {
    const input = "foo"
    source := strings.NewReader(input) // 创建一个io.Reader
    dest := bytes.NewBuffer(make([]byte, 0)) // 创建一个io.Writer
    err := copySourceToDest(source, dest)
    if err != nil {
        t.FailNow()
    }
    got := dest.String()
    if got != input {
        t.Errorf("expected: %s, got: %s", input, got)
    }
}
```

​	在示例中，source 是一个 *strings.Reader，而 dest 是一个 *bytes.Buffer。在这里，我们在不创建任何文件的情况下测试 copySourceToDest方法

​	实际上，向接口添加方法会降低其可重用性级别。 io.Reader 和 io.Writer 是强大的抽象，因为它们再简单不过了。此外，我们还可以结合细粒度的接口来创建更高级别的抽象。 io.ReadWriter 就是这种情况，它结合了Reader和Writer的行为

```go
type ReadWriter interface {
    Reader
		Writer
}
```

​	现在，让我们讨论一下那些情况下推荐使用接口

### 什么时候使用接口

​	我们何时应该在Go中创建接口？让我们看看下面三个带来价值的具体案例，下面的三个案例给了我们一个大概的概念：

* 通用的行为
* 解耦
* 限制行为

#### 通用的行为

​	我们讨论的第一个情况就是在多种类型实现同样的行为时使用接口。在这种情况下，我们可以分解出接口内的行为。如果我们查看标准库，我们可以找到许多对此类用例的示例。例如，可以通过三种方式对集合进行排序：

* 检索集合中的元素数
* 检查一个元素是否必须排在另一个元素之前
* 交换两个元素数

​	因此，在sort包中添加了下面的接口

```go
type Interface interface{
  Len() int
  Less(i, j int) bool
  Swap(i, j int)
}
```

​	该接口具有强大的可重用性，因为它包含了对任何基于索引的集合排序的常见行为

​	在整个排序包中，我们可以找到几十种实现。例如，如果在某个时候我们计算了一个整数集合，并且我们想要对其进行排序，那么我们是否一定对实现类型感兴趣？排序算法是归并排序还是快速排序重要吗？在很多情况下，我们不在乎。因此，可以抽象排序行为，我们可以依赖于 sort.Interface。
找到正确的抽象来分解行为也可以带来很多好处。例如，sort 包提供了也依赖于 sort.Interface 的实用函数，例如检查集合是否已经排序。例如：

```go
func IsSorted(data Interface) bool {
    n := data.Len()
    for i := n - 1; i > 0; i-- {
        if data.Less(i, i-1) {
            return false
        }
}
	return true
}
```

​	因为sort.Interface是有正确的抽象级别，所以它非常有价值。现在让我们看看使用接口时的另一个主要用例

#### 解耦

​	另一个重要的用例是关于将我们的代码与实现解耦。如果我们依赖抽象而不是具体的实现，那么实现的本身是可以替换为另一个，甚至无需更改我们的代码，这就是里氏替换原则。解耦的另一个好处可能与单元测试有关。假如我们要实现一个创建新客户并存储的CreateNewCustomer方法

```go
type CustomerService struct {
    store mysql.Store 
}

func (cs CustomerService) CreateNewCustomer(id string) error { 
  customer := Customer{id: id}
return cs.store.StoreCustomer(customer)
}
```

​	现在，如果我们想要测试这个方法该怎么办，由于CustomerService依赖于实际实现来存储Customer，我们不得不通过集成测试来测试它，这需要启动一个Mysql的实例（除非我们使用替代技术，例如 go-sqlmock，但这不是本节的范围），尽管集成测试很有帮助，但这并不总是我们想要做的。为了给我们更多的灵活性，我们应该将 CustomerService 与实际实现分离，这可以通过如下接口来完成：	

```go
type customerStorer interface {
    StoreCustomer(Customer) error
}

type CustomerService struct {
    storer customerStorer // 这里依赖的是接口，具体的接口实现可以多种
}

func (cs CustomerService) CreateNewCustomer(id string) error { 
  customer := Customer{id: id}
	return cs.storer.StoreCustomer(customer)
}

```

​	因为现在存储Customer是通过接口来完成的，所以这可以使得我们的测试方法更加的灵活，例如：

* 通过集成测试使用具体实现
* 通过单元测试使用模拟

​	现在让我们讨论另一个用例：限制行为

#### 限制行为

​	我们将讨论的最后一个用例乍一看可能非常违反直觉。它是关于将类型限制为特定行为。假设我们实现了一个自定义配置包来处理动态配置。我们通过 IntConfig 结构为 int 配置创建一个特定的容器，该结构还公开了两个方法：Get 和 Set。该代码如下所示：

```go
type IntConfig struct {
    // ...
}
func (c *IntConfig) Get() int {
    // Retrieve configuration
}
func (c *IntConfig) Set(value int) {
    // Update configuration
}
```

​	现在，假设我们收到一个 IntConfig ，其中包含一些特定的配置，例如阈值。然而，在我们的代码中，我们只对检索配置值感兴趣，并且我们不想更新它。如果我们不想更改我们的配置包代码，我们如何从语义上强制执行此配置是只读的？通过创建将行为限制为仅检索配置值的抽象

```go
type intConfigGetter interface {
    Get() int
}
```

​	然后，在我们的代码中，我们可以依赖intConfigGetter而不依赖具体的实现

```go
type Foo struct {
    threshold intConfigGetter
}

func NewFoo(threshold intConfigGetter) Foo {
    return Foo{threshold: threshold}
}


func (f Foo) Bar()  {
	threshold := f.threshold.Get()
  // ... 
}
```

​	在此示例中，配置 getter 被注入到 NewFoo 方法中。它不会影响此函数的客户端，因为它在实现 intConfigGetter 时仍然可以传递 IntConfig 结构。那么，我们只能读取Bar方法中的配置，不能修改。因此，我们也可以出于各种原因（例如语义强制）使用接口将类型限制为特定行为

​	在本节中，我们看到了接口通常被视为带来价值的三个潜在用例：分解出常见行为、解耦以及将类型限制为特定行为。 同样，这个列表并不详尽，但它应该让我们大致了解接口在 Go 中何时有用。现在，让我们结束这一节，讨论接口污染问题

### 接口污染

​	在 Go 项目中过度使用接口是很常见的。 也许开发人员的背景是 C# 或 Java，他们发现在具体类型之前创建接口很自然。 然而，这不是 Go 中的工作方式。

​	正如我们所讨论的，接口是用来创建抽象的。在很多时候，我们应该去发现抽象而不是去创造抽象，这也就意味着如果没有直接的理由，我们不应该开始就在我们的代码中去创建抽象。我们不应该设计接口，而是等待具体的需求。换句话说，我们应该在需要时创建接口，而不是预见可能需要接口的时候创建接口

​	如果我们过度使用接口，主要问题是什么？ 答案是它们使代码流更加复杂。 添加无用的间接级别不会带来任何价值； 它创建了一个毫无价值的抽象，使代码更难阅读、理解和推理。 如果我们没有足够的理由添加一个接口，并且不清楚接口如何使代码更好，我们应该考虑为什么要添加这个接口。 为什么不直接调用实现呢？

​	我们不要试图抽象地解决问题，而是解决现在必须解决的问题。 最后但同样重要的是，如果不清楚接口如何使代码更好，我们可能应该考虑删除它以使我们的代码更简单。

## 生产者端接口

​	在上一节中我们了解了接口在何时时被认为有价值的，但是作为Go程序员经常会思考一个问题：接口应该放在哪里？

​	在深入探讨这个主题之前，我们先介绍一些术语：

* 生产端：在与具体实现相同的包中定义的接口

<img src="https://img.ethanleo.top/uPic/image-20221210214703514.png" alt="image-20221210214703514" style="zoom:53%;" />

* 消费端：具体的实现和接口不在同一个包中

<img src="https://img.ethanleo.top/uPic/image-20221210214854510.png" alt="image-20221210214854510" style="zoom:50%;" />



​	开发人员在生产者端创建接口以及具体实现是很常见的。 这种设计可能是具有 C# 或 Java 背景的开发人员的习惯。 但是在 Go 中，大多数情况下这不是我们应该做的。

​	让我们来看看以下示例。在这里，我们创建一个特定的包来存储并检索客户数据。 同时，仍然在同一个包中，我们决定所有调用都必须通过以下接口。

```go
package store

type CustomerStorage interface {
	StoreCustomer(customer Customer) error
	GetCustomer(id string) (Customer, error)
	UpdateCustomer(customer Customer) error
	GetAllCustomer() ([]Customer, error)
	GetCustomerWithoutContract() ([]Customer, error)
	GetCustomersWithNegativeBalance() ([]Customer, error)
}

```

​	我们可能认为我们有一些很好的理由在生产者端创建和公开这个接口。 也许这是将客户端代码与实际实现分离的好方法。 或者，也许我们可以预见到它会帮助客户创建测试替身。 不管是什么原因，这都不是 Go 中的最佳实践。

​	如前所述，接口在 Go 中是隐式满足的，与具有显式实现的语言相比，它往往是游戏规则的改变者。在大多数情况下，遵循的方法类似于我们在上一节中描述的方法:**抽象应该被发现，而不是被创造**。这意味着生产者不能为所有客户强制执行给定的抽象。 相反，由客户决定是否需要某种形式的抽象，然后确定满足其需求的最佳抽象级别。

​	在前面的示例中，也许一个客户端对解耦其代码不感兴趣。 也许另一个客户想要解耦其代码，但只对 GetAllCustomers 方法感兴趣。 在这种情况下，此客户端可以使用单个方法创建接口，从外部包引用 Customer 结构：

```go
package client

type customerGetter interface {
	GetAllCustomers() ([]store.Customer, error)
}
```

下图显示了包组织，有几点需要注意

<img src="https://img.ethanleo.top/uPic/image-20221210220603419.png" alt="image-20221210220603419" style="zoom:50%;" />



* 因为customersGetter接口只在客户端包中使用，所以可以保持不导出
* 在图中，它看起来像循环依赖。 但是，从store到client没有依赖性，因为接口是隐式满足的。 这就是为什么这种方法在具有显式实现的语言中并不总是可行的原因。

​	client包中根据需要定义了最准确的抽象（这里只有一种方法）。它与接口隔离原则（SOLID原则）的概念有关，该原则指出不应强迫任何客户端依赖它不使用的方法。 因此，在这种情况下，最好的做法是将具体的实现暴露在生产者端，让客户端自己决定如何使用，是否需要抽象。

​	为了完整起见,我们看看这种生产者端的接口在标准库中的使用。例如encoding包定义了encoding/json或encoding/binary等其他子包实现的接口。 是encoding包错了吗？显然不是，在这种情况下，encoding包中定义的抽象在整个标准库中使用，语言设计者知道预先创建这些抽象很有价值。 我们回到上一节的讨论：如果您认为抽象可能对想象的未来有帮助，或者至少，如果您不能证明这种抽象是有效的，请不要创建抽象。

​	在大多数情况下，接口应该位于消费者端。 然而，在特定的情况下（例如，当我们知道而不是预见抽象对消费者有帮助时），我们可能希望在生产者方面拥有它。 如果我们这样做了，我们应该努力让它尽可能地小，增加它的可重用潜力并使其更容易组合。	

## 返回接口

​	在设计函数签名时，我们可能必须返回接口或具体实现。让我们来看看为什么在许多情况下返回接口在Go中并不是一个很好的做法。

​	我们刚刚介绍了为什么接口通常存在于消费者方面。 下图 显示了如果一个函数返回一个接口而不是一个结构会发生什么。 我们将看到它会导致问题。

<img src="https://img.ethanleo.top/uPic/image-20221210224822768.png" alt="image-20221210224822768" style="zoom:50%;" />



​	在 store 包中，我们定义了一个 InMemoryStore 结构，它实现了 Store 接口。 同时，我们创建一个NewInMemoryStore 函数来返回一个 Store 接口。 在此设计中，实现包与客户端包之间存在依赖关系，这听起来可能有点奇怪。

​	例如，客户端包不能再调用 NewInMemoryStore 函数;否则，将存在循环依赖。有一种解决方案就是从另一个包调用这个函数并且向client注入一个Store的实现。

​	此外，如果另一个客户端使用 InMemoryStore 结构会怎样？ 在那种情况下，也许我们想将 Store 接口移到另一个包，或者返回到实现包，但我们讨论了为什么在大多数情况下这不是最佳实践。 它看起来像代码的味道

​	因此，一般来说返回接口会限制灵活性，因为我们强制所有客户端使用一种特定类型的抽象，在大多数情况下，我们可以从 Postel 定律中得到启发（https://datatracker.ietf.org/doc/html/rfc761）

*Be conservative in what you do, be liberal in what you accept from others.*

(对你所做的事情要保守，对你从别人那里接受的东西要自由。)

​																																	—Transmission Control Protocol	

​	上面这句话的意思运用在Go中就是：

* 返回具体的结构体而不是接口
* 接收的尽可能事接口

​	当然，也存在一个例外。作为软件工程师，我们知道规则永远不会百分百正确。最容易发现的就是error这个接口，他是很多函数返回的接口。在io包中也可以看到这样的情况

```go
func LimitReader(r Reader, n int64) Reader { 
  return &LimitedReader{r, n} 
}

```

​	在这段代码中，函数的返回值是一个io.Reader的接口。这里就和我们之前讨论的是冲突的，为什么会这样？io.Reader是一个前期抽象，语言设计者事先知道这种抽象级别会有所帮助（例如，在可重用性和可组合性方面）。

​	总而言之，在大多数情况下，我们不应该返回接口，而应该返回具体的实现。 否则，由于包依赖性，它会使我们的设计更加复杂，并且会限制灵活性，因为所有客户端都必须依赖相同的抽象。 同样，结论与前面的部分类似：如果我们知道（不是预见）抽象对客户端有帮助，我们可以考虑返回一个接口。 否则，我们不应该强制抽象； 他们应该被客户发现。 如果客户端出于某种原因需要抽象实现，它仍然可以在客户端执行此操作。

​	在下一章中，我们将讨论关于`any`使用的一些错误

## any 不表达任何东西

​	在Go中，如果一个接口不具有任何方法，我们称之为空接口。在 Go 1.18 中，预先声明的类型 any 成为空接口的别名

因此，所有出现的`interface{}`都可以替换成`any`。在很多情况下，`any`会被认为过度概括，因为它没有传达任何的信息。让我们来看看一些概念，在来讨论一些潜在的问题

​	一个`any`类型可以包含任何值的类型

```go
func main() {
	var i any

	i = 42 // int 
	i = "foo"  // string 
	i = struct {  // struct
		s string
	}{
		s: "bar",
	}

	i = f // function 
	_ = i // 分配给空白标识符以便示例编译
}
func f() {
	
} 
```

​	在为any类型赋值时，我们丢失了所有类型信息，这需要类型断言才可以从i变量中获取到任何有用的信息。如上面的案例所示。让我们看另一个例子，其中使用any是不准确的

```go
package store

type Customer struct {
	// some fields
}

type Contract struct {
	// some fields
}

type Store struct {
}

func (s *Store) Get(id string) (any, error) { // 返回 any
	// ... 
}

func (s *Store) Set(id string, v any) error { // 接收 any
	// ... 
}

```

​	尽管Store编译方面没有任何问题，但我还是需要花点时间考虑方法签名。因为我们接收并且返回any参数，所以这些方法缺乏表现力。如果后面的开发人员需要使用Store结构体，他们不得不深入研究文档或者阅读源码去理解如何使用这些方法。因此，接受或返回 any 类型并不能传达有意义的信息。 此外，因为在编译时没有安全措施，所以没有什么可以阻止调用者使用任何数据类型（例如 int）调用这些方法：

```go
s := store.Store()
s.Set("foo", 42)
```

​	通过使用any,我们失去了Go作为静态语言的一些好处。相反，我们应该避免使用any类型，尽可能使我们的函数签名明确。对于我们的示例，这可能就需要每种类型去复制Get和Set方法：

```go
func (s *Store) GetContract(id string) (Contract, error) { 
  // ...
}
func (s *Store) SetContract(id string, contract Contract) error { 
  // ...
}
func (s *Store) GetCustomer(id string) (Customer, error) {
  // ...
}
func (s *Store) SetCustomer(id string, customer Customer) error { 
  // ...
}
```

​	在上面的示例中，方法是具有表现力的，减少了不理解的风险。拥有更多的方法不一定是问题，因为用户也可以使用接口创建自己的抽象。例如，如果用户只对Contract方法感兴趣，他可以这样写：

```go
type ContractStorer interface {
    GetContract(id string) (store.Contract, error)
    SetContract(id string, contract store.Contract) error
}
```

​	any在什么情况下有用呢？让我们来看一下标准库，看看两个函数或方法接收any的例子。第一个示例在encoding/json包中。因为我们可以编组任何类型，所以Marshal函数接收一个any参数：

```go
func Marshal(v any) ([]byte, error) {
    // ...
}
```

​	另一个例子是在 database/sql 包中。 如果查询是参数化的（例如，SELECT * FROM FOO WHERE id=?），参数可以是任何类型。 因此，它也使用任何参数:

```go
func (db *DB) QueryContext(ctx context.Context, query string, args ...any) (*Rows, error) {
   var rows *Rows
   var err error
   var isBadConn bool
   for i := 0; i < maxBadConnRetries; i++ {
      rows, err = db.query(ctx, query, args, cachedOrNewConn)
      isBadConn = errors.Is(err, driver.ErrBadConn)
      if !isBadConn {
         break
      }
   }
   if isBadConn {
      return db.query(ctx, query, args, alwaysNewConn)
   }
   return rows, err
}
```

​	总之，如果确实需要接受或返回任何可能的类型（例如，涉及封送处理或格式化时），any 可能会有所帮助。 一般来说，我们应该不惜一切代价避免过度概括我们编写的代码。 如果能提高其他方面（例如代码表达能力），也许一点点重复代码偶尔会更好。
​	接下来，我们将讨论另一种抽象类型：泛型

## 对何时使用泛型感到困扰

​	Go 1.18 为语言添加了泛型。 简而言之，这允许使用稍后可以指定并在需要时实例化的类型编写代码。 但是，何时使用泛型以及何时不使用泛型可能会令人困惑。 在本节中，我们将描述 Go 中泛型的概念，然后查看常见的使用和误用。

### 简介

​	考虑以下从 map[string]int 类型中提取所有键的函数：

```go
func getKeys(m map[string]int) []string {
   var keys []string
   for k := range m {
      keys = append(keys, k)
   }
   return keys
}
```

​	如果我们想对另一种map类型（例如 map[int]string）使用类似的功能怎么办？在泛型出现之前，Go 开发人员有几个选择：使用代码生成、反射或复制代码。例如，我们可以编写两个函数，每个函数对应一种map类型，甚至可以尝试扩展 getKeys 以接受不同的map类型:

```go
func getKeys(m any) ([]any, error) {
   switch t := m.(type) {
   default:
      return nil, fmt.Errorf("unknown type: %T", t)
   case map[string]int:
      var keys []any
      for k := range t {
         keys = append(keys, k)
      }
      return keys, nil
   case map[int]string:
      // Copy the extraction logic
   }
   return nil, nil
}
```

​	通过上面的例子，我们发现了一些问题。首先，它增加了样板代码。实际上，当我们想增加一种类型的map时候，我们需要复制循环的内容。同时，现在这个方法接收any类型的参数，这也就是说我们失去了Go作为一种类型化语言的一些好处。实际上，检查类型是否支持是在运行时而不是在编译时完成的。因此我们在方法返回的时候还需要额外添加一个错误返回值。最后，因为键的类型可以是int或string，我们不得不返回一个any类型的切片来返回分解出来的键类型。这种方法增加了调用方的工作量，因为用户可能会需要执行键的类型检查或额外的转换。多亏了泛型，我们现在可以使用类型参数重构这段代码。

​	类型参数是我们可以与函数和类型一起使用的泛型类型。例如，以下的函数接收类型参数：

```go
func foo[T any](t T) { // T 是一个类型参数
  // .... 
}
```

​	当我们调用foo的时候，我们传递任何类型的类型参数。提供类型参数称为类型实例化，该工作在编译时完成。这将类型安全作为语言核心功能的一部分，避免了运行时的开销。

​	让我们回到getKeys这个函数，并且使用类型参数对其进行重构

```go
func getKeys[K comparable, V any](m map[K]V) []K {
 	var keys []K
  for k := range m {
    keys = append(keys, k)
  }
  return keys
}
```

​	为了处理映射，我们定义了两种类型参数。 首先，值可以是任意类型：V any。 但是，在 Go 中，映射键不能是 any 类型。 例如，我们不能使用切片

```go
var m map[[]byte]int
```

​	此代码导致编译错误：无效的映射键类型 []byte。 因此，我们没有接受任何键类型，而是有义务限制类型参数，以便键类型满足特定要求。 这里，要求键类型必须是可比较的（我们可以使用==或!=）。 因此，我们将 K 定义为可比较的而不是any

​	限制类型参数以匹配特定要求称为约束。约束是一种接口类型，可以包含：

* 一组行为(方法)
* 任意类型

​	让我们来看看后者的具体的例子。想象一下，我们不想接受映射键类型的任何可比较类型。例如：我们想将其限制为int或string类型。我们可以这样定义自定义约束：

```go
type customConstraint interface {
	~int | ~string
}

func getKeys[K customConstraint,V any](m map[K]V) []K {
  // 一样的实现
}
```

​	首先，我们定义一个 customConstraint 接口，使用联合运算符 | 将类型限制为 int 或 string （稍后我们将讨论 ~ 的使用）。 K 现在是一个 customConstraint，而不是以前的 comparable

​	getKeys 的签名强制我们可以使用任何值类型的映射调用它，但键类型必须是 int 或string。例如，在调用方：

```go
m = map[string]int{
    "one":   1,
    "two":   2,
    "three": 3,
}
keys := getKeys(m)

```

​	请注意，Go 可以推断 getKeys 是使用字符串类型参数调用的。 之前的调用等同于：

```go
keys := getKeys[string](m)
```

### ~int和int的比较

​	使用 ~int 的约束和使用 int 的约束有什么区别？ 使用 int 将其限制为该类型，而 ~int 限制其基础类型为 int 的所有类型。 为了说明，让我们想象一个约束，我们想将一个类型限制为实现 String() 字符串方法的任何 int 类型：

```go
type customConstraint interface {
    ~int
    String() string
}
```

​	使用此约束将类型参数限制为自定义类型。 例如：

```go
type customInt int
func (i customInt) String() string {
    return strconv.Itoa(int(i))
}

```

​	因为 customInt 是一个 int 并且实现了 String() string 方法，所以 customInt 类型满足定义的约束。 但是，如果我们将约束更改为包含 int 而不是 ~int，则使用 customInt 会导致编译错误，因为 int 类型未实现 `String() string`

​	到目前为止，我们已经讨论了对函数使用泛型的示例。 但是，我们也可以将泛型与数据结构一起使用。 例如，我们可以创建一个包含任何类型值的链表。 为此，我们将编写一个 Add 方法来附加一个节点:

```go
type Node [T any] struct{
  Val T
  next *Node[T]
}

func (n *Node[T]) Add(next *Node[T]){
  n.next = next 
}
```

​	在示例中，我们使用类型参数来定义 T 并使用 Node中的两个字段。 关于方法，接收器被实例化。 事实上，因为 Node 是通用的，它也必须遵循定义的类型参数。
​	关于类型参数要注意的最后一件事是它们不能与方法参数一起使用，只能与函数参数或方法接收器一起使用。 例如，以下方法将无法编译：	（在go中，方法是具有具体接受者的，函数是指不属于任何结构体）

```go
type Foo struct {}
func (Foo) bar[T any](t T) {}
./main.go:29:15: methods cannot have type parameters
```

​	如果我们想将泛型和方法一起使用，那么接受者需要成为类型参数。接下来，我们来看看什么时候应该和不应该使用泛型的具体情况

### 常见的用途和误用

​	泛型什么时候有用？ 让我们讨论一些推荐使用泛型的常见用途：

* **定义数据结构**：例如，如果我们实现二叉树、链表或堆，我们可以使用泛型来分解元素类型
* 使用any类型的切片、map和channel的函数：例如，合并两个通道的功能适用于任何通道类型。 因此，我们可以使用类型参数来分解通道类型

```go
func merge[T any] (ch1, ch2 <- chan T) <-chan T {
   // ...
}
```

* 分解出行为而不是类型：例如，sort包中，sort.Interface中有三个函数

```go
type Interface interface {
    Len() int
    Less(i, j int) bool
    Swap(i, j int)
}
```

​	此接口由不同的函数使用，例如 sort.Ints 或 sort.Float64s。 使用类型参数，我们可以分解出排序行为（例如，通过定义一个包含切片和比较函数的结构）

```go
type SliceFn[T any] struct {
  S []T
  Compare func(T ,T) bool
}


func (s SliceFn[T]) Len()  {return len(s.S)}
func (s SliceFn[T]) Less(i, j int) bool { return s.Compare(s.S[i], s.S[j]) } 
func (s SliceFn[T]) Swap(i, j int) { s.S[i], s.S[j] = s.S[j], s.S[i] }

```

​	然后，因为 SliceFn 结构实现了 sort.Interface，我们可以使用 sort.Sort(sort.Interface) 函数对提供的切片进行排序：

```go
s := SliceFn[int]{
          S: []int{3, 2, 1},
          Compare: func(a, b int) bool {
										return a < b 
          },
      }
      sort.Sort(s)
      fmt.Println(s.S)

[1 2 3]
```

​	在此示例中，分解出一种行为使我们能够避免为每种类型创建一个函数

​	相反，什么时候建议我们不要使用泛型？

* 调用类型参数方法时：考虑接收一个io.Writer的 函数并调用 Write 方法，例如

```go
func foo[T io.Writer](w T) {
          b := getBytes()
          _, _ = w.Write(b)
}
```

​	在这种情况下，使用泛型不会给我们的代码带来任何价值。 我们应该使 w 参数直接成为 io.Writer

* 当它使我们的代码更复杂时：泛型从来都不是强制性的，就像 Go开发人员，我们已经在没有他们的情况下生活了十多年。 如果我们正在编写通用函数或结构并且我们发现它不会使我们的代码更清晰，我们可能应该重新考虑我们针对该特定用例的决定

​	尽管泛型在特定情况下会有所帮助，但我们应该谨慎选择何时使用它们以及何时不使用它们。 一般来说，如果我们要回答何时不使用泛型，我们可以找到与何时不使用接口的相似之处。 事实上，泛型引入了一种抽象形式，我们必须记住，不必要的抽象会带来复杂性。

​	同样，我们不要用不必要的抽象污染我们的代码，让我们现在专注于解决具体问题。 这意味着我们不应该过早地使用类型参数。 让我们等到我们即将编写样板代码时再考虑使用泛型。

​	在下一节中，我们将讨论使用类型嵌入时可能出现的问题。

## 类型嵌入可能遇到的问题

​	创建结构时，Go 提供了嵌入类型的选项。 但如果我们不理解类型嵌入的所有含义，这有时会导致意想不到的行为。 在本节中，我们将研究如何嵌入类型、它们带来的影响以及可能出现的问题

​	在 Go 中，如果一个结构字段没有名字声明，它就被称为嵌入的。 例如：

```go
type Foo struct {
  Bar  // 嵌入字段
}

type Bar struct {
  Baz int
}
```

​	在 Foo 结构中，声明 Bar 类型时没有关联的名称； 因此，它是一个嵌入式字段。

​	我们使用嵌入来提升嵌入类型的字段和方法。 因为 Bar 包含一个 Baz 字段，所以该字段被提升为 Foo。 因此，Baz 可以从 Foo 获得

```go
foo := Foo{}
foo.Baz = 42
```

​	请注意，Baz 可从两种不同方式获得：使用 Foo.Baz 的升级路径或通过 Bar、Foo.Bar.Baz 的名义路径。 两者都涉及同一个字段。

![image-20221211190659489](https://img.ethanleo.top/uPic/image-20221211190659489.png)



### 接口和嵌入

​	嵌入同样可以在接口中使用，在下面的示例中，io.ReadWriter 由一个 io.Reader 和一个 io.Writer 组成：

```go
type ReadWriter interface {
  Reader 
  Writer
}
```

​	现在我们已经了解了什么是嵌入类型，让我们看看一个错误的用法。在下文中，我们实现了一个包含一些内存中数据的结构，我们希望使用互斥锁来保护它免受并发访问：

```go
type InMem struct {
  sync.Mutex
  m  map[string]int
}

func New() *InMem {
  return &InMen {m: make(map[string]int)}
}
```

​	我们这里不导出map，这样用户就不能直接与它交互，而只能通过导出方法进行交互。同时，加入了互斥域。因此，我们可以实现这样的一个Get方法

```go
func (i *InMem) Get(key string) (int, bool) {
    i.Lock()
  	v, contains := i.m[key]
  	i.Unlock()
  	return v, contains
}
```

​	因为嵌入了互斥锁，所以我们可以直接通过i去访问Lock和Unlock方法。

​	我们提到这样的例子是类型嵌入的错误用法。 这是什么原因？ 由于sync.Mutex是内嵌类型，所以提供Lock和Unlock方法。 因此，这两种方法都对使用 InMem 的外部用户可见

```go
m := inmem.New()
m.Lock() // ??
```

​	这样的嵌入可能是不需要的。在大多数情况下，互斥锁是我们想要封装在结构中并且使外部不可见的东西。因此，在这种情况下我们不应该将其设置为嵌入字段

```go
type InMem struct{
  mu sync.Mutex // 将sync.Mutex当作成员变量而不是嵌入
  m map[string]int 
}
```

​	因为互斥锁不是嵌入的，也没有导出，所以不能从外部客户端访问它。 现在让我们看另一个例子，但这次嵌入可以被认为是一种正确的方法。

​	我们想要编写一个io.WriteCloser 并公开两个方法 Write 和 Close 的自定义日志器。如果 io.WriteCloser 没有嵌入，我们需要这样写：

```go
type Logger struct {
    writeCloser io.WriteCloser
}

func (l Logger) Write(p []byte) (int, error) {
	return l.writeCloser.Write(p)
}

func (l Logger) Close() error {
  return l.writeCloser.Close()
}

func main() {
  l := Logger{writeCloser: os.Stdout}
  _, _ = l.Write([]byte("foo"))
   _ = l.Close()
}
```

​	Logger 必须同时提供 Write 和 Close 方法，后者只会将调用转发给 io.WriteCloser。 但是，如果该字段现在变成嵌入式，我们可以删除这些转发方法:

```go
type Logger struct {
  io.WriteCloser
}

func main() {
    l := Logger{WriteCloser: os.Stdout}
    _, _ = l.Write([]byte("foo"))
    _ = l.Close()
}
```

​	对于具有两个导出的 Write 和 Close 方法的客户端，它保持不变。 但是该示例阻止实现这些附加方法并且不需要转发调用。 另外，随着 Write 和 Close 被提升，这意味着 Logger 满足 io.WriteCloser 接口

### 嵌入与OOP子类化（面向对象）

​	将嵌入与 OOP 子类化区分开来有时会令人困惑。 主要区别与方法接收者的身份有关。 我们来看下图。 左侧表示嵌入到 Y 中的类型 X，而在右侧，Y 扩展了 X。

![image-20221211195115179](https://img.ethanleo.top/uPic/image-20221211195115179.png)



​	通过嵌入，嵌入类型仍然是方法的接收者。 相反，通过子类化，子类成为方法的接收者。

​	通过嵌入，Foo 的接收者仍然是 X。然而，通过子类化，Foo 的接收者变成了子类 Y。嵌入是关于组合，而不是继承	

​	关于类型嵌入，我们应该得出什么结论？ 首先，让我们注意它很少是必需的，这意味着无论用例是什么，我们都可以在没有类型嵌入的情况下解决它。 类型嵌入主要是为了方便：在大多数情况下，促进行为。如果我们决定使用类型嵌入，我们需要牢记两个主要约束

* 它不应该单独用作一些语法糖来简化对字段的访问（例如 Foo.Baz() 而不是 Foo.Bar.Baz()）。 如果这是唯一的理由，我们就不要嵌入内部类型，而是使用字段
* 它不应该提升我们想要从外部隐藏的数据（字段）或行为（方法）：例如，如果它允许客户端访问应该对结构保持私有的锁定行为

​	在下一节中，我们将讨论处理可选配置的常见模式。

## 不使用功能选项模式

​	在设计 API 时，可能会出现一个问题：我们如何处理可选配置？ 有效地解决这个问题可以提高我们的 API 的便利性。 本节将通过一个具体示例介绍处理可选配置的不同方法。

​	假设我们必须设计一个库来公开一个函数用于创建HTTP服务器。此函数将接受不同的输入：地址和端口，下面显示了函数的骨架

```go
func NewServer(addr string, port int) (*http.Server, error) { 
  // ...
}

```

​	我们库的客户已经开始使用这个功能了，大家都很高兴。 但在某些时候，我们的客户开始抱怨这个功能有些受限并且缺少其他参数（例如，写入超时和连接上下文）。 但是，我们注意到添加新的函数参数会破坏兼容性，迫使客户端修改调用 NewServer 的方式。 同时，我们希望通过这种方式丰富端口管理相关的逻辑

* 如果端口没有被设置，我们将使用默认端口
* 如果这个端口无效，那么我们会返回错误
* 如果这个端口为0，那么我们会使用随机端口
* 否则，我们将使用客户提供的端口

![image-20221211200059803](https://img.ethanleo.top/uPic/image-20221211200059803.png)

​	我们如何以 API 友好的方式实现此功能？ 让我们看看不同的选项。

### Config Struct

​	因为 Go 不支持函数签名中的可选参数，第一种可能的方法是使用配置结构来传达什么是必需的，什么是可选的。 例如，强制参数可以作为函数参数存在，而可选参数可以在 Config 结构中处理:

```go
type Config struct {
   Port        int
}

func NewServer(addr string, cfg Config) {
  
}
```

​	此解决方案修复了兼容性问题。 事实上，如果我们添加新的选项，它不会在客户端中断。 但是，这种方法并没有解决我们与端口管理相关的需求。 事实上，我们应该记住，如果没有提供结构字段，它会被初始化为零值

*  int类型的零值是0
* float类型的零值是0.0
* string类型的零值是""
* Nil会作为，slice，map，channel，指针,接口和function的零值

​	因此，在下面的例子中，两个结构是相等的：

```go
c1 := httplib.Config {
  Port: 0,
}

c2 := http.lib.Config {
  
}
```

​	在我们的示例中，我们需要找到一个方法来区分是用户设置为0还是没有进行端口的设置。有一种选择是将配置结构中的所有参数作为指针处理

```go
type Config struct {
  Port *int
}
```

​	使用整数指针，在语义上，我们可以突出值 0 和缺失值（nil 指针）之间的区别。

​	这个选项可行，但它有几个缺点。 首先，客户端提供整数指针并不方便。 客户端必须创建一个变量，然后以这种方式传递一个指针

```go
port := 0
config := httplib.Config{
  Port: &port, // 使用integer指针
}
```

​	它本身并不是一个亮点，但整个 API 使用起来不太方便。 此外，我们添加的选项越多，代码就会变得越复杂。
第二个缺点是客户端使用我们的默认配置库将需要以这种方式传递一个空结构

```go
httplib.NewServer("localhost", httplib.Config{})
```

​	这段代码看起来不太好。 读者将不得不理解这个神奇结构的含义

​	另一种选择是使用经典的构建器模式，如下一节所述

### 建造者模式

​	构建器模式最初是四人组设计模式的一部分，为各种对象创建问题提供了灵活的解决方案。 Config 的构造与结构本身是分开的。 它需要一个额外的结构体 ConfigBuilder，它接收配置和构建 Config 的方法

​	让我们看一个具体的例子，看看它如何帮助我们设计一个友好的 API 来满足我们所有的需求，包括端口管理

```go
type Config struct {
  Port int
}

type ConfigBuilder struct{
  port *int
}

func (b *ConfigBuilder) Port(port int) *ConfigBuilder{
  b.port = &port
  return b 
}

func (b *ConfigBuilder) Build() (Config, error) {
   cfg := Config{}
  
  if b.port == nil {
     cfg.Port = defaultHTTPPort
  }else {
    if *b.port == 0 {
        cfg.Port = randomPort()
    } else if *b.port < 0 {
        return Config{}, errors.New("port should be positive")
    } else {
        cfg.Port = *b.port
    }
  }
  return cfg,nil
}


func NewServer(addr string, config Config) (*http.Server, error) { 
  // ...
}
```

​	ConfigBuilder 结构保存客户端配置。 它公开了一个 Port 方法来设置端口。 通常，这样的配置方法会返回构建器本身，以便我们可以使用方法链接（例如，builder.Foo("foo").Bar("bar")）。 它还公开了一个 Build 方法，该方法包含初始化端口值的逻辑（指针是否为 nil 等），并在创建后返回一个 Config 结构

​	然后，客户端将按以下方式使用我们基于构建器的 API（我们假设我们已将代码放在 httplib 包中

```go
builder :=httplib.ConfigBuilder{}
builder.Port(8080)
cfg, err := builder.Build()
if err != nil{
  return err 
}

server, err := httplib.NewServer("localhost", cfg)
if err != nil {
	return err
}
```

​	首先，客户端创建一个 ConfigBuilder 并使用它来设置一个可选字段，例如端口。 然后，它调用 Build 方法并检查错误。 如果成功，配置将传递给 NewServer

​	这种方法使端口管理更加方便。 不需要传递整数指针，因为 Port 方法接受整数。 但是，如果客户端想要使用默认配置，我们仍然需要传递一个可以为空的配置结构

```go
server, err := httplib.NewServer("localhost", nil)
```

​	在某些情况下，另一个缺点与错误管理有关。 在会抛出异常的编程语言中，如果输入无效，构建器方法（例如 Port）会引发异常。 如果我们想保持链接调用的能力，函数就不能返回错误。

​	因此，我们必须延迟Build方法中的验证，如果客户端可以传递多个选项，但我们想要精确处理端口无效的情况，这会使错误处理更加复杂

​	现在让我们看看另一种称为函数选项模式的方法，它依赖于可变参数

### 功能选项模式

​	我们要讨论的最后一种方法是功能选项模式。 尽管有不同的实现方式，变化很小，但主要思想如下

* 未导出的结构包含配置：options
* 每个options都是一个返回相同类型的函数：type Option func(options *options) error 。例如：WithPort 接受一个int类型的参数，并返回一个Option的结构类型
* ![image-20221211212244202](https://img.ethanleo.top/uPic/image-20221211212244202.png)

​	这是options结构、options类型和 WithPort 选项的 Go 实现

```go
type Options struct {
  port *int
}

type Option func (options *options) error 

func WithPort(port int) Option {
  return func (options *options) error {
    if port < 0 {
      return errors.New("port should be positive")
    }
    options.port = &port
    return nil
  }
}
```

​	这里，WithPort 返回一个闭包。 闭包是一个匿名函数，它从其主体外部引用变量； 在这种情况下，端口变量。 闭包遵循 Option 类型并实现端口验证逻辑。 每个配置字段都需要创建一个包含类似逻辑的公共函数（按照惯例以 With 前缀开头）：在需要时验证输入并更新配置结构。

让我们看一下提供者端的最后一部分：NewServer 实现。 我们会将选项作为可变参数传递。 因此，我们必须迭代这些选项来改变选项配置结构：

```go
func NewServer(addr string ，opts ...Options) (*http.Server,error){
  var options Options
  for _ , opt :=range opts {
    err := opt(&options)
    if err!=nil{
      return nil,err
    }
  }
 	
  var port int 
  if options.port == nil {
    port = defaultHTTPPort
  } else {
    if *options.port == 0 {
      port = randomPort()
    }else{
      port = *options.port
    } 
  }
  // ...
}
```

​	我们首先创建一个空的Options选项结构。然后，我们便利每一个Option参数并执行他们以改变Option结构（option类型是一个函数)。一旦构建了选项结构，我们就可以实现有关端口管理的最终逻辑

​	因为 NewServer 接受可变选项参数，所以客户端现在可以通过在强制地址参数后传递多个选项来调用此 API，例如：

```go
server, err := httplib.NewServer("localhost",
        httplib.WithPort(8080),
        httplib.WithTimeout(time.Second))
```

​	但是，如果客户端需要默认配置，则不必提供参数（例如，一个空结构，正如我们在前面的方法中看到的那样)。

```go
server, err := httplib.NewServer("localhost")
```

​	此模式是功能选项模式。 它提供了一种方便且 API 友好的方式来处理选项。 尽管构建器模式可以是一个有效的选项，但它有一些小的缺点，这些缺点往往使功能选项模式成为 Go 中处理此问题的惯用方法。 我们还要注意，这种模式用于不同的 Go 库，例如 gRPC

​	下一节将讨论另一个常见错误：组织不当。

## 项目组织不当

​	Go 语言的维护者对于在 Go 中构建项目没有很强的约定。 然而，多年来出现了一种布局：project-layout(https://github.com/golang-standards/project-layout)

​	如果我们的项目足够小（只有几个文件），或者如果我们的组织已经创建了它的标准，则可能不值得使用或迁移到项目布局。 否则，它可能值得考虑。 让我们看看这个布局，看看主要目录是什么：

* /cmd 本项目的主干,每个应用程序的目录名应该与你想要的可执行文件的名称相匹配(例如，`/cmd/myapp`)
* /internal 私有应用程序和库代码。这是你不希望其他人在其应用程序或库中导入代码。
* /pkg 外部应用程序可以使用的库代码,其他项目会导入这些库，希望它们能正常工作，所以在这里放东西之前要三思
* /test 额外的外部测试应用程序和测试数据。你可以随时根据需求构造`/test`目录,Go 中的单元测试与源文件位于同一包中。但是，例如，公共 API 测试或集成测试应该位于 /test 中。
* /configs 配置文件存放目录
* /docs 设计文档和用户文档
* /examples 你的应用程序和/或公共库的示例
* /api OpenAPI/Swagger 规范，JSON 模式文件，协议定义文件。
* /web 特定于 Web 应用程序的组件:静态 Web 资产、服务器端模板和 SPAs
* /build 打包和持续集成 (CI) 文件目录
* /scripts 用于分析、安装等的脚本
* /vendor 应用程序依赖项（例如，Go 模块依赖项）

没有像其他一些语言那样的 /src 目录。 理由是 /src 太通用了； 因此，此布局有利于 /cmd、/internal 或 /pkg 等目录

### 包组织

​	在 Go 中，没有子包的概念。 然而，我们可以决定在子目录中组织包。 如果我们看一下标准库，net 目录是这样组织的

```go
/net
	/http
		client.go
		... 
	/smtp
		auth.go
		...
  addrselect.go
	...
```

​	net 既充当包又充当包含其他包的目录。 但是net/http并不继承自net，也没有对net包的特定访问权限。 net/http 内部的元素只能看到导出的net包元素。 子目录的主要好处是将包保存在一个高内聚的地方

​	关于整体组织，有不同的思想流派。 例如，我们应该按上下文还是按层来组织我们的应用程序？ 这取决于我们的喜好。 我们可能喜欢按上下文（例如客户上下文、合同上下文等）对代码进行分组，或者我们可能喜欢遵循六边形架构原则并按技术层分组。 如果我们做出的决定适合我们的用例，那么只要我们保持一致，它就不会是一个错误的决定。

​	关于包，我们应该遵循多种最佳实践。 首先，我们应该避免过早打包，因为它可能会使我们的项目过于复杂。 有时，最好使用一个简单的组织，并在我们了解项目包含的内容后让我们的项目不断发展，而不是强迫自己预先制定完美的结构

​	粒度是另一个需要考虑的重要因素。 我们应该避免有几十个 nano 包只包含一个或两个文件。 如果我们这样做了，那是因为我们可能错过了这些包之间的一些逻辑联系，使读者更难理解我们的项目。 反过来，我们也应该避免稀释包名含义的巨大包

​	包命名也应谨慎考虑。 众所周知（作为开发人员），命名很难。 为了帮助客户理解 Go 项目，我们应该根据它们提供的内容而不是它们包含的内容来命名我们的包。 此外，命名应该是有意义的。 因此，包名应该简短、简洁、富有表现力，并且按照惯例，一个小写单词

​	关于要导出的内容，规则非常简单。 我们应该尽可能地减少应该导出的内容，以减少包之间的耦合，并隐藏不必要的导出元素。 如果我们不确定是否导出一个元素，我们应该默认不导出它。 后面如果发现需要导出，可以调整我们的代码。 我们还要记住一些例外情况，例如导出字段以便可以使用 encoding/json 解组结构

​	组织一个项目并不简单，但遵循这些规则应该有助于使其更容易维护。 但是，请记住，一致性对于简化可维护性也很重要。 因此，让我们确保在代码库中尽可能保持一致。	

## 创建实用程序包

​	本节讨论一种常见的不良做法：创建共享包，例如 utils、common 和 base。 我们将检查这种方法存在的问题，并学习如何改进我们的组织。

​	让我们看一个受 Go 官方博客启发的示例。 它是关于实现一个集合数据结构（一个忽略值的映射）。 在 Go 中执行此操作的惯用方法是通过 map[K]struct{} 类型来处理它，其中 K 可以是映射中允许的任何类型作为键，而值是 struct{} 类型。 实际上，值类型为 struct{} 的映射表明我们对值本身不感兴趣。 让我们在 util 包中公开两个方法

```go
package util 

func NewStringSet(...string) map[string]struct{} {
   // 创建一个string set
}


func SortStringSet(map[string]struct{}) []string {
  // ... 返回一个排序后的list
}
```

​	用户可能会这样使用这个包

```go
set := util.NewStringSet("c", "a", "b")
fmt.Println(util.SortStringSet(set))
```

​	这里的问题是 util 没有意义。 我们可以称它为 common、shared 或 base，但它仍然是一个毫无意义的名称，无法提供任何有关包提供内容的信息。

​	我们应该创建一个富有表现力的包名称，例如 stringset，而不是实用程序包。 例如:

```go
package stringset
func New(...string) map[string]struct{} { ... }
func Sort(map[string]struct{}) []string { ... }
```

​	在这个例子中，我们去掉了NewStringSet和SortStringSet的后缀，分别变成了New和Sort。 在客户端，它现在看起来像这样

```go
set := stringset.New("c", "a", "b")
fmt.Println(stringset.Sort(set))
```

​	我们甚至可以更进一步。 我们可以创建特定类型并将 Sort 作为方法公开，而不是公开实用程序函数

```go
package stringset
type Set map[string]struct{}
func New(...string) Set { ... }
func (s Set) Sort() []string { ... }
```

​	此更改使调用更加简单。 只有一个对 stringset 包的引用

```go
set := stringset.New("c", "a", "b")
fmt.Println(set.Sort())
```

​	通过这个小的重构，我们摆脱了一个无意义的包名来暴露一个富有表现力的 API。 正如 Dave Cheney（Go 的项目成员）所提到的，我们经常会找到处理通用设施的实用程序包。 例如，如果我们决定有一个客户端和一个服务器包，我们应该把公共类型放在哪里？ 在这种情况下，也许一种解决方案是将客户端、服务器和公共代码组合到一个包中。

​	命名包是应用程序设计的关键部分，我们也应该谨慎对待这一点。 根据经验，创建没有有意义名称的共享包不是一个好主意；命名包是应用程序设计的关键部分，我们也应该谨慎对待这一点。 根据经验，创建没有有意义名称的共享包不是一个好主意； 这包括实用程序包，如 utils、common 或 base；此外，请记住，以包提供的内容而非包含的内容命名包是提高其表现力的有效方法。

​	在下一节中，我们将讨论包和包冲突

## 包名冲突

​	当变量名称与现有包名称冲突时会发生包冲突，从而阻止包被重用。 让我们看一个具体的示例，其中包含一个公开 Redis 客户端的库

```go
package redis

type Client struct { ... }

func NewClient() *Client { ... }

func (c *Client) Get(key string) (string, error) { ... }
```

​	现在，让我们跳到客户端。 尽管包名为 redis，但在 Go 中创建一个名为 redis 的变量是完全有效的：

```go
redis := redis.NewClient()
v, err := redis.Get("foo")
```

​	在这里，redis 变量名与 redis 包名冲突。 即使这是允许的，也应该避免。 实际上，在 redis 变量的整个范围内，将无法访问 redis 包。假设一个限定符在整个函数中引用了一个变量和一个包名。 在这种情况下，代码阅读者可能不清楚限定词指的是什么。 有哪些选择可以避免这种碰撞？ 第一个选项是使用不同的变量名。 例如

```go
redisClient := redis.NewClient()
v, err := redisClient.Get("foo")
```

​	这可能是最直接的方法。 然而，如果出于某种原因我们更愿意保留名为 redis 的变量，我们可以使用包导入。 使用包导入，我们可以使用别名来更改限定符以引用 redis 包。 例如

```go
import redisapi "mylib/redis"

redis := redisapi.NewClient()
v, err := redis.Get("foo")
```

​	这里，我们使用redisapi导入别名来引用redis包，这样我们就可以保留我们的变量名redis

​	还要注意，我们应该避免变量和内置函数之间的命名冲突。 例如：

```go
copy := copyFile(src, dst)
```

​	在这种情况下，只要复制变量存在，就无法访问复制内置函数。 综上所述，我们应该防止变量名冲突，避免歧义。 如果我们遇到冲突，我们应该找到另一个有意义的名称或使用导入别名。

​	在下一节中，我们将看到与代码文档相关的常见错误。

## 缺少代码文档

​	文档是编码的一个重要方面。 它简化了客户使用 API 的方式，但也有助于维护项目。 在 Go 中，我们应该遵循一些规则来使我们的代码符合地道。 让我们检查一下这些规则。
​	首先，必须记录每个导出的元素。 无论是结构体、接口、函数还是其他东西，如果导出，都必须记录在案。 约定是添加注释，从导出元素的名称开始。 例如

```go
// Customer is a customer representation.
type Customer struct{}
// ID returns the customer identifier.
func (c Customer) ID() string { return "" }
```

​	按照惯例，每条注释都应该是一个完整的句子，并以标点符号结尾。 还要记住，当我们记录一个函数（或方法）时，我们应该突出这个函数打算做什么，而不是它是如何做的； 这个属于核心功能和注释，不是文档。 此外，理想情况下，文档应该提供足够的信息，使调用者不必查看我们的代码即可了解如何使用导出的元素。

```go
Deprecated elements
使用这样的注解 注释已经被废弃的方法 // Deprecated: comment this way:

// ComputePath returns the fastest path between two points.
// Deprecated: This function uses a deprecated way to compute
// the fastest path. Use ComputeFastestPath instead.
func ComputePath() {}

调用者在调用该方法的时候就会得到一个警告
```

​	在记录变量或常量时，我们可能有兴趣传达两个方面：其目的和内容。 前者应该作为代码文档存在，以便对外部客户有用。 不过，后者不一定是公开的。 例如

```go
// DefaultPermission is the default permission used by the store engine. 
const DefaultPermission = 0o644 // Need read and write accesses.
```

​	此常量表示默认权限。 代码文档传达了它的目的，而常量旁边的注释描述了它的实际内容（读和写访问）。

​	为了帮助客户和维护者了解包的范围，我们还应该记录每个包。 约定是以 // Package 开始注释，后跟包名称

```go
// Package math provides basic constants and mathematical functions. //
// This package does not guarantee bit-identical results
// across architectures.
package math
```

​	包注释的第一行应该简洁。 那是因为它会出现在包中。 然后，我们可以在以下几行中提供我们需要的所有信息

![image-20221211233008091](https://img.ethanleo.top/uPic/image-20221211233008091.png)

​	总之，我们应该记住，每个导出的元素都需要被记录下来。 记录我们的代码不应该成为一种约束。 我们应该抓住机会确保它能帮助调用者和维护者理解我们代码的目的。

​	最后，在本章的最后一节，我们将看到一个关于工具的常见错误：不使用 linters

## 不使用 linters

​	linter 是一种用于分析代码和捕获错误的自动工具。 本节的范围不是提供现有 linter 的详尽列表； 否则，它将很快被弃用。 但是我们应该理解并记住为什么 linters 对于大多数 Go 项目都是必不可少的

​	为了理解为什么 linters 很重要，让我们举一个具体的例子。 在错误 #1“意外的变量阴影”中，我们讨论了与变量阴影相关的潜在错误。 使用 vet（Go 工具集中的标准 linter）和 shadow，我们可以检测阴影变量：

```go
package main
import "fmt"
func main() {
i := 0 if true {
i := 1
        fmt.Println(i)
    }
    fmt.Println(i)
}
```

因为 vet 包含在 Go 二进制文件中，所以我们首先安装 shadow，将其与 Go vet 链接，然后在前面的示例中运行它

```she
$ go install \
golang.org/x/tools/go/analysis/passes/shadow/cmd/shadow
$ go vet -vettool=$(which shadow)
./main.go:8:3:
declaration of "i" shadows declaration at line 6
```

​	如我们所见， vet 告诉我们变量 i 在这个例子中被隐藏了。 使用适当的 linters 可以帮助我们的代码更健壮并检测潜在的错误。

​	同样，本节的目标不是列出所有可用的 linter。 但是，如果您不是linters 的普通用户，这里有一个你可能想使用的列表：

* https://golang.org/cmd/vet/  一个标准的 Go 分析器
* https://github.com/kisielk/errcheck 一个错误检查器
* https://github.com/fzipp/gocyclo Gocyclo 计算 Go 源代码中函数的[圈复杂度。](https://en.wikipedia.org/wiki/Cyclomatic_complexity)
* https://github.com/jgautheron/goconst 一个重复的字符串常量分析器

除了 linters，我们还应该使用代码格式化程序来修复代码风格。 以下是一些代码格式化程序的列表，供您尝试：

* https://golang.org/cmd/gofmt/ 一个标准的 Go 代码格式化程序
* https://godoc.org/golang.org/x/tools/cmd/goimports  一个标准的 Go 导入格式化程序

​	同时，我们还应该看看 golangci-lint (https://github.com/golangci/golangci-lint)。 它是一种 linting 工具，在许多有用的 linter 和格式化程序之上提供了一个外观。 此外，它允许并行运行 linters 以提高分析速度，这非常方便。Linters 和格式化程序是提高代码库质量和一致性的有效方法。 让我们花点时间了解我们应该使用哪一个，并确保我们自动执行它们（例如 CI 或 Git 预提交挂钩）。





































































