# Pod

## Pod是什么？

Pod是kubernetes中创建和管理的，**最小的可部署的计算单元**，是一组(一个或多个)容器。这些容器共享存储，网络。

<img src="/Users/lizi/Library/Application Support/typora-user-images/image-20220810193757769.png" alt="具有两个容器的Pod"  />

其实Pod是一组**逻辑概念**，因为Pod共享一组NameSpace,Cgroup。

<!--more-->

## 使用Pod

创建一个Pod可以使用两种方式一种是通过命令行的方式(这里用nginx的镜像举例子)

`kubectl run nginx --image=nginx --restart=Never `

第二种是通过Yaml文件的方式创建Pod

```yaml
# nginx-pod.yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
spec:
  containers:
  - image: nginx
    name: nginx
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Never
status: {}


# kubectl apply -f nginx-pod.yaml
```

在创建完成之后，我们可以查看pod的一个创建状态

```shell
~ kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          6m50s
```

通过STATUS的值，可以看到Nginx已经被创建出来了。下面我们尝试创建出一个具有两个容器的Pod

