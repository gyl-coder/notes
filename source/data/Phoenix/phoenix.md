## 项目介绍（Phoenix）

Phoenix是一个有状态的响应式微服务框架。它提供了高可用、动态伸缩、分布式事务、内存管理、实时监控等关键能力。擅长解决金融领域中业务复杂，性能要求高的场景。

Phoenix是一套响应式微服务开发框架，致力于打造有状态服务。在应用系统架构方面，提供了包括客户端开发、服务端开发、消息通讯架构、服务端状态数据存储和恢复、计算层和消息层监控、服务端运维等架构方案。

而在业务开发方面，Phoenix提供了一套领域驱动设计模式(DDD)的实现模型，业务开发中的领域模型定义以聚合根(Aggregate Root)的形式作为业务计算的基本单元，在整个业务计算的流程中，聚合根对象之间仅以消息(Message)作为数据交互接口。框架对聚合根对象的状态管理和多层次交互的实现进行了封装。Phoenix可以支持用无状态的方式来开发高价值的有状态场景。



## 简介（初衷，优势，为什么造轮子）



## 微服务怎么做服务间通讯

基于消息中间件，事件驱动、异步调用



## 基于消息通讯怎么保证消息不丢？

https://gitlab.iquantex.com/kunlun/phoenix/phoenix/-/wikis/phoenix/%E4%BA%8B%E5%8A%A1%E9%99%90%E6%B5%81&%E5%8F%AF%E9%9D%A0%E6%80%A7%E6%8A%95%E9%80%92%E8%AE%BE%E8%AE%A1

- 消息的顺序处理问题
- 幂等问题

之前的版本，严重依赖消息中间件Kafka自身的消息可靠性。

之后基于种种原因（依赖过重，金融机构不太信任kafka，需要运维成本），不在依赖kafka来做消息的可靠性保证

http://www.voidcn.com/article/p-gwyqtmen-bnb.html

**Phoenix 通过超时、重传、确认、去重的机制来保证消息的可靠投递，不丢不重。**

**Phoenix 自己实现的消息可靠投递机制 + 定时重试。**

可靠性投递是什么？

事务可靠性投递解决的是事务在投递过程中因为网络等原因导致消息丢失事务处理不完的情况。

事务可靠性投递，针对不同的场景分两个级别：

1. C端场景：在C端的场景下，对于客户端一笔事务的处理是允许超时的。该种场景下，事务可靠性投递可以采用 ”重试一定次数就放弃重试，移除该笔事务“策略，并且是可以考虑不需要持久化状态的，在AT重启后，虽然状态丢失，但是客户端可以发起重试。如何C端要支持事务的最终一致，那么也是需要持久化的。
2. Q端场景：在Q端场景下，很多时候是要求严格处理上游服务事件的的。该种模型下，可靠性投递可以采用”重试很多次数，可以放弃重试（不过要提醒告警）“，另外是需要严格持久化状态的，防止在AT重启后状态丢失，丢失当下状态的事件的处理的。

Phoenix 中的实现，维护了一个可靠性投递的聚合根（eventsotre,eventsourcing...），ReceiverActor接收到（新事务、结束事务）都会以不同的事件去触发该聚合根修改状态。同时有一个定时器（ReceiverSupervisorActor）每隔2秒会发送一次心跳（查询未完成事务），然后发送未完成事务消息。

Phoenix 默认的重试策略（重试策略是一个接口，可以提供扩展）为：超过重试间隔时间就重试，并且每一次重试，最多重试 50 个消息（参数可配置）否则就保留，待下次继续重试。

不移除的原因是：我们要保证phoenix只要是从kafka成功拉取的消息，就要驱动到一个结果

客户端也存在一个重试， 待重试次数超过 3 次（每个10秒重试一次）就移除，并返回超时。

#### 幂等处理

聚合ID：聚合类型@聚合根类型@聚合根ID

实体聚合根幂等ID： msgId和业务自定义幂等

聚合根在处理消息时，先进行内存幂等（内存幂等使用布隆过滤器有可能误判），之后在进行数据库幂等。

聚合根正常处理消息（handle（act）  --  apply（on）  --   append）

handle/apply 异常时会销毁聚合根，然后重新恢复到之前的状态。

- 主键冲突（aggregateId + version）  脑裂
- 唯一键冲突（aggregateId + idempotentId） 内存幂等能够保证不存在唯一键冲突

遇到异常均会清理掉当前聚合根，然后重新eventsourcing恢复聚合根状态。

## phx遇到洪峰流量碰到的问题是什么？最先遇到瓶颈的可能是啥？如何解决

https://gitlab.iquantex.com/kunlun/phoenix/phoenix/-/wikis/phoenix/%E4%BA%8B%E5%8A%A1%E9%99%90%E6%B5%81&%E5%8F%AF%E9%9D%A0%E6%80%A7%E6%8A%95%E9%80%92%E8%AE%BE%E8%AE%A1

瓶颈： 事务

解决方案： 限流

限流限的是什么？限的是外部流量进入本微服务集群的流量(本集群代表的是几个服务组成的)，限流的策略是一种背压策略，发送发（AT）的下游（EN和TN）在处理完本集群事情之后会回复给AT完成消息。AT基于统计当下下游有多少未完成的事情(注意这里为什么不是事务)来做是否拉取消息。

这里解释一下，为什么是本集群的事情，而不是事务。因为对于TN来说有一种场景是跨集群的事务，这里我们不能等整个事务完成再给AT回复，而是等这个事务在本集群的事情处理完之后就回复。当然对于EN来说，就没有这个问题。

#### 怎样判断一个消息在本集群内事情处理完？

1. EN：对于EN来说，如果是AT投递给EN的消息，EN处理完之后可以直接回复给AT，代表本集群处理完毕。
2. TN：对于TN来说，要处理的场景就复杂了。TN不能再收到AT消息处理完之后就直接回复说处理完了，因为这样不能表现出集群的真实处理能力（有可能集群内还要再处理），同时也不能等整个事务处理完了再回复（前面已经解释过）。同时TN是有可能一个消息返回多个消息的，这多个消息有可能一些是在本集群内处理，一些是要路由到外部集群处理。所以这块设计要严谨推销才行，

初步想法： TN内的事务状态机要维护一个队列，如果一个消息的返回消息是要本集群的处理的，就加到队列上，如果本集群内相关的消息处理完了（队列为空），就回复给AT消息处理完的标志。相关设计参见



## 数据可靠性和一致性怎么做？

**可靠性：**事件存储依赖数据库（Eventstore）

聚合根每一次的处理都会产生一个相应的事件（状态变更事件），每一个事件都会被存储到数据库中。

**一致性：**事件版本管理，数据库做唯一键检查

#### 内存快照如何保证一致性

版本号



## EventSourcing



## CQRS

CQRS 将系统中的操作分为两类，即「命令」(Command) 与「查询」(Query)。命令则是对会引起数据发生变化操作的总称，即我们常说的新增，更新，删除这些操作，都是命令。而查询则和字面意思一样，即不会对数据产生变化的操作，只是按照某些条件查找数据。

CQRS 的核心思想是将这两类不同的操作进行分离，然后在两个独立的「服务」中实现。这里的「服务」一般是指两个独立部署的应用。在某些特殊情况下，也可以部署在同一个应用内的不同接口上。



## Phoenix 中分布式事务实现

分布式事务

TCC、SAGA、二阶段提交

TCC的CC失败怎么处理？

http://10.116.18.21:32042/docs/phoenix-2.x/03-phoenix-transaction/phoenix-transaction-module



KAFKA怎么保证消息不丢？消息不丢配置

​				发送端ACK机制，服务端冗余设计，消费端commit offset控制

发送端和消费端的要求？

​				同上

消费端消费后失败，怎么进行重试？
						目前是超时重试。









mq选型

分片

cqrs

内存大小，es时间

有状态/无状态的理解

团队规模，公司规模，公司业务

phoenix里有什么展现java能力的亮点

phx优点是什么，使用场景是什么，单线程模型如何使用cpu的多核特性

phx和db的关系

phx如何保证内存可靠性和灾备



phx里你主要负责的模块？ 多少人做这个框架，成员怎么分布

介绍下actor模型

介绍下akka-cluster和akka-sharding

介绍akka里的actor的生命周期

介绍phx的模型，水平扩容如何实现，提升易用性和性能是什么意思，phx里akka的用法

介绍下saga模型，介绍下事务应该怎么编排





## 开发过程中遇到的问题

phoenix-server 剥离 spring

