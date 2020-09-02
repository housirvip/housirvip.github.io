---
title: Docker+Consul服务发现
date: 2018-05-20 14:36:37
tags:
  - Docker
  - Web
  - Consul
---

## 前言

[Docker从入门到实践](https://yeasy.gitbooks.io/docker_practice/)
细节部分讲的不是特别细，很多知识还是得各种查资料。

[Docker文档](https://docs.docker.com/)

[consul文档](https://www.consul.io/docs/guides/index.html)
如果你选择 Consul，这个是必读的

[consul的docker镜像](https://hub.docker.com/r/progrium/consul/)
包装 Consul 的镜像，简化了 Consul 的部署

## 架构

![](http://static.nicesite.vip/2018-06-12-084651.jpg)

* 首先架构应该是，简单的，可迭代演进的。这样可操作性和可维护性会更强。
* 谈架构，无外乎就是高可用和可扩展，脱离这两个都是耍流氓。还有就是省钱，动不动20台服务器，创业公司伤不起。所以，解决好了就是好架构。
* 监控方案是后续迭代演进的事，你必须要保证你的系统正常运转，才能缩短开发周期。留出更多的时间，你可以做这些重要的事。
* 关于负载均衡器，有很多备选方案，现在云服务这么发达，可选的方案也很多，甚至有跨机房的负载均衡。比自己搭 nginx+keepalived 要方便的多。
* 选择 Consul ，用于服务发现，解决的是服务互访的问题。

### 架构原理

第一步，所有应用启动之后会向 Consu 集群注册自己，注册的信息包括

- 所属数据中心 DC1
- 所属数据中心的宿主机节点
- 所属节点的服务，服务访问方式 ip ，端口

应用在启动的时候往 Consul 注册Api发送注册服务信息。后期 Consul 会负责服务节点的健康检查。

第二步，当应用间存在访问时，如 Api 网关访问微服务，Web应用访问微服务，微服务之间互访。这里可以使用Consul Api 定期请求服务状态的方式，来获取可用的节点，后面会详细介绍。请求到节点后还可以在应用程序级别做一些负载均衡策略。

### 安装虚拟机

VirtualBox 作为实验性项目，使用 VirutalBox 可以快速构建你想要的物理环境，而且 Docker和 Virtualbox 搭配的很好，使用 Docker-machine 可以非常简单的管理所有虚拟机。

## 开始

好了，万事具备。现在我们开始创建虚拟机。

使用 Docker 工具包自带的 Docker-machine 工具，可以帮你快速创建一个 Docker 宿主机。

在这个架构中，我们一共只需要创建 3 台宿主机

Docker-machine 命令后面会用的比较频繁，所以我们改个短点的名字。

这里我用 zsh，bash 类似。

```shell
$ vi ~/.zshrc
#增加
alias dm="docker-machine"
```

依次创建3台虚拟机

 ```Shell
$ dm create -d "virtualbox" node1
$ dm create -d "virtualbox" node2
$ dm create -d "virtualbox" node3
 ```

ip 是自动分配的，不出意外的话，会得到下面对应的 ip（如果真出意外了，就改改 ip 吧）

`$ dm ls`

```shell
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
node1   -        virtualbox   Running   tcp://192.168.99.100:2376           v17.06.0-ce
node2   -        virtualbox   Running   tcp://192.168.99.101:2376           v17.06.0-ce
node3   -        virtualbox   Running   tcp://192.168.99.102:2376           v17.06.0-ce
```

### 配置 Consul Server

宿主机 node1 我们新开一个终端 

`$ dm ssh node1`

启动第一台 Consul Server，非常简单，一条命令搞定

```shell
$ docker run -h node1 --name consul -d -v /data:/data --restart=always \
    -p   8300:8300 \
    -p   8301:8301 \
    -p   8301:8301/udp \
    -p   8302:8302 \
    -p   8302:8302/udp \
    -p   8400:8400 \
    -p   8500:8500 \
progrium/consul -server \
-bootstrap-expect 3 \
-advertise 192.168.99.100
```

下面来解释下各个参数

`-h` 节点名字

`—name` 容器（container）名称，后期用来方便启动关闭，看日志等，这个一定要写

`-d` 后台运行

`-v /data:/data` 使用宿主机的 /data 目录映射到容器内部的 /data,用于保存 Consul 的注册信息，要不 Docker 一重启，数据是不保留的。

`--restart=always`  这个可以活得长一点

下面几个参数都是 Consul 集群用的，非集群模式可以不使用。

```shell
-p   8301:8301 
-p   8301:8301/udp 
-p   8302:8302 
-p   8302:8302/udp
```

`progrium/consul` 镜像名称，本地没有就自动从公共 Docker 库下载

后面的都是 Consul 的参数：

`-server`   以服务节点启动

`-bootstrap-expect 3 ` 预期的启动节点数 3，最少是 3，要不达不到 Cluster 的效果，如果只有一台机器的话可以修改 3 为 1

`-advertise 192.168.99.100`  告诉集群，我的 ip 是什么，就是注册集群用的

执行完毕后 ，使用 `docker ps` 看下，是否运行正常。`docker logs` 就不用看了，里面各种警告和错误，其实那都是假象。

但是 Consul Cluster 你必须明白，只有3个 Consul Server 节点都启动正常了，整个集群才能正常启动。

### 配置下一台 Consul Server

开启一个新的终端

```shell
$ dm ssh node2 #进入 node2 宿主机
$ docker run -h node2 --name consul -d -v /data:/data   --restart=always \
    -p   8300:8300 \
    -p   8301:8301/udp \
    -p   8302:8302 \
    -p   8302:8302/udp \
    -p   8400:8400 \
    -p   8500:8500 \
progrium/consul -server \
-advertise 192.168.99.101 \
-join  192.168.99.100
$ dm ssh node3 #进入 node3 宿主机
$ docker run -h node3 --name consul -d -v /data:/data   --restart=always \
    -p   8300:8300 \
    -p   8301:8301/udp \
    -p   8302:8302 \
    -p   8302:8302/udp \
    -p   8400:8400 \
    -p   8500:8500 \
progrium/consul -server \
-advertise 192.168.99.102 \
-join  192.168.99.100
```

这里多了一个参数
`-join  192.168.99.100` 代表的是加入 node1 建立好的 Consul Server

### Consul 配置完毕

检查是否成功，没出意外的话，就看到下面的界面 http://192.168.99.100:8500

![](http://static.nicesite.vip/2018-06-12-084532.jpg)

## 总结

总的来说，这个已经算是极简的架构了。当然，docker的生命周期远不止这些，比如ci，发布上线，灰度发布等。docker远没有传说中那么简单，美妙。docker有很多的好处，但是需要DevOPS做很多的工作。

在实践过程中，我发现有一个或许是更优的架构。那就是seneca的方案，使用事件相应作为微服务的提供方式，这样就避免了服务发现这件事，完全不需要注册服务，选择服务这么麻烦。也不用为服务发现服务搭建一个集群确保其高可用。

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 