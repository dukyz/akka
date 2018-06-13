# Actor是什么?

The previous section about @ref:[Actor Systems](actor-systems.md) explained how actors form hierarchies and are the smallest unit when building an application. This section looks at one such actor in isolation, explaining the concepts you encounter while implementing it. For a more in depth reference with all the
details please refer to @ref:[Actors](../actors.md).

上一节关于[Actor系统的](https://doc.akka.io/docs/akka/current/general/actor-systems.html)部分解释了参与者在构建应用程序时如何形成层次结构并且是最小单位。本节单独看一个这样的角色，解释你在实现时遇到的概念。有关所有细节的更深入参考，请参阅[演员](https://doc.akka.io/docs/akka/current/actors.html)。

An actor is a container for [State](#state), [Behavior](#behavior), a [Mailbox](#mailbox), [Child Actors](#child-actors) and a [Supervisor Strategy](#supervisor-strategy). All of this is encapsulated behind an [Actor Reference](#actor-reference). One noteworthy aspect is that actors have an explicit lifecycle, they are not automatically destroyed when no longer referenced; after having created one, it is your responsibility to make sure that it will eventually be terminated as well—which also gives you control over how resources are released [When an Actor Terminates](#when-an-actor-terminates).

演员是[国家](https://doc.akka.io/docs/akka/current/general/actors.html#state)，[行为](https://doc.akka.io/docs/akka/current/general/actors.html#behavior)，[邮箱](https://doc.akka.io/docs/akka/current/general/actors.html#mailbox)，[儿童演员](https://doc.akka.io/docs/akka/current/general/actors.html#child-actors)和[监督者战略](https://doc.akka.io/docs/akka/current/general/actors.html#supervisor-strategy)的容器。所有这些都封装在[Actor参考之后](https://doc.akka.io/docs/akka/current/general/actors.html#actor-reference)。一个值得注意的方面是演员有一个明确的生命周期，当不再被引用时它们不会被自动销毁; 创建一个后，它是你的责任，以确保它最终将被终止 - 这也让你控制如何资源被释放[当演员终止](https://doc.akka.io/docs/akka/current/general/actors.html#when-an-actor-terminates)。

## Actor引用

As detailed below, an actor object needs to be shielded from the outside in order to benefit from the actor model. Therefore, actors are represented to the outside using actor references, which are objects that can be passed around freely and without restriction. This split into inner and outer object enables transparency for all the desired operations: restarting an actor without needing to update references elsewhere, placing the actual actor object on remote hosts, sending messages to actors in completely different applications.
But the most important aspect is that it is not possible to look inside an actor and get hold of its state from the outside, unless the actor unwisely publishes this information itself.

如下所述，演员对象需要从外部屏蔽，以便从演员模型中受益。因此，演员通过演员引用被表示到外部，演员引用是可以自由而不受限制地传递的对象。这种拆分为内部对象和外部对象的操作可以实现所有操作的透明度：重新启动一个actor，而不需要在别处更新引用，将实际的actor对象放在远程主机上，并将消息发送给完全不同的应用程序中的actor。但是最重要的方面是，除非演员本身不明智地发布这些信息，否则不可能从外部看待演员并从外部获得其状态。

## 状态

Actor objects will typically contain some variables which reflect possible states the actor may be in. This can be an explicit state machine (e.g. using the @ref:[FSM](../fsm.md) module), or it could be a counter, set of listeners,
pending requests, etc. These data are what make an actor valuable, and they must be protected from corruption by other actors. The good news is that Akka actors conceptually each have their own light-weight thread, which is completely shielded from the rest of the system. This means that instead of
having to synchronize access using locks you can just write your actor code without worrying about concurrency at all.

Actor对象通常会包含一些变量，这些变量反映了actor可能处于的可能状态。这可以是显式状态机（例如使用[FSM](https://doc.akka.io/docs/akka/current/fsm.html)模块），也可以是计数器，侦听器集合，待处理请求等。这些数据是什么使演员有价值，他们必须得到保护，免受其他演员的腐败。好消息是，阿卡演员在概念上都有自己的轻量级线程，它完全屏蔽了系统的其他部分。这意味着，不必使用锁来同步访问，你可以编写你的代码而不用担心并发性。

Behind the scenes Akka will run sets of actors on sets of real threads, where typically many actors share one thread, and subsequent invocations of one actor may end up being processed on different threads. Akka ensures that this implementation detail does not affect the single-threadedness of handling the
actor’s state.

在幕后，Akka将在真正的线程集上运行一组演员，通常很多演员共享一个线程，并且一个演员的后续调用可能最终在不同线程上处理。Akka确保此实现细节不会影响处理actor的状态的单线程性。

Because the internal state is vital to an actor’s operations, having inconsistent state is fatal. Thus, when the actor fails and is restarted by its supervisor, the state will be created from scratch, like upon first creating
the actor. This is to enable the ability of self-healing of the system.

由于内部状态对于演员的操作至关重要，因此状态不一致是致命的。因此，当演员失败并由其主管重新启动时，状态将从头开始创建，就像首次创建演员时一样。这是为了使系统能够自我修复。

Optionally, an actor's state can be automatically recovered to the state before a restart by persisting received messages and replaying them after restart (see @ref:[Persistence](../persistence.md)).

或者，通过持续接收消息并在重新启动后重播它们（参见[持久性](https://doc.akka.io/docs/akka/current/persistence.html)），可以将参与者的状态自动恢复到重新启动之前的状态。

## 行为

Every time a message is processed, it is matched against the current behavior of the actor. Behavior means a function which defines the actions to be taken in reaction to the message at that point in time, say forward a request if the client is authorized, deny it otherwise. This behavior may change over time, e.g. because different clients obtain authorization over time, or because the actor may go into an “out-of-service” mode and later come back. These changes are achieved by either encoding them in state variables which are read from the behavior logic, or the function itself may be swapped out at runtime, see the
`become` and `unbecome` operations. However, the initial behavior defined during construction of the actor object is special in the sense that a restart of the actor will reset its behavior to this initial one.

每次处理消息时，都会与演员的当前行为进行匹配。行为意味着一个函数，它定义了当时对消息作出的反应所采取的行动，例如，如果客户端被授权请求转发请求，否则拒绝。这种行为可能会随着时间而改变，例如，因为不同的客户端随着时间的推移获得授权，或者因为参与者可能进入“停止服务”模式并且随后回来。这些更改是通过将它们编码到从行为逻辑读取的状态变量中实现的，或者可以在运行时将函数本身换出，请参阅`become`和`unbecome`操作。然而，在构造actor对象时定义的初始行为是特殊的，因为重新启动actor会将其行为重置为初始行为。

## 信箱

An actor’s purpose is the processing of messages, and these messages were sent to the actor from other actors (or from outside the actor system). The piece which connects sender and receiver is the actor’s mailbox: each actor has exactly one mailbox to which all senders enqueue their messages. Enqueuing
happens in the time-order of send operations, which means that messages sent from different actors may not have a defined order at runtime due to the apparent randomness of distributing actors across threads. Sending multiple messages to the same target from the same actor, on the other hand, will enqueue them in the same order.

演员的目的是处理消息，并将这些消息从其他演员（或演员系统外部）发送给演员。连接发送者和接收者的部分是演员的邮箱：每个演员只有一个邮箱，所有发件人都将邮件排入邮箱。排队按照发送操作的时间顺序发生，这意味着由于跨线程分布角色的明显随机性，从不同角色发送的消息可能在运行时没有定义的顺序。另一方面，从同一个演员向同一个目标发送多条消息将按照相同的顺序排列它们。

There are different mailbox implementations to choose from, the default being a FIFO: the order of the messages processed by the actor matches the order in which they were enqueued. This is usually a good default, but applications may need to prioritize some messages over others. In this case, a priority mailbox
will enqueue not always at the end but at a position as given by the message priority, which might even be at the front. While using such a queue, the order of messages processed will naturally be defined by the queue’s algorithm and in general not be FIFO.

有不同的邮箱实现可供选择，默认为FIFO：邮件处理的邮件顺序与邮件队列顺序相匹配。这通常是一个很好的默认设置，但应用程序可能需要优先考虑其他消息的优先级。在这种情况下，优先邮箱将不会始终排在最后，而是排在邮件优先级给出的位置，这可能甚至会在前面。在使用这样的队列时，处理的消息的顺序自然将由队列的算法定义，并且通常不是FIFO。

An important feature in which Akka differs from some other actor model implementations is that the current behavior must always handle the next dequeued message, there is no scanning the mailbox for the next matching one.Failure to handle a message will typically be treated as a failure, unless this behavior is overridden.

Akka与其他actor模型实现不同的一个重要特征是，当前行为必须处理下一个出队消息，并且不会扫描下一个匹配的邮箱。如果未能处理消息，则通常将其视为失败，除非此行为被覆盖。

## 子级Actors

Each actor is potentially a supervisor: if it creates children for delegating sub-tasks, it will automatically supervise them. The list of children is maintained within the actor’s context and the actor has access to it.
Modifications to the list are done by creating (`context.actorOf(...)`) or stopping (`context.stop(child)`) children and these actions are reflected immediately. The actual creation and termination actions happen behind the scenes in an asynchronous way, so they do not “block” their supervisor.

每个角色都可能是一名主管：如果它创建子任务来委派子任务，它将自动监督它们。孩子的名单保存在演员的上下文中，演员可以访问它。对列表的修改通过创建（`context.actorOf(...)`）或停止（`context.stop(child)`）子来完成，并且这些操作立即反映出来。实际的创建和终止操作以异步的方式发生在幕后，所以他们不会“阻止”他们的主管。

## 监管Strategy

The final piece of an actor is its strategy for handling faults of its children. Fault handling is then done transparently by Akka, applying one of the strategies described in @ref:[Supervision and Monitoring](supervision.md) for each incoming failure. As this strategy is fundamental to how an actor system is structured, it cannot be changed once an actor has been created.

演员的最后一部分是处理其孩子的错误的策略。然后，Akka透明地处理故障处理，应用[监督和监测中](https://doc.akka.io/docs/akka/current/general/supervision.html)描述的每种传入故障中的一种策略。由于该策略对于演员系统的结构是如何基础的，所以一旦创建演员就不能改变它。

Considering that there is only one such strategy for each actor, this means that if different strategies apply to the various children of an actor, the children should be grouped beneath intermediate supervisors with matching strategies, preferring once more the structuring of actor systems according to the splitting of tasks into sub-tasks.

考虑到每个演员只有一个这样的策略，这意味着如果不同的策略适用于演员的不同的孩子，那么应该将孩子们分组在中间主管之下，采用匹配的策略，再次优先考虑演员系统的结构化将任务分解为子任务。

## 当一个Actor终止时

Once an actor terminates, i.e. fails in a way which is not handled by a restart, stops itself or is stopped by its supervisor, it will free up its resources, draining all remaining messages from its mailbox into the system’s
“dead letter mailbox” which will forward them to the EventStream as DeadLetters. The mailbox is then replaced within the actor reference with a system mailbox,redirecting all new messages to the EventStream as DeadLetters. This is done on a best effort basis, though, so do not rely on it in order to
construct “guaranteed delivery”.

一旦演员终止，即失败的方式不是通过重启处理，停止自己或被其主管阻止，它将释放其资源，将其邮箱中的所有剩余邮件排入系统的“死信邮箱”中会将它们作为DeadLetters转发给EventStream。邮箱然后在主角色引用中替换为系统邮箱，将所有新邮件以DeadLetters方式重定向到EventStream。尽管这是在尽力而为的基础上完成的，但不要依赖它来构建“保证交付”。

The reason for not just silently dumping the messages was inspired by our tests: we register the TestEventListener on the event bus to which the dead letters are forwarded, and that will log a warning for every dead letter received—this has been very helpful for deciphering test failures more quickly. It is conceivable that this feature may also be of use for other purposes.

我们的测试启发了不仅仅是悄悄地转储消息的原因：我们在事件总线上注册了TestEventListener，死亡信件被转发到该事件总线上，并且会为收到的每一封死信记录警告 - 这对解密非常有用更快地测试故障。可以想象的是，该特征也可以用于其他目的。