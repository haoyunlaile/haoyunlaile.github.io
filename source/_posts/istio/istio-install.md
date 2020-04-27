---
title: istio install
date: 2020-04-23
tags: istio
categories:	istio
---

## 需求

搭建istio基础环境

## 安装步骤

在安装 Istio 之前，需要一个运行着 Kubernetes 的环境，安装步骤可以参考前面的文章

下载istio，然后解压，然后将 `istioctl` 增加到 path 环境变量中 

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-1.5.1
export PATH=$PWD/bin:$PATH
```

 新建`istio-1.5.1.yaml` 配置文件、按照官方文档操作安装会出现错误，导致不能正常进行sidecar 自动注入

<!--more--> 

```
vim istio-1.5.1.yaml
```

```bash
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    egressGateways:
    - name: istio-egressgateway
      enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi

    ingressGateways:
    - name: istio-ingressgateway
      enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
        service:
          ports:
            ## You can add custom gateway ports in user values overrides, but it must include those ports since helm replaces.
            # Note that AWS ELB will by default perform health checks on the first port
            # on this list. Setting this to the health check port will ensure that health
            # checks always work. https://github.com/istio/istio/issues/12503
            - port: 15020
              targetPort: 15020
              name: status-port
            - port: 80
              targetPort: 8080
              name: http2
            - port: 443
              targetPort: 8443
              name: https
            - port: 31400
              targetPort: 31400
              name: tcp
              # This is the port where sni routing happens
            - port: 15443
              targetPort: 15443
              name: tls

    policy:
      enabled: false
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 100Mi

    telemetry:
      k8s:
        resources:
          requests:
            cpu: 50m
            memory: 100Mi

    pilot:
      k8s:
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                apiVersion: v1
                fieldPath: metadata.namespace
          - name: GODEBUG
            value: gctrace=1
          - name: PILOT_TRACE_SAMPLING
            value: "100"
          - name: CONFIG_NAMESPACE
            value: istio-config
        resources:
          requests:
            cpu: 10m
            memory: 100Mi

  addonComponents:
    kiali:
      enabled: true
    grafana:
      enabled: true
    tracing:
      enabled: true
    prometheus:
      enabled: true

  values:
    global:
      disablePolicyChecks: false
      proxy:
        accessLogFile: /dev/stdout
        includeIPRanges: 192.168.16.0/20,192.168.32.0/20
        autoInject: enabled  #配置自动注入
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
    sidecarInjectorWebhook:
      enableNamespacesByDefault: true

    pilot:
      autoscaleEnabled: false

    mixer:
      adapters:
        useAdapterCRDs: false
        kubernetesenv:
          enabled: true
        prometheus:
          enabled: true
          metricsExpiryDuration: 10m
        stackdriver:
          enabled: false
        stdio:
          enabled: true
          outputAsJson: false
      policy:
        autoscaleEnabled: false
      telemetry:
        autoscaleEnabled: false

    gateways:
      istio-egressgateway:
        autoscaleEnabled: true
      istio-ingressgateway:
        autoscaleEnabled: true
    kiali:
      createDemoSecret: true
```

安装对应配置

```bash
istioctl manifest apply -f istio-1.5.1.yaml
```

 验证是否安装成功 

```bash
kubectl get svc -n istio-system

NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
grafana                     ClusterIP      10.106.222.1     <none>        3000/TCP                                                                     72m
istio-egressgateway         ClusterIP      10.105.147.175   <none>        80/TCP,443/TCP,15443/TCP                                                     72m
istio-ingressgateway        LoadBalancer   10.101.90.130    <pending>     15020:31121/TCP,80:31729/TCP,443:31903/TCP,31400:32746/TCP,15443:31084/TCP   72m
istio-pilot                 ClusterIP      10.101.28.124    <none>        15010/TCP,15011/TCP,15012/TCP,8080/TCP,15014/TCP,443/TCP                     80m
istiod                      ClusterIP      10.99.35.177     <none>        15012/TCP,443/TCP                                                            80m
jaeger-agent                ClusterIP      None             <none>        5775/UDP,6831/UDP,6832/UDP                                                   72m
jaeger-collector            ClusterIP      10.109.237.212   <none>        14267/TCP,14268/TCP,14250/TCP                                                72m
jaeger-collector-headless   ClusterIP      None             <none>        14250/TCP                                                                    72m
jaeger-query                ClusterIP      10.103.4.63      <none>        16686/TCP                                                                    72m
kiali                       ClusterIP      10.100.49.221    <none>        20001/TCP                                                                    72m
prometheus                  ClusterIP      10.110.124.176   <none>        9090/TCP                                                                     72m
tracing                     ClusterIP      10.106.75.109    <none>        80/TCP                                                                       72m
zipkin                      ClusterIP      10.103.9.94      <none>        9411/TCP 
```

 确保关联的 Kubernetes pod 已经部署，并且 `STATUS` 为 `Running` 

```bash
kubectl get pods -n istio-system

NAME                                    READY   STATUS    RESTARTS   AGE
grafana-5f6f8cbf75-trjl6                1/1     Running   0          73m
istio-egressgateway-74896c8487-9qnwg    1/1     Running   0          73m
istio-ingressgateway-56f7dd5d6b-9c22z   1/1     Running   0          73m
istio-tracing-9dd6c4f7c-qr7vl           1/1     Running   0          73m
istiod-756bd84654-fqp7b                 1/1     Running   0          73m
istiod-756bd84654-hxpqt                 1/1     Running   0          73m
kiali-869c6894c5-p4h7r                  1/1     Running   0          73m
prometheus-c89875c74-lvq52              2/2     Running   0          73m
```

卸载istio

```bash
istioctl manifest generate --set profile=demo | kubectl delete -f -
```

## 参考文献

 https://preliminary.istio.io/zh/docs/setup/getting-started/ 