---
title: Minikube install
date: 2020-04-21
tags: ["kubernetes","minikube"]
categories:	kubernetes
---

## 需求

安装kubernetes - Minikube本地环境

## 准备条件

1. 一台VPS（本文使用**阿里云香港** - centos7.7）- 用国内的服务器折腾的好一会儿都被墙了，先不把时间浪费在这，直接上香港的服务器
2. 一台能SSH连接到VPS的本地电脑 （推荐连接工具xshell）

## 安装步骤

在安装前需要配置国内的镜像源，这里推荐使用阿里云的

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

### 安装kubectl

```bash
yum install -y kubectl
```

  shell 中开启 kubectl 命令自动补全 

```bash
yum install bash-completion -y
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

### 安装minukube

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
mkdir -p /usr/local/bin/
install minikube /usr/local/bin/
su - docker
```

###  启动本地 Kubernetes 集群、 检查集群的状态 

```
minikube start

* minikube v1.9.2 on Centos 7.7.1908
* Automatically selected the docker driver
* Starting control plane node m01 in cluster minikube
* Pulling base image ...
* Downloading Kubernetes v1.18.0 preload ...
    > preloaded-images-k8s-v2-v1.18.0-docker-overlay2-amd64.tar.lz4: 542.91 MiB
* Creating Kubernetes in docker container with (CPUs=2) (4 available), Memory=2200MB (7821MB available) ...
* Preparing Kubernetes v1.18.0 on Docker 19.03.2 ...
  - kubeadm.pod-network-cidr=10.244.0.0/16
* Enabling addons: default-storageclass, storage-provisioner
! Enabling 'default-storageclass' returned an error: running callbacks: [chmod: chmod deploy/addons/storageclass/storageclass.yaml.tmpl: permission denied]
* Done! kubectl is now configured to use "minikube"

```

```
minikube status

m01
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

```
kubectl cluster-info

Kubernetes master is running at https://172.17.0.2:8443
KubeDNS is running at https://172.17.0.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 开启Kubernetes dashboard服务

```bash
minikube dashboard --url

* Enabling dashboard ...
* Verifying dashboard health ...
* Launching proxy ...
* Verifying proxy health ...
http://127.0.0.1:33457/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

###  开启kube-proxy端口映射，使其可以远程访问

```
kubectl proxy --port=33458 --address='0.0.0.0' --accept-hosts='^.*' &
```

这里需要记得去阿里云的安全组配置33458端口外网可以访问

然后就可以在浏览器访问k8s的dashborad了

http://127.0.0.1:33458/

![k8s-dashboard](/images/k8s-dashboard.png)

清理 minikube 的本地状态 

```bash
minikube delete
```



## 参考文献

 https://juejin.im/post/5b8a4536e51d4538c545645c 

 https://kubernetes.io/zh/docs/tasks/tools/install-minikube/ 

 https://github.com/kubernetes/minikube 

 https://minikube.sigs.k8s.io/docs/handbook/dashboard/ 