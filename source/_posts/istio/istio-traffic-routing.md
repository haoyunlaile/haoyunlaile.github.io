---
title: Istio 流量劫持原理
date: 2020-05-29
tags: istio
categories:	istio
---

## 开篇

Istio 流量劫持的文章其实目前可以在servicemesher社区找到一篇非常详细的文章，可查阅：[Istio 中的 Sidecar 注入及透明流量劫持过程详解](https://www.servicemesher.com/blog/sidecar-injection-iptables-and-traffic-routing/)。特别是博主整理的那张“流量劫持示意图”，已经可以很清晰的看出来劫持流程。这里我借着那张图片解释一版该图片的文字版本。在开始文字版前如果对`iptables`命令如果不是非常了解的话建议先重点看下下面的两篇文章，深入浅出的解释了该命令的概念及用法：

1. [iptables概念]( http://www.zsythink.net/archives/1199 )  -  以通俗易懂的方式描述iptables的相关概念 
2. [iptables指南]( https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html )  -  iptables命令用法指南

这里引用iptables的一张报文流向图（版权归原博主所有）

![iptables-routing](/images/iptables-routing.png)

> 当客户端访问服务器的web服务时，客户端发送报文到网卡，而tcp/ip协议栈是属于内核的一部分，所以，客户端的信息会通过内核的TCP协议传输到用户空间中的web服务中，而此时，客户端报文的目标终点为web服务所监听的套接字（IP：Port）上，当web服务需要响应客户端请求时，web服务发出的响应报文的目标终点则为客户端，这个时候，web服务所监听的IP与端口反而变成了原点。 -- 引用自 zsythink

上面这部分描述相当重要，它是理解sidecar在进行流量劫持的基础之一。

下面我们分析下昨天`istio-init`启动时执行的`istio-iptables`命令

```shell
nsenter -t 8533 -n iptables -t nat -S

# 默认
# 为PREROUTING/INPUT/OUTPUT/POSTROUTING链设置策略为接收数据包(ACCEPT)
-P PREROUTING ACCEPT
-P INPUT ACCEPT
-P OUTPUT ACCEPT
-P POSTROUTING ACCEPT

# 自定义4个istio的规则链
-N ISTIO_INBOUND
-N ISTIO_IN_REDIRECT
-N ISTIO_OUTPUT
-N ISTIO_REDIRECT

# 进入PREROUTING链tcp协议请求全部定向至 ISTIO_INBOUND 自定义链进行规则匹配
-A PREROUTING -p tcp -j ISTIO_INBOUND
# 进入OUTPUT链tcp协议请求全部定向至 ISTIO_OUTPUT 自定义链进行规则匹配
-A OUTPUT -p tcp -j ISTIO_OUTPUT

# 入口
# tcp协议请求且请求端口为22/15090/15021/15020的请求停止执行当前链中的后续Rules，并执行下一个链
-A ISTIO_INBOUND -p tcp -m tcp --dport 22 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15090 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15021 -j RETURN
-A ISTIO_INBOUND -p tcp -m tcp --dport 15020 -j RETURN
# tcp协议且端口不是22/15090/15021/15020的请求全部定向至 ISTIO_IN_REDIRECT
-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT
# 将重定向于此的tcp协议请求流量全部重定向至15006端口(envoy入口流量端口)
-A ISTIO_IN_REDIRECT -p tcp -j REDIRECT --to-ports 15006

# 出口
#  源IP地址为localhost且数据包出口为 ”lo“ 的流量都返回到它的调用点中的下一条链执行(1)
-A ISTIO_OUTPUT -s 127.0.0.6/32 -o lo -j RETURN
#  目的地非localhost，数据包出口为 ”lo“，是istio-proxy用户的流量转发至 ISTIO_REDIRECT (2)
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --uid-owner 1337 -j ISTIO_IN_REDIRECT
#  数据包出口为 ”lo“，非istio-proxy用户的流量都返回到它的调用点中的下一条链执行(1)
-A ISTIO_OUTPUT -o lo -m owner ! --uid-owner 1337 -j RETURN
#  istio-proxy 用户的流量都返回到它的调用点中的下一条链执行
-A ISTIO_OUTPUT -m owner --uid-owner 1337 -j RETURN
#  目的地非localhost，数据包出口为 ”lo“，是istio-proxy用户组的流量转发至 ISTIO_REDIRECT(2)
-A ISTIO_OUTPUT ! -d 127.0.0.1/32 -o lo -m owner --gid-owner 1337 -j ISTIO_IN_REDIRECT
#  数据包出口为 ”lo“ 且非istio-proxy用户组流量都返回到它的调用点中的下一条链执行(1)
-A ISTIO_OUTPUT -o lo -m owner ! --gid-owner 1337 -j RETURN
#  到 istio-proxy 用户组的流量都返回到它的调用点中的下一条链执行(1)
-A ISTIO_OUTPUT -m owner --gid-owner 1337 -j RETURN
#  所有目的地为localhost的流量都返回到它的调用点中的下一条链执行(1)
-A ISTIO_OUTPUT -d 127.0.0.1/32 -j RETURN
#  其他不满足上述rules的流量全部转发到 ISTIO_REDIRECT  (2)
-A ISTIO_OUTPUT -j ISTIO_REDIRECT
#  将重定向于此的tcp协议请求流量全部重定向至15001端口(envoy出口流量端口)
-A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
```

> -m = --match. istio-proxy 用户身份运行， uid-owner 1337 为用户ID / gid-owner 1337 为用户组，即 sidecar 所处的用户空间，这也是 istio- proxy 容器默认使用的用户。

注意文中打括号的地方

(1) 代表流量会**直接执行下一个拦截链**，本文中下一个拦截链为`POSTROUTING`链

(2) 代表流量会**被重定向至envoy出口流量端口** 

根据上面的规则小结一下：

> ISTIO_INBOUND 链：所有进入Pod但非指定端口(如22)的流量全部重定向至15006端口(envoy入口流量端口)进行拦截处理。
>
> ISTIO_OUTPUT 链：所有流出Pod由 istio-proxy 用户空间发出且目的地不为localhost的流量全部重定向至15001端口（envoy出口流量端口），其他流量全部直接放行至下一个POSTROUTING链，不用被envoy拦截处理。

其实仔细思考下可以看到，流量拦截主要发生在两个地方：

1. 用户请求到达Pod时对应流量会被拦截至sidecar进行处理，由sidecar请求业务服务
2. 当业务服务响应用户请求时该响应再次被拦截至sidecar，由sidecar响应用户

再看下iptables nat 表的规则

```shell
nsenter -t 8533 -n iptables -t nat -L -v


Chain PREROUTING (policy ACCEPT 3435 packets, 206K bytes)
 pkts bytes target     prot opt in     out     source               destination         
 3435  206K ISTIO_INBOUND  tcp  --  any    any     anywhere             anywhere      (1)       

Chain INPUT (policy ACCEPT 3435 packets, 206K bytes)                                  (5)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 599 packets, 54757 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   22  1320 ISTIO_OUTPUT  tcp  --  any    any     anywhere             anywhere            

Chain POSTROUTING (policy ACCEPT 599 packets, 54757 bytes)                            (8)
 pkts bytes target     prot opt in     out     source               destination         

Chain ISTIO_INBOUND (1 references)                                                    (2) 
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:22
    1    60 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15090
 3434  206K RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15021
    0     0 RETURN     tcp  --  any    any     anywhere             anywhere             tcp dpt:15020
    0     0 ISTIO_IN_REDIRECT  tcp  --  any    any     anywhere             anywhere  (3)        

Chain ISTIO_IN_REDIRECT (3 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15006                                                                     (4)

Chain ISTIO_OUTPUT (1 references)                                                     (6)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 RETURN     all  --  any    lo      127.0.0.6            anywhere            
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner UID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner UID match 1337
   22  1320 RETURN     all  --  any    any     anywhere             anywhere             owner UID match 1337
    0     0 ISTIO_IN_REDIRECT  all  --  any    lo      anywhere            !localhost            owner GID match 1337
    0     0 RETURN     all  --  any    lo      anywhere             anywhere             ! owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             anywhere             owner GID match 1337
    0     0 RETURN     all  --  any    any     anywhere             localhost           
    0     0 ISTIO_REDIRECT  all  --  any    any     anywhere             anywhere            

Chain ISTIO_REDIRECT (1 references)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 REDIRECT   tcp  --  any    any     anywhere             anywhere             redir ports 15001
```

让我们一起再来仔细看下面这个图片（版权归原博主所有），同步观察上面iptables chain的规则。这里的分析主要针对红色的数字一对一解释：

![envoy-sidecar-traffic-interception](/images/envoy-sidecar-traffic-interception-jimmysong-blog.png)

1.  `productpage` 服务对`reviews` 服务发送 TCP 连接请求 
2. 请求进入`reviews`服务所在Pod内核空间，被netfilter拦截入口流量，经过`PREROUTING`链然后转发至`ISTIO_INBOUND`链
3. 在被`ISTIO_INBOUND`链被这个规则`-A ISTIO_INBOUND -p tcp -j ISTIO_IN_REDIRECT`拦截再次转发至`ISTIO_IN_REDIRECT`链
4. **`ISTIO_IN_REDIRECT`链直接重定向至 envoy监听的 `15006` 入口流量端口**
5. 在 envoy 内部经过一系列入口流量治理动作后，发出TCP连接请求 `reviews` 服务，这一步对envoy来说属于出口流量，会被netfilter拦截转发至出口流量`OUTPUT`链
6. `OUTPUT`链转发流量至`ISTIO_OUTPUT`链
7. 目的地为localhost，不能匹配到转发规则链，直接`RETURN`到下一个链，即`POSTROUTING`链
8. sidecar发出的请求到达`reviews`服务`9080`端口
9. `reviews`服务处理完业务逻辑后响应sidecar，这一步对`reviews`服务来说属于出口流量，再次被netfilter拦截转发至出口流量`OUTPUT`链
10. `OUTPUT`链转发流量至`ISTIO_OUTPUT`链
11. 发送非localhost请求且为`istio-proxy`用户空间的流量被转发至` ISTIO_REDIRECT`链
12. **` ISTIO_REDIRECT`链直接重定向至 envoy监听的 `15001` 出口流量端口**
13. 在 envoy 内部经过一系列出口流量治理动作后继续发送响应数据，响应时又会被netfilter拦截转发至出口流量`OUTPUT`链
14. `OUTPUT`链转发流量至`ISTIO_OUTPUT`链
15. 流量直接`RETURN`到下一个链，即`POSTROUTING`链 

针对上文其实我还有两个疑问点，还请大家不吝指教：

- 上面的理解没有写第`16`点，博主的图中的`16`点还会再进` ISTIO_REDIRECT`链，我们可以看到` ISTIO_REDIRECT`链中只有一个改写端口转发的规则，这样岂不是会进入一个死循环？或者是我还没有理解清楚
- envoy 转发流量是不是自己新建立的tcp 连接请求还是通过修改请求报文地址来实现的。因为对c++了解有限，无法查阅其源码去一探究竟

从整个流量拦截流程大家也可以看出，路径这么长， 在大并发场景下肯定会损失转发性能。目前业界有一些框架在试着缩短这个拦截路径，让大家拭目以待吧。



## 参考文献

https://www.servicemesher.com/blog/sidecar-injection-iptables-and-traffic-routing/ 

https://www.frozentux.net/iptables-tutorial/cn/iptables-tutorial-cn-1.1.19.html#REDIRECTTARGET

http://www.zsythink.net/archives/1199