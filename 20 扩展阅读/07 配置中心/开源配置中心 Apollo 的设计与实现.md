# 开源配置中心 Apollo 的设计与实现



- 宋顺



**2017 年 9 月 19 日

**[语言 & 开发](https://www.infoq.cn/topic/development)[架构](https://www.infoq.cn/topic/architecture)



通过一个开源配置中心的设计思路与实现细节，让咱们来了解配置与配置中心会涉及到一些什么知识点。本文来自宋顺在“携程技术沙龙——海量互联网基础架构”上的分享。

## What is Apollo

随着程序功能的日益复杂，程序的配置日益增多：各种功能的开关、参数的配置、服务器的地址……

对程序配置的期望值也越来越高：配置修改后实时生效，灰度发布，分环境、分集群管理配置，完善的权限、审核机制……

在这样的大环境下，传统的通过配置文件、数据库等方式已经越来越无法满足开发人员对配置管理的需求。Apollo 配置中心应运而生！

### Apollo 简介

Apollo（阿波罗）是携程框架部门研发的 **开源配置管理中心**，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性。

Apollo 支持 4 个维度管理 Key-Value 格式的配置：

- application (应用)
- environment (环境)
- cluster (集群)
- namespace (命名空间)

同时，Apollo 基于开源模式开发，[开源地址](https://github.com/ctripcorp/apollo)。

### 什么是配置？

既然 Apollo 定位于配置中心，那么在这里有必要先简单介绍一下什么是配置。

按照我们的理解，配置有以下几个属性：

**配置是独立于程序的只读变量**

1. 配置首先是独立于程序的，同一份程序在不同的配置下会有不同的行为
2. 其次，配置对于程序是只读的，程序通过读取配置来改变自己的行为，但是程序不应该去改变配置
3. 常见的配置有：DB Connection Str、Thread Pool Size、Buffer Size、Request Timeout、Feature Switch、Server Urls 等

**配置伴随应用的整个生命周期**

配置贯穿于应用的整个生命周期，应用在启动时通过读取配置来初始化，在运行时根据配置调整行为

**配置可以有多种加载方式**

配置也有很多种加载方式，常见的有程序内部 hard code，配置文件，环境变量，启动参数，基于数据库等

**配置需要治理**

1、权限控制

由于配置能改变程序的行为，不正确的配置甚至能引起灾难，所以对配置的修改必须有比较完善的权限控制

2、不同环境、集群配置管理

同一份程序在不同的环境（开发，测试，生产）、不同的集群（如不同的数据中心）经常需要有不同的配置，所以需要有完善的环境、集群配置管理

3、框架类组件配置管理

有一类比较特殊的配置——框架类组件配置，比如 CAT 客户端的配置。

虽然这类框架类组件是由其他团队开发、维护，但是运行时是在业务实际应用内的，所以本质上可以认为框架类组件也是应用的一部分。这类组件对应的配置也需要有比较完善的管理方式

## Why Apollo

正是基于配置的特殊性，所以 Apollo 从设计之初就立志于成为一个有治理能力的配置管理平台，目前提供了以下的特性：

统一管理不同环境、不同集群的配置

1. Apollo 提供了一个统一界面集中式管理不同环境（environment）、不同集群（cluster）、不同命名空间（namespace）的配置
2. 同一份代码部署在不同的集群，可以有不同的配置，比如 ZooKeeper 的地址等
3. 通过命名空间（namespace）可以很方便地支持多个不同应用共享同一份配置，同时还允许应用对共享的配置进行覆盖

配置修改实时生效（热发布）

用户在 Apollo 修改完配置并发布后，客户端能实时（1 秒）接收到最新的配置，并通知到应用程序（客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送）

版本发布管理

所有的配置发布都有版本概念，从而可以方便地支持配置的回滚

灰度发布

支持配置的灰度发布，比如点了发布后，只对部分应用实例生效，等观察一段时间没问题后再推给所有应用实例

权限管理、发布审核、操作审计

1、应用和配置的管理都有完善的权限管理机制，对配置的管理还分为了编辑和发布两个环节，从而减少人为的错误

2、所有的操作都有审计日志，可以方便地追踪问题

客户端配置信息监控

可以在界面上方便地看到配置在被哪些实例使用

提供 Java 和.Net 原生客户端

1. 提供了 Java 和.Net 的原生客户端，方便应用集成
2. 支持 Spring Placeholder, Annotation 和 Spring Boot 的 ConfigurationProperties，方便应用使用（需要 Spring 3.1.1+）
3. 同时提供了 Http 接口，非 Java 和.Net 应用也可以方便地使用

提供开放平台 API

Apollo 自身提供了比较完善的统一配置管理界面，支持多环境、多数据中心配置管理、权限、流程治理等特性。不过 Apollo 出于通用性考虑，不会对配置的修改做过多限制，只要符合基本的格式就能保存，不会针对不同的配置值进行针对性的校验，如数据库用户名、密码，Redis 服务地址等。

对于这类应用配置，Apollo 支持应用方通过开放平台 API 在 Apollo 进行配置的修改和发布，并且具备完善的授权和权限控制。

比如 Redis 服务器地址由 Redis 治理系统经过校验后通过开放平台 API 配置到 Apollo，进而下发到所有使用 Redis 的应用程序。

当 Redis 治理系统发现某个机房的 Redis 全部发生故障时，可以通过 Apollo 开放平台 API 把另一个机房的 Redis 服务器地址实时下发到应用程序，从而实现 Redis 故障的秒级切换（Redis 跨机房同步可以参考携程开源 Redis 多数据中心解决方案 XPipe）。

部署简单

1. 配置中心作为基础服务，可用性要求非常高，这就要求 Apollo 对外部依赖尽可能地少
2. 目前唯一的外部依赖是 MySQL，所以部署非常简单，只要安装好 Java 和 MySQL 就可以让 Apollo 跑起来
3. Apollo 还提供了打包脚本，一键就可以生成所有需要的安装包，并且支持自定义运行时参数

## Apollo at a glance

### 基础模型

如下即是 Apollo 的基础模型：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/76/56/760f44154fd180fb3b18c002d6990e56.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/1.jpg)

- 用户在配置中心对配置进行修改并发布
- 配置中心通知 Apollo 客户端有配置更新
- Apollo 客户端从配置中心拉取最新的配置、更新本地配置并通知到应用

### 界面概览

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/22/06/2241f37867041e7ac3754b95ee7c2606.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/2.jpg)

上图是 Apollo 配置中心中一个项目的配置首页

- 在页面左上方的环境列表模块展示了所有的环境和集群，用户可以随时切换
- 页面中央展示了两个 namespace(application 和 FX.apollo) 的配置信息，默认按照表格模式展示、编辑。用户也可以切换到文本模式，以文件形式查看、编辑
- 页面上可以方便地进行发布、回滚、灰度、授权、查看更改历史和发布历史等操作

### 添加 / 修改配置项

用户可以通过配置中心界面方便的添加 / 修改配置项：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/17/ca/17200e042ca32309f96049ee15b352ca.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/3.jpg)

输入配置信息：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/f1/33/f1269365211689dcbb23e47a6d10a933.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/4.jpg)

### 发布配置

通过配置中心发布配置：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/a7/87/a7f5aca6583d8d41a86757dc3383ac87.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/5.jpg)

填写发布信息：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/20/f3/20d7f2c72554e70219d1c5dc06a825f3.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/6.jpg)

### 客户端获取配置（Java API 样例）

配置发布后，就能在客户端获取到了，以 Java API 方式为例，获取配置的示例代码如下：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/91/e8/91d87ddc25c1b27314c73a865a8cb4e8.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/7.jpg)

### 客户端监听配置变化（Java API 样例）

通过上述获取配置代码，应用就能实时获取到最新的配置了。不过在某些场景下，应用还需要在配置变化时获得通知，比如数据库连接的切换等，所以 Apollo 还提供了监听配置变化的功能，Java 示例如下：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/51/d8/5111825da328d6fa62b0d71baf14ded8.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/8.jpg)

### Spring 集成样例

Apollo 和 Spring 也可以很方便地集成，只需要标注 @EnableApolloConfig 后就可以通过 @Value 获取配置信息：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/7f/53/7f4f922a229eb86acfcf3b1ea39da853.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/9.jpg)

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/06/40/06051674d37ec23c0ac1ab84aec68f40.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/10.jpg)

## Apollo in depth

通过上面的介绍，相信大家已经对 Apollo 有了一个初步的了解，接下来我们深入了解一下 Apollo 的核心概念和背后的设计。

### 关键概念

## application (应用)

这个很好理解，就是实际使用配置的应用，Apollo 客户端在运行时需要知道当前应用是谁，从而可以去获取对应的配置。

每个应用都需要有唯一的身份标识——appId，我们认为应用身份是跟着代码走的，所以需要在代码中配置：

- Java 客户端通过 classpath:/META-INF/app.properties 来指定 appId
- .Net 客户端通过 app.config 来指定 appId

## environment (环境)

配置对应的环境，Apollo 客户端在运行时需要知道当前应用处于哪个环境，从而可以去获取应用的配置。

我们认为环境和代码无关，同一份代码部署在不同的环境就应该能够获取到不同环境的配置。所以环境默认是通过读取机器上的配置（server.properties 中的 env 属性）指定的。

不过为了开发方便，我们也支持运行时通过 System Property 等指定，server.properties 文件路径如下：

- Windows: C:\opt\settings\server.properties
- Linux/Mac: /opt/settings/server.properties
- cluster (集群)

一个应用下不同实例的分组，比如典型的可以按照数据中心分，把上海机房的应用实例分为一个集群，把北京机房的应用实例分为另一个集群。对不同的 cluster，同一个配置可以有不一样的值，如 ZooKeeper 地址。

集群默认是通过读取机器上的配置（server.properties 中的 idc 属性）指定的，不过也支持运行时通过 System Property 指定。

## namespace (命名空间)

一个应用下不同配置的分组，可以简单地把 namespace 类比为文件，不同类型的配置存放在不同的文件中，如数据库配置文件，RPC 配置文件，应用自身的配置文件等。

应用可以直接读取到公共组件的配置 namespace，如 DAL，RPC 等。应用也可以通过继承公共组件的配置 namespace 来对公共组件的配置做调整，如 DAL 的初始数据库连接数。

### 总体设计

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/f3/8c/f3420d8539b93eee3c13f6eb10dc9a8c.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/11.jpg)

上图简要描述了 Apollo 的总体设计，我们可以从下往上看：

- Config Service 提供配置的读取、推送等功能，服务对象是 Apollo 客户端
- Admin Service 提供配置的修改、发布等功能，服务对象是 Apollo Portal（管理界面）
- Config Service 和 Admin Service 都是多实例、无状态部署，所以需要将自己注册到 Eureka 中并保持心跳
- 在 Eureka 之上我们架了一层 Meta Server 用于封装 Eureka 的服务发现接口
- Client 通过域名访问 Meta Server 获取 Config Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Client 侧会做 load balance、错误重试
- Portal 通过域名访问 Meta Server 获取 Admin Service 服务列表（IP+Port），而后直接通过 IP+Port 访问服务，同时在 Portal 侧会做 load balance、错误重试
- 为了简化部署，我们实际上会把 Config Service、Eureka 和 Meta Server 三个逻辑角色部署在同一个 JVM 进程中

## 为什么选择 Eureka

为什么我们采用 Eureka 作为服务注册中心，而不是使用传统的 zk、etcd 呢？我大致总结了一下，有以下几方面的原因：

1. 它提供了完整的 Service Registry 和 Service Discovery 实现首先是提供了完整的实现，并且也经受住了 Netflix 自己的生产环境考验，相对使用起来会比较省心。
2. 和 Spring Cloud 无缝集成 我们的项目本身就使用了 Spring Cloud 和 Spring Boot，同时 Spring Cloud 还有一套非常完善的开源代码来整合 Eureka，所以使用起来非常方便。

另外，Eureka 还支持在我们应用自身的容器中启动，也就是说我们的应用启动完之后，既充当了 Eureka 的角色，同时也是服务的提供者。这样就极大的提高了服务的可用性。

这一点是我们选择 Eureka 而不是 zk、etcd 等的主要原因，为了提高配置中心的可用性和降低部署复杂度，我们需要尽可能地减少外部依赖。
\3. 开源

最后一点是开源，由于代码是开源的，所以非常便于我们了解它的实现原理和排查问题。

### 客户端设计

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/c8/db/c82ff0dc296996a7a0662114ccf21ddb.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/12.jpg)

上图简要描述了 Apollo 客户端的实现原理：

1、客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。

2、客户端还会定时从 Apollo 配置中心服务端拉取应用的最新配置。

这是一个 fallback 机制，为了防止推送机制失效导致配置不更新。

客户端定时拉取会上报本地版本，所以一般情况下，对于定时拉取的操作，服务端都会返回 304 - Not Modified。

定时频率默认为每 5 分钟拉取一次，客户端也可以通过在运行时指定 System Property: apollo.refreshInterval 来覆盖，单位为分钟。

3、客户端从 Apollo 配置中心服务端获取到应用的最新配置后，会保存在内存中

4、客户端会把从服务端获取到的配置在本地文件系统缓存一份

在遇到服务不可用，或网络不通的时候，依然能从本地恢复配置

5、应用程序从 Apollo 客户端获取最新的配置、订阅配置更新通知

## 配置更新推送实现

前面提到了 Apollo 客户端和服务端保持了一个长连接，从而能第一时间获得配置更新的推送。

长连接实际上我们是通过 Http Long Polling 实现的，具体而言：

1、客户端发起一个 Http 请求到服务端

2、服务端会保持住这个连接 30 秒

- 如果在 30 秒内有客户端关心的配置变化，被保持住的客户端请求会立即返回，并告知客户端有配置变化的 namespace 信息，客户端会据此拉取对应 namespace 的最新配置
- 如果在 30 秒内没有客户端关心的配置变化，那么会返回 Http 状态码 304 给客户端

3、客户端在收到服务端请求后会立即重新发起连接，回到第一步考虑到会有数万客户端向服务端发起长连，在服务端我们使用了 async servlet(Spring DeferredResult) 来服务 Http Long Polling 请求。

### 可用性考虑

配置中心作为基础服务，可用性要求非常高，下面的表格描述了不同场景下 Apollo 的可用性：

(点击放大图像)

[![img](https://static001.infoq.cn/resource/image/2a/51/2ac37724eaad0ed2dc172b32d4f64951.jpg)](https://www.infoq.cn/mag4media/repositories/fs/articles//zh/resources/13.jpg)

## Contribute to Apollo

Apollo 从开发之初就是以开源模式开发的，所以也非常欢迎有兴趣、有余力的朋友一起加入进来。

服务端开发使用的是 Java，基于 Spring Cloud 和 Spring Boot 框架。客户端目前提供了 Java 和.Net 两种实现。

[Github 地址](https://github.com/ctripcorp/apollo)。

欢迎大家发起 Pull Request！

## 作者介绍

**宋顺**，携程框架研发部技术专家。2016 年初加入携程，主要负责中间件产品的相关研发工作。毕业于复旦大学软件工程系，曾就职于大众点评，担任后台系统技术负责人。

------

感谢[雨多田光](http://www.infoq.com/cn/author/雨多田光)对本文的审校。

给InfoQ 中文站投稿或者参与内容翻译工作，请邮件至[ editors@cn.infoq.com ](mailto:editors@cn.infoq.com)。也欢迎大家通过新浪微博（[ @InfoQ ](http://www.weibo.com/infoqchina)，[ @丁晓昀](http://weibo.com/u/1451714913)），微信（微信号：[ InfoQChina ](http://www.geekbang.org/ivtw)）关注我们。

2017 年 9 月 19 日 17:4931838

文章版权归极客邦科技InfoQ所有，未经许可不得转载。



原文：https://www.infoq.cn/article/open-source-configuration-center-apollo