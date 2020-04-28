---
title: centos7 install docker
date: 2020-04-21 8:00
tags: docker
categories:	docker
---

## 需求

安装docker运行环境

## 准备条件

1. 一台VPS（本文使用**阿里云香港** - centos7.7）
2. 一台能SSH连接到VPS的本地电脑 （推荐连接工具xshell）

## 安装步骤

安装docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh --mirror Aliyun
```

配置docker访问加速

```bash
tee /etc/docker/daemon.json <<-'EOF'
{
 "registry-mirrors": [
 	 "https://ze9vyrof.mirror.aliyuncs.com",
     "https://registry.docker-cn.com",
     "http://f1361db2.m.daocloud.io",
     "https://docker.mirrors.ustc.edu.cn"
 ]
}
EOF
```

<!--more--> 

创建用户并加入docker组

```
useradd -g docker docker
usermod -aG docker docker
```

加入开启启动、启动docker应用

```
systemctl enable docker
systemctl start docker
```

测试 Docker 是否安装正确

```bash
docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
0e03bdcc26d7: Pulling fs layer 
latest: Pulling from library/hello-world
0e03bdcc26d7: Pull complete 
Digest: sha256:8e3114318a995a1ee497790535e7b88365222a21771ae7e53687ad76563e8e76
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

## 参考文献

 https://docs.docker.com/engine/install/centos/ 