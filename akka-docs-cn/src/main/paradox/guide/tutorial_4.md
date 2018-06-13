# 第4部分: 使用设备组

Let's take a closer look at the main functionality required by our use case. In a complete IoT system for monitoring home temperatures, the steps for connecting a device sensor to our system might look like this:

让我们更进一步的看看我们用例所需的主要功能。在完整的用于监测家庭温度的物联网系统中，将设备传感器连接到我们系统中的步骤如下所示：

1. A sensor device in the home connects through some protocol.
1. 家中的传感器设备通过某种协议连接。
1. The component managing network connections accepts the connection.
1. 管理网络连接的组件接受连接。
1. The sensor provides its group and device ID to register with the device manager component of our system.
1. 传感器提供其属组和设备ID以注册到我们系统中的设备管理组件。
1. The device manager component handles registration by looking up or creating the actor responsible for keeping sensor state.
1. 设备管理器组件通过查找或创建负责保持传感器状态的Actor来处理注册行为。
1. The actor responds with an acknowledgement, exposing its `ActorRef`.
1. Actor返回确认标示以响应，并暴露其`ActorRef`。
1. The networking component now uses the `ActorRef` for communication between the sensor and device actor without going through the device manager.
1. 现在网络组件不再通过设备管理器，而使用`ActorRef`在传感器和设备Actor之间的通信。

Steps 1 and 2 take place outside the boundaries of our tutorial system. In this chapter, we will start addressing steps 3-6 and create a way for sensors to register with our system and to communicate with actors. But first, we have another architectural decision &#8212; how many levels of actors should we use to represent device groups and device sensors?

步骤1和步骤2不在我们的教程系统中。在本章中，我们将讨论步骤3-步骤6，并创建一种在系统中注册传感器并与Actor通讯的一种方法。但首先，我们要决定另一个架构上的问题 - 我们应该使用层级的Actor来表示设备组和设备传感器？

One of the main design challenges for Akka programmers is choosing the best granularity for actors. In practice, depending on the characteristics of the interactions between actors, there are usually several valid ways to organize a system. In our use case, for example, it would be possible to have a single actor maintain all the groups and devices  &#8212; perhaps using hash maps. It would also be reasonable to have an actor for each group that tracks the state of all devices in the same home.

Akka程序员面临的主要设计挑战之一是为Actor选择最佳粒度。在实践中，根据Actor之间相互作用的特点，通常有好几种有效的方式来组织系统。例如，在我们的用例中，可能会设计一个Actor来维护所有组和设备 - 也许会利用哈希映射。同样也可以为每个组创建一个Actor来跟踪同一家庭中的所有设备状态。

The following guidelines help us choose the most appropriate actor hierarchy:

下列指导原则有助于帮我们选择最合适的Actor层级：

  * In general, prefer larger granularity. Introducing more fine-grained actors than needed causes more problems than it solves.
  * 一般来说，偏好较大的粒度。引入多于所需的细粒度Actor相对于解决问题，反而可能会导致更多问题。
  * Add finer granularity when the system requires:
  * 当系统需要下列特性时增加更细的粒度:
      * Higher concurrency.
      * 更高的并发性。
      * Complex conversations between actors that have many  states. We will see a very good example for this in the next chapter.
      * Actor之间存在多状态的复杂对话。在下一章我们将看到一个很好的例子。
      * Sufficient state that it makes sense to divide into smaller actors.
      * 有足够的状态，分成更小的Actor更有意义。
      * Multiple unrelated responsibilities. Using separate actors allows individuals to fail and be restored with little impact on others.
      * 多重无关的责任。使用独立的Actor可以让其独自失败并且恢复，同时对其他Actor造成的影响最小。

## 设备管理器层级

Considering the principles outlined in the previous section, We will model the device manager component as an actor tree with three levels:

考虑到上一节中概述的原则，我们将设备管理器模型化为三级Actor树：

* The top level supervisor actor represents the system component for devices. It is also the entry point to look up and create device group and device actors.
* 顶级监管Actor代表设备的系统组件。它也是查找和创建设备组与设备Actor的入口点。
* At the next level, group actors each supervise the device actors for one group id (e.g. one home). They also provide services, such as querying temperature readings from all of the available devices in their group.
* 在下一级别，组Actor各自监管按组ID划分为小组的设备Actor（例如，一个家庭）。它还提供查询组中所有可用设备的温度读数等服务。
* Device actors manage all the interactions with the actual device sensors, such as storing temperature readings.
* 设备Actor管理与实际设备传感器有关的所有交互，例如存储温度读数。

![device manager tree](diagrams/device_manager_tree.png)

We chose this three-layered architecture for these reasons:

出于以下原因，我们选择了这种三层架构：

* Having groups of individual actors:
* 为个体Actor建立属组可以：
    * Isolates failures that occur in a group. If a single actor managed all device groups, an error in one group that causes a restart would wipe out the state of groups that are otherwise non-faulty.
    * 隔离组中发生的故障。如果单个Actor管理所有设备组，那么一个组中引发重启的错误则可能导致那些无故障组中状态的损毁。
    * Simplifies the problem of querying all the devices belonging to a group. Each group actor only contains state related to its group.
    * 简化了查询组中所有设备的问题。每个组Actor只包含与其组相关的状态。
    * Increases parallelism in the system. Since each group has a dedicated actor, they run concurrently and we can query multiple groups concurrently.
    * 增加系统中的并行性。由于每个组都有一个专门的Actor，他们同时运行，我们就可以同时查询多个组。


* Having sensors modeled as individual device actors:
* 将传感器建模为单个Actor可以：
    * Isolates failures of one device actor from the rest of the devices in the group.
    * 将一个设备Actor的故障与组中其余设备的故障隔离开来。
    * Increases the parallelism of collecting temperature readings. Network connections from different sensors communicate with their individual device actors directly, reducing contention points.
    * 提高收集温度读数的并行度。来自不同传感器的网络连接直接与其对应设备Actor进行通信，从而减少争用点。

With the architecture defined, we can start working on the protocol for registering sensors.

通过定义架构，我们可以开始着手传感器注册协议。

## 注册协议

As the first step, we need to design the protocol both for registering a device and for creating the group and device actors that will be responsible for it. This protocol will be provided by the `DeviceManager` component itself because that is the only actor that is known and available up front: device groups and device actors are created on-demand.

作为第一步，我们需要设计协议，用于注册设备以及创建响应协议的设备组和设备Actor。该协议由`DeviceManager`这个组件提供，因为这是仅有的前置有效的Actor：设备组和设备Actor是按需创建的。

Looking at registration in more detail, we can outline the necessary functionality:

更详细地看注册功能，我们可以概述出必要的功能：

1. When a `DeviceManager` receives a request with a group and device id:
1. 当一个 `DeviceManager`收到一个包含组和设备ID的请求时：
    * If the manager already has an actor for the device group, it forwards the request to it.
    * 如果这个管理者已经拥有设备组Actor，它会将请求转发给它。
    * Otherwise, it creates a new device group actor and then forwards the request.
    * 否则，它将创建一个新的设备组Actor，然后转发该请求。
1. The `DeviceGroup` actor receives the request to register an actor for the given device:
1. `DeviceGroup` Actor接收到注册给定设备Actor的请求：
    * If the group already has an actor for the device, the group actor forwards the request to the device actor.
    * 如果该组已经有该设备Actor，则组Actor将该请求转发给设备Actor。
    * Otherwise, the `DeviceGroup` actor first creates a device actor and then forwards the request.
    * 否则，`DeviceGroup`Actor首先创建一个设备Actor，然后转发请求。
1. The device actor receives the request and sends an acknowledgement to the original sender. Since the device actor acknowledges receipt (instead of the group actor), the sensor will now have the `ActorRef` to send messages directly to its actor.
1. 设备Actor收到请求并向原始发件人发送确认标示。由于是设备Actor确认的接收（而不是组Actor），因此传感器可以通过`ActorRef`直接向其对应的Actor发送消息。

The messages that we will use to communicate registration requests and their acknowledgement have a simple definition:

我们用来传递注册请求和返回确认标示的消息有一个简单的定义：

Scala
:   @@snip [DeviceManager.scala]($code$/scala/tutorial_4/DeviceManager.scala) { #device-manager-msgs }

Java
:   @@snip [DeviceManager.java]($code$/java/jdocs/tutorial_4/DeviceManager.java) { #device-manager-msgs }

In this case we have not included a request ID field in the messages. Since registration happens once, when the component connects the system to some network protocol, the ID is not important. However, it is usually a best practice to include a request ID.

在这种情况下，我们没有在消息中包含请求ID字段。由于注册一旦发生，当组件将系统连接到某个网络协议时(感觉不顺，看英文吧！！)，该ID并不重要。但是，包含请求ID通常是最佳做法。

Now, we'll start implementing the protocol from the bottom up. In practice, both a top-down and bottom-up approach can work, but in our case, we benefit from the bottom-up approach as it allows us to immediately write tests for the new features without mocking out parts that we will need to build later.

现在，我们将从下到上开始实施该协议。在实践中，自上而下和自下而上的方法都可行，但在我们的案例中，我们倾向于自下而上的方法，因为它允许我们立即为新功能编写测试，而不用去模拟我们稍后才去创建的部分。

## 为设备Actor添加注册支持

At the bottom of our hierarchy are the `Device` actors. Their job in the registration process is simple: reply to the registration request with an acknowledgment to the sender. It is also prudent to add a safeguard against requests that come with a mismatched group or device ID.

我们层次结构的底层是`Device`Actor。他们在注册过程中的作用很简单：以向发件人发送确认标示的方式回复注册请求。同时对带有不匹配的组或设备ID的请求信息添加安全保护。

*We will assume that the ID of the sender of the registration message is preserved in the upper layers.* We will show you in the next section how this can be achieved.

*我们假设注册消息的发送者ID被上层保存。*我们将在下一节向您展示如何实现这一目标。

The device actor registration code looks like the following. Modify your example to match.

设备Actor注册代码如下所示。修改您的示例以匹配。

Scala
:   @@snip [Device.scala]($code$/scala/tutorial_4/Device.scala) { #device-with-register }

Java
:   @@snip [Device.java]($code$/java/jdocs/tutorial_4/Device.java) { #device-with-register }

@@@ note { .group-scala }

We used a feature of scala pattern matching where we can check to see if a certain field equals an expected value. By bracketing variables with backticks, like `` `variable` ``, the pattern will only match if it contains the value of `variable` in that position.

我们使用了scala模式匹配功能，用来检查某个字段是否等于期望值。通过用反引号括起变量，就像``variable``这样，该模式只有在适当位置包含该`variable`值时才匹配。

@@@

We can now write two new test cases, one exercising successful registration, the other testing the case when IDs don't match:

我们现在可以编写两个新的测试用例，一个用于成功注册，另一个用于测试ID不匹配的情况：

Scala
:   @@snip [DeviceSpec.scala]($code$/scala/tutorial_4/DeviceSpec.scala) { #device-registration-tests }

Java
:   @@snip [DeviceTest.java]($code$/java/jdocs/tutorial_4/DeviceTest.java) { #device-registration-tests }

@@@ note

We used the `expectNoMsg()` helper method from @scala[`TestProbe`]@java[`TestKit`]. This assertion waits until the defined time-limit and fails if it receives any messages during this period. If no messages are received during the waiting period, the assertion passes. It is usually a good idea to keep these timeouts low (but not too low) because they add significant test execution time.

我们使用了`TestProbe`中`expectNoMsg()`helper方法。该断言一直等到定义的时限，如果在此期间内收到任何消息则失败。如果在等待期间没有收到消息，则断言通过。通常延时设的低些比较好（但不能太低），因为它们明显的增加了测试的执行时间。

@@@

## 向设备组Actor添加注册支持

We are done with registration support at the device level, now we have to implement it at the group level. A group actor has more work to do when it comes to registrations, including:

我们完成了设备级别的注册支持，现在我们必须在设备组级别实施。设备组Actor在注册时有更多工作要做，包括：

* Handling the registration request by either forwarding it to an existing device actor or by creating a new actor and forwarding the message.
* 处理注册请求，要么将注册请求转发给现有设备Actor，要么创建新的Actor然后转发请求信息。
* Tracking which device actors exist in the group and removing them from the group when they are stopped.
* 跟踪组内有哪些设备Actor，并在其停止时将其从组中删除。

### 处理注册请求

A device group actor must either forward the request to an existing child, or it should create one. To look up child actors by their device IDs we will use a @scala[`Map[String, ActorRef]`]@java[`Map<String, ActorRef>`].

设备组Actor必须将请求转发给现有的子级或者创建一个。要通过设备ID查找子Actor，我们将使用`Map[String, ActorRef]`。

We also want to keep the the ID of the original sender of the request so that our device actor can reply directly. This is possible by using `forward` instead of the @scala[`!`] @java[`tell`] operator. The only difference between the two is that `forward` keeps the original sender while @scala[`!`] @java[`tell`] sets the sender to be the current actor. Just like with our device actor, we ensure that we don't respond to wrong group IDs. Add the following to your source file:

我们还希望保留请求的原始发件人的ID，以便我们的设备Actor可以直接回复。这可以通过使用`forward`而不是`!` 操作符来实现。两者之间的唯一区别是`forward`保留原始发件人，而`!` 将发件人设置为当前的Actor。就像我们的设备Actor一样，我们确保我们不会对错误的组ID进行响应。将以下内容添加到您的源文件中：

Scala
:   @@snip [DeviceGroup.scala]($code$/scala/tutorial_4/DeviceGroup.scala) { #device-group-register }

Java
:   @@snip [DeviceGroup.java]($code$/java/jdocs/tutorial_4/DeviceGroup.java) { #device-group-register }

Just as we did with the device, we test this new functionality. We also test that the actors returned for the two different IDs are actually different, and we also attempt to record a temperature reading for each of the devices to see if the actors are responding.

就像我们对设备所做的那样，我们测试了这个新功能。我们还测试了对两个不同ID返回的Actor实际上是不同的，我们也试图记录每个设备的温度读数以便查看Actor是否响应。

Scala
:   @@snip [DeviceGroupSpec.scala]($code$/scala/tutorial_4/DeviceGroupSpec.scala) { #device-group-test-registration }

Java
:   @@snip [DeviceGroupTest.java]($code$/java/jdocs/tutorial_4/DeviceGroupTest.java) { #device-group-test-registration }

If a device actor already exists for the registration request, we would like to use the existing actor instead of a new one. We have not tested this yet, so we need to fix this:

如果注册请求设备Actor的已经存在，我们希望直接使用现有的Actor而不是新的。我们还没有对这个进行测试，所以我们需要解决这个问题：

Scala
:   @@snip [DeviceGroupSpec.scala]($code$/scala/tutorial_4/DeviceGroupSpec.scala) { #device-group-test3 }

Java
:   @@snip [DeviceGroupTest.java]($code$/java/jdocs/tutorial_4/DeviceGroupTest.java) { #device-group-test3 }

### 跟踪组中的设备Actor

So far, we have implemented logic for registering device actors in the group. Devices come and go, however, so we will need a way to remove device actors from the @scala[`Map[String, ActorRef]`] @java[`Map<String, ActorRef>`]. We will assume that when a device is removed, its corresponding device actor is simply stopped. Supervision, as we discussed earlier, only handles error scenarios &#8212; not graceful stopping. So we need to notify the parent when one of the device actors is stopped.

到目前为止，我们已经实现了在组中注册设备Actor的逻辑。但是新设备来旧设备去，所以我们需要一种方法来从`Map[String, ActorRef]`中移除设备Actor。假设当一个设备被移除时，其相应的设备Actor会被简单的停止。正如我们前面讨论过的，监督只处理错误情况 - 而不是优雅的停止。所以我们需要在其中设备Actor停止时通知其父级。

Akka provides a _Death Watch_ feature that allows an actor to _watch_ another actor and be notified if the other actor is stopped. Unlike supervision, watching is not limited to parent-child relationships, any actor can watch any other actor as long as it knows the `ActorRef`. After a watched actor stops, the watcher receives a `Terminated(actorRef)` message which also contains the reference to the watched actor. The watcher can either handle this message explicitly or will fail with a `DeathPactException`. This latter is useful if the actor can no longer perform its own duties after the watched actor has been stopped. In our case, the group should still function after one device have been stopped, so we need to handle the `Terminated(actorRef)` message.

Akka提供了一个*监视死亡*功能，允许Actor*监视*另一个Actor，并在其他Actor停止时被通知。与监督不同，观看不限于父子级别关系，任何Actor只要知道其他Actor的`ActorRef`就可以监视其他Actor。被监视的Actor停止后，监视者会收到一条包含被监视Actor引用的`Terminated(actorRef)`消息。监视者或者可以明确地处理这个消息，或者触发失败并显示一个`DeathPactException`异常。后面这个特性在被监视Actor停止后，监视Actor就无法履行自己的职能的情况下非常有用。在我们的案例中，当一个设备停止后，该属组应该仍然保持运行，因此我们需要处理`Terminated(actorRef)`消息。

Our device group actor needs to include functionality that:

我们的设备组Actor需要包含以下功能：

  1. Starts watching new device actors when they are created.
  2. 监视新创建的设备Actor。
  3. Removes a device actor from the @scala[`Map[String, ActorRef]`] @java[`Map<String, ActorRef>`] &#8212; which maps devices to device actors &#8212; when the notification indicates it has stopped.
  4. 当被通知设备停止后，从设备和设备Actor的映射表`Map[String, ActorRef]`中将设备Actor删除。

Unfortunately, the `Terminated` message only contains the `ActorRef` of the child actor. We need the actor's ID to remove it from the map of existing device to device actor mappings. To be able to do this removal, we need to introduce another placeholder, @scala[`Map[ActorRef, String]`] @java[`Map<ActorRef, String>`], that allow us to find out the device ID corresponding to a given `ActorRef`.

不幸的是，这条`Terminated`消息只包含了子级Actor的`ActorRef`。我们需要Actor的ID将其从现有设备映射表中删除。为了做到这一点，我们需要引入另一个占位符`Map[ActorRef, String]`，以便我们能够找出与给定 `ActorRef`对应的设备ID 。

Adding the functionality to identify the actor results in this:

添加标识Actor的功能结果如下：

Scala
:   @@snip [DeviceGroup.scala]($code$/scala/tutorial_4/DeviceGroup.scala) { #device-group-remove }

Java
:   @@snip [DeviceGroup.java]($code$/java/jdocs/tutorial_4/DeviceGroup.java) { #device-group-remove }

So far we have no means to get which devices the group device actor keeps track of and, therefore, we cannot test our new functionality yet. To make it testable, we add a new query capability (message @scala[`RequestDeviceList(requestId: Long)`] @java[`RequestDeviceList`]) that simply lists the currently active device IDs:

到目前为止，我们没有办法得知设备组Actor能跟踪哪些设备，因此我们还无法测试我们的新功能。为了使其可测试，我们添加了一个新的查询功能（消息`RequestDeviceList(requestId: Long)`），列出当前活动的设备ID：

Scala
:   @@snip [DeviceGroup.scala]($code$/scala/tutorial_4/DeviceGroup.scala) { #device-group-full }

Java
:   @@snip [DeviceGroup.java]($code$/java/jdocs/tutorial_4/DeviceGroup.java) { #device-group-full }

We are almost ready to test the removal of devices. But, we still need the following capabilities:

我们准备好了测试设备的移除功能。但是我们仍需要以下功能：

 * To stop a device actor from our test case. From the outside, any actor can be stopped by simply sending a special the built-in message, `PoisonPill`, which instructs the actor to stop.
 * 在我们的测试案例中停止设备Actor。从外面，任何Actor都可以通过发送一个特殊的内置消息`PoisonPill`来使目标Actor停止。
 * To be notified once the device actor is stopped. We can use the _Death Watch_ facility for this purpose, too. The @scala[`TestProbe`] @java[`TestKit`] has two messages that we can easily use, `watch()` to watch a specific actor, and `expectTerminated` to assert that the watched actor has been terminated.
 * 一旦设备Actor停止，就会收到通知。为此我们可以使用*监视死亡*功能。`TestProbe` 有两种简单易用的方法，`watch()`目标Actor和通过`expectTerminated`来断言被监视Actor是否终结。

We add two more test cases now. In the first, we just test that we get back the list of proper IDs once we have added a few devices. The second test case makes sure that the device ID is properly removed after the device actor has been stopped:

我们再增加两个测试用例。首先，我们要测试添加了一些设备后我们能返回正确的ID列表。第二个我们要测试在停止设备Actor后设备ID能够被正确的移除:

Scala
:   @@snip [DeviceGroupSpec.scala]($code$/scala/tutorial_4/DeviceGroupSpec.scala) { #device-group-list-terminate-test }

Java
:   @@snip [DeviceGroupTest.java]($code$/java/jdocs/tutorial_4/DeviceGroupTest.java) { #device-group-list-terminate-test }

## 创建设备管理员Actor

Going up to the next level in our hierarchy, we need to create the entry point for our device manager component in the `DeviceManager` source file. This actor is very similar to the device group actor, but creates device group actors instead of device actors:

继续往上，我们需要在`DeviceManager`源文件中为设备管理器组件创建入口。这个Actor与设备组Actor非常相似，只不过创建的是设备组Actor而不是设备Actor：

Scala
:   @@snip [DeviceManager.scala]($code$/scala/tutorial_4/DeviceManager.scala) { #device-manager-full }

Java
:   @@snip [DeviceManager.java]($code$/java/jdocs/tutorial_4/DeviceManager.java) { #device-manager-full }

We leave tests of the device manager as an exercise for you since it is very similar to the tests we have already written for the group actor.

我们将设备管理器的测试留作练习，因为它与设备组Actor的测试非常相似。

## 下一步

We have now a hierarchical component for registering and tracking devices and recording measurements. We have seen how to implement different types of conversation patterns, such as:

我们现在有一个层级组件用于注册和跟踪设备并记录测量结果。我们已经看到了如何实现不同类型的会话模式，例如：

 * Request-respond (for temperature recordings)
 * 请求-响应（用于温度记录）
 * Delegate-respond (for registration of devices)
 * 委托-响应（用于设备注册）
 * Create-watch-terminate (for creating the group and device actor as children)
 * 创建-观察-终止（用于创建设备组和设备）

In the next chapter, we will introduce group query capabilities, which will establish a new conversation pattern of scatter-gather. In particular, we will implement the functionality that allows users to query the status of all the devices belonging to a group.

在下一章中，我们将介绍设备组查询功能，这将建立一个新的分散-聚集对话模式。特别是，我们将实现用户查询属于某个组的所有设备状态。