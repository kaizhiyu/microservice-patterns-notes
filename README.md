# 微服务架构模式

> 参考：A pattern language for microservices，仅作学习交流用。
>
> https://microservices.io/patterns/

# 微服务的模式语言

> [译注]：模式语言提供了讨论问题的交流术语，它明确了特定场景、特定问题的解决方案和延伸性思考。模式语言主要目的是帮助开发者解决在设计和编程中遇到的共同的问题，即清晰的问题陈述、体现问题的解决方案以及推动解决方案的力量（Force）的清晰表述。
>
> 微服务架构作为一个现在流行的服务架构，也有一套属于自己的模式。这篇文章是微服务架构相关模式语言的一个提纲。Chris Richardson 从不同的角度，对相关的模式进行了分类。可以点击链接查看每个模式的详细描述。下图通过虚线框细分了不同的微服务模式。

![img](https://microservices.io/i/MicroservicePatternLanguage.jpg)

# 核心模式（Application architecture patterns）

您为应用程序选择哪一种架构？

- [单体架构（Monolithic architecture）](https://microservices.io/patterns/cn/monolithic.html) - 采用单一部署单元的方式架构应用
- [微服务架构（Microservices architecture）](https://microservices.io/patterns/cn/microservices.html) - 采用一组松耦合服务的方式架构应用

# 服务拆分（Decomposition）

如何把应用拆分为若干个服务？

- [根据业务能力拆分（Decompose by business capability）](https://microservices.io/patterns/cn/decomposition/decompose-by-business-capability.html)new - 根据业务能力界定服务的范围
- [根据领域的子域拆分（Decompose by subdomain）](https://microservices.io/patterns/cn/decomposition/decompose-by-subdomain.html)new - 根据领域驱动设计中子域的概念界定服务的范围

# 部署模式（Deployment patterns）

如何部署应用程序的服务？

- [单主机上部署服务的多个实例（Multiple service instances per host）](https://microservices.io/patterns/cn/deployment/multiple-services-per-host.html) - 把服务的多个实例部署在一台主机上
- [单主机上部署服务的单个实例（Single service instance per host）](https://microservices.io/patterns/cn/deployment/single-service-per-host.html) - 把服务的单一实例部署在它独享的主机上
- [服务实例与虚拟机一一对应（Service instance per VM）](https://microservices.io/patterns/cn/deployment/service-per-vm.html) - 把服务的单一实例部署在它独享的虚拟机上
- [服务实例与容器一一对应（Service instance per Container）](https://microservices.io/patterns/cn/deployment/service-per-container.html) - 把服务的单一实例部署在它独享的容器上
- [无服务器部署（Serverless deployment）](https://microservices.io/patterns/deployment/serverless-deployment.html) - 使用无服务器部署平台部署服务实例
- [服务部署平台（Service deployment platform）](https://microservices.io/patterns/deployment/service-deployment-platform.html)new - 使用提供服务抽象能力的高度自动化部署平台部署服务实例

# 需要关注的边界问题（Cross cutting concerns）

如何处理服务实例与外界交互的问题？

- [微服务的基底（Microservice chassis）](https://microservices.io/patterns/microservice-chassis.html) - 一个用于服务实例与外界交互和简化服务开发的框架
- [配置信息外部化（Externalized configuration）](https://microservices.io/patterns/externalized-configuration.html)new - 把类似数据库连接、访问密钥等配置信息外部化

# 通信模式（Communication patterns）

## 风格

应该选择怎样的通信机制来进行服务间通讯和外部客户端通讯？

- [远程过程调用（Remote Procedure Invocation）](https://microservices.io/patterns/cn/communication-style/rpi.html) - 使用基于 RPI 协议的服务间通讯方式
- [消息（Messaging）](https://microservices.io/patterns/cn/communication-style/messaging.html) - 使用异步消息进行服务间通讯
- [领域独用协议（Domain-specific protocol）](https://microservices.io/patterns/cn/communication-style/domain-specific.html)new - 使用特定领域的通讯协议（如 SIP，等）

## 外部 API

如何处理外部客户端与服务之间的通讯？

- [API 网关（API gateway）](https://microservices.io/patterns/cn/apigateway.html) - 为每一类客户端提供一个访问服务的独特接口
- [服务前端的后端（Backend for front-end）](https://microservices.io/patterns/cn/apigateway.html) - 为每一类客户端都提供一个独立的 API 网关

## 服务发现

一个基于 RPI 的客户端如何在网络上发现服务实例的位置？

- [客户端服务发现（Client-side discovery）](https://microservices.io/patterns/cn/client-side-discovery.html) - 客户端通过直接查询服务注册表获取服务实例的位置
- [服务器端服务发现（Server-side discovery）](https://microservices.io/patterns/cn/server-side-discovery.html) - 路由模块通过查询服务注册表获取服务实例的位置
- [服务注册表（Service registry）](https://microservices.io/patterns/cn/service-registry.html) - 一个记录了服务实例位置的数据
- [自注册（Self registration）](https://microservices.io/patterns/cn/self-registration.html) - 服务实例自己完成向服务注册表的注册
- [第三方注册（3rd party registration）](https://microservices.io/patterns/cn/3rd-party-registration.html) - 通过第三方模块来进行服务实例信息到服务注册表的注册过程

## 可靠性

如何避免由于服务故障或网络中断所引起的故障蔓延到其他服务？

- [断路器（Circuit Breaker）](https://microservices.io/patterns/cn/reliability/circuit-breaker.html) - 当远端服务返回的故障率超过一定的阀值时，客户端代理（比如 API 网关）对远程服务的调用将立刻返回失败的信息

# 数据管理（Data management）

如何实现数据一致性和查询？

- [每服务数据库（Database per Service）](https://microservices.io/patterns/cn/data/database-per-service.html) - 每个服务都拥有它私有的数据库
- [共享数据库（Shared database）](https://microservices.io/patterns/cn/data/shared-database.html) - 服务之间共享同一个数据库
- [事件驱动架构（Event-driven architecture）](https://microservices.io/patterns/cn/data/event-driven-architecture.html) - 使用事件来维护服务间的数据一致性
- [事件溯源（Event sourcing）](https://microservices.io/patterns/cn/data/event-sourcing.html) - 以一连串事件的方式来持久化聚合
- [事务日志跟踪（Transaction log tailing）](https://microservices.io/patterns/cn/data/transaction-log-tailing.html) - 跟踪数据库的日志变更并由此对外发布消息
- [数据库触发器（Database triggers）](https://microservices.io/patterns/cn/data/database-triggers.html) - 使用触发器来捕获对数据的修改
- [应用程序事件（Application events）](https://microservices.io/patterns/cn/data/transactional-outbox.html) - 应用程序从消息队列获取事件并插入数据库中
- [命令查询职责分离（CQRS）](https://microservices.io/patterns/cn/data/cqrs.html) - 维护一个或者多个重要的数据视图以供高效率的查询

# 安全（Security）

如何向服务实例传递访问客户端的身份信息？

- [访问令牌（Access Token）](https://microservices.io/patterns/cn/security/access-token.html)new - 服务实例通过访问令牌来安全地传递客户端的身份信息

# 测试（Testing）

如何更便捷的测试？

- [服务组件测试（Service Component Test）](https://microservices.io/patterns/cn/testing/service-component-test.html)new - a test suite that tests a service in isolation using test doubles for any services that it invokes
- [服务集成协议测试（Service Integration Contract Test）](https://microservices.io/patterns/cn/testing/service-integration-contract-test.html)new - a test suite for a service that is written by the developers of another service that consumes it

# 可观测性（Observability）

如何掌握一个运行中微服务应用的行为并进行有效的故障排错？

- [应用日志（Log aggregation）](https://microservices.io/patterns/cn/observability/application-logging.html)new - 聚合应用程序产生的日志文件
- [应用指标（Application metrics）](https://microservices.io/patterns/cn/observability/application-metrics.html)new - 在代码中实现收集应用运营过程中各类指标的功能
- [审计日志（Audit logging）](https://microservices.io/patterns/cn/observability/audit-logging.html)new - 把用户行为记录在数据库中供日后核查
- [分布式追踪（Distributed tracing）](https://microservices.io/patterns/cn/observability/distributed-tracing.html)new - 在服务代码中针对每一个外部访问，都分配一个唯一的标识符，并在跨服务访问时传递这个标识符以供追踪分布式引发的问题。例如，当通过一个集中式服务处理外部请求时，记录请求本身的信息以及请求的开始和结束时间。
- [异常追踪（Exception tracking）](https://microservices.io/patterns/cn/observability/exception-tracking.html)new - 把所有服务程序代码触发的异常信息都汇聚到集中的异常追踪服务，并根据一定的逻辑对开发者或运维人员发出通知。
- [健康检查 API（Health check API）](https://microservices.io/patterns/cn/observability/health-check-api.html)new - 一个监控服务可调用的 API，通常返回服务健康度信息，或对 ping 等心跳检查请求做出响应。

# UI 模式（UI patterns）

如何将源自多个服务的信息组织在一起生成 UI 界面或 Web 页面？

- [服务器端页面碎片化元素构建（Server-side page fragment composition）](https://microservices.io/patterns/cn/ui/server-side-page-fragment-composition.html)new - 在服务器端通过编排由多个业务或领域相关后端服务生成的 HTML 片段来构建前端输出的页面内容
- [客户端 UI 构建（Client-side UI composition）](https://microservices.io/patterns/cn/ui/client-side-ui-composition.html)new - 在客户端通过编排由多个业务或领域相关 UI 组件生成的 HTML 片段来构建前端输出的页面内容



