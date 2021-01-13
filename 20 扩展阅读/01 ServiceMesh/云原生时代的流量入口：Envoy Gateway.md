# 云原生时代的流量入口：Envoy Gateway

**2020 年 7 月 29 日

流量入口代理作为互联网系统的门户组件，具备众多选型：从老牌代理 HAProxy、Nginx，到微服务 API 网关 Kong、Zuul，再到容器化 Ingress 规范与实现，不同选型间功能、性能、可扩展性、适用场景参差不齐。当云原生时代大浪袭来，Envoy 这一 CNCF 毕业数据面组件为更多人所知。那么，优秀“毕业生”Envoy 能否成为云原生时代下流量入口标准组件？



## 背景 —— 流量入口的众多选型与场景

在互联网体系下，凡是需要对外暴露的系统几乎都需要网络代理：较早出现的 HAProxy、Nginx 至今仍在流行；进入微服务时代后，功能更丰富、管控能力更强的 API 网关又成为流量入口必备组件；在进入容器时代后，Kubernetes Ingress 作为容器集群的入口，是容器时代微服务的流量入口代理标准。关于这三类典型的七层代理，核心能力对比如下：

![img](https://static001.infoq.cn/resource/image/3a/bf/3a007dea7e38c1cc0bec2ec55f43d4bf.jpg)

从上述核心能力对比来看：

- HAProxy&Nginx在具备基础路由功能基础上，性能、稳定性经历多年考验。Nginx的下游社区OpenResty提供了完善的Lua扩展能力，使得Nginx可以更广泛的应用与扩展，如API网关Kong即是基于Nginx+OpenResty实现。
- API网关作为微服务对外API流量暴露的基础组件，提供比较丰富的功能和动态管控能力。
- Ingress作为Kubernetes入口流量的标准规范，具体能力视实现方式而定。如基于Nginx的Ingress实现能力更接近于Nginx，Istio Ingress Gateway基于Envoy+Istio控制面实现，功能上更加丰富（本质上Istio Ingress Gateway能力上强于通常的Ingress实现，但未按照Ingress规范实现）。

那么问题来了：同样是流量入口，在云原生技术趋势下，能否找到一个能力全面的技术方案，让流量入口标准化？



## Envoy 核心能力介绍

[Envoy](https://github.com/envoyproxy/envoy)是一个为云原生应用设计的开源边缘与服务代理（ENVOY IS AN OPEN SOURCE EDGE AND SERVICE PROXY, DESIGNED FOR CLOUD-NATIVE APPLICATIONS，@envoyproxy.io），是云原生计算基金会（CNCF）第三个毕业的项目，GitHub 目前有 13k+ Star。

Envoy 有以下主要特性：

- 基于现代C++开发的L4/L7高性能代理。
- 透明代理。
- 流量管理。支持路由、流量复制、分流等功能。
- 治理特性。支持健康检查、熔断、限流、超时、重试、故障注入。
- 多协议支持。支持HTTP/1.1，HTTP/2，GRPC，WebSocket等协议代理与治理。
- 负载均衡。加权轮询、加权最少请求、Ring hash、Maglev、随机等算法支持。支持区域感知路由、故障转移等特性。
- 动态配置API。提供健壮的管控代理行为的接口，实现Envoy动态配置热更新。
- 可观察性设计。提供七层流量高可观察性，原生支持分布式追踪。
- 支持热重启。可实现Envoy的无缝升级。
- 自定义插件
- 能力。Lua与多语言扩展沙箱WebAssembly。

总体来说，Envoy 是一个功能与性能都非常优秀的“双优生”。在实际业务流量入口代理场景下，Envoy 具备先天优势，可以作为云原生技术趋势流量入口的标准技术方案：



### 1. 较 HAProxy、Nginx 更丰富的功能

相较于 HAProxy、Nginx 提供流量代理所需的基本功能（更多高级功能通常需要通过扩展插件方式实现），Envoy 本身基于 C++已经实现了相当多代理所需高级功能，如高级负载均衡、熔断、限流、故障注入、流量复制、可观测性等。更为丰富的功能不仅让 Envoy 天生就可以用于多种场景，原生 C++的实现相较经过扩展的实现方式性能优势更为明显。



### 2. 与 Nginx 相当，远高于传统 API 网关的性能

在性能方面，Envoy 与 Nginx 在常用协议代理（如 HTTP）上性能相当。与传统 API 网关相比，性能优势明显。如下为 Envoy 与几种业务常用的 API 网关选型在 8 核物理机容器运行环境下，简单路由代理性能对比数据：（网易内部环境实测数据，仅供参考）

**Envoy Gateway：**



![img](https://static001.infoq.cn/resource/image/21/9d/214465f9d568a02f393d9dcfd8fd2a9d.png)



**基于 Java 的异步化 API 网关：**

![img](https://static001.infoq.cn/resource/image/50/80/5095fcb5f9c2434d3a42226a8bb0ff80.png)



**Kong：**

![img](https://static001.infoq.cn/resource/image/87/23/8728870eaeb7df99yyd09448a4c5ef23.png)



**APISIX：**

![img](https://static001.infoq.cn/resource/image/26/cb/2681bb49065b06a783d0b8ca687ffecb.png)

从以上性能数据可以看出，相同条件下：

- Envoy的TPS可以达到12W左右；
- 基于Java的异步化API网关最高可到2.8W左右；
- 基于Nginx的Kong，TPS可以到5W左右；
- 基于Nginx并相较Kong有一定优化的APISIX可以到9W左右。



可以看出：

- 简单路由代理场景下，Envoy性能优势已经比较明显；
- 复杂路由与治理功能场景下，Envoy原生C++实现功能的性能较通过Java Filter、OpenResty等扩展相比，优势会更加明显。



### 3. 动态管控能力强，具备数据面标准 xDS 协议

Envoy 具备静态配置与动态 API 两种配置模式。动态 API 方式灵活性更强，作为 Envoy 推荐的配置方式。Envoy 以 xDS（x Discovery Service）作为动态 API 的标准协议，其中包括了多种维度资源的发现协议：

- LDS(Listener Discovery Service)
- RDS(Route Discovery Service)
- CDS(Cluster Discovery Service)
- EDS(Endpoint Discovery Service)
- ADS(Aggregated Discovery Service)

xDS 的多种协议覆盖了包括监听器（Listener）、路由（Route）、后端集群（Cluster）、后端实例（Endpoint），可以通过协议对代理所需所有配置进行动态管控。其中 ADS 不是一个实际意义上的 xDS，它提供了一个汇聚的功能，以实现需要多个同步 xDS 访问的时候可以在一个 stream 中完成的作用。

由于 Envoy 已经以 CNCF 毕业项目的姿态成为了云原生数据面的事实标准组件，xDS 也相应成为云原生数据面事实动态 API 标准。这里不对 xDS 协议进行深入的介绍，感兴趣的同学可以通过社区与博客深入了解。



### 4. 天然亲和容器环境

Envoy 作为云原生社区的数据面标准组件，其本身并没有直接与 Kubernetes 或容器耦合。通过 xDS 协议的对接，Istio Pilot 等容器亲和的控制面组件可以将服务、实例、路由等配置信息推送至 Envoy。即使是在容器环境，Envoy 也很快能实现服务发现，即实现容器环境服务的代理和治理。所以，Envoy 天然亲和容器环境，可以作为容器环境 API 网关和 Ingress 的数据面选型。



### 5. 多语言扩展沙箱——WASM

WASM，即 WebAssembly，是由主流浏览器厂商组成的 W3C 社区团体制定的一个新的规范，首先看下来自 Mozilla 的官方定义：

WebAssembly 是一种新的编码方式，可以在现代的网络浏览器中运行

> 它是一种低级的类汇编语言，具有紧凑的二进制格式，可以接近原生的性能运行，并为诸如C/C ++等语言提供一个编译目标，以便它们可以在 Web上运行。它也被设计为可以与JavaScript共存，允许两者一起工作。

使用 WebAssembly 扩展 Envoy 的好处是：

- 避免修改 Envoy；
- 避免网络远程调用（check & report）；
- 通过动态装载（重载）来避免重启 Envoy；
- 隔离性；
- 实时A/B测试；

WASM 为 Envoy 带来了使用多语言扩展数据面的能力，即可以不局限地通过 C++、Java、Lua、JS 等语言进行数据面扩展，这对于流量代理数据面将是一个巨大的彩蛋。该特性已在 2020 年正式发布落地。相信在不远的未来，开发者使用自己最为擅长的语言进行流量入口能力扩展不再是梦想！



## Envoy Gateway 设计解析

基于云原生时代的技术趋势以及 Envoy 的功能、性能的“双优”特性，网易轻舟云原生团队提出基于 Envoy 实现标准的流量入口设计，并基于此进行了大规模业务生产落地。



### 逻辑架构

逻辑架构设计如下图所示：

![img](https://static001.infoq.cn/resource/image/f2/e0/f26d8be9850f74c4096b7e5d41b210e0.png)



整体架构主要包括数据面、控制面两部分，实现方面则是扩展了 Istio ingressgateway：

- 数据面

以 Envoy 作为数据面组件，负责南北向数据流量的代理、路由、治理、遥测等；通过 filterchain 进行扩展，目前支持基于 Lua、C++语言编写插件，WASM 落地后支持多语言方式扩展；通过 xDS 与控制面组件进行配置下发等动态控制。

- 控制面

以 Istio Pilot 作为基本控制面组件，同时提供 API 层、控制台供用户或第三方平台接入。

Istio Pilot 在这里主要包括如下作用：

1、作为 xDS Server。与 Envoy 的 xDS Client 建立 GRPC 通信连接，是与 Envoy 交互的基础控制面。可支持控制不同场景下 Envoy(ServiceMesh Sidecar & API 网关 Gateway & Ingress & 入口七层代理)。

2、对接注册中心（Discovery）。支持对接 Kubernetes API Server（K8S 注册中心能力），具备扩展 Consul、Eureka 等其他注册中心能力。支持同时对接多种注册中心，并将注册信息经过配置转换，下发至 Envoy。

3、配置处理（Config）。对于要下发至 Envoy 的所有配置，均通过 Pilot 进行配置处理，再下发至 Envoy。

4、模型抽象与接口封装(Mesh Configuration Protocol & Service Mesh Interface，简称 MCP & SMI)。Pilot 对 Envoy 的基础模型进行了抽象，并进行基础接口的封装，使得其他平台对 Envoy 的控制可以更加清晰与优雅。对外暴露的接口包括 GRPC 与 Kubernetes CRD 两种形式。

API Plane 作为 API 平面，主要⽤于对 K8S CRD、MCP 或 SMI 接口做面向业务功能的 API 组合、转换处理。

管理控制台是控制面最上层组件，也是研发、运维人员直接操作的平台组件。



### 场景地图

在 Envoy Gateway 技术栈基础上，可以适应入口七层代理、API 网关、Ingress、单元化机房路由、FaaS 函数路由等多种场景落地应用。



![img](https://static001.infoq.cn/resource/image/05/5a/0538d1548c21abb3e2a48ce1a229e35a.jpg)



1、入口七层代理

![img](https://static001.infoq.cn/resource/image/94/94/9420b76bb4cbd392f856a0fb0a8a9994.png)



在流量入口场景下，可以承担 HAProxy、Nginx 等传统代理职责。



2、API 网关

![img](https://static001.infoq.cn/resource/image/17/16/17db7dd2eb1c957b3ec3ccf7c6e3b416.png)



在微服务场景下，承担分解流量、路由、治理、审计等丰富职责。



3、Ingress

![img](https://static001.infoq.cn/resource/image/b2/52/b2a1151221051f0ae2324651b7ffa052.png)



在容器化场景下，可以作为能力更丰富的 Ingress 实现。



4、通用网关

![img](https://static001.infoq.cn/resource/image/16/4c/1696c84edf34dd057c4206aa9229c64c.png)



在完整使用 Envoy Gateway 方案下，可以省去原有入口七层代理 -> API 网关/Ingress 的链路，由一层通用网关（Envoy Gateway）完成。



### 落地路线

目前 Envoy 以两类角色在业界落地：一是作为 Service Mesh 数据面组件选型，目前在 Istio 等多种服务网格框架落地；二是作为流量入口代理，目前较多的是以 API 网关形式实现，如 Gloo、Ambassador 等。网易内部多个业务团队已经实现了数据面技术栈从 Java 异步化代理、Kong 到 Envoy 的升级，基于网易轻舟 Envoy Gateway 作为流量入口标准组件。

由于 Envoy Gateway 与 Service Mesh 使用了相同的 Envoy+Istio 技术栈，使得不论 Envoy 作为 Gateway，或 Service Mesh 中的 Sidecar，都可以被统⼀控制面管控，在提升了该技术栈整体可维护性基础上，还可以帮助业务逐步演进到 Service Mesh 架构：如果业务对 Service Mesh 架构尚存疑虑，可以先落地 Envoy Gateway，后续待时机成熟再引入 Service Mesh 架构，完成整体微服务技术架构的演进与统一。



## 网易轻舟 Envoy Gateway 大规模落地

基于以上背景与设计，网易轻舟云原生团队实现了 Envoy Gateway，并且已经在网易多个核心业务大规模落地：

- 网易传媒（新闻）已经实现全站流量通过轻舟Envoy Gateway暴露
- 网易严选已经实现上云服务全部流量通过轻舟Envoy Gateway暴露
- 网易有道、云信、Lofter等网易核心互联网业务流量通过轻舟Envoy Gateway暴露

如下为网易轻舟 Envoy Gateway 控制台部分线上业务管理视图：

![img](https://static001.infoq.cn/resource/image/0f/fd/0fc12d22008637043fbdccd0167cbdfd.jpg)



![img](https://static001.infoq.cn/resource/image/30/a6/30369698f90d7c03f6e59678a36129a6.jpg)



## 写在最后

Envoy Gateway 能否真正意义成为云原生时代流量入口标准目前尚不得知，但在业务落地过程中，我们已经看到了这套技术栈为业务所能带来的架构、性能、治理、可观测性等方面的红利。在越来越多业务接入 Envoy Gateway、Service Mesh 过程中，相关的工程化平台建设、排障工具等也在不断完善，我们拭目以待。



**参考链接**

Envoy 官方文档：https://www.envoyproxy.io/docs

Istio 官方文档：https://istio.io/docs

Envoy 实践：https://www.jianshu.com/p/90f9ee98ce70

WebAssembly 中文官方文档：[https://www.wasm.com.cn](https://www.wasm.com.cn/)

Service Mesh 发展趋势(续)：棋到中盘路往何方：https://msd.misuland.com/pd/3545776840385758572_



**作者简介：**

裴斐，网易杭州研究院轻舟云原生技术专家、架构师。10 年企业级平台架构和开发经验，目前主要负责网易轻舟微服务治理团队，专注于企业微服务架构及云原生技术的研究与落地工作。带领团队完成网易轻舟 Service Mesh、微服务框架 NSF、API 网关等多个项目在网易集团落地及商业化产品输出。



2020 年 7 月 29 日 11:443359

文章版权归极客邦科技InfoQ所有，未经许可不得转载。



原文：https://www.infoq.cn/article/SF5sl4IlUtUxuED3Musl?utm_source=related_read_bottom&utm_medium=article