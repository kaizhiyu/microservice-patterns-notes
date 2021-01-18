# 腾讯云中间件团队在Service Mesh中的实践与探索

- 腾讯云中间件

**2021 年 1 月 11 日

Service Mesh 作为腾讯微服务平台（TSF）支持的微服务架构之一，产品化命名为 Mesh 微服务平台（Tencent Service Mesh Framework，简称 TSF Mesh），提供下一代微服务架构 - 服务网格（Service Mesh）的解决方案，覆盖公有云、私有云和本地化部署等多种场景。从 2018 年 8 月推出首个版本以来，已经陆续在金融、新零售、工业互联网，以及公司内部等生产环境落地。在产品落地过程中，遇到了一系列技术挑战，如非 Kubernetes 环境的支持、多租户隔离、与 Spring Cloud 服务框架的互通、海量服务实例下的域名解析等等。针对这些问题，通过自研以及社区合作，最终得以解决。本文主要从用户场景出发，以生产实践探索过程中遇到的挑战为切入点，梳理和总结应对的解决方案，以期望对 Service Mesh 的认识、对 TSF Mesh 产品的了解有所帮助。



## 背景介绍

什么是 Service Mesh ? 根据 Buoyant CEO，Service Mesh 理念的提出者和先行者，William Morgan 定义，Service Mesh（服务网格）是一个专注于处理服务间通信的基础设施层。用于解决服务间复杂拓扑中的可靠请求传递，是云原生技术栈的关键组件之一。

2018 年被很多人称为 “Service Mesh 之年”，这一年，业界几乎所有大厂都在尝试推出自己的 Service Mesh 产品。Service Mesh 中的明星项目 Istio 在这一年也是蓄势待发，作为 Google、IBM、Lyft 联合开发的开源项目，陆续发布了 0.5、0.6、0.7、0.8 和 1.0 版本，到 2018 年 7 月 31 日 1.0 GA 时，对 Istio 来说是一个重要的里程碑，官方宣称所有的核心功能都可以用于生产。

以 GitHub 上的 star 数量的角度来看一下 Istio 在近几年的受欢迎程度，下图显示的是 Istio 的 GitHub star 数量随时间变化曲线。可以看到在 2018 年，Istio 的 star 数量大概增长了一万，目前已经超过 2.2 万 颗星，其增长趋势也非常平稳。

![img](https://static001.geekbang.org/infoq/3b/3b14693f689bd52460894adbf56d6906.png)



早在 2017 年，腾讯云中间件团队就选定 Istio 为技术路线，开始 Service Mesh 的相关预研和研发工作。作为腾讯微服务平台（TSF）的无侵入式微服务框架的核心实现，于 18 年初在腾讯广告平台投入，打磨稳定后，陆续开始对外输出，目前在金融、工业互联网等领域都有落地案例。

产品落地过程并非一帆风顺，会遇到一些问题和挑战。接下来，首先以开源 Istio 为切入点，介绍一下 TSF Mesh,之后对 TSF Mesh 产品化探索过程中的部分典型问题以及解决方案进行梳理和分享。



## TSF Mesh 介绍

Mesh 微服务平台（Tencent Service Mesh Framework，简称 TSF Mesh），基于 Service Mesh 的理念，为应用提供服务自动注册与发现、服务路由、鉴权、限流、熔断等服务治理能力，且应用无需对源代码进行侵入式改造，即可与该服务框架进行集成。

在开发选型上，基于业界达到商用标准的开源软件 Istio 进行构建，主要原因如下：

- Istio 功能相对完备，mesh 该有的能力都有。
- 社区活跃，资源丰富，CNCF 成员，代表云原生标准化。
- Golang（Istio）& C++14（envoy）都是高性能语言，且运行起来资源使用灵活，独立性好，无 JVM 等外部依赖。



在了解 TSF Mesh 架构之前，先回顾一下 Istio 的架构图，如下图所示：



![img](https://static001.geekbang.org/infoq/3c/3c4d2a24ec765bcef693a20187e6156e.png)



在上面的架构图中，Istio Mesh 分为两块：数据面板和控制面板。envoy 在 Istio 中扮演数据面板的角色，作为服务的代理，被部署为 sidecar，服务无需感知 envoy 的存在；控制面板包含 Pilot，Mixer，Citadel 等组件。这些组件的主要功能如下：

- Envoy: 作为底层的代理，通常选用其扩展版本 istio-proxy，用于调度服务网格中所有服务的出入站流量。包含了丰富的内置功能，例如动态服务发现，负载均衡，HTTP/2&gRPC 代理，熔断器，健康检查，基于百分比流量拆分的灰度发布，故障注入，性能指标等。
- Pilot: 控制面的核心组件，为 Envoy 提供服务发现、智能路由（如 AB 测试、灰度发布）和弹性流量管理功能（如超时、重试、熔断），负责将高层的抽象的路由规则转化成低级的 envoy 的配置。
- Mixer: 提供策略检查和遥测功能。
- Citadel: 安全组件，提供了自动生成、分发、轮换与撤销密钥和证书功能。



TSF Mesh 整体架构上，其核心能力与开源的 Istio 保持一致，同时对 envoy、Pilot、Mixer、Pilot-agent 组件做了增强，并且新增组件 Apiserver 和 Mesh-dns。外围能力聚焦在安全性、易用性、可维护性和可观测性，如下图所示：

![img](https://static001.geekbang.org/infoq/27/272108e0467852ba73c0ddc8b49d662a.png)



运营支撑提供了运营端管理和租户端管理，比如租户端的角色管理，集群管理，命名空间管理，应用管理，服务治理等；运营端提供资源管理等。监控系统提供了日志功能，链路追踪，调用链拓扑图，指标监控等。基础组件为限流、注册中心、配置中心、日志采集和实时监控提供支撑。Paas 为应用部署提供支撑，比如 aPaas 等。

TSF Mesh 保留 Istio 所有的原生特性，同时对 Service Mesh 叠加了部分商业特性，如下：

- 平台解耦：支持K8S/VM/裸金属服务器环境
- 新旧兼容：支持 Spring Cloud 应用、Service Mesh 应用互通，统一治理
- 多租户隔离、管理支持
- 调用链、日志、监控落盘
- 高可用性



## TSF Mesh 产品化挑战

**1. 支持异构的计算平台**

尽管 Istio 强调自身可扩展性的重要性在于适配各种不同的平台，也可以对接其他服务发现机制，但在实际场景中，通过深入分析 Istio 几个版本的代码和设计，可以发现其重要的能力都是基于 Kubernetes 进行构建的。以下面两点为例：

- **服务配置管理**

Istio 的所有路由规则和控制策略都是通过 Kubernetes CRD 实现，可以说 Istio 的 APIServer 就是 Kubernetes 的 APIServer，数据也自然地被存在了对应 Kubernetes 的 Etcd 中。如下图所示：

![img](https://static001.geekbang.org/infoq/ac/ac74f5674886ea8f9b7babfccf05a4fd.png)



1. **服务发现及健康检查**

Istio 的服务发现机制基于 Kubernetes 的 Services/Endpoints，从 Kube-apiserver 中获取 Service 和 Endpoint，然后将其转换成 Istio 服务模型的 Service 和 ServiceInstance。同时 Istio 不提供域名解析能力，域名访问机制也依赖于 kube-dns, coreDNS 等构建。节点健康检查能力基于 LivenessProbe/ReadinessProbe 机制实现 。

在实际场景中，TSF 的用户并非都是 Kubernetes 用户，例如公司内部的一个业务因历史遗留问题，不能完全容器化改造，同时存在 VM 和容器环境，场景如下：

![img](https://static001.geekbang.org/infoq/1a/1a239f98d61a56a1260ed2cda1dd927c.png)



从上面的业务场景可以看出，业务要求能够将其部署在自研 Paas 以及 Kubernetes 的容器、虚拟机以及裸金属的服务都可以通过 Service Mesh 进行相互访问。



为了实现多平台的部署，必须与 Kubernetes 进行解耦。在脱离 Kubernetes 后，Istio 面临以下四个问题：

- 服务的动态配置管理不可用
- 服务节点健康检查不可用
- 服务自动注册与反注册能力不可用
- 流量劫持不可用

针对这 4 个问题，TSF Mesh 团队对 Istio 的能力进行了扩展和增强，增强后的架构如下：



![img](https://static001.geekbang.org/infoq/a4/a410be682a4562103d5ad0c716265748.png)



增强 Pilot 的 consul 适配器，与 consul 注册中心对接；增加 Apiserver 实现元数据转换；增强 Pilot-agent，实现 VM 的自动注入，服务注册，envoy 管理。经过改造后，Service Mesh 成功与 Kubernetes 平台解耦，组网变得更加简洁，通过 GRPC 和 REST API 可以对数据面进行全方位的控制，可从容适配任何的底层部署环境，对于私有云客户可以提供更好的体验。



**2. 支持多租户**

租户的概念不止局限于集群的用户，它可以包含为一组计算，网络，存储等资源组成的工作负载集合。而在多租户场景中，需要对不同的租户提供尽可能的安全隔离，以最大程度的避免恶意租户对其他租户的攻击，同时需要保证租户之间公平地分配共享集群资源。

在公有云或私有云场景下，用户对隐私和隔离看得非常重要。往往不同用户/租户之间，服务配置、节点信息、控制信息等资源数据是隔离的，互相不可见。但是 Istio 本身并不支持这种级别的隔，需要框架集成者去扩展。

Istio 依托于 kubernetes 能力，可实现 “soft-multitenancy”，即单一 Kubernetes 控制平面和多个 Istio 控制平面以及多个服务网格相结合；每个租户都有自己的一个控制平面和一个服务网格。

其它租户模式，比如单独的 Istio 控制平面控制多集群网格的场景，Istio 并不支持。在这种场景下，每个租户一个网格，集群管理员控制和监控整个 Istio 控制面以及所有网格，租户管理员只能控制特定的网格。这种场景与云环境下的多租户概念比较稳合，对此 TSF Mesh 通过数据建模，实现了这种租户模式，即单控制面多集群网格。基本架构如下图所示：



![img](https://static001.geekbang.org/infoq/60/6008d4bcf54bf0ad3bc5a96c0f3208ed.png)



在上图中，实现了租户管理、租户数据的隔离存储、Pilot 控制面缓存增加租户索引。在这种场景下，各租户只能看到自身的集群资源，包括计算资源、逻辑资源、应用资源等，其它租户创建的集群资源不可见，sidecar 只能从控制端同步到本租户的配置和服务 xDS 信息。



**3. 服务寻址**

在侵入式框架下，目标服务的标识通常是服务名，服务注册与发现是强关联的，通过服务发现机制实现服务名到服务实例的寻址。在 Service Mesh 机制下，对应用是无侵入的，服务发现机制只能下沉到 Service Mesh，这意味着客户端通过目标服务标识名称的访问方式，需要域名解析能力的支持。

Istio 下的应用使用完全限定域名 FQDN（fully qualified domain name）进行相互调用，基于 FQDN 的寻址依赖 DNS 服务器，Istio 官方对 DNS 服务器的说明如下：

> *Istio does not provide a DNS. Applications can try to resolve the FQDN using the DNS service present in the underlying platform (kube-dns, mesos-dns, etc.).*

从上面的描述看出，Istio 并不提供 DNS 的能力，依托于平台的能力，如 kubernetes 平台下的 kube-dns 。以 Istio 的官方提供的 demo：bookinfo 为例，Reviews 与 Ratings 之间的完整的服务调用会经过以下过程：



![img](https://static001.geekbang.org/infoq/b5/b5e1291bfc7f0f52f4760edfe0cc643e.png)



从图上可以看出，Reviews 和 Ratings 的互通，kube-dns 主要实现 2 个功能：

- 服务的 DNS 请求被 kube-dns 接管
- kube-dns 将服务名解析成可被 iptables 接管的虚拟 IP（clusterIP）

正如前面提到的，用户的生产环境不一定包含 kubernetes 或者 kube-dns，我们需要另外寻找一种机制来实现上面的两个功能。

在 DNS 选型上，有集中式和分布式两种方案，集中式 DNS：代表有 kube-dns, CoreDNS 等，通过内置或者插件的方式，实现与服务注册中心进行数据同步。

集中式 DNS 存在以下问题：组网中额外增加一套 DNS 集群，并且一旦 DNS Server 集群不可用，所有数据面节点在 DNS 缓存失效后都无法工作，因此需要为 DNS Server 考虑高可用甚至容灾等一系列后续需求，会导致后期运维成本增加。

分布式 DNS 将服务 DNS 的能力下沉到数据平面。分布式 DNS 运行在数据面节点上，DNS 无单点故障，无需考虑集群容灾等问题，只需要有机制可以重新拉起即可。由于与业务进程运行在同一节点，因此其资源占用率必须控制得足够低，才不会对业务进程产生影响。

综合考虑，最终 TSF Mesh 选用了分布式 DNS 的方案，以独立进程作为 DNS Server，如下图所示



![img](https://static001.geekbang.org/infoq/8e/8ede3f50db45b8af82511ea81ff90e0e.png)



图中 Mesh-dns 通过 Pilot 同步服务信息，当应用通过服务名调用时，会进入 Mesh-dns 进行域名的本地解析，然后流量被 iptables 接管，之后到达 envoy，最后由 envoy 动态路由到 upstream；对于其它非 mesh 服务的域名解析，Mesh-dns 会透明传输，走默认的 DNS。通过配置缓存本地化以及异常退出后自动拉起并加载配置，保证在异常情况下的高可用。

值得一提的是 Istio 暴力流量接管问题，这个也是大家诟病比较多的。由于 Istio 的数据面针对 kubernetes 容器内流量进行全接管，但是对于虚拟机或裸金属场景可能不适用，毕竟虚拟机或裸金属上可能不仅仅只有 mesh 的服务。因此，需要考虑细粒度的接管方案，使得 mesh 与非 mesh 应用在同一个虚拟机/容器中可以共存。TSF Mesh 对这块能力也做了增强，只需要少量的 iptables 规则，即可完成 mesh 与非 mesh 流量的筛选。



**4. 与异构服务框架互通**

微服务框架可以分为侵入式和非侵入式两种，目前比较主流的微服务框架 Spring Cloud，基于 Spring Boot 开发，提供一套完整的微服务解决方案，包括服务注册与发现，配置中心，全链路监控，API 网关，熔断器，远程调用等开源组件，并且可以根据需求对部分组件进行扩展和替换。与 Service Mesh 之处不同在于，Spring Cloud 是一种侵入式的微服务框架，需要 SDK 支撑，并且技术栈受限于 Java。

出于功能重叠、语言壁垒 、耦合性，开发运维成本，技术门槛，云原生生态等多方面的因素，有相当一部分用户开始尝试 Service Mesh，或者往 Service Mesh 迁移和转型，但仍然存在一些遗留的 Spring Cloud 的服务，希望能与 Service Mesh 中的服务互通。用户期望支持的架构如下图所示：

![img](https://static001.geekbang.org/infoq/44/4442b4b2fa625ed31a88630578aa2fdd.png)



在上面这个架构中，最大的挑战在于涉及了两个不同的微服务框架之间的互通。这两个微服务框架从架构模式、概念模型、功能逻辑，技术栈上，都存在较大的差异。唯一相共的点，就是他们都是微服务框架，可以将应用的能力通过服务的形式提供出来，给外部用户调用，外部用户实际上并不感知服务的具体形态。

基于这个共同点，为了使得不同框架下的服务能够正常工作，TSF 团队做了大量的开发工作，将两个微服务框架，从部署模式、服务及功能模型上进行了拉通，主要包括如下几点：

| **对齐能力** | **说明 **                                                    |
| ------------ | ------------------------------------------------------------ |
| 服务模型     | 基于统一的服务元数据模型，针对 Service Mes和Spring Cloud 的服务注册发现机制进行拉通 |
| 服务 API     | 基于标准 API 模型（OpenAPI v3），针对两边框架的 API 级别服务治理能力进行拉通 |
| 服务路由     | 基于标准权重算法以及标签模型，针对 Service Mesh virtual-service 和 Spring Cloud ribbon 能力进行拉通。 |
| 限流         | 基于标准令牌桶架构和模型，以及条件匹配规则，对 mixer 及 spring cloud ratelimiter 能力进行拉通。 |
| 熔断器       | 增强 envoy 的能力，实现业界标准熔断器，支持服务级别、API级别和实例级别，与 Spring Cloud 拉通 |
| 鉴权         | 基于标签模型，支持条件匹配规则的黑白名单规则，与 Spring Cloud 拉通 |



**5. 可观测性**

在上一小节，提到了 Service Mesh 与 Spring Cloud 的能力互通，TSF Mesh 为了提供更好的用户体验，在日志、监控和调用链方面的能力也与 Spring Cloud 拉通，在 envoy 标准 Tracers 能力（envoy.zipkin）的基础上，增加了 envoy.local 类型，使其支持监控和调用链日志落到本地挂载盘，由 TSF 的 APM 系统采集并分析，实现 mesh 应用与 spring 应用的调用链串接。

如下图展示了两种不同微服务架构下，一致的服务依赖拓扑能力。user、shop、promotion 为 Service Mesh 应用，provider-demo 为 Spring Cloud 应用，服务间的箭头表示了调用关系。

![img](https://static001.geekbang.org/infoq/98/98087005d998d69a08c252975bd3d7d3.png)



## 总结与展望

TSF Mesh 作为腾讯微服务平台 TSF 的 Service Mesh 解决方案，在持续交付中，帮助企业客户解决传统集中式架构转型的困难，打造大规模高可用的分布式系统架构，实现业务、产品的快速落地。本文主要从用户实际场景出发，挑选了 TSF Mesh 产品化过程中遇到的部分典型问题和应对的解决方案，进行梳理和介绍，希望对 TSF Mesh 产品的了解以及技术演进思路有所帮助。还有一些问题和解决办法，涉及较深的技术细节，或显枯燥，并未一一罗列，比如性能优化相关，mixer 相关，自定义协议相关，部署相关等等。

TSF Mesh 团队拥抱开源协同，努力跟进 Service Mesh 的技术发展趋势，积极参与社区贡献。就技术发展趋势，有些点仍值得后续探讨，比如控制面单体化，UDPA（通用数据平面 API）的标准化演进，wasm 在 envoy 中扮演的角色，mixer 下沉，协议扩展，性能优化等等。

回顾过去，从 "Service Mesh" 和 "Istio" 这两个词汇第一次进入公众视野到如今，有将近四年的时间，见证了数据面板的争奇斗艳，也亲历了 xDS 的“快速”演变，架构与性能之间的妥协也从未停歇。总之，一句话：流年笑掷，未来可期。



------

头图：Unsplash

作者：吕晓明

原文：https://mp.weixin.qq.com/s/20UJMs4U5YEUfxV6dS3oJg

原文：腾讯云中间件团队在 Service Mesh 中的实践与探索

来源：腾讯云中间件 - 微信公众号 [ID：gh_6ea1bc2dd5fd]

转载：著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

2021 年 1 月 11 日 12:39951