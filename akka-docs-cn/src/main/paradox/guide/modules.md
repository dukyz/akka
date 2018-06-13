# Akka库和模块概述

Before delving into some best practices for writing actors, it will be helpful to preview the most commonly used Akka libraries. This will help you start thinking about the functionality you want to use in your system. All core Akka functionality is available as Open Source Software (OSS). Lightbend sponsors Akka development but can also help you with [commercial offerings ](https://www.lightbend.com/platform/subscription) such as training, consulting, support, and [Enterprise Suite](https://www.lightbend.com/platform/production) &#8212; a comprehensive set of tools for managing Akka systems.

在深入研究编写Actor的一些最佳实践之前，先预览最常用的Akka库会很有帮助。这将帮助您开始考虑您想要在系统中使用的功能。所有核心Akka功能都可以作为开源软件（OSS）使用。Lightbend赞助Akka开发，但也可以帮助您获得[商业产品](https://www.lightbend.com/platform/subscription)，如培训，咨询，支持和[企业套件](https://www.lightbend.com/platform/production) - 用于管理Akka系统的全套工具

The following capabilities are included with Akka OSS and are introduced later on this page:

Akka 开源软件包含以下功能，稍后会在此页面中介绍：

* [Actor library](#actor-library)
* [Remoting](#remoting)
* [Cluster](#cluster)
* [Cluster Sharding](#cluster-sharding)
* [Cluster Singleton](#cluster-singleton)
* [Cluster Publish-Subscribe](#cluster-publish-subscribe)
* [Persistence](#persistence)
* [Distributed Data](#distributed-data)
* [Streams](#streams)
* [HTTP](#http)

With a Lightbend subscription, you can use [Enterprise Suite](https://www.lightbend.com/platform/production) in production. Enterprise Suite includes the following extensions to Akka core functionality:

借助Lightbend订阅，您可以在生产环境中使用[Enterprise Suite](https://www.lightbend.com/platform/production)。Enterprise Suite包含Akka核心功能的以下扩展：

* [Split Brain Resolver](https://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html) &#8212; Detects and recovers from network partitions, eliminating data inconsistencies and possible downtime.
* [网络脑裂处理](https://developer.lightbend.com/docs/akka-commercial-addons/current/split-brain-resolver.html) - 从网络分区检测和恢复，消除数据不一致性和可能的停机时间。
* [Configuration Checker](https://developer.lightbend.com/docs/akka-commercial-addons/current/config-checker.html) &#8212; Checks for potential configuration issues and logs suggestions.
* [配置检查器](https://developer.lightbend.com/docs/akka-commercial-addons/current/config-checker.html) - 检查潜在的配置问题并记录建议。
* [Diagnostics Recorder](https://developer.lightbend.com/docs/akka-commercial-addons/current/diagnostics-recorder.html) &#8212; Captures configuration and system information in a format that makes it easy to troubleshoot issues during development and production.
* [诊断记录器](https://developer.lightbend.com/docs/akka-commercial-addons/current/diagnostics-recorder.html) - 以易于在开发和生产过程中排除故障的格式捕获配置和系统信息。
* [Thread Starvation Detector](https://developer.lightbend.com/docs/akka-commercial-addons/current/starvation-detector.html) &#8212; Monitors an Akka system dispatcher and logs warnings if it becomes unresponsive.
* [线程饥饿检测器](https://developer.lightbend.com/docs/akka-commercial-addons/current/starvation-detector.html) - 监视Akka系统调度程序，并在无响应时记录警告。

This page does not list all available modules, but overviews the main functionality and gives you an idea of the level of sophistication you can reach when you start building systems on top of Akka.

本页并未列出所有可用的模块，但概述了主要功能，并让您了解在Akka之上开始构建系统时可以达到的复杂程度。

### Actor库

The core Akka library is `akka-actor`. But, actors are used across Akka libraries, providing a consistent, integrated model that relieves you from individually solving the challenges that arise in concurrent or distributed system design. From a birds-eye view,actors are a programming paradigm that takes encapsulation, one of the pillars of OOP, to its extreme.Unlike objects, actors encapsulate not only their
state but their execution. Communication with actors is not via method calls but by passing messages. While this difference may seem minor, it is actually what allows us to break clean from the limitations of OOP when it comes to concurrency and remote communication. Don’t worry if this description feels too high level to fully grasp yet, in the next chapter we will explain actors in detail. For now, the important point is that this is a model that handles concurrency and distribution at the fundamental level instead of ad hoc patched attempts to bring these features to OOP.

Akka的核心库是 `akka-actor`。但是，Actor在Akka库的基础上提供了一致的集成的模型，可以让您从单独解决并发或分布式系统设计中出现的挑战中解脱出来。从大方面看，Actor是一种编程范式，它将OOP的核心——封装发挥到了极致。与对象不同，Actor不仅封装自己的状态，还封装自己的执行。与Actor的通信不是通过方法调用，而是通过传递消息。虽然这种差异可能看起来很小，但它确实可以让我们在并发和远程通信方面打破OOP的局限。如果这个描述感觉太高深而不能完全掌握，不要担心，下一章我们将详细解释Actor。目前，重要的一点是，这是一种从基本层面处理并发和分布式的模型，而不是临时修补尝试将这些功能带入OOP。

Challenges that actors solve include the following:

Actor解决的挑战包括以下内容：

* How to build and design high-performance, concurrent applications.
* 如何构建和设计高性能并发应用程序。
* How to handle errors in a multi-threaded environment.
* 如何处理多线程环境中的错误。
* How to protect my project from the pitfalls of concurrency.
* 如何保护我的项目免受并发的隐患。

### 远程处理

Remoting enables actors that live on different computers, to seamlessly exchange messages.While distributed as a JAR artifact, Remoting resembles a module more than it does a library. You enable it mostly with configuration and it has only a few APIs. Thanks to the actor model, a remote and local message send looks exactly the same. The patterns that you use on local systems translate directly to remote systems.You will rarely need to use Remoting directly, but it provides the foundation on which the Cluster subsystem is built.

远程处理让位于不同计算机上的Actor能够无缝地交换消息。虽然以JAR包形式，但Remoting相比于库其更类似于模块。您主要通过配置启用它，它只有少数几个API。多亏了Actor模型，远程和本地消息发送看起来完全一样。您在本地系统上使用的模式可直接转换到远程系统。您很少需要直接使用Remoting，但它提供了构建Cluster子系统的基础。

Challenges Remoting solves include the following:

Remoting解决的挑战包括以下内容

* How to address actor systems living on remote hosts.
* 如何定位在远程主机上的Actor系统。
* How to address individual actors on remote actor systems.
* 如何定位远程Actor系统上的单个Actor。
* How to turn messages to bytes on the wire.
* 如何将消息转换为线路上的字节。
* How to manage low-level, network connections (and reconnections) between hosts, detect crashed actor systems and hosts,all transparently.
* 如何透明的管理主机之间的低级别网络连接（和重新连接）、检测崩溃。
* How to multiplex communications from an unrelated set of actors on the same network connection, all transparently.
* 如何透明的在同一网络连接上多路复用来自不相关Actor的通信。

### 集群

If you have a set of actor systems that cooperate to solve some business problem, then you likely want to manage these set of systems in a disciplined way. While Remoting solves the problem of addressing and communicating with components of remote systems, Clustering gives you the ability to organize these into a "meta-system" tied together by a membership protocol. **In most cases, you want to use the Cluster module instead of using Remoting directly.**Clustering provides an additional set of services on top of Remoting that most real world applications need.

如果你有一套Actor系统来合作解决一些业务问题，那么你可能希望以一种严谨的方式来管理这些系统。虽然Remoting解决了寻址和与远程系统组件进行通信的问题，但集群能够使您将这些组织成由成员协议绑定在一起的“元系统”。**在大多数情况下，您希望使用集群模块而不是直接使用Remoting。**集群基于Remoting为大多数真实世界的应用程序提供了更多的服务。

Challenges the Cluster module solves include the following:

集群模块解决的问题包括：

* How to maintain a set of actor systems (a cluster) that can communicate with each other and consider each other as part of the cluster.
* 如何维护一组可以相互通信并将对方视为集群一部分的Actor系统。
* How to introduce a new system safely to the set of already existing members.
* 如何安全地将新系统引入已有成员的集群中。
* How to reliably detect systems that are temporarily unreachable.
* 如何可靠地检测暂时无法访问的系统。
* How to remove failed hosts/systems (or scale down the system) so that all remaining members agree on the remaining subset of the cluster.
* 如何删除发生故障的主机/系统（或缩小系统），以便其余的成员仍旧同意在同一个集群子集中。
* How to distribute computations among the current set of members.
* 如何在当前集群之间分配计算。
* How to designate members of the cluster to a certain role, in other words, to provide certain services and not others.
* 如何将群集成员指定为某个具体角色，换言之，提供某项具体服务。

### 群集分片

Sharding helps to solve the problem of distributing a set of actors among members of an Akka cluster.
Sharding is a pattern that mostly used together with Persistence to balance a large set of persistent entities
(backed by actors) to members of a cluster and also migrate them to other nodes when members crash or leave.

分片有助于解决在Akka集群成员中分配一组Actor的问题。分片（Sharding）是一种模式，主要与持久化一起使用来在集群成员中平衡持久化实体（由Actor支持），并在成员节点崩溃或退出时将其迁移到其他节点。

Challenges that Sharding solves include the following:

分片解决的问题包括以下几点：

* How to model and scale out a large set of stateful entities on a set of systems.
* 如何在一组系统上建模和扩展大量的状态实体。
* How to ensure that entities in the cluster are distributed properly so that load is properly balanced across the machines.
* 如何确保群集中的实体正确分布，以便负载在各台计算机之间进行适当平衡。
* How to ensure migrating entities from a crashed system without losing the state.
* 如何确保从崩溃的系统迁移实体而不会丢失状态。
* How to ensure that an entity does not exist on multiple systems at the same time and hence keeps consistent.
* 如何确保一个实体不会同时存在于多个系统上，从而保持一致。

### 集群单例

A common (in fact, a bit too common) use case in distributed systems is to have a single entity responsible
for a given task which is shared among other members of the cluster and migrated if the host system fails.
While this undeniably introduces a common bottleneck for the whole cluster that limits scaling,
there are scenarios where the use of this pattern is unavoidable. Cluster singleton allows a cluster to select an actor system which will host a particular actor while other systems can always access said service independently from where it is.

分布式系统中常见的（实际上有点太常见）用例是让一个实体负责给定的任务，该任务由群集的其他成员共享并在主机系统出现故障时进行迁移。虽然这无可否认地给整个集群带来了一个限制扩展的瓶颈，但也存在这种模式的不可避免的使用情况。集群单例机制允许集群选择将其托管于特定Actor系统，而其他Actor系统始终可以位置透明的访问该服务。

The Singleton module can be used to solve these challenges:

单例模块可以用来解决这些问题：

* How to ensure that only one instance of a service is running in the whole cluster.
* 如何确保整个群集中只有一个服务实例正在运行。
* How to ensure that the service is up even if the system hosting it currently crashes or shuts down during the process of scaling down.
* 如何确保服务仍在运行，即使托管它的系统正处在崩溃或关闭过程中。
* How to reach this instance from any member of the cluster assuming that it can migrate to other systems over time.
* 如何在服务可以随时间迁移到其他系统的情况下，也能从集群的任何成员访问此服务实例。

### 集群发布 - 订阅

For coordination among systems, it is often necessary to distribute messages to all, or one system of a set of interested systems in a cluster. This pattern is usually called publish-subscribe and this module solves this exact problem. It is possible to broadcast messages to all subscribers of a topic or send a message to an arbitrary actor that has expressed interest.

对于系统之间的协调，通常需要将消息分发给群集中的所有成员或一组对此感兴趣的成员。这种模式通常称为发布 - 订阅，集群发布-订阅模块解决了这个问题。它可以将消息广播给主题的所有订阅者，或者将消息发送给任何表示对此有兴趣的Actor。

Cluster Publish-Subscribe is intended to solve the following challenges:

集群发布订阅旨在解决以下问题：

* How to broadcast messages to an interested set of parties in a cluster.
* 如何将消息广播到群集中感兴趣的一组参与者。
* How to send a message to a member from an interested set of parties in a cluster.
* 如何将消息由群集中感兴趣的一组参与者发送给某一成员。
* How to subscribe and unsubscribe for events of a certain topic in the cluster.
* 如何订阅和取消订阅群集中某个主题的事件。

### 持久化

Just like objects in OOP, actors keep their state in volatile memory. Once the system is shut down, gracefully or because of a crash, all data that was in memory is lost. Persistence provides patterns to enable actors to persist events that lead to their current state. Upon startup, events can be replayed to restore the state of the entity hosted by the actor. The event stream can be queried and fed into additional processing pipelines (an external Big Data cluster for example) or alternate views (like reports).

就像面向对象中的对象一样，演员将自己的状态保存在易失内存中。一旦系统关闭，正常或因为崩溃，内存中的所有数据都将丢失。持久性提供了模式，使演员能够坚持导致当前状态的事件。启动后，可以重播事件以恢复演员托管的实体的状态。可以查询事件流并将其添加到其他处理管道（例如外部大数据群集）或备用视图（如报告）中。

Persistence tackles the following challenges:

持久化解决以下问题：

* How to restore the state of an entity/actor when system restarts or crashes.
* 如何在系统重新启动或崩溃时恢复实体/Actor的状态。
* How to implement a [CQRS system](https://msdn.microsoft.com/en-us/library/jj591573.aspx).
* 如何实施[CQRS系统](https://msdn.microsoft.com/en-us/library/jj591573.aspx)。
* How to ensure reliable delivery of messages in face of network errors and system crashes.
* 如何确保在网络错误和系统崩溃的情况下可靠地传递消息。
* How to introspect domain events that have led an entity to its current state.
* 如何对导致实体到其当前状态的域事件进行内省。
* How to leverage [Event Sourcing](https://martinfowler.com/eaaDev/EventSourcing.html) in your application to support long-running processes while the project continues to evolve.
* 如何在项目不断迭代演化的同时，利用[事件驱动](https://martinfowler.com/eaaDev/EventSourcing.html)来支持长期运行的处理流程。

### 分布式数据

In situations where eventual consistency is acceptable, it is possible to share data between nodes in
an Akka Cluster and accept both reads and writes even in the face of cluster partitions. This can be
achieved using [Conflict Free Replicated Data Types](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type) (CRDTs), where writes on different nodes can
happen concurrently and are merged in a predictable way afterward. The Distributed Data module
provides infrastructure to share data and a number of useful data types.

在可以接受最终一致性的情况下，Akka集群中的节点之间共享数据，即使面对集群分区也可以接受读取和写入。这可以通过使用[无冲突复制数据类型](https://en.wikipedia.org/wiki/Conflict-free_replicated_data_type)（CRDT）来实现，其中不同节点上的写入可以并行发生并在之后能以可预测的方式合并。分布式模块提供了共享数据的底层机制和一些有用的数据类型。

Distributed Data is intended to solve the following challenges:

分布式数据旨在解决以下问题：

* How to accept writes even in the face of cluster partitions.
* 如何在面对集群分区的情况下接受写入。
* How to share data while at the same time ensuring low-latency local read and write access.
* 如何在确保低延迟的本地读写情况下共享数据。

### 流

Actors are a fundamental model for concurrency, but there are common patterns where their use requires the user to implement the same pattern over and over. Very common is the scenario where a chain, or graph, of actors, need to process a potentially large, or infinite, stream of sequential events and properly coordinate resource usage so that faster processing stages does not overwhelm slower ones in the chain or  graph. Streams provide a higher-level abstraction on top of actors that simplifies writing such processing networks, handling all the fine details in the background and providing a safe, typed, composable programming model. Streams is also an implementation of the [Reactive Streams standard](http://www.reactive-streams.org) which enables integration with all third party implementations of that standard.

Actor是并发的基本模型，但有一些常见的模式，其使用要求用户反复实施相同的模式。非常常见的情况是，基于Actor的链或图需要处理很大或无限的连续事件流，并需要正确协调资源使用情况，以便处理快的环节不会对处理慢的环节造成压力。Streams基于Actor提供了更高级别的抽象，简化了编写处理网络，处理背景中的所有细节，并提供安全的，类型化的，可组合的编程模型。Streams也是[Reactive Streams标准](http://www.reactive-streams.org/)的一个实现，它可以与该标准的所有第三方实现集成。

Streams solve the following challenges:

流解决了以下问题：

* How to handle streams of events or large datasets with high performance, exploiting concurrency and keep resource usage tight.
* 如何处理高性能的事件流或大型数据集，充分利用并发性并保持资源利用率。
* How to assemble reusable pieces of event/data processing into flexible pipelines.
* 如何将可重复使用的事件/数据处理单元组合成灵活的管道。
* How to connect asynchronous services in a flexible way to each other, and have good performance.
* 如何以一种灵活的方式将异步服务彼此连接，并使之具有良好的性能。
* How to provide or consume Reactive Streams compliant interfaces to interface with a third party library.
* 如何提供或使用Reactive Streams兼容接口来与第三方库进行对接。

### HTTP

The de facto standard for providing APIs remotely, internal or external, is [HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol). Akka provides a library to construct or consume such HTTP services by giving a set of tools to create HTTP services (and serve them) and a client that can be used to consume other services. These tools are particularly suited to streaming in and out a large set of data or real-time events by leveraging the underlying model of Akka Streams.

[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol)是实际上对远程、内部、外部提供API的标准形式。Akka通过类库提供了一系列工具来创建和消费HTTP服务以及一个客户端来使用其他相关服务。这些工具特别适合通过利用基于Akka Streams的模型来处理大量数据或实时事件流入和流出。

Some of the challenges that HTTP tackles:

HTTP处理的一些问题：

* How to expose services of a system or cluster to the external world via an HTTP API in a performant way.
* 如何以高性能的方式通过HTTP API向外部世界公开系统或集群的服务。
* How to stream large datasets in and out of a system using HTTP.
* 如何使用HTTP处理大数据集的流入流出。
* How to stream live events in and out of a system using HTTP.
* 如何使用HTTP处理实时事件的流入流出。

### Example of module use

Akka modules integrate together seamlessly. For example, think of a large set of stateful business objects, such as documents or shopping carts, that website users access. If you model these as sharded entities, using Sharding and Persistence, they will be balanced across a cluster that you can scale out on-demand. They will be available during spikes that come from advertising campaigns or before holidays will be handled, even if some systems crash. You can also easily take the real-time stream of domain events with Persistence Query and use Streams to pipe them into a streaming Fast Data engine. Then, take the output of that engine as a Stream, manipulate it using Akka Streams operators and expose it as web socket connections served by a load balanced set of HTTP servers hosted by your cluster to power your real-time business analytics tool.

Akka模块无缝集成在一起。例如，考虑网站用户访问的大量有状态业务对象，例如文档或购物车。如果使用分片实体，则用到分片和持久化模块，它们会在集群中分布，并按需扩展。即使某些系统崩溃，它们也能在广告系列或节假日之前处理的峰值期间有效。您还可以使用持久性查询轻松获取实时的领域事件流，并使用Streams将它们导入流式快速数据引擎。然后，将该引擎的输出视为Stream，使用Akka Streams操作符对其进行操作，并将其公开为由集群托管的负载均衡HTTP服务器集合提供的Web套接字连接，以便为您的实时业务分析工具提供支持。(**这一段看英文吧，实在不好翻译！！**)

We hope this preview caught your interest! The next topic introduces the example application we will build in the tutorial portion of this guide. 

我们希望这个预览引起你的兴趣！下一个主题介绍我们将在本指南的教程部分中构建的示例应用程序。