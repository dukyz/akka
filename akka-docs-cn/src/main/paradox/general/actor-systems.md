# Actor系统

Actors are objects which encapsulate state and behavior, they communicate exclusively by exchanging messages which are placed into the recipient’s mailbox. In a sense, actors are the most stringent form of object-oriented programming, but it serves better to view them as persons: while modeling a solution with actors, envision a group of people and assign sub-tasks to them,arrange their functions into an organizational structure and think about how to escalate failure (all with the benefit of not actually dealing with people,which means that we need not concern ourselves with their emotional state or moral issues). The result can then serve as a mental scaffolding for building the software implementation.

参与者是封装状态和行为的对象，它们通过交换放置在收件人邮箱中的邮件进行通信。从某种意义上讲，演员是面向对象编程的最严格形式，但是更好地将他们视为人：在与演员建模的解决方案中，设想一群人并为他们分配子任务，将他们的功能安排到一个组织结构，并考虑如何升级失败（所有的好处都不在于与人打交道，这意味着我们不必关心自己的情绪状态或道德问题）。然后，结果可以作为构建软件实施的心理脚手架。

@@@ note

An ActorSystem is a heavyweight structure that will allocate 1…N Threads,so create one per logical application.

ActorSystem是一个重量级结构，将分配1 ... N个线程，因此每个逻辑应用程序都要创建一个。

@@@

## 层次结构

Like in an economic organization, actors naturally form hierarchies. One actor,which is to oversee a certain function in the program might want to split up its task into smaller, more manageable pieces. For this purpose it starts child actors which it supervises. While the details of supervision are explained @ref:[here](supervision.md), we shall concentrate on the underlying concepts in this section. The only prerequisite is to know that each actor has exactly one supervisor, which is the actor that created it.

就像在一个经济组织中一样，演员自然会形成等级制。一个负责监督节目中某个功能的演员可能希望将其任务分成更小，更易于管理的部分。为此目的，它开始监督儿童演员。虽然[这里](https://doc.akka.io/docs/akka/current/general/supervision.html)解释了监督的细节，但我们将集中讨论本节中的基本概念。唯一的先决条件是知道每个演员都有一个主管，这是创建它的主角。

The quintessential feature of actor systems is that tasks are split up and delegated until they become small enough to be handled in one piece. In doing so, not only is the task itself clearly structured, but the resulting actors can be reasoned about in terms of which messages they should process, how they should react normally and how failure should be handled. If one actor does not have the means for dealing with a certain situation, it sends a corresponding failure message to its supervisor, asking for help. The recursive structure then allows to handle failure at the right level.

演员系统的典型特征是将任务分解并分配，直到他们变得足够小以至于能够一件件地处理。在这样做的过程中，不仅任务本身结构清晰，而且所产生的参与者可以根据他们应该处理的消息，他们应该如何正常应对以及应该如何处理失败来推理。如果一个演员没有处理某种情况的手段，它会向其主管发出相应的失败信息，寻求帮助。递归结构允许在正确的级别处理失败。

Compare this to layered software design which easily devolves into defensive programming with the aim of not leaking any failure out: if the problem is communicated to the right person, a better solution can be found than if trying to keep everything “under the carpet”.

将其与分层软件设计进行比较，该设计很容易进入防御性编程，目的在于不会泄漏任何故障：如果将问题传达给合适的人，可以找到一个更好的解决方案，而不是将所有内容都放在“地毯下”。

Now, the difficulty in designing such a system is how to decide who should supervise what. There is of course no single best solution, but there are a few guidelines which might be helpful:

现在，设计这样一个系统的难度在于如何决定谁应该监督什么。没有单一的最佳解决方案，但有一些可能有用的指导方针：

 * If one actor manages the work another actor is doing, e.g. by passing on sub-tasks, then the manager should supervise the child. The reason is that the manager knows which kind of failures are expected and how to handle them.
 * 如果一个演员管理另一个演员正在做的工作，例如通过传递子任务，那么经理应该监督孩子。原因是经理知道哪种故障是预期的以及如何处理它们。
 * If one actor carries very important data (i.e. its state shall not be lost if avoidable), this actor should source out any possibly dangerous sub-tasks to children it supervises and handle failures of these children as appropriate. Depending on the nature of the requests, it may be best to create a new child for each request, which simplifies state management for collecting the replies. This is known as the “Error Kernel Pattern” from Erlang.
 * 如果一个演员携带非常重要的数据（即如果可避免的话，其状态不会丢失），则该演员应该向其监督的儿童寻找任何可能危险的子任务，并酌情处理这些儿童的失败。根据请求的性质，最好为每个请求创建一个新子项，这样可以简化收集回复的状态管理。这被称为Erlang的“错误内核模式”。
 * If one actor depends on another actor for carrying out its duty, it should watch that other actor’s liveness and act upon receiving a termination notice. This is different from supervision, as the watching party has no influence on the supervisor strategy, and it should be noted that a functional dependency alone is not a criterion for deciding where to place a certain child actor in the hierarchy.
 * 如果一个演员依靠另一个演员履行其职责，则应该观看其他演员的活跃并在收到终止通知后采取行动。这与监督不同，因为观察方对监督者策略没有影响，应该指出的是，功能依赖本身并不是决定将某个儿童参与者置于何处的标准。

There are of course always exceptions to these rules, but no matter whether you follow the rules or break them, you should always have a reason.

这些规则总是有例外，但无论你遵守规则还是破坏规则，你都应该有一个理由。

## 配置容器(翻译的有点怪，再看看)

The actor system as a collaborating ensemble of actors is the natural unit for managing shared facilities like scheduling services, configuration, logging, etc. Several actor systems with different configuration may co-exist within the same JVM without problems, there is no global shared state within Akka itself.Couple this with the transparent communication between actor systems—within one node or across a network connection—to see that actor systems themselves can be used as building blocks in a functional hierarchy.

作为演员协作集的演员系统是管理共享设施的自然单元，如调度服务，配置，日志记录等。具有不同配置的几个参与者系统可以共存于相同的JVM中，没有问题，没有全局共享状态在Akka内部。将这与演员系统（在一个节点内或通过网络连接）之间的透明通信结合起来，可以看到演员系统本身可以用作功能层次结构中的构建块。

## Actor最佳实践

  1. Actors should be like nice co-workers: do their job efficiently without bothering everyone else needlessly and avoid hogging resources. Translated to programming this means to process events and generate responses (or more requests) in an event-driven manner. Actors should not block (i.e. passively wait while occupying a Thread) on some external entity—which might be a lock, a network socket, etc.—unless it is unavoidable; in the latter case see below.
  2. 演员应该像个好同事一样：高效地完成工作，而不必麻烦其他人，避免浪费资源。转化为编程意味着以事件驱动的方式处理事件并生成响应（或更多请求）。参与者不应该在某个外部实体（可能是锁，网络套接字等）上阻塞（即占用线程时被动等待），除非它是不可避免的; 在后一种情况下见下文。
  3. Do not pass mutable objects between actors. In order to ensure that, prefer immutable messages. If the encapsulation of actors is broken by exposing their mutable state to the outside, you are back in normal Java concurrency land with all the drawbacks.
  4. 不要在actor之间传递可变对象。为了确保，喜欢不可变的消息。如果演员的封装通过将外部可变状态暴露给外部来破坏，那么您又回到了正常的Java并发领域，带来了所有的缺点。
  5. Actors are made to be containers for behavior and state, embracing this means to not routinely send behavior within messages (which may be tempting using Scala closures). One of the risks is to accidentally share mutable state between actors, and this violation of the actor model unfortunately
  breaks all the properties which make programming in actors such a nice experience.
  6. 行为者被定义为行为和状态的容器，拥抱这种意味着不会在消息中常规发送行为（这可能是使用Scala关闭的诱惑）。其中一个风险是在演员之间意外地共享可变状态，并且这种对演员模型的违反不幸地打破了在演员中编程这样一个很好的体验的所有属性。
  7. Top-level actors are the innermost part of your Error Kernel, so create them sparingly and prefer truly hierarchical systems. This has benefits with respect to fault-handling (both considering the granularity of configuration and the performance) and it also reduces the strain on the guardian actor,which is a single point of contention if over-used.
  8. 顶级角色是您的错误内核的最核心部分，因此请谨慎地创建它们，并且更喜欢真正的分层系统。这对于故障处理（无论是考虑配置的粒度还是性能）都有好处，并且还减少了监护人角色的压力，如果过度使用，这是一个单一的争用点。

## 您不需关心的

An actor system manages the resources it is configured to use in order to run the actors which it contains. There may be millions of actors within one such system, after all the mantra is to view them as abundant and they weigh in at an overhead of only roughly 300 bytes per instance. Naturally, the exact order in which messages are processed in large systems is not controllable by the application author, but this is also not intended. Take a step back and relax while Akka does the heavy lifting under the hood.

参与者系统管理它配置用来运行其包含的参与者的资源。在一个这样的系统中可能有成千上万的演员，毕竟这个咒语要把它们看作是丰富的，而且它们的权重只是每个实例只有大约300字节的开销。自然地，在大型系统中处理消息的确切顺序不能由应用程序作者控制，但这也不是意图。退后一步，放松一下，而阿卡在引擎盖下举重。

## 终止ActorSystem

When you know everything is done for your application, you can call the `terminate` method of `ActorSystem`. That will stop the guardian actor, which in turn will recursively stop all its child actors, the system guardian.

当你知道你的应用程序完成了一切时，你可以调用`terminate`方法`ActorSystem`。这将阻止监护人演员，反过来将递归地停止其所有的儿童演员，系统监护人。

If you want to execute some operations while terminating `ActorSystem`,look at @ref:[`CoordinatedShutdown`](../actors.md#coordinated-shutdown).

如果你想在终止时执行一些操作`ActorSystem`，请看[`CoordinatedShutdown`](https://doc.akka.io/docs/akka/current/actors.html#coordinated-shutdown)