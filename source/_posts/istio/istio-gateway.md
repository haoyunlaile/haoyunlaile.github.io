---
title: istio gateway
date: 2020-04-27
tags: ["istio","istio gateway"]
categories:	istio
---

## Istio  Ingress gateway 网关

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

| ield     | Type     | Description                                                  | Required |
| -------- | -------- | ------------------------------------------------------------ | -------- |
| servers  | Server[] | A list of server specifications.                             | Yes      |
| selector | map      | One or more labels that indicate a specific set of pods/VMs on which this gateway configuration should be applied. The scope of label search is restricted to the configuration namespace in which the the resource is present. In other words, the Gateway resource must reside in the same namespace as the gateway workload instance. | Yes      |

### `Server` 配置信息 

| Field             | Type         | Description                                                  | Required |
| ----------------- | ------------ | ------------------------------------------------------------ | -------- |
| `port`            | `Port`       | The Port on which the proxy should listen for incoming connections. | Yes      |
| `hosts`           | `string[]`   | One or more hosts exposed by this gateway. While typically applicable to HTTP services, it can also be used for TCP services using TLS with SNI. A host is specified as a `dnsName` with an optional `namespace/` prefix. The `dnsName` should be specified using FQDN format, optionally including a wildcard character in the left-most component (e.g., `prod/*.example.com`). Set the `dnsName` to `*` to select all `VirtualService` hosts from the specified namespace (e.g.,`prod/*`).The `namespace` can be set to `*` or `.`, representing any or the current namespace, respectively. For example, `*/foo.example.com` selects the service from any available namespace while `./foo.example.com` only selects the service from the namespace of the sidecar. The default, if no `namespace/` is specified, is `*/`, that is, select services from any namespace. Any associated `DestinationRule` in the selected namespace will also be used.A `VirtualService` must be bound to the gateway and must have one or more hosts that match the hosts specified in a server. The match could be an exact match or a suffix match with the server’s hosts. For example, if the server’s hosts specifies `*.example.com`, a `VirtualService` with hosts `dev.example.com` or `prod.example.com` will match. However, a `VirtualService` with host `example.com` or `newexample.com` will not match.NOTE: Only virtual services exported to the gateway’s namespace (e.g., `exportTo` value of `*`) can be referenced. Private configurations (e.g., `exportTo` set to `.`) will not be available. Refer to the `exportTo` setting in `VirtualService`, `DestinationRule`, and `ServiceEntry` configurations for details. | Yes      |
| `tls`             | `TLSOptions` | Set of TLS related options that govern the server’s behavior. Use these options to control if all http requests should be redirected to https, and the TLS modes to use. | No       |
| `defaultEndpoint` | `string`     | The loopback IP endpoint or Unix domain socket to which traffic should be forwarded to by default. Format should be `127.0.0.1:PORT` or `unix:///path/to/socket` or `unix://@foobar` (Linux abstract namespace). | No       |

### `Port` 配置信息 

| Field      | Type     | Description                                                  | Required |
| ---------- | -------- | ------------------------------------------------------------ | -------- |
| `number`   | `uint32` | A valid non-negative integer port number.                    | Yes      |
| `protocol` | `string` | The protocol exposed on the port. MUST BE one of HTTP\|HTTPS\|GRPC\|HTTP2\|MONGO\|TCP\|TLS. TLS implies the connection will be routed based on the SNI header to the destination without terminating the TLS connection. | Yes      |
| `name`     | `string` | Label assigned to the port.                                  | No       |

### `Server.TLSOptions` 配置信息

| Field                   | Type          | Description                                                  | Required |
| ----------------------- | ------------- | ------------------------------------------------------------ | -------- |
| `httpsRedirect`         | `bool`        | If set to true, the load balancer will send a 301 redirect for all http connections, asking the clients to use HTTPS. | No       |
| `mode`                  | `TLSmode`     | Optional: Indicates whether connections to this port should be secured using TLS. The value of this field determines how TLS is enforced. | No       |
| `serverCertificate`     | `string`      | REQUIRED if mode is `SIMPLE` or `MUTUAL`. The path to the file holding the server-side TLS certificate to use. | No       |
| `privateKey`            | `string`      | REQUIRED if mode is `SIMPLE` or `MUTUAL`. The path to the file holding the server’s private key. | No       |
| `caCertificates`        | `string`      | REQUIRED if mode is `MUTUAL`. The path to a file containing certificate authority certificates to use in verifying a presented client side certificate. | No       |
| `credentialName`        | `string`      | The credentialName stands for a unique identifier that can be used to identify the serverCertificate and the privateKey. The credentialName appended with suffix “-cacert” is used to identify the CaCertificates associated with this server. Gateway workloads capable of fetching credentials from a remote credential store such as Kubernetes secrets, will be configured to retrieve the serverCertificate and the privateKey using credentialName, instead of using the file system paths specified above. If using mutual TLS, gateway workload instances will retrieve the CaCertificates using credentialName-cacert. The semantics of the name are platform dependent. In Kubernetes, the default Istio supplied credential server expects the credentialName to match the name of the Kubernetes secret that holds the server certificate, the private key, and the CA certificate (if using mutual TLS). Set the `ISTIO_META_USER_SDS` metadata variable in the gateway’s proxy to enable the dynamic credential fetching feature. | No       |
| `subjectAltNames`       | `string[]`    | A list of alternate names to verify the subject identity in the certificate presented by the client. | No       |
| `verifyCertificateSpki` | `string[]`    | An optional list of base64-encoded SHA-256 hashes of the SKPIs of authorized client certificates. Note: When both verify*certificate*hash and verify*certificate*spki are specified, a hash matching either value will result in the certificate being accepted. | No       |
| `verifyCertificateHash` | `string[]`    | An optional list of hex-encoded SHA-256 hashes of the authorized client certificates. Both simple and colon separated formats are acceptable. Note: When both verify*certificate*hash and verify*certificate*spki are specified, a hash matching either value will result in the certificate being accepted. | No       |
| `minProtocolVersion`    | `TLSProtocol` | Optional: Minimum TLS protocol version.                      | No       |
| `maxProtocolVersion`    | `TLSProtocol` | Optional: Maximum TLS protocol version.                      | No       |
| `cipherSuites`          | `string[]`    | Optional: If specified, only support the specified cipher list. Otherwise default to the default cipher list supported by Envoy. | No       |

### `Server.TLSOptions.TLSmode` 配置信息

| Name               | Description                                                  |
| ------------------ | ------------------------------------------------------------ |
| `PASSTHROUGH`      | The SNI string presented by the client will be used as the match criterion in a VirtualService TLS route to determine the destination service from the service registry. |
| `SIMPLE`           | Secure connections with standard TLS semantics.              |
| `MUTUAL`           | Secure connections to the downstream using mutual TLS by presenting server certificates for authentication. |
| `AUTO_PASSTHROUGH` | Similar to the passthrough mode, except servers with this TLS mode do not require an associated VirtualService to map from the SNI value to service in the registry. The destination details such as the service/subset/port are encoded in the SNI value. The proxy will forward to the upstream (Envoy) cluster (a group of endpoints) specified by the SNI value. This server is typically used to provide connectivity between services in disparate L3 networks that otherwise do not have direct connectivity between their respective endpoints. Use of this mode assumes that both the source and the destination are using Istio mTLS to secure traffic. |
| `ISTIO_MUTUAL`     | Secure connections from the downstream using mutual TLS by presenting server certificates for authentication. Compared to Mutual mode, this mode uses certificates, representing gateway workload identity, generated automatically by Istio for mTLS authentication. When this mode is used, all other fields in `TLSOptions` should be empty. |

### `Server.TLSOptions.TLSProtocol` 配置信息

| Name       | Description                                   |
| ---------- | --------------------------------------------- |
| `TLS_AUTO` | Automatically choose the optimal TLS version. |
| `TLSV1_0`  | TLS version 1.0                               |
| `TLSV1_1`  | TLS version 1.1                               |
| `TLSV1_2`  | TLS version 1.2                               |
| `TLSV1_3`  | TLS version 1.3                               |

## 参考文献

 https://preliminary.istio.io/zh/docs/concepts/traffic-management/#gateways 

 https://preliminary.istio.io/zh//blog/2018/v1alpha3-routing/

 https://preliminary.istio.io/zh/docs/reference/config/networking/gateway/#Gateway 