

#  grpc-gateway的基本使用

## 简介

grpc-gateway是protoc的以一个插件，它会读取proto文件中的grpc服务的定义并将其生成RestFul JSON API,并将其反向代理到我们的grpc服务中

<!--more-->

<img src="https://grpc-ecosystem.github.io/grpc-gateway/assets/images/architecture_introduction_diagram.svg" style="zoom:50%;" />

grpc-gatway是在grpc上做的一个拓展。但是grpc并不能很好的支持客户端，以及传统的RESTful API。因此grpc-gateway诞生了，该项目可以为我们的grpc服务提供HTTP+JSON接口。

## 插件安装

首先，我们在项目中去创建一个tools的文件，然后运行`go mod tidy`

```go
package tools

import (
    _ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway"
    _ "github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2"
    _ "google.golang.org/grpc/cmd/protoc-gen-go-grpc"
    _ "google.golang.org/protobuf/cmd/protoc-gen-go"
)
```

最后我们运行`go install`将其安装在我们的`$GOBIN`目录下

```sh
go install \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2 \
    google.golang.org/protobuf/cmd/protoc-gen-go \
    google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

### grpc 插件安装

[grpc](https://grpc.io/docs/languages/go/quickstart/)文档

```sh
go install google.golang.org/protobuf/cmd/protoc-gen-go@v1.28
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@v1.2
```

## 使用

### 一、定义proto文件

```protobuf
syntax = "proto3";

option go_package = "/pb;";

package grpc.gateway.demo.proto.examplepb;

import "google/api/annotations.proto";

// 导入api注释依赖

// 定义服务
service BookService {
    // 创建书籍
    rpc CreateBook (CreateBookRequest) returns (CreateBookResponse) {
        option (google.api.http) = {
            post: "/v1/books"
            body: "*"
        };
    };

    // 获取书籍
    rpc GetBook (GetBookRequest) returns (GetBookResponse) {
        option (google.api.http) = {
            get: "/v1/books/{id}"
        };
    }


//    rpc UpdateBook (UpdateBookRequest) returns (UpdateBookResponse) {
//
//    };
//
//
//    rpc DeleteBook (DeleteBookRequest) returns (DeleteBookResponse) {
//
//    };
//
//    rpc ListBook (ListBookRequest) returns (ListBookResponse) {};


}

message Book {
    // int64 会自动转换为string 进行返回
    int32 id = 1;
    string name = 2;

}

// 定义接收参数
message CreateBookRequest {
    string name = 1;

}


message CreateBookResponse {
    string code = 1;
    string message = 2;
    Book data = 3;
}


message GetBookRequest {
    int32 id = 1;
}


message GetBookResponse {
    string code = 1;
    string message = 2;
    Book data = 3;
}
```

我们在使用grpc-gateway这个框架的时候，需要使用proto文件，将我们的grpc服务进行http+json的一个暴露，以此来对外达到一个提供http+json服务的接口

### 二、生成go文件

在编写好proto文件之后，我们需要使用插件将proto文件生成对应的go文件。

由于我们依赖了google的proto文件，所以我们在使用protoc生成go文件的时候，需要将依赖的proto文件复制到我们的项目中，依赖的proto文件仓库[google/api](https://github.com/googleapis/googleapis/tree/master/google/api)

```te
google/api/annotations.proto
google/api/field_behavior.proto
google/api/http.proto
google/api/httpbody.proto

```

我们还需要依赖的[google/protobuf](https://github.com/protocolbuffers/protobuf/tree/main/src/google/protobuf/compiler)

```tex
google/protobuf/descriptor.proto
```



```she
protoc -I . --grpc-gateway_out=../gen \
    --grpc-gateway_opt logtostderr=true \
    --grpc-gateway_opt paths=source_relative \
    --go_out=../gen \
    --go_opt paths=source_relative \
    --go-grpc_out=../gen \
    --go-grpc_opt paths=source_relative \
    ./examplepb/book.proto

```

将需要依赖的文件放入项目中，我们就可以使用protoc生成go文件，这里需要用到三个插件

* go
* grpc
* grpc-gateway

### 三、启动服务

#### 实现BookService

```go
package service

import (
	"context"
	"errors"
	"sync"

	pb "grpc-gateway-demo/gen/examplepb"
)

type BookService struct {
	pb.UnimplementedBookServiceServer
}

func NewBookService() *BookService {
	l := localStorage{
		Count: 0,
		DB:    make(map[int32]*pb.Book),
	}
	db = &l
	return &BookService{}
}

var db *localStorage

type localStorage struct {
	Count int32
	DB    map[int32]*pb.Book
	mux   sync.Mutex
}

func (l *localStorage) getId() int32 {
	l.Count = l.Count + 1
	return l.Count
}

func (l *localStorage) Store(d *pb.Book) error {
	if d == nil {
		return errors.New("data is nil")
	}

	if d.Id <= 0 {
		return errors.New("illegal id")
	}
	l.DB[d.Id] = d
	return nil
}

func (l *localStorage) Load(id int32) (*pb.Book, error) {
	if id <= 0 {
		return nil, errors.New("illegal id")
	}
	book := l.DB[id]
	return book, nil
}

func (b *BookService) CreateBook(ctx context.Context, req *pb.CreateBookRequest) (*pb.CreateBookResponse, error) {
	resp := &pb.CreateBookResponse{}
	db.mux.Lock()
	defer db.mux.Unlock()
	id := db.getId()
	book := pb.Book{
		Name: req.GetName(),
		Id:   id,
	}

	err := db.Store(&book)
	if err != nil {
		return resp, err
	}
	resp.Data = &book
	return resp, nil
}

func (b *BookService) GetBook(ctx context.Context, req *pb.GetBookRequest) (*pb.GetBookResponse, error) {
	resp := &pb.GetBookResponse{}
	db.mux.Lock()
	defer db.mux.Unlock()

	book, err := db.Load(req.GetId())
	if err != nil {
		return resp, err
	}
	resp.Data = book
	return resp, nil
}

```

#### 启动gRPC

```go
package server

import (
   "fmt"
   "io"
   "net"
   "os"

   "google.golang.org/grpc"
   "google.golang.org/grpc/grpclog"

   "grpc-gateway-demo/example/book/config"
   "grpc-gateway-demo/example/book/service"
   pb "grpc-gateway-demo/gen/examplepb"
)

// Run 启动rpc服务
func Run() error {
   log := grpclog.NewLoggerV2(os.Stdout, io.Discard, io.Discard)
   grpclog.SetLoggerV2(log)
   grpcAddr := config.GetRpcAddr()
   l, err := net.Listen("tcp", grpcAddr)
   if err != nil {
      grpclog.Errorf("tcp listen failed: %v", err)
      return err
   }
   defer func() {
      if err = l.Close(); err != nil {

         fmt.Fprintln(os.Stderr, err)
      }
   }()

   s := grpc.NewServer()

   //     注册服务
   registerServer(s)
   log.Infof("Serving gRPC on %s", l.Addr())
   return s.Serve(l)
}

func registerServer(s *grpc.Server) {
   pb.RegisterBookServiceServer(s, service.NewBookService())
}
```

#### 启动gateway

```go
package gateway

import (
   "context"
   "io"
   "net/http"
   "os"

   "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
   "google.golang.org/grpc"
   "google.golang.org/grpc/credentials/insecure"
   "google.golang.org/grpc/grpclog"

   "grpc-gateway-demo/example/book/config"
   pb "grpc-gateway-demo/gen/examplepb"
)

func Run() error {
   log := grpclog.NewLoggerV2(os.Stdout, io.Discard, io.Discard)
   grpclog.SetLoggerV2(log)
   ctx := context.Background()
   // 创建grpc连接
   conn, err := grpc.DialContext(ctx, config.GetRpcAddr(), grpc.WithTransportCredentials(insecure.NewCredentials()))
   if err != nil {
      log.Error("dial failed: %v", err)
      return err
   }
   mux := runtime.NewServeMux()
   err = newGateway(ctx, conn, mux)
   if err != nil {
      log.Error("register handler failed: %v", err)
      return err
   }
   server := http.Server{
      Addr:    config.GetHttpAddr(),
      Handler: mux,
   }
   log.Infof("Serving Http on %s", server.Addr)
   err = server.ListenAndServe()
   if err != nil {
      return err
   }
   return nil
}

func newGateway(ctx context.Context, conn *grpc.ClientConn, mux *runtime.ServeMux) error {

   err := pb.RegisterBookServiceHandler(ctx, mux, conn)
   if err != nil {
      return err
   }
   return nil
}
```

#### 启动main

```go
package main

import (
   "fmt"
   "os"

   "grpc-gateway-demo/example/book/gateway"
   "grpc-gateway-demo/example/book/server"
)

func main() {
   go func() {
      err := server.Run()
      if err != nil {
         fmt.Fprintln(os.Stderr, err)
         os.Exit(1)
      }
   }()

   err := gateway.Run()
   fmt.Fprintln(os.Stderr, err)
   os.Exit(1)

}
```

## 自定义路由

有些时候，rpc的服务不能满足我们的需求，比如文件上传下载，使用proto文件定义api以及实现是无法实现的，这个时候需要我们额外的添加上自定义路由来完成相关操作。

对于自定义路由也并不复杂，只需要在启动gateway的时候，将我们指定的路由添加到`mux`中即可

```go
	mux := runtime.NewServeMux()

	if err = mux.HandlePath(http.MethodPost, "/v1/objects", handler.Upload); err != nil {
		return err
	}

	if err = mux.HandlePath(http.MethodGet, "/v1/objects/{name}", handler.Download); err != nil {
		return err
	}

```

handler的实现

```go
package handler

import (
   "encoding/json"
   "fmt"
   "io"
   "net/http"
   "os"
)

func Upload(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
   file, header, err := r.FormFile("file")
   if err != nil {
      w.WriteHeader(http.StatusInternalServerError)
      fmt.Fprintln(w, err)
      return
   }
   defer file.Close()

   if header == nil {
      return
   }

   data, err := io.ReadAll(file)
   if err != nil {
      w.WriteHeader(http.StatusInternalServerError)
      fmt.Fprintln(w, err)
      return
   }

   err = os.WriteFile(header.Filename, data, 0777)
   if err != nil {
      w.WriteHeader(http.StatusInternalServerError)
      fmt.Fprintln(w, err)
      return
   }
   w.Header().Set("content-type", "application/json")
   w.WriteHeader(http.StatusOK)

   resp := map[string]string{
      "code":    "0000",
      "message": "上传成功",
      "data":    header.Filename,
   }
   rb, err := json.Marshal(resp)
   if err != nil {
      w.WriteHeader(http.StatusInternalServerError)
      fmt.Fprintln(w, err)
      return
   }
   w.Write(rb)
   return

}

func Download(w http.ResponseWriter, r *http.Request, pathParams map[string]string) {
   n := pathParams["name"]
   fi, err := os.Stat(n)
   if err != nil {
      fmt.Fprintln(w, err)
      return
   }

   data, err := os.ReadFile(n)
   if err != nil {
      fmt.Fprintln(w, err)
      return
   }
   w.Header().Set("Content-Type", "application/octet-stream")
   w.Header().Set("Content-Disposition", fi.Name())
   w.Write(data)
   return

}
```

## 使用中间件

grpc是允许我们使用中间件的，也就是在请求到达真正的处理逻辑的时候可以进行预制的拦截，grpc的服务端和客户端都为我们提供了拦截去的选项，简单来说就是需要我们进行配置

### 服务端

在服务端的` grpc.ServerOption`中有一个`grpc.UnaryInterceptor`的方法可以让我们添加拦截器

### 客户端

在客户端的`grpc.DialOption`中有一个`grpc.WithChainUnaryInterceptor`的方法让我们添加拦截器



### 读取请求头

用户自定义的请求头，需要我们额外的进行读取的配置，指定好那些请求头需要被我们读取，并且传递。这里需要借助到

`runtime.WithIncomingHeaderMatcher`,这个方法是我们在创建ServeMux时候需要指定的一个ServeMuxOption，假设我们需要用到`x-user-id	`	这个请求头，我们只需要

```go
sp := []runtime.ServeMuxOption{
 
   runtime.WithIncomingHeaderMatcher(func(s string) (string, bool) {
      if s == "X-User-Id" {
         return s, true
      }
      return runtime.DefaultHeaderMatcher(s)
   }),
}
mux := runtime.NewServeMux(sp...)
```

这样，x-user-id这个请求头的信息就会被传递到grpc的上下文中，这个时候，我们就可以在下文中获取到x-user-id的值

```go
userId := ""
md, _ := metadata.FromIncomingContext(ctx)
if len(md["x-user-id"]) == 0 {
   return userId
}

userId = strings.Join(md["x-user-id"], ",")
```

