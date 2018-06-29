# Actor系统

Actors are objects which encapsulate state and behavior, they communicate exclusively by exchanging messages which are placed into the recipient’s mailbox. In a sense, actors are the most stringent form of object-oriented programming, but it serves better to view them as persons: while modeling a solution with actors, envision a group of people and assign sub-tasks to them,arrange their functions into an organizational structure and think about how to escalate failure (all with the benefit of not actually dealing with people,which means that we need not concern ourselves with their emotional state or moral issues). The result can then serve as a mental scaffolding for building the software implementation.

Actor本质是封装行为和状态的对象，它们专门通过以消息交换的方式进行通信，消息会搁置在收件人的信箱中。从某种意义上讲，Actor是面向对象编程的最严格形式，但更好的是将他们视为人类：在Actor建模的解决方案中，设想有一群人，并为他们分配子任务，将他们按功能安排到一个组织结构中，并考虑如何处理失败（好在这不是真的在与人打交道，所以我们不必关心情绪状态或道德问题）。而这个模型可作为构建软件实现的心理框架。

@@@ note

An ActorSystem is a heavyweight structure that will allocate 1…N Threads,so create one per logical application.

ActorSystem是一个利用多线程的重量级结构，因此每个逻辑应用程序只需创建一个。

@@@

## 层次结构

Like in an economic organization, actors naturally form hierarchies. One actor,which is to oversee a certain function in the program might want to split up its task into smaller, more manageable pieces. For this purpose it starts child actors which it supervises. While the details of supervision are explained @ref:[here](supervision.md), we shall concentrate on the underlying concepts in this section. The only prerequisite is to know that each actor has exactly one supervisor, which is the actor that created it.

就像在一个经济组织中一样，Actor会自然形成层级。一个负责程序中某个功能的Actor可能希望将其任务分成更小、更易于管理的部分。为此，它启动了可监督的子级Actor。虽然[这里](https://doc.akka.io/docs/akka/current/general/supervision.html)解释了监督的细节，但我们将集中讨论本节的基本概念。唯一要提前知道的是每个Actor都有一个主管，也就是创建它的Actor。

The quintessential feature of actor systems is that tasks are split up and delegated until they become small enough to be handled in one piece. In doing so, not only is the task itself clearly structured, but the resulting actors can be reasoned about in terms of which messages they should process, how they should react normally and how failure should be handled. If one actor does not have the means for dealing with a certain situation, it sends a corresponding failure message to its supervisor, asking for help. The recursive structure then allows to handle failure at the right level.

Actor系统的精髓是将任务分解并分配，直到他们足够小以便于能够一件件地处理。在这样做的过程中，不仅任务本身结构清晰，而且可以根据应该如何处理的消息、如何正常应对以及如何处理失败来创建Actor。如果一个Actor没有处理某种任务的手段，那么它会向其主管发出失败信息寻求帮助。而递归的结构体系使得系统可以在合适的层级处理失败。

Compare this to layered software design which easily devolves into defensive programming with the aim of not leaking any failure out: if the problem is communicated to the right person, a better solution can be found than if trying to keep everything “under the carpet”.

与分层软件设计进行比较，该设计很容易拥有防御性编程风格，一种旨在不会泄漏任何故障的编程方式：如果将问题传达给合适的人，相比于试图掩盖所有问题，这往往可以找到一个更好的解决方案。

Now, the difficulty in designing such a system is how to decide who should supervise what. There is of course no single best solution, but there are a few guidelines which might be helpful:

现在，设计这样一个系统的难点在于如何决定谁应该监督什么。这里没有最佳解决方案，但有一些可能有用的指导方针：

 * If one actor manages the work another actor is doing, e.g. by passing on sub-tasks, then the manager should supervise the child. The reason is that the manager knows which kind of failures are expected and how to handle them.
 * 比如一个Actor管理另一个Actor正在做的工作，例如，管理者监督子级Actor,向其传递子任务。因为管理者知道哪种故障是预期的以及该如何处理它们。
 * If one actor carries very important data (i.e. its state shall not be lost if avoidable), this actor should source out any possibly dangerous sub-tasks to children it supervises and handle failures of these children as appropriate. Depending on the nature of the requests, it may be best to create a new child for each request, which simplifies state management for collecting the replies. This is known as the “Error Kernel Pattern” from Erlang.
 * 比如Actor携带非常重要的数据（如果可避免异常的话，状态将不会丢失），那么这个Actor应该将潜在的危险的子任务派发给下级Actor，监督它们并酌情处理这些子级Actor的失败。根据请求的性质，最好为每个请求创建一个新的子级Actor，这样可以简化收集回复过程的状态管理。这被称为Erlang的“Error Kernel模式”。
 * If one actor depends on another actor for carrying out its duty, it should watch that other actor’s liveness and act upon receiving a termination notice. This is different from supervision, as the watching party has no influence on the supervisor strategy, and it should be noted that a functional dependency alone is not a criterion for deciding where to place a certain child actor in the hierarchy.
 * 比如一个Actor依靠另一个Actor来履行其职责，那么它应该监视其他Actor的生存状态并在收到终止通知后采取响应措施。这与监督不同，因为监视方对监督策略没有影响，应该指出的是，功能依赖并不是决定将某个子级Actor置于层级结构何处的依据。

There are of course always exceptions to these rules, but no matter whether you follow the rules or break them, you should always have a reason.

这些规则总是有例外，但无论你遵守规则还是破坏规则，你都应该有一个理由。

## 配置单元

The actor system as a collaborating ensemble of actors is the natural unit for managing shared facilities like scheduling services, configuration, logging, etc. Several actor systems with different configuration may co-exist within the same JVM without problems, there is no global shared state within Akka itself.Couple this with the transparent communication between actor systems—within one node or across a network connection—to see that actor systems themselves can be used as building blocks in a functional hierarchy.

Actor系统作为Actor的协作整体是管理共享设施如调度服务，配置，日志记录等的天然单元。不同配置的Actor系统可以共存于相同的JVM中，Akka内部没有全局共享状态。将这与Actor系统之间（在一个节点内或通过网络连接）的透明通信结合起来，可以将Actor系统本身看做是功能层次结构中的构造块。

## Actor最佳实践

    1. Actors should be like nice co-workers: do their job efficiently without bothering everyone else needlessly and avoid hogging resources. Translated to programming this means to process events and generate responses (or more requests) in an event-driven manner. Actors should not block (i.e. passively wait while occupying a Thread) on some external entity—which might be a lock, a network socket, etc.—unless it is unavoidable; in the latter case see below.
    2. Actor应该好同事一样：高效地完成工作，而不必麻烦他人，避免浪费资源。在编程中这意味着以事件驱动的方式处理事件并生成响应（或更多请求）。Actor不应该在某个外部实体（可能是锁，网络套接字等）上阻塞（占用线程时被动等待），除非它是不可避免的; 在后一种情况可见下文。
    3. Do not pass mutable objects between actors. In order to ensure that, prefer immutable messages. If the encapsulation of actors is broken by exposing their mutable state to the outside, you are back in normal Java concurrency land with all the drawbacks.
    4. 不要在Actor之间传递可变对象。使用不可变的消息来确保这点。如果通过将可变状态暴露给外部而破坏了Actor的封装，那么您又将承受常规Java并发领域所带来的所有缺点。
    5. Actors are made to be containers for behavior and state, embracing this means to not routinely send behavior within messages (which may be tempting using Scala closures). One of the risks is to accidentally share mutable state between actors, and this violation of the actor model unfortunately
      breaks all the properties which make programming in actors such a nice experience.
    6. Actor被定义为行为和状态的容器，这意味着不要在常规消息中发送行为（Scala闭包的能力）。其中一个风险是在Actor之间意外地共享可变状态，并且这种对Actor模型的违反行为会打破这些使得Actor编程体验美好的所有特性。
    7. Top-level actors are the innermost part of your Error Kernel, so create them sparingly and prefer truly hierarchical systems. This has benefits with respect to fault-handling (both considering the granularity of configuration and the performance) and it also reduces the strain on the guardian actor,which is a single point of contention if over-used.
    8. 顶级Actor是您的Error Kernel的最核心部分，因此请谨慎地创建它们，并尽可能的使用的分层系统。这有利于故障处理（考虑到配置的粒度和性能），并且还减少了监护人Actor的压力，毕竟过度使用的话，这是一个单一的争用点。

## 您不需关心的

An actor system manages the resources it is configured to use in order to run the actors which it contains. There may be millions of actors within one such system, after all the mantra is to view them as abundant and they weigh in at an overhead of only roughly 300 bytes per instance. Naturally, the exact order in which messages are processed in large systems is not controllable by the application author, but this is also not intended. Take a step back and relax while Akka does the heavy lifting under the hood.

Actor系统管理其配置用来运行其包含的Actor的资源。在一个这样的系统中可能有成千上万的Actor，总之可以把它们看作是大量的，而且它们的每个实例只有大约300字节的开销。在大型系统中应用程序的作者无法控制处理消息的确切顺序，当然这也不是故意的。退一步放轻松，Akka会帮你处理大量的底层工作。

## 终止ActorSystem

When you know everything is done for your application, you can call the `terminate` method of `ActorSystem`. That will stop the guardian actor, which in turn will recursively stop all its child actors, the system guardian.

当你知道应用程序已经完成所有任务时，你可以调用`ActorSystem`的`terminate`方法。这将停止监护人Actor，并且递归停止其所有的子级Actor。

If you want to execute some operations while terminating `ActorSystem`,look at @ref:[`CoordinatedShutdown`](../actors.md#coordinated-shutdown).

如果你想在终止`ActorSystem`时执行一些操作，请参看[`CoordinatedShutdown`](https://doc.akka.io/docs/akka/current/actors.html#coordinated-shutdown)