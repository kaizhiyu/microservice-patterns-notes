# 基于 DDD、CQRS、微服务和事件溯源构建系统



- Jan Stenberg

- 盖磊



**2019 年 11 月 03 日

**[敏捷](https://www.infoq.cn/topic/agile)[架构](https://www.infoq.cn/topic/architecture)[微服务](https://www.infoq.cn/topic/microservice)



![基于DDD、CQRS、微服务和事件溯源构建系统](https://static001.infoq.cn/resource/image/73/17/737250118d29659db47848403e155017.png?x-oss-process=image/crop,y_2,w_1067,h_598/resize,w_726,h_408)

对于构建系统来说，模块化是至关重要的，但实现模块化需要应对一些反模块化的做法。两种典型的反模块化做法就是在不考虑业务领域的情况下走捷径和拆分微服务，这会增加技术债务。最近，在由[AxonIQ](https://axoniq.io/)举办的阿姆斯特丹[事件驱动微服务大会](https://axoniq.io/event-overview/event-driven-microservices-conference)上，[Allard Buijze](https://twitter.com/allardbz)分享了他的一些想法，以及基于[DDD](http://www.methodsandtools.com/archive/archive.php?id=97)、[CQRS](https://microservices.io/patterns/data/cqrs.html)、微服务和[事件溯源](https://microservices.io/patterns/data/event-sourcing.html)构建系统的亲身经验。



Buijze 是 AxonIQ 的 CTO。他指出，另一个反模块化做法就是名词驱动设计（noun-driven design）。名词驱动设计用于发现应用程序中的对象，通常是通过查找应用程序中出现的名词，而非采用那些用于创建服务的对象。如果开发人员需要一次性重新部署多个服务，或是某个服务需要依赖于其他服务，就会从单体应用转为[分布式单体应用](https://www.simplethread.com/youre-not-actually-building-microservices/)，但潜在的问题并没有得到解决，这并不是微服务架构。



Buijze 认为，转向微服务需要一个过程，无法一步将业务模型转为微服务。反之，我们应该从那些具有良好结构和模块化的单体应用着手，根据需求逐步拆分出新的微服务。他强调，创建微服务的需求应该是非功能性需求。



Buijze 指出，将组件抽取成微服务的重点在于实现[位置透明性](https://docs.axoniq.io/reference-guide/architecture-overview#location-transparency)。组件不应该知道或者假定知道与之交互的其他组件的位置。这意味着当组件可被抽取为微服务时就无需重写现有组件与新服务之间的通信。



解决位置透明性问题的通常做法是使用事件。一个服务无需直接调用与之通信的其他服务，而是通过发布事件并设置服务去监听事件。Buijze 指出，使用事件的一个重要特点是实现了依赖关系的转置，即订阅事件的组件或服务可以直接监听发送的事件。



Buijze 介绍了只使用事件会出现的一个常见错误，即事件也会被用于间接请求发生于其他服务中的操作，这会增加服务间的耦合度，可能导致双向依赖，使得服务间依赖关系难以编排。如下例，图中没有负责业务流程的服务：



![img](https://static001.infoq.cn/resource/image/b5/ba/b58051f952cee0ff9c7462500ae699ba.jpg)



组件间会因为不同的原因发起通信。除了使用事件之外，还存在另外两种消息，一种是用于表示改变事物的“命令”（Command），另一是用于获取信息的“查询”（Query）。在上例中，订单服务（Order）可使用“命令”去请求支付（Payment）和交货（Shipping）服务，交货服务可使用“查询”去请求订单服务细节。Buijze 建议我们应该像使用事件那样同等使用命令和查询。



事件溯源是另一个与事件相关的概念。在使用事件溯源时，组件中存储的并非实体的具体状态，而是导致每个实体状态发生变更的事件。Buijze 认为，事件源就是获取所有的事实，并且只关心事实。但他也指出，事件溯源必须被正确实现，否则就无法确保事实的完整性。要验证事件溯源是否被正确实现，可以试着抛开事件存储以外的东西。如果应用程序的状态可重现，说明事件溯源做对了。Buijze 进一步指出，要在服务中使用事件溯源，服务必须使用自己发布的事件来保持一致的状态。



从业务和技术方面来看，事件溯源都有一些优势，审计和单一数据源就是两个很好的例子。 事件溯源同样存在一些挑战，例如增加了存储规模、实现复杂度高。但 Buijze 认为，这些问题现已不复存在，目前最大的挑战在于事件思维（event thinking）。在 Buijze 看来，事件是一种更为自然的理解应用程序行为的思维方式。他建议，我们应该将注意力放在行为上，而非状态上。事件和命令描述了应用程序的功能，因此让应用程序的行为变得更容易理解。但他并不建议对所有应用程序使用事件溯源。与其他工具一样，事件溯源仅在合适的场景才能发挥作用。



最后，Buijze 强调，所有的通信都是某种形式的合约。基于事件架构的一个缺点是需要对很多未知的组件建立合约。因此，我们要对事件和合约的范围加以约束，避免各个有界上下文之间存在耦合。在同一上下文中，所有服务使用同一种语言，并且能理解所有的事件。但是很多事件，尤其是事件溯源中的事件，只在特定的上下文中是有用的，不应该被发布到上下文之外。对于一些公共事件，一种好的做法是将事件转换为公共 API。



Martin Fowler 在 2015 年发表的一篇博文中提到，他已经注意到[几](https://www.martinfowler.com/bliki/MonolithFirst.html)[乎所有成功的微服务均源于单体应用](https://www.martinfowler.com/bliki/MonolithFirst.html)。但此后不久，Stefan Tilkov 在其博客中提出了不同的看法，即如果目标是实现微服务架构，那么[从单体应用着手是完全错误的做法](https://martinfowler.com/articles/dont-start-monolith.html)。



在[欧洲](https://dddeurope.com/2019/speakers/eric-evans/)[2019 ](https://dddeurope.com/2019/speakers/eric-evans/)[DDD大会](https://dddeurope.com/2019/speakers/eric-evans/)的一个演讲中，Eric Evans 介绍了各种类型的“有界上下文”，其中一些类型尤其适用于基于事件的系统。



大会演讲全程录像，并将于近几周内公开发布。



**原文链接：**



[Sense and Nonsense in Event Thinking and Microservices](https://www.infoq.com/news/2019/10/event-thinking-microservices/)



2019 年 11 月 03 日 08:002285

文章版权归极客邦科技InfoQ所有，未经许可不得转载。



原文：https://www.infoq.cn/article/tGFkAm1SyyP3IWiBmKBH