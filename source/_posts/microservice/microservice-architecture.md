---
title: 微服务平台架构
date: 2020-04-30 19:30
tags: microservice
categories:	微服务
---

##  微服务概念

微服务是一种用于构建应用的架构方案。微服务架构有别于更为传统的单体式方案，可将应用拆分成多个核心功能。每个功能都被称为一项服务，可以单独构建和部署，这意味着各项服务在工作和出现故障时不会相互影响。 

![monolithic-vs-microservices](/images/microservice/monolithic-vs-microservices.png)

## 微服务组件

下图为搭建微服务平台常用到的一些生态组件

![微服务架构组件](/images/microservice/微服务架构组件.svg)

<!--  more -->

下图为对应生态组件的一些开源实现，相关源码在github中全部可以找到

![微服务架构组件实现](/images/microservice/微服务架构组件实现.svg)

后续我会根据之前的一些使用经验来试着分析这中间部分组件的架构及原理(给自己挖坑)，这么多不知道一个个写完不知道要到什么时候了！！！

## 参考文献

[https://zh.wikipedia.org/wiki/微服務](https://zh.wikipedia.org/wiki/微服務) 

https://microservices.io/ 