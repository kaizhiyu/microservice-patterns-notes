# 从Spring Cloud到Kubernetes的微服务迁移实践

- UCloud 技术团队

**2020 年 4 月 09 日



## 写在前面

要出发周边游（以下简称要出发）是国内知名的主打「周边游」的在线旅行网站，以国内各大城市为中心，覆盖其周边旅游市场，提供包含酒店、门票、餐饮等在内的 1 – 3 天短途旅行套餐。为了降低公司内部各个业务模块的耦合度，提高开发、交付及运维效率，我们在 2017 年就基于 Spring Cloud 完成了公司内部业务微服务化的改造，并在 2019 年实现了 Spring Cloud 至 UCloud UK8S 平台的迁移。

本文从**要出发的业务架构、Prometheus JVM 监控、基于 HPA 的峰值弹性伸缩、基于 Elastic 的 APM 链路跟踪及 Istio 服务治理**等方面介绍了我们基于 UCloud UK8S 的 Spring Cloud 改造实践。



## Why K8S and Why UK8S

Spring Cloud 作为当下主流的微服务框架，在功能层面为服务治理定义了智能路由、熔断机制、服务注册与发现等一系列的标准，并提供了对应的库和组件来实现这些标准特性，对微服务周边环境提供了最大力度的支持。



![img](https://static001.infoq.cn/resource/image/a4/a5/a419ab76ea7e49b1aafdec12da538aa5.png)



改造前，Spring Cloud 的业务架构如下：服务发现部分采用了 Spring Cloud 的 Eureka 组件，熔断器组件采用了 Hystrix，服务网关使用了 Zuul 和 Spring Cloud Gateway（历史原因），分布式配置主要采用了 Spring Cloud Config（部分小组使用了 Apollo），并通过 Feign 实现了客户服务端的负载均衡。

但 Spring Cloud 也有一些不可避免的缺点，如基于不同框架的不同组件带来的高应用门槛及学习成本、代码级别对诸多组件进行控制的需求与微服务多语言协作的目标背道而驰。

在我们内部，由于历史原因，不同小组所使用的 API 网关架构不统一，且存在多套 Spring Cloud，给统一管理造成了不便；Spring Cloud 无法实现灰度发布，也给公司业务发布带来了一定不便。更重要的是，**作为一家周边游网站，我们经常会举行一些促销活动，面临在业务峰值期资源弹性扩缩容的需求**，仅仅依靠 Spring Cloud 也无法实现资源调度来满足业务自动扩缩容的需求。

在决定向 UK8S 转型时，我们也考虑过使用 Kubespray 自建 K8S 集群，并通过 Cloud Provider 实现 K8S 集群与云资源的对接，例如使用 Load Balance、Storage Class、Cluster Autoscaler（CA） 等，但在这种情况下，新增 Node 节点需要单独去部署安装 Cloud Provider，给运维工作带来了一定的复杂性。

UK8S 实现了与内部 UHost 云主机、ULB 负载均衡、UDisk 云盘等产品的无缝连接，我们能够在 UK8S 集群内部轻松创建、调用以上产品。在峰值弹性的场景下，也能够**通过 UK8S 内部的 CA 插件，实现 Node 级别的资源自动扩缩容，极大提升了运维效率**。通过其 CNI 插件，UK8S 与 UCloud 自身 VPC 网络相连接，**无需采用其他开源网络解决方案，降低了网络复杂度**；而 UK8S 原生无封装的特质，也**给了更大的改造空间，并且能够在出现故障时自己快速排查定位解决**。



## 整体业务架构

从 Spring Cloud 到 UK8S 的过程，也是内部服务模块再次梳理、统一的过程，在此过程中，我们对整体业务架构做了如下改动：

1.去掉原有的 Eureka，改用 Spring Cloud Kubernetes 项目下的 Discovery。Spring Cloud 官方推出的项目 Spring Cloud Kubernetes 提供了通用的接口来调用 Kubernetes 服务，让 Spring Cloud 和 Spring Boot 程序能够在 Kubernetes 环境中更好运行。在 Kubernetes 环境中，ETCD 已经拥有了服务发现所必要的信息，没有必要再使用 Eureka，通过 Discovery 就能够获取 Kubernetes ETCD 中注册的服务列表进行服务发现。

2.去掉 Feign 负载均衡，改用 Spring Cloud Kubernetes Ribbon。Ribbon 负载均衡模式有 Service / Pod 两种，在 Service 模式下，可以使用 Kubernetes 原生负载均衡，并通过 Istio 实现服务治理。

3.网关边缘化。网关作为原来的入口，全部去除需要对原有代码进行大规模的改造，我们把原有的网关作为微服务部署在 Kubernetes 内，并利用 Istio 来管理流量入口。同时，我们还去掉了熔断器和智能路由，整体基于 Istio 实现服务治理。

4.分布式配置 Config 统一为 Apollo。Apollo 能够集中管理应用在不同环境、不同集群的配置，修改后实时推送到应用端，并且具备规范的权限、流程治理等特性。

5.增加 Prometheus 监控，特别是对 JVM 一些参数和一些定义指标的监控，并基于监控指标实现了 HPA 弹性伸缩。



![img](https://static001.infoq.cn/resource/image/d5/ba/d5bae51fd0f999959d20080976e9d5ba.png)



Kubernetes 化后业务架构将控制平面和数据平面分开。Kubernetes Master 天然作为控制平面，实现整套业务的控制，不部署任何实际业务。数据平面中包含了基于 Java、PHP、Swoole、.NET Core 等不同语言或架构的项目。由于不同语言对机器性能有着不同要求，我们通过 Kubernetes 中节点 Label，将各个项目部署在不同配置的 Node 节点上，做到应用间互不干扰。



## 基于 Prometheus 的 JVM 监控

在 Spring Cloud 迁移到 Kubernetes 后，我们仍需要获取 JVM 的一系列底层参数，对服务的运行状态进行实时监控。Prometheus 是目前较为成熟的监控插件，而 Prometheus 也提供了 Spring Cloud 插件，可以获取到 JVM 的底层参数，进行实时监控。

我们设置了响应时间、请求数、JVM Memory、JVM Misc、Garbage Collection 等一系列详尽的参数，为问题解决、业务优化提供可靠的依据。

![img](https://static001.infoq.cn/resource/image/84/bd/84ef4b04a93ccca7ad96a7128d01d0bd.png)



![img](https://static001.infoq.cn/resource/image/cc/2f/cc462ecadc5ce78f6661f88a13ac912f.png)



## 基于 HPA 的峰值弹性伸缩

要出发作为一家周边游服务订购平台，**在业务过程中经常会涉及到景区、酒店门票抢购等需要峰值弹性的场景**。Kubernetes 的 HPA 功能为弹性伸缩场景提供了很好的实现方式。

在 Kubernetes 中，HPA 通常通过 Pod 的 CPU、内存利用率等实现，但在 Java 中，内存控制通过 JVM 实现，当内存占用过高时，JVM 会进行内存回收，但 JVM 并不会返回给主机或容器，单纯基于 Pod / CPU 指标进行集群的扩缩容并不合理。我们通过 Prometheus 获取 Java 中 http_server_requests_seconds_count（请求数）参数，通过适配器将其转化成 Kubernetes API Server 能识别的参数，并基于这一指标实时动态调整 Pod 的数量。

![img](https://static001.infoq.cn/resource/image/8e/f4/8ee6bc00e4432df5654fdb9c7d7adff4.png)



UK8S 产品也提供了自身的集群伸缩插件，通过设置伸缩组，并匹配相应的伸缩条件，**能够及时创建相应的云主机作为 Node 节点，方便我们在业务高峰时期更快速高效地拉起资源。**



## 基于 Elastic 的 APM 链路跟踪

微服务框架下，一次请求往往需要涉及到多个服务，因此服务性能监控和排查就变得复杂；不同服务可能由不同的团队开发，甚至使用不同的编程语言来实现；服务有可能部署在几千台服务器，横跨多个不同的数据中心。

因此，就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题。

目前市面有很多开源的 APM 组件，Zipkin、Pinpoint、Skywalking 等等。我们最终选择了基于 Elastic 开源的 apm-server。正是由于市面上有太多的监控开源项目，但是各项目之间无法很好的互通。 而 Elastic 通过 filebeat 收集业务日志，通过 metricbeat 监控应用服务性能，通过 apm-server 实现服务间的 tracing，并把数据统一存放在 es，很好的将 logging、metrics、tracing 整合到一起，打破了各项目之间的壁垒，能够更快速的协助运维及开发定位故障，保障系统的稳定性。



![img](https://static001.infoq.cn/resource/image/ed/02/edbebed7be1e134c66c73098b4070602.png)



## Istio 服务治理

基于应用程序安全性、可观察性、持续部署、弹性伸缩和性能、对开源工具的集成、开源控制平面的支持、方案成熟度等考虑，我们最终选择了 Istio 作为服务治理的方案，主要涉及以下几个部分：

1.Istio-gateway 网关：Ingress Gateway 在逻辑上相当于网格边缘的一个负载均衡器，用于接收和处理网格边缘出站和入站的网络连接，其中包含开放端口和 TLS 的配置等内容，实现集群内部南北流量的治理。

2.Mesh 网关：Istio 内部的虚拟 Gateway，代表网格内部的所有 Sidecar，实现所有网格内部服务之间的互相通信，即东西流量的治理。

3.流量管理：在去除掉 Spring Cloud 原有的熔断、智能路由等组件后，我们通过对 Kubernetes 集群内部一系列的配置和管理，实现了 http 流量管理的功能。包括使用 Pod 签对具体的服务进程进行分组（例如 V1/V2 版本应用）并实现流量调度，通过 Istio 内的 Destination Rule 单独定义服务负载均衡策略，根据来源服务、URL 进行重定向实现目标路由分流等，通过 MenQuota、RedisQuota 进行限流等。

4.遥测：通过 Prometheus 获取遥测数据，实现灰度项目成功率、东西南北流量区分、服务峰值流量、服务动态拓扑的监控。

![img](https://static001.infoq.cn/resource/image/9d/d1/9d1e4b4df164203854d9fa32038992d1.png)



![img](https://static001.infoq.cn/resource/image/a5/7b/a534368712ecdce4fd39010511db1c7b.png)



## 总结

目前我们已将旗下「云客赞」社交电商 App 全部迁移至 UK8S，开发语言包括 Java、PHP-FPM、NodeJS 等等。结合 CI/CD，能快速实现服务迭代以及新项目上线，大大提升了开发以及运维的工作效率；通过完善的日志、监控、链路跟踪及告警系统，能够快速的定位故障，并且根据遥测数据提前预判峰值，通过 HPA 实现服务自动伸缩，科学的分配资源，大大降低了计算资源成本；通过 Istio 服务治理，很好的实现了流量的管理，并且基于此轻松的实现了灰度发布。

接下来，我们将更加丰富 CI/CD 流水线，加入单元测试、代码扫描、性能测试等提升测试效率；引入 chatops 丰富运维手段；借助 Istio 实现多云管理进一步保障业务的稳定性。



## 作者介绍

王琼，「要出发周边游」运维架构师兼运维经理，负责公司云原生落地和企业容器化改造。2016 年开始接触 K8S，在 K8S 以及 Service Mesh 领域持续深耕，致力于搭建生产级可用的容器服务平台。

2020 年 4 月 09 日 16:57726



原文：https://www.infoq.cn/article/5Hdy3Aur1egKToCwmRfo?utm_source=related_read_bottom&utm_medium=article