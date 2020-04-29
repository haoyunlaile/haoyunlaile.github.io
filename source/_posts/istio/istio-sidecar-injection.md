---
title: Istio Proxy 注入
date: 2020-04-29
tags: ["istio","istio-proxy"]
categories:	istio
---

## sidecar注入

简单来说，Sidecar 注入会将额外容器的配置添加到 Pod 模板中。Istio 服务网格目前所需的容器有：

`istio-init` [init 容器](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)用于设置 iptables 规则，以便将入站/出站流量通过 sidecar 代理。初始化容器与应用程序容器在以下方面有所不同：

- 它在启动应用容器之前运行，并一直运行直至完成。
- 如果有多个初始化容器，则每个容器都应在启动下一个容器之前成功完成。

因此，您可以看到，对于不需要成为实际应用容器一部分的设置或初始化作业来说，这种容器是多么的完美。在这种情况下，`istio-init` 就是这样做并设置了 `iptables` 规则。

`istio-proxy` 这个容器是真正的 sidecar 代理（基于 Envoy）。

<!--more--> 

下面的内容描述了向 pod 中注入 Istio sidecar 的两种方法：使用 [`istioctl`](https://preliminary.istio.io/zh/docs/reference/commands/istioctl) 手动注入或启用 pod 所属命名空间的 Istio sidecar 注入器自动注入。

手动注入直接修改配置，如 deployment，并将代理配置注入其中。

当 pod 所属`namespace`启用自动注入后，自动注入器会使用准入控制器在创建 Pod 时自动注入代理配置。

通过应用 `istio-sidecar-injector` ConfigMap 中定义的模版进行注入。

### 自动注入 sidecar

当你在一个`namespace`中设置了 `istio-injection=enabled` 标签，且 injection webhook 被启用后，任何新的 pod 都有将在创建时自动添加 sidecar.  请注意，区别于手动注入，自动注入发生在 pod 层面。你将看不到 deployment 本身有任何更改 。

```shell
kubectl label namespace xxx istio-inhection=enabled
```

### 手动注入 sidecar

手动注入 deployment ，需要使用 使用 `istioctl kube-inject`

```shell
istioctl kube-inject -f samples/sleep/sleep.yaml | kubectl apply -f -
```

 默认情况下将使用集群内的配置，或者使用该配置的本地副本来完成注入。 

```shell
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.config}' > inject-config.yaml
kubectl -n istio-system get configmap istio-sidecar-injector -o=jsonpath='{.data.values}' > inject-values.yaml
kubectl -n istio-system get configmap istio -o=jsonpath='{.data.mesh}' > mesh-config.yaml
```

 指定输入文件，运行 `kube-inject` 并部署 

```shell
istioctl kube-inject \
    --injectConfigFile inject-config.yaml \
    --meshConfigFile mesh-config.yaml \
    --valuesFile inject-values.yaml \
    --filename samples/sleep/sleep.yaml \
    | kubectl apply -f -
```

 验证 sidecar 已经被注入到 READY 列下 `2/2` 的 sleep pod 中 

```shell
kubectl get pod  -l app=sleep
NAME                     READY   STATUS    RESTARTS   AGE
sleep-64c6f57bc8-f5n4x   2/2     Running   0          24s
```

### 卸载 sidecar 自动注入器

```shell
kubectl delete mutatingwebhookconfiguration istio-sidecar-injector
kubectl -n istio-system delete service istio-sidecar-injector
kubectl -n istio-system delete deployment istio-sidecar-injector
kubectl -n istio-system delete serviceaccount istio-sidecar-injector-service-account
kubectl delete clusterrole istio-sidecar-injector-istio-system
kubectl delete clusterrolebinding istio-sidecar-injector-admin-role-binding-istio-system
```

上面的命令不会从 pod 中移除注入的 sidecar。需要进行滚动更新或者直接删除对应的pod，并强制 deployment 重新创建新pod。 

## 参考文献

 https://preliminary.istio.io/zh/docs/setup/additional-setup/sidecar-injection/#automatic-sidecar-injection 

 https://preliminary.istio.io/zh/blog/2019/data-plane-setup/ 