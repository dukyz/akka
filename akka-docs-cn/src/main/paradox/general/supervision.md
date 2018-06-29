# 监督和监视

This chapter outlines the concept behind supervision, the primitives offered and their semantics. For details on how that translates into real code, please refer to the corresponding chapters for Scala and Java APIs.

本章概述了监督背后的概念，其提供的原语及其语义。有关如何转换为真实代码的细节，请参阅Scala和Java API的相应章节。

<a id="supervision-directives"></a>
## 监督

As described in @ref:[Actor Systems](actor-systems.md) supervision describes a dependency relationship between actors: the supervisor delegates tasks to subordinates and therefore must respond to their failures.  When a subordinate detects a failure (i.e. throws an exception), it suspends itself and all its subordinates and
sends a message to its supervisor, signaling failure.  Depending on the nature of the work to be supervised and the nature of the failure, the supervisor has a choice of the following four options:

正如[Actor系统](https://doc.akka.io/docs/akka/current/general/actor-systems.html)中所描述的，监督描述了Actor之间的依赖关系：监督者将任务委派给下属，因此必须对他们的失败做出响应。当下属检测到故障（即抛出异常）时，它会挂起自己及其所有下属，并向其监督员发送消息，表明发生故障。根据要监督的工作性质和失败的性质，监督者可以有以下四种选择：

  1. Resume the subordinate, keeping its accumulated internal state
  2. 恢复下属，保持其积累的内部状态
  3. Restart the subordinate, clearing out its accumulated internal state
  4. 重启下属，清除其累积的内部状态
  5. Stop the subordinate permanently
  6. 永久停止下属
  7. Escalate the failure, thereby failing itself
  8. 向上级抛出失败，从而让自己失败

It is important to always view an actor as part of a supervision hierarchy,which explains the existence of the fourth choice (as a supervisor also is subordinate to another supervisor higher up) and has implications on the first three: resuming an actor resumes all its subordinates, restarting an actor entails restarting all its subordinates (but see below for more details),similarly terminating an actor will also terminate all its subordinates. It should be noted that the default behavior of the `preRestart` hook of the `Actor` class is to terminate all its children before restarting, but this hook can be overridden; the recursive restart applies to all children left after this hook has been executed.

始终把一个Actor视为监督等级的一部分这很重要，这解释了第四个选择的存在（因为监督者也从属于另一个上级监督者）并且对前三个有影响：恢复一个Actor会恢复其所有下属Actor，重启一个Actor需要重启其所有下属（更多细节见下文），同样地终止Actor也将终止其所有下属。应该注意的是，`Actor`类中的`preRestart`方法的默认行为是在重启之前终止其所有子级，但是可以覆盖此方法; 递归重启适用于执行该方法后留下的所有剩余子项。

Each supervisor is configured with a function translating all possible failure causes (i.e. exceptions) into one of the four choices given above; notably, this function does not take the failed actor’s identity as an input. It is quite easy to come up with examples of structures where this might not seem flexible enough, e.g. wishing for different strategies to be applied to different subordinates. At this point it is vital to understand that supervision is about forming a recursive fault handling structure. If you try to do too much at one level, it will become hard to reason about, hence the recommended way in this case is to add a level of supervision.

每个监督者都配置了一个功能，将所有可能的故障原因（如异常）转换为上述四种选择之一; 特别是，这个函数不会将失败的Actor身份作为输入。提出不够灵活的结构示例是很容易的，例如，希望将不同的策略应用于不同的下属。在这一点上，理解监督是关于形成递归的故障处理结构是非常重要的。如果你想在一个层面做太多的事情，事情就变得就很难推理，因此在这种情况下推荐的方法是增加监督层级。

Akka implements a specific form called “parental supervision”. Actors can only be created by other actors—where the top-level actor is provided by the library—and each created actor is supervised by its parent. This restriction makes the formation of actor supervision hierarchies implicit and encourages sound design decisions. It should be noted that this also guarantees that actors cannot be orphaned or attached to supervisors from the outside, which might otherwise catch them unawares. In addition, this yields a natural and clean shutdown procedure for (sub-trees of) actor applications.

Akka实现了一种称为“家长监督”的具体形式。Actor只能由其他Actor创建 - 顶级Actor由Akka类库提供 - 每个创建的Actor都由其父级监督。这种限制使得Actor监督层次隐式地形成，并鼓励合理的设计决策。应该指出的是，这也保证Actor不会孤立存在或附属于外部的监督者。另外，这为Actor应用程序提供了一个自然且干净的关闭过程。

@@@ warning

Supervision related parent-child communication happens by special system messages that have their own mailboxes separate from user messages. This implies that supervision related events are not  deterministically ordered relative to ordinary messages. In general, the user cannot influence the order of normal messages and failure notifications. For details and example see the @ref:[Discussion: Message Ordering](message-delivery-reliability.md#message-ordering) section.

监督中的父子级沟通利用特殊的系统消息，这些消息拥有与用户消息不同的邮箱。这意味着与监督有关的事件相对于普通消息不是确定顺序的。通常，用户不能影响正常消息和失败通知的顺序。有关详细信息和示例，请参阅[讨论：消息排序](https://doc.akka.io/docs/akka/current/general/message-delivery-reliability.html#message-ordering)部分。

@@@

<a id="toplevel-supervisors"></a>
## 顶级监督者

![guardians.png](guardians.png)

An actor system will during its creation start at least three actors, shown in the image above. For more information about the consequences for actor paths see @ref:[Top-Level Scopes for Actor Paths](addressing.md#toplevel-paths).

Actor系统在其创建过程中将启动至少三个Actor，如上图所示。有关Actor路径后果的更多信息，请参阅[Actor路径的顶层范围](https://doc.akka.io/docs/akka/current/general/addressing.html#toplevel-paths)。

<a id="user-guardian"></a>
### `/user`: 守卫Actor

The actor which is probably most interacted with is the parent of all user-created actors, the guardian named `"/user"`. Actors created using `system.actorOf()` are children of this actor. This means that when this guardian terminates, all normal actors in the system will be shutdown, too. It also means that this guardian’s supervisor strategy determines how the top-level normal actors are supervised. Since Akka 2.1 it is possible to configure this using the setting `akka.actor.guardian-supervisor-strategy`,which takes the fully-qualified class-name of a `SupervisorStrategyConfigurator`. When the guardian escalates a failure, the root guardian’s response will be to terminate the guardian, which in effect will shut down the whole actor system.

最经常与之交互的Actor可能是所有用户创建的Actor的父级，名为`"/user"`的监督者。利用`system.actorOf()`方法创建的Actor是这个Actor的子级。这意味着当这个监督者终止时，系统中的所有正常Actor也将被关闭。这也意味着这位监督者的监督策略决定了顶级的一般Actor如何受到监督。从Akka 2.1开始，可以通过实现`SupervisorStrategyConfigurator`特质的实例来设置`akka.actor.guardian-supervisor-strategy`项，这需要以完全限定类名来配置这个监督策略。当这个监督者将失败向上传递时，根监督者的回应将是终止这个监督者，这实际上也将关闭整个Actor系统。

### `/system`: 系统守卫

This special guardian has been introduced in order to achieve an orderly shut-down sequence where logging remains active while all normal actors terminate, even though logging itself is implemented using actors. This is realized by having the system guardian watch the user guardian and initiate its own
shut-down upon reception of the `Terminated` message. The top-level system actors are supervised using a strategy which will restart indefinitely upon all types of `Exception` except for `ActorInitializationException` and `ActorKilledException`, which will terminate the child in question.  All other throwables are escalated,which will shut down the whole actor system.

这个特殊的监督者在系统实现有序化关闭的时候已经介绍过了，当所有的一般Actor都被停止、日志还有效时，即使日志本身也是通过Actor实现的。/system监督者监视/user监督者,并在接收到`Terminated`消息时启动自我关闭。顶级系统Actor们接受监督策略，将根据除了`ActorInitializationException`和`ActorKilledException`之外所有类型的`Exception`而不断重启，上述两个异常将终止有问题的子级。所有其他的异常会向上抛出，这将关闭整个Actor系统。(**这段操他妈的，写了些啥，这××从句用的，谁他妈写的！！妈了个隔壁的**)(**请自行理解英文吧**)

### `/`: 根守护

The root guardian is the grand-parent of all so-called “top-level” actors and supervises all the special actors mentioned in @ref:[Top-Level Scopes for Actor Paths](addressing.md#toplevel-paths) using the `SupervisorStrategy.stoppingStrategy`, whose purpose is to terminate the child upon any type of `Exception`. All other throwables will be
escalated … but to whom? Since every real actor has a supervisor, the supervisor of the root guardian cannot be a real actor. And because this means that it is “outside of the bubble”, it is called the “bubble-walker”. This is a synthetic `ActorRef` which in effect stops its child upon the first sign of trouble and sets the actor system’s `isTerminated` status to `true` as soon as the root guardian is fully terminated (all children recursively stopped).

根监督者是所有所谓“顶级”Actor的祖父，其利用`SupervisorStrategy.stoppingStrategy`策略监督[Actor路径的顶级范围](https://doc.akka.io/docs/akka/current/general/addressing.html#toplevel-paths)中提及的所有特殊Actor，该策略的目的是终止任何类型的子级`Exception`。所有其他的异常将向上抛出...但是抛给谁？由于每个真正的Actor都有一名监督者，而根监督者的上级监督者不可能是真正的Actor。因为这意味着它是“泡沫之外”，所以它被称为“冒泡者”。这是一种综合性的`ActorRef`，它在遇到问题时及时停止其子级，并在根监督者完全停止（所有子级被递归停止）后立即将Actor系统的`isTerminated`状态设置为`true`。

<a id="supervision-restart"></a>
## 重启

When presented with an actor which failed while processing a certain message,causes for the failure fall into three categories:

当一个Actor因处理某条消息而导致失败，导致失败的原因分为三类：

 * Systematic (i.e. programming) error for the specific message received
 * 收到特定的系统（编程）错误消息
 * (Transient) failure of some external resource used during processing the message
 * 在处理消息期间使用的某些外部资源的（瞬态）失败
 * Corrupt internal state of the actor
 * Actor内部状态损坏

Unless the failure is specifically recognizable, the third cause cannot be ruled out, which leads to the conclusion that the internal state needs to be cleared out. If the supervisor decides that its other children or itself is not affected by the corruption—e.g. because of conscious application of the error kernel pattern—it is therefore best to restart the child. This is carried out by creating a new instance of the underlying `Actor` class and replacing the failed instance with the fresh one inside the child’s `ActorRef`;the ability to do this is one of the reasons for encapsulating actors within special references. The new actor then resumes processing its mailbox, meaning that the restart is not visible outside of the actor itself with the notable exception that the message during which the failure occurred is not re-processed.

除非故障具体可识别，否则不能排除第三个原因，也就是内部状态需要清除的结论。如果监督者认为其他子级或自己不受损坏的影响 - 例如由于有意识地应用错误内核模式 - 最好重新启动子级。通过创建一个新的底层`Actor`实例并用子级`ActorRef`中的新实例替换失败实例; 这样做也是将特殊引用封装在Actor中的原因之一。然后新Actor恢复并处理其邮箱中的邮件，这意味着重启在Actor外部是不可见的，除了那些发生异常的信息(**翻译不懂，自行体会原文吧，无力吐槽呃**)。

The precise sequence of events during a restart is the following:

重启过程中精确的事件顺序如下：

  1. suspend the actor (which means that it will not process normal messages until resumed), and recursively suspend all children
  2. 挂起Actor（这意味着它直到恢复之前不会处理正常的消息），并递归挂起所有的子级
  3. call the old instance’s `preRestart` hook (defaults to sending termination requests to all children and calling `postStop`)
  4. 调用旧实例的`preRestart`（默认行为是向子节点发送终止请求并调用`postStop`）
  5. wait for all children which were requested to terminate (using `context.stop()`) during `preRestart` to actually terminate;this—like all actor operations—is non-blocking, the termination notice from the last killed child will effect the progression to the next step
  6. 等待所有在`preRestart`过程中被要求终止（使用`context.stop()`）的子级真正的终止; 就像所有Actor的操作一样这都是非阻塞的，最后一个被终止的子级的终止通知将会使得重启流程进行至下一步。
  7. create new actor instance by invoking the originally provided factory again
  8. 再次调用原先提供的工厂方法来创建新的Actor实例
  9. invoke `postRestart` on the new instance (which by default also calls `preStart`)
  10. 新的实例调用`postRestart`（默认也会调用`preStart`）
  11. send restart request to all children which were not killed in step 3;restarted children will follow the same process recursively, from step 2
  12. 将重启请求发送给在步骤3中未被杀死的所有子级; 从第2步开始，重启的子级将按递归方式执行相同的过程
  13. resume the actor
  14. 恢复Actor

## 生命周期监测

@@@ note

Lifecycle Monitoring in Akka is usually referred to as `DeathWatch`

Akka生命周期监测通常被称为 `DeathWatch`

@@@

In contrast to the special relationship between parent and child described above, each actor may monitor any other actor. Since actors emerge from creation fully alive and restarts are not visible outside of the affected supervisors, the only state change available for monitoring is the transition from alive to dead. Monitoring is thus used to tie one actor to another so that it may react to the other actor’s termination, in contrast to supervision which reacts to failure.

与之前描述的父子级之间的特殊关系相比，每个Actor可以监视任何其他Actor。由于Actor从创建开始并且重启对外界不可见，唯一可进行监控的状态变化是从生到死。监视因此用于将两个Actor联系起来，以便一个可以对另一个Actor的终止做出反应，而监督则可以对失败做出反应。

Lifecycle monitoring is implemented using a `Terminated` message to be received by the monitoring actor, where the default behavior is to throw a special `DeathPactException` if not otherwise handled. In order to start listening for `Terminated` messages, invoke `ActorContext.watch(targetActorRef)`.  To stop listening, invoke `ActorContext.unwatch(targetActorRef)`.  One important property is that the message will be delivered irrespective of the order in which the monitoring request and target’s termination occur, i.e. you still get the message even if at the time of registration the target is already dead.

生命周期监控是通过监视Actor接收`Terminated`消息来实现的，默认是如果不给于另外的处理，那么会抛出一个特殊的`DeathPactException`异常。监听`Terminated`消息，调用`ActorContext.watch(targetActorRef)`。停止监听，调用`ActorContext.unwatch(targetActorRef)`。一个重要的特性是不管监视请求和目标终止的顺序如何，消息仍将传递，比如：在注册时目标已经死亡，您仍将收到消息。

Monitoring is particularly useful if a supervisor cannot simply restart its children and has to terminate them, e.g. in case of errors during actor initialization. In that case it should monitor those children and re-create them or schedule itself to retry this at a later time.

当监督者不能重启其子级并必须终止它们时，监视功能尤其有用。例如在Actor初始化期间出现错误。在那种情况下，应该监视那些子级并重新创建它们或安排稍后重试。

Another common use case is that an actor needs to fail in the absence of an external resource, which may also be one of its own children. If a third party terminates a child by way of the `system.stop(child)` method or sending a `PoisonPill`, the supervisor might well be affected.

另一个常见的用例是，在缺少外部资源的情况下，Actor需要失败，外部资源也可能是其自己的子级之一。如果第三方通过`system.stop(child)`方法或以发送 `PoisonPill`的形式终止子级，则监督者也可能会受到影响。

<a id="backoff-supervisor"></a>
### 延迟重启的BackoffSupervisor模式

Provided as a built-in pattern the `akka.pattern.BackoffSupervisor` implements the so-called 
*exponential backoff supervision strategy*, starting a child actor again when it fails, each time with a growing time delay between restarts.

`akka.pattern.BackoffSupervisor`作为内置模式，提供了所谓的*指数级重试监督策略*，当子级失败重启时，每次重启之间的时间间隔均有所增加。

This pattern is useful when the started actor fails <a id="^1" href="#1">[1]</a> because some external resource is not available,
and we need to give it some time to start-up again. One of the prime examples when this is useful is
when a @ref:[PersistentActor](../persistence.md) fails (by stopping) with a persistence failure - which indicates that the database may be down or overloaded, in such situations it makes most sense to give it a little bit of time
to recover before the peristent actor is started.

因为某些外部资源不可用而使得Actor启动失败[[1\]](https://doc.akka.io/docs/akka/current/general/supervision.html#1)，我们需要给外部资源点时间让它可用时，这种模式非常有用。一个主要例子是当[PersistentActor](https://doc.akka.io/docs/akka/current/persistence.html)在发生持久化时失败（以停止的方式） - 这意味着数据库可能关闭或超载，在这种情况下最好是给数据库一点时间让它能在持久化Actor启动前恢复。

> <a id="1" href="#^1">[1]</a> A failure can be indicated in two different ways; by an actor stopping or crashing.
>
> [[1\]](https://doc.akka.io/docs/akka/current/general/supervision.html#^1)两种方式可以导致失败; Actor的停止或崩溃

The following Scala snippet shows how to create a backoff supervisor which will start the given echo actor after it has stopped because of a failure, in increasing intervals of 3, 6, 12, 24 and finally 30 seconds:

下面的Scala片段展示了如何创建一个重试supervisor，echo Actor由于失败而停止，并以3,6,12,24和最终30秒的间隔来启动：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-stop }

The above is equivalent to this Java code:

以上相当于这段Java代码：

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-imports }

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-stop }

Using a `randomFactor` to add a little bit of additional variance to the backoff intervals is highly recommended, in order to avoid multiple actors re-start at the exact same point in time,for example because they were stopped due to a shared resource such as a database going down and re-starting after the same configured interval. By adding additional randomness to the re-start intervals the actors will start in slightly different points in time, thus avoiding large spikes of traffic hitting the recovering shared database or other resource that they all need to contact.

强烈建议使用`randomFactor`对重试间隔添加一点差异，以免多个Actor在完全相同的时间点重新启动，例如因为共享资源（如数据库）的停止而导致在相同的配置时间间隔后重新启动。通过在重启间隔中增加额外的随机性，Actor将在稍微不同的时间点重启，从而避免涌向待恢复资源的峰值流量。

The `akka.pattern.BackoffSupervisor` actor can also be configured to restart the actor after a delay when the actor crashes and the supervision strategy decides that it should restart.

该`akka.pattern.BackoffSupervisor`也可以用于配置Actor崩溃后的依据监督策略的延迟重启，。

The following Scala snippet shows how to create a backoff supervisor which will start the given echo actor after it has crashed because of some exception, in increasing intervals of 3, 6, 12, 24 and finally 30 seconds:

下面的Scala片段展示了如何创建一个重试supervisor，echo Actor由于某些异常而崩溃，之后以3,6,12,24和最终30秒的间隔来启动：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-fail }

The above is equivalent to this Java code:

以上相当于这段Java代码：

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-imports }

@@snip [BackoffSupervisorDocTest.java]($code$/java/jdocs/pattern/BackoffSupervisorDocTest.java) { #backoff-fail }

The `akka.pattern.BackoffOptions` can be used to customize the behavior of the back-off supervisor actor, below are some examples:

`akka.pattern.BackoffOptions`可用于定制负责重试的监督者Actor的行为，下面是一些例子：

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-custom-stop }

The above code sets up a back-off supervisor that requires the child actor to send a `akka.pattern.BackoffSupervisor.Reset` message to its parent when a message is successfully processed, resetting the back-off. It also uses a default stopping strategy, any exception will cause the child to stop.

上面的代码设置了一个负责重试的监督者，它要求子级Actor当成功处理消息时，向其父节点发送`akka.pattern.BackoffSupervisor.Reset`消息，复位重试的属性。它也使用默认的停止策略，任何异常都会导致子级停止。

@@snip [BackoffSupervisorDocSpec.scala]($code$/scala/docs/pattern/BackoffSupervisorDocSpec.scala) { #backoff-custom-fail }

The above code sets up a back-off supervisor that restarts the child after back-off if MyException is thrown, any other exception will be escalated. The back-off is automatically reset if the child does not throw any errors within 10 seconds.

上面的代码设置了一个负责重试的监督者，如果重试过程中子级抛出MyException，则会重启子级，而其他异常则会向上抛出。如果子级在10秒内没有抛出任何错误，重置会自动复位。

## 一对一 vs.多对一

There are two classes of supervision strategies which come with Akka:`OneForOneStrategy` and `AllForOneStrategy`. Both are configured with a mapping from exception type to supervision directive (see
[above](#supervision-directives)) and limits on how often a child is allowed to fail before terminating it. The difference between them is that the former applies the obtained directive only to the failed child, whereas the latter applies it to all siblings as well. Normally, you should use the `OneForOneStrategy`, which also is the default if none is specified explicitly.

Akka有两类监督策略：`OneForOneStrategy`和`AllForOneStrategy`。两者都配置了异常类型与监督指令的映射关系（见[上文](https://doc.akka.io/docs/akka/current/general/supervision.html#supervision-directives)），并限制了子级在终止之前允许失败的频率。它们之间的区别在于，前者将指令仅应用于失败的子级，而后者将其应用于子级及其所有兄弟姐妹。通常情况下，你应该使用`OneForOneStrategy`，如果没有明确指定，这也是默认值。

The `AllForOneStrategy` is applicable in cases where the ensemble of children has such tight dependencies among them, that a failure of one child affects the function of the others, i.e. they are inextricably linked. Since a restart does not clear out the mailbox, it often is best to terminate the children
upon failure and re-create them explicitly from the supervisor (by watching the children’s lifecycle); otherwise you have to make sure that it is no problem for any of the actors to receive a message which was queued before the restart but processed afterwards.

`AllForOneStrategy`适用于子级之间存在紧密的依赖关系，一个子级的失败会影响其他人的功能时，也就是说，它们是密不可分的。由于重启不会清除邮箱，因此最好是在失败时终止该子级，并通过监督者明确地重新创建子级（通过观察子级的生命周期）; 否则你必须确保所有Actor都能够没有问题地继续处理重启前接受的信息。

Normally stopping a child (i.e. not in response to a failure) will not automatically terminate the other children in an all-for-one strategy; this can easily be done by watching their lifecycle: if the `Terminated` message is not handled by the supervisor, it will throw a `DeathPactException ` which (depending on its supervisor) will restart it, and the default `preRestart` action will terminate all children. Of course this can be handled explicitly as well.

通常在多对一策略中停止一个子级（不是对失败的回应）不会自动停止其他的子级; 这可以通过观察他们的生命周期来完成：如果`Terminated`消息不被监督者处理，它将会抛出一个`DeathPactException`（取决于其监督者）重启它，并且默认在`preRestart`动作中停止所有子级。当然这也可以被明确处理。

Please note that creating one-off actors from an all-for-one supervisor entails that failures escalated by the temporary actor will affect all the permanent ones. If this is not desired, install an intermediate supervisor; this can very easily be done by declaring a router of size 1 for the worker, see @ref:[Routing](../routing.md).

请注意，根据多对一策略创建一次性Actor会引起临时Actor的失败向上抛出，导致常规Actor受影响。如果这不是想要的结果，请利用中间监督者; 这可以通过为工作人员声明大小为1的路由来实现，请参阅[路由](https://doc.akka.io/docs/akka/current/routing.html)。