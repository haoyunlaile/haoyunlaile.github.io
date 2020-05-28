---
title: Istio Gateway
date: 2020-04-27
tags: ["istio","istio-gateway"]
categories:	istio
---

## Istio  Ingress Gateway 概述

![istio-gateways](/images/istio-gateways.svg)

​                                                                    *Istio 服务网格中的网关* 

使用网关为网格来管理入站和出站流量，可以让用户指定要进入或离开网格的流量。

网关配置被用于运行在网格内独立 Envoy 代理中，而不是服务工作负载的应用 Sidecar 代理。

 [`Gateway`](https://preliminary.istio.io/zh/docs/reference/config/networking/gateway/) 用于为 HTTP / TCP 流量配置负载均衡器，并不管该负载均衡器将在哪里运行。网格中可以存在任意数量的 [`Gateway`](https://preliminary.istio.io/zh/docs/reference/config/networking/gateway/)，并且多个不同的 [`Gateway`](https://preliminary.istio.io/zh/docs/reference/config/networking/gateway/) 实现可以共存。实际上，通过在配置中指定一组工作负载（Pod）标签，可以将 Gateway 配置绑定到特定的工作负载，从而允许用户通过编写简单的 Gateway Controller 来重用现成的网络设备。

`Gateway` 只用于配置 L4-L6 功能（例如，对外公开的端口，TLS 配置），所有主流的 L7 代理均以统一的方式实现了这些功能。然后，通过在 `Gateway` 上绑定 `VirtualService` 的方式，可以使用标准的 Istio 规则来控制进入 `Gateway` 的 HTTP 和 TCP 流量。 

<!--more--> 

例如，下面这个简单的 `Gateway` 配置了一个 Load Balancer，以允许访问 host `bookinfo.com` 的 https 外部流量进入网格中： 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    app: my-ingress-gateway
  servers:
  - port:
      number: 443
      name: https
      protocol: HTTPS
    hosts:
    - bookinfo.com
    tls:
      mode: SIMPLE
      serverCertificate: /tmp/tls.crt
      privateKey: /tmp/tls.key
```

 要为进入上面的 Gateway 的流量配置相应的路由，必须为同一个 host 定义一个 `VirtualService`（在下一节中描述），并使用配置中的 `gateways` 字段绑定到前面定义的 `Gateway` 上： 

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
    - bookinfo.com
  gateways:
  - bookinfo-gateway # <---- bind to gateway
    http:
  - match:
    - uri:
        prefix: /reviews
    route:
    ...
```

然后就可以为出口流量配置带有路由规则的虚拟服务。 

###  `Gateway` 配置信息 

| Field    | Type     | Description                                  | Required |
| -------- | -------- | -------------------------------------------- | -------- |
| servers  | Server[] | 开放的服务列表                               | 是       |
| selector | map      | 通过这个Label来找到执行 Gateway 规则的 Envoy | 是       |

### `Server` 配置信息 

| Field             | Type         | Description                                                  | Required |
| ----------------- | ------------ | ------------------------------------------------------------ | -------- |
| `port`            | `Port`       | 服务对外监听的端口                                           | 是       |
| `hosts`           | `string[]`   | Gateway 发布的服务地址，是一个 FQDN 域名，可以支持左侧通配符来进行模糊查询 | 是       |
| `tls`             | `TLSOptions` | TLS安全配置                                                  | 否       |
| `defaultEndpoint` | `string`     | 默认情况下，应将流量转发到的环回IP端点或Unix域套接字         | 否       |

### `Port` 配置信息 

| Field      | Type     | Description                                                  | Required |
| ---------- | -------- | ------------------------------------------------------------ | -------- |
| `number`   | `uint32` | 一个有效的端口号                                             | 是       |
| `protocol` | `string` | 所使用的协议，支持HTTP\|HTTPS\|GRPC\|HTTP2\|MONGO\|TCP\|TLS. | 是       |
| `name`     | `string` | 给端口分配一个名称                                           | 否       |

### `Server.TLSOptions` 配置信息

| Field                   | Type          | Description                                                  | Required |
| ----------------------- | ------------- | ------------------------------------------------------------ | -------- |
| `httpsRedirect`         | `bool`        | 是否要做 HTTP 重定向                                         | 否       |
| `mode`                  | `TLSmode`     | 在配置的外部端口上使用 TLS 服务时，可以取 PASSTHROUGH、SIMPLE、MUTUAL、AUTO_PASSTHROUGH 这 4 种模式 | 否       |
| `serverCertificate`     | `string`      | 服务端证书的路径。当模式是 SIMPLE 和 MUTUAL 时必须指定       | 否       |
| `privateKey`            | `string`      | 服务端密钥的路径。当模式是 SIMPLE 和 MUTUAL 时必须指定       | 否       |
| `caCertificates`        | `string`      | CA 证书路径。当模式是 MUTUAL 时指定                          | 否       |
| `credentialName`        | `string`      | 用于唯一标识服务端证书和秘钥。Gateway 使用 credentialName 从远端的凭证存储中获取证书和秘钥，而不是使用 Mount 的文件 | 否       |
| `subjectAltNames`       | `string[]`    | SAN 列表，SubjectAltName 允许一个证书指定多个域名            | 否       |
| `verifyCertificateSpki` | `string[]`    | 授权客户端证书的SKPI的base64编码的SHA-256哈希值的可选列表    | 否       |
| `verifyCertificateHash` | `string[]`    | 授权客户端证书的十六进制编码SHA-256哈希值的可选列表          | 否       |
| `minProtocolVersion`    | `TLSProtocol` | TLS 协议的最小版本                                           | 否       |
| `maxProtocolVersion`    | `TLSProtocol` | TLS 协议的最大版本                                           | 否       |
| `cipherSuites`          | `string[]`    | 指定的加密套件，默认使用 Envoy 支持的加密套件                | 否       |

### `Server.TLSOptions.TLSmode` 配置信息

| Name               | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `PASSTHROUGH`      | 客户端提供的SNI字符串将用作VirtualService TLS路由中的匹配条件，以根据服务注册表确定目标服务 |
| `SIMPLE`           | 使用标准TLS语义的安全连接                                    |
| `MUTUAL`           | 通过提供服务器证书进行身份验证，使用双边TLS来保护与下游的连接 |
| `AUTO_PASSTHROUGH` | 与直通模式相似，不同之处在于具有此TLS模式的服务器不需要关联的VirtualService即可从SNI值映射到注册表中的服务。目标详细信息（例如服务/子集/端口）被编码在SNI值中。代理将转发到SNI值指定的上游（Envoy）群集（一组端点）。 |
| `ISTIO_MUTUAL`     | 通过提供用于身份验证的服务器证书，使用相互TLS使用来自下游的安全连接 |

### `Server.TLSOptions.TLSProtocol` 配置信息

| Name       | Description     |
| ---------- | --------------- |
| `TLS_AUTO` | 自动选择DLS版本 |
| `TLSV1_0`  | TLS 1.0         |
| `TLSV1_1`  | TLS 1.1         |
| `TLSV1_2`  | TLS 1.2         |
| `TLSV1_3`  | TLS 1.3         |

## 参考文献

 https://preliminary.istio.io/zh/docs/concepts/traffic-management/#gateways 

 https://preliminary.istio.io/zh//blog/2018/v1alpha3-routing/

 https://preliminary.istio.io/zh/docs/reference/config/networking/gateway/#Gateway 