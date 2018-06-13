# 示例介绍

When writing prose, the hardest part is often composing the first few sentences. There is a similar "blank canvas" feeling when starting to build an Akka system. You might wonder: Which should be the first actor? Where should it live? What should it do?Fortunately &#8212; unlike with prose &#8212; established best practices can guide us through these initial steps. In the remainder of this guide, we examine the core logic of a simple Akka application to introduce you to actors and show you how to formulate solutions with them. The example demonstrates common patterns that will help you kickstart your Akka projects.

写散文时，最难的部分往往是撰写前几句话。开始构建Akka系统时有类似的“空白画布”感觉。你可能会想：哪个应该是第一个Actor？它应该放在哪？它应该做些什么？幸运的是与散文不同 - 既定的最佳实践可以指导我们完成这些初始步骤。在本指南的其他部分中，我们将展示一个Akka应用程序的核心逻辑，借此将你带入Actor世界并向您展示如何使用它们制定解决方案。该示例演示了可以帮助您启动Akka项目的常见模式。

## 先决条件
You should have already followed the instructions in the @scala[[Akka Quickstart with Scala guide](http://developer.lightbend.com/guides/akka-quickstart-scala/)] @java[[Akka Quickstart with Java guide](http://developer.lightbend.com/guides/akka-quickstart-java/)] to download and run the Hello World example. You will use this as a seed project and add the functionality described in this tutorial.

您应该已经按照[Akka Quickstart with Scala guide](http://developer.lightbend.com/guides/akka-quickstart-scala/) 和[Akka Quickstart with Java guide](http://developer.lightbend.com/guides/akka-quickstart-java/)的说明下载并运行Hello World示例。使用它作为种子模板并添加本教程中描述的功能。

## IoT示例用例

In this tutorial, we'll use Akka to build out part of an Internet of Things (IoT) system that reports data from sensor devices installed in customers' homes. The example focuses on temperature readings. The target use case simply allows customers to log in and view the last reported temperature from different areas of their homes. You can imagine that such sensors could also collect relative humidity or other interesting data and an application would likely support reading and changing device configuration, maybe even alerting home owners when sensor state falls outside of a particular range.

在本教程中，我们将使用Akka构建物联网（IoT）系统的一部分，该系统报告了安装在客户家中的传感器设备的数据。该示例着重于温度读数。目标用例允许用户登录并查看他们家中不同区域统计的最后温度。你可以想象，这种传感器也可以收集相对湿度或其他有趣的数据，并且应用程序可能会支持读取和更改设备配置，甚至可能在传感器状态超出特定范围时提醒家庭主人。

In a real system, the application would be exposed to customers through a mobile app or browser. This guide concentrates only on the core logic for storing temperatures that would be called over a network protocol, such as HTTP. It also includes writing tests to help you get comfortable and proficient with testing actors.

在真实系统中，应用程序通过移动APP或浏览器的方式呈现给客户。本指南只是关注于存储温度的核心逻辑，这可以通过网络协议(如HTTP)调用。本指南还包括编写测试，以帮助您熟练的对Actor进行测试。

The tutorial application consists of two main components:

教程应用程序由两个主要组件组成：

 * **Device data collection:** &#8212; maintains a local representation of the remote devices. Multiple sensor devices for a home are organized into one device group.
 * **设备数据收集：** - 维护远程设备的本地表示。家中的多个传感器设备将被组织到一个设备组中。
 * **User dashboard:** &#8212; periodically collects data from the devices for a logged in user's home and presents the results as a report.
 * **用户仪表板：** - 定期从设备中收集登录用户的家中数据，并将结果呈现为报告。

The following diagram illustrates the example application architecture. Since we are interested in the state of each sensor device, we will model devices as actors. The running application will create as many instances of device actors and device groups as necessary.

下图解释了示例程序的体系结构。由于我们对每个传感器设备的状态感兴趣，因此我们将设备作为Actor进行建模。运行的应用程序将按需创建尽可能多的设备Actor和设备组实例。

![box diagram of the architecture](diagrams/arch_boxes_diagram.png)

## 你在本教程中能学到什么

This tutorial introduces and illustrates:

本教程介绍并说明：

* The actor hierarchy and how it influences actor behavior
* Actor的层次以及它是如何影响Actor的行为的
* How to choose the right granularity for actors
* 如何为Actor选择正确的粒度
* How to define protocols as messages
* 如何将协议定义为消息
* Typical conversational styles
* 典型的对话风格


Let's get started by learning more about actors.

让我们开始更多地了解Actor。


