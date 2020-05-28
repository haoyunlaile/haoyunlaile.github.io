---
title: Istio DestinationRule
date: 2020-04-28 19:00
tags: ["istio","istio-destinationrule"]
categories:	istio
---

## Istio  DestinationRule 目标规则

###  概念及示例

与`VirtualService`一样，`DestinationRule`也是 Istio 流量路由功能的关键部分。您可以将虚拟服务视为将流量如何路由到给定目标地址，然后使用目标规则来配置该目标的流量。在评估虚拟服务路由规则之后，目标规则将应用于流量的“真实”目标地址。

特别是，您可以使用目标规则来指定命名的服务子集，例如按版本为所有给定服务的实例分组。然后可以在虚拟服务的路由规则中使用这些服务子集来控制到服务不同实例的流量。

目标规则还允许您在调用整个目的地服务或特定服务子集时定制 Envoy 的流量策略，比如您喜欢的负载均衡模型、TLS 安全模式或熔断器设置。在目标规则参考中可以看到目标规则选项的完整列表。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: my-destination-rule
spec:
  host: my-svc
  trafficPolicy:
    loadBalancer:
      simple: RANDOM
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
    trafficPolicy:
      loadBalancer:
        simple: ROUND_ROBIN
  - name: v3
    labels:
      version: v3
```

每个子集都是基于一个或多个 `labels` 定义的，在 Kubernetes 中它是附加到像 Pod 这种对象上的键/值对。这些标签应用于 Kubernetes 服务的 Deployment 并作为 `metadata` 来识别不同的版本。

除了定义子集之外，目标规则对于所有子集都有默认的流量策略，而对于该子集，则有特定于子集的策略覆盖它。定义在 `subsets` 上的默认策略，为 `v1` 和 `v3` 子集设置了一个简单的随机负载均衡器。在 `v2` 策略中，轮询负载均衡器被指定在相应的子集字段上。

### host字段

使用 Kubernetes `Service`的短名称。含义同VirtualService 中destination 的 host字段一致。服务 一定要存在于对应的服务注册中心中，否则会被忽略。

### loadBalancer字段

默认情况下，Istio 使用轮询的负载均衡策略，实例池中的每个实例依次获取请求。Istio 同时支持如下的负载均衡模型，可以在 `DestinationRule` 中为流向某个特定服务或服务子集的流量指定这些模型。

- 随机：请求以随机的方式转到池中的实例。
- 权重：请求根据指定的百分比转到实例。
- 最少请求：请求被转到最少被访问的实例。

### subsets字段

`subsets`是服务端点的集合，可以用于 A/B 测试或者分版本路由等场景。可以将一个服务的流量切分成N份供客户端分场景使用。`name`字段定义后主要供 `VirtualService` 里destination 使用。 每个子集都是在`host`对应服务的基础上基于一个或多个 `labels` 定义的，在 Kubernetes 中它是附加到像 Pod 这种对象上的键/值对。这些标签应用于 Kubernetes 服务的 Deployment 并作为 `metadata` 来识别不同的版本。 

## DestinationRule配置

| Field           | Type            | Description                                                  | Required |
| --------------- | --------------- | ------------------------------------------------------------ | -------- |
| `host`          | `string`        | 表示规则的适用对象，取值是在服务注册中心注册的服务名，可以是网格内的服务，也可以是以 ServiceEnrty 方式注册的网格外的服务。如果这个服务名在服务注册中心不存在，则这个规则无效。host 如果取短域名，则会根据规则所在的命名空间进行解析。 | Yes      |
| `trafficPolicy` | `TrafficPolicy` | 流量策略，包括负载均衡、连接池策略、异常点检查等             | No       |
| `subsets`       | `Subset[]`      | 是定义的一个服务的子集，经常用来定义一个服务版本，结合 VirtualService 使用 | No       |
| `exportTo`      | `string[]`      | 当前destination rule要导出的 namespace 列表。 应用于 service 的 destination rule 的解析发生在 namespace 层次结构的上下文中。 destination rule 的导出允许将其包含在其他 namespace 中的服务的解析层次结构中。 此功能为服务所有者和网格管理员提供了一种机制，用于控制跨 namespace 边界的 destination rule 的可见性<br/>如果未指定任何 namespace，则默认情况下将 destination rule 导出到所有 namespace<br/>值`.` 被保留，用于定义导出到 destination rule 被声明所在的相同 namespace 。类似的值`*`保留，用于定义导出到所有 namespaces<br/>NOTE：在当前版本中，exportTo值被限制为`.`或`*`（即， 当前namespace或所有namespace） | No       |

### subsets配置

| Field           | Type            | Description                                                  | Required |
| --------------- | --------------- | ------------------------------------------------------------ | -------- |
| `name`          | `string`        | 服务名和 `subset` 名称可以用于路由规则中的流量拆分           | Yes      |
| `labels`        | `map`           | 使用标签对服务注册表中的服务端点进行筛选                     | No       |
| `trafficPolicy` | `TrafficPolicy` | 应用到这一 `subset` 的流量策略。缺省情况下 `subset` 会继承 `DestinationRule` 级别的策略，这一字段的定义则会覆盖缺省的继承策略 | No       |

具体细节的参数明细可查阅：https://preliminary.istio.io/zh/docs/reference/config/networking/destination-rule/#DestinationRule 

## 参考文献

 https://preliminary.istio.io/zh/docs/concepts/traffic-management/#destination-rules 

 https://preliminary.istio.io/zh/docs/reference/config/networking/destination-rule/ 

 https://preliminary.istio.io/zh/docs/reference/config/networking/destination-rule/#DestinationRule 