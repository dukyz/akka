# 监督和监视

This chapter outlines the concept behind supervision, the primitives offered and their semantics. For details on how that translates into real code, please refer to the corresponding chapters for Scala and Java APIs.

本章概述了监督背后的概念，提供的原语及其语义。有关如何转换为真实代码的详细信息，请参阅Scala和Java API的相应章节。

<a id="supervision-directives"></a>
## 监督的手段？监督是什么意思（Supervision Means）

As described in @ref:[Actor Systems](actor-systems.md) supervision describes a dependency relationship between actors: the supervisor delegates tasks to subordinates and therefore must respond to their failures.  When a subordinate detects a failure (i.e. throws an exception), it suspends itself and all its subordinates and
sends a message to its supervisor, signaling failure.  Depending on the nature of the work to be supervised and the nature of the failure, the supervisor has a choice of the following four options:

正如[Actor系统中](https://doc.akka.io/docs/akka/current/general/actor-systems.html)所描述的，监督描述了参与者之间的依赖关系：主管将任务委派给下属，因此必须对他们的失败做出响应。当下属检测到故障（即引发异常）时，它会挂起自己及其所有下属，并向其管理员发送消息，表明发生故障。根据要监督的工作性质和失败的性质，主管可以选择以下四种选择：

  1. Resume the subordinate, keeping its accumulated internal state
  2. 恢复下属，保持其积累的内部状态
  3. Restart the subordinate, clearing out its accumulated internal state
  4. 重新启动下属，清除其累积的内部状态
  5. Stop the subordinate permanently
  6. 永久停止下属
  7. Escalate the failure, thereby failing itself
  8. 升级失败，从而失败

It is important to always view an actor as part of a supervision hierarchy,which explains the existence of the fourth choice (as a supervisor also is subordinate to another supervisor higher up) and has implications on the first three: resuming an actor resumes all its subordinates, restarting an actor entails restarting all its subordinates (but see below for more details),similarly terminating an actor will also terminate all its subordinates. It should be noted that the default behavior of the `preRestart` hook of the `Actor` class is to terminate all its children before restarting, but this hook can be overridden; the recursive restart applies to all children left after this hook has been executed.

重要的是始终把一个参与者视为监督等级的一部分，这就解释了第四个选择的存在（因为监督者也从属于另一个上级主管）并且对前三个有影响：恢复一个参与者恢复所有参与者下属，重新启动一个演员需要重新启动所有的下属（但更多细节见下文），类似地终止演员也将终止其所有下属。应该注意的是，类的`preRestart`钩子的默认行为`Actor`是在重新启动之前终止其所有子级，但是可以覆盖此钩子; 递归重启适用于执行该钩子后留下的所有子项。

Each supervisor is configured with a function translating all possible failure causes (i.e. exceptions) into one of the four choices given above; notably, this function does not take the failed actor’s identity as an input. It is quite easy to come up with examples of structures where this might not seem flexible enough, e.g. wishing for different strategies to be applied to different subordinates. At this point it is vital to understand that supervision is about forming a recursive fault handling structure. If you try to do too much at one level, it will become hard to reason about, hence the recommended way in this case is to add a level of supervision.

每个主管都配置了一个功能，将所有可能的故障原因（即例外）转换为上述四种选择之一; 值得注意的是，这个函数不会将失败的actor的身份作为输入。提出可能不够灵活的结构示例是很容易的，例如希望将不同的策略应用于不同的下属。在这一点上，理解监督是关于形成递归故障处理结构是非常重要的。如果你想在一个层面做太多的事情，就很难推理，因此在这种情况下推荐的方法是增加监督水平。

Akka implements a specific form called “parental supervision”. Actors can only be created by other actors—where the top-level actor is provided by the library—and each created actor is supervised by its parent. This restriction makes the formation of actor supervision hierarchies implicit and encourages sound design decisions. It should be noted that this also guarantees that actors cannot be orphaned or attached to supervisors from the outside, which might otherwise catch them unawares. In addition, this yields a natural and clean shutdown procedure for (sub-trees of) actor applications.

阿卡实施了一种称为“家长监督”的具体形式。参与者只能由其他参与者创建 - 顶级参与者由图书馆提供 - 每个创建的参与者都由其父级监督。这种限制使得行为者监督层次的形成是隐含的，并鼓励合理的设计决策。应该指出的是，这也保证演员不会孤立或附属于外部监督，否则他们可能会不知不觉地抓住他们。另外，这为actor应用程序的（子树）产生了一个自然且干净的关闭过程。

@@@ warning

Supervision related parent-child communication happens by special system messages that have their own mailboxes separate from user messages. This implies that supervision related events are not  deterministically ordered relative to ordinary messages. In general, the user cannot influence the order of normal messages and failure notifications. For details and example see the @ref:[Discussion: Message Ordering](message-delivery-reliability.md#message-ordering) section.

监督相关的亲子沟通发生在特殊的系统消息中，这些消息拥有与用户消息分开的自己的邮箱。这意味着与监督有关的事件不是相对于普通的消息而确定性地排序的。通常，用户不能影响正常消息和失败通知的顺序。有关详细信息和示例，请参阅[讨论：消息订购](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html#message-ordering)部分。

@@@

<a id="toplevel-supervisors"></a>
## 顶级监督者

![guardians.png](guardians.png)

An actor system will during its creation start at least three actors, shown in the image above. For more information about the consequences for actor paths see @ref:[Top-Level Scopes for Actor Paths](addressing.md#toplevel-paths).

演员系统将在其创建过程中开始至少三名演员，如上图所示。有关演员路径后果的更多信息，请参阅演员路径的[顶层范围](https://doc.akka.io/docs/akka/current/general/addressing.html#toplevel-paths)。

<a id="user-guardian"></a>
### `/user`: 守卫Actor

The actor which is probably most interacted with is the parent of all user-created actors, the guardian named `"/user"`. Actors created using `system.actorOf()` are children of this actor. This means that when this guardian terminates, all normal actors in the system will be shutdown, too. It also means that this guardian’s supervisor strategy determines how the top-level normal actors are supervised. Since Akka 2.1 it is possible to configure this using the setting `akka.actor.guardian-supervisor-strategy`,which takes the fully-qualified class-name of a `SupervisorStrategyConfigurator`. When the guardian escalates a failure, the root guardian’s response will be to terminate the guardian, which in effect will shut down the whole actor system.

可能与其最相互作用的演员是所有用户创建的演员的父母，该名为监护人`"/user"`。创建`system.actorOf()`的演员是这个演员的子女。这意味着当这个监护人终止时，系统中的所有正常参与者也将被关闭。这也意味着这位监护人的主管策略决定了顶级正常参与者如何受到监督。由于Akka 2.1可以使用设置进行配置，该设置`akka.actor.guardian-supervisor-strategy`采用a的完全限定类名`SupervisorStrategyConfigurator`。当监护人升级失败时，根监护人的回应将是终止监护人，这实际上将关闭整个参与者系统。

### `/system`: 系统守卫

This special guardian has been introduced in order to achieve an orderly shut-down sequence where logging remains active while all normal actors terminate, even though logging itself is implemented using actors. This is realized by having the system guardian watch the user guardian and initiate its own
shut-down upon reception of the `Terminated` message. The top-level system actors are supervised using a strategy which will restart indefinitely upon all types of `Exception` except for `ActorInitializationException` and `ActorKilledException`, which will terminate the child in question.  All other throwables are escalated,which will shut down the whole actor system.

这个特殊的监护人已经被引入，以便实现有序的关闭序列，即在日志本身是使用参与者实现的情况下，在所有普通参与者终止的情况下日志保持活动状态。这是通过让系统监护人监视用户监护人并在接收到该`Terminated`消息时启动其自身关闭来实现的。顶级系统参与者使用一种策略进行监督，该策略将无限期地重启所有类型，`Exception`除了`ActorInitializationException`和`ActorKilledException`将终止有问题的孩子。所有其他的投掷物都会升级，这将关闭整个演员系统。

### `/`: 根守护

The root guardian is the grand-parent of all so-called “top-level” actors and supervises all the special actors mentioned in @ref:[Top-Level Scopes for Actor Paths](addressing.md#toplevel-paths) using the `SupervisorStrategy.stoppingStrategy`, whose purpose is to terminate the child upon any type of `Exception`. All other throwables will be
escalated … but to whom? Since every real actor has a supervisor, the supervisor of the root guardian cannot be a real actor. And because this means that it is “outside of the bubble”, it is called the “bubble-walker”. This is a synthetic `ActorRef` which in effect stops its child upon the first sign of trouble and sets the actor system’s `isTerminated` status to `true` as soon as the root guardian is fully terminated (all children recursively stopped).

根监护人是所有所谓的“顶级”演员的祖父，并监督使用该[演员路径的顶级范围中](https://doc.akka.io/docs/akka/current/general/addressing.html#toplevel-paths)提及的所有特殊演员`SupervisorStrategy.stoppingStrategy`，其目的是终止任何类型的儿童`Exception`。所有其他的throwables将升级...但是谁？由于每个真正的演员都有一名主管，所以根监护人的主管不能成为真正的演员。因为这意味着它是“泡沫之外”，所以它被称为“冒泡者”。这是一种综合性的方法`ActorRef`，它实际上在遇到问题的第一个迹象时阻止其孩子，并在根监护人完全终止（所有孩子递归停止）后立即设置角色系统的`isTerminated`状态`true`。

<a id="supervision-restart"></a>
## 重启意味着？重启的手段？（What Restarting Means）

When presented with an actor which failed while processing a certain message,causes for the failure fall into three categories:

当呈现在处理某个消息时失败的演员时，导致失败的原因分为三类：

 * Systematic (i.e. programming) error for the specific message received
 * 系统（即编程）错误收到特定的消息
 * (Transient) failure of some external resource used during processing the message
 * （瞬态）在处理消息期间使用的某些外部资源失败
 * Corrupt internal state of the actor
 * 腐败的演员内部状态

Unless the failure is specifically recognizable, the third cause cannot be ruled out, which leads to the conclusion that the internal state needs to be cleared out. If the supervisor decides that its other children or itself is not affected by the corruption—e.g. because of conscious application of the error kernel pattern—it is therefore best to restart the child. This is carried out by creating a new instance of the underlying `Actor` class and replacing the failed instance with the fresh one inside the child’s `ActorRef`;the ability to do this is one of the reasons for encapsulating actors within special references. The new actor then resumes processing its mailbox, meaning that the restart is not visible outside of the actor itself with the notable exception that the message during which the failure occurred is not re-processed.

除非故障具体可识别，否则不能排除第三个原因，这导致了内部状态需要清除的结论。如果主管决定其他孩子或自己不受腐败的影响 - 例如由于有意识地应用错误内核模式 - 最好重新启动孩子。这是通过创建一个新的底层`Actor`类的实例并用子内的新实例替换失败的实例来实现的`ActorRef`; 执行此操作的能力是将参与者封装在特殊参考中的原因之一。新角色然后恢复处理其邮箱，这意味着重启在actor外部是不可见的，除了重新处理发生故障的消息之外的明显例外。

The precise sequence of events during a restart is the following:

重启过程中的精确事件顺序如下：

  1. suspend the actor (which means that it will not process normal messages until resumed), and recursively suspend all children
  2. 暂停演员（这意味着它不会处理正常的消息直到恢复），并递归地挂起所有的孩子
  3. call the old instance’s `preRestart` hook (defaults to sending termination requests to all children and calling `postStop`)
  4. 调用旧实例的`preRestart`挂钩（默认为将终止请求发送给所有子节点并调用`postStop`）
  5. wait for all children which were requested to terminate (using `context.stop()`) during `preRestart` to actually terminate;this—like all actor operations—is non-blocking, the termination notice from the last killed child will effect the progression to the next step
  6. 等待所有被要求终止（使用`context.stop()`）的孩子`preRestart`实际终止; 这就像所有演员的操作一样都是非阻塞的，最后一个被杀死的孩子的终止通知会影响到下一步的进展
  7. create new actor instance by invoking the originally provided factory again
  8. 再次调用原先提供的工厂创建新的actor实例
  9. invoke `postRestart` on the new instance (which by default also calls `preStart`)
  10. 调用`postRestart`新的实例（默认情况下也调用`preStart`）
  11. send restart request to all children which were not killed in step 3;restarted children will follow the same process recursively, from step 2
  12. 将重新启动请求发送给在步骤3中未被杀死的所有孩子; 从第2步开始，重新启动的孩子将按递归方式执行相同的过程
  13. resume the actor
  14. 恢复演员

## 生命周期监测手段？生命周期意味着？（What Lifecycle Monitoring Means）

@@@ note

Lifecycle Monitoring in Akka is usually referred to as `DeathWatch`

阿卡生命周期监测通常被称为 `DeathWatch`

@@@

In contrast to the special relationship between parent and child described above, each actor may monitor any other actor. Since actors emerge from creation fully alive and restarts are not visible outside of the affected supervisors, the only state change available for monitoring is the transition from alive to dead. Monitoring is thus used to tie one actor to another so that it may react to the other actor’s termination, in contrast to supervision which reacts to failure.

与上面描述的父母与孩子之间的特殊关系相比，每个演员可以监视任何其他演员。由于演员从创作中脱颖而出并且在受影响的主管之外看不到重新开始，唯一可以进行监控的状态变化是从活着转变为死亡。监控因此用于将一个参与者与另一个参与者联系起来，以便它可以对另一个参与者的终止做出反应，而监督则对失败做出反应。

Lifecycle monitoring is implemented using a `Terminated` message to be received by the monitoring actor, where the default behavior is to throw a special `DeathPactException` if not otherwise handled. In order to start listening for `Terminated` messages, invoke `ActorContext.watch(targetActorRef)`.  To stop listening, invoke `ActorContext.unwatch(targetActorRef)`.  One important property is that the message will be delivered irrespective of the order in which the monitoring request and target’s termination occur, i.e. you still get the message even if at the time of registration the target is already dead.

生命周期监控是`Terminated`通过监控参与者接收到的消息来实现的，其中缺省行为是在特殊情况下抛出一个特殊`DeathPactException`情况，否则不予处理。为了开始监听`Terminated`消息，调用`ActorContext.watch(targetActorRef)`。要停止监听，请调用`ActorContext.unwatch(targetActorRef)`。一个重要的特性是不管监视请求和目标终止发生的顺序如何，即使在注册时目标已经死亡，您仍然收到消息。

Monitoring is particularly useful if a supervisor cannot simply restart its children and has to terminate them, e.g. in case of errors during actor initialization. In that case it should monitor those children and re-create them or schedule itself to retry this at a later time.

如果监督人员不能重新启动其子代并且必须终止它们，例如在参与者初始化期间出现错误，监视尤其有用。在那种情况下，它应该监视那些孩子并重新创建它们或安排在晚些时候重试。

Another common use case is that an actor needs to fail in the absence of an external resource, which may also be one of its own children. If a third party terminates a child by way of the `system.stop(child)` method or sending a `PoisonPill`, the supervisor might well be affected.

另一个常见的用例是，在没有外部资源的情况下，演员需要失败，外部资源也可能是其自己的子项之一。如果第三方通过该`system.stop(child)`方法终止儿童或发送a `PoisonPill`，则主管可能会受到影响。

<a id="backoff-supervisor"></a>
### 延迟重新启动BackoffSupervisor模式

Provided as a built-in pattern the `akka.pattern.BackoffSupervisor` implements the so-called 
*exponential backoff supervision strategy*, starting a child actor again when it fails, each time with a growing time delay between restarts.

作为内置模式提供`akka.pattern.BackoffSupervisor`实施所谓的*指数退避监督策略*，当其失败时再次启动子动作者，每次重启之间的时间延迟增加。

This pattern is useful when the started actor fails <a id="^1" href="#1">[1]</a> because some external resource is not available,
and we need to give it some time to start-up again. One of the prime examples when this is useful is
when a @ref:[PersistentActor](../persistence.md) fails (by stopping) with a persistence failure - which indicates that the database may be down or overloaded, in such situations it makes most sense to give it a little bit of time
to recover before the peristent actor is started.

当启动的actor失败[[1\]](https://doc.akka.io/docs/akka/current/general/supervision.html#1)时，这种模式非常有用，因为某些外部资源不可用，我们需要给它一些时间来重新启动。有用的主要示例之一是[PersistentActor](https://doc.akka.io/docs/akka/current/persistence.html)在发生[持久性](https://doc.akka.io/docs/akka/current/persistence.html)失败时（通过停止）失败 - 这表明数据库可能已关闭或超载，因此在这种情况下最有意义的是给它一点时间在永久演员开始之前恢复。

> <a id="1" href="#^1">[1]</a> A failure can be indicated in two different ways; by an actor stopping or crashing.
>
> [[1\]](https://doc.akka.io/docs/akka/current/general/supervision.html#^1)失败可以用两种不同的方式表示; 由演员停止或撞击

The following Scala snippet shows how to create a backoff supervisor which will start the given echo actor after it has stopped because of a failure, in increasing intervals of 3, 6, 12, 24 and finally 30 seconds:

下面的Scala片段展示了如何创建一个backoff supervisor，它将在由于失败而停止之后，以3,6,12,24和30秒的间隔增加来启动给定的echo actor：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-stop }

The above is equivalent to this Java code:

以上相当于这个Java代码：

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-imports }

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-stop }

Using a `randomFactor` to add a little bit of additional variance to the backoff intervals is highly recommended, in order to avoid multiple actors re-start at the exact same point in time,for example because they were stopped due to a shared resource such as a database going down and re-starting after the same configured interval. By adding additional randomness to the re-start intervals the actors will start in slightly different points in time, thus avoiding large spikes of traffic hitting the recovering shared database or other resource that they all need to contact.

`randomFactor`强烈建议使用a 向补偿间隔添加一点额外的差异，以避免多个参与者在完全相同的时间点重新启动，例如因为它们由于共享资源（如数据库）而停止在相同的配置时间间隔后下降并重新启动。通过在重新启动间隔中增加额外的随机性，参与者将在稍微不同的时间点开始，从而避免流向复原共享数据库或他们都需要联系的其他资源的大量峰值流量。

The `akka.pattern.BackoffSupervisor` actor can also be configured to restart the actor after a delay when the actor crashes and the supervision strategy decides that it should restart.

该`akka.pattern.BackoffSupervisor`演员也可以配置延迟后重新启动的演员时，演员崩溃并监督战略决定，它应该重新启动。

The following Scala snippet shows how to create a backoff supervisor which will start the given echo actor after it has crashed because of some exception, in increasing intervals of 3, 6, 12, 24 and finally 30 seconds:

下面的Scala片段展示了如何创建一个backoff supervisor，它会在由于某些异常而崩溃之后，以3,6,12,24和30秒的间隔增加来启动给定的echo actor：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-fail }

The above is equivalent to this Java code:

以上相当于这个Java代码：

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-imports }

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-fail }

The `akka.pattern.BackoffOptions` can be used to customize the behavior of the back-off supervisor actor, below are some examples:

该`akka.pattern.BackoffOptions`可用于定制回退主管演员的行为，下面是一些例子：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-custom-stop }

The above code sets up a back-off supervisor that requires the child actor to send a `akka.pattern.BackoffSupervisor.Reset` message to its parent when a message is successfully processed, resetting the back-off. It also uses a default stopping strategy, any exception will cause the child to stop.

上面的代码设置了一个退避监督员，`akka.pattern.BackoffSupervisor.Reset`当成功处理消息时，要求子actor向其父节点发送消息，重置回退。它也使用默认的停止策略，任何异常都会导致孩子停下来。

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-custom-fail }

The above code sets up a back-off supervisor that restarts the child after back-off if MyException is thrown, any other exception will be escalated. The back-off is automatically reset if the child does not throw any errors within 10 seconds.

上面的代码设置了一个退避监督器，如果抛出MyException，退出后会重新启动孩子，任何其他异常都会升级。如果孩子在10秒内没有抛出任何错误，补偿会自动复位。

## 一对一 vs.多对一

There are two classes of supervision strategies which come with Akka:`OneForOneStrategy` and `AllForOneStrategy`. Both are configured with a mapping from exception type to supervision directive (see
[above](#supervision-directives)) and limits on how often a child is allowed to fail before terminating it. The difference between them is that the former applies the obtained directive only to the failed child, whereas the latter applies it to all siblings as well. Normally, you should use the `OneForOneStrategy`, which also is the default if none is specified explicitly.

阿卡有两类监督策略：`OneForOneStrategy`和`AllForOneStrategy`。两者都配置了从异常类型到监督指令的映射（见[上文](https://doc.akka.io/docs/akka/current/general/supervision.html#supervision-directives)），并限制了孩子在终止之前允许失败的频率。它们之间的区别在于，前者将获得的指令仅应用于失败的孩子，而后者将其应用于所有兄弟姐妹。通常情况下，你应该使用`OneForOneStrategy`，如果没有明确指定，这也是默认值。

The `AllForOneStrategy` is applicable in cases where the ensemble of children has such tight dependencies among them, that a failure of one child affects the function of the others, i.e. they are inextricably linked. Since a restart does not clear out the mailbox, it often is best to terminate the children
upon failure and re-create them explicitly from the supervisor (by watching the children’s lifecycle); otherwise you have to make sure that it is no problem for any of the actors to receive a message which was queued before the restart but processed afterwards.

在`AllForOneStrategy`适用的情况下，孩子们的合奏其中有如此严格的依存关系，一个孩子的失败会影响其他的功能，也就是说，它们是密不可分的。由于重新启动不会清除邮箱，因此通常最好是在失败时终止孩子，并通过监督员明确地重新创建孩子（通过观察孩子的生命周期）; 否则你必须确保任何演员接收到重新启动前排队的消息但事后处理都没有问题。

Normally stopping a child (i.e. not in response to a failure) will not automatically terminate the other children in an all-for-one strategy; this can easily be done by watching their lifecycle: if the `Terminated` message is not handled by the supervisor, it will throw a `DeathPactException ` which (depending on its supervisor) will restart it, and the default `preRestart` action will terminate all children. Of course this can be handled explicitly as well.

通常阻止一个孩子（即不是对失败的回应）不会自动终止其他孩子的一对一策略; 这可以通过观察他们的生命周期来完成：如果`Terminated`消息不由主管处理，它将会抛出一个`DeathPactException`（取决于其主管）重启它，并且默认`preRestart`动作将终止所有的孩子。当然这也可以明确处理。

Please note that creating one-off actors from an all-for-one supervisor entails that failures escalated by the temporary actor will affect all the permanent ones. If this is not desired, install an intermediate supervisor; this can very easily be done by declaring a router of size 1 for the worker, see @ref:[Routing](../routing.md).

请注意，创建一位全能主管的一次性演员需要临时演员升级失败会影响所有永久演员。如果不需要，请安装中间监督员; 这可以通过为工作人员声明大小为1的路由器完成，请参阅[路由](https://doc.akka.io/docs/akka/current/routing.html)。