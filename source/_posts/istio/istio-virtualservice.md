---
title: Istio VirtualService
date: 2020-04-28 13:00
tags: ["istio","istio-virtualservice"]
categories:	istio
---

## Istio  VirtualService 虚拟服务

###  概念及示例

`VirtualService` 描述了一个或多个用户可寻址目标到网格内实际工作负载之间的映射 。 虚拟服务让您配置如何在服务网格内将请求路由到服务，这基于 Istio 和平台提供的基本的连通性和服务发现能力。每个虚拟服务包含一组路由规则，Istio 按顺序评估它们，Istio 将每个给定的请求匹配到虚拟服务指定的实际目标地址。您的网格可以有多个虚拟服务，也可以没有，取决于您的使用场景。 

虚拟服务在增强 Istio 流量管理的灵活性和有效性方面，发挥着至关重要的作用，通过对客户端请求的目标地址与真实响应请求的目标工作负载进行解耦来实现。虚拟服务同时提供了丰富的方式，为发送至这些工作负载的流量指定不同的路由规则。 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  gateways:
  - bookinfo-gateway
  - mesh
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

<!-- more -->

### gateways字段

通过将 `VirtualService` 绑定到同一 Host 的 `Gateway` 配置（如前一节所述 ），可向网格外部暴露这些 Host。

```
gateways:
  - bookinfo-gateway
  - mesh
```

网格内部通信存在一个默认的`mesh` 保留字， `mesh` 用来指代网格中的所有 Sidecar。当这一字段被省略时，就会使用缺省值（`mesh`），也就是针对网格中的所有 Sidecar 生效。如果提供了 `gateways` 字段，这一规则就只会应用到声明的 `Gateway` 之中。要让规则同时对 `Gateway` 和网格内服务生效，需要显式的将 `mesh` 加入 `gateways` 列表。 

###  Hosts字段

`VirtualService` 中  `hosts` 字段列举虚拟服务的目标主机 ——即用户指定的目标或是路由规则设定的目标。这是客户端向服务发送请求时使用的一个或多个地址 。  

```yaml
hosts:
    - bookinfo.com
```

虚拟服务目的地可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或显式地指向一个完全限定域名（FQDN）。您也可以使用通配符（“*”）前缀，让您创建一组匹配所有服务的路由规则。虚拟服务的 `hosts` 字段实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址。这让您可以为没有路由到网格内部的虚拟主机建模。 

### 路由规则

在 `http` 字段包含了虚拟服务的路由规则，用来描述匹配条件和路由行为，它们把 HTTP/1.1、HTTP2 和 gRPC 等流量发送到 hosts 字段指定的目标（您也可以用 `tcp` 和 `tls` 片段流量设置路由规则）。一个路由规则包含了指定的请求要流向哪个目标地址，具有 0 或多个匹配条件，取决于您的使用场景。 

#### 匹配条件

示例中的第一个路由规则有一个条件，因此以 `match` 字段开始。在本例中，您希望此路由应用于来自 ”jason“ 用户的所有请求，所以使用 `headers`、`end-user` 和 `exact` 字段选择适当的请求。 

```yaml
- match:
   - headers:
       end-user:
         exact: jason
```

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings-route
spec:
  hosts:
  - ratings.prod.svc.cluster.local
  http:
  - match:
    - headers:
        end-user:
          exact: jason
      uri:
        prefix: "/ratings/v2/"
      ignoreUriCase: true
    route:
    - destination:
        host: ratings.prod.svc.cluster.local
```

#### Destination

route 部分的 `destination` 字段指定了符合此条件的流量的实际目标地址。与虚拟服务的 `hosts` 不同，destination 的 host 必须是存在于 Istio 服务注册中心的实际目标地址，否则 Envoy 不知道该将请求发送到哪里。可以是一个有代理的服务网格，或者是一个通过服务入口被添加进来的非网格服务。本示例运行在 Kubernetes 环境中，host 名为一个 Kubernetes 中运行着`Service` 的名称：

```yaml
route:
- destination:
    host: reviews
    subset: v2
```

请注意，在该示例和本页其它示例中，为了简单，我们使用 Kubernetes `Service`的短名称设置 destination 的 host。在评估此规则时，Istio 会添加一个基于虚拟服务命名空间的域后缀，这个虚拟服务包含要获取主机的完全限定名的路由规则。在我们的示例中使用短名称也意味着您可以复制并在任何喜欢的命名空间中尝试它们。

>  只有在目标主机和虚拟服务位于相同的 Kubernetes 命名空间时才可以使用这样的短名称 , 建议您在生产环境中指定完全限定的主机名。  

destination 片段还指定了 Kubernetes 服务的子集，将符合此规则条件的请求转入其中。在本例中子集名称是 v2。您可以在DestinationRule章节中看到如何定义服务子集。

### Subset字段

`subset` 不属于 Istio 创建的 CRD，但是它是一条重要的配置信息，有必要单独说明下。`subset` 是服务端点的集合，可以用于 A/B 测试或者分版本路由等场景。另外在 `subset` 中可以覆盖服务级别的即 `VirtualService` 中的定义的流量策略。

以下是`subset` 的配置信息。对于 Kubernetes 中的服务，一个 `subset` 相当于使用 label 的匹配条件选出来的 `Service`

| Field           | Type            | Description                                                  | Required |
| --------------- | --------------- | ------------------------------------------------------------ | -------- |
| `name`          | `string`        | 服务名和 `subset` 名称可以用于路由规则中的流量拆分           | Yes      |
| `labels`        | `map`           | 使用标签对服务注册表中的服务端点进行筛选                     | No       |
| `trafficPolicy` | `TrafficPolicy` | 应用到这一 `subset` 的流量策略。缺省情况下 `subset` 会继承 `DestinationRule` 级别的策略，这一字段的定义则会覆盖缺省的继承策略 | No       |

#### 路由规则优先级

**路由规则**按从上到下的顺序选择，虚拟服务中定义的第一条规则有最高优先级。本示例中，不满足第一个路由规则的流量均流向一个默认的目标，该目标在第二条规则中指定。因此，第二条规则没有 match 条件，直接将流量导向 v3 子集。 我们建议提供一个默认的“无条件”或基于权重的规则作为每一个虚拟服务的最后一条规则，从而确保流经虚拟服务的流量至少能够匹配一条路由规则。 

## VirtualService配置

| Field      | Type          | Description                                                  | Required |
| ---------- | ------------- | ------------------------------------------------------------ | -------- |
| `hosts`    | `string[]`    | 流量的目标主机。可以是带有通配符前缀的 DNS 名称，也可以是 IP 地址。根据所在平台情况，还可能使用短名称来代替 FQDN。这种场景下，短名称到 FQDN 的具体转换过程是要靠下层平台完成的。**一个主机名只能在一个 VirtualService 中定义。**同一个 `VirtualService` 中可以用于控制多个 HTTP 和 TCP 端口的流量属性。Kubernetes 用户注意：当使用服务的短名称时（例如使用 `reviews`，而不是 `reviews.default.svc.cluster.local`），Istio 会根据规则所在的命名空间来处理这一名称，而非服务所在的命名空间。假设 “default” 命名空间的一条规则中包含了一个 `reviews` 的 `host` 引用，就会被视为 `reviews.default.svc.cluster.local`，而不会考虑 `reviews` 服务所在的命名空间。**为了避免可能的错误配置，建议使用 FQDN 来进行服务引用。** `hosts` 字段对 HTTP 和 TCP 服务都是有效的。网格中的服务也就是在服务注册表中注册的服务，必须使用他们的注册名进行引用；只有 `Gateway` 定义的服务才可以使用 IP 地址。 | Yes      |
| `gateways` | `string[]`    | `Gateway` 名称列表，Sidecar 会据此使用路由。`VirtualService` 对象可以用于网格中的 Sidecar，也可以用于一个或多个 `Gateway`。这里公开的选择条件可以在协议相关的路由过滤条件中进行覆盖。保留字 `mesh` 用来指代网格中的所有 Sidecar。当这一字段被省略时，就会使用缺省值（`mesh`），也就是针对网格中的所有 Sidecar 生效。如果提供了 `gateways` 字段，这一规则就只会应用到声明的 `Gateway` 之中。要让规则同时对 `Gateway` 和网格内服务生效，需要显式的将 `mesh` 加入 `gateways` 列表。 | No       |
| `http`     | `HTTPRoute[]` | HTTP 流量规则的有序列表。这个列表对名称前缀为 `http-`、`http2-`、`grpc-` 的服务端口，或者协议为 `HTTP`、`HTTP2`、`GRPC` 以及终结的 TLS，另外还有使用 `HTTP`、`HTTP2` 以及 `GRPC` 协议的 `ServiceEntry` 都是有效的。进入流量会使用匹配到的第一条规则。 | No       |
| `tls`      | `TLSRoute[]`  | 一个有序列表，对应的是透传 TLS 和 HTTPS 流量。路由过程通常利用 `ClientHello` 消息中的 SNI 来完成。TLS 路由通常应用在 `https-`、`tls-` 前缀的平台服务端口，或者经 `Gateway` 透传的 HTTPS、TLS 协议端口，以及使用 HTTPS 或者 TLS 协议的 `ServiceEntry` 端口上。**注意：没有关联 VirtualService 的 https- 或者 tls- 端口流量会被视为透传 TCP 流量。** | No       |
| `tcp`      | `TCPRoute[]`  | 一个针对透传 TCP 流量的有序路由列表。TCP 路由对所有 HTTP 和 TLS 之外的端口生效。进入流量会使用匹配到的第一条规则。 | No       |
| `exportTo` | `string[]`    | 当前vritual service要导出的 namespace 列表。 应用于 vritual service 的解析发生在 namespace 层次结构的上下文中。 vritual service 的导出允许将其包含在其他 namespace 中的服务的解析层次结构中。 此功能为服务所有者和网格管理员提供了一种机制，用于控制跨 namespace 边界的 vritual service 的可见性<br/>如果未指定任何 namespace，则默认情况下将 vritual service rule 导出到所有 namespace<br/>值`.` 被保留，用于定义导出到 vritual service 被声明所在的相同 namespace 。类似的值`*`保留，用于定义导出到所有 namespaces<br/>NOTE：在当前版本中，exportTo值被限制为`.`或`*`（即， 当前namespace或所有namespace） |          |

### HTTPRoute配置

| Field           | Type                     | Description                                                  | Required |
| --------------- | ------------------------ | ------------------------------------------------------------ | -------- |
| `name`          | `string`                 | The name assigned to the route for debugging purposes. The route’s name will be concatenated with the match’s name and will be logged in the access logs for requests matching this route/match. | No       |
| `match`         | `HTTPMatchRequest[]`     | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. | No       |
| `route`         | `HTTPRouteDestination[]` | A http rule can either redirect or forward (default) traffic. The forwarding target can be one of several versions of a service (see glossary in beginning of document). Weights associated with the service version determine the proportion of traffic it receives. | No       |
| `redirect`      | `HTTPRedirect`           | A http rule can either redirect or forward (default) traffic. If traffic passthrough option is specified in the rule, route/redirect will be ignored. The redirect primitive can be used to send a HTTP 301 redirect to a different URI or Authority. | No       |
| `rewrite`       | `HTTPRewrite`            | Rewrite HTTP URIs and Authority headers. Rewrite cannot be used with Redirect primitive. Rewrite will be performed before forwarding. | No       |
| `timeout`       | `Duration`               | Timeout for HTTP requests.                                   | No       |
| `retries`       | `HTTPRetry`              | Retry policy for HTTP requests.                              | No       |
| `fault`         | `HTTPFaultInjection`     | Fault injection policy to apply on HTTP traffic at the client side. Note that timeouts or retries will not be enabled when faults are enabled on the client side. | No       |
| `mirror`        | `Destination`            | Mirror HTTP traffic to a another destination in addition to forwarding the requests to the intended destination. Mirrored traffic is on a best effort basis where the sidecar/gateway will not wait for the mirrored cluster to respond before returning the response from the original destination. Statistics will be generated for the mirrored destination. | No       |
| `mirrorPercent` | `UInt32Value`            | Percentage of the traffic to be mirrored by the `mirror` field. If this field is absent, all the traffic (100%) will be mirrored. Max value is 100. | No       |
| `corsPolicy`    | `CorsPolicy`             | Cross-Origin Resource Sharing policy (CORS). Refer to [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) for further details about cross origin resource sharing. | No       |
| `headers`       | `Headers`                | Header manipulation rules                                    | No       |

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  http:
  - match:
    - uri:
        prefix: /reviews
    route:
    - destination:
        host: reviews
  - match:
    - uri:
        prefix: /ratings
    route:
    - destination:
        host: ratings
        ...
```

### TCPRoute配置

| Type    | Description           | Required                                                     |      |
| ------- | --------------------- | ------------------------------------------------------------ | ---- |
| `match` | `L4MatchAttributes[]` | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. | No   |
| `route` | `RouteDestination[]`  | The destination to which the connection should be forwarded to. | No   |

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-Mongo
spec:
  hosts:
  - mongo.prod.svc.cluster.local
  tcp:
  - match:
    - port: 27017
    route:
    - destination:
        host: mongo.backup.svc.cluster.local
        port:
          number: 5555
```



### TLSRoute配置

| Type    | Description            | Required                                                     |      |
| ------- | ---------------------- | ------------------------------------------------------------ | ---- |
| `match` | `TLSMatchAttributes[]` | Match conditions to be satisfied for the rule to be activated. All conditions inside a single match block have AND semantics, while the list of match blocks have OR semantics. The rule is matched if any one of the match blocks succeed. | Yes  |
| `route` | `RouteDestination[]`   | The destination to which the connection should be forwarded to. | No   |

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo-sni
spec:
  hosts:
  - "*.bookinfo.com"
  gateways:
  - mygateway
  tls:
  - match:
    - port: 443
      sniHosts:
      - login.bookinfo.com
    route:
    - destination:
        host: login.prod.svc.cluster.local
  - match:
    - port: 443
      sniHosts:
      - reviews.bookinfo.com
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
```

具体细节的参数明细可查阅：https://preliminary.istio.io/zh/docs/reference/config/networking/virtual-service/#VirtualService

## 参考文献

 https://preliminary.istio.io/zh/docs/concepts/traffic-management/#virtual-services 

 https://preliminary.istio.io/zh//blog/2018/v1alpha3-routing/

 https://preliminary.istio.io/zh/docs/reference/config/networking/virtual-service/#VirtualService

 https://jimmysong.io/istio-handbook/concepts/traffic-management-basic.html 