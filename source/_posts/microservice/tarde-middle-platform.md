---
title: 交易中台技术栈全景
date: 2020-04-30 20:00
tags: 交易中台
categories:	微服务
---

## 交易中台技术全景图

![交易中台技术栈全景](/images/microservice/交易中台技术栈全景.svg)

上图是个人根据之前的一些工作积累描绘出来的，当然这些只是冰山一角。其中大部分组件都有在公司实际使用过，里面都是开源组件，因为平常工作都是使用Java，所以基本都是从Java里面做的选型。

至于最终技术栈的选择，每个人有不同的认知及经验差异，可能会有其他的一些更好的想法，这个非常好。没有最好的，只有更合适的。可以结合公司的需求，团队的成员熟知度等因素综合考量后，完成这个技术体系的搭建即可。

<!--  more -->

下面的表格为项目对应的github地址，方便查阅

| 序号 | 组件名         | github地址                               |
| :--: | -------------- | ---------------------------------------- |
|  1   | apollo         | https://github.com/ctripcorp/apollo      |
|  2   | nacos          | https://github.com/alibaba/nacos         |
|  3   | soul           | https://github.com/Dromara/soul          |
|  4   | redis          | https://github.com/antirez/redis         |
|  5   | sentinel       | https://github.com/alibaba/Sentinel      |
|  6   | rocketmq       | https://github.com/apache/rocketmq       |
|  7   | lmstfy         | https://github.com/bitleak/lmstfy        |
|  8   | saturn         | https://github.com/vipshop/Saturn        |
|  9   | flink          | https://github.com/apache/flink          |
|  10  | shardingsphere | https://github.com/apache/shardingsphere |
|  11  | seata          | https://github.com/seata/seata           |
|  12  | elasticsearch  | https://github.com/elastic/elasticsearch |
|  13  | porter         | https://github.com/sxfad/porter          |
|  14  | skywalking     | https://github.com/apache/skywalking     |
|  15  | prometheus     | https://github.com/prometheus/prometheus |
|  16  | id-generator   | https://github.com/Meituan-Dianping/Leaf |



##  中台实践体验

1. 中台是一把手工程，全员共识是关键
2. 中台是一次变革，避免急功近利，企业要有中长期投入的准备
3. 要选择成熟的技术平台，关注稳定性和未来 
4. 中台本身不能解决所有问题。跟任何的方法论一样，只适用于特定场景、特定问题