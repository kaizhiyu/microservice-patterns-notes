# 京东数科统一接入网关 JDDLB 性能优化之 QAT 加速卡

- 京东数科

**2021 年 1 月 11 日

**[语言 & 开发](https://www.infoq.cn/topic/development)[编程语言](https://www.infoq.cn/topic/programing-languages)[其他](https://www.infoq.cn/topic/others)



> 京东数科JDDLB作为京东数科最重要的公网流量入口，承接了很多重要业务的公网流量。目前，已成功接替商业设备F5所承载的流量，并在数次618、11.11大促中体现出优越的功能、性能优势。



## **一、京东数科 JDDLB 整体架构**



<img src="https://static001.infoq.cn/resource/image/a9/95/a9844eca67d4f3b9d3c09aa5fcac9a95.png" alt="img" style="zoom:50%;" />



图 1 京东数科 JDDLB 整体架构



JDDLB 整体架构的核心包括：基于 DPDK 自主研发的四层负载 SLB，定制开发扩展功能的 NGINX，以及统一管控运维平台。其主要特点为：

- **高性能**：具备千万级并发和百万级新建能力。
- **高可用**：通过 ECMP、会话同步、健康检查等，提供由负载本身至业务服务器多层次的高可用。
- **可拓展**：支持四层/七层负载集群、业务服务器的横向弹性伸缩、灰度发布。
- **四层负载能力**：通过ospf 向交换机宣告vip；支持ECMP、session 同步；支持均衡算法如轮询、加权轮询、加权最小连接数、优先级、一致性哈希；FullNAT转发模式方便部署等。
- **七层负载能力**：支持基于域名和URL的转发规则配置；支持均衡算法如轮询、基于源 IP 哈希、基于cookie等。
- **SSL/TLS能力**：证书、私钥、握手策略的管理配置；支持 SNI 配置；支持基于多种加速卡的SSL卸载硬件加速等。
- **流量防控**：提供一定的 Syn-Flood 防护能力；与应用防火墙结合后提供 WAF 防护能力；提供网络流量控制手段如 Qos 流控、ACL 访问控制等。
- **管控平台**：支持多种维度的网络和业务指标监控和告警。



此外，借助于 JDDLB 现有能力特性，方便扩展其他新功能。例如，借助于 NGINX 的 SSL/TLS 硬件优化性能以及连接高并发处理能力，可以实现基于 MQTT 协议的、支持 SSL/TLS 协议、安全的推送长连接网关等。

本文针对 JDDLB 中七层负载的 SSL/TLS 性能优化方法之一——将耗 CPU 计算资源的加解密算法计算卸载到 QAT 加速卡——进行概述性介绍。



## 二、优化方案性能提升对比

### 1、测试方法

执行机部署适配 QAT 引擎后的 nginx，发包测试机进行压测灌包，在 CPU 负载达到 100%后比较得出 nginx 在进行 QAT 优化后的新建 connection 速率对比。



### **2、测试场景**

<img src="https://static001.infoq.cn/resource/image/79/02/79be898c30f16ffa68644abe5260eb02.png" alt="img" style="zoom:50%;" />

### **3、本地测试数据对比**



▷ 使用单张加速卡性能对比



![img](https://static001.infoq.cn/resource/image/7a/61/7a69717b4ce2c501af39347912b77b61.png)



▷ 使用双加速卡性能对比



![img](https://static001.infoq.cn/resource/image/50/e0/50db7cb7e34b32d416ec7951eb36dee0.png)



此优化方案，通过 NGINX 进行 HTTPS 新建速率实测，与软件加解密场景做对比：

- 使用单加速卡，rsa平均新建速率提升3倍，ecdh新建速率提升2.5倍
- 使用双加速卡：rsa 平均新建速率提升6 倍，ecdh新建速率提升5.5倍



此优化方案所带来的性能提升主要依赖于：

- 对比fsl 加速卡、qat 采用用户态驱动的方式，实现了内核态到用户态到内存零拷贝。
- NGINX采用异步模式调用OpenSSL API，代替传统的同步模式调用。
- 驱动支持多加速卡同时进行卸载加速。



## 三、硬件加解密异步处理

### 1、异步框架概述

JDD-LB 基于 nginx 原生的异步处理框架上拓展出针对异步硬件引擎的异步事件处理机制，整体框架如下图所示：



![img](https://static001.infoq.cn/resource/image/2c/6a/2cfbde71a0507ca51905f7yyce09af6a.png)



硬件加解密的异步框架整体依赖 nginx 的 epoll 异步框架处理机制和 openssl 协程机制。原生的 nginx epoll 框架负责网络 I/O 的异步事件处理，在此基础上 jdd-lb 重新增加了 async fd 异步监听任务，在原有的连接结构体中增加新的异步来源用来接收异步引擎的通知，当执行 OpenSSL 相关操作时，把返回的事件 fd 加载到 jdd-lb 的异步事件框架中，openssl 当检测到硬件执行完相关操作后就会唤醒相关事件进行后续操作的执行。

关于协程的详细介绍可以参考[《UAG性能优化之freescale加速卡》](http://mp.weixin.qq.com/s?__biz=MzI0MDc5NzQ2MQ==&mid=2247496741&idx=1&sn=3753e29db756d667f7e41b1dc6add814&chksm=e917e65fde606f496bee23186e8b96b566d6ad0787d0da8d7511d162bd11167834bdaacece94&scene=21#wechat_redirect)，里面有关于协程实现的基本语义、切换机制等详细介绍，这里不在赘述，总而言之，openssl 协程机制实现在线程内的多个任务切换管理。



这里涉及到两个问题：

- 异步任务的上下文切换以及通知机制；
- Nginx如何获取async fd，并加入epoll监听队列中，并与openssl 以及qat engine协同完成一次ssl 握手的；



### 2、交互流程

以 ssl 握手为例，nginx、openssl、qat 引擎各个模块之间的交互流程如下图所示：



<img src="https://static001.infoq.cn/resource/image/84/fd/8456311220d1543a28edc2fe0b2e98fd.png" alt="img" style="zoom:80%;" />



- ASYNC_start_job：nginx 调用ssl lib库接口SSL_do_handshake, 开启一个异步任务。
- Rsa/ecdh 加解密操作。
- Qat 引擎将加密消息发送给驱动，创建异步事件监听fd，将fd绑定到异步任务的上下文中。
- qat_pause_job: 调用该接口保存异步任务执行的堆栈信息，任务暂时被挂起，等待硬件加解密操作完成。同时进程堆栈切换到nginx io调用主流程，ssl返回WANT_ASYNC,nginx开始处理其他等待时间。
- nginx io 处理框架获取保存在异步任务上下文中的asyncfd，并添加到epoll队列中启动监听。
- 加速卡处理任务完成，qat 引擎调用qat_wake_job接口唤醒任务（也就是将async fd 标记为可读），这里有一个问题，这个qat_wake_job是什么触发执行的？qat 为nginx 提供了多种轮训方式去轮训加速卡响应队列，目前jdd-lb 采用的是启发式轮训的方式，具体参数可以在配置文件中定义。
- Nginx处理异步事件重新调用异步任务框架的ASYNC_start_job接口，这时候程序切换上下文，堆栈执行后跳回之前pase job的地方。



## **四、用户态驱动实现内存零拷贝**



区别于上期我们介绍的[freescale加速卡方案](http://mp.weixin.qq.com/s?__biz=MzI0MDc5NzQ2MQ==&mid=2247496741&idx=1&sn=3753e29db756d667f7e41b1dc6add814&chksm=e917e65fde606f496bee23186e8b96b566d6ad0787d0da8d7511d162bd11167834bdaacece94&scene=21#wechat_redirect)、qat 采用用户态驱动的实现方式，利用 linux uio+mmap 技术实现了内存零拷贝。接下来介绍 qat 实现内存零拷贝的基本原理。

在弄清楚这个问题前，我们先介绍一下 uio 技术的基本原理，简而言之：UIO（Userspace I/O）是运行在用户空间的 I/O 技术，Linux 系统中的驱动设备一般都是运行在内核空间，UIO 则是将驱动的很少一部分运行在内核空间，而在用户空间实现驱动的绝大多数功能。



<img src="https://static001.infoq.cn/resource/image/a5/36/a502c386552ca81fde9d7350b243c836.png" alt="img" style="zoom:50%;" />



工作原理图

我们先明确一个用户态程序如何操作一个 pci 设备?首先需要找到它的寄存器并对其进行配置，而找到寄存器的前提是拿到外设的基地址，即：通过“基地址+寄存器偏移” 就能找到寄存器所在的地址，然后就可以配置了。



综上，qat 的内核模块可以分为两块基本内容：

（1）PCI 设备驱动，获取 pci 设备的配置空间信息，申请内核资源；也就是拿到加速卡的基址寄存器地址，映射设备的地址空间。

（2）注册 UIO 设备，实现用户态驱动的主要功能：映射寄存器地址到用户空间，在内核态屏蔽设备中断；

利用 UIO 框架实现将设备的寄存器地址映射至用户空间后，QAT 还需要完成用户态进程与硬件之间消息传递。QAT 中消息队列的创建是在进程运行时创建，同样利用 mmap 原理将一段物理地址映射至用户态进程，为了避免内核在运行过程中出现内存碎片导致性能下降，QAT 新增了 usdm 内核模块用于管理和维护页表。



如下图所示，显示了 QAT 如何将设备地址及消息队列映射到用户空间：



<img src="https://static001.infoq.cn/resource/image/fc/ba/fc5abf78c8e893dde43a204dfdfb6fba.png" alt="img" style="zoom:80%;" />



综上所示，QAT 通过 UIO+ mmap 的方式实现了内核态到用户态到内存零拷贝，相比于 freescale 和 cavium 等其他加速卡厂商，QAT 在这一方面存在着一定优势。



关于内存映射的问题，jdd-lb nginx 多进程之间必须要保证物理地址的对齐偏移量一致，jdd-lb 在实际部署和上线过程中遇到过类似问题：

- **现象描述**

装载 qat 加速卡机器出现宕机，我们抓取宕机的 crash 日志，如下图：



<img src="https://static001.infoq.cn/resource/image/df/3f/df30b776176007bfbaef8f7c0ae4c33f.png" alt="img" style="zoom:50%;" />



系统挂之前操作消息队列（send_null_msg），内核被踩了，另外，在挂之前截图内核日志，发现大量内存申请失败的信息：



- **原因分析**

基于以上现象分析，机器挂之前的主要行为：1、nginx 正在频繁 reload；2、预分配的大页内存池不够了，新起的 worker 进程采用 ioctl 的方式从内核申请 2m 页，也就是我们在修复问题二之前采取的内存分配方式，这点可以从内核日志中分配内存失败的打印推断出来，因为只有通过 ioctl 方式申请才会出现 userMemAlloc failed 的打印。3、worker 退出前访问了消息队列，这个是导致宕机的直接原因。

总结一下三条行为的因果关系，频繁 reload > 老的 worker 还没彻底 shutdown，新的 worker 已经起来了, 同一时间出现大量 nginx > 预分配的大页内存池被占满 > 新的 worker 采用 ioctl 的方式去申请> 有个 worker 释放时访问消息队列 > 机器挂了。

根据以上分析，在机器挂之前 nginx 的 worker 进程是同时存在两种内存分配方式，但是 qat 是同时支持两种内存分配方式的，如果 ioctl 方式申请内存失败最多出现 nginx qat 加速卡卡降级，在问题二中已经说明了，并不会出现宕机的行为，难道是两种方式不能共存？



- **问题验证**

带着上述疑问，采取人为手段在测试环境复现该场景，先是继续在 nginx 拉起前人为的消耗完 2m 页，在 nginxworker 代码中加入调试控制，shutting down 之前 sleep100 秒（测试环境下 nginx shutdown 的速度非常快，并不能完全模拟线上环境的场景），然后运行一个 nginx 不断 reload 的脚本，最后，线上环境的问题在测试环境复现，crash 的日志与线上环境一致。

问题到这一步，只能说明两种内存分配方式共存的场景下，nginx 释放消息队列时会引起宕机，同样的测试场景，如果只采用一种内存分配的方式申请，都不会出现宕机。总之，知其然不知其所以然。然后经过阅读源代码加点日志分析判断，最终找到点眉目，下面放结论。

usdm 内核模块会维护物理地址和虚拟地址的映射关系，进程在释放队列后会去操作加速卡的 CSRs，而这个物理地址是通过创建的消息队列的物理地址偏移后计算出来的，虽然两种内存申请方式按 2m 的大小申请的，但是通过大页内存的方式申请的物理页是按 2m 的大小保存的，而通过 ioctl 方式申请的物理页是按照 4k 大小保存的，因此，如果通过同时存在两种内存分配方式的话，计算得到的 CSR 的物理地址就不一致了。画个图说明一下：



![img](https://static001.infoq.cn/resource/image/68/a1/6812c9a04de7cayy4e477e9ce22999a1.png)

## **五、进程级别的加速引擎调度**

QAT 的引擎启动主要完成 3 个步骤：

（1）完成应用程序注册：打开字符串设备/dev/qat_dev_processs，将 app 配置信息写入驱动，App 的信息必须与驱动配置中定义的配置块一致，也就是/etc/dh895xcc_dev0.conf 这个配置文件中的配置信息。

（2）完成服务的注册（qat 支持 cy 和 dc 两种服务），并完成加速卡硬件相关资源的初始化操作。

（3）获取服务实例，这里的 instances 可以理解为应用程序与加速卡相互通信的通道，实际上就是与底层的加速卡和消息队列建立绑定关系。所以 8950 加速卡最多支持 128 个 instance（考虑硬件队列最多就 256 个）。

需要说明的是：qat 的 instance 资源是进程级别，那么每个 worker 在启动之前都会调用 qat_engine_init 函数，qat 用了一个钩子使得 master 进程在 fork 之后子进程调用 qat_engine_init 函数，同时在 fork 之前释放引擎（master 本身不需要引擎资源），流程如下图所示：



![img](https://static001.infoq.cn/resource/image/c3/eb/c36e4645be03d71de4a18ae7fa0330eb.png)



## **六、QAT 组件框架概览**



![img](https://static001.infoq.cn/resource/image/cf/54/cf7b1286cf2fbd37c3c5f8ca70448f54.png)



- **Application**

应用层主要包含两块内容：（1）qat 异步框架的 patch，该 patch 提供对异步模式的支持；（2）qat 引擎，engine 是 openssl 本身支持的一种机制，用以抽象各种加密算法的实现方式，intel 提供了 qat 引擎的开源代码用以专门支持 qat 加速。

- **SAL（service access layer）**

服务接入层，给上层 Application 提供加速卡接入服务，目前 qat 主要提供 crypto 和 compression 两种服务，每一种服务都相互独立，接入层封装了一系列实用的接口，包括创建实例，初始化消息队列、发送\接受请求等。

- **ADF（acceleration driver framework）**

加速卡驱动框架，提供 SAL 需要的驱动支持，如上图，包括 intel_qat.ko、8950pci 驱动、usdm 内存管理驱动等。



## **七、总结与思考**

截止目前，JDD-LB 在硬件加速领域，已经同时支持飞思卡尔与 intel QAT 硬件方案，为有效替代 f5 提供了性能保证，成功实现核心网络组件自主可控，为构建金融级的网关架构赋能行业打下坚实的基础。

未来 JDD-LB 将持续构建接入层网关能力体系。

- **安全与合规**

作为京东数科统一流量接入入口，JDD-LB 将持续构建金融级的通信安全基础设施，打造全方位的安全防护体系。

- **多协议支持**

JDD-LB 在高效接入能力建设方面将持续投入，通过引入 QUIC 协议，将提升用户在弱网场景下的用户支付体验。

通过 MQTT 协议可以通过非常小的接入成本实现新设备和协议接入，积极拥抱万物互联。



文章转载自： 京东数科技术说（ID：JDDTechTalk）

原文链接：[京东数科统一接入网关JDDLB性能优化之QAT加速卡](https://mp.weixin.qq.com/s/NmgT2p_Yq-j9HpswtISvpA)

2021 年 1 月 11 日 08:00860

文章版权归极客邦科技InfoQ所有，未经许可不得转载。



原文：https://www.infoq.cn/article/AGJdlX34hccxJNtH9svI