# 术语，概念

In this chapter we attempt to establish a common terminology to define a solid ground for communicating about concurrent,distributed systems which Akka targets. Please note that, for many of these terms, there is no single agreed definition.We simply seek to give working definitions that will be used in the scope of the Akka documentation.

在本章中，我们试图建立一个通用术语来为Akka所针对的并发分布式系统进行通信定义坚实的基础。请注意，对于这些条款中的很多条款，没有一个商定的定义。我们试图给出将在Akka文档范围内使用的工作定义。

## 并发与并行(Concurrency vs. Parallelism)

Concurrency and parallelism are related concepts, but there are small differences. *Concurrency* means that two or more tasks are making progress even though they might not be executing simultaneously. This can for example be realized with time slicing where parts of tasks are executed sequentially and mixed with
parts of other tasks. *Parallelism* on the other hand arise when the execution can be truly simultaneous.

并发性和并行性是相关的概念，但有一些细微差别。*并发*意味着即使两个或多个任务可能不同时执行，也有两项或更多任务正在取得进展。例如，这可以通过时间切片来实现，其中部分任务按顺序执行并与其他任务的部分混合。*并行*，另一方面出现时，执行才能真正同步。

## 异步与同步(Asynchronous vs. Synchronous)

A method call is considered *synchronous* if the caller cannot make progress until the method returns a value or throws an exception. On the other hand, an *asynchronous* call allows the caller to progress after a finite number of steps, and the completion of the method may be signalled via some additional mechanism (it might be a registered callback, a Future,or a message).

如果调用方在方法返回值或抛出异常之前无法进展，则方法调用被认为是*同步*的。另一方面，*异步*调用允许调用者在有限数量的步骤后进行，并且方法的完成可以通过一些额外的机制（它可能是注册回调，未来或消息）发出信号。

A synchronous API may use blocking to implement synchrony, but this is not a necessity. A very CPU intensive task might give a similar behavior as blocking. In general, it is preferred to use asynchronous APIs, as they guarantee that the system is able to progress. Actors are asynchronous by nature: an actor can progress after a message send without waiting for the actual delivery to happen.

同步API可能使用阻塞来实现同步，但这不是必需的。CPU密集型任务可能会产生与阻塞相似的行为。一般来说，最好使用异步API，因为它们保证系统能够进步。演员本质上是异步的：演员可以在发送消息之后进展，而不用等待实际发送。

## 非阻塞与阻塞(Non-blocking vs. Blocking)

We talk about *blocking* if the delay of one thread can indefinitely delay some of the other threads. A good example is a resource which can be used exclusively by one thread using mutual exclusion. If a thread holds on to the resource indefinitely (for example accidentally running an infinite loop) other threads waiting on the resource can not progress.In contrast, *non-blocking* means that no thread is able to indefinitely delay others.

如果一个线程的延迟可以无限期地延迟其他一些线程，我们讨论*阻塞*。一个很好的例子是一个资源，它可以被一个线程排他地使用。如果线程无限期地持有资源（例如意外运行无限循环），则等待资源的其他线程无法继续进行。相反，*非阻塞*意味着没有线程能够无限延迟其他线程。

Non-blocking operations are preferred to blocking ones, as the overall progress of the system is not trivially guaranteed when it contains blocking operations.

非阻塞操作优于阻塞操作，因为当系统包含阻塞操作时，系统的整体进度并不平凡。

## 死锁与饥饿与实时锁定(Deadlock vs. Starvation vs. Live-lock)

*Deadlock* arises when several participants are waiting on each other to reach a specific state to be able to progress.As none of them can progress without some other participant to reach a certain state (a "Catch-22" problem) all affected subsystems stall. Deadlock is closely related to *blocking*, as it is necessary that a participant thread be able to delay the progression of other threads indefinitely.

当几个参与者相互等待达到一个特定的状态以便能够进步时，会出现*僵局*。如果没有其他参与者达到某个特定状态（“Catch-22”问题），他们都不会进步，所有受影响的子系统都会停止。死锁与*阻塞*密切相关，因为参与者线程必须能够无限期地延迟其他线程的进程。

In the case of *deadlock*, no participants can make progress, while in contrast *Starvation* happens, when there are participants that can make progress, but there might be one or more that cannot. Typical scenario is the case of a naive scheduling algorithm that always selects high-priority tasks over low-priority ones. If the number of incoming high-priority tasks is constantly high enough, no low-priority ones will be ever finished.

在*僵局中*，没有参与者可以取得进展，相反，当有参与者可以取得进展时，会发生*饥饿*，但可能有一个或多个参与者无法取得进展。典型的场景是天真调度算法，它总是选择高优先级任务而不是低优先级任务。如果输入的高优先级任务的数量持续足够高，则不会完成低优先级的任务。

*Livelock* is similar to *deadlock* as none of the participants make progress. The difference though is that instead of being frozen in a state of waiting for others to progress, the participants continuously change their state. An example scenario when two participants have two identical resources available. They each try to get the resource, but they also check if the other needs the resource, too. If the resource is requested by the other participant, they try to get the other instance of the resource. In the unfortunate case it might happen that the two participants "bounce" between the two resources, never acquiring it, but always yielding to the other.

因为没有参与者取得进展，所以*活锁*类似于*死锁*。但不同之处在于，不是在等待别人进步的状态下被冻结，而是参与者不断地改变他们的状态。两个参与者拥有两个相同的可用资源时的示例场景。他们每个人都试图获得资源，但他们也检查另一方是否也需要资源。如果资源被其他参与者请求，他们会尝试获取资源的其他实例。在不幸的情况下，可能发生这两个参与者在两种资源之间“反弹”，从未获得它，但总是屈服于另一个。

## 比赛条件(Race Condition)

We call it a *Race condition* when an assumption about the ordering of a set of events might be violated by external non-deterministic effects. Race conditions often arise when multiple threads have a shared mutable state, and the operations of thread on the state might be interleaved causing unexpected behavior. While this is a common case, shared state is not necessary to have race conditions. One example could be a client sending unordered packets (e.g UDP datagrams) `P1`, `P2` to a server. As the packets might potentially travel via different network routes, it is possible that the server receives `P2` first and `P1` afterwards. If the messages contain no information about their sending order it is impossible to determine by the server that they were sent in a different order. Depending on the meaning of the packets this can cause race conditions.

当一组事件的排序假设可能被外部非确定性效应侵犯时，我们称之为*竞争条件*。当多个线程具有共享的可变状态时，通常会出现竞争状态，并且状态上的线程操作可能会交错，从而导致意外行为。虽然这是一种常见的情况，但共享状态对于竞争条件没有必要。一个示例可以是客户端发送无序分组（例如UDP数据报）`P1`，`P2`到一个服务器。由于数据包可能会通过不同的网络路由传输，因此服务器可能`P2`首先接收数据包`P1`之后。如果消息中不包含有关其发送顺序的信息，则服务器无法确定它们是以不同的顺序发送的。根据数据包的含义，这可能会导致竞争条件。

@@@ note

The only guarantee that Akka provides about messages sent between a given pair of actors is that their order is always preserved. see @ref:[Message Delivery Reliability](message-delivery-reliability.md)

Akka提供关于在给定的演员对之间发送的消息的唯一保证是他们的顺序始终保留。看[邮件传递可靠性](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html)

@@@

## 非阻塞保证（进展条件）(Non-blocking Guarantees (Progress Conditions))

As discussed in the previous sections blocking is undesirable for several reasons, including the dangers of deadlocks and reduced throughput in the system. In the following sections we discuss various non-blocking properties with different strength.

正如前面部分所讨论的那样，由于多种原因，阻塞是不可取的，包括死锁的危险和系统吞吐量的降低。在下面的章节中，我们将讨论具有不同强度的各种非阻塞属性。

### 等待自由度(Wait-freedom)

A method is *wait-free* if every call is guaranteed to finish in a finite number of steps. If a method is
*bounded wait-free* then the number of steps has a finite upper bound.

如果每个呼叫都保证以有限数量的步骤完成，则方法是无*等待的*。如果一个方法有*界等待，*那么步数有一个有限的上界。

From this definition it follows that wait-free methods are never blocking, therefore deadlock can not happen.Additionally, as each participant can progress after a finite number of steps (when the call finishes), wait-free methods are free of starvation.

从这个定义可以看出，无等待方法永远不会阻塞，因此不会发生死锁。另外，由于每个参与者都可以在有限的步骤之后（当呼叫结束时）进展，所以免等待方法没有饥饿。

### 锁定自由(Lock-freedom)

*Lock-freedom* is a weaker property than *wait-freedom*. In the case of lock-free calls, infinitely often some method finishes in a finite number of steps. This definition implies that no deadlock is possible for lock-free calls. On the other hand, the guarantee that *some call finishes* in a finite number of steps is not enough to guarantee that *all of them eventually finish*. In other words, lock-freedom is not enough to guarantee the lack of starvation.

*锁定自由*是一种比*等待**自由*更弱的特性。在无锁呼叫的情况下，无限地经常有某些方法以有限数量的步骤结束。这个定义意味着无锁定呼叫不会发生死锁。另一方面，保证*某些通话*在有限数量的步骤中完成并不足以保证*所有**通话**都能最终完成*。换句话说，锁定自由度不足以保证缺乏饥饿。

### 梗阻自由(Obstruction-freedom)

*Obstruction-freedom* is the weakest non-blocking guarantee discussed here. A method is called *obstruction-free* if there is a point in time after which it executes in isolation (other threads make no steps, e.g.: become suspended), it finishes in a bounded number of steps. All lock-free objects are obstruction-free, but the opposite is generally not true.

*梗阻自由度*是这里讨论的最弱的无阻塞的保证。如果某个方法在某个时间点之后独立执行（其他线程没有执行任何步骤，例如：变为暂停），则该方法被称为*无阻塞*方法，它会以有限数量的步骤结束。所有无锁物体都是无阻塞的，但相反的情况通常不是这样。

*Optimistic concurrency control* (OCC) methods are usually obstruction-free. The OCC approach is that every participant tries to execute its operation on the shared object, but if a participant detects conflicts from others, it rolls back the modifications, and tries again according to some schedule. If there is a point in time, where one of the participants is the only one trying, the operation will succeed.

*乐观并发控制*（OCC）方法通常无障碍。OCC方法是每个参与者都试图在共享对象上执行其操作，但是如果参与者检测到来自其他人的冲突，它将回滚修改，并根据某个时间表再次尝试。如果有一个时间点，其中一个参与者是唯一一个参与者尝试，则该操作将成功。

## 推荐文献(Recommended literature)

 * The Art of Multiprocessor Programming, M. Herlihy and N Shavit, 2008. ISBN 978-0123705914
 * Java Concurrency in Practice, B. Goetz, T. Peierls, J. Bloch, J. Bowbeer, D. Holmes and D. Lea, 2006. ISBN 978-0321349606
