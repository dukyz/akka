# 位置透明

The previous section describes how actor paths are used to enable location transparency. This special feature deserves some extra explanation, because the related term “transparent remoting” was used quite differently in the context of programming languages, platforms and technologies.

上一章节描述了如何利用Actor路径来达到位置透明的效果。这个特殊的特性需要额外的解释一下，因为在编程语言，平台和技术方面，术语“transparent remoting”的使用方式完全不同。

## 默认分布

Everything in Akka is designed to work in a distributed setting: all interactions of actors use purely message passing and everything is asynchronous. This effort has been undertaken to ensure that all functions are
available equally when running within a single JVM or on a cluster of hundreds of machines. The key for enabling this is to go from remote to local by way of optimization instead of trying to go from local to remote by way of generalization. See [this classic paper](http://doc.akka.io/docs/misc/smli_tr-94-29.pdf) for a detailed discussion on why the second approach is bound to fail.

Akka中的所有内容都是为在分布式环境中工作而设计的：Actor的所有交互都使用纯粹的消息传递，并且所有内容都是异步的。这一努力的目的是确保不管在单JVM还是数百台机器的集群中运行时，所有功能均可用。实现这一目标的关键是通过从远程到本地的优化，而不是通过从本地到远程的泛化。请参阅[此经典文章](http://doc.akka.io/docs/misc/smli_tr-94-29.pdf)，里面详细讨论了为什么第二种方法肯定会失败。

## 打破透明的方式（Ways in which Transparency is Broken）

What is true of Akka need not be true of the application which uses it, since designing for distributed execution poses some restrictions on what is possible. The most obvious one is that all messages sent over the wire must be serializable. While being a little less obvious this includes closures which are used as actor factories (i.e. within `Props`) if the actor is to be created on a remote node.

对Akka来说对的东西，对于使用Akka的应用程序来说则不一定，因为分布式执行的设计在可能性上有一定的局限。最明显的是消息通过线路发送时必须是可序列化的。如果Actor被在远程节点创建时，这过程也包含了被当做Actor工厂使用的闭包（在`Props`内），这点就稍微不明显了 (**草草草！！！**)。

Another consequence is that everything needs to be aware of all interactions being fully asynchronous, which in a computer network might mean that it may take several minutes for a message to reach its recipient (depending on configuration). It also means that the probability for a message to be lost is
much higher than within one JVM, where it is close to zero (still: no hard guarantee!).

另一个结果是，所有事情都要意识到所有的交互都是完全异步的，这在计算机网络中可能意味着消息可能需要几分钟才能到达其接收者（取决于配置）。这也意味着消息丢失的概率要比在一个JVM内接近零（仍然没有硬性保证！）的概率要高。

## 如何使用Remoting?

We took the idea of transparency to the limit in that there is nearly no API for the remoting layer of Akka: it is purely driven by configuration. Just write your application according to the principles outlined in the previous sections, then specify remote deployment of actor sub-trees in the configuration file. This way, your application can be scaled out without having to touch the code. The only piece of the API which allows programmatic influence on remote deployment is that `Props` contain a field which may be set to a specific `Deploy` instance; this has the same effect as putting an equivalent deployment into the configuration file (if both are given, configuration file wins).

透明可以做到什么程度？答案是几乎没有用于Akka远程层的API：它完全由配置驱动。只需根据前几节中概述的原则编写应用程序，然后在配置文件中指定Actor子树的远程部署。这样，你的应用程序不需接触代码就可以被扩展。唯一会对远程部署造成程序化影响的是`Props`API，它含有可以指定`Deploy`实例的参数; 这与将等同的部署放入配置文件中一样（如果两者都有，配置文件获胜）。

<a id="symmetric-communication"></a>
## 点对点 vs. 客户端对服务器

Akka Remoting is a communication module for connecting actor systems in a peer-to-peer fashion,and it is the foundation for Akka Clustering. The design of remoting is driven by two (related) design decisions:

Akka Remoting是以点到点的方式连接Actor系统的通信模块，它是Akka Clustering的基础。远程处理的设计被两个（相关）设计决定所驱动：

  1. Communication between involved systems is symmetric: if a system A can connect to a system B
    then system B must also be able to connect to system A independently.
  2. 相关系统之间的通信是对称的：如果系统A可以连接到系统B，则系统B也必须能够独立地连接到系统A.
  3. The role of the communicating systems are symmetric in regards to connection patterns: there is no system that only accepts connections, and there is no system that only initiates connections.
  4. 通信系统的角色对于连接模式是对称的：没有只接受连接的系统，也没有只发起连接的系统。

The consequence of these decisions is that it is not possible to safely create pure client-server setups with predefined roles (violates assumption 2).For client-server setups it is better to use HTTP or Akka I/O.

上述决定的结论是，不可能利用预定义的规则（违反假设2）创建纯C/S模式。对于C/S模式，最好使用HTTP或Akka I / O。

**Important**: Using setups involving Network Address Translation, Load Balancers or Docker containers violates assumption 1, unless additional steps are taken in the network configuration to allow symmetric communication between involved systems.In such situations Akka can be configured to bind to a different network address than the one used for establishing connections between Akka nodes.See @ref:[Akka behind NAT or in a Docker container](../remoting.md#remote-configuration-nat).

**重要提示**：当涉及使用网络地址转换，负载均衡或Docker容器时，会违反假设1，除非在网络配置中采取额外手段以允许系统之间的对称通信。在这种情况下，可以为Akka配置与建立Akka节点之间连接地址不同的网络地址。请参阅[NAT后面或Docker容器中的Akka](https://doc.akka.io/docs/akka/current/remoting.html#remote-configuration-nat)。

## 简述利用路由功能扩展

In addition to being able to run different parts of an actor system on different nodes of a cluster, it is also possible to scale up onto more cores by multiplying actor sub-trees which support parallelization (think for example a search engine processing different queries in parallel). The clones can then be routed to in different fashions, e.g. round-robin. The only thing necessary to achieve this is that the developer needs to declare a certain actor as “withRouter”, then—in its stead—a router actor will be created which will spawn
up a configurable number of children of the desired type and route to them in the configured fashion. Once such a router has been declared, its configuration can be freely overridden from the configuration file, including mixing it with the remote deployment of (some of) the children. Read more about this in @ref:[Routing](../routing.md).

除了能够在一个集群的不同节点上运行Actor系统的不同部分之外，还可以通过支持并行化的多Actor子树来扩展到更多的核心（想象搜索引擎并行处理不同的查询）。然后克隆的子树可以被不同的方式路由，例如循环。唯一需要做到是开发人员要将某个Actor声明为“withRouter”，然后这个路由Actor将生成一系列所需类型的子节点并以配置的方式将信息路由给他们。一旦声明了这样的路由器，就可以自由的从配置文件中覆盖其配置，包括与子节点的远程部署混合。阅读更多关于这个[路由](https://doc.akka.io/docs/akka/current/routing.html)。