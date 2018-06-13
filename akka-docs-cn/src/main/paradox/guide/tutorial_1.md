# 第1部分：Actor架构

Use of Akka relieves you from creating the infrastructure for an actor system and from writing the low-level code necessary to control basic behavior. To appreciate this, let's look at the relationships between actors you create in your code and those that Akka creates and manages for you internally, the actor lifecycle, and failure handling.

使用Akka可以帮助您减轻创建Actor系统基础结构和编写控制基本行为所需的底层代码的麻烦。要了解这一点，让我们来深入看看在代码中创建的Actor和Akka创建及管理的内部行为之间的关系，如角色生命周期和失败处理。

## Akka Actor层次结构

An actor in Akka always belongs to a parent. Typically, you create an actor by calling  @java[`getContext().actorOf()`] @scala[`context.actorOf()`]. Rather than creating a "freestanding" actor, this injects the new actor as a child into an already existing tree: the creator actor becomes the
_parent_ of the newly created _child_ actor. You might ask then, who is the parent of the _first_ actor you create?

Akka的Actor总是隶属于其父级。通常情况下，你通过调用`context.actorOf()`创建一个actor 。而不是创建一个“独立”的Actor，这种方式将Actor作为一个子节点注入到现有的树中：负责创建的Actor成为被创建的子Actor的*父级*。那么你可能会问，谁是你创造的*第一个*Actor的父级？

As illustrated below, all actors have a common parent, the user guardian. New actor instances can be created under this actor using `system.actorOf()`. As we covered in the @scala[[Quickstart Guide](https://developer.lightbend.com/guides/akka-quickstart-scala/)] @java[[Quickstart Guide](https://developer.lightbend.com/guides/akka-quickstart-java/)], creation of an actor returns a reference that is a valid URL. So, for example, if we create an actor named `someActor` with `system.actorOf(…, "someActor")`, its reference will include the path `/user/someActor`.

如下图所示，所有Actor都有一个共同的父母，即用户监护人。新的Actor实例可以通过`system.actorOf()`被这个Actor创建。正如在[快速入门指南](https://developer.lightbend.com/guides/akka-quickstart-scala/)中所介绍的，创建一个Actor会返回一个有效URL的引用。因此，例如，如果我们利用`system.actorOf(…, "someActor")`创建一个名为`someActor`的actor ，其引用将包含路径`/user/someActor`

![box diagram of the architecture](diagrams/actor_top_tree.png)

In fact, before you create an actor in your code, Akka has already created three actors in the system. The names of these built-in actors contain _guardian_ because they _supervise_ every child actor in their path. The guardian actors include:

事实上，在您创建Actor之前，Akka已经在系统中创建了三个Actor。这些内置Actor的名字包含监护人*，*因为他们*监督*他们路径下的每个子Actor。监护人Actor包括：

 - `/` the so-called _root guardian_. This is the parent of all actors in the system, and the last one to stop when the system itself is terminated.
 - `/`所谓的*根监护人*。这是系统中所有Actor的父级，系统终止时的最后一个Actor。
 - `/user` the _guardian_. **This is the parent actor for all user created actors**. Don't let the name `user` confuse you, it has nothing to do with end users, nor with user handling. Every actor you create using the Akka library will have the constant path `/user/` prepended to it.
 - `/user`*监护人*。**这是所有由用户创建的Actor的父Actor**。不要被名称`user`困惑，它与最终用户无关，也与用户处理无关。您使用Akka库创建的每个Actor都将拥有前缀`/user/`。
 - `/system` the _system guardian_.
 - `/system`*系统的监护人*。

In the Hello World example, we have already seen how `system.actorOf()`, creates an actor directly under `/user`. We call this a _top level_ actor, even though, in practice it is only on the top of the _user defined_ hierarchy. You typically have only one (or very few) top level actors in your `ActorSystem`.
We create child, or non-top-level, actors by invoking `context.actorOf()` from an existing actor. The `context.actorOf()` method has a signature identical to `system.actorOf()`, its top-level counterpart.

在Hello World示例中，我们已经看到了如何利用`system.actorOf()`直接在 `/user`下创建一个Actor。我们称之为*顶级Actor*，尽管在实践中它只是在*用户能定义的*层次结构的顶部。通常在`ActorSystem`中只有一个（或很少几个）顶级演员。我们通过在现有的Actor中调用`context.actorOf()`来创建非顶级的子Actor。该`context.actorOf()`方法与其顶级副本`system.actorOf()`具有相同的函数签名。

The easiest way to see the actor hierarchy in action is to simply print `ActorRef` instances. In this small experiment, we create an actor, print its reference, create a child of this actor, and print the child's reference. We start with the Hello World project, if you have not downloaded it, download the Quickstart project from the @scala[[Lightbend Tech Hub](http://developer.lightbend.com/start/?group=akka&project=akka-quickstart-scala)]@java[[Lightbend Tech Hub](http://developer.lightbend.com/start/?group=akka&project=akka-quickstart-java)].

查看Actor层次结构的最简单方法是打印`ActorRef`实例。在这个小实验中，我们创建一个Actor，打印其引用，创建该Actor的一个子Actor，并打印该子Actor的引用。我们从Hello World项目开始，如果您尚未下载，请从[Lightbend Tech Hub](http://developer.lightbend.com/start/?group=akka&project=akka-quickstart-scala)下载Quickstart项目。

In your Hello World project, navigate to the `com.lightbend.akka.sample` package and create a new @scala[Scala file called `ActorHierarchyExperiments.scala`]@java[Java file called `ActorHierarchyExperiments.java`] here. Copy and paste the code from the snippet below to this new source file. Save your file and run `sbt "runMain com.lightbend.akka.sample.ActorHierarchyExperiments"` to observe the output.

在Hello World项目中，在`com.lightbend.akka.sample`包并创建一个名为`ActorHierarchyExperiments.scala`的新文件。将下面的代码复制并粘贴到这个新的文件中。保存您的文件并运行`sbt "runMain com.lightbend.akka.sample.ActorHierarchyExperiments"`以观察输出。

Scala
:   @@snip [ActorHierarchyExperiments.scala]($code$/scala/tutorial_1/ActorHierarchyExperiments.scala) { #print-refs }

Java
:   @@snip [ActorHierarchyExperiments.java]($code$/java/jdocs/tutorial_1/ActorHierarchyExperiments.java) { #print-refs }

Note the way a message asked the first actor to do its work. We sent the message by using the parent's reference: @scala[`firstRef ! "printit"`]@java[`firstRef.tell("printit", ActorRef.noSender())`]. When the code executes, the output includes the references for the first actor and the child it created as part of the `printit` case. Your output should look similar to the following:

注意第一个Actor完成其工作的方式。我们通过其父级的引用发送消息：`firstRef ! "printit"`。当代码执行时，输出结果包含了第一个Actor的引用以及它创建的子Actor的引用。您的输出应该看起来类似于以下内容：

```
First: Actor[akka://testSystem/user/first-actor#1053618476]
Second: Actor[akka://testSystem/user/first-actor/second-actor#-1544706041]
```

Notice the structure of the references:

注意引用的结构：

* Both paths start with `akka://testSystem/`. Since all actor references are valid URLs, `akka://` is the value of the protocol field.
* 两条路径都以`akka://testSystem/`开头。由于所有Actor的引用都是有效的URL，因此`akka://`是协议字段的值。
* Next, just like on the World Wide Web, the URL identifies the system. In this example, the system is named `testSystem`, but it could be any other name. If remote communication between multiple systems is enabled, this part of the URL includes the hostname so other systems can find it on the network.
* 接下来，就像在万维网上一样，URL标识了系统。在这个例子中，系统被命名`testSystem`，但也可以是任何其他名称。如果启用了多个系统之间的远程通信，则这部分URL还包含主机名，以便其他系统可以在网络上找到它。
* Because the second actor's reference includes the path `/first-actor/`, it identifies it as a child of the first.
* 因为第二个Actor的引用包括路径`/first-actor/`，所以它将被标识为第一个Actor的子级。
* The last part of the actor reference, `#1053618476` or `#-1544706041`  is a unique identifier that you can ignore in most cases.
* Actor的最后部分，`#1053618476`或者`#-1544706041`是在大多数情况下可以被忽略的一个唯一标识符。

Now that you understand what the actor hierarchy looks like, you might be wondering: _Why do we need this hierarchy? What is it used for?_

现在你明白了Actor的层次结构，你可能会想：*为什么我们需要这个层次？它是干什么用的？*

An important role of the hierarchy is to safely manage actor lifecycles. Let's consider this next and see how that knowledge can help us write better code.

层次结构的一个重要作用是管理演员的生命周期。接下来我们来考虑一下，看看如何用这些知识来帮助我们编写更好的代码。

### Actor生命周期

Actors pop into existence when created, then later, at user requests, they are stopped. Whenever an actor is stopped, all of its children are _recursively stopped_ too.This behavior greatly simplifies resource cleanup and helps avoid resource leaks such as those caused by open sockets and files. In fact, a commonly overlooked difficulty when dealing with low-level multi-threaded code is the lifecycle management of various concurrent resources.

Actor从被创建时存在到之后用户请求时停止。当Actor停止时，其所有的子Actor都会*递归地停止*。此行为极大地简化了资源清理，并有助于避免资源泄漏，如由打开的套接字和文件造成的资源泄漏等。事实上，当处理低级多线程代码时常常被忽视的难点是各种并发资源的生命周期管理。

To stop an actor, the recommended pattern is to call @scala[`context.stop(self)`]@java[`getContext().stop(getSelf())`] inside the actor to stop itself, usually as a response to some user defined stop message or when the actor is done with its job. Stopping another actor is technically possible by calling @scala[`context.stop(actorRef)`]@java[`getContext().stop(actorRef)`], but **It is considered a bad practice to stop arbitrary actors this way**: try sending them a `PoisonPill` or custom stop message instead.

为了停止Actor，推荐的模式是在Actor内部调用`context.stop(self)`以停止自己，通常这作为对用户自定义的停止消息的响应或Actor完成其工作时的响应。理论上可以通过调用`context.stop(actorRef)`来停止另一个Actor，但**以这种方式停止任意Actor被认为是一种很不好的做法**：取而代之的是应该向他们发送一个`PoisonPill`或自定义的停止消息。

The Akka actor API exposes many lifecycle hooks that you can override in an actor implementation. The most commonly used are `preStart()` and `postStop()`.

Akka actor API公开了许多关于生命周期钩子，您可以在Actor中覆盖这些钩子。最常用的是`preStart()`和`postStop()`。

 * `preStart()` is invoked after the actor has started but before it processes its first message.
 * `preStart()` 在Actor启动之后，处理第一条消息之前被调用。
 * `postStop()` is invoked just before the actor stops. No messages are processed after this point.
 * `postStop()`在Actor停止之前被调用。在这之后不在处理消息。

Let's use the `preStart()` and `postStop()` lifecycle hooks in a simple experiment to observe the behavior when we stop an actor. First, add the following 2 actor classes to your project:

让我们用一个简单的实验来观察停止一个Actor时，`preStart()`和`postStop()`在生命周期中的行为表现。首先，将以下两个actor类添加到您的项目中：

Scala
:   @@snip [ActorHierarchyExperiments.scala]($code$/scala/tutorial_1/ActorHierarchyExperiments.scala) { #start-stop }

Java
:   @@snip [ActorHierarchyExperiments.java]($code$/java/jdocs/tutorial_1/ActorHierarchyExperiments.java) { #start-stop }

And create a 'main' class like above to start the actors and then send them a `"stop"` message:

创建一个类似上面的'main'类来启动Actor，然后向他们发送`"stop"`消息：

Scala
:   @@snip [ActorHierarchyExperiments.scala]($code$/scala/tutorial_1/ActorHierarchyExperiments.scala) { #start-stop-main }

Java
:   @@snip [ActorHierarchyExperiments.java]($code$/java/jdocs/tutorial_1/ActorHierarchyExperiments.java) { #start-stop-main }

You can again use `sbt` to start this program. The output should look like this:

你可以再次使用`sbt`来启动这个程序。输出应该如下所示：

```
first started
second started
second stopped
first stopped
```

When we stopped actor `first`, it stopped its child actor, `second`, before stopping itself. This ordering is strict, _all_ `postStop()` hooks of the children are called before the `postStop()` hook of the parent
is called.

当我们在停止Actor`first`时，它在停止自己之前，先停止了Actor`second`。这个顺序是严格的，子级的`postStop()`会在父级的`postStop()`前调用。

The @ref:[Actor Lifecycle](../actors.md#actor-lifecycle) section of the Akka reference manual provides details on the full set of lifecyle hooks.

Akka参考手册的[Actor生命周期](https://doc.akka.io/docs/akka/current/actors.html#actor-lifecycle)部分提供了有关整套生命周期的详细信息。

### 失败处理

Parents and children are connected throughout their lifecycles. Whenever an actor fails (throws an exception or an unhandled exception bubbles out from `receive`) it is temporarily suspended. As mentioned earlier, the failure information is propagated to the parent, which then decides how to handle the exception caused by the child actor. In this way, parents act as supervisors for their children. The default _supervisor strategy_ is to stop and restart the child. If you don't change the default strategy all failures result in a restart.

父母和孩子在他们的整个生命周期中都有联系。每当一个actor失败时（抛出一个异常或者一个未处理的异常从中冒出`receive`），它暂时被暂停。如前所述，故障信息被传播给父母，父母然后决定如何处理由该孩子行为引起的异常。通过这种方式，父母担任他们孩子的主管。默认的*主管策略*是停止并重新启动孩子。如果您不更改默认策略，则所有故障都会导致重新启动。

Let's observe the default strategy in a simple experiment. Add the following classes to your project, just as you did with the previous ones:

让我们在这个简单的实验中观察其默认策略。将以下类添加到您的项目中，就像之前那样：

Scala
:   @@snip [ActorHierarchyExperiments.scala]($code$/scala/tutorial_1/ActorHierarchyExperiments.scala) { #supervise }

Java
:   @@snip [ActorHierarchyExperiments.java]($code$/java/jdocs/tutorial_1/ActorHierarchyExperiments.java) { #supervise }

And run with:

运行：

Scala
:   @@snip [ActorHierarchyExperiments.scala]($code$/scala/tutorial_1/ActorHierarchyExperiments.scala) { #supervise-main }

Java
:   @@snip [ActorHierarchyExperiments.java]($code$/java/jdocs/tutorial_1/ActorHierarchyExperiments.java) { #supervise-main }

You should see output similar to the following:

您应该看到类似于以下内容的输出：

```
supervised actor started
supervised actor fails now
supervised actor stopped
supervised actor started
[ERROR] [03/29/2017 10:47:14.150] [testSystem-akka.actor.default-dispatcher-2] [akka://testSystem/user/supervising-actor/supervised-actor] I failed!
java.lang.Exception: I failed!
        at tutorial_1.SupervisedActor$$anonfun$receive$4.applyOrElse(ActorHierarchyExperiments.scala:57)
        at akka.actor.Actor$class.aroundReceive(Actor.scala:513)
        at tutorial_1.SupervisedActor.aroundReceive(ActorHierarchyExperiments.scala:47)
        at akka.actor.ActorCell.receiveMessage(ActorCell.scala:519)
        at akka.actor.ActorCell.invoke(ActorCell.scala:488)
        at akka.dispatch.Mailbox.processMailbox(Mailbox.scala:257)
        at akka.dispatch.Mailbox.run(Mailbox.scala:224)
        at akka.dispatch.Mailbox.exec(Mailbox.scala:234)
        at akka.dispatch.forkjoin.ForkJoinTask.doExec(ForkJoinTask.java:260)
        at akka.dispatch.forkjoin.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1339)
        at akka.dispatch.forkjoin.ForkJoinPool.runWorker(ForkJoinPool.java:1979)
        at akka.dispatch.forkjoin.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:107)
```

We see that after failure the supervised actor is stopped and immediately restarted. We also see a log entry reporting the exception that was handled, in this case, our test exception. In this example we used `preStart()` and `postStop()` hooks which are the default to be called after and before restarts, so we cannot distinguish from inside the actor whether it was started for the first time or restarted. This is usually the right thing to do, the purpose of the restart is to set the actor in a known-good state, which usually means a clean starting stage. **What actually happens though is that the `preRestart()` and `postRestart()` methods are called which, if not overridden, by default delegate to `postStop()` and `preStart()` respectively**. You can experiment with overriding these additional methods and see how the output changes.

我们可以看到，受监督的Actor在失败后将停止并立即重新启动。在我们这个异常测试中，我们还会看到一个日志条目，报告已处理的异常。在这个例子中，我们使用默认在重启前后调用的`preStart()`和`postStop()`，所以我们难以从内部区分Actor是第一次启动还是重启。而这通常是正确的，重新启动的目的是将Actor设置为已知状态，这通常意味着一个干净的开始阶段。**实际上是preRestart()和postRestart()方法被调用，如果没有被覆盖，默认情况下他们分别被委托给postStop()和preStart()方法**。您可以尝试重写这些附加方法并查看输出有何改变。

For the impatient, we also recommend looking into the @ref:[supervision reference page](../general/supervision.md) for more in-depth
details.

对于没有耐心的读者，我们也建议看看[监督引用页面](https://doc.akka.io/docs/akka/current/general/supervision.html)以获得更深入的细节。

# Summary
We've learned about how Akka manages actors in hierarchies where parents supervise their children and handle exceptions. We saw how to create a very simple actor and child. Next, we'll apply this knowledge to our example use case by modeling the communication necessary to get information from device actors. Later, we'll deal with how to manage the actors in groups.

我们已经了解到Akka在层级结构中如何通过父级监督子级并处理异常情况。我们看到了如何创建一个非常简单的子Actor。接下来，我们将通过建模从设备Actor中获取的信息来将这些知识应用于我们的例子中。稍后，我们将讨论如何管理组中的Actor。

