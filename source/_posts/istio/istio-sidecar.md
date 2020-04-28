---
title: Istio Sidecar
date: 2020-04-28 21:00
tags: ["istio","istio-sidecar"]
categories:	istio
---

## Istio  Sidecar

###  概念及示例

 `Sidecar`描述了sidecar代理的配置。默认情况下，Istio 让每个 Envoy 代理都可以访问来自和它关联的工作负载的所有端口的请求，然后转发到对应的工作负载。您可以使用 [sidecar](https://preliminary.istio.io/zh/docs/reference/config/networking/sidecar/#Sidecar) 配置去做下面的事情：

- 微调 Envoy 代理接受的端口和协议集。
- 限制 Envoy 代理可以访问的服务集合。

您可能希望在较庞大的应用程序中限制这样的 sidecar 可达性，配置每个代理能访问网格中的任意服务可能会因为高内存使用量而影响网格的性能。

您可以指定将 sidecar 配置应用于特定命名空间中的所有工作负载，或者使用 `workloadSelector` 选择特定的工作负载。例如，下面的 sidecar 配置将 `bookinfo` 命名空间中的所有服务配置为仅能访问运行在相同命名空间和 Istio 控制平面中的服务。

>  每个名称空间只能有一个没有任何配置 `workloadSelector`的`Sidecar`配置 ， 如果`Sidecar`给定名称空间中存在多个不使用选择器的配置，则系统的行为是不确定的。 

<!-- more-->

下面声明的`Sidecar`在根名称空间中声明了全局默认配置`istio-config`，该配置在所有名称空间中配置了sidecar，以仅允许将流量发送到同一名称空间中的其他工作负载以及该名称空间中的服务`istio-system`。 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: `istio-config`
spec:
  egress:
  - hosts:
    - "./*"
    - "istio-system/*"
```

下面声明的`Sidecar`在配置`prod-us1` 命名空间覆盖全局默认以上定义，使出口流量到公共服务中`prod-us1`，`prod-apis`和`istio-system` 命名空间。 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  egress:
  - hosts:
    - "prod-us1/*"
    - "prod-apis/*"
    - "istio-system/*"
```

下面的示例`Sidecar`在`prod-us1`名称空间中声明一个配置，该配置接受端口9080上的入站HTTP通信并将其转发到侦听Unix域套接字的附加工作负载实例。在出口方向上，除了`istio-system`名称空间外，Sidecar仅代理绑定到端口9080的HTTP流量，以用于`prod-us1`名称空间中的服务 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  ingress:
  - port:
      number: 9080
      protocol: HTTP
      name: somename
    defaultEndpoint: unix:///var/run/someuds.sock
  egress:
  - port:
      number: 9080
      protocol: HTTP
      name: egresshttp
    hosts:
    - "prod-us1/*"
  - hosts:
    - "istio-system/*"
```

## Sidecar配置

| Field                   | Type                     | Description                                                  | Required |
| ----------------------- | ------------------------ | ------------------------------------------------------------ | -------- |
| `workloadSelector`      | `WorkloadSelector`       | Criteria used to select the specific set of pods/VMs on which this `Sidecar` configuration should be applied. If omitted, the `Sidecar` configuration will be applied to all workload instances in the same namespace. | No       |
| `ingress`               | `IstioIngressListener[]` | Ingress specifies the configuration of the sidecar for processing inbound traffic to the attached workload instance. If omitted, Istio will automatically configure the sidecar based on the information about the workload obtained from the orchestration platform (e.g., exposed ports, services, etc.). If specified, inbound ports are configured if and only if the workload instance is associated with a service. | No       |
| `egress`                | `IstioEgressListener[]`  | Egress specifies the configuration of the sidecar for processing outbound traffic from the attached workload instance to other services in the mesh. | Yes      |
| `outboundTrafficPolicy` | `OutboundTrafficPolicy`  | This allows to configure the outbound traffic policy. If your application uses one or more external services that are not known apriori, setting the policy to `ALLOW_ANY` will cause the sidecars to route any unknown traffic originating from the application to its requested destination. | No       |

### WorkloadSelector配置

`WorkloadSelector`指定的标准来确定是否`Gateway`， `Sidecar`或`EnvoyFilter`配置可被应用到一个代理。匹配条件包括与代理关联的元数据，工作负载实例信息或代理在初始握手期间提供给Istio的任何其他信息。如果指定了多个条件，则必须匹配所有条件才能选择工作负载实例。当前，仅支持基于标签的选择机制。 

| Field    | Type  | Description                                                  | Required |
| -------- | ----- | ------------------------------------------------------------ | -------- |
| `labels` | `map` | 一个或多个标签，指示`Sidecar`应在其上应用此配置的一组特定的Pod / VM 。标签搜索的范围仅限于存在资源的配置名称空间。 | Yes      |

## 参考文献

https://preliminary.istio.io/zh/docs/reference/config/networking/sidecar/#Sidecar