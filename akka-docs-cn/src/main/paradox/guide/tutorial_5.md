# 第5部分：查询设备组

The conversational patterns that we have seen so far are simple in the sense that they require the actor to keep little or no state. Specifically:

我们目前看到的会话模式很简单，是因为Actor没有状态或者保持很少状态。特别是：

* Device actors return a reading, which requires no state change
* 设备Actor返回读数，这不需要改变状态
* Record a temperature, which updates a single field
* 记录温度，只需更新单个字段
* Device Group actors maintain group membership by simply adding or removing entries from a map
* 设备组Actor只需通过添加或删除Map中的条目来维护设备组成员资格

In this part, we will use a more complex example. Since homeowners will be interested in the temperatures throughout their home, our goal is to be able to query all of the device actors in a group. Let us start by investigating how such a query API should behave.

在这一部分中，我们将使用一个更复杂的例子。由于房主会对整个家中的温度感兴趣，因此我们的目标是能够查询设备组中的所有设备Actor。让我们先从研究这样一个查询API应该具有怎样的行为开始。

## 处理可能的情况

The very first issue we face is that the membership of a group is dynamic. Each sensor device is represented by an actor that can stop at any time. At the beginning of the query, we can ask all of the existing device actors for the current temperature. However, during the lifecycle of the query:

我们面临的第一个问题是，设备组成员是动态的。每个Actor代表的传感器设备都可能随时停止。在查询开始时，我们可以询问所有设备Actor当前的温度。然而，在查询的生命周期中：

 * A device actor might stop and not be able to respond back with a temperature reading.
 * 设备Actor可能会停止，无法用温度读数回应。
 * A new device actor might start up and not be included in the query because we weren't aware of it.
 * 可能会启动新的设备Actor并且它不被包含在本次查询中，因为我们还没有发现它。

These issues can be addressed in many different ways, but the important point is to settle on the desired behavior. The following works well for our use case:

这些问题可以用不同的方式解决，但重要的一点是要解决需要解决的行为。以下内容适用于我们的用例：

 * When a query arrives, the group actor takes a _snapshot_ of the existing device actors and will only ask those actors for the temperature.
 * 当查询时，设备组Actor只会获取当时存在的设备Actor快照，并只会向这些Actor询问温度。
 * Actors that start up _after_ the query arrives are simply ignored.
 * 查询*之后*启动的Actor将被忽略。
 * If an actor in the snapshot stops during the query without answering, we will simply report the fact that it stopped to the sender of the query message.
 * 如果快照中的Actor在查询期间停止从而应答，我们将如实向查询消息的发送者报告其停止的情况。

Apart from device actors coming and going dynamically, some actors might take a long time to answer. For example, they could be stuck in an accidental infinite loop, or fail due to a bug and drop our request. We don't want the query to continue indefinitely, so we will consider it complete in either of the following cases:

除了设备Actor动态地新增删除外，一些Actor可能响应时间较长。例如，他们可能陷入意外的无限循环，或者由于BUG而失败并放弃响应我们的请求。我们不希望查询无限期地继续下去，因此我们会在以下任一情况下认为它已完成：

* All actors in the snapshot have either responded or have confirmed being stopped.
* 快照中的所有Actor要么都已响应要么确认正在停止。
* We reach a pre-defined deadline.
* 达到预定义的时限。

Given these decisions, along with the fact that a device in the snapshot might have just started and not yet received a temperature to record, we can define four states for each device actor, with respect to a temperature query:

鉴于这些决定，以及快照中的设备可能刚刚启动并且尚未收到记录温度这一事实，我们可以针对每个设备Actor定义四种温度查询状态：

 * It has a temperature available: @scala[`Temperature(value)`] @java[`Temperature`].
 * 有可用的温度：`Temperature(value)`。
 * It has responded, but has no temperature available yet: `TemperatureNotAvailable`.
 * 能回应，但没有可用的温度：`TemperatureNotAvailable`。
 * It has stopped before answering: `DeviceNotAvailable`.
 * 在回答之前就已经停止：`DeviceNotAvailable`。
 * It did not respond before the deadline: `DeviceTimedOut`.
 * 响应超时：`DeviceTimedOut`。

Summarizing these in message types we can add the following to `DeviceGroup`:

总结上述消息类型，我们可以将以下内容添加到`DeviceGroup`中：

Scala
:   @@snip [DeviceGroup.scala]($code$/scala/tutorial_5/DeviceGroup.scala) { #query-protocol }

Java
:   @@snip [DeviceGroup.java]($code$/java/jdocs/tutorial_5/DeviceGroup.java) { #query-protocol }

## 实现查询功能

One approach for implementing the query involves adding code to the group device actor. However, in practice this can be very cumbersome and error prone. Remember that when we start a query, we need to take a snapshot of the devices present and start a timer so that we can enforce the deadline. In the meantime, _another query_ can arrive. For the second query, of course, we need to keep track of the exact same information but in isolation from the previous query. This would require us to maintain separate mappings between queries and device actors.

我们通过向设备组Actor中添加代码来实现查询功能。但是，实际上这可能非常麻烦并且容易出错。记得当我们开始查询时，我们需要获取当前设备的快照并启动一个计时器，以便我们能够确认时限。而同时可能发生*另一个查询*。对于第二个查询我们也要跟踪完全相同的信息，但还要与前一个查询隔离。这就需要我们维护查询和设备Actor之间的分离的映射关系。

Instead, we will implement a simpler, and superior approach. We will create an actor that represents a _single query_ and that performs the tasks needed to complete the query on behalf of the group actor. So far we have created actors that belonged to classical domain objects, but now, we will create an actor that represents a process or a task rather than an entity. We benefit by keeping our group device actor simple and being able to better test query capability in isolation.

相反，我们可以实现一个更简单，更优越的方法。我们将创建一个代表*单个查询*的Actor，并让其执行代表设备组Actor完成查询的任务。到目前为止，我们已经创建了属于经典领域对象的Actor，但是现在，我们将创建一个代表一个进程或一个任务而不是一个实体的的Actor。保持设备组Actor的简单性和独立的测试查询功能将带来更好的效果。

### 定义查询Actor

First, we need to design the lifecycle of our query actor. This consists of identifying its initial state, the first action it will take, and the cleanup &#8212; if necessary. The query actor will need the following information:

首先，我们需要设计查询Actor的生命周期。这包括识别其初始状态，首先采取的行动以及可能必要的清理。查询Actor需要以下信息：

 * The snapshot and IDs of active device actors to query.
 * 待查询的活动设备Actor的快照和ID。
 * The ID of the request that started the query (so that we can include it in the reply).
 * 查询请求的ID（以便我们可以包含在回复中）。
 * The reference of the actor who sent the query. We will send the reply to this actor directly.
 * 查询发送者的引用。我们会直接向这个Actor发送回复。
 * A deadline that indicates how long the query should wait for replies. Making this a parameter will simplify testing.
 * 等待回复的时限。将其作为参数可简化测试。

#### 调度查询超时

Since we need a way to indicate how long we are willing to wait for responses, it is time to introduce a new Akka feature that we have not used yet, the built-in scheduler facility. Using the scheduler is simple:

由于我们需要一种方法来表示我们愿意等待响应多长时间，那么是时候引入一种我们还没用过的新功能了，即内置的调度器。使用调度器很简单：

* We get the scheduler from the `ActorSystem`, which, in turn,is accessible from the actor's context: @scala[`context.system.scheduler`]@java[`getContext().getSystem().scheduler()`]. This needs an @scala[implicit] `ExecutionContext` which is basically the thread-pool that will execute the timer task itself. In our case, we use the same dispatcher as the actor by @scala[importing `import context.dispatcher`] @java[passing in `getContext().dispatcher()`].
* 我们从`ActorSystem`中获得调度器，而调度器又可以从actor的上下文中被访问：`context.system.scheduler`。这里需要一个执行定时器任务的隐含的线程池 `ExecutionContext`。在我们的例子中，我们通过导入`import context.dispatcher`使用与Actor相同的调度器。
* The @scala[`scheduler.scheduleOnce(time, actorRef, message)`] @java[`scheduler.scheduleOnce(time, actorRef, message, executor, sender)`] method will schedule the message `message` into the future by the specified `time` and send it to the actor `actorRef`.
* `scheduler.scheduleOnce(time, actorRef, message)`方法将在指定的时间`time`后，向具体的Actor引用`actorRef`，发送消息`message`。

We need to create a message that represents the query timeout. We create a simple message `CollectionTimeout` without any parameters for this purpose. The return value from `scheduleOnce` is a `Cancellable` which can be used to cancel the timer if the query finishes successfully in time.  At the start of the query, we need to ask each of the device actors for the current temperature. To be able to quickly
detect devices that stopped before they got the `ReadTemperature` message we will also watch each of the actors. This way, we get `Terminated` messages for those that stop during the lifetime of the query, so we don't need to wait until the timeout to mark these as not available.

我们需要创建一个能表示查询超时的消息。为此我们创建了一个没有任何参数的简单消息`CollectionTimeout`。如果查询能及时完成，则`scheduleOnce`返回消息`Cancellable`用于关闭定时器。在查询开始时，我们需要询问每个设备Actor当前的温度。为了能够快速检测到在获取`ReadTemperature`消息之前就停止的设备，我们还将监视每个Actor。通过这种方式，我们可以通过获取`Terminated`消息，得知那些在查询生命周期中停止的设备，这样我们就不需要等待超时才能将这些设备标记为不可用。

Putting this together, the outline of our `DeviceGroupQuery` actor looks like this:

综合起来，我们`DeviceGroupQuery`Actor大致如下所示：

Scala
:   @@snip [DeviceGroupQuery.scala]($code$/scala/tutorial_5/DeviceGroupQuery.scala) { #query-outline }

Java
:   @@snip [DeviceGroupQuery.java]($code$/java/jdocs/tutorial_5/DeviceGroupQuery.java) { #query-outline }

#### 跟踪Actor状态

The query actor, apart from the pending timer, has one stateful aspect, tracking the set of actors that: have replied, have stopped, or have not replied. One way to track this state is to create a mutable field in the actor @scala[(a `var`)]. A different approach takes advantage of the ability to change how an actor responds to messages. A `Receive` is just a function (or an object, if you like) that can be returned from another function. By default, the `receive` block defines the behavior of the actor, but it is possible to change it multiple times during the life of the actor. We simply call `context.become(newBehavior)` where `newBehavior` is anything with type `Receive` @scala[(which is just a shorthand for `PartialFunction[Any, Unit]`)].  We will leverage this feature to track the state of our actor.

除了等待中的计时器以外，查询Actor有一个有状态的方面，跟踪那些已经回复、已经停止或未回复的Actor集合。跟踪这种状态的一种方法是在Actor中创建一个可变字段（ `var`）。另一种不同的方法是利用了改变Actor响应消息的能力。`Receive`只是一个通过函数返回的函数（或者对象，如果你喜欢的话）。默认情况下，`receive`块定义了Actor的行为，但我们通过`context.become(newBehavior)`可以在Actor的生命周期中多次改变它。这里`newBehavior`是`Receive` （一个简写的`PartialFunction[Any, Unit]`）类型。我们将利用此功能来跟踪我们Actor的状态。

For our use case:

对于我们的用例：

1. Instead of defining `receive` directly, we delegate to a `waitingForReplies` function to create the `Receive`.
1. 不直接定义`receive`，而是委托给一个`waitingForReplies`函数来创建`Receive`。
1. The `waitingForReplies` function will keep track of two changing values:
1. 该`waitingForReplies`函数将跟踪两个变化值:
   * a `Map` of already received replies
   * 一个已经收到的回复的`Map`
   * a `Set` of actors that we still wait on
   * 一个还在等待中的Actor的`Set`
1. We have three events to act on:
1. 我们要做如下三件事：
   * We can receive a `RespondTemperature` message from one of the devices.
   * 我们可以从某台设备收到消息`RespondTemperature`。
   * We can receive a `Terminated` message for a device actor that has been stopped in the meantime.
   * 我们可以收到一条`Terminated`消息，表示在此期间设备Actor已停止。
   * We can reach the deadline and receive a `CollectionTimeout`.
   * 我们可以在超时后收到一条`CollectionTimeout`消息。

In the first two cases, we need to keep track of the replies, which we now simply delegate to a method `receivedResponse`, which we will discuss later. In the case of timeout, we need to simply take all the actors that have not yet replied yet (the members of the set `stillWaiting`) and put a `DeviceTimedOut` as the status in the final reply. Then we reply to the submitter of the query with the collected results and stop the query actor.

在前两种情况下，我们需要跟踪回复的消息，我们这个委托给方法`receivedResponse`，这个我们稍后讨论。在超时的情况下，我们将获取尚未回复的所有Actor（`stillWaiting`组的成员）并将`DeviceTimedOut`作为最终答复的状态。然后，我们将收集的结果答复给查询提交者，并停止查询Actor。

To accomplish this, add the following to your `DeviceGroupQuery` source file:

要做到这一点，请将以下内容添加到您的`DeviceGroupQuery`源文件中：

Scala
:   @@snip [DeviceGroupQuery.scala]($code$/scala/tutorial_5/DeviceGroupQuery.scala) { #query-state }

Java
:   @@snip [DeviceGroupQuery.java]($code$/java/jdocs/tutorial_5/DeviceGroupQuery.java) { #query-state }

It is not yet clear how we will "mutate" the `repliesSoFar` and `stillWaiting` data structures. One important thing to note is that the function `waitingForReplies` **does not handle the messages directly. It returns a `Receive` function that will handle the messages**. This means that if we call `waitingForReplies` again, with different parameters,then it returns a brand new `Receive` that will use those new parameters.

目前尚不清楚如何修改`repliesSoFar`和`stillWaiting`数据结构。需要注意的一点是`waitingForReplies` 函数**不直接处理消息。它返回一个处理消息的`Receive`函数**。这意味着如果我们利用不同的参数再次调用`waitingForReplies`，那么它会返回一个包含这些新参数的全新的`Receive`函数。

We have seen how we can install the initial `Receive` by simply returning it from `receive`. In order to install a new one, to record a new reply, for example, we need some mechanism. This mechanism is the method `context.become(newReceive)` which will _change_ the actor's message handling function to the provided `newReceive` function. You can imagine that before starting, your actor automatically calls `context.become(receive)`, i.e. installing the `Receive` function that is returned from `receive`. This is another important observation: **it is not `receive` that handles the messages,it just returns a `Receive` function that will actually handle the messages**.

我们已经看到我们如何通过`receive`函数来装载初始的`Receive`类型实例。例如，为了记录一个新的回复，我们需要一定的机制来装载一个新的Receive类型实例。这就是`context.become(newReceive)`，这将*改变*Actor的消息处理功能。你可以想象，在开始之前，你的Actor会自动调用`context.become(receive)`，即装载`Receive`类型实例-即函数`receive`的返回内容。另外重要的一点是：**receive函数本身不处理消息，它只是返回一个处理消息的Receive类型的函数实例**。

We now have to figure out what to do in `receivedResponse`. First, we need to record the new result in the map `repliesSoFar` and remove the actor from `stillWaiting`. The next step is to check if there are any remaining actors we are waiting for. If there is none, we send the result of the query to the original requester and stop the query actor. Otherwise, we need to update the `repliesSoFar` and `stillWaiting` structures and wait for more messages.

现在我们要弄清楚`receivedResponse`要做什么。首先，我们需要在Map`repliesSoFar`中记录新的结果并从`stillWaiting`中删除对应Actor。下一步是检查是否还有正在等待的其他Actor。如果没有，我们给原始请求者发送查询结果并停止查询Actor。否则，我们需要更新`repliesSoFar`和`stillWaiting`并等待更多消息。

In the code before, we treated `Terminated` as the implicit response `DeviceNotAvailable`, so `receivedResponse` does not need to do anything special. However, there is one small task we still need to do. It is possible that we receive a proper response from a device actor, but then it stops during the lifetime of the query. We don't want this second event to overwrite the already received reply. In other words, we don't want to receive `Terminated` after we recorded the response. This is simple to achieve by calling `context.unwatch(ref)`. This method also ensures that we don't receive `Terminated` events that are already in the mailbox of the actor. It is also safe to call this multiple times, only the first call will have any effect, the rest is simply ignored.

在之前的代码中，我们将`Terminated`视为`DeviceNotAvailable`的隐式响应，因此`receivedResponse`不需要做任何事。但是，我们仍然要做件小事。设备Actor可能会在我们收到来自其正确响应后停止。我们不希望后者事件覆盖已收到的回复。换句话说，我们不想在记录了回应之后还收到`Terminated`消息。这个需求可以通过调用`context.unwatch(ref)`实现。这种方法还可以确保即使`Terminated`消息已经在Actor邮箱中了，我们也不会收到这个消息。多次调用这个函数也是安全的，因为只有第一次调用会产生影响，而后面的都会被忽略。

With all this knowledge, we can create the `receivedResponse` method:

有了这些知识，我们将创建方法`receivedResponse`：

Scala
:   @@snip [DeviceGroupQuery.scala]($code$/scala/tutorial_5/DeviceGroupQuery.scala) { #query-collect-reply }

Java
:   @@snip [DeviceGroupQuery.java]($code$/java/jdocs/tutorial_5/DeviceGroupQuery.java) { #query-collect-reply }

It is quite natural to ask at this point, what have we gained by using the `context.become()` trick instead of
just making the `repliesSoFar` and `stillWaiting` structures mutable fields of the actor (i.e. `var`s)? In this simple example, not that much. The value of this style of state keeping becomes more evident when you suddenly have _more kinds_ of states. Since each state might have temporary data that is relevant itself, keeping these as fields would pollute the global state of the actor, i.e. it is unclear what fields are used in what state. Using parameterized `Receive` "factory" methods we can keep data private that is only relevant to the state. It is still a good exercise to rewrite the query using @scala[`var`s] @java[mutable fields] instead of `context.become()`. However, it is recommended to get comfortable with the solution we have used here as it helps structuring more complex actor code in a cleaner and more maintainable way.

通过使用`context.become()`而不是将 `repliesSoFar`和`stillWaiting`变成可变变量（即`var`），到底有什么好处？在这一点上发问是自然而然的，在这个简单的例子中。当你突然拥有*更多*的状态时，这种保持状态的风格的优势就会变得更加明显。由于每个状态可能都有与自身相关的临时数据，所以将这些数据保存为字段会污染该Actor的全局状态，即不清楚在哪种状态下使用了哪些字段。使用参数化的`Receive`“工厂”方法，我们可以保持仅与状态相关的数据私密性。用`var`代替`context.become()`重写查询也是一个很好的习惯。不过，建议您习惯我们在此使用的解决方案，因为它有助于以更清洁和更易维护的方式构建更复杂的Actor代码。

Our query actor is now done:

我们的查询演员现在完成了：

Scala
:   @@snip [DeviceGroupQuery.scala]($code$/scala/tutorial_5/DeviceGroupQuery.scala) { #query-full }

Java
:   @@snip [DeviceGroupQuery.java]($code$/java/jdocs/tutorial_5/DeviceGroupQuery.java) { #query-full }

### 测试查询Actor

Now let's verify the correctness of the query actor implementation. There are various scenarios we need to test individually to make sure everything works as expected. To be able to do this, we need to simulate the device actors somehow to exercise various normal or failure scenarios. Thankfully we took the list of collaborators (actually a `Map`) as a parameter to the query actor, so we can easily pass in @scala[`TestProbe`] @java[`TestKit`] references. In our first test, we try out the case when there are two devices and both report a temperature:

现在我们来验证查询Actor实现的正确性。我们需要单独进行测试以确保一切按预期工作。为了做到这一点，我们需要模拟设备Actor以某种方式来执行各种正常或故障场景。值得庆幸的是，我们将协作者列表（一个 `Map`）作为查询Actor的参数，因此我们可以轻易传入`TestProbe`引用。在我们的第一个测试中，我们将尝试两个设备并报告温度的情况：

Scala
:   @@snip [DeviceGroupQuerySpec.scala]($code$/scala/tutorial_5/DeviceGroupQuerySpec.scala) { #query-test-normal }

Java
:   @@snip [DeviceGroupQueryTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupQueryTest.java) { #query-test-normal }

That was the happy case, but we know that sometimes devices cannot provide a temperature measurement. This scenario is just slightly different from the previous:

这是令人高兴的情况，但我们知道有时设备不能提供温度值。这种情况与以前的情况略有不同：

Scala
:   @@snip [DeviceGroupQuerySpec.scala]($code$/scala/tutorial_5/DeviceGroupQuerySpec.scala) { #query-test-no-reading }

Java
:   @@snip [DeviceGroupQueryTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupQueryTest.java) { #query-test-no-reading }

We also know, that sometimes device actors stop before answering:

我们也知道，有时候设备Actor在回应之前就停止了：

Scala
:   @@snip [DeviceGroupQuerySpec.scala]($code$/scala/tutorial_5/DeviceGroupQuerySpec.scala) { #query-test-stopped }

Java
:   @@snip [DeviceGroupQueryTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupQueryTest.java) { #query-test-stopped }

If you remember, there is another case related to device actors stopping. It is possible that we get a normal reply from a device actor, but then receive a `Terminated` for the same actor later. In this case, we would like to keep the first reply and not mark the device as `DeviceNotAvailable`. We should test this, too:

如果您还记得，还有另一个与设备Actor停止相关的案例。我们可能会收到来自设备Actor的正常答复，但随后会收到同一Actor的`Terminated`答复。在这种情况下，我们希望保留第一个答复，而不是将该设备标记为`DeviceNotAvailable`。我们也应该测试一下：

Scala
:   @@snip [DeviceGroupQuerySpec.scala]($code$/scala/tutorial_5/DeviceGroupQuerySpec.scala) { #query-test-stopped-later }

Java
:   @@snip [DeviceGroupQueryTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupQueryTest.java) { #query-test-stopped-later }

The final case is when not all devices respond in time. To keep our test relatively fast, we will construct the
`DeviceGroupQuery` actor with a smaller timeout:

最后一种情况是，并非所有设备都能及时响应。为了保持我们的测试速度相对较快，我们在构建`DeviceGroupQuery`Actor时将超时期限调小：

Scala
:   @@snip [DeviceGroupQuerySpec.scala]($code$/scala/tutorial_5/DeviceGroupQuerySpec.scala) { #query-test-timeout }

Java
:   @@snip [DeviceGroupQueryTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupQueryTest.java) { #query-test-timeout }

Our query works as expected now, it is time to include this new functionality in the `DeviceGroup` actor now.

现在我们的查询按预期工作，现在是时候在`DeviceGroup`Actor中引入这个新功能了。

## 将查询功能添加到组

Including the query feature in the group actor is fairly simple now. We did all the heavy lifting in the query actor itself, the group actor only needs to create it with the right initial parameters and nothing else.

设备组Actor中包含的查询功能现在非常简单。我们在查询Actor中完成了所有繁重的工作，设备组Actor只需要使用正确的初始参数创建它，而不需做其他事。

Scala
:   @@snip [DeviceGroup.scala]($code$/scala/tutorial_5/DeviceGroup.scala) { #query-added }

Java
:   @@snip [DeviceGroup.java]($code$/java/jdocs/tutorial_5/DeviceGroup.java) { #query-added }

It is probably worth restating what we said at the beginning of the chapter. By keeping the temporary state that is only relevant to the query itself in a separate actor we keep the group actor implementation very simple. It delegates everything to child actors and therefore does not have to keep state that is not relevant to its core business. Also, multiple queries can now run parallel to each other, in fact, as many as needed. In our case querying an individual device actor is a fast operation, but if this were not the case, for example, because the remote sensors need to be contacted over the network, this design would significantly improve throughput.

我们在本章开头所说的话可能值得重申。通过将与查询本身相关的临时状态保留在单独的Actor中，我们得以保持设备组Actor的实现非常简单。它将所有事情都委托给子级Actor，因此不必保留与其核心业务无关的状态。而且，现在可以按需并行运行多个查询。在我们的例子中，查询单个设备Actor是很快的，但如果情况并非如此，例如，因为需要通过网络联系远程传感器，则此设计将显着提高吞吐量。

We close this chapter by testing that everything works together. This test is just a variant of the previous ones, now exercising the group query feature:

我们通过完成工作协同测试来结束本章。此测试是以前测试的一个变体，现在正在使用设备组查询功能：

Scala
:   @@snip [DeviceGroupSpec.scala]($code$/scala/tutorial_5/DeviceGroupSpec.scala) { #group-query-integration-test }

Java
:   @@snip [DeviceGroupTest.java]($code$/java/jdocs/tutorial_5/DeviceGroupTest.java) { #group-query-integration-test }

## 概要

In the context of the IoT system, this guide introduced the following concepts, among others. You can follow the links to review them if necessary:

在物联网系统的背景下，本指南介绍了以下概念。如有必要，您可以按照链接查看它们：

* @ref:[The hierarchy of actors and their lifecycle(Actor的阶层和他们的生命周期)](tutorial_1.md)
* @ref:[The importance of designing messages for flexibility(设计灵活的消息的重要性)](tutorial_3.md)
* @ref:[How to watch and stop actors, if necessary(如有必要，如何监视和停止Actor)](tutorial_4.md#keeping-track-of-the-device-actors-in-the-group)

## 下一步

To continue your journey with Akka, we recommend:

要继续与Akka的学习，我们建议：

* Start building your own applications with Akka, make sure you [get involved in our amazing community](http://akka.io/get-involved) for help if you get stuck.
* 开始用Akka构建自己的应用程序，确保您陷入困境时请在[我们出色的社区中](http://akka.io/get-involved)寻求帮助。
* If you’d like some additional background, read the rest of the reference documentation and check out some of the @ref:[books and videos](../additional/books.md) on Akka.
* 如果您想要一些额外的背景信息，请阅读参考文档的其余部分，并参阅Akka上的一些[书籍和视频](https://doc.akka.io/docs/akka/current/additional/books.html)。

To get from this guide to a complete application you would likely need to provide either an UI or an API. For this we recommend that you look at the following technologies and see what fits you:

要从学习本指南到搭建完整的应用程序，您可能需要提供UI或API。为此，建议您查看以下适合您情况的具体技术：

 * [Akka HTTP](https://doc.akka.io/docs/akka-http/current/introduction.html) is a HTTP server and client library, making it possible to publish and consume HTTP endpoints
 * [Akka HTTP](https://doc.akka.io/docs/akka-http/current/introduction.html)是一个HTTP服务器和客户端库，可以发布和使用HTTP节点
 * [Play Framework](https://www.playframework.com) is a full fledged web framework that is built on top of Akka HTTP, it integrates well with Akka and can be used to create a complete modern web UI 
 * [Play Framework](https://www.playframework.com/)是一个完全成熟的Web框架，建立在Akka HTTP之上，与Akka很好地集成，可用于创建完整的Web UI
 * [Lagom](https://www.lagomframework.com) is an opinionated microservice framework built on top of Akka, encoding many best practices around Akka and Play 
 * [Lagom](https://www.lagomframework.com/)是一个基于Akka的有见地的微服务框架，汇集了许多有关Akka和Play的最佳实践
