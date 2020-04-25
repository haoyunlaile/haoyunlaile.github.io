---
title: kubeadm built kubernetes cluster
date: 2020-04-22
tags: ["kubernetes","kubeadm"]
categories:	kubernetes
---

## 需求

kubeadm 搭建kubernetes集群环境

## 准备条件

1. 三台VPS（本文使用**阿里云香港** - centos7.7）- 用国内的服务器折腾的好一会儿都被墙了，先不把时间浪费在这，直接上香港的服务器
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

### 安装kubelet、kubeadm、kubectl

```bash
# 禁用SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
systemctl enable --now kubelet
```

### 确保流量路由配置

```bash
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 初始化kubeadm

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16 --ignore-preflight-errors=Swap --upload-certs

W0423 19:49:48.841139   10624 configset.go:202] WARNING: kubeadm cannot validate component configs for API groups [kubelet.config.k8s.io kubeproxy.config.k8s.io]
[init] Using Kubernetes version: v1.18.2
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [izj6cbqyoktgsdsn8q6woqz kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 172.31.199.150]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [izj6cbqyoktgsdsn8q6woqz localhost] and IPs [172.31.199.150 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [izj6cbqyoktgsdsn8q6woqz localhost] and IPs [172.31.199.150 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
W0423 19:49:54.088376   10624 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
W0423 19:49:54.090110   10624 manifests.go:225] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.002596 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.18" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
c19b62a94f05d67b78200edba2e17e755e790606f19a935889714d32e73b663d
[mark-control-plane] Marking the node izj6cbqyoktgsdsn8q6woqz as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node izj6cbqyoktgsdsn8q6woqz as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: hn5kpn.qkrhb88jauu9aglz
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.31.199.150:6443 --token hn5kpn.qkrhb88jauu9aglz \
    --discovery-token-ca-cert-hash sha256:598ff568cd107d9ef9f780f86bd90051a0857da5197f5f8c19ed0dae8290366d 
```

请备份好 `kubeadm init` 输出中的 `kubeadm join` 命令，因为后面会需要这个命令来给集群添加节点

```bash
kubeadm join 172.31.199.150:6443 --token hn5kpn.qkrhb88jauu9aglz \
    --discovery-token-ca-cert-hash sha256:598ff568cd107d9ef9f780f86bd90051a0857da5197f5f8c19ed0dae8290366d 
```

### 允许普通用户可以运行 kubectl 

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 配置Calico网络

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 接下来将node节点加入集群

node节点同样上前面的流程安装docker、kubelet、kubeadm、kubectl。都安装完成后执行上面的`kubeadm join`命令

```bash
kubeadm join 172.31.199.150:6443 --token okdyiw.o8qjxl4v3avct79p \
>     --discovery-token-ca-cert-hash sha256:fb936d4c8da2dc276c6b447eea36d90bd9a206e1344c487418067faf580948e6 

W0422 12:27:17.321756    2133 join.go:346] [preflight] WARNING: JoinControlPane.controlPlane settings will be ignored when control-plane flag is not set.
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.18" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

然后在控制面板的机器上查看集群节点信息

```bash
kubectl get nodes

NAME                      STATUS     ROLES    AGE    VERSION
izj6cbqyoktgsdsn8q6woqz   Ready      master   55m    v1.18.2
izj6cbwwgp62jm8oudjra8z   Ready      <none>   2m7s   v1.18.2
izj6cbwwgp62jm8oudjra9z   NotReady   <none>   8s     v1.18.2
```

至此，kubeadm集群节点安装成功

清理集群节点信息

```bash
kubeamd reset
```

清理完成后记得执行下面的命令后重新配置config文件，不然会报错

> Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of “crypto/rsa: verification error” while trying to verify candidate authority certificate “kubernetes”)

```bash
rm -rf $HOME/.kube
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```



## 参考文献

 https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/ 

 https://kubernetes.io/zh/docs/setup/independent/create-cluster-kubeadm 

 https://kubernetes.io/zh/docs/reference/setup-tools/kubeadm/kubeadm/ 

 https://docs.projectcalico.org/getting-started/kubernetes/quickstart