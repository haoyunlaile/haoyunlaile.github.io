---
title: Istio ServiceEntry
date: 2020-04-28 20:00
tags: ["istio","istio-serviceentry"]
categories:	istio
---

## Istio  ServiceEntry 外部服务引入

###  概念及示例

使用[服务入口`Service Entry`来添加一个入口到 Istio 内部维护的服务注册中心。添加了服务入口后，Envoy 代理可以向服务发送流量，就好像它是网格内部的服务一样。配置服务入口允许您管理运行在网格外的服务的流量，它包括以下几种能力：

- 为外部目标 redirect 和转发请求，例如来自 web 端的 API 调用，或者流向遗留老系统的服务。
- 为外部目标定义重试、超时和故障注入策略。
- 添加一个运行在虚拟机的服务来扩展您的网格。
- 从逻辑上添加来自不同集群的服务到网格，在 Kubernetes 上实现一个[多集群 Istio 网格](https://preliminary.istio.io/zh/docs/setup/install/multicluster/gateways/#configure-the-example-services)。

<!-- more-->

您不需要为网格服务要使用的每个外部服务都添加服务入口。默认情况下，Istio 配置 Envoy 代理会将请求传递给未知服务。但是，您不能使用 Istio 的特性来控制没有在网格中注册的目标流量。

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: svc-entry
spec:
  hosts:
  - ext-svc.example.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  location: MESH_EXTERNAL
  resolution: DNS
```

您指定的外部资源使用 `hosts` 字段。可以使用完全限定名或通配符作为前缀域名。

您可以配置虚拟服务和目标规则，以更细粒度的方式控制到服务入口的流量，这与网格中的任何其他服务配置流量的方式相同。例如，下面的目标规则配置流量路由以使用双向 TLS 来保护到 `ext-svc.example.com` 外部服务的连接，我们使用服务入口配置了该外部服务：

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-httpbin
  namespace : egress
spec:
  hosts:
  - httpbin.com
  exportTo:
  - "."
  location: MESH_EXTERNAL
  ports:
  - number: 80
    name: http
    protocol: HTTP
  resolution: DNS
```

## ServiceEntry配置

| Field             | Type         | Description                                                  | Required |
| ----------------- | ------------ | ------------------------------------------------------------ | -------- |
| `hosts`           | `string[]`   | 绑定到 `ServiceEntry` 上的主机名。可以是一个带有通配符前缀的 DNS 名称。如果服务不是 HTTP 协议的，例如 `mongo`、TCP 以及 HTTPS 中，`hosts` 中的 DNS 名称会被忽略，这种情况下会使用 `endpoints` 中的 `address` 以及 `port` 来甄别调用目标。 | Yes      |
| `addresses`       | `string[]`   | 服务相关的虚拟 IP。可以是 CIDR 前缀。对 HTTP 服务来说，这一字段会被忽略，而会使用 HTTP 的 `HOST/Authority` Header。而对于非 HTTP 服务，例如 `mongo`、TCP 以及 HTTPS 中，这些主机会被忽略。如果指定了一个或者多个 IP 地址，对于在列表范围内的 IP 的访问会被判定为属于这一服务。如果地址字段为空，服务的鉴别就只能靠目标端口了。在这种情况下，被访问服务的端口一定不能和其他网格内的服务进行共享。换句话说，这里的 Sidecar 会简单的做为 TCP 代理，将特定端口的访问转发到指定目标端点的 IP、主机上去。就无法支持 Unix socket 了。 | No       |
| `ports`           | `Port[]`     | 和外部服务关联的端口。如果 `endpoints` 是 Unix socket 地址，这里必须只有一个端口。 | Yes      |
| `location`        | `Location`   | 用于指定该服务的位置，属于网格内部还是外部。                 | No       |
| `resolution`      | `Resolution` | 主机的服务发现模式。在没有附带 IP 地址的情况下，为 TCP 端口设置解析模式为 NONE 时必须小心。在这种情况下，对任何 IP 的指定端口的流量都是允许的（例如 `0.0.0.0:`）。 | Yes      |
| `endpoints`       | `Endpoint[]` | 一个或者多个关联到这一服务的 `endpoint`                      | No       |
| `exportTo`        | `string[]`   | 此服务导出到的名称空间列表。导出服务允许它被其他名称空间中定义的边车，网关和虚拟服务使用。此功能为服务所有者和网格管理员提供了一种机制，可以控制跨名称空间边界的服务的可见性。 如果未指定名称空间，则默认情况下会将服务导出到所有名称空间 | No       |
| `subjectAltNames` | `string[]`   | 实施此服务的工作负载实例允许使用的使用者备用名称列表。此信息用于强制执行安全命名。如果指定，则代理将验证服务器证书的使用者备用名称是否与指定值之一匹配。 | No       |

### ServiceEntry.Location

| Name            | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `MESH_EXTERNAL` | 表示服务在网格外部。通常用于指示通过API使用的外部服务。      |
| `MESH_INTERNAL` | 表示服务是网格的一部分。通常用于指示在扩展服务网格以包括不受管理的基础架构时显式添加的服务 |

具体细节的参数明细可查阅： https://preliminary.istio.io/zh/docs/reference/config/networking/service-entry 

## 参考文献

 https://preliminary.istio.io/zh/docs/concepts/traffic-management/#service-entries 

 https://preliminary.istio.io/zh/docs/reference/config/networking/service-entry 

 https://skyao.io/learning-istio/crd/network/serviceentry.html 