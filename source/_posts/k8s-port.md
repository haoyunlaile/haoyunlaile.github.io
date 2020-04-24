---
title: kubernetes port list
date: 2020-04-22
tags: kubernetes
categories:	kubernetes
---

## 搭建k8s环境所需端口清单

### 控制平面节点端口清单

| 协议 | 方向 | 端口范围  |          作用           |            使用者            |
| :--: | :--: | :-------: | :---------------------: | :--------------------------: |
| TCP  | 入站 |   6443    |  Kubernetes API 服务器  |           所有组件           |
| TCP  | 入站 | 2379-2380 | etcd server client API  |     kube-apiserver, etcd     |
| TCP  | 入站 |   10250   |       Kubelet API       |  kubelet 自身、控制平面组件  |
| TCP  | 入站 |   10251   |     kube-scheduler      |     kube-scheduler 自身      |
| TCP  | 入站 |   10252   | kube-controller-manager | kube-controller-manager 自身 |

### Node节点端口清单

| 协议 | 方向 |  端口范围   |      作用       |           使用者           |
| :--: | :--: | :---------: | :-------------: | :------------------------: |
| TCP  | 入站 |    10250    |   Kubelet API   | kubelet 自身、控制平面组件 |
| TCP  | 入站 | 30000-32767 | NodePort 服务** |          所有组件          |

## 参考资料

 https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ 