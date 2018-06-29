# Akka和Java内存模型

A major benefit of using the Lightbend Platform, including Scala and Akka, is that it simplifies the process of writing concurrent software.  This article discusses how the Lightbend Platform, and Akka in particular, approaches shared memory in concurrent applications.

包括Scala和Akka在内的Lightbend平台的一个主要优点就是它简化了编写并发软件的过程。本文讨论Lightbend平台特别是Akka如何在并发应用程序内处理共享内存。

## Java内存模型

Prior to Java 5, the Java Memory Model (JMM) was ill defined. It was possible to get all kinds of strange results when shared memory was accessed by multiple threads, such as:

在Java 5之前，Java内存模型（JMM）没有定义好。共享内存被多个线程访问时，可能会得到各种奇怪的结果，例如：

 * a thread not seeing values written by other threads: a visibility problem
 * 一个线程看不到其他线程写的值：一个可见性问题
 * a thread observing 'impossible' behavior of other threads, caused by instructions not being executed in the order expected: an instruction reordering problem.
 * 一个线程观察到其他线程的“不可能”行为，这是由于指令没有按预期的顺序执行：一个指令重新排序问题。

With the implementation of JSR 133 in Java 5, a lot of these issues have been resolved. The JMM is a set of rules based on the "happens-before" relation, which constrain when one memory access must happen before another, and conversely,when they are allowed to happen out of order. Two examples of these rules are:

随着Java 5中JSR 133的实现，这些问题许多已经得到解决。JMM是基于“happens-before”关系的一组规则，它约束了一个存储器访问必须发生在另一个存储器访问之前，反之，则是它们被允许无序访问时。这些规则的两个例子是：

 * **The monitor lock rule:** a release of a lock happens before every subsequent acquire of the same lock.
 * **监视器锁定规则：**锁在下一次请求之前被释放。
 * **The volatile variable rule:** a write of a volatile variable happens before every subsequent read of the same volatile variable
 * **易失性变量规则：**易失性变量的写操作在该变量的读操作之前完成

Although the JMM can seem complicated, the specification tries to find a balance between ease of use and the ability to write performant and scalable concurrent data structures.

尽管JMM看起来很复杂，但该规范试图在易用性和编写高性能和可伸缩并发数据结构的能力之间找到平衡点。

## Actor和Java内存模型

With the Actors implementation in Akka, there are two ways multiple threads can execute actions on shared memory:

通过Akka中的Actor实现，多线程可以通过两种方式在共享内存上执行操作：

 * if a message is sent to an actor (e.g. by another actor). In most cases messages are immutable, but if that message is not a properly constructed immutable object, without a "happens before" rule, it would be possible for the receiver to see partially initialized data structures and possibly even values out of thin air (longs/doubles).
 * 如果一条消息被发送给Actor（由另一个Actor）。在大多数情况下，消息是不可变的，但是如果消息不是一个正确构造的不可变对象，在没有“happens before”规则下，接收者可能会看到部分初始化的数据结构，甚至可能一些无中生有的值。
 * if an actor makes changes to its internal state while processing a message, and accesses that state while processing another message moments later. It is important to realize that with the actor model you don't get any guarantee that the same thread will be executing the same actor for different messages.
 * 如果一个Actor在处理消息时更改了其内部状态，并在随后处理另一条消息时访问该状态。很重要的一点是，利用Actor模型无法保证这个Actor使用同一个线程来处理这两条消息。

To prevent visibility and reordering problems on actors, Akka guarantees the following two "happens before" rules:

为了防止Actor的可视性和重新排序问题，Akka保证以下两个“happens before”规则：

 * **The actor send rule:** the send of the message to an actor happens before the receive of that message by the same actor.
 * **Actor发送规则：**对一个Actor的发送消息行为，一定发生在这个Actor接收这条消息之前。
 * **The actor subsequent processing rule:** processing of one message happens before processing of the next message by the same actor.
 * **Actor后续处理规则：**一个Actor处理一条消息，一定是发生在处理下条消息之前。

@@@ note

In layman's terms this means that changes to internal fields of the actor are visible when the next message
is processed by that actor. So fields in your actor need not be volatile or equivalent.

通俗地说，这意味着当Actor处理下一条消息时，Actor内部字段的更改是可见的。所以你的Actor的字段不必是易变的或等效的(**？？？？**)。

@@@

Both rules only apply for the same actor instance and are not valid if different actors are used.

这两条规则只适用于相同的Actor实例，而不同的Actor则不适用。

## Future和Java内存模型

The completion of a Future "happens before" the invocation of any callbacks registered to it are executed.

Future的完成发生在注册在它上面的回调对象被执行之前。

We recommend not to close over non-final fields (final in Java and val in Scala), and if you *do* choose to close over non-final fields, they must be marked *volatile* in order for the current value of the field to be visible to the callback.

我们建议不要隐藏非final字段（Java的final和Scala的val），如果你确实要选择隐藏非final字段，那它们必须被标注*volatile*，以使得字段的当前值对回调对象可见。(**啊啊啊啊阿西吧！！！！这close over啥意思？？？**)

If you close over a reference, you must also ensure that the instance that is referred to is thread safe.We highly recommend staying away from objects that use locking, since it can introduce performance problems and in the worst case, deadlocks.Such are the perils of synchronized.

如果关闭引用，则还必须确保所引用对应的实例是线程安全的。我们强烈建议远离使用锁的对象，因为它可能会导致性能问题，最坏的情况是会导致死锁。这就是同步的危险。

<a id="jmm-shared-state"></a>
## Actor和共享可变状态

Since Akka runs on the JVM there are still some rules to be followed.

由于Akka运行在JVM上，因此还有一些规则需要遵循。

 * Closing over internal Actor state and exposing it to other threads
 * 关闭内部Actor状态并将其暴露给其他线程

@@snip [SharedMutableStateDocSpec.scala]($code$/scala/docs/actor/SharedMutableStateDocSpec.scala) { #mutable-state }

 * Messages **should** be immutable, this is to avoid the shared mutable state trap.
 * 消息**应该**是不可变的，这是为了避免共享可变状态陷阱。