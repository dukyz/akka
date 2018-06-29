# Actor是什么

The previous section about @ref:[Actor Systems](actor-systems.md) explained how actors form hierarchies and are the smallest unit when building an application. This section looks at one such actor in isolation, explaining the concepts you encounter while implementing it. For a more in depth reference with all the details please refer to @ref:[Actors](../actors.md).

上一节关于[Actor系统](https://doc.akka.io/docs/akka/current/general/actor-systems.html)的部分解释了Actor如何构建层次结构以及它是我们构建应用程序时的最小单位。本节单独来说Actor，解释你在实现具体Actor时遇到的相关概念。有关更深入的细节，请参阅[Actor](https://doc.akka.io/docs/akka/current/actors.html)。

An actor is a container for [State](#state), [Behavior](#behavior), a [Mailbox](#mailbox), [Child Actors](#child-actors) and a [Supervisor Strategy](#supervisor-strategy). All of this is encapsulated behind an [Actor Reference](#actor-reference). One noteworthy aspect is that actors have an explicit lifecycle, they are not automatically destroyed when no longer referenced; after having created one, it is your responsibility to make sure that it will eventually be terminated as well—which also gives you control over how resources are released [When an Actor Terminates](#when-an-actor-terminates).

Actor是[状态](https://doc.akka.io/docs/akka/current/general/actors.html#state)，[行为](https://doc.akka.io/docs/akka/current/general/actors.html#behavior)，[信箱](https://doc.akka.io/docs/akka/current/general/actors.html#mailbox)，[子级Actor](https://doc.akka.io/docs/akka/current/general/actors.html#child-actors)和[监督战略](https://doc.akka.io/docs/akka/current/general/actors.html#supervisor-strategy)的容器。所有的这些都封装在[Actor引用](https://doc.akka.io/docs/akka/current/general/actors.html#actor-reference)的后面。一个值得注意的方面是Actor有一个明确的生命周期，当它不再被引用时是不会被自动销毁的; 在创建了一个Actor之后，你有责任确保它最终能被终止 - 这也把在[演员终止](https://doc.akka.io/docs/akka/current/general/actors.html#when-an-actor-terminates)时释放资源的控制权交给了你。

## Actor引用

As detailed below, an actor object needs to be shielded from the outside in order to benefit from the actor model. Therefore, actors are represented to the outside using actor references, which are objects that can be passed around freely and without restriction. This split into inner and outer object enables transparency for all the desired operations: restarting an actor without needing to update references elsewhere, placing the actual actor object on remote hosts, sending messages to actors in completely different applications.
But the most important aspect is that it is not possible to look inside an actor and get hold of its state from the outside, unless the actor unwisely publishes this information itself.

如下所述，Actor对象需要对外部屏蔽，以便从Actor模型中受益。因此，Actor在外部以引用的方式表示，引用是不受约束可自由传递的对象。这种拆分为内部对象和外部对象的方式可以实现以下操作的透明性：重新启动一个Actor而不需要更新别处的引用；将实际的Actor对象放在远程主机上；将消息发送给完全不同的应用程序中的Actor。但是最重要的方面是，除非Actor傻傻的公布了其自身的状态，否则不可能从外部获取其内部状态。

## 状态

Actor objects will typically contain some variables which reflect possible states the actor may be in. This can be an explicit state machine (e.g. using the @ref:[FSM](../fsm.md) module), or it could be a counter, set of listeners,
pending requests, etc. These data are what make an actor valuable, and they must be protected from corruption by other actors. The good news is that Akka actors conceptually each have their own light-weight thread, which is completely shielded from the rest of the system. This means that instead of
having to synchronize access using locks you can just write your actor code without worrying about concurrency at all.

Actor对象通常会包含一些变量，这些变量反映了Actor可能处于的状态。这可以是显式状态机（e.g. 使用[FSM](https://doc.akka.io/docs/akka/current/fsm.html)模块），也可以是计数器，监听器集合，待处理的请求等。这些数据使得Actor有价值，他们必须得到保护以免其他Actor的干扰。好消息是，每个Akka Actor有自己概念上的轻量级线程，能够完全屏蔽系统的其他部分。这意味着，你不必使用锁来进行同步访问，可以随意编写你的代码而不用担心并发性。

Behind the scenes Akka will run sets of actors on sets of real threads, where typically many actors share one thread, and subsequent invocations of one actor may end up being processed on different threads. Akka ensures that this implementation detail does not affect the single-threadedness of handling the
actor’s state.

在底层，Akka Actor运行在一组真正的线程池上，通常很多Actor共享一个线程，并且一个Actor的随后调用可能在不同的线程上完成。Akka保证这个实现细节不会影响Actor状态处理的单线程性。

Because the internal state is vital to an actor’s operations, having inconsistent state is fatal. Thus, when the actor fails and is restarted by its supervisor, the state will be created from scratch, like upon first creating
the actor. This is to enable the ability of self-healing of the system.

由于内部状态对于Actor的操作至关重要，因此状态不一致是致命的。因此，当Actor失败并由其监督者重启时，状态将从头开始创建，就像首次创建Actor时那样。这是为了使系统能够自我修复。

Optionally, an actor's state can be automatically recovered to the state before a restart by persisting received messages and replaying them after restart (see @ref:[Persistence](../persistence.md)).

或者，通过持久化收到的消息和重启后回放（参见[持久性](https://doc.akka.io/docs/akka/current/persistence.html)），可以将Actor的状态自动恢复到重启之前。

## 行为

Every time a message is processed, it is matched against the current behavior of the actor. Behavior means a function which defines the actions to be taken in reaction to the message at that point in time, say forward a request if the client is authorized, deny it otherwise. This behavior may change over time, e.g. because different clients obtain authorization over time, or because the actor may go into an “out-of-service” mode and later come back. These changes are achieved by either encoding them in state variables which are read from the behavior logic, or the function itself may be swapped out at runtime, see the
`become` and `unbecome` operations. However, the initial behavior defined during construction of the actor object is special in the sense that a restart of the actor will reset its behavior to this initial one.

每次处理消息时，消息都会与Actor的当前行为进行匹配。行为是一个函数，它定义了当前时间点Actor能对消息作出的反应，例如，如果客户端被授权则转发请求，否则拒绝。这种行为可能随着时间而改变，例如，因为不同的客户端随着时间的推移而获得授权，或者因为Actor进入“停止服务”模式并且随后恢复。这些变化或者通过将它们编码到“行为”可读的状态变量中实现，或者在运行时由行为函数置换，请参阅`become`和`unbecome`操作。然而，在构造Actor对象时定义的初始行为是特殊的，因为重启Actor时会将其重置为初始行为。

## 信箱

An actor’s purpose is the processing of messages, and these messages were sent to the actor from other actors (or from outside the actor system). The piece which connects sender and receiver is the actor’s mailbox: each actor has exactly one mailbox to which all senders enqueue their messages. Enqueuing
happens in the time-order of send operations, which means that messages sent from different actors may not have a defined order at runtime due to the apparent randomness of distributing actors across threads. Sending multiple messages to the same target from the same actor, on the other hand, will enqueue them in the same order.

Actor的目的是处理消息，这些消息是从其他演员（或外部系统）发来的。而连接发送者和接收者的部分是Actor的信箱：每个Actor只有一个信箱，所有发件人都将消息排入信箱。排队按照消息发送时间顺序进行，这意味着由于跨线程分布Actor的明显随机性，从不同Actor发来的消息可能没有确定的顺序。而从同一个Actor向同一个目标发送的消息将按照相同的发出顺序排列它们。

There are different mailbox implementations to choose from, the default being a FIFO: the order of the messages processed by the actor matches the order in which they were enqueued. This is usually a good default, but applications may need to prioritize some messages over others. In this case, a priority mailbox
will enqueue not always at the end but at a position as given by the message priority, which might even be at the front. While using such a queue, the order of messages processed will naturally be defined by the queue’s algorithm and in general not be FIFO.

有不同的信箱实现可供选择，默认为FIFO：邮件处理的顺序与邮件入队顺序一致。这通常是一个很好的默认选择，但应用程序可能需要优先考虑其他消息的优先级。在这种情况下，优先级邮箱将不会始终将信件排在最后，而是按照邮件优先级给出的位置，这甚至可能会排在前面。当使用这样的队列时，处理消息的顺序自然由队列算法决定，而且通常不是FIFO。

An important feature in which Akka differs from some other actor model implementations is that the current behavior must always handle the next dequeued message, there is no scanning the mailbox for the next matching one.Failure to handle a message will typically be treated as a failure, unless this behavior is overridden.

Akka与其他Actor模型不同的一个重要特征是，当前行为必须处理下一个出队消息，而不会去信箱里扫描下一个能匹配的信件。如果未能处理消息，则将其视为失败，除非此行为被覆盖。

## 子级Actors

Each actor is potentially a supervisor: if it creates children for delegating sub-tasks, it will automatically supervise them. The list of children is maintained within the actor’s context and the actor has access to it.
Modifications to the list are done by creating (`context.actorOf(...)`) or stopping (`context.stop(child)`) children and these actions are reflected immediately. The actual creation and termination actions happen behind the scenes in an asynchronous way, so they do not “block” their supervisor.

每个Actor都可能是一个监督者：如果它创建子级Actor来委派子任务，它将自动监督它们。子级的名单保存在Actor的上下文中。对名单列表的修改通过创建行为（`context.actorOf(...)`）或停止行为（`context.stop(child)`）来完成，操作结果能够立即反映出来。真正的创建和终止操作都以异步的方式发生在底层，所以这不会“阻塞”他们的监督者。

## 监管策略

The final piece of an actor is its strategy for handling faults of its children. Fault handling is then done transparently by Akka, applying one of the strategies described in @ref:[Supervision and Monitoring](supervision.md) for each incoming failure. As this strategy is fundamental to how an actor system is structured, it cannot be changed once an actor has been created.

Actor最后一部分是处理其子级错误的策略。Akka能够透明地进行故障处理，应用[监督和监测](https://doc.akka.io/docs/akka/current/general/supervision.html)中描述的策略对每种故障进行处理。由于该策略是Actor系统的结构基础，所以一旦创建了Actor它就不能改变了。

Considering that there is only one such strategy for each actor, this means that if different strategies apply to the various children of an actor, the children should be grouped beneath intermediate supervisors with matching strategies, preferring once more the structuring of actor systems according to the splitting of tasks into sub-tasks.

考虑到每个Actor只有一个这样的策略，这意味着如果要使不同的策略适用于Actor不同的子级，那么应该通过匹配策略将子级Actor分组布置在中间监督者之下，再次验证了根据子任务分解来构建Actor系统。

## 当一个Actor终止时

Once an actor terminates, i.e. fails in a way which is not handled by a restart, stops itself or is stopped by its supervisor, it will free up its resources, draining all remaining messages from its mailbox into the system’s
“dead letter mailbox” which will forward them to the EventStream as DeadLetters. The mailbox is then replaced within the actor reference with a system mailbox,redirecting all new messages to the EventStream as DeadLetters. This is done on a best effort basis, though, so do not rely on it in order to construct “guaranteed delivery”.

一旦Actor终止，例如：以不能通过重启来处理的失败，停止自己或被其上级停止，它将释放其资源，将其信箱中的所有剩余邮件排入系统的“死信信箱”中，死信信箱会将它们作为DeadLetters转发给EventStream。然后Actor引用内的信箱被替换为系统信箱，将所有新信件以DeadLetters的形式重定向到EventStream。尽管这是在底层完成的，但不要依赖它来构建“交付保证”。(**尼玛……这整个一章都翻译的很烂！！！编者和上一大章节肯定不是一个人**)

The reason for not just silently dumping the messages was inspired by our tests: we register the TestEventListener on the event bus to which the dead letters are forwarded, and that will log a warning for every dead letter received—this has been very helpful for deciphering test failures more quickly. It is conceivable that this feature may also be of use for other purposes.

从我们的测试来看，不要悄悄地转储消息，因为我们在事件总线上注册了监听器TestEventListener，死亡信件会被转发到该事件总线上，并且每收到一封死信都会记录一条警告 - 这对更快地解释故障原因非常有效。可以想象的到，该特征也可以用于其他目的。