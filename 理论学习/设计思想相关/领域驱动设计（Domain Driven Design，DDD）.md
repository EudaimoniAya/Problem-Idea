 目前大多数的Web应用都是三层架构：
   ![[Pasted image 20260326233637.png]]
 * Presentation：UI、Controller、BFF（Backend For Frontend）
 * Application：Service、Validation/CRUD
 * Infrastructure（Infra）：Database、Library
但是这种架构存在一些问题：
* Application容易过于庞大和臃肿，容易导致业务逻辑不够清晰，对于项目的维护会很困难。
* Application对Infra的依赖性过高，如果Infra需要进行一些技术层次的升级（如更先进的Database，或新的第三方轮子）
* 当体系过于庞大，那么拆分成微服务就会很困难
种种原因导致传统三层架构，对于一个项目，尤其是正在演进的项目存在阻碍，而DDD在此之上进行了改造：
* Application层和Infra层之间应该新增一个Domain层，即领域。
  关于领域，Web的业务逻辑分为两类：一类是固定不变的常用逻辑，比如订单正常下单的常用业务逻辑，这类逻辑的核心内容可能存在相似，甚至是固定不变的。这样的逻辑应该放在Domain层。
* Application层主要就用来负责接入不同的use case（应用场合），基于不同的应用场合会产生不同的业务逻辑。比如某个网络应用，用户从手机端接入和从网络端接入，它的业务逻辑可能会产生些许的不同，那么这些逻辑应该放在Application层。
* 由于Infra层大部分包含的是第三方类库，以及一些著名的开源轮子，如消息队列、缓存等。所以DDD希望应用程序的所有层尽可能独立于Infra层，因此Infra在该架构中的位置出现了变化：已有的三层对Infra层应该没有任何依赖关系，这个就通过防腐层进行隔离
  ![[Pasted image 20260327002448.png]]
* Presentation层和Application层之间传输的东西，应该是Rest DTO（Data Transfer Object），它用来声明应用和用户之间的接口数据定义，它不应该轻易修改
* Application和Domain传输的是entity，因为Rest DTO不应该轻易修改，但是Domain层的业务逻辑可能随时发生变化
---
![[Pasted image 20260327003800.png]]
![[Pasted image 20260327003917.png]]
![[Pasted image 20260327003955.png]]
![[Pasted image 20260327004021.png]]
