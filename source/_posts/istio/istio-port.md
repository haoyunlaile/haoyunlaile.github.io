---
title: Istio 使用的端口
date: 2020-04-25
tags: istio
categories:	istio
---

## Istio 常用端口

| 端口  | 协议 | 使用者                                                       | 描述                                  |
| ----- | ---- | :----------------------------------------------------------- | ------------------------------------- |
| 8060  | HTTP | Citadel                                                      | GRPC 服务器                           |
| 8080  | HTTP | Citadel agent                                                | SDS service 监控                      |
| 9090  | HTTP | Prometheus                                                   | Prometheus                            |
| 9091  | HTTP | Mixer                                                        | 策略/遥测                             |
| 9876  | HTTP | Citadel, Citadel agent                                       | ControlZ 用户界面                     |
| 9901  | GRPC | Galley                                                       | 网格配置协议                          |
| 15000 | TCP  | Envoy                                                        | Envoy 管理端口 (commands/diagnostics) |
| 15001 | TCP  | Envoy                                                        | Envoy 传出                            |
| 15006 | TCP  | Envoy                                                        | Envoy 传入                            |
| 15004 | HTTP | Mixer, Pilot                                                 | 策略/遥测 - `mTLS`                    |
| 15010 | HTTP | Pilot                                                        | Pilot service - XDS pilot - 发现      |
| 15011 | TCP  | Pilot                                                        | Pilot service - `mTLS` - Proxy - 发现 |
| 15014 | HTTP | Citadel, Citadel agent, Galley, Mixer, Pilot, Sidecar Injector | 控制平面监控                          |
| 15020 | HTTP | Ingress Gateway                                              | Pilot 健康检查                        |
| 15029 | HTTP | Kiali                                                        | Kiali 用户界面                        |
| 15030 | HTTP | Prometheus                                                   | Prometheus 用户界面                   |
| 15031 | HTTP | Grafana                                                      | Grafana 用户界面                      |
| 15032 | HTTP | Tracing                                                      | Tracing 用户界面                      |
| 15443 | TLS  | Ingress and Egress Gateways                                  | SNI                                   |
| 15090 | HTTP | Mixer                                                        | Proxy                                 |
| 42422 | TCP  | Mixer                                                        | 遥测 - Prometheus                     |

<!--more--> 

## 参考文献

 https://preliminary.istio.io/docs/ops/deployment/requirements/ 