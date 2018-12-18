---
layout: post
title: Spring Cloud
category: arch
comments: true
---

## 一、什么是Spring Cloud?

Spring Cloud 基于 Spring Boot，为微服务体系开发中的架构问题，提供了一整套的解决方案——服务注册与发现，服务消费，服务保护与熔断，网关，分布式调用追踪，分布式配置管理等。

核心功能：

- 分布式/版本化配置
- 服务注册和发现
- 路由
- 服务和服务之间的调用
- 负载均衡
- 断路器
- 分布式消息传递

### 1.1 组件架构

![spring-cloud-coms](/images/201812/springcloud-com.png)

流程：

- (来自Mobile、页面的）请求统一通过 API 网关（Zuul）来访问内部服务。
- 网关接收到请求后，从注册中心（Eureka）获取可用服务。
- 由 Ribbon 进行均衡负载后，分发到后端具体实例。
- 微服务之间通过 Feign 进行通信处理业务。
- Hystrix 负责处理服务超时熔断。
- Turbine 监控服务间的调用和熔断相关指标。

### 1.2 工具框架
（真多）

- Spring Cloud Config 配置中心，利用 Git 集中管理程序的配置。
- Spring Cloud Netflix 集成众多Netflix的开源软件。
- Spring Cloud Netflix Eureka 服务中心（类似于管家的概念，需要什么直接从这里取，就可以了），一个基于 REST的服务，用于定位服务，以实现云端中间层服务发现和故障转移。
- Spring Cloud Netflix Hystrix熔断器，容错管理工具，旨在通过熔断机制控制服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。
- Spring Cloud Netflix Zuul 网关，是在云平台上提供动态路由，监控，弹性，安全等边缘服务的框架。Web网站后端所有请求的前门。
- Spring Cloud Netflix Archaius 配置管理API，包含一系列配置管理API，提供动态类型化属性、线程安全配置操作、轮询框架、回调机制等功能。
- Spring Cloud Netflix Ribbon 负载均衡。
- Spring Cloud Netflix Fegin REST客户端。
- Spring Cloud Bus 消息总线，利用分布式消息将服务和服务实例连接在一起，用于在一个集群中传播状态的变化。
- Spring Cloud for Cloud Foundry 利用 Pivotal Cloudfoundry 集成你的应用程序。
- Spring Cloud Cloud Foundry Service Broker 为建立管理云托管服务的服务代理提供了一个起点。
- Spring Cloud Cluster 集群工具，基于 Zookeeper, Redis, Hazelcast, Consul实现的领导选举和平民状态模式的抽象和实现。
- Spring Cloud Consul 基于 Hashicorp Consul 实现的服务发现和配置管理。
- Spring Cloud Security 安全控制，在 Zuul 代理中为 OAuth2 REST 客户端和认证头转发提供负载均衡。
- Spring Cloud Sleuth 分布式链路监控，SpringCloud 应用的分布式追踪系统，和 Zipkin，HTrace，ELK 兼容。
- Spring Cloud Data Flow 一个云本地程序和操作模型，组成数据微服务在一个结构化的平台上。
- Spring Cloud Stream 消息组件，基于 Redis，Rabbit，Kafka 实现的消息微服务，简单声明模型用以在 Spring Cloud应用中收发消息。
- Spring Cloud Stream App Starters 基于 Spring Boot 为外部系统提供 Spring 的集成。
- Spring Cloud Task 短生命周期的微服务，为 Spring Booot 应用简单声明添加功能和非功能特性。
- Spring Cloud Task App Starters。
- Spring Cloud Zookeeper 服务发现和配置管理基于 Apache Zookeeper。
- Spring Cloud for Amazon Web Services 快速和亚马逊网络服务集成。
- Spring Cloud Connectors 便于PaaS应用在各种平台上连接到后端像数据库和消息经纪服务。
- Spring Cloud Starters （项目已经终止并且在 Angel.SR2 后的版本和其他项目合并）
- Spring Cloud CLI 命令行工具，插件用 Groovy 快速的创建 Spring Cloud 组件应用。

### 1.3 优缺点

一些优点

- 有强大的 Spring 社区、Netflix 等公司支持，并且开源社区贡献非常活跃。
- 标准化的将微服务的成熟产品和框架结合一起，Spring Cloud 提供整套的微服务解决方案，开发成本较低，且风险较小。
- 基于 Spring Boot，具有简单配置、快速开发、轻松部署、方便测试的特点。
- 支持 REST 服务调用，相比于 RPC，更加轻量化和灵活（服务之间只依赖一纸契约，不存在代码级别的强依赖），有利于跨语言服务的实现，以及服务的发布部署。另外，结合 Swagger，也使得服务的文档一体化。
- 提供了 Docker 及 Kubernetes 微服务编排支持。
- 国内外企业应用非常多，经受了大公司的应用考验（比如 Netfilx 公司），以及强大的开源社区支持。

一些问题

- 支持 REST 服务调用，可能因为接口定义过轻，导致定义文档与实际实现不一致导致服务集成时的问题（可以使用统一文档和版本管理解决，比如 Swagger）。
- 另外，REST 服务调用性能会比 RPC 低一些（但也不是强绑定）
- Spring Cloud 整合了大量组件，相关文档比较复杂，需要针对性的进行阅读。

## 二、和Dubbo的对比
### 2.1 性能方面

![vs](/images/201812/dubbo-vs-springcloud.png)

Spring Cloud 抛弃了 Dubbo 的 RPC 通信，采用的是基于 HTTP 的 REST 方式。严格来说，这两种方式各有优劣。虽然从一定程度上来说，后者牺牲了服务调用的性能，但也避免了上面提到的原生 RPC 带来的问题。而且 REST 相比 RPC 更为灵活，服务提供方和调用方的依赖只依靠一纸契约，不存在代码级别的强依赖，这在强调快速演化的微服务环境下，显得更加合适。

问题：为什么 Dubbo 比 Spring Cloud 性能要高一些？

回答：因为 Dubbo 采用单一长连接和 NIO 异步通讯（保持连接/轮询处理），使用自定义报文的 TCP 协议，并且序列化使用定制 Hessian2 框架，适合于小数据量大并发的服务调用，以及服务消费者机器数远大于服务提供者机器数的情况，但不适用于传输大数据的服务调用。而 Spring Cloud 直接使用 HTTP 协议（但也不是强绑定，也可以使用 RPC 库，或者采用 HTTP 2.0 + 长链接方式（Fegin 可以灵活设置））。

### 2.2 组件方面

![vs](/images/201812/dubbo-vs-springcloud2.png)

Dubbo 专注 RPC 和服务治理，Spring Cloud 则是一个微服务架构生态。

### 2.3 内部组件对比

- ZooKeeper 和 Eureka 的区别  
鉴于服务发现对服务化架构的重要性，Dubbo 实践通常以 ZooKeeper 为注册中心（Dubbo 原生支持的 Redis 方案需要服务器时间同步，且性能消耗过大）。针对分布式领域著名的 CAP 理论（C——数据一致性，A——服务可用性，P——服务对网络分区故障的容错性），Zookeeper 保证的是 CP ，但对于服务发现而言，可用性比数据一致性更加重要，AP 胜过 CP，而 Eureka 设计则遵循 AP 原则。

### 2.4 Dubbo 和 Spring Cloud 比喻

使用 Dubbo 构建的微服务架构就像组装电脑，各环节我们的选择自由度很高，但是最终结果很有可能因为一条内存质量不行就点不亮了，总是让人不怎么放心，但是如果你是一名高手，那这些都不是问题；而 Spring Cloud 就像品牌机，在 Spring Source 的整合下，做了大量的兼容性测试，保证了机器拥有更高的稳定性，但是如果要在使用非原装组件外的东西，就需要对其基础有足够的了解。

## REF
> [Java 微服务框架选型（Dubbo 和 Spring Cloud）](http://www.cnblogs.com/xishuai/archive/2018/04/13/dubbo-and-spring-cloud.html)