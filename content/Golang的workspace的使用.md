---
title: Golang的workspace的使用
 
categories: golang
comments: true
---


# 什么是go的workspaces?

go在1.18的时候发布了workspaces的概念[https://go.dev/doc/tutorial/workspaces]，可以让我们在编写多模块的项目的时候，变得更加的方便

# 示例
假如我们有一个比较大的项目，我们将其分解成两个模块，一个是通用模块common,一个是业务模块service。在service模块中需要用到common模块中的方法
```shell
    mkdir workspace
    cd workspace
```

创建common模块
```shell
    mkdir common
    cd common

    go mod init github.com/ethan/common
    touch adder.go
```
adder.go中存在一个公共的add方法，可以提供给其他的模块调用
```go
// adder.go
package common


func Adder(x int, y int) int {
	return x+y
}
```


创建service模块
```shell
    cd ..
    mkdir service
    cd service
    go mod init github.com/ethan/service

    touch service.go
```
在service.go 文件中，我们需要用到common模块里的Adder方法
```go
// service.go

package main

import (
	"fmt"
	"github.com/ethan/common"
)

func main() {
	fmt.Println(common.Adder(1, 2))
}
```
此时我们的项目结构
![20230706203308](https://img.ethanleo.top/20230706203308.png)
我们通过import导入了common包，这个时候我们需要通过`go mod tidy`将我们需要导入的包进行同步,这时我们会遇到错误
```shell
go: finding module for package github.com/ethan/common
github.com/ethan/service imports
        github.com/ethan/common: cannot find module providing package github.com/ethan/common: module github.com/ethan/common:
```
因为我们并没有将我们依赖的包推送到远程仓库，我们希望我们依赖的包是本地版本的common，当然我们可以通过`replace` 来解决这个问题
```shell
go mod edit -replace github.com/ethan/common=../common
```
执行完这个命令后我们的go.mod文件会被更新
```go

replace github.com/ethan/common => ../common

require github.com/ethan/common v0.0.0-00010101000000-000000000000

```

此时我们再使用`go mod tidy`可以进行更新，这样我们就可以使用本地的包

```go

go run .

3

```
通过`replace`我们可以达到依赖本地包的相关，但是我们可以采用更加方便的workspace的模式啦进行重构

首先我们删除`./service/go.mod`里的replace，在父目录中创建一个`go.work`文件
```shell
go work init
```
此时的目录结构
![20230706204212](https://img.ethanleo.top/20230706204212.png)
我们通过`go work use ./common`来表示我们需要使用到common包，这时我们的go.work文件中就会多出一行
```shell
use ./common
```

经过上面的修改，我们依然可以正常运行我们的程序
```go

go run .

3

```

# 注意

不要将我们的go.work文件推入远程仓库，在进行任何推送更改之前，确保我们讲go.work添加到了`.gitignore`或将其删除


