---
title: Protocol Buffer基本使用
categories: protobuffer
comments: true
---

# Protocol Buffers的基本使用

## 什么是Protocol buffers

Protocol Buffer是一个由 Google 开发的协议，允许结构化数据的序列化和反序列化协议缓冲。谷歌开发它的目的是提供一种比 XML 更好的方式来使系统进行通信。因此，他们致力于使其比 XML 更简单、更小、更快、更易于维护。这个协议甚至超过了 JSON，具有更好的性能、更好的可维护性和更小的体积。

<!--more-->

## 使用Protocol Buffers的好处

Protocol buffer 可以支持多语言，以及跨平台

* 解析快
* 传输速度快，体积小
* 支持多语言

支持的语言包括

* [C++](https://developers.google.com/protocol-buffers/docs/reference/cpp-generated#invocation)
* [C#](https://developers.google.com/protocol-buffers/docs/reference/csharp-generated#invocation)
* [Java](https://developers.google.com/protocol-buffers/docs/reference/java-generated#invocation)
* [Kotlin](https://developers.google.com/protocol-buffers/docs/reference/kotlin-generated#invocation)
* [Objective-C](https://developers.google.com/protocol-buffers/docs/reference/objective-c-generated#invocation)
* [PHP](https://developers.google.com/protocol-buffers/docs/reference/php-generated#invocation)
* [Python](https://developers.google.com/protocol-buffers/docs/reference/python-generated#invocation)
* [Ruby](https://developers.google.com/protocol-buffers/docs/reference/ruby-generated#invocation)
* [Go](https://github.com/protocolbuffers/protobuf-go)
* [Dart](https://github.com/google/protobuf.dart)

## 下载安装Protocol Buffers 编译器

通过[github ](https://github.com/protocolbuffers/protobuf/releases) release页面，下载protoc

![image-20230115110519332](https://img.ethanleo.top/uPic/image-20230115110519332.png)

打开release页面，我们可以直接找到protoc的安装包，将其下载并且配置即可

安装 Go protocol buffers 生成插件

```go
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
```

编译产生的插件`protoc-gen-go`会安装在`$GOBIN`下，默认就是`$GOPATH/bin`路径下

## Protocol Buffer基本语法

首先我们需要创建一个proto文件，在proto文件中，我们需要通过`syntax`关键字来指定我们需要用到的protocol buffers的版本。除此之外，我们还需要添加`option go_package`来指定生成的包地址，`package`字段使用于proto之间的依赖指定的路径，同时proto文件中的每一行都需要使用`;`结尾

### 定义消息类型

我们这里定义一个Person类型的消息

```protobuf
// 指定proto语言版本
syntax = "proto3";

// 生成go文件的包路径
option go_package = "/pb";

// proto文件包的路径
package grpc.gateway.demo.proto.examplepb;


message Person {
    int32 id = 1;
    string name = 2;
    string email = 3;
    float salary = 4;
    bool sex = 5;
}
```

通过protoc生成go文件

`protoc -I=. --go_out=. ./proto/examplepb/person.proto `

通过上面编写的简单proto文件，我们可以发现，proto文件中定义message与我们创建一个go中的结构体类似，包括内部对元素的类型定义，我们也是非常熟悉的。但是我们也还有一些出去基本类型之外的复杂类型，例如:枚举、map、数组等，我们需要如何去定义

#### repeated

repeated关键字的作用是用来定义数组，使用方式是`repeated 数组类型 属性名称 =  字段编号;`

```protobuf
message Person {
	repeated string name = 1;
}
```

#### map

map类型的定义方式是`map <键类型，值类型> 属性名称 = 字段编号;` ，这里需要注意对于map的键类型，只能定义为基本数据类型，但是值的类型可以是任何支持的类型

```protobuf
message Person {
	map <string, Pet> pets =1;
}

message Pet {
	string name = 1;
}
```

#### enum

对于枚举的定义，我们需要用到enum关键字。

```protobuf
enum PhoneType {
    PHONE_TYPE_UNSPECIFIED = 0;
    PHONE_TYPE_WORK = 1;
    PHONE_TYPE_HOME = 2;
}
```

在枚举的定义中，我们需要指定一个零值`    PHONE_TYPE_UNSPECIFIED = 0;` 

## 序列化与反序列化

```go
package proto

import (
   "fmt"
   "log"
   "os"
   "testing"

   "google.golang.org/protobuf/proto"

   "grpc-gateway-demo/pb"
)

var fName = "person.txt"

func TestPerson(t *testing.T) {
   person := &pb.Person{}
   phone := &pb.Phone{}
   pet := &pb.Pet{
      Name: "Leo",
      Age:  1,
   }

   phone.PhoneNumber = "1234567789"
   phone.PhoneType = pb.PhoneType_PHONE_TYPE_WORK
   // phone.PhoneType = 1

   person.Name = "Ethan"
   person.Email = "xxxx@yeah.net"
   person.Phone = phone
   person.Pets = map[string]*pb.Pet{}
   person.Pets[pet.Name] = pet

   b, err := proto.Marshal(person)
   if err != nil {
      log.Fatalf("proto marshal failed: %v", err)
      return
   }

   f, err := os.Create(fName)
   defer f.Close()
   if err != nil {
      log.Fatalf("os create failed: %v", err)
      return
   }

   _, err = f.Write(b)
   if err != nil {
      log.Fatalf("write file failed: %v", err)
      return
   }

}

func TestReadPerson(t *testing.T) {
   b, err := os.ReadFile(fName)
   if err != nil {
      log.Fatalf("read file failed: %v", err)
      return
   }
   p := &pb.Person{}
   err = proto.Unmarshal(b, p)
   if err != nil {
      log.Fatalf("proto unmarshal failed: %v", err)
      return
   }

   fmt.Println(p)
}
```

上面，我们了解了protocol buffers的基本使用。我们来看看如何去使用生成的go文件中的内容