# 消息传递可靠性

Akka helps you build reliable applications which make use of multiple processor cores in one machine (“scaling up”) or distributed across a computer network (“scaling out”). The key abstraction to make this work is that all interactions between your code units—actors—happen via message passing, which is why the precise semantics of how messages are passed between actors deserve their own chapter.

Akka利用机器的多个处理器内核（“纵向扩展”）和计算机网络的分布（“横向扩展”）帮助你构建可靠的应用程序。这项工作的关键是，你的代码单元（Actor）之间的所有交互都是通过消息传递发生的，这就是为什么消息在Actor之间传递的精确语义应该用具体的一个章节来介绍。

In order to give some context to the discussion below, consider an application which spans multiple network hosts. The basic mechanism for communication is the same whether sending to an actor on the local JVM or to a remote actor, but of course there will be observable differences in the latency of delivery
(possibly also depending on the bandwidth of the network link and the message size) and the reliability. In case of a remote message send there are obviously more steps involved which means that more can go wrong. Another aspect is that local sending will just pass a reference to the message inside the same JVM,
without any restrictions on the underlying object which is sent, whereas a remote transport will place a limit on the message size.

为了给下面的讨论提供一些背景知识，请想象跨越多个网络主机的应用程序。基本的通信机制是相同的，无论是发送给本地JVM上的Actor还是发送给远程Actor，当然在传输延迟（可能取决于网络带宽和消息大小）和稳定性上还是会有明显差异的。如果发送远程消息，会涉及更多步骤，这意味着更多出错的可能性。另一方面是本地发送仅仅是在同一JVM内传递消息的引用，而不会对发送的基础对象进行任何限制，而远程传输则对消息大小有限制。

Writing your actors such that every interaction could possibly be remote is the safe, pessimistic bet. It means to only rely on those properties which are always guaranteed and which are discussed in detail below.  This has of course some overhead in the actor’s implementation. If you are willing to sacrifice full
location transparency—for example in case of a group of closely collaborating actors—you can place them always on the same JVM and enjoy stricter guarantees on message delivery. The details of this trade-off are discussed further below.

编写你的Actor，让每一次互动都可能是远程的，这是安全的也是悲观的做法。这意味着只能依赖那些始终能够保证的属性，这些在下面详细讨论。这对Actor的实现肯定会造成一些开销。如果你愿意牺牲完整的位置透明性（例如一组密切协作的Actor），那么可以将它们放在同一个JVM上，并享受更严格的消息传递保证。这种折衷方案我们在接下来进一步讨论。

As a supplementary part we give a few pointers at how to build stronger reliability on top of the built-in ones. The chapter closes by discussing the role of the “Dead Letter Office”.

作为补充部分，我们将就如何基于内置的可靠性机制建立更强大的可靠性机制提供一些指导。本章最后讨论了“死信办公室”的作用。

## 一般性规则

These are the rules for message sends (i.e. the `tell` or `!` method, which also underlies the `ask` pattern):

这些是消息发送的规则（`tell`or `!`方法，也是`ask`模式的基础）：

 * **at-most-once delivery**, i.e. no guaranteed delivery
 * **最多一次交货**，即没有交付保证
 * **message ordering per sender–receiver pair**
 * **每对发送-接收者的消息排序**

The first rule is typically found also in other actor implementations while the second is specific to Akka.

第一条规则通常在其他actor实现中也可以找到，而第二条规则则专门针对Akka。

### 讨论：“最多一次”意味着什么?

When it comes to describing the semantics of a delivery mechanism, there are three basic categories:

描述交付机制的语义时，有三个基本类别：

 * **at-most-once** delivery means that for each message handed to the mechanism, that message is delivered zero or one times; in more casual terms it means that messages may be lost.
 * **最多一次** 意味着对于交给该机制的每条消息，该消息被递送零或一次; 也就是说，这意味着信息可能会丢失。
 * **at-least-once** delivery means that for each message handed to the mechanism potentially multiple attempts are made at delivering it, such that at least one succeeds; again, in more casual terms this means that messages may be duplicated but not lost.
 * **至少一次** 意味着对于交付给该机制的每条消息，会有潜在地多次交付尝试，使得至少一个成功; 也就是说，这意味着消息可能重复但不会丢失。
 * **exactly-once** delivery means that for each message handed to the mechanism exactly one delivery is made to the recipient; the message can neither be lost nor duplicated.
 * **恰好一次** 意味着对于交给该机制的每条消息，只交付一次给收件人; 该消息既不会丢失也不会重复。

The first one is the cheapest—highest performance, least implementation overhead—because it can be done in a fire-and-forget fashion without keeping state at the sending end or in the transport mechanism. The second one requires retries to counter transport losses, which means keeping state at the sending
end and having an acknowledgement mechanism at the receiving end. The third is most expensive—and has consequently worst performance—because in addition to the second it requires state to be kept at the receiving end in order to filter out duplicate deliveries.

第一个是最简单、消耗最少的实现方式 - 因为它可以以一种发完即忘的方式发送消息，而不需在发送端或传输机制中保持状态。第二个要求利用重试来抵消交付失败，这意味着要在发送端持有状态并且在接收端具有确认机制。第三个代价最昂贵，因此性能最差 - 因为在第二个的基础上，它还需要将状态保持在接收端，以便过滤掉重复的交付。

### 讨论：为什么没有保证交付？

At the core of the problem lies the question what exactly this guarantee shall mean:

问题的核心在于这个保证究竟意味着什么：

  1. The message is sent out on the network?
  2. 该消息已经发到网络上了？
  3. The message is received by the other host?
  4. 该消息被另一个主机接收了？
  5. The message is put into the target actor's mailbox?
  6. 该消息被放入目标Actor的邮箱中了？
  7. The message is starting to be processed by the target actor?
  8. 消息开始由目标Actor处理了？
  9. The message is processed successfully by the target actor?
  10. 消息被目标Actor处理成功了？

Each one of these have different challenges and costs, and it is obvious that there are conditions under which any message passing library would be unable to comply; think for example about configurable mailbox types and how a bounded mailbox would interact with the third point, or even what it would mean to decide upon the “successfully” part of point five.

每种都有不同的实现难度和成本，显而易见的是，在任何消息传递库都不能遵守的情况下，想想可配置的邮箱类型，以及有界的邮箱如何与第三点交互，或者甚至第五点的“成功”部分到底意味着什么。

Along those same lines goes the reasoning in [Nobody Needs Reliable Messaging](http://www.infoq.com/articles/no-reliable-messaging). The only meaningful way for a sender to know whether an interaction was successful is by receiving a business-level acknowledgement message, which is not something Akka could make up on its own (neither are we
writing a “do what I mean” framework nor would you want us to).

根据[没有人需要可靠消息](http://www.infoq.com/articles/no-reliable-messaging)中的推理。发件人知道交互成功的唯一有意义的方式是接收一个业务级的确认消息，这不是Akka自己可以提供的东西（也不是我们写一个“照我说的做”框架，或你想让我们做）。

Akka embraces distributed computing and makes the fallibility of communication explicit through message passing, therefore it does not try to lie and emulate a leaky abstraction. This is a model that has been used with great success in Erlang and requires the users to design their applications around it. You can
read more about this approach in the [Erlang documentation](http://www.erlang.org/faq/academic.html) (section 10.9 and 10.10), Akka follows it closely.

Akka支持分布式计算，并通过消息传递使得通信的易错性更明显，因此它不会说谎和模拟抽象漏洞。这是一个在Erlang中取得巨大成功的模型，并要求用户围绕它设计应用程序。您可以在[Erlang文档](http://www.erlang.org/faq/academic.html)（第10.9和10.10节）中阅读关于这种方法的更多信息，Akka密切关注它。

Another angle on this issue is that by providing only basic guarantees those use cases which do not need stronger reliability do not pay the cost of their implementation; it is always possible to add stronger reliability on top of basic ones, but it is not possible to retro-actively remove reliability in order to gain more performance.

这个问题换个角度看，只提供基本的保证，那些不需要更高可靠性的用例就不会有更高的成本; 总是可以在基本的可靠性基础上增加更高的可靠性，但不能为了获得更高的性能而反向移除可靠性。

<a id="message-ordering"></a>
### 讨论：消息顺序

The rule more specifically is that *for a given pair of actors, messages sent directly from the first to the second will not be received out-of-order.* The word *directly* emphasizes that this guarantee only applies when sending with the *tell* operator to the final destination, not when employing mediators or other message dissemination features (unless stated otherwise).

具体地说，对于给定的一对Actor，从第一个到第二个直接发送的消息将不会乱序接收。*“直接”*一词强调，这个保证只适用于利用*tell*方法发送到最终目的地时，而不是采用中介或其他消息传播功能（除非另有说明）。

The guarantee is illustrated in the following:

如下所示：

> Actor `A1` sends messages `M1`, `M2`, `M3` to `A2`
>
> Actor`A1`发送消息`M1`，`M2`，`M3`至`A2`
>
> Actor `A3` sends messages `M4`, `M5`, `M6` to `A2`
>
> Actor`A3`发送消息`M4`，`M5`，`M6`至`A2`

This means that:

这意味着：

  1. If `M1` is delivered it must be delivered before `M2` and `M3`
  2. 如果`M1`交付，它必须发生在交付`M2`和交付`M3`之前
  3. If `M2` is delivered it must be delivered before `M3`
  4. 如果`M2`交付，则必须发生在交付`M3`之前
  5. If `M4` is delivered it must be delivered before `M5` and `M6`
  6. 如果`M4`交付，它必须发生在交付`M5`和交付`M6`之前
  7. If `M5` is delivered it must be delivered before `M6`
  8. 如果`M5`交付，则必须发生在交付`M6`之前
  9. `A2` can see messages from `A1` interleaved with messages from `A3`
  10. `A2`可以交错地看到来自`A1`与`A3`的消息
  11. Since there is no guaranteed delivery, any of the messages may be dropped, i.e. not arrive at `A2`
  12. 由于没有保证交付，任何消息都可能被丢弃，即没有到达 `A2`


@@@ note

It is important to note that Akka’s guarantee applies to the order in which messages are enqueued into the recipient’s mailbox. If the mailbox implementation does not respect FIFO order (e.g. a `PriorityMailbox`),
then the order of processing by the actor can deviate from the enqueueing order.

值得注意的是，Akka的保证适用于邮件排入收件人邮箱的顺序。如果邮箱不采用FIFO排序（例如 `PriorityMailbox`），则Actor的处理顺序可能会与排队顺序有所不同。

@@@

Please note that this rule is **not transitive**:

请注意，此规则**不具有传递性**：

> Actor `A` sends message `M1` to actor `C`
>
> Actor`A`发送消息`M1`给Actor`C`
>
> Actor `A` then sends message `M2` to actor `B`
>
> 之后Actor`A`发送消息`M2`给Actor`B`
>
> Actor `B` forwards message `M2` to actor `C`
>
> Actor`B`将消息`M2`转发给Actor`C`
>
> Actor `C` may receive `M1` and `M2` in any order
>
> Actor`C`可以以任何顺序接收`M1`和`M2`

Causal transitive ordering would imply that `M2` is never received before `M1` at actor `C` (though any of them might be lost). This ordering can be violated due to different message delivery latencies when `A`, `B` and `C` reside on different network hosts, see more below.

因果传递排序(**什么鬼这是？？**)意味着Actor C 不会在 `M1` 之前 接受 `M2`（尽管他们中的任何一个都可能会丢失）。这个顺序可能因不同的消息传递延迟时(`A`、`B`、`C`驻留在不同的网络主机上时)而受到破坏，详见下文。

@@@ note

Actor creation is treated as a message sent from the parent to the child, with the same semantics as discussed above. Sending a message to an actor in a way which could be reordered with this initial creation message means that the message might not arrive because the actor does not exist yet. An example
where the message might arrive too early would be to create a remote-deployed actor R1, send its reference to another remote actor R2 and have R2 send a message to R1. An example of well-defined ordering is a parent which creates an actor and immediately sends a message to it.

Actor的创建被视为从父级发送给子级的消息，与上述具有相同的语义。向一个刚创建的Actor发送消息，而这个消息与Actor的初始创建消息顺序错乱，是由于这个Actor可能还不存在而导致消息没有到达。消息可能到达太早的例子是创建远程部署的Actor R1，然后将该引用发送给另一个远程Actor R2并让R2向R1发送消息。而顺序正常的例子是创建一个Actor并立即向其发送消息。

@@@

#### 失败的通讯（Communication of Failure）

Please note, that the ordering guarantees discussed above only hold for user messages between actors. Failure of a child of an actor is communicated by special system messages that are not ordered relative to ordinary user messages. In particular:

请注意，上面讨论的顺序保证只适用于Actor之间的用户消息。Actor的子级的失败是通过特殊的系统消息来传递的，这些消息的顺序与普通的用户消息无关。尤其是：

> Child actor `C` sends message `M` to its parent `P`
>
> 子级Actor`C`向其父级`P`发送消息`M`
>
> Child actor fails with failure `F`
>
> 子级Actor因故障 `F`而失败
>
> Parent actor `P` might receive the two events either in order `M`, `F` or `F`, `M`
>
> 父级Actor`P`可能会按顺序`M`，`F`或者`F`，`M`接收这两个事件

The reason for this is that internal system messages has their own mailboxes therefore the ordering of enqueue calls of a user and system message cannot guarantee the ordering of their dequeue times.

原因在于内部系统消息具有自己的邮箱，因此用户和系统消息的入队顺序不能保证出队顺序。

## In-JVM（本地）消息的发送规则

### 小心你在这一节做了什么！

Relying on the stronger reliability in this section is not recommended since it will bind your application to local-only deployment: an application may have to be designed differently (as opposed to just employing some message exchange patterns local to some actors) in order to be fit for running on a cluster of machines. Our credo is “design once, deploy any way you wish”, and to achieve this you should only rely on [The General Rules](#the-general-rules).

不建议依靠本节中更强的可靠性，因为它会将你的应用程序绑定到仅限本地的部署：应用程序可能要以不同的方式来设计（与只使用本地信息交换模式相反）以便在集群上运行。我们的信条是“设计一次，随便部署”，为了实现这个目标，你应该只遵照[“一般性规则”](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html#the-general-rules)。

### 本地消息发送的可靠性

The Akka test suite relies on not losing messages in the local context (and for non-error condition tests also for remote deployment), meaning that we actually do apply the best effort to keep our tests stable. A local `tell` operation can however fail for the same reasons as a normal method call can on the JVM:

Akka测试套件依赖于本地上下文中不丢失消息（也用于无错误情况的远程部署），这意味着我们实际上会尽最大努力来保持我们的测试稳定。然而，本地`tell`操作也可能会像JVM上的常规方法调用那样而失败：

 * `StackOverflowError`
 * `OutOfMemoryError`
 * other `VirtualMachineError`

In addition, local sends can fail in Akka-specific ways:

另外，本地发送可能会以Akka特定的方式失败：

 * if the mailbox does not accept the message (e.g. full BoundedMailbox)
 * 如果邮箱不接受邮件（例如，BoundedMailbox满了）
 * if the receiving actor fails while processing the message or is already terminated
 * 如果接收方在处理消息时失败或者已经终止

While the first is clearly a matter of configuration the second deserves some thought: the sender of a message does not get feedback if there was an exception while processing, that notification goes to the supervisor instead.This is in general not distinguishable from a lost message for an outside observer.

然而第一个是配置问题，但第二个问题却值得考虑一下：如果处理过程中出现异常，则消息的发送者不会得到反馈，通知会被传给监督者。这对于外部观察者来说，通常难以与丢失的信息区分开来。

### 本地消息发送的顺序

Assuming strict FIFO mailboxes the aforementioned caveat of non-transitivity of the message ordering guarantee is eliminated under certain conditions. As you will note, these are quite subtle as it stands, and it is even possible that future performance optimizations will invalidate this whole paragraph. The possibly non-exhaustive list of counter-indications is:

假设严格的FIFO信箱在某些条件下消除了消息顺序保证的非传递性的警告。正如你将注意到的，它们现在非常微妙，未来的性能优化甚至有可能会使整个段落失效。可能非详尽的反指示清单是：

 * Before receiving the first reply from a top-level actor, there is a lock which protects an internal interim queue, and this lock is not fair; the implication is that enqueue requests from different senders which arrive during the actor’s construction (figuratively, the details are more involved) may be reordered depending on low-level thread scheduling. Since completely fair locks do not exist on the JVM this is unfixable.
 * 在收到顶级Actor的第一个回复之前，会有一个锁保护这一个内部临时队列，这个锁是不公平; 这意味着在Actor构建期间入队的不同发送者的请求可能会根据低级线程调度来重新排序（这是形象一点说，细节更复杂）。由于JVM上不存在完全公平的锁，因此这是不可修复的。
 * The same mechanism is used during the construction of a Router, more precisely the routed ActorRef, hence the same problem exists for actors deployed with Routers.
 * 路由器构建时使用了相同的机制，更确切地说是路由功能的ActorRef，因此对于使用路由器部署的Actor也存在同样的问题。
 * As mentioned above, the problem occurs anywhere a lock is involved during enqueueing, which may also apply to custom mailboxes.
 * 如上所述，问题发生在入队期间涉及锁的任何位置，这也会出现在自定义邮箱。

This list has been compiled carefully, but other problematic scenarios may have escaped our analysis.

这份清单已经过仔细编制，但其他情况下的问题可能在我们的分析之外。

### 本地顺序如何与网络顺序相关联

The rule that *for a given pair of actors, messages sent directly from the first to the second will not be received out-of-order* holds for messages sent over the network with the TCP based Akka remote transport protocol.

基于TCP协议和Akka远程传输协议，*对于给定的一对Actor，从第一个Actor到第二个Actor直接发送的消息不会被乱序接收。*

As explained in the previous section local message sends obey transitive causal ordering under certain conditions. This ordering can be violated due to different message delivery latencies. For example:

正如前面部分所解释的，在特定条件下，本地消息发送服从因果传递顺序。由于消息传递延迟不同，此顺序可能会被打破。例如：

> Actor `A` on node-1 sends message `M1` to actor `C` on node-3
>
> 节点1上的Actor`A` 向节点3上的Actor`C` 发送消息`M1`
>
> Actor `A` on node-1 then sends message `M2` to actor `B` on node-2
>
> 节点1上的Actor`A` 向节点2上的Actor`B` 发送消息`M2`
>
> Actor `B` on node-2 forwards message `M2` to actor `C` on node-3
>
> 节点2上的Actor`B` 向节点3上的Actor`C` 发送消息`M2`
>
> Actor `C` may receive `M1` and `M2` in any order
>
> Actor`C`可以以任何顺序接收`M1`、`M2`

It might take longer time for `M1` to "travel" to node-3 than it takes for `M2` to "travel" to node-3 via node-2.

`M1`“传播”到节点3可能比`M2`通过节点2“传播”到节点3 需要更长的时间。

## Higher-level abstractions

## 更高层次的抽象

Based on a small and consistent tool set in Akka's core, Akka also provides powerful, higher-level abstractions on top it.

基于Akka核心中的一个小而一致的工具集，Akka还提供了强大的，更高层次的抽象。

### 消息模式

As discussed above a straight-forward answer to the requirement of reliable delivery is an explicit ACK–RETRY protocol. In its simplest form this requires 

如上所述，对可靠传送的直接回答是明确的ACK-RETRY协议。其最简单的形式需要：

 * a way to identify individual messages to correlate message with acknowledgement
 * 一种识别单个消息并与确认标示相关联的方式
 * a retry mechanism which will resend messages if not acknowledged in time
 * 一种如果没有及时得到确认标示便将重新发送消息的重试机制
 * a way for the receiver to detect and discard duplicates
 * 一种让接收端检测并去重的方式

The third becomes necessary by virtue of the acknowledgements not being guaranteed to arrive either. An ACK-RETRY protocol with business-level acknowledgements is supported by @ref:[At-Least-Once Delivery](../persistence.md#at-least-once-delivery) of the Akka Persistence module. Duplicates can be detected by tracking the identifiers of messages sent via @ref:[At-Least-Once Delivery](../persistence.md#at-least-once-delivery). Another way of implementing the third part would be to make processing the messages idempotent on the level of the business logic.

由于确认标示不能保证到达，因此第三点很必要。具有业务级别确认的ACK-RETRY协议由Akka持久性模块的[至少一次交付](https://doc.akka.io/docs/akka/current/persistence.html#at-least-once-delivery)提供支持。重复也可以通过[至少一次交付](https://doc.akka.io/docs/akka/current/persistence.html#at-least-once-delivery)中的跟踪信息标示来检测。还有就是在业务逻辑层面上幂等处理消息。

### 事件驱动

Event sourcing (and sharding) is what makes large websites scale to billions of users, and the idea is quite simple: when a component (think actor) processes a command it will generate a list of events representing the effect of the command. These events are stored in addition to being applied to the component’s state. The nice thing about this scheme is that events only ever are appended to the storage, nothing is ever mutated; this enables perfect replication and scaling of consumers of this event stream (i.e. other
components may consume the event stream as a means to replicate the component’s state on a different continent or to react to changes). If the component’s state is lost—due to a machine failure or by being pushed out of a cache—it can easily be reconstructed by replaying the event stream (usually employing
snapshots to speed up the process). @ref:[Event sourcing](../persistence.md#event-sourcing) is supported by Akka Persistence.

事件驱动（和分片）是大型网站能扩展到数十亿用户的原因，这个想法很简单：当一个组件（想象Actor）处理命令时，它会产生表示这个命令效果的一系列事件。这些事件除了应用于组件的状态之外，还会被存储起来。这种方案的好处在于，事件只是被追加到存储里，而没有任何变化; 这可以完美地复制和扩展此事件流的消费者（其他组件可能会以消费事件流作为在不同区域的组件复制状态或对更改作出反应的方式）。如果组件的状态由于机器故障或被清出缓存而丢失，则可以通过重播事件流（通常采用快照来加速进程）来重构组件的状态。Akka持久性支持[事件驱动](https://doc.akka.io/docs/akka/current/persistence.html#event-sourcing)。

### 具有明确确认标示的信箱

By implementing a custom mailbox type it is possible to retry message processing at the receiving actor’s end in order to handle temporary failures. This pattern is mostly useful in the local communication context where delivery guarantees are otherwise sufficient to fulfill the application’s requirements.

通过实现自定义邮箱类型，可以在接收端重试消息处理，以处理临时故障。这种模式在本地通讯上下文中非常有用，其中交付保证足以满足应用程序的要求。

Please note that the caveats for [The Rules for In-JVM (Local) Message Sends](#the-rules-for-in-jvm-local-message-sends) do apply.

请注意，[“In-JVM（本地）消息发送规则”](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html#the-rules-for-in-jvm-local-message-sends)的注意事项。

<a id="deadletters"></a>
## 死亡信件

Messages which cannot be delivered (and for which this can be ascertained) will be delivered to a synthetic actor called `/deadLetters`. This delivery happens on a best-effort basis; it may fail even within the local JVM (e.g. during actor termination). Messages sent via unreliable network transports will be lost without turning up as dead letters.

当消息无法被交付（可以确认）时，消息将被传递给一个合成Actor`/deadLetters`。交付总是竭尽全力; 即使在本地JVM中也可能失败（例如，在Actor终止时）。通过不可靠的网络传输发送的消息将丢失，而不会成为死信。

### 我能用死亡信件做什么?

The main use of this facility is for debugging, especially if an actor send does not arrive consistently (where usually inspecting the dead letters will tell you that the sender or recipient was set wrong somewhere along the way). In order to be useful for this purpose it is good practice to avoid sending to deadLetters where possible, i.e. run your application with a suitable dead letter logger (see more below) from time to time and clean up the log output. This exercise—like all else—requires judicious application of common sense: it
may well be that avoiding to send to a terminated actor complicates the sender’s code more than is gained in debug output clarity.

这个工具的主要用途是调试，特别是如果一个Actor发送的信息没有到达（通常检查死信会告诉你，发件人或收件人哪里设置错了）。为了达到这个目的，最好避免给死信发消息，例如不时地在程序中利用死信日志记录（见下面更多内容）并清理日志输出。这个练习 - 就像所有其他 - 需要正确地应用常识：避免向终止的Actor发送消息，比起调试输出的清晰，发送者的代码会更复杂(**呵呵呵呵呵呵……**)。

The dead letter service follows the same rules with respect to delivery guarantees as all other message sends, hence it cannot be used to implement guaranteed delivery. 

死信服务遵循与所有其他消息发送相同的交付保证规则，因此它不能用于实现保证交付。

### 我如何收到死亡信件？

An actor can subscribe to class `akka.actor.DeadLetter` on the event stream, see @ref:[Event Stream ](../event-bus.md#event-stream) for how to do that. The subscribed actor will then receive all dead letters published in the (local) system from that point onwards. Dead letters are not propagated over the network, if you want to collect them in one place you will have to subscribe one actor per network node and forward them manually. Also consider that dead letters are generated at that node which can determine that a send operation is failed, which for a remote send can be the local system (if no network connection can be established) or the remote one
(if the actor you are sending to does not exist at that point in time).

Actor可以在事件流中订阅`akka.actor.DeadLetter`，请参阅[事件流](https://doc.akka.io/docs/akka/current/event-bus.html#event-stream)了解如何执行此操作。订阅的Actor将从那时起收到在（本地）系统中发布的所有死信。死信不会通过网络传播，如果你想要收集它们到某处，则必须在每个网络节点上创建一个订阅Actor并手动转发。还需要考虑该节点是否可以决定发送失败而生成死信，对于一个远程发送，决定发送失败的节点可能是本地系统（不能建立网络连接）也可能是远程系统（目标Actor在那个时间点不存在）。

### 死亡信件（通常）不找麻烦

Every time an actor does not terminate by its own decision, there is a chance that some messages which it sends to itself are lost. There is one which happens quite easily in complex shutdown scenarios that is usually benign: seeing a `akka.dispatch.Terminate` message dropped means that two termination requests were given, but of course only one can succeed. In the same vein, you might see `akka.actor.Terminated` messages from children while stopping a hierarchy of actors turning up in dead letters if the parent is still watching the child when the parent terminates.

每次一个Actor没有通过自己的决定终止时，就有可能出现发送给自己的一些消息丢失。在复杂的关机情况下很容易发生这种情况，通常是良性的：看到`akka.dispatch.Terminate`消息被丢弃意味着有两个终止请求，但只有一个成功。同样，如果父级在终止时仍然在监视子级，当停止Actor层级时候，你可能会看到子级的`akka.actor.Terminated`消息转变为死信。