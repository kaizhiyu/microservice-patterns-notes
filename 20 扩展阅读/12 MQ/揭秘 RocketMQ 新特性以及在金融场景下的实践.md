# 揭秘 RocketMQ 新特性以及在金融场景下的实践

- 陈广胜

**2020 年 3 月 10 日

2019 年末， RocketMQ 正式发布了 4.6.0 版本，增加了“ Request-Reply ”的同步调用的新特性。“ Request-Reply ”这个新特性是由微众银行的开发者们总结实践经验，并反馈给社区的。接下来本文会详细介绍此新特性。



## “ Request-Reply ”是什么



![img](https://static001.infoq.cn/resource/image/60/02/60b53759bc620bf641ed3a9e5456e402.png)



图 1.1 “ Request-Reply ”模式



在以往的消息中间件的使用中， Producer 和 Consumer 只负责发送消息和消费消息，彼此之间不会通信。而 “ Request-Reply ”模式允许 Producer 发出消息后，以同步或者异步的形式等待 Consumer 消费这条消息并返回一个响应消息，从而达到类似 RPC 的调用效果。在整个“ Request-Reply ”调用过程中（简称 RR 调用）， Producer 首先发出一条消息，消息经由 Broker 被 Consumer 获取并消费；Consumer 消费完这条消息后，会将针对该消息的响应作为另外一条消息发送出来，最终回到 Producer 。为了便于描述，称此时的 Producer 为请求方，发出的消息为“请求消息”；Consumer 称为服务方，返回的消息称为“响应消息”。



“ Request-Reply ”模式使得 RocketMQ 具备了同步调用的能力，拓展了 RocketMQ 的使用场景，使其具有更多的应用可能性。开发者可以利用这个特性快速搭建自己的消息服务总线，实现 RPC 调用框架；由于请求以消息的形式存储在 Broker ，便于收集信息做调用链追踪和分析；在微服务领域，也有着广泛的应用场景。



## “ Request-Reply ”的实现逻辑



在 RR 调用中涉及到 Producer、Broker、Consumer 三个角色。



### Producer 的实现逻辑



![img](https://static001.infoq.cn/resource/image/13/cc/13f0fe4b3e87bd18a9f53a6a10636acc.png)



图 2.1 producer 示意图



**1、对请求消息增加对应的标识**



Producer 发送请求消息时，需要在消息的 Properties 里增加 RR 调用的标识，其中关键的字段有 Correlation_Id、REPLY_TO_CLIENT。Correlation_Id 用来唯一标识一次 RR 请求，通过这个属性来匹配同一个 RR 调用的请求消息和响应消息。REPLY_TO_CLIENT 用来标识请求消息的发出方，其值为 Producer 的 ClientId 。



作为请求方的 Producer 只需增加对应标识到消息中，在消息的发送逻辑上与原始 Producer 保持一致。



**2、发完请求消息后等待响应消息。**



请求方每次执行 Request 之后，会创建 RequestResponseFuture 对象，并且以 Correlation_Id 作为 key 记录到 ResponseFutureTable 中。执行 Request 的线程通过 RequestResponseFuture 里定义的 CountDownLatch 实现阻塞。当响应消息回到 Producer 实例时，根据响应消息中的 Correlation_Id 从 ResponseFutureTable 中获取对应地 RequestResponseFuture ，激活 CountDownLatch 唤醒阻塞的线程，执行对响应消息的处理。



![img](https://static001.infoq.cn/resource/image/95/76/95afb66147320de5c4defcf42f6c3b76.png)



图 2.2 RequestResponseFuture 结构



### Consumer 的实现逻辑



![img](https://static001.infoq.cn/resource/image/b8/64/b8b66cfb3c6533e0bfb9382d32eb5464.png)



图 2.3 consumer 示意图



Consumer 只需要在正常消费一条请求消息后，创建响应消息并发送出去即可。创建响应消息时必须使用提供的工具类来创建，避免丢失 Correlation_Id、REPLY_TO_CLIENT 等标识和关联 RR 请求的属性。



### Broker 的实现逻辑



![img](https://static001.infoq.cn/resource/image/2c/5c/2ca98c2b0a4e588b46242548e648095c.png)



图 2.4 Broker 示意图



Broker 对请求消息的处理与原先的处理逻辑一样，但是对于响应消息则是采用主动 Push 的形式将消息推给请求方。服务方 Consumer 将响应消息发送到 Reply_topic 上， Broker 收到响应消息后会交由 ReplyMessageProcessor 处理。Processor 会将响应消息落到 CommitLog 中，并且根据响应消息中的 REPLY_TO_CLIENT 得到请求方的 ClientId ，通过 ClientId 找到对应的 Producer 实例及其 Channel ，将响应消息直接推送给它。



所有的响应消息都会发送到 Reply_topic 上，这个 Topic 是由 Broker 自动创建的系统 Topic ，以“集群名 _REPLY_TOPIC ”的格式命名。Reply topic 用于做路由发现，让响应消息能够发回到请求消息来源的那个集群，目的是保证响应消息回到的 Broker 是请求方有连接的 Broker 。采用 Broker 主动推送响应消息的目的也是为了保证响应消息能够精准回到发出请求消息的实例上。



## “ Request-Reply ”在金融场景下的实践



金融业务要求服务要持续稳定，能够提供 7x24 小时稳定可用的服务，并且容错能力要足够强，对节点故障能够快速屏蔽影响，保证成功率，快速恢复。因此，微众银行根据具体的使用场景增加了应用多活、服务就近、熔断等特性，构建了安全可靠的金融级消息总线 DeFiBus 。



![img](https://static001.infoq.cn/resource/image/2a/aa/2abe3b877dd50dc149020d3fa57bedaa.png)



图 3.1 总线架构图



如图所示， DeFiBus 自上而下分别是总线层、应用层、 DB 层。



总线层有两个非常重要的服务，分别是 GNS 和 GSL 。对每个客户，会根据客户信息并且按照权重分配到规划好的 DCN 内，实现数据层面的分片。GNS 服务是在数据层面进行的分片寻址，确定客户所在的 DCN 。在服务层面，会将服务部署到不同的区域，在调用服务时会先访问 GSL 服务，做服务层面的分片寻址，确定当前要访问的服务在哪个 DCN 。从数据和服务两个维度做分片，由 GNS 和 GSL 做分片寻址，最终由总线实现请求到 DCN 的自动路由。



请求从流量入口进来经由 GNS 和 GSL 寻址，确定服务所在的 DCN 后，总线会将请求自动路由到对应服务所在的 DCN 区域，交由应用处理。每个 DCN 内的应用只处理本 DCN 内的请求。应用会访问同 DCN 内预先分配的主 DB ， DB 层会有一个多副本来提高可靠性。 为了提升服务的可用性和可靠性， DeFiBus 的开发者针对“ Request-Reply ”的使用做了多个方面的优化和改造。



### 快速失败和重试



![img](https://static001.infoq.cn/resource/image/ab/3a/aba28894b602ce61c9b67cd58fa8743a.png)



图 3.2 快速失败和重试示意图



从使用方视角来看，业务的超时时间等同于整个完整 RR 调用的超时时间。一次 RR 调用内部会涉及 2 次消息的发送，当 Broker 有故障时，可能会出现消息发送超时。因此，内部发送消息的超时时间设置会根据业务超时时间自动调整为较小的值，为失败重试留足更多的时间。比如业务超时时间为 3s ，则设置发送消息的超时为 1s 。通过调整消息发送超时时间来快速发现 Broker 的故障。当发现 Broker 的故障后， Producer 会立即重试另外一个 Broker ，并隔离失败的 Broker 。在隔离结束前， Producer 不会再将消息发到隔离的 Broker 上。



### 熔断机制



![img](https://static001.infoq.cn/resource/image/bb/1c/bb924e8eab0786176d6ff7b87d06a31c.png)



图 3.3 熔断示意图



熔断机制是指当某个队列消息堆积达到指定阈值后，不再往这个队列发送消息，使得这个队列对应的服务实例暂时熔断。



为了实现熔断机制，队列增加了“队列深度”属性。队列深度指一个队列中堆积在 Broker 上未被 Consumer 拉取的消息量。当 Consumer 发生故障或者处理异常，首先触发客户端的流控机制，随后拉消息请求会被不断地延迟，此时消息会堆积在 Broker 上。当 Broker 发现某个队列堆积的消息量超过阈值，会标记队列为熔断。Producer 发送消息时如果目标队列已经熔断，则会收到队列熔断的响应码，并立即重试，发送消息到另外的队列，同时将熔断的队列标记为隔离。在隔离解除前， Producer 不会再往隔离的队列发送消息。



### 隔离机制



队列级别的隔离机制主要用于 Producer 的重试和服务的熔断机制。



![img](https://static001.infoq.cn/resource/image/31/8b/317681ccc4854b0724a5798e921c448b.png)



图 3.4 隔离示意图



当 Broker 故障时， Consumer 拉消息会触发隔离机制。原生 RocketMQ 的 Consumer 实现中，由 PullMessageService 单个线程向所有 Broker 发送拉消息请求。当这些 Broker 中有节点故障时， PullMessageService 线程会因为与故障 broker 建立连接或者请求响应变慢，导致线程暂时阻塞，这会让其它正常 Broker 的消息处理耗时变高甚至超时。因此，开发者为拉消息增加了一个备用线程，一旦发现拉消息的请求执行时间超过阈值，则标记这个 Broker 为隔离，对应的所有拉消息的请求转交给备用线程执行，保证 PullMessageService 执行的都是正常的 Broker 的请求。通过线程隔离来保证部分 Broker 的故障不会影响 Consumer 实例拉消息的时效。



### 队列动态扩容/缩容



队列动态扩容和缩容目的是保持队列数和 Consumer 实例数的一致，使得负载均衡后每个实例消费的队列数一样。在 Producer 均匀发送的情况下，使得 Consumer 实例不会因为分到的队列数量不一样而出现负载不均衡。



扩容/缩容通过动态调整 Topic 配置的 ReadQueueNum 和 WriteQueueNum 来实现。



在扩容时，首先增加可读队列个数，保证 Consumer 先完成监听，再增加可写队列个数，使得 Producer 可以往新增加的队列发消息。



![img](https://static001.infoq.cn/resource/image/e8/e6/e807d938f7167c71c0be93d80b3b0be6.png)



图 3.5 队列扩容示意图



队列缩容与扩容的过程相反，先缩小可写队列个数，不再往即将缩掉的队列发消息，等到 Consumer 将该队列里的消息全部消费完成后，再缩小可读队列个数，完成缩容过程。



![img](https://static001.infoq.cn/resource/image/f8/d4/f8fcacc9644e691f2cba975e3492b5d4.png)



图 3.6 队列缩容示意图



### 负载均衡过渡



RocketMQ Consumer 在负载均衡结果发生变化时，会将老结果直接更新为新结果，是一个 A 到 B 跳变的过程。当 Consumer 和 Broker 多的时候，不同的 Consumer 在负载均衡时获取到的 Consumer 个数以及队列个数可能出现不一致，导致负载均衡结果不一致。当结果不一致时就会出现队列漏听和重复听的问题。对于同步调用场景，队列出现漏听会导致漏听队列上的消息处理耗时变高甚至超时，导致调用失败。



负载均衡过渡则是将负载均衡结果变化过程增加了一个过渡态，在过渡态的时候， Consumer 会继续保留上一次负载均衡的结果，直到一个负载均衡周期结束或者感知到新的属主已经监听上这个队列时，才释放老的结果。



![img](https://static001.infoq.cn/resource/image/97/32/9719bc3c960de97985c5ba7a98a48832.png)



图 3.7 负载均衡过渡示意图



### 同城应用多活



为了达到高可用和容灾的一些要求，服务会部署在至少两个数据中心。当一个数据中心有某个服务全部故障不可用时，其他数据中心正常的实例能自动接管这部分流量。在部署的时候，请求方和服务方在两个数据中心都会部署，当两中心都正常时，请求方会依照服务就近的原则，将请求发到同 IDC 内，跨 IDC 只通过心跳维持连接。服务方订阅时优先监听同 IDC 内的队列。



![img](https://static001.infoq.cn/resource/image/4f/ca/4fb92f4c892dfba15e0247265cf4a5ca.png)



图 3.8 正常情况示意图



当且仅当另外一个 IDC 中没有存活的服务实例时，服务方才会跨 IDC 接管其他 IDC 的队列。如图，当数据中心 2 的应用 B 实例全部挂掉后，部署在数据中心 1 的实例 1 、 2 、3 在负载均衡时首先对同 IDC 内的队列进行分配，然后检查发现数据中心 2 有队列但无存活的应用 B 实例，此时会将数据中心 2 的队列分配给数据中心 1 的实例，实现跨 IDC 的自动接管。



![img](https://static001.infoq.cn/resource/image/20/fd/2058ec72ba0ea1e99c867670a18740fd.png)



图 3.9 应用故障情况示意图



当某一个数据中心的 Broker 全部挂掉之后，请求方会跨 IDC 进行发送。如图，在数据中心 2 的 Broker 全部故障后，应用 A 的实例 4~6 会将请求发送到数据中心 1 ，根据服务就近原则，这部分请求会由数据中心 1 的应用 B 实例 1~3 处理，从而保证 Broker 故障后，经由数据中心 2 进来的请求也能被正常处理。



![img](https://static001.infoq.cn/resource/image/6e/8d/6eb4800af8b7260d998497f848c3fd8d.png)



图 3.10 Broker 故障情况示意图



## 四、结语

本文主要介绍了 RocketMQ 新特性——“ Request-Reply ”模式。此模式下， Producer 在发出消息后，会等待 Consumer 消费并返回响应消息，达到类似 RPC 调用的效果。“ Request-Reply ”模式让 RocketMQ 具备了同步调用的能力，在此基础上，开发者可以开发更多新的特性。为了更好的服务于金融场景，微众银行又增加了应用多活，服务就近，熔断等新的特性，构建了安全可靠的金融级消息总线 DeFiBus 。目前微众银行已经将大部分成果通过 DeFiBus 开源出来，后续在分片和寻址方面也会有更通用的实践总结和成果介绍，欢迎各位了解关注！



**作者介绍**：

陈广胜，Apache RocketMQ Committer，DeFiBus 创始人，微众银行技术专家，中间件平台相关产品负责人，曾就职于 IBM 和华为，负责运营商云和大数据平台建设。



**本文转载自公众号阿里巴巴中间件（ID：Aliware_2018）。**

**原文链接**：

https://mp.weixin.qq.com/s?__biz=MzU4NzU0MDIzOQ==&mid=2247489065&idx=2&sn=cbe12c37a89ac470287b57f3ea3d824b&chksm=fdeb2449ca9cad5f26c2968788f0f3ca02ac5ee14162a0df9c0ef8b6c8ec85fe214923227114&scene=27#wechat_redirect

2020 年 3 月 10 日 14:001625



原文：https://www.infoq.cn/article/b3NVxs1t2K1Ygrof5qbI