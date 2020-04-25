---
title: istio 1.6架构及性能
date: 2020-04-25
tags: istio
categories:	istio
---

##  Istio 架构

 Istio 服务网格从逻辑上分为数据平面和控制平面。

- **数据平面** 由一组智能代理（[Envoy](https://www.envoyproxy.io/)）组成，被部署为 sidecar。这些代理负责协调和控制微服务之间的所有网络通信。他们还收集和报告所有网格流量的遥测数据。
- **控制平面** 管理并配置代理来进行流量路由。



## Istio 核心组件

下图展示了组成每个平面的不同组件： 

![istio-arch](/images/istio-arch.svg)

 Istio 中的流量分为数据平面流量和控制平面流量。

- 数据平面流量是指工作负载的业务逻辑发送和接收的消息
- 控制平面流量是指在 Istio 组件之间发送的配置和控制消息用来编排网格的行为
-  Istio  中的流量管理特指数据平面流量

### Envoy

Istio 使用 [Envoy](https://envoyproxy.github.io/envoy/) 代理的扩展版本。Envoy 是用 C++ 开发的高性能代理，用于协调服务网格中所有服务的入站和出站流量。Envoy 代理是唯一与数据平面流量交互的 Istio 组件。

Envoy 代理被部署为服务的 sidecar，在逻辑上为服务增加了 Envoy 的许多内置特性，例如:

- 动态服务发现
- 负载均衡
- TLS 终端
- HTTP/2 与 gRPC 代理
- 熔断器
- 健康检查
- 基于百分比流量分割的分阶段发布
- 故障注入
- 丰富的指标

 这种 sidecar 部署允许 Istio 提取大量关于流量行为的信号作为属性。Istio 可以使用这些属性来实施策略决策，并将其发送到监视系统以提供有关整个网格行为的信息。 

由 Envoy 代理启用的一些 Istio 的功能和任务包括:

- 流量控制功能：通过丰富的 HTTP、gRPC、WebSocket 和 TCP 流量路由规则来执行细粒度的流量控制。
- 网络弹性特性：重试设置、故障转移、熔断器和故障注入。
- 安全性和身份验证特性：执行安全性策略以及通过配置 API 定义的访问控制和速率限制。
- 基于 WebAssembly 的可插拔扩展模型，允许通过自定义策略实施和生成网格流量的遥测。

### Pilot

`Pilot` 为 Envoy sidecar 提供服务发现、用于智能路由的流量管理功能（例如，A/B 测试、金丝雀发布等）以及弹性功能（超时、重试、熔断器等）。

`Pilot` 将控制流量行为的高级路由规则转换为特定于环境的配置，并在运行时将它们传播到 sidecar。Pilot 将特定于平台的服务发现机制抽象出来，并将它们合成为任何符合 [Envoy API](https://www.envoyproxy.io/docs/envoy/latest/api/api) 的 sidecar 都可以使用的标准格式。

下图展示了平台适配器和 Envoy 代理如何交互。

![pilot-discovery](/images/pilot-discovery.svg)

1. 平台启动一个服务的新实例，该实例通知其平台适配器。
2. 平台适配器使用 `Pilot` 抽象模型注册实例。
3. `Pilot` 将流量规则和配置派发给 `Envoy` 代理，来传达此次更改。

### Citadel

`Citadel`通过内置的身份和证书管理，可以支持强大的服务到服务以及最终用户的身份验证。您可以使用 Citadel 来升级服务网格中的未加密流量。使用 `Citadel` operator 可以执行基于服务身份的策略。

### Galley

`Galley` 是 Istio 的配置验证、提取、处理和分发组件。它负责将其余的 Istio 组件与从底层平台（例如 Kubernetes）获取用户配置的细节隔离开来。



## Istio 部署模型

您可以将单个网格配置为包括多集群。多集群部署可为您提供更大程度的隔离和可用性，但会增加复杂性。 如果您的系统具有高可用性要求，则可能需要集群跨多个可用区和地域。 对于应用变更或新的版本，您可以在一个集群中配置金丝雀发布，这有助于把对用户的影响降到最低。 此外，如果某个集群有问题，您可以暂时将流量路由到附近的集群，直到解决该问题为止。 

 ![istio-multi-cluster](/images/istio-multi-cluster.svg)



## Istio 1.6 性能总结

[Istio 负载测试](https://github.com/istio/tools/tree/master/perf/load) 网格包含了 **1000** 个服务和 **2000** 个 sidecar，全网格范围内，QPS 为 70,000。 在使用 Istio 1.6 运行测试后，我们得到了如下结果：

- 通过代理的 QPS 有 1000 时，Envoy 使用了 **0.5 vCPU** 和 **50 MB 内存**。
- 网格总的 QPS 为 1000 时，`istio-telemetry` 服务使用了 **0.6 vCPU**。
- Pilot 使用了 **1 vCPU** 和 **1.5 GB** 内存。
- **90%** 的情况 Envoy 代理只增加了 **2.8 ms** 的延迟。

## 参考文献

 https://preliminary.istio.io/docs/ops/deployment/architecture/ 

 https://preliminary.istio.io/zh/docs/ops/deployment/architecture/ 

 https://preliminary.istio.io/zh/docs/ops/deployment/deployment-models/

 https://preliminary.istio.io/docs/ops/deployment/performance-and-scalability/ 

