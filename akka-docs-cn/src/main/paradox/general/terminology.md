# 术语，概念(整个这章建议自己查资料)

In this chapter we attempt to establish a common terminology to define a solid ground for communicating about concurrent,distributed systems which Akka targets. Please note that, for many of these terms, there is no single agreed definition.We simply seek to give working definitions that will be used in the scope of the Akka documentation.

在本章中，我们试图建立一套通用术语来为Akka中所针对的并发分布式系统打下坚实的交流基础。请注意，在这些术语当中，很多术语没有共识的定义。我们只是试图简单的给出在Akka文档范围内使用的工作定义。

## 并发与并行(Concurrency vs. Parallelism)

Concurrency and parallelism are related concepts, but there are small differences. *Concurrency* means that two or more tasks are making progress even though they might not be executing simultaneously. This can for example be realized with time slicing where parts of tasks are executed sequentially and mixed with
parts of other tasks. *Parallelism* on the other hand arise when the execution can be truly simultaneous.

并发性和并行性是相关的概念，但有一些细微差别。*并发*意味着两个或多个任务正在取得进展，哪怕他们并不是同时执行。例如，这可以通过时间切片来理解，其中部分任务按顺序执行并且与其他任务的部分混合。而*并行*，是真正意义上的同时执行。

## 异步与同步(Asynchronous vs. Synchronous)

A method call is considered *synchronous* if the caller cannot make progress until the method returns a value or throws an exception. On the other hand, an *asynchronous* call allows the caller to progress after a finite number of steps, and the completion of the method may be signaled via some additional mechanism (it might be a registered callback, a Future,or a message).

如果一个方法调用在返回结果或抛出异常之前无法继续进行，则方法调用被认为是*同步*的。另一方面，*异步*调用允许调用者在有限的步骤后继续进行，并且方法的完成可以通过一些额外的机制通知（比如：注册回调，Future或消息）。

A synchronous API may use blocking to implement synchrony, but this is not a necessity. A very CPU intensive task might give a similar behavior as blocking. In general, it is preferred to use asynchronous APIs, as they guarantee that the system is able to progress. Actors are asynchronous by nature: an actor can progress after a message send without waiting for the actual delivery to happen.

同步API可能使用阻塞来实现同步，但这不是必需的。CPU密集型任务可能会产生与阻塞相似的行为。一般来说，最好使用异步API，因为它们保证系统能够继续进行。Actor本质上是异步的：Actor可以在发送消息之后就执行下一步，而不必等待实际传送行为的发生。

## 非阻塞与阻塞(Non-blocking vs. Blocking)

We talk about *blocking* if the delay of one thread can indefinitely delay some of the other threads. A good example is a resource which can be used exclusively by one thread using mutual exclusion. If a thread holds on to the resource indefinitely (for example accidentally running an infinite loop) other threads waiting on the resource can not progress.In contrast, *non-blocking* means that no thread is able to indefinitely delay others.

我们在如果一个线程的延迟可以无限期地延迟其他一些线程的时候讨论*阻塞*。一个很好的例子是，一个资源可以被一个线程排他使用。如果某个线程无限期地持有资源（例如意外的进入了无限循环），则等待这个资源的其他线程则无法继续进行。相反，*非阻塞*意味着没有线程能够无限延迟其他线程。

(**关于阻塞非阻塞这段，Akka解释的不恰当！！！**)

Non-blocking operations are preferred to blocking ones, as the overall progress of the system is not trivially guaranteed when it contains blocking operations.

非阻塞操作优于阻塞操作，因为当系统包含阻塞操作时，系统的整体进度并不能轻易保证。

## 死锁与饿死与实时锁定(Deadlock vs. Starvation vs. Live-lock)

*Deadlock* arises when several participants are waiting on each other to reach a specific state to be able to progress.As none of them can progress without some other participant to reach a certain state (a "Catch-22" problem) all affected subsystems stall. Deadlock is closely related to *blocking*, as it is necessary that a participant thread be able to delay the progression of other threads indefinitely.

死锁往往出现在，当几个参与者都在等待其余人以便达到特定状态才能够继续进行时。如果没有其他参与者达到某个特定状态（“Catch-22”问题），他们都不会继续进行，所有受影响的子系统都会停止。死锁与*阻塞*密切相关，因为参与者线程能够无限期地延迟其他线程的继续。

In the case of *deadlock*, no participants can make progress, while in contrast *Starvation* happens, when there are participants that can make progress, but there might be one or more that cannot. Typical scenario is the case of a naive scheduling algorithm that always selects high-priority tasks over low-priority ones. If the number of incoming high-priority tasks is constantly high enough, no low-priority ones will be ever finished.

在*死锁*中，没有参与者可以取得进展，然而在饿死中，有的参与者可以取得进展，但可能有一个或多个参与者无法取得进展。典型的场景是朴素调度算法，它总是选择高优先级任务而不是低优先级任务。如果输入的高优先级任务的数量一直很多，则低优先级的任务将无法完成。

*Livelock* is similar to *deadlock* as none of the participants make progress. The difference though is that instead of being frozen in a state of waiting for others to progress, the participants continuously change their state. An example scenario when two participants have two identical resources available. They each try to get the resource, but they also check if the other needs the resource, too. If the resource is requested by the other participant, they try to get the other instance of the resource. In the unfortunate case it might happen that the two participants "bounce" between the two resources, never acquiring it, but always yielding to the other.

*活锁*类似于*死锁*，也是没有参与者取得进展。但不同之处在于，不是一直在等待别人取得进展，而是参与者不断地改变自己的状态。一个事例场景，两个参与者拥有两个相同的可用资源。他们每个人都试图获得资源，但他们也检查另一方是否也需要资源。如果资源被另一个参与者请求，他们则会尝试获取资源的另一个实例。但在不幸的情况下，可能会发生两个参与者在两个资源之间“反弹”，总是让步于另一个参与值以致于总是获取不到资源。

## 竞争条件(Race Condition)

We call it a *Race condition* when an assumption about the ordering of a set of events might be violated by external non-deterministic effects. Race conditions often arise when multiple threads have a shared mutable state, and the operations of thread on the state might be interleaved causing unexpected behavior. While this is a common case, shared state is not necessary to have race conditions. One example could be a client sending unordered packets (e.g UDP datagrams) `P1`, `P2` to a server. As the packets might potentially travel via different network routes, it is possible that the server receives `P2` first and `P1` afterwards. If the messages contain no information about their sending order it is impossible to determine by the server that they were sent in a different order. Depending on the meaning of the packets this can cause race conditions.

当一组事件的假设排序可能被外部不确定因素影响时，我们称之为*竞争条件*。当多个线程具有共享的可变状态时，并且线程对于状态的操作可能会相互引起意料外的行为，这时便会出现竞争状态。虽然这是一种常见的情况，但共享状态不是竞争条件的必要条件。比如说客户端向服务器发送无序数据包（例如UDP数据报）`P1`，`P2`。由于数据包可能会通过不同的网络路由器，因此服务器可能先接受`P2`再接收`P1`。如果消息中不包含有关其发送顺序的信息，则服务器无法确定它们是以不同的顺序发送的。根据数据包的含义，这可能会导致竞争条件。

@@@ note

The only guarantee that Akka provides about messages sent between a given pair of actors is that their order is always preserved. see @ref:[Message Delivery Reliability](message-delivery-reliability.md)

关于Akka在给定Actor之间发送的消息的唯一能保证的是他们的顺序。看[消息传递可靠性](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html)

@@@

## 非阻塞保证（继续进行的条件）(Non-blocking Guarantees (Progress Conditions))

As discussed in the previous sections blocking is undesirable for several reasons, including the dangers of deadlocks and reduced throughput in the system. In the following sections we discuss various non-blocking properties with different strength.

正如前面部分所讨论的那样，由于死锁的危害和系统吞吐量的降低等多种原因，阻塞是不可取的。在下面的章节中，我们将讨论具有不同强度的各种非阻塞特性。

### 无等待(Wait-freedom)

A method is *wait-free* if every call is guaranteed to finish in a finite number of steps. If a method is
*bounded wait-free* then the number of steps has a finite upper bound.

如果每个调用都保证在有限的步骤内完成，则方法是无*等待的*。如果一个方法*有界无等待*的，那么步数有一个上限。

From this definition it follows that wait-free methods are never blocking, therefore deadlock can not happen.Additionally, as each participant can progress after a finite number of steps (when the call finishes), wait-free methods are free of starvation.

从这个定义可以看出，无等待方法永远不会阻塞，因此不会发生死锁。另外，由于每个参与者都可以在有限的步骤之后（当调用结束时）继续进行，所以无等待方法没有饿死。

### 无锁定(Lock-freedom)

*Lock-freedom* is a weaker property than *wait-freedom*. In the case of lock-free calls, infinitely often some method finishes in a finite number of steps. This definition implies that no deadlock is possible for lock-free calls. On the other hand, the guarantee that *some call finishes* in a finite number of steps is not enough to guarantee that *all of them eventually finish*. In other words, lock-freedom is not enough to guarantee the lack of starvation.

*无锁定*是一种比*无等待*更弱的特性。在无锁定调用的情况下，方法以有限的步骤结束。这个定义意味着无锁定调用不会发生死锁。另一方面，保证*某些调用*在有限的步骤中完成并不足以保证*所有调用*都能最终完成。换句话说，无锁定不足以保证无饿死。

### 无阻塞(Obstruction-freedom)

*Obstruction-freedom* is the weakest non-blocking guarantee discussed here. A method is called *obstruction-free* if there is a point in time after which it executes in isolation (other threads make no steps, e.g.: become suspended), it finishes in a bounded number of steps. All lock-free objects are obstruction-free, but the opposite is generally not true.

*无阻塞*是这里讨论的最弱的无阻塞特性。如果某个方法在某个时间点之后独立执行（其他线程没有执行任何步骤，例如：变为挂起），则该方法被称为*无阻塞*方法，它会在有限的步骤内结束。所有无锁定都是无阻塞的，但反之不然。

*Optimistic concurrency control* (OCC) methods are usually obstruction-free. The OCC approach is that every participant tries to execute its operation on the shared object, but if a participant detects conflicts from others, it rolls back the modifications, and tries again according to some schedule. If there is a point in time, where one of the participants is the only one trying, the operation will succeed.

*乐观并发控制*（OCC）方法通常是无阻塞。OCC方法是每个参与者都试图在共享对象上执行其操作，但是如果参与者检测到来自其他人的冲突，它将回滚其修改，并根据某个时间调度再次尝试。如果有一个时间点，其中一个参与者是唯一一个参与者，则其操作将成功。

## 推荐文献(Recommended literature)

 * The Art of Multiprocessor Programming, M. Herlihy and N Shavit, 2008. ISBN 978-0123705914
 * Java Concurrency in Practice, B. Goetz, T. Peierls, J. Bloch, J. Bowbeer, D. Holmes and D. Lea, 2006. ISBN 978-0321349606
