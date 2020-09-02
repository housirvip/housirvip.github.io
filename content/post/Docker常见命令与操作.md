---
title: Docker常见命令与操作
date: 2018-05-20 12:03:19
tags:
  - Docker
  - Linux
---

## Docker 命令入门

### 拉取镜像

`docker pull <镜像名:tag>`

例如 `docker pull progrium/consul:latest`

### 列出存在的镜像

`docker images` 结果显示举例

```shell
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
swaggerapi/swagger-editor    latest              aaa304fc1b89        2 months ago        35.9MB
daocloud.io/library/ubuntu   16.04               14f60031763d        10 months ago       120MB
progrium/consul              latest              09ea64205e55        2 years ago         69.4MB
```

### 删除镜像

`docker rmi <镜像名> `

`docker rmi $(docker images | grep none | awk '{print $3}' | sort -r)` 删除所有镜像

### 创建容器

`docker run -t -i ubuntu:16.04 /bin/bash `

`docker run -d -p 80:80 -v /data:/data --name ubuntu ubuntu:16.04 /bin/bash `

`--name` 参数可以指定启动后的容器名字，如果不指定则docker会帮我们取一个名字

`-i -t` 立即进入到容器命令行操作，其中，`-t` 选项让Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上， `-i` 则让容器的标准输入保持打开。

后台运行可以填写 `-d`

`-p <host_port:contain_port>` 是端口暴露或者端口映射

```shell
-p 11211:11211 这个即是默认情况下，绑定主机所有网卡（0.0.0.0）的11211端口到容器的11211端口上
-p 127.0.0.1:11211:11211 只绑定localhost这个接口的11211端口
-p 127.0.0.1::5000
-p 127.0.0.1:80:8080
```

`-v <host_path:container_path>` 是目录映射

### 操作容器

#### 查看正在运行的容器

`docker ps`

`docker ps -a ` 为查看所有的容器，包括已经停止的

#### 启动关闭和重启

```shell
docker start <container_id_or_name>
docker stop <container_id_or_name>
docker restart <container_id_or_name>
docker kill <container_id_or_name>
```

#### 进入到正在运行的容器

`docker exec -it <container_id_or_name> /bin/bash`

操作完成后，直接 `ctrl + d` 或者直接输入 `exit` 退出即可

#### 删除容器

`docker rm <container_id_or_name>`

#### 查看容器user密码

`docker logs <container_id_or_name> 2>&1 | grep '^User: ' | tail -n1`

因为docker容器启动时的root用户的密码是随机分配的。所以，通过这种方式就可以得到redmine容器的root用户的密码了

#### 查看容器日志

`docker logs -f <container_id_or_name>`

## Docker 进阶操作

> 详见 Gitbook [Docker从入门到实践](https://github.com/yeasy/docker_practice)

### 修改Docker本地镜像与容器的存储位置

#### 方法一、软链接

默认情况下Docker的存放位置为：`/var/lib/docker` 可以通过下面命令查看具体位置：

```shell
sudo docker info | grep "Docker Root Dir"
```

解决这个问题，最直接的方法当然是挂载分区到这个目录，但是我的数据盘还有其他东西，这肯定不好管理，所以采用修改镜像和容器的存放路径的方式达到目的

这个方法里将通过软连接来实现

首先停掉Docker服务：

```shell
systemctl restart docker
或者
service docker stop
```

然后移动整个`/var/lib/docker`目录到目的路径：

```shell
mv /var/lib/docker /root/data/docker
ln -s /root/data/docker /var/lib/docker
```

这时候启动Docker时发现存储目录依旧是`/var/lib/docker`，但是实际上是存储在数据盘的，你可以在数据盘上看到容量变化

#### 方法二、修改镜像和容器的存放路径

指定镜像和容器存放路径的参数是`--graph=/var/lib/docker`，我们只需要修改配置文件指定启动参数即可。

Docker 的配置文件可以设置大部分的后台进程参数，在各个操作系统中的存放位置不一致，在 Ubuntu 中的位置是：`/etc/default/docker`，在 CentOS 中的位置是：`/etc/sysconfig/docker`。

如果是 CentOS 则添加下面这行：

```shell
OPTIONS=--graph="/root/data/docker" --selinux-enabled -H fd://
```

如果是 Ubuntu 则添加下面这行（因为 Ubuntu 默认没开启 selinux）：

```Shell
OPTIONS=--graph="/root/data/docker" -H fd://
# 或者
DOCKER_OPTS="-g /root/data/docker"
```

最后重新启动，Docker 的路径就改成 `/root/data/docker` 了

### 生成新的 image

`build`命令可以从`Dockerfile`和上下文来创建镜像：
`docker build [OPTIONS] PATH | URL | -`上面的`PATH`或`URL`中的文件被称作上下文，build  image的过程会先把这些文件传送到docker的服务端来进行的
如果`PATH`直接就是一个单独的`Dockerfile`文件则可以不需要上下文；如果`URL`是一个Git仓库地址，那么创建image的过程中会自动`git  clone`一份到本机的临时目录，它就成为了本次build的上下文。无论指定的`PATH`是什么，`Dockerfile`是至关重要的，请参考 [Dockerfile  Reference](http://docs.docker.com/reference/builder/)。
请看下面的例子：

```dockerfile
# cat Dockerfile 
FROM seanlook/nginx:bash_vim
EXPOSE 80
ENTRYPOINT /usr/sbin/nginx -c /etc/nginx/nginx.conf && /bin/bash

# docker build -t seanlook/nginx:bash_vim_Df .
Sending build context to Docker daemon 73.45 MB
Sending build context to Docker daemon 
Step 0 : FROM seanlook/nginx:bash_vim
 ---> aa8516fa0bb7
Step 1 : EXPOSE 80
 ---> Using cache
 ---> fece07e2b515
Step 2 : ENTRYPOINT /usr/sbin/nginx -c /etc/nginx/nginx.conf && /bin/bash
 ---> Running in e08963fd5afb
 ---> d9bbd13f5066
Removing intermediate container e08963fd5afb
Successfully built d9bbd13f5066 
```

### 给镜像打上标签

tag 的作用主要有两点：一是为镜像起一个容易理解的名字，二是可以通过`docker tag`来重新指定镜像的仓库，这样在`push`时自动提交到仓库。

```shell
# 将同一IMAGE_ID的所有tag，合并为一个新的
docker tag 195eb90b5349 seanlook/ubuntu:rm_test

# 新建一个tag，保留旧的那条记录
docker tag Registry/Repos:Tag New_Registry/New_Repos:New_Tag
```

 

 