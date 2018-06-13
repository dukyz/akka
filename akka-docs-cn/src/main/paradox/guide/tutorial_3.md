# 第3部分：使用设备Actor

In the previous topics we explained how to view actor systems _in the large_, that is, how components should be represented, how actors should be arranged in the hierarchy. In this part, we will look at actors _in the small_ by implementing the device actor.

在前面的主题中，我们解释了如何从宏观角度来查看Actor系统，即组件应该如何表示，Actor应该如何安排在层次结构中。在这一部分中，我们将通过实施具体的设备Actor来从微观角度进一步观察。

If we were working with objects, we would typically design the API as _interfaces_, a collection of abstract methods to be filled out by the actual implementation. In the world of actors, protocols take the place of interfaces. While it is not possible to formalize general protocols in the programming language, we can compose their most basic element, messages. So, we will start by identifying the messages we will want to send to device actors.

如果我们使用对象，我们通常会将API设计为*接口*，那是一组通过实际实现来满足的抽象方法。在Actor的世界里，协议取代了接口。虽然不可能在编程语言中形式化通用协议，但我们可以编写它们最基本的元素，即消息。因此，我们将首先确定我们要发送给设备Actor的消息。

Typically, messages fall into categories, or patterns. By identifying these patterns, you will find that it becomes easier to choose between them and to implement them. The first example demonstrates the _request-respond_ message pattern.

通常，消息属于类别或模式。通过识别这些模式，您会发现在它们之间进行选择并实施它们变得更加容易。第一个例子演示了*请求响应*消息模式。

## 识别设备的消息

The tasks of a device actor will be simple:

设备Actor的任务很简单：

 * Collect temperature measurements
 * 收集温度测量值
 * When asked, report the last measured temperature
 * 当被问到时，报告上次测量的温度

However, a device might start without immediately having a temperature measurement. Hence, we need to account for the case where a temperature is not present. This also allows us to test the query part of the actor without the write part present, as the device actor can simply report an empty result.

但是，设备可能刚启动不会立即进行温度测量。因此，我们需要考虑温度数据不存在的情况。这也允许我们在没写入数据的情况下测试Actor的查询部分，因为设备Actor可以简单地报告空的结果。

The protocol for obtaining the current temperature from the device actor is simple. The actor:

从设备Actor获取当前温度的协议很简单。该Actor：

  1. Waits for a request for the current temperature.
  2. 等待当前温度的请求。
  3. Responds to the request with a reply that either:
  4. 以下列任一形式回应请求：
    * contains the current temperature or,
    * 包含当前温度或者,
    * indicates that a temperature is not yet available.
    * 温度尚不可用。

We need two messages, one for the request, and one for the reply. Our first attempt might look like the following:

我们需要两条消息，一条用于请求，另一条用于答复。我们的第一次尝试可能如下所示：

Scala
:   @@snip [DeviceInProgress.scala]($code$/scala/tutorial_3/DeviceInProgress.scala) { #read-protocol-1 }

Java
:   @@snip [DeviceInProgress.java]($code$/java/jdocs/tutorial_3/DeviceInProgress.java) { #read-protocol-1 }

These two messages seem to cover the required functionality. However, the approach we choose must take into account the distributed nature of the application. While the basic mechanism is the same for communicating with an actor on the local JVM as with a remote actor, we need to keep the following in mind:

这两条消息似乎涵盖了所需的功能。但是，我们选择的方法必须考虑到应用程序的分布式特性。虽然与本地JVM上的Actor进行通信与远程Actor进行通信的基本机制相同，但我们需要牢记以下几点：

* There will be observable differences in the latency of delivery between local and remote messages, because factors like network link bandwidth and the message size also come into play.
* 由于网络链路带宽和消息大小等因素，本地和远程消息之间的传输延迟会有明显差异。
* Reliability is a concern because a remote message send involves more steps, which means that more can go wrong.
* 可靠性是一个重要问题，因为远程消息发送涉及多个步骤，这意味着更多出错的可能性。
* A local send will just pass a reference to the message inside the same JVM, without any restrictions on the underlying object which is sent, whereas a remote transport will place a limit on the message size.
* 本地发送只会在同一个JVM中传递对消息的引用，而不会对发送的底层对象有任何限制，而远程传输将对消息大小有限制。

In addition, while sending inside the same JVM is significantly more reliable, if an actor fails due to a programmer error while processing the message, the effect is basically the same as if a remote network request fails due to the remote host crashing while processing the message. Even though in both cases, the service recovers after a while (the actor is restarted by its supervisor, the host is restarted by an operator or by a monitoring system) individual requests are lost during the crash. **Therefore, writing your actors such that every message could possibly be lost is the safe, pessimistic bet.**

另外，在同一个JVM里面发送信息的可靠性要高得多，如果一个Actor是由于程序员错误而引起处理消息失败，那么其效果基本等同于由于远程主机处理消息时崩溃而导致远程请求失败。即使在这两种情况下，服务也会在一段时间后恢复（Actor由其监督者重新启动，主机则由操作员或监控系统重新启动），但个别请求在崩溃期间会丢失。**因此，在每一条信息都有可能丢失的共识下编写Actor是安全的，这是悲观的赌注。(这段话不明白什么意思！！看英文)**

But to further understand the need for flexibility in the protocol, it will help to consider Akka message ordering and message delivery guarantees. Akka provides the following behavior for message sends:

但为了进一步理解协议的灵活性需求，它将有助于考虑Akka消息排序和消息传递保证。Akka为消息发送提供以下行为：

 * At-most-once delivery, that is, no guaranteed delivery.
 * 最多交付一次，即没有保证交付。
 * Message ordering is maintained per sender, receiver pair.
 * 消息排序是按照每组“发送者—接收者”来维护的。

The following sections discuss this behavior in more detail:

以下各节将更详细地讨论此行为：

* [消息传递](#message-delivery)
* [消息排序](#message-ordering)

### 消息传递

The delivery semantics provided by messaging subsystems typically fall into the following categories:

消息传递子系统提供的传递语义通常分为以下几类：

 * **At-most-once delivery** &#8212; each message is delivered zero or one time; in more causal terms it means that messages can be lost, but are never duplicated.
 * **最多交付一次** - 每个消息传递零次或一次; 在许多场景中，这意味着消息可能会丢失，但不会重复
 * **At-least-once delivery** &#8212; potentially multiple attempts are made to deliver each message, until at least one succeeds; again, in more causal terms this means that messages can be duplicated but are never lost.
 * **至少一次交付** - 可能会多次尝试传递每条消息，直到至少一次成功; 同样的在许多场景中，这意味着消息可能会重复，但永远不会丢失。
 * **Exactly-once delivery** &#8212; each message is delivered exactly once to the recipient; the message can neither be lost nor be duplicated.
 * **精准一次交付** - 每条消息只发送给接收方一次; 该消息既不会丢失也不会重复。

The first behavior, the one used by Akka, is the cheapest and results in the highest performance. It has the least implementation overhead because it can be done in a fire-and-forget fashion without keeping the state at the sending end or in the transport mechanism. The second, at-least-once, requires retries to counter transport losses. This adds the overhead of keeping the state at the sending end and having an acknowledgment mechanism at the receiving end. Exactly-once delivery is most expensive, and results in the worst performance: in addition to the overhead added by at-least-once delivery, it requires the state to be kept at the receiving end in order to filter out duplicate deliveries.

第一种是最廉价同时也是性能最高方式。它实现开销最小，因为它不需要在发送端或传输机制中保持状态，发完即忘。第二种，至少一次交付，需要重试来应对运输损失。这增加了在发送端保持状态和接收端建立确认机制的开销。确切的一次交付是最昂贵的，同时性能最差：除了至少一次交付所增加的开销外，它还要求在接收端保留状态以便过滤掉重复的交付。

In an actor system, we need to determine exact meaning of a guarantee &#8212; at which point does the system consider the delivery as accomplished:

在Actor系统中，我们需要确定“保证”的确切含义 - 即系统在哪一点上认为交付已完成：

  1. When the message is sent out on the network?
  2. 当消息在网络上发送出去？
  3. When the message is received by the target actor's host?
  4. 当消息被目标Actor所在主机收到时？
  5. When the message is put into the target actor's mailbox?
  6. 当消息被放入到目标Actor的邮箱中时？
  7. When the message target actor starts to process the message?
  8. 当目标Actor开始处理消息时？
  9. When the target actor has successfully processed the message?
  10. 当目标Actor成功处理了该消息时？

Most frameworks and protocols that claim guaranteed delivery actually provide something similar to points 4 and 5. While this sounds reasonable, **is it actually useful?** To understand the implications, consider a simple, practical example: a user attempts to place an order and we only want to claim that it has successfully processed once it is actually on disk in the orders database.

大多数声称能保证交付的框架和协议实际上提供了类似于第4点和第5点判断标准。虽然这听起来合理，**但它实际上有用吗？**要理解其含义，请考虑一个简单实用的示例：一个用户尝试下订单，我们只有当消息被写入订单数据库的硬盘时才声称该消息被成功处理了。

If we rely on the successful processing of the message, the actor will report success as soon as the order has been submitted to the internal API that has the responsibility to validate it, process it and put it into the database. Unfortunately,immediately after the API has been invoked any of the following can happen:

如果我们是依赖对消息的成功处理，只要订单已提交给负责验证、处理及持久化的内部API，Acotr就会立刻报告成功。而不幸的是，在调用API后以下几种情况可能随之发生：

 * The host can crash.
 * 主机可能会崩溃。
 * Deserialization can fail.
 * 反序列化可能失败。
 * Validation can fail.
 * 验证可能会失败。
 * The database might be unavailable.
 * 数据库可能不可用。
 * A programming error might occur.
 * 程序可能出错

This illustrates that the **guarantee of delivery** does not translate to the **domain level guarantee**. We only want to report success once the order has been actually fully processed and persisted. **The only entity that can report success is the application itself, since only it has any understanding of the domain guarantees required. No generalized framework can figure out the specifics of a particular domain and what is considered a success in that domain**.

这里指出**保证交付**不同于**领域级别保证交付**。我们只想在订单实际被处理并持久化后报告交付成功。**唯一可宣告报告交付成功的实体是应用程序，因为只有它才理解领域级别保证交付的需求。没有一个通用的框架可以指出某个特定领域的细节情况以及该领域怎样的行为才算交付成功**。

In this particular example, we only want to signal success after a successful database write, where the database acknowledged that the order is now safely stored. **For these reasons Akka lifts the responsibilities of guarantees to the application itself, i.e. you have to implement them yourself. This gives you full control of the guarantees that you want to provide**. Now, let's consider the message ordering that Akka provides to make it easy to reason about application logic.

在这个特定的例子中，我们只想在数据库成功写入之后，即数据库确认订单已被安全地存储入库，才发出成功信号。**由于这些种种原因，Akka将保证的责任交给了应用程序，即您必须自行实施交付保证策略。这使您可以完全控制您要提供的保证机制**。现在，让我们考虑Akka提供的消息排序，以便轻松推理应用程序逻辑。

### Message Ordering

In Akka, for a given pair of actors, messages sent directly from the first to the second will not be received out-of-order. The word directly emphasizes that this guarantee only applies when sending with the tell operator directly to the final destination, but not when employing mediators.

在Akka里，对于一对给定的Actor，从第一个Actor直接发送到第二个Actor的消息是不会顺序错乱的。上述强调只有在利用tell操作符直接发送到最终目的地时才能保证顺序接受，而不是在使用中介传递时。

If:

 * Actor `A1` sends messages `M1`, `M2`, `M3` to `A2`.
 * Actor`A1`向`A2`发送消息`M1`，`M2`，`M3`。
 * Actor `A3` sends messages `M4`, `M5`, `M6` to `A2`.
 * Actor`A3`向`A2`发送消息`M4`，`M5`，`M6`。

This means that, for Akka messages:

这意味着，对于Akka消息：

 * If `M1` is delivered it must be delivered before `M2` and `M3`.
 * 如果`M1`交付，那么它肯定在`M2`和`M3`之前交付。
 * If `M2` is delivered it must be delivered before `M3`.
 * 如果`M2`交付，那么它肯定在`M3`之前交付。
 * If `M4` is delivered it must be delivered before `M5` and `M6`.
 * 如果`M4`交付，那么它肯定在`M5`和`M6`之前交付。
 * If `M5` is delivered it must be delivered before `M6`.
 * 如果`M5`交付，那么它肯定在`M6`之前交付。
 * `A2` can see messages from `A1` interleaved with messages from `A3`.
 * `A2`可以看到来自`A1`与`A3`的顺序交错的消息。
 * Since there is no guaranteed delivery, any of the messages may be dropped, i.e. not arrive at `A2`.
 * 由于没有保证交付，任何消息都可能被丢弃，即没有到达`A2`。

These guarantees strike a good balance: having messages from one actor arrive in-order is convenient for building systems that can be easily reasoned about, while on the other hand allowing messages from different actors to arrive interleaved provides sufficient freedom for an efficient implementation of the actor system.

这些保证取得了很好的平衡：来自同一个Actor的消息按顺序到达，便于构建易于推理的系统，而另一方面，允许来自不同Actor的消息交错到达，为实现高效的Actor系统提供了足够的自由。

For the full details on delivery guarantees please refer to the @ref:[reference page](../general/message-delivery-reliability.md).

有关交付保证的完整详细信息，请参阅[参考页面](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html)。

## 为设备消息添加灵活性

Our first query protocol was correct, but did not take into account distributed application execution. If we want to implement resends in the actor that queries a device actor (because of timed out requests), or if we want to query multiple actors, we need to be able to correlate requests and responses. Hence, we add one more field to our messages, so that an ID can be provided by the requester  (we will add this code to our app in a later step):

我们的第一个查询协议是正确的，但没有考虑到分布式应用程序的执行情况。如果我们想在Actor中重新执行查询设备（由于超时请求），或者我们想查询多个actor，我们就需要能够关联请求和响应。因此，我们在消息中添加一个字段，以便请求者可以提供一个ID（我们将在稍后的步骤中将此代码添加到我们的应用程序中）：

Scala
:   @@snip [DeviceInProgress.scala]($code$/scala/tutorial_3/DeviceInProgress.scala) { #read-protocol-2 }

Java
:   @@snip [DeviceInProgress2.java]($code$/java/jdocs/tutorial_3/inprogress2/DeviceInProgress2.java) { #read-protocol-2 }

## 定义设备Actor和读取协议

As we learned in the Hello World example, each actor defines the type of messages it will accept. Our device actor has the responsibility to use the same ID parameter for the response of a given query, which would make it look like the following.

正如我们在Hello World示例中学到的，每个Actor都定义了它要接受的消息类型。我们的设备Actor有责任在给定查询的响应中使用相同的ID参数，这会使其看起来像下面那样。

Scala
:   @@snip [DeviceInProgress.scala]($code$/scala/tutorial_3/DeviceInProgress.scala) { #device-with-read }

Java
:   @@snip [DeviceInProgress2.java]($code$/java/jdocs/tutorial_3/inprogress2/DeviceInProgress2.java) { #device-with-read }

Note in the code that:

在代码中注意：

* The @scala[companion object]@java[static method] defines how to construct a `Device` actor. The `props` parameters include an ID for the device and the group to which it belongs, which we will use later.
* 在伴随对象中定义了如何构建一个`Device`演员。`props`的参数包括设备ID和设备所属的组，我们将在稍后使用它们。
* The @scala[companion object]@java[class] includes the definitions of the messages we reasoned about previously.
* 伴随对象也包含了对我们之前推理的消息的定义。
* In the `Device` class, the value of `lastTemperatureReading` is initially set to @scala[`None`]@java[`Optional.empty()`], and the actor will simply report it back if queried.
* 在`Device`类中，`lastTemperatureReading`初始值设置为`None`来应付后来的查询。

## 测试Actor

Based on the simple actor above, we could write a simple test. In the `com.lightbend.akka.sample` package in the test tree of your project, add the following code to a @scala[`DeviceSpec.scala`]@java[`DeviceTest.java`] file.@scala[(We use ScalaTest but any other test framework can be used with the Akka Testkit)].

基于上面简单的Actor，我们可以写一个简单的测试。在`com.lightbend.akka.sample`项目的测试目录中，将以下代码添加到`DeviceSpec.scala`文件中。（我们使用ScalaTest，但任何其他的测试框架也可以和Akka Testkit一起使用）。

You can run this test @java[by running `mvn test` or] by running `test` at the sbt prompt.

您可以通过在sbt提示符下运行`test`来运行此测试。

Scala
:   @@snip [DeviceSpec.scala]($code$/scala/tutorial_3/DeviceSpec.scala) { #device-read-test }

Java
:   @@snip [DeviceTest.java]($code$/java/jdocs/tutorial_3/DeviceTest.java) { #device-read-test }

Now, the actor needs a way to change the state of the temperature when it receives a message from the sensor.

现在，当Actor接收到来自传感器的消息时，它需要一种方法来改变温度状态。

## 添加写入协议

The purpose of the write protocol is to update the `currentTemperature` field when the actor receives a message that contains the temperature. Again, it is tempting to define the write protocol as a very simple message, something like this:

写入协议是为了在Actor收到包含温度的消息时更新`currentTemperature`字段。同样，也可以将写入协议定义的十分简单，例如：

Scala
:   @@snip [DeviceInProgress.scala]($code$/scala/tutorial_3/DeviceInProgress.scala) { #write-protocol-1 }

Java
:   @@snip [DeviceInProgress3.java]($code$/java/jdocs/tutorial_3/DeviceInProgress3.java) { #write-protocol-1 }

However, this approach does not take into account that the sender of the record temperature message can never be sure if the message was processed or not. We have seen that Akka does not guarantee delivery of these messages and leaves it to the application to provide success notifications. In our case, we would like to send an acknowledgment to the sender once we have updated our last temperature recording, e.g. @scala[`final case class TemperatureRecorded(requestId: Long)`]@java[`TemperatureRecorded`].
Just like in the case of temperature queries and responses, it is a good idea to include an ID field to provide maximum flexibility.

但是，这种方法并没有考虑到记录温度信息的Actor能不能确定消息被处理。我们已经知道，Akka是不能对此提供保证的，并将该机制留给应用程序让其来实现成功后的通知。在我们的场景中，一旦我们更新了上次的温度记录，我们将向发件人发送确认标示，例如`final case class TemperatureRecorded(requestId: Long)`，就像查询温度和其响应那样，这里包含一个ID字段以便提供最大的灵活性。

## 具有读取和写入消息的演员

Putting the read and write protocol together, the device actor looks like the following example:

将读写协议放在一起，设备Actor看起来像下面的例子：

Scala
:  @@snip [Device.scala]($code$/scala/tutorial_3/Device.scala) { #full-device }

Java
:  @@snip [Device.java]($code$/java/jdocs/tutorial_3/Device.java) { #full-device }

We should also write a new test case now, exercising both the read/query and write/record functionality together:

我们现在还应编写一个新的测试用例，同时执行读取/查询和写入/记录功能：

Scala:
:   @@snip [DeviceSpec.scala]($code$/scala/tutorial_3/DeviceSpec.scala) { #device-write-read-test }

Java:
:   @@snip [DeviceTest.java]($code$/java/jdocs/tutorial_3/DeviceTest.java) { #device-write-read-test }

## What's Next?

So far, we have started designing our overall architecture, and we wrote the first actor that directly corresponds to the domain. We now have to create the component that is responsible for maintaining groups of devices and the device actors themselves.

目前为止，我们已经开始设计我们的整体架构，并且编写了第一个与领域直接相关的Actor。我们现在还要创建负责维护设备和设备组的组件。