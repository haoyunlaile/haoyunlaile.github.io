---
title: jaeger tracing
date: 2020-05-08 20:00
tags: ["opentracing","jaeger"]
categories:	微服务
---

##  简介

 [Jaeger](https://www.jaegertracing.io/)是 Uber 推出的一款开源分布式追踪系统（已从CNCF毕业），兼容 OpenTracing API。  它用于监视和诊断基于微服务的分布式系统，功能包括：

1.  分布式上下文传播 
2. 分布式链路跟踪 
4. 服务依赖分析 

## 示例

### Trace列表视图

![traces-jaeger-index](/images/microservice/traces-jaeger-index.png)

### Trace明细视图

![trace-detail-jaeger](/images/microservice/trace-detail-jaeger.png)



## 技术栈

- 基于Go实现
- 数据支持多种类型的后端存储
  - [Cassandra 3.4+](https://www.jaegertracing.io/docs/1.17/deployment/#cassandra)
  - [Elasticsearch 5.x, 6.x, 7.x](https://www.jaegertracing.io/docs/1.17/deployment/#elasticsearch)
  - [Kafka](https://www.jaegertracing.io/docs/1.17/deployment/#kafka)
  - memory storage

## 架构

 Jaeger可以作为单个进程进行部署，也可以作为可扩展的分布式系统进行部署。

 Jaeger 主要由以下几部分组成，架构非常清晰： 

- Jaeger Client - 为不同语言实现了符合 OpenTracing 标准的 SDK。应用程序通过 API 写入数据，client library 把 trace 信息按照应用程序指定的采样策略传递给 jaeger-agent.
- Agent - 它是一个监听在 UDP 端口上接收 span 数据的网络守护进程，它会将数据批量发送给 collector。它被设计成一个基础组件，部署到所有的宿主机上。Agent 将 client library 和 collector 解耦，为 client library 屏蔽了路由和发现 collector 的细节.
- Collector - 接收 jaeger-agent 发送来的数据，然后将数据写入后端存储。Collector 被设计成无状态的组件，因此您可以同时运行任意数量的 jaeger-collector。 当前，我们的管道会分析数据并为其建立索引，执行任何转换并最终存储它们。  Jaeger的存储设备是一个可插拔组件，目前支持  Cassandra, Elasticsearch and Kafka 存储.
- Query - 接收查询请求，然后从后端存储系统中检索 trace 并通过 UI 进行展示.
-  Ingester - 后端存储被设计成一个可插拔的组件，支持将数据写入 Cassandra, Elasticsearch.

Jaeger包含两种架构方案：

**一、收集器数据直接写入存储架构（tracing数据直接写入存储)**

![architecture-v1](/images/microservice/architecture-v1.png)



**二、收集器数据缓冲后异步写入存储架构（tracing数据通过kafka缓冲后再异步消费写入存储)** 

![architecture-v2](/images/microservice/architecture-v2.png)

> 个人推荐采用第二种架构方式部署



## 部署

为了快速搭建Jaeger环境，这里安装基于Helm部署（需要先搭建 Kubernetes 集群），可以参考前面写的文章来搭建。从  https://github.com/jaegertracing/helm-charts/tree/master/charts/jaeger  这里可以找到详细的部署流程，可以一步一步跟着执行部署。这里采用 **收集器数据直接写入存储架构** 部署

```shell
helm install jaeger jaegertracing/jaeger
```

> 官方推荐使用jaeger-operator来部署，可参考： https://www.jaegertracing.io/docs/1.17/operator/ 
>

安装完成后查看服务状态

```shell
kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                         AGE
jaeger-agent       ClusterIP   10.97.3.215      <none>        5775/UDP,6831/UDP,6832/UDP,5778/TCP,14271/TCP   7m45s
jaeger-cassandra   ClusterIP   None             <none>        7000/TCP,7001/TCP,7199/TCP,9042/TCP,9160/TCP    7m45s
jaeger-collector   ClusterIP   10.111.141.231   <none>        14250/TCP,14267/TCP,14268/TCP,14269/TCP         7m45s
jaeger-query       ClusterIP   10.97.103.64     <none>        80/TCP,16687/TCP                                7m45s
```

要访问jaeger ui 需要查看`jaeger-query`项目对外暴露的端口，我们看到通过helm安装，我们采用的默认配置，这里的网络类型是`ClusterIP`,如果想外网访问可以先临时改成`NodePort`的方式，执行如下命令编辑对应配置：

```shell
kubectl edit service jaeger-query
```

找到最下面的`ClusterIP`改成`NodePort`保存即可，保存后会自动生效

```shell
kubectl get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                                         AGE
jaeger-agent       ClusterIP   10.97.3.215      <none>        5775/UDP,6831/UDP,6832/UDP,5778/TCP,14271/TCP   8m38s
jaeger-cassandra   ClusterIP   None             <none>        7000/TCP,7001/TCP,7199/TCP,9042/TCP,9160/TCP    8m38s
jaeger-collector   ClusterIP   10.111.141.231   <none>        14250/TCP,14267/TCP,14268/TCP,14269/TCP         8m38s
jaeger-query       NodePort    10.97.103.64     <none>        80:31067/TCP,16687:31381/TCP                    8m38s
```

可以发现现在`jaeger-query`的网络类型已经变成了`NodePort`，现在可以通过流量访问Jaeger Ui了

这里的地址是 http://47.57.100.110:31067/search  (注意，IP地址及端口根据自己控制台的实际输出填入就行)

进入页面后可以到刚才部署的UI界面，并查询`jaeger-query`项目本身的tracing信息。

我在列表页面找到一个trace_id: `73c00aa573bf1ed0` 临时保存下它，后面分析会用到，打开后界面如下。

![jaeger-query-ui](/images/microservice/jaeger-query-ui.png)



## traces存储结构

我们可以在jaeger源代码中找到后端cassandra的存储结构，具体信息可以看这里，位置比较隐蔽：

https://github.com/jaegertracing/jaeger/blob/master/plugin/storage/cassandra/schema/v001.cql.tmpl 

不过我们可以登录Pod查看创建后的数据结构信息(cassandra)。让我们一探究竟，首先登入cassandra对应的docker镜像，然后通过cql 连接cassandra集群。

> 如果对cql不了解的可以查看对应文档： https://cassandra.apache.org/doc/latest/cql/ 

```shell
kubectl exec -it jaeger-cassandra-0 --container jaeger-cassandra -- /bin/bash
cqlsh

Connected to jaeger at 127.0.0.1:9042.
[cqlsh 5.0.1 | Cassandra 3.11.6 | CQL spec 3.4.4 | Native protocol v4]
Use HELP for help.
```

进入对应的space，查看里面对应的表信息

```cql
cqlsh> desc keyspaces;    #查看有哪些keyspaces

jaeger_v1_test  system_auth  system_distributed
system_schema   system       system_traces     

cqlsh> use jaeger_v1_test;   #切换到jaeger对应的space
cqlsh:jaeger_v1_test> desc tables;    #查看jaeger space下面的表信息

service_name_index  service_names  service_operation_index  traces            
dependencies_v2     tag_index      duration_index           operation_names_v2
```

我们可以一个一个的表信息查看。这里我们主要看下保存我们trace信息的表 `service_name_index`

```CQL
cqlsh:jaeger_v1_test> desc traces;

CREATE TABLE jaeger_v1_test.traces (
    trace_id blob,
    span_id bigint,
    span_hash bigint,
    duration bigint,
    flags int,
    logs list<frozen<log>>,
    operation_name text,
    parent_id bigint,
    process frozen<process>,
    refs list<frozen<span_ref>>,
    start_time bigint,
    tags list<frozen<keyvalue>>,
    PRIMARY KEY (trace_id, span_id, span_hash)
) WITH CLUSTERING ORDER BY (span_id ASC, span_hash ASC)
    AND bloom_filter_fp_chance = 0.01
    AND caching = {'keys': 'ALL', 'rows_per_partition': 'NONE'}
    AND comment = ''
    AND compaction = {'class': 'org.apache.cassandra.db.compaction.TimeWindowCompactionStrategy', 'compaction_window_size': '1', 'compaction_window_unit': 'HOURS', 'max_threshold': '32', 'min_threshold': '4'}
    AND compression = {'chunk_length_in_kb': '64', 'class': 'org.apache.cassandra.io.compress.LZ4Compressor'}
    AND crc_check_chance = 1.0
    AND dclocal_read_repair_chance = 0.0
    AND default_time_to_live = 172800
    AND gc_grace_seconds = 10800
    AND max_index_interval = 2048
    AND memtable_flush_period_in_ms = 0
    AND min_index_interval = 128
    AND read_repair_chance = 0.0
    AND speculative_retry = 'NONE';
```

还记得我们开始保存的那个trace_id: `73c00aa573bf1ed0` 么，现在我们可以在这个表中查看它是如何保存的，我们可以使用下面的cql进行查询，查询前需要对界面上的trace_id进行补位填充`0x0000000000000000` ，这里一定要注意，最终在cql里面查询的trace_id为：`0x000000000000000073c00aa573bf1ed0`。

```
cqlsh:jaeger_v1_test> expand on;
Now Expanded output is enabled
cqlsh:jaeger_v1_test> select * from traces where trace_id=0x000000000000000073c00aa573bf1ed0;

@ Row 1
----------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 trace_id       | 0x000000000000000073c00aa573bf1ed0
 span_id        | 2300299680491247480
 span_hash      | 1417161953846781420
 duration       | 204491
 flags          | 1
 logs           | [{ts: 1589363774632310, fields: [{key: 'event', value_type: 'string', value_string: 'searching', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'trace_id', value_type: 'string', value_string: '4c423cfb69721367', value_bool: False, value_long: 0, value_double: 0, value_binary: null}]}]
 operation_name | readTrace
 parent_id      | 0
 process        | {service_name: 'jaeger-query', tags: [{key: 'jaeger.version', value_type: 'string', value_string: 'Go-2.22.1', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'hostname', value_type: 'string', value_string: 'jaeger-query-55c77745b5-ff8tt', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'ip', value_type: 'string', value_string: '192.168.61.148', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'client-uuid', value_type: 'string', value_string: '363a86b295da9842', value_bool: False, value_long: 0, value_double: 0, value_binary: null}]}
 refs           | [{ref_type: 'child-of', trace_id: 0x000000000000000073c00aa573bf1ed0, span_id: 8340678215617945296}]
 start_time     | 1589363774627367
 tags           | [{key: 'db.statement', value_type: 'string', value_string: '\n\t\tSELECT trace_id, span_id, parent_id, operation_name, flags, start_time, duration, tags, logs, refs, process\n\t\tFROM traces\n\t\tWHERE trace_id = ?', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'db.type', value_type: 'string', value_string: 'cassandra', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'component', value_type: 'string', value_string: 'gocql', value_bool: False, value_long: 0, value_double: 0, value_binary: null}, {key: 'internal.span.format', value_type: 'string', value_string: 'proto', value_bool: False, value_long: 0, value_double: 0, value_binary: null}]

此处省略4个Row....
(5 rows)
```

因为找到这个trace_id包含了5个span，所以这里查询出来了5条记录，可以通过这段文本及上面的图片进行一一观察，可以发现存储结构还是非常清晰的，UI界面需要展示的信息基本都可以很容易从里面取到。

我们再回过头来看看jaeger client 库thrift的结构（源码见：jaeger.thrift）

```idl
# 标签
struct Tag {
  1: required string  key
  2: required TagType vType
  3: optional string  vStr
  4: optional double  vDouble
  5: optional bool    vBool
  6: optional i64     vLong
  7: optional binary  vBinary
}

# 日志
struct Log {
  1: required i64       timestamp
  2: required list<Tag> fields
}

enum SpanRefType { CHILD_OF, FOLLOWS_FROM }

# Span 之间的关系
struct SpanRef {
  1: required SpanRefType refType
  2: required i64         traceIdLow
  3: required i64         traceIdHigh
  4: required i64         spanId
}

# Span
struct Span {
  1:  required i64           traceIdLow   # the least significant 64 bits of a traceID
  2:  required i64           traceIdHigh  # the most significant 64 bits of a traceID; 0 when only 64bit IDs are used
  3:  required i64           spanId       # unique span id (only unique within a given trace)
  4:  required i64           parentSpanId # since nearly all spans will have parents spans, CHILD_OF refs do not have to be explicit
  5:  required string        operationName
  6:  optional list<SpanRef> references   # causal references to other spans
  7:  required i32           flags        # a bit field used to propagate sampling decisions. 1 signifies a SAMPLED span, 2 signifies a DEBUG span.
  8:  required i64           startTime
  9:  required i64           duration
  10: optional list<Tag>     tags
  11: optional list<Log>     logs
}
```

基本上可以跟存储的数据结构一一对应上。



## 采样策略

Jaeger客户端支持4种采样策略，分别是：

1.  **Constant** (`sampler.type=const`)  采样率的可设置的值为 0 和 1，分别表示关闭采样和全部采样 
2.  **Probabilistic** (`sampler.type=probabilistic`)   按照概率采样，取值可在 0 至 1 之间，例如设置为 0.5 的话意为只对 50% 的请求采样 
3.  **Rate Limiting** (`sampler.type=ratelimiting`)  设置每秒的采样次数上限 。 例如，当sampler.param = 2.0时，它将以每秒2条迹线的速率对请求进行采样。 
4.  **Remote** (`sampler.type=remote`)  此为默认策略。 采样遵循远程设置，取值的含义和 `probabilistic` 相同，都意为采样的概率，只不过设置为 `remote` 后，Client 会从 Jaeger Agent 中动态获取采样率设置。 

为了最大程度地减少开销，Jaeger默认采用 0.1% 的采样策略采集数据 (1000次里面采集1次)。



## 客户端

所有Jaeger客户端库都支持OpenTracing API ，下面这些都是官方支持的客户端库

| 语言    | GitHub Repo                                                  |
| :------ | :----------------------------------------------------------- |
| Go      | [jaegertracing/jaeger-client-go](https://github.com/jaegertracing/jaeger-client-go) |
| Java    | [jaegertracing/jaeger-client-java](https://github.com/jaegertracing/jaeger-client-java) |
| Node.js | [jaegertracing/jaeger-client-node](https://github.com/jaegertracing/jaeger-client-node) |
| Python  | [jaegertracing/jaeger-client-python](https://github.com/jaegertracing/jaeger-client-python) |
| C++     | [jaegertracing/jaeger-client-cpp](https://github.com/jaegertracing/jaeger-client-cpp) |
| C#      | [jaegertracing/jaeger-client-csharp](https://github.com/jaegertracing/jaeger-client-csharp) |

其他语言的客户端库还在开发中，具体进展可以来这里查看 [issue #366](https://github.com/jaegertracing/jaeger/issues/366) 



## 参考文献

https://www.jaegertracing.io/docs/ 

https://github.com/jaegertracing/jaeger 