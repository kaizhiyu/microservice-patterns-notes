# 技术分享：浅谈Service Mesh在瓜子的实践

- zeyaries

**2019 年 7 月 01 日

> 过去三年，[微服务](https://zh.wikipedia.org/wiki/微服務)成为业界的技术热点，大量互联网公司都在做微服务架构落地，新一代微服务开发技术悄然兴起，[Service Mesh](https://jimmysong.io/posts/what-is-a-service-mesh/) 便是其中之一，该技术起初由 Linkerd 的 CEO William 提出，其中文翻译为服务网格，功能在于处理服务间通信，负责实现请求的可靠传递。本文，瓜子效能团队分享了在 K8S 的基础上，通过 Sidecar 模式进行 Service Mesh 的实践经历。



![img](https://static001.geekbang.org/wechat/images/73/73da4e90ae9b24d62f30482c96e8444e.jpeg)



## 一、背景



起初，[瓜子](https://www.guazi.com/bj/?ca_s=pz_baidu&ca_n=tbmkbturl&scode=10103000312)内部各业务线团队为了方便、快速地开发后端服务，使用了各种传统后端开发框架，顺利保障各项业务上线。随着时间的推移，瓜子的系统规模越来越大，架构复杂度也越来越高。为了更好的应对传统开发方式带来的挑战，瓜子开始落地微服务化，后端项目不断拆分。在这个过程中，微服务化给我们带来了更好的灵活性、扩展性，良好的故障弹性以及不受技术栈限制等优势。



但是，随着瓜子业务的高速发展，业务开发范围进一步增大，微服务的缺点逐渐暴露出来。综合前期的技术框架以及微服务架构，主要存在如下不足：



1. 当前通信框架过多、依赖版本众多，维护及后续扩展存在诸多不便，比如加入 SSO 验签中间件、黑白名单中间件时，需要针对不同的框架进行开发，效率及维护都有很大挑战；
2. 前期框架在服务治理方面能力较弱，没有服务注册与服务发现，不便于后续微服务架构推行；
3. 服务实例快速增长，请求在各服务间穿梭变得越来越复杂，每个服务出现问题都可能造成整个项目出现异常，并且不容易定位到具体问题，给运维加大了难度；
4. 每个微服务都需要重复实现一些基础功能，比方说鉴权管理、负载均衡、熔断等，对代码侵入性强，重复工作的同时提升后续技术替换成本。



根据上述不足，瓜子内部开始考虑实践 [Service Mesh](https://www.infoq.cn/article/2017/12/why-service-mesh)，这给瓜子带来了如下好处：



1. 业务团队更加专注核心业务逻辑和功能，不用过多关注基础设施；
2. 一套基础设施能够灵活支持多种语言的业务开发，很好的解决服务异构化程度较高的场景；
3. 业务团队与基础架构团队解耦，基础设施与业务应用代码解耦。



## 二、Service Mesh 实践



### 2.1 整体架构



作为一种基础设施，[K8S](https://zh.wikipedia.org/wiki/Kubernetes) 的 pod 天然可以支持多个 Container，能够非常方便地运行 [sidecar ](http://dockone.io/article/6012)模式。因此，我们决定在 K8S 的基础上，通过 sidecar 模式进行 Service Mesh 实践。



具体架构图如下所示：



![img](https://static001.geekbang.org/wechat/images/6c/6c2fb57170c4f65a689074c67473a2dc.png)



图 1 Service Mesh 框架图



在 Sidecar 部署方式中，每个应用容器都会部署一个伴生容器。对于 Service Mesh，sidecar 接管进出应用程序容器的所有网络流量。



基于 sidecar，我们可以实现服务之间的调用拦截，服务之间的的所有流量都会经过 sidecar，并通过其进行转发。所有 sidecar 组成了一个服务网格，再通过统一的地方与各 sidecar 交互，就能控制网格中的流量运转。



### 2.2 gRPCx（Service Mesh 实现的载体）



我们采用 gRPC 作为通信框架，在原生 gRPC 的基础上，结合瓜子业务情况对其进行增强，帮助用户更加方便的使用 gRPC 相关功能。同时，整合并收敛前期 web 框架，解决前期框架过多维护及扩展困难的问题。gRPCx 提供可插拔的中间件，用户在使用过程中能够方便地加入埋点、验签、鉴权、监控等功能。支持上下文穿透，gRPCx 能够将 traceID、用户 ID 等信息合并到 gRPC 请求中，一起发给服务端。另外，gRPCx 较为完整地满足服务开发及治理的基础功能，包括优雅的服务注册、发现及下线，服务端负载均衡及高可用等功能。基于服务注册与服务发现，服务在 K8S 上部署时可以没有域名，直接通过 IP 访问服务，减少 DNS 服务的压力。



gRPCx 是我们实现 Service Mesh 的关键所在，其框架如下图所示：



![img](https://static001.geekbang.org/wechat/images/e4/e4e4ddfcab3dfdef47e8b3852deb4c81.png)



图 2 gRPCx 框架图



gRPCx 相关组件可以被定义为在微服务拓扑结构中处理各个服务之间通信的基础设施层，不仅能够帮助降低微服务体系结构的相关复杂性，同时也能够提供服务治理等功能。



Registry：使用 ETCD 作为注册中心，用于服务注册及发现；



gRPC server：使用 gRPCx 框架实现，用于提供业务服务功能；



gRPC bridge：与 gRPC server 通过 sidecar 的方式部署在 K8S 的同一个 pod 中，用于获取 gRPC server 的服务信息，并将其注册到 Registry 中。同时提供 HTTP 桥接功能，将 HTTP 请求转换为 gRPC 请求发送给 gRPC server;



gRPC proxy：作为 gRPC 请求的入口，根据 gRPC 请求的服务信息在 Registry 中查找服务信息进行服务发现，然后将 gRPC 请求转发到目标业务服务器，从而访问 gRPC 服务；



gRPC gateway：作为 HTTP 请求的入口，根据请求的服务信息在 Registry 中查找服务信息进行服务发现，然后将 HTTP 请求转换为 gRPC 请求发往目标业务服务器上，从而使得 HTTP 请求能够访问 gRPC 服务。



### 2.3 服务治理



基于 K8S 的 sidecar 模式，我们将复杂的服务治理从业务服务中分离出来，并将这部分功能放入到 sidecar 中进行处理。sidecar 中的服务代理提供诸如流量及熔断控制、服务注册与发现、监控、验签以及安全埋点等功能特性，开发人员在使用时只关注自己的业务功能开发即可。同时，通过 sidecar 与 gRPCx 结合的方式，我们实现了 gRPC 应用提供 HTTP 以及 gRPC 两种访问接口，使用起来较为灵活。



**2.3.1 HTTP 请求桥接**



![img](https://static001.geekbang.org/wechat/images/2b/2bcc90882a3d70d92ed41ef06955edfd.png)



图 3 gRPC bridge 请求桥接工作流程图



基于服务调用方可以通过 HTTP 方式访问 gRPC 服务，gRPC bridge 会将 HTTP 请求转换为 gRPC 请求，然后再发往 gRPC server。这种方式使 gRPC server 也具备提供 HTTP 服务的能力，方便需要使用 HTTP 请求的调用。



**2.3.2 服务注册**



![img](https://static001.geekbang.org/wechat/images/e4/e4fd0727e7285b151b5e9779110a6db8.png)



图 4 服务注册工作流程图**服务发布**



gRPC server 使用 gRPCx 框架创建服务后，将提供服务信息获取接口。我们基于 K8S 将 gRPC bridge 作为 sidecar 与 gRPC server 部署在同一个 pod 中，gRPC bridge 通过配置信息获取 gRPC server 的端口信息，然后监听 gRPC server 服务信息获取接口，当 gRPC server 服务启动后，将服务信息注册到 Registry 中。



**服务下线**



gRPC bridge 提供服务下线功能，当 gRPC server 服务停止后，其会从 Registry 摘除对应节点信息，并通过监听 gRPC server 服务信息获取接口，当 gRPC server 服务信息更新后，会更新 Registry 中对应节点的信息。



用户能够通过 Registry 的 web UI，方便查看其服务被注册的具体情况。



**2.3.3 服务发现**



考虑到 K8S 的服务发现基于 DNS 寻址实现，部署到 K8S 上面的服务会生成一条 DNS 记录指向其被分配的的 cluster IP，其他服务在通过 K8S namespace+ 服务名去调用服务。我们基于 gRPC 实现的服务发现的粒度更细，能够使用具体接口（service name+method name）进行服务发现，并且支持 HTTP 与 gRPC 两种调用的服务发现，使用起来更加灵活方便。而且服务发现会有降级策略，Registry 宕机后，服务发现将会基于内存中存储的信息进行。因此，我们并没有使用 K8S 的服务发现。



**基于 gRPC proxy 的服务发现**



![img](https://static001.geekbang.org/wechat/images/4a/4a514cb2c90038653816ddd15147fb41.png)



图 5 gRPC proxy 服务发现工作流程图



当 gRPC 请求发往 gRPC proxy（该服务地址确定，后续不会发生变化）后，proxy 根据服务信息进行服务发现，获取 Registry 中对应的服务地址，然后会将 gRPC 请求转发到目标 gRPC server 上。



**基于 gRPC gateway 的服务发现**



![img](https://static001.geekbang.org/wechat/images/a9/a975c7262c2d71224fe27729fc2c7e70.png)



图 6 gRPC gateway 服务发现工作流程图



当 HTTP 请求发往 gRPC gateway（该服务地址确定，后续不会发生变化）后，gateway 根据服务信息进行服务发现，获取 Registry 中对应的服务地址，然后会将 HTTP 请求转换为 gRPC 请求并发送到目标 gRPC server 上。



当服务部署多个实例时，gRPC gateway 跟 gRPC proxy 会采用 RoundRobin 负载均衡策略，最终路由到其中一个服务实例上。



### 2.4 健康检查与容灾



**2.4.1 健康检查**



使用 gRPCx 框架创建服务后，gRPC server 将提供 heartbeat 的 gRPC 接口。我们的健康检查是基于宿主机提供的 HTTP 形式的心跳检查，将心跳检查发送给 gRPC bridge，由 gRPC bridge 将 HTTP 形式的心跳检查转换为 gRPC 形式的心跳检查，gRPC bridge 并将 gRPC 的返回转换为 HTTP 的返回，发给心跳发起服务。



**Liveliness**



在服务启动成功后心跳检测间隔时间，如果检测的 StatusCode 非 200 会自动重启服务所在容器。



**Readiness**



属性在服务启动成功后心跳检测间隔时间，连续 3 次检查失败后，会将当前流量摘除，应用不会重启。当下一次检测正常时，就会恢复流量。在滚动更新实例时，会先将其中一个老实例摘除流量并终止老实例，同时起一个新实例。在 readiness 检测正常后，就会分配流量给新实例，同时更新下一个老实例。这样就会保证流量不丢失。



服务关闭或者服务不可用时，gRPC bridge 会从 Registry 中摘除服务对应的相关信息；服务启动或者可用时，gRPC bridge 会将服务对应的 IP 及 API 信息注册到 Registry 中。



**2.4.2 容灾单独**



通过 gRPC bridge 服务主动探测也存在隐患，当 gRPC bridge 出现问题或者是 gRPC bridge 到 gRPC server 网络存在问题时，其无法调用 gRPC server 接口，从而无法到 Registry 中更新与摘除相关服务。此时，我们通过一个独立运行的 gRPC monitor 实例监控 gRPC server 服务的可用性，若 gRPC server 服务存在问题不可用时，gRPC monitor 会到 Registry 中摘除相关节点信息，调用方能够及时感知服务下线，进一步保障服务注册与下线功能的完整性。



Registry 采用的是 ETCD 集群，随着业务发展，服务进一步增多，Registry 的压力会越来越大。由于 ETCD 使用 Raft 协议维护集群内各个节点状态的一致性，通过水平扩展 Registry 的节点数量来提升读性能，但是会降低集群的写性能，这是我们后续需要优化的一个点。



![img](https://static001.geekbang.org/wechat/images/b0/b020a25f1794ffef20e854b99249843d.png)



图 7 Registry 不可用时, 服务发现工作流程图



我们的服务发现采用本地内存缓存作为降级方案，当 Registry 集群完全宕机或者 Registry 集群连接不可达时，不会影响服务的正常调用。此外，基于 ETCD 的 watch 机制，当 Registry 中的服务信息发生变化时，能够及时更新内存中存储的服务信息。



出于对系统稳定性及安全性考虑，我们在服务 sidecar 中的 gRPC bridge 加入了熔断机制，当满足一定条件后，gRPC bridge 直接将请求熔断，并不会将请求转发给 gRPC server，减轻了服务端压力。接下来，我们会围绕 sidecar 中的 gRPC bridge 扩展出更多实用功能，方便开发人员使用。



### 2.5 日志与安全



![img](https://static001.geekbang.org/wechat/images/ac/ac15f4c6f695449000d691eefccc4265.png)



图 8 日志全链路追溯示意图



在微服务架构中，随着服务数量的增多，各节点之前的调用关系变得越来复杂，对我们查找问题带来了挑战。我们有时会为了追溯一个问题，统计几个甚至几十个服务日志信息，然后再进行问题查找，这样使用起来非常不方便，不能快速、准确的定位问题。gRPCx 框架服务在其他服务调用请求传入时就可通过 gRPCx 中间件自动加上标签 (traceID，如果传入的请求已经存在标签，则后续使用已存在的标签)，该 traceID 与请求一起传入到后台服务中，后台服务从 context 中获取该 traceID，与瓜子日志组件联合使用，在日志收集时自动加上 traceID，方便追踪全链路上与该请求相关的调用。如果涉及到多个服务之间的调用，通过该 traceID 能够很好地串联请求调用链，使开发人员非常容易的进行问题查找及追溯，减少与微服务结构相关的复杂性。



同时，基于 gRPCx 中间件，方便实现鉴权、验签、安全统一埋点等功能。我们在中间件提供了鉴权、验签等功能，统一接入瓜子 SSO，避免开发人员在业务代码中重复加入这些功能，将鉴权、验签等功能从业务代码中剥离出来，减少与业务代码的耦合，让业务代码更加干净。基于中间件，可以方便地进行安全埋点，将请求数据中需要收集的信息发送至 Kafka，并通过 Kafka 存入 Hive，方便后续分析使用。



## 三、总结与展望



在 K8S 的基础上，瓜子通过 sidecar 模式，使用一个与主服务独立的代理服务（使用 gRPC 与主服务进行通信），让服务之间的通信更加简单、高效。在 sidecar 中运行的 gRPC bridge 不仅能够提供业务之外的功能，比如路由、监控、日志收集和访问控制等，而且还能作为一个透明的基础结构层，存在于微服务与外部网络之间。在 gRPC bridge 的帮助下，一套 gRPC 服务代码能够同时提供 HTTP、gRPC 的服务接口，方便开发人员使用，也方便将业务拆分。调用方使用 gRPC gateway、gRPC proxy 能够更加方便、快捷的调用各个微服务，再辅以鉴权、验签以及安全埋点等中间件，配合可靠的全链路日志追溯、优雅的服务治理、成熟的健康检查机制等，瓜子整个微服务体系变得更加完善。



现阶段，瓜子的 Service Mesh 实践还处于起步阶段，存在诸多不足。未来，瓜子将结合复杂的业务环境，在不断完善现有 gRPCx 生态的基础上，围绕 Service Mesh 开展工作，例如，更加完善的微服务监控体系、故障注入和容错、高级服务路由。服务熔断与降级、更加完备的容灾、高可用策略等，降低单个应用自身复杂度，并在 K8S 的支撑下，简化部署、管理、监控、维护等较为繁琐性的工作。我们的开发人员只用在此基础上享受基于 gRPCx 生态的 Service Mesh 带来的使用便利。



**作者介绍**

zeyaries，瓜子效能团队一员，效能团队致力于提升瓜子技术团队的研发效率，为瓜子研发相关人员提供工具支撑。通过对前沿技术不断地探索与研究，落地应用到实际中，帮助业务团队提升交付速率以及交付质量。



2019 年 7 月 01 日 08:008018

文章版权归极客邦科技InfoQ所有，未经许可不得转载。



原文：https://www.infoq.cn/article/oeQ0Wm4e4OP*1LMN6IEo