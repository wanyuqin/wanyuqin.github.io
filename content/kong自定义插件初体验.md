---
title: Kong自定义插件初体验
categories: 日常学习
comments: true
---

此文章记录一下Kong自定义插件初次编写的过程，如果文章中有错误请及时提醒并帮忙更正

## 安装开发环境
这里使用[Pongo](https://github.com/Kong/kong-pongo) 来进行插件的测试，而Pongo的使用需要具备docker-compose和curl，再这些都具备之后就可以安装Pongo CLI了

<!-- more -->

``` shell
PATH=$PATH:~/.local/bin
git clone https://github.com/Kong/kong-pongo.git
mkdir -p ~/.local/bin
ln -s $(realpath kong-pongo/pongo.sh) ~/.local/bin/pongo
```
## 参考插件模版
[kong-plugin](https://github.com/Kong/kong-plugin) 为我们提供了一个简单的插件模版，这个模版使用的是Lua语言进行的开发
```sehll
git clone https://github.com/Kong/kong-plugin.git kong-api-version-plugin
```
打开这个项目
![enter image description here](https://img.ethanleo.top/20220509224116_afCtHi_703295BE-436A-40FC-B850-3D4AEAA96DBA.png)

### 目录结构
* kong/plugins 插件内容主要存放的目录，plugins下面的每个文件夹都表示一个功能插件
* spec 测试文件存放的目录

这里我把原来的文件名以及插件名称修改成了api-version，再修改完成之后我们可以执行一下 ` pongo run` ,此命令会将spec目录下的测试方法全部执行一遍，以保证我们的插件编写无误，同时`pongo run`会将我们的插件进行加载。


在我们的测试通过之后，我们就可以先尝试对kong进行插件安装了，我们执行 `pongo shell`	进入到pongo的控制台，初始化kong的表，并且启动kong
```
kong migrations bootstrap --force
kong start
```
再启动完kong之后，我们进行验证 执行`curl -i http://localhost:8000/`,如果你看到返回了一个错误，并且错误信息是`{"message":"no Route matched with those values"}`,说明kong是正常启动的

## 测试插件
在进行插件测试之前，我们需要添加一个kong的服务，这里的话用kong`s Admin API 去进行操作
```
curl -i -X POST \
 --url http://localhost:8001/services/ \
 --data 'name=example-service' \
 --data 'url=http://konghq.com'
```
在添加完service之后，我们需要添加一个route，这样kong就可以进行代理我们的请求
```shell
curl -i -X POST \
 --url http://localhost:8001/services/example-service/routes \
 --data 'hosts[]=example.com'
```

最后我们将插件添加进入Service
```shell
curl -i -X POST \
 --url http://localhost:8001/services/example-service/plugins/ \
 --data 'name=api-version'
```
再以上操作都执行结束并且成功后，我们就可以对插件进行测试，执行
`curl -I -H "Host:example.com http://localhost:8000/"`
如果返回的结果的头信息中带有`Bye-World`的key就说明一切顺利

## 编写自己的插件
上面的步骤都是介绍如何安装开发环境，以及体验kong模版，接下来我们尝试写一个最简单的插件（这里只是简单实验一下，毕竟没有深入研究实验）
在上面介绍kong的模版项目的时候有说到plugins文件夹，这里面存放的是插件开发的主要文件而每个插件文件夹中也有两个主要的文件：
*  *handler.lua * 这个文件里面编写了自定义插件主要的业务逻辑
*  *schema.lua*  这个文件根据官方的解释是说，存放一些额外的配置，比如用户可以提供key/value来改变行为，那么这些行为就存放在此，但是本人还没有进行深入了解，所以这个文件暂时就不太清楚是在那些情况下可以发挥作用

那么在简单了解了那些主要文件以及他们的作用之后，我们就可以开始自己的编码了，我这边写了一个比较简单的插件，主要的功能就是检查请求头中是否带有**x-user-id**这个key，如果没有就返回403，如果有那么就将这个k-v写入返回的头信息中，下面贴一下 *handler.lua* 的代码
![enter image description here](https://img.ethanleo.top/20220509224219_SEjPLC_A80C9954-EEC3-42DB-B973-0769D0334DF6.png)
其实我们自定义的方法都算是实现了kong内置的一些方法
具体的编写内容大家可以参考[custom-logic](https://docs.konghq.com/gateway-oss/2.0.x/plugin-development/custom-logic/?&_ga=2.71838099.1440650078.1652062070-1620358928.1651822858#available-contexts) 

再写完自己的业务逻辑代码之后，还需要在spec文件夹下面创建两个测试代码文件，这里的话我是根据模版的测试代码进行修改的，具体的没有深入研究，编写两个测试方法，一个是带有x-user-id请求头的测试方法，一个是不带有的测试方法，具体代码如下
![enter image description here](https://img.ethanleo.top/20220509224257_RC514I_68F9D490-B6DD-4902-BEDE-5D523B6179E5.png)

在完成`pongo run `测试之后，我们可以重复 **测试插件** 的步骤进行我们自定义插件的测试

参考文章：
 [How to Create a Custom Lua Plugin for Kong Gateway](https://konghq.com/blog/custom-lua-plugin-kong-gateway) 
[官方文档](https://docs.konghq.com/gateway-oss/2.0.x/plugin-development/custom-logic/?&_ga=2.71838099.1440650078.1652062070-1620358928.1651822858#available-contexts)