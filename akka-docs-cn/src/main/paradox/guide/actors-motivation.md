# 为什么现在的系统需要一个新的编程模型

The actor model was proposed decades ago by @extref[Carl Hewitt](wikipedia:Carl_Hewitt#Actor_model) as a way to handle parallel processing in a high performance network &#8212; an environment that was not available at the time. Today, hardware and infrastructure capabilities have caught up with and exceeded Hewitt's vision. Consequently, organizations building distributed systems with demanding requirements encounter challenges that cannot fully be solved with a traditional object-oriented programming (OOP) model, but that can benefit from the actor model.

Actor模型早在几十年前就被@extref[Carl Hewitt](wikipedia:Carl_Hewitt#Actor_model)提出，用来在高性能网络中处理并行计算问题，这是当时还没有的环境。如今硬件和基础设施能力已经赶上并超越了Hewitt当时的预见。因此，组织在构建分布式系统过程中陆续的碰到由传统面向对象模型无法完全解决的挑战，而这些问题却能从Actor模型中获益。

Today, the actor model is not only recognized as a highly effective solution &#8212; it has been proven in production for some of the world's most demanding applications. To highlight issues that the actor model addresses, this topic discusses the following mismatches between traditional programming assumptions and the reality of modern multi-threaded, multi-CPU architectures:

今天，Actor模型不仅仅被认为是高效的解决方案，他已经在世界上最苛刻的生产环境中得到证实。为了突出Actor模型能处理的问题，这个主题讨论了现代多线程多CPU架构与传统编程假设的不匹配。

* [The challenge of encapsulation(封装的挑战)](#the-illusion-of-encapsulation)
* [The illusion of shared memory on modern computer architectures(现代计算机体系结构上共享内存的误识)](#The-illusion-of-shared-memory-on-modern-computer-architectures)
* [The illusion of a call stack(堆栈调用的误识)](#the-illusion-of-a-call-stack)


## 封装的挑战

A core pillar of OOP is _encapsulation_. Encapsulation dictates that the internal data of an object is not accessible directly from the outside;it can only be modified by invoking a set of curated methods. The object is responsible for exposing safe operations that protect the invariant nature of its encapsulated data.

封装是OOP的核心支柱。封装规定了一个对象的内部数据是不能够通过外部直接访问的；只能通过一系列的策略方法来修改。对象负责暴露安全的操作行为来保护内部封装数据的不变性。

For example, operations on an ordered binary tree implementation must not allow violation of the tree ordering invariant. Callers expect the ordering to be intact and when querying the tree for a certain piece of data, they need to be able to rely on this constraint.

举个例子，在针对有序二叉树的操作中，不能够违反树排序的不变性。调用者希望排序的完整的，并且当查询树的某一部分数据时，他们需要依赖这个约束。

When we analyze OOP runtime behavior, we sometimes draw a message sequence chart showing the interactions of method calls. For example:

当我们分析OOP的运行时行为时，我们有时候会画一幅信息序列图来描述方法间的相互调用。如下例所示：

![sequence chart](diagrams/seq_chart.png)

Unfortunately, the above diagram does not accurately represent the _lifelines_ of the instances during execution.In reality, a _thread_ executes all these calls, and the enforcement of invariants occurs on the same thread from which the method was called. Updating the diagram with the thread of execution, it looks like this:

不幸的是，上图并没有准确的表述实例在执行期间的生命线，实际上，一个线程执行了所有这些调用，并且不变量的执行与方法的调用发生在同一线程中，更新图中的线程执行，像这样：

![sequence chart with thread](diagrams/seq_chart_thread.png)

The significance of this clarification becomes clear when you try to model what happens with _multiple threads_.Suddenly, our neatly drawn diagram becomes inadequate. We can try to illustrate multiple threads accessing the same instance:

当你试着对多线程建模的时候，这项说明的重要性就愈发清晰。突然我们整齐绘制的图示就变得不充分了，我们试着描述多线程访问同一实例。

![sequence chart with threads interacting](diagrams/seq_chart_multi_thread.png)

There is a section of execution where two threads enter the same method. Unfortunately, the encapsulation model of objects does not guarantee anything about what happens in that section. Instructions of the two invocations can be interleaved in arbitrary ways which eliminate any hope for keeping the invariants intact without some type of coordination between two threads. Now, imagine this issue compounded by the existence of many threads.

这是两个线程进入同一个函数的执行部分。不幸的是，对象的封装模型无法对那里发生的事情做任何保证。这两个执行可以以任意方式交错，这消除了两个线程在缺少某种协调作用下而保持变量不变的任何希望。现在想象一下在多线程下的这个问题。

The common approach to solving this problem is to add a lock around these methods. While this ensures that at most one thread will enter the method at any given time, this is a very costly strategy:

一般来说解决这个问题的方法是围绕这些方法加锁。这保证任意时间最多能有一个线程进入这个方法，这是消耗性很大的策略。

 * Locks _seriously limit_ concurrency, they are very costly on modern CPU architectures,requiring heavy-lifting from the operating system to suspend the thread and restore it later.
 * 锁严重限制了并发性，他们在现代CPU架构中代价高昂，需要对操作系统施加重负使它暂停线程以及稍后回复。
 * The caller thread is now blocked, so it cannot do any other meaningful work. Even in desktop applications this is unacceptable, we want to keep user-facing parts of applications (its UI) to be responsive even when a long background job is running. In the backend, blocking is outright wasteful.
   One might think that this can be compensated by launching new threads, but threads are also a costly abstraction.
 * 调用者线程现在被阻塞住了，所以他干不了任何有意义的事。甚至在桌面应用程序中这都是难以接受的，我们希望能够保持用户界面的响应，即使需要在后台长时间运行。
   在后台，阻塞是彻底的浪费。有人说可以开启另一个线程来弥补这一问题，但线程仍然是代价高昂的抽象。
 * Locks introduce a new menace: deadlocks.
 * 锁引入了新的威胁：死锁

These realities result in a no-win situation:

这导致了双输的局面

 * Without sufficient locks, the state gets corrupted.
 * 没有足够的锁，这些状态就会起冲突。
 * With many locks in place, performance suffers and very easily leads to deadlocks.
 * 用了锁的话，性能就会受影响，而且还可能会导致死锁。

Additionally, locks only really work well locally. When it comes to coordinating across multiple machines,
the only alternative is distributed locks. Unfortunately, distributed locks are several magnitudes less efficient than local locks and usually impose a hard limit on scaling out. Distributed lock protocols require several communication round-trips over the network across multiple machines, so latency goes through the roof.

另外，锁只能在本地良好运行，当涉及到协调多台机器时，唯一的选择就是分布式锁。不幸的是，分布式锁比本地锁效率低几个数量级，而且会带来扩展上的限制。
分布式锁的协议会在多态机器之间来回进行通信，因此就增加了延时

In Object Oriented languages we rarely think about threads or linear execution paths in general.
We often envision a system as a network of object instances that react to method calls, modify their internal state,then communicate with each other via method calls driving the whole application state forward:

在面向对象语言中，我们很少考虑线程或者线性执行路径。我们经常把系统想象成一个响应函数调用、修改内部状态、并通过函数调用来相互通讯驱动系统状态的对象实例的网络

![network of interacting objects](diagrams/object_graph.png)

However, in a multi-threaded distributed environment, what actually happens is that threads "traverse" this network of object instances by following method calls.As a result, threads are what really drive execution:

然而，在一个多线程分布式环境中，实际上发生的是线程通过方法的调用“遍历”对象实例的网络，所以，实际上驱动执行的线程。

![network of interactive objects traversed by threads](diagrams/object_graph_snakes.png)

**In summary**:

**总之**:

 * **Objects can only guarantee encapsulation (protection of invariants) in the face of single-threaded access,multi-thread execution almost always leads to corrupted internal state. Every invariant can be violated by having two contending threads in the same code segment.**
 * **对象只能在面对单线程访问时保证封装（对不变量的保护），在多线程中总是会导致内部状态的冲突。通过在同一代码段内的两个竞争线程可以干扰每个不变量**
 * **While locks seem to be the natural remedy to uphold encapsulation with multiple threads, in practice they are inefficient and easily lead to deadlocks in any application of real-world scale.**
 * **虽然锁是对于多线程封装的天然支撑，但实际上他们很没效率，而且容易在现实应用中导致死锁**
 * **Locks work locally, attempts to make them distributed exist, but offer limited potential for scaling out.**
 * **在本地应用锁，尝试让他们分布，但对扩展潜力要加以限制 **

## The illusion of shared memory on modern computer architectures
## 现代计算架构的内存共享图解

Programming models of the 80'-90's conceptualize that writing to a variable means writing to a memory location directly(which somewhat muddies the water that local variables might exist only in registers). On modern architectures - if we simplify things a bit - CPUs are writing to @extref[cache lines](wikipedia:CPU_cache)
instead of writing to memory directly. Most of these caches are local to the CPU core, that is, writes by one core are not visible by another. In order to make local changes visible to another core, and hence to another thread,the cache line needs to be shipped to the other core's cache.

80,90年代的编程模型认为写变量意味着直接写入内存的某地。（这有点混淆了本地变量可能只存在于寄存器中）。在现代架构中，我们把问题简化了，只接写入CPU而不是内存，
许多这种CPU缓存是针对CPU核心的，也就是一个核心写入的内容另一个核心是看不见的，为了使一个核心所做的更改能够被另一个核心看到，并因此被另一个线程看到，
需要将缓存行传达另一个缓存中。

On the JVM, we have to explicitly denote memory locations to be shared across threads by using _volatile_ markers or `Atomic` wrappers. Otherwise, we can access them only in a locked section. Why don't we just mark all variables as volatile? Because shipping cache lines across cores is a very costly operation! Doing so would implicitly stall the cores involved from doing additional work, and result in bottlenecks on the cache coherence protocol (the protocol CPUs use to transfer cache lines between main memory and other CPUs).
The result is magnitudes of slowdown.

在JVM中，我们得明确的利用易失性标签或`Atomic`包装来指示跨线程共享的内存。否则我们只能进入锁定的部分，为什么我们不能把所有变量都标记为易变的？
因为在多个核心之间传送缓存线是十分消耗性能的操作！这样做会使核心增加额外的工作，导致缓存一致性协议的瓶颈（CPU用来在主内存与其他CPU之间传输缓存线的协议）。
使得呈现出指数级的速度下降。

Even for developers aware of this situation, figuring out which memory locations should be marked as volatile,or which atomic structures to use is a dark art.

即使是开发人员也意识到，标记出哪些内存位置是易失的或者该使用哪种原子结构是黑暗艺术。

**In summary**:

**总结**：

 * **There is no real shared memory anymore, CPU cores pass chunks of data (cache lines) explicitly to each other just as computers on a network do. Inter-CPU communication and network communication have more in common than many realize. Passing messages is the norm now be it across CPUs or networked computers.**
 * **再也没有真正的共享内存，CPU内核将数据块（缓存行）显式地传递给对方就像网络上的计算机一样。CPU之间的通信和网络通信有许多共同之处。传递消息现在是通过CPU或联网计算机的标准。**
 * **Instead of hiding the message passing aspect through variables marked as shared or using atomic data structures,a more disciplined and principled approach is to keep state local to a concurrent entity and propagate data or events between concurrent entities explicitly via messages.**
 * **不再通过标记共享变量或者使用原子数据结构来隐藏消息传递行为，而是采用更严格和更有原则的方法将并发实体的状态保持在本地，并通过消息显示地在并发实体间传播事件或者数据**

## The illusion of a call stack

## 堆栈调用的误识

Today, we often take call stacks for granted. But, they were invented in an era where concurrent programming was not as important because multi-CPU systems were not common. Call stacks do not cross threads and hence,do not model asynchronous call chains.

今天，我们经常把堆栈调用为理所当然。但是，它们是在一个并发编程并不重要的时代发明的，因为多核CPU系统并不常见。调用堆栈不会跨线程，因此也没有对异步调用链创建模型。

The problem arises when a thread intends to delegate a task to the "background". In practice, this really means delegating to another thread. This cannot be a simple method/function call because calls are strictly local to the thread. What usually happens, is that the "caller" puts an object into a memory location shared by a worker thread ("callee"), which in turn, picks it up in some event loop. This allows the "caller" thread to move on and do other tasks.

问题出现在线程打算将任务委托给“后台”时。实际中，这通常意味着委托给另一个线程。这并不是一个简单的方法/函数调用问题，因为那些调用对于线程来说是内部的。通常发生的情况是，“调用者”将一个对象放入由工作线程（“被调用者”）共享的内存位置，然后工作线程在事件循环中处理它。这允许“调用者”线程继续前进并执行其他任务。

The first issue is, how can the "caller" be notified of the completion of the task? But a more serious issue arises when a task fails with an exception. Where does the exception propagate to? It will propagate to the exception handler of the worker thread completely ignoring who the actual "caller" was:

第一个问题是，如何通知“调用者”完成任务？但是更严重的问题是，如果任务失败且出现异常，异常会传播到哪里？它会传播到工作线程的异常处理程序，而完全忽略实际的“调用者”是谁：

![exceptions cannot propagate between different threads](diagrams/exception_prop.png)

This is a serious problem. How does the worker thread deal with the situation? It likely cannot fix the issue as it is usually oblivious of the purpose of the failed task. The "caller" thread needs to be notified somehow,
but there is no call stack to unwind with an exception. Failure notification can only be done via a side-channel, for example putting an error code where the "caller" thread otherwise expects the result once ready.If this notification is not in place, the "caller" never gets notified of a failure and the task is lost!
**This is surprisingly similar to how networked systems work where messages/requests can get lost/fail without any notification.**

这是一个严重的问题。工作者如何处理这种情况？它可能无法解决问题，因为它通常不知道失败任务的目的。“调用者”线程需要以某种方式被告知失败，但是没有堆栈调用来处理异常。失败通知只能通过旁路完成，例如在“调用者”线程准备好之后等待结果时输入错误代码。但如果此通知没有及时出现，“调用者”将永远不会收到失败通知，并且任务丢失！**这与网络系统的工作方式惊人的相似，即消息/请求可能在没有任何通知的情况下丢失/失败。**

This bad situation gets worse when things go really wrong and a worker backed by a thread encounters a bug and ends up in an unrecoverable situation. For example, an internal exception caused by a bug bubbles up to the root of the thread and makes the thread shut down. This immediately raises the question, who should restart the normal operation of the service hosted by the thread, and how should it be restored to a known-good state? At first glance, this might seem manageable, but we are suddenly faced by a new, unexpected phenomena: the actual task,that the thread was currently working on, is no longer in the shared memory location where tasks are taken from (usually a queue). In fact, due to the exception reaching to the top, unwinding all of the call stack, the task state is fully lost! **We have lost a message even though this is local communication with no networking involved (where message losses are to be expected).**

当发生错误并且被线程隐藏的处理单元遇到了BUG，并且结束后无法恢复时，事情只能变得更糟。例如，由一个错误引起的内部异常反馈到线程的根部使得线程关闭。这就立即引发了一个问题：谁应该重启该线程所托管的服务，以及如何将其恢复到已知状态？乍一看，这看起来可以控制，但我们突然面临一个新的意外现象：当前线程正在处理的实际任务已经不在之前从共享内存中取出时的那个位置（通常是一个队列）。事实上，由于例外达到顶端，展开所有调用堆栈，任务状态完全丢失！**这即使是本地通信不涉及到网络（消息可能会丢失），我们也会丢失消息。**

**In summary:**

**总结：**

 * **To achieve any meaningful concurrency and performance on current systems, threads must delegate tasks among each other in an efficient way without blocking. With this style of task-delegating concurrency (and even more so with networked/distributed computing) call stack-based error handling breaks down and new,explicit error signaling mechanisms need to be introduced. Failures become part of the domain model.**
 * **为了在当前系统上实现有意义的并发和执行，线程必须以有效的方式将任务委派给对方而不受阻塞。有了这种任务委托并发性（在网络化/分布式计算中更是如此），调用基于堆栈的错误处理会崩溃，需要引入新的显式错误信号机制。失败成为领域模型的一部分。**
 * **Concurrent systems with work delegation needs to handle service faults and have principled means to recover from them.Clients of such services need to be aware that tasks/messages might get lost during restarts.Even if loss does not happen, a response might be delayed arbitrarily due to previously enqueued tasks (a long queue), delays caused by garbage collection, etc. In face of these, concurrent systems should handle response deadlines in the form of timeouts, just like networked/distributed systems.**
 * **具有工作委托的并行系统需要处理服务故障并具有从其恢复的原则手段。这些服务的客户端需要知道在重新启动期间任务/消息可能会丢失。即使没有发生丢失，由于先前入队的任务（长队列），垃圾收集造成的延迟等，响应可能会被任意推迟。面对这些情况，并发系统应该以超时的形式处理响应期限，就如联网/分布式系统那样。**

Next, let's see how use of the actor model can overcome these challenges.

接下来，让我们看看演员模型的使用如何克服这些挑战。