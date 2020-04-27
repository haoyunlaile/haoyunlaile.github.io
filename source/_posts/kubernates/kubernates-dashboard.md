---
title: kubernetes dashboard install
date: 2020-04-22
tags: kubernetes-dashboard
categories:	kubernetes
---

## 需求

基于网页查看Kubernetes 用户界面 

## 安装步骤

在控制面板节点部署dashborad

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
```

### 放开外网端口映射

> 如果需要外网访问，需要使用NodePort的方式对外暴露端口，不能使用`kubectl proxy`的方式，因为该方式只能通过http访问，非本地环境无法正常登录，在这里折腾了好几个小时，主要还是没有一字一句看官方文档。

更改原文件`type: ClusterIP` 为`type: NodePort `后保存

<!--more--> 

```
kubectl -n kubernetes-dashboard edit service kubernetes-dashboard

# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
...
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  resourceVersion: "343478"
  selfLink: /api/v1/namespaces/kubernetes-dashboard/services/kubernetes-dashboard
  uid: 8e48f478-993d-11e7-87e0-901b0e532516
spec:
  clusterIP: 10.100.124.90
  externalTrafficPolicy: Cluster
  ports:
  - port: 443
    protocol: TCP
    targetPort: 8443
  selector:
    k8s-app: kubernetes-dashboard
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}
```

### 下一步获取nodeport对外开放的https端口，注意这里为32443端口

```bash
kubectl -n kubernetes-dashboard get service kubernetes-dashboard

NAME                   TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.98.33.83   <none>        443:32443/TCP   77m
```

### 同时启动监控指标收集服务，不然会dashborad无法展示数据图表

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.6/components.yaml
```

然后就可以访问下面的地址

`https://<master-ip>:<nodePort> `

访问上面的地址会出现登录的界面，如下图：

![k8s-dashboard-login](/images/k8s-dashboard-login.png)

这里选择使用token登录

创建dashboard对应的admin账户

```bash
touch dashboard-admin.yml
vi dashboard-admin.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

```bash
kubectl apply -f dashboard-admin.yml
```

然后通过如下命令获取登录的token

```bash
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')

Name:         admin-user-token-5j9gg
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 8b1c0aa8-9ee1-4c06-a983-6cc8ebecf8b2

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Im5BZWtISUdnVnloMDJiRjdLZ0pJdTMxNXZ2YTdtY2U2Z0p3QURlblFnSEEifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLTVqOWdnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbbWluLXVzZXIiLCJrddWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI4YjFjMGFhOC05ZWUxLTRjMDYtYTk4My02Y2M4ZWJlY2Y4YjIiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZXJuZXRlcy1kYXNoYm9hcmQ6YWRtaW4tdXNlciJ9.C4Ma3tp6GdMCusjPEaQNqm_92-PEEm02-68OvsMq1eExPvMZxYYrvmwSWwOnJIps5mL2BEu1XqchsWlNFYpawe5HIk_zrimfff-NpwVRqxu0qPt0MxN0KzVgMm5hOaOYKYJW0zz1mpFZI8-uvqdDzwJGFan7vLH1KTCUt5gTHlv-KJyYa6zmE2QKl0-IATcesCF0sU51K2F5NeSU9dvE9hJ92mcETuGwXsuPo5aPSu-1yi1WFnaWDQrcJseXxOWREaYv0o-9swCZOYYBdNy7G4h6xB6cWxUD7C5Un4lB-5VaBqD0D_hS5Cwh3S5ETKYikag6-tB_sOdG7w-KuONicQ

```

取上图的token字段粘贴进登录界面即可。

> 注意，有的文章会写此方式获取到的token还需要进行base64解密，可能是因为版本原因，本人测试是可以直接复制后进行登录的

登录成功后界面如图

![k8s-dashboard-nodes.png](/images/k8s-dashboard-nodes.png)

终于装好了，踩了不少坑，主要还是不熟悉。后面切记认真仔细阅读官方文档。

## 参考文档

 https://kubernetes.io/zh/docs/tasks/access-application-cluster/web-ui-dashboard/ 

 https://github.com/kubernetes/dashboard 

 https://github.com/kubernetes-sigs/metrics-server

 https://github.com/kubernetes/dashboard/blob/master/docs/user/accessing-dashboard/1.7.x-and-above.md 