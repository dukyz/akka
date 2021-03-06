# Akka简介

Welcome to Akka, a set of open-source libraries for designing scalable, 
resilient systems that span processor cores and networks. 
Akka allows you to focus on meeting business needs instead of writing low-level code to 
provide reliable behavior, fault tolerance, and high performance.

欢迎使用akka，akka是一套用来设计基于多核心、跨网络、可扩展、高弹性系统开源工具集。Akka能让你专注于业务需求的实现，
而不是为了容错性、高可用、可靠的行为去写底层的代码。

Many common practices and accepted programming models do not address important challenges
inherent in designing systems for modern computer architectures. To be
successful, distributed systems must cope in an environment where components
crash without responding, messages get lost without a trace on the wire, and
network latency fluctuates. These problems occur regularly in carefully managed
intra-datacenter environments - even more so in virtualized architectures.

通常的实践和公认的编程模型并没有解决在现代计算架构中固有的重要挑战。为了获得成功，分布式系统必须处理环境中的组件崩溃无响应、
信息丢失无迹可寻、网络延时波动不稳定。这些问题在被仔细管理着的数据中心环境中常常发生，在虚拟架构中更是如此。

To help you deal with these realities, Akka provides:

为了帮助您解决这些现实问题，Akka提供了：

 * Multi-threaded behavior without the use of low-level concurrency constructs like
   atomics or locks &#8212; relieving you from even thinking about memory visibility issues.
   
 * 多线程行为，并非采用锁或原子操作这样底层的并发结构，甚至让您不用去考虑内存的可见性问题。
 
 * Transparent remote communication between systems and their components &#8212; 
 relieving you from writing and maintaining difficult networking code.
 
 * 让系统组件之间的远程通讯透明化，让您远离编写和维护不同的网络代码。 
 
 * A clustered, high-availability architecture that is elastic, scales in or out, on demand &#8212; 
 enabling you to deliver a truly reactive system.
 
 * 一个集群化的、高可用的架构，按需弹性扩展，从而使您能提供一个真正的响应式系统。

Akka's use of the actor model provides a level of abstraction that makes it
easier to write correct concurrent, parallel and distributed systems. The actor
model spans the full set of Akka libraries, providing you with a consistent way
of understanding and using them. Thus, Akka offers a depth of integration that
you cannot achieve by picking libraries to solve individual problems and trying
to piece them together.

Akka使用Actor模型进行了一层抽象，借此来让编写正确的并行、并发、分布式系统更加容易。Actor模型在Akka工具集中贯穿始终，
并且更加便利的助你使用和理解Akka。所以Akka提供了深度的集成，你不需要针对个别问题考虑如何使用类库以及如何集成他们。

By learning Akka and how to use the actor model, you will gain access to a vast
and deep set of tools that solve difficult distributed/parallel systems problems
in a uniform programming model where everything fits together tightly and
efficiently.

通过学习Akka和学习如何使用Actor模型，您将获得大量的深入性的工具集，这种工具内聚而高效，
并以统一的编程模型来解决困难的分布式/并行系统的问题。

## 如何开始


If this is your first experience with Akka, we recommend that you start by running a simple Hello World project. 
See the @scala[[Quickstart Guide](http://developer.lightbend.com/guides/akka-quickstart-scala)] @java[[Quickstart Guide](http://developer.lightbend.com/guides/akka-quickstart-java)] for
instructions on downloading and running the Hello World example. 
The *Quickstart* guide walks you through example code that introduces how to define actor systems, 
actors, and messages as well as how to use the test module and logging. 
Within 30 minutes, you should be able to run the Hello World example and learn how it is constructed.

如果这是你第一次接触Akka，我们建议您先运行一个简单的Hello World项目。参照@scala[[Quickstart Guide](http://developer.lightbend.com/guides/akka-quickstart-scala)] @java[[Quickstart Guide](http://developer.lightbend.com/guides/akka-quickstart-java)] 
的指示下载并运行Hello World例子。 *快速开始* 带着您浏览例子代码，向您介绍如何定义Actor系统、Actor和消息，也让您知道如何使用测试模块和日志。
在30分钟内，您就能知道如何运行Hello World实例以及知道他是怎么构建的。


This *Getting Started* guide provides the next level of information. 
It covers why the actor model fits the needs of modern distributed systems and includes a tutorial that will help further your knowledge of Akka. 
Topics include:

这个 *开始学习* 提供了接下来的信息。包括为什么Actor模型适用于现代的分布式系统，以及为了更进一步的学习而预先需要了解的知识，主题包括：

* @ref:[Why modern systems need a new programming model](actors-motivation.md)
* @ref:[How the actor model meets the needs of concurrent, distributed systems](actors-intro.md)
* @ref:[Overview of Akka libraries and modules](modules.md)
* A @ref:[more complex example](tutorial.md) that builds on the Hello World example to illustrate common Akka patterns.
* @ref:[more complex example](tutorial.md) 这个基于Hello World例子，介绍了Akka的常规模式
