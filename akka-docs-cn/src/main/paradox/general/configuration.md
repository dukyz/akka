# 配置

You can start using Akka without defining any configuration, since sensible default values are provided. Later on you might need to amend the settings to change the default behavior or adapt for specific runtime environments. Typical examples of settings that you might amend:

你可以在不定义任何配置的情况下开始使用Akka，因为Akka提供了明智的默认值。稍后你可能需要修改设置以适应特定的运行时环境。你可能会修改的设置如下：

 * log level and logger backend
 * 日志级别和记录后端
 * enable remoting
 * 启用远程处理
 * message serializers
 * 消息序列化
 * definition of routers
 * 路由定义
 * tuning of dispatchers
 * 调度员调整

Akka uses the [Typesafe Config Library](https://github.com/typesafehub/config), which might also be a good choice for the configuration of your own application or library built with or without Akka. This library is implemented in Java with no external dependencies; you should have a look at its documentation (in particular about [ConfigFactory](http://typesafehub.github.io/config/v1.2.0/com/typesafe/config/ConfigFactory.html)), which is only summarized in the following.

Akka使用[Typesafe Config Library](https://github.com/typesafehub/config)，这对于使用或不使用Akka构建自己的应用程序来说都是一个很好的选择。这个库是用Java实现的，没有外部依赖; 你应该看看它的文档（特别是关于[ConfigFactory](http://typesafehub.github.io/config/v1.2.0/com/typesafe/config/ConfigFactory.html)），这些只在下面进行了总结。

@@@ warning

If you use Akka from the Scala REPL from the 2.9.x series, and you do not provide your own ClassLoader to the ActorSystem, start the REPL with "-Yrepl-sync" to work around a deficiency in the REPLs provided Context ClassLoader.

如果你在Scala REPL 2.9.x系列中使用Akka，并且没有将自己的ClassLoader提供给ActorSystem，请使用“-Yrepl-sync”参数启动REPL以解决REPLs提供的Context ClassLoader中的缺陷。

@@@

## 从哪里读取配置

All configuration for Akka is held within instances of `ActorSystem`, or put differently, as viewed from the outside, `ActorSystem` is the only consumer of configuration information. While constructing an actor system, you can either pass in a `Config` object or not, where the second case is equivalent to passing `ConfigFactory.load()` (with the right class loader). This means roughly that the default is to parse all `application.conf`, `application.json` and `application.properties` found at the root of the class path—please refer to the aforementioned documentation for details. The actor system then merges in all `reference.conf` resources found at the root of the class path to form the fallback configuration, i.e. it internally uses

从外部来看，Akka的所有配置都保存在不同的`ActorSystem`实例中，也就是说`ActorSystem`是配置信息的唯一消费者。在构造一个Actor系统的时候，你可以传递一个`Config`对象或不传递，第二种情况相当于传递`ConfigFactory.load()`（使用正确的类加载器）。这大致意味着默认解析所有的在根路径下找到的`application.conf`，`application.json`和`application.properties` - 请参阅上述文档以获取详细信息。然后Actor系统将之与根路径下的`reference.conf`中的资源合并以形成备用配置，即它在内部使用

```scala
appConfig.withFallback(ConfigFactory.defaultReference(classLoader))
```

The philosophy is that code never contains default values, but instead relies upon their presence in the `reference.conf` supplied with the library in question.

设计哲学是不在代码中包含默认值，而是依赖它们在`reference.conf`中提供的默认值。

Highest precedence is given to overrides given as system properties, see [the HOCON specification](https://github.com/typesafehub/config/blob/master/HOCON.md) (near the
bottom). Also noteworthy is that the application configuration—which defaults to `application`—may be overridden using the `config.resource` property (there are more, please refer to the [Config docs](https://github.com/typesafehub/config/blob/master/README.md)).

[HOCON规范](https://github.com/typesafehub/config/blob/master/HOCON.md)（底部附近）给出了覆盖作为系统属性的最高优先级。另外值得注意的是应用程序配置 - 默认为`application`- 可以使用`config.resource`属性覆盖（更多请参阅[配置文档](https://github.com/typesafehub/config/blob/master/README.md)）。

@@@ note

If you are writing an Akka application, keep your configuration in `application.conf` at the root of the class path. If you are writing an Akka-based library, keep its configuration in `reference.conf` at the root
of the JAR file.

如果你正在编写Akka应用程序，请将你的配置保存在根路径下的`application.conf`中。如果你正在编写基于Akka的类库，请将其配置保存在JAR文件的根路径下的`reference.conf`中。

@@@

## 什么时候使用JarJar, OneJar, Assembly 或者任意jar-bundler

@@@ warning

Akka's configuration approach relies heavily on the notion of every module/jar having its own reference.conf file, all of these will be discovered by the configuration and loaded. Unfortunately this also means that if you put/merge multiple jars into the same jar, you need to merge all the reference.confs as well. Otherwise all defaults will be lost and Akka will not function.

Akka的配置方法重度依赖于每个模块/ jar都具有自己的reference.conf文件这个概念，所有这些都将由配置发现并加载。但不幸的是，这也意味着如果你将多个jar放入/合并到同一个jar中，你需要合并所有的reference.conf。否则，所有默认设置都将丢失，Akka将无法使用。

@@@

If you are using Maven to package your application, you can also make use of the [Apache Maven Shade Plugin](http://maven.apache.org/plugins/maven-shade-plugin) support for [Resource Transformers](http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#AppendingTransformer) to merge all the reference.confs on the build classpath into one. 

如果你使用Maven来打包应用程序，那么你还可以使用[Apache Maven Shade Plugin](http://maven.apache.org/plugins/maven-shade-plugin)的[资源转换器](http://maven.apache.org/plugins/maven-shade-plugin/examples/resource-transformers.html#AppendingTransformer)将构建路径中的所有reference.conf合并为一个。

The plugin configuration might look like this:

插件配置可能如下所示：

```
<plugin>
 <groupId>org.apache.maven.plugins</groupId>
 <artifactId>maven-shade-plugin</artifactId>
 <version>1.5</version>
 <executions>
  <execution>
   <phase>package</phase>
   <goals>
    <goal>shade</goal>
   </goals>
   <configuration>
    <shadedArtifactAttached>true</shadedArtifactAttached>
    <shadedClassifierName>allinone</shadedClassifierName>
    <artifactSet>
     <includes>
      <include>*:*</include>
     </includes>
    </artifactSet>
    <transformers>
      <transformer
       implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
       <resource>reference.conf</resource>
      </transformer>
      <transformer
       implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
       <manifestEntries>
        <Main-Class>akka.Main</Main-Class>
       </manifestEntries>
      </transformer>
    </transformers>
   </configuration>
  </execution>
 </executions>
</plugin>
```

## 自定义application.conf

A custom `application.conf` might look like this:

自定义`application.conf`可能如下所示：

```
# In this file you can override any option defined in the reference files.
# Copy in parts of the reference files and modify as you please.

akka {

  # Loggers to register at boot time (akka.event.Logging$DefaultLogger logs
  # to STDOUT)
  loggers = ["akka.event.slf4j.Slf4jLogger"]

  # Log level used by the configured loggers (see "loggers") as soon
  # as they have been started; before that, see "stdout-loglevel"
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  loglevel = "DEBUG"

  # Log level for the very basic logger activated during ActorSystem startup.
  # This logger prints the log messages to stdout (System.out).
  # Options: OFF, ERROR, WARNING, INFO, DEBUG
  stdout-loglevel = "DEBUG"

  # Filter of log events that is used by the LoggingAdapter before
  # publishing log events to the eventStream.
  logging-filter = "akka.event.slf4j.Slf4jLoggingFilter"

  actor {
    provider = "cluster"

    default-dispatcher {
      # Throughput for default Dispatcher, set to 1 for as fair as possible
      throughput = 10
    }
  }

  remote {
    # The port clients should connect to. Default is 2552.
    netty.tcp.port = 4711
  }
}
```

## 包括文件（Including files）

Sometimes it can be useful to include another configuration file, for example if you have one `application.conf` with all environment independent settings and then override some settings for specific environments.

有时包含其他配置文件会很有用，例如，如果你有一个`application.conf`包含所有与环境无关的配置，然后覆盖某型特定环境中的设置。

Specifying system property with `-Dconfig.resource=/dev.conf` will load the `dev.conf` file, which includes the `application.conf`

指定系统属性`-Dconfig.resource=/dev.conf`将加载`dev.conf`文件，其中包括`application.conf`

### dev.conf

```
include "application"

akka {
  loglevel = "DEBUG"
}
```

More advanced include and substitution mechanisms are explained in the [HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md) specification.

更高级的包含和替换机制在[HOCON](https://github.com/typesafehub/config/blob/master/HOCON.md)规范中进行了解释。

<a id="dakka-log-config-on-start"></a>
## 日志配置

If the system or config property `akka.log-config-on-start` is set to `on`, then the complete configuration is logged at INFO level when the actor system is started. This is useful when you are uncertain of what configuration is used.

如果系统或配置属性`akka.log-config-on-start`被设置为`on`，则当Actor系统启动时，将在INFO日志级别记录完整的配置信息。当你不确定使用了什么配置时，这很有用。

If in doubt, you can also easily and nicely inspect configuration objects before or after using them to construct an actor system:

如果有疑问，你还可以在使用它们构建Actor系统之前或之后检查配置对象：

@@@vars
```
Welcome to Scala $scala.binary_version$ (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0).
Type in expressions to have them evaluated.
Type :help for more information.

scala> import com.typesafe.config._
import com.typesafe.config._

scala> ConfigFactory.parseString("a.b=12")
res0: com.typesafe.config.Config = Config(SimpleConfigObject({"a" : {"b" : 12}}))

scala> res0.root.render
res1: java.lang.String =
{
    # String: 1
    "a" : {
        # String: 1
        "b" : 12
    }
}
```
@@@

The comments preceding every item give detailed information about the origin of the setting (file & line number) plus possible comments which were present, e.g. in the reference configuration. The settings as merged with the reference and parsed by the actor system can be displayed like this:

每个项目前面的注释提供了关于设置来源（文件和行号）以及可能存在的注释的详细信息，例如在参考配置中。与参考文件合并并由Actor系统解析的设置可以如下显示：

```java
final ActorSystem system = ActorSystem.create();
System.out.println(system.settings());
// this is a shortcut for system.settings().config().root().render()
```

## 关于ClassLoaders的一个词（A Word About ClassLoaders）

In several places of the configuration file it is possible to specify the fully-qualified class name of something to be instantiated by Akka. This is done using Java reflection, which in turn uses a `ClassLoader`. Getting
the right one in challenging environments like application containers or OSGi bundles is not always trivial, the current approach of Akka is that each `ActorSystem` implementation stores the current thread’s context class loader (if available, otherwise just its own loader as in `this.getClass.getClassLoader`) and uses that for all reflective accesses. This implies that putting Akka on the boot class path will yield 
`NullPointerException` from strange places: this is simply not supported.

在配置文件的一些地方，可以指定由Akka实例化的完全限定类名。这是通过使用Java反射来完成的，而Java反射又使用了`ClassLoader`。在应用程序容器或OSGi等具有挑战性的环境中正确使用是有难度的，目前Akka的方法是每个`ActorSystem`都存有当前线程上下文的类加载器（如果其可用，否则只是它自己的加载器`this.getClass.getClassLoader`）并将其用于所有反射访问。这意味着把Akka放在引导类路径上会在奇怪的地方产生`NullPointerException`：简单说就是这不被支持。

## 应用程序特定设置

The configuration can also be used for application specific settings.A good practice is to place those settings in an @ref:[Extension](../extending-akka.md#extending-akka-settings).

该配置也可以用于应用程序特定的设置。一个好的做法是将这些设置放在[扩展](https://doc.akka.io/docs/akka/current/extending-akka.html#extending-akka-settings)中。

## 配置多个ActorSystem

If you have more than one `ActorSystem` (or you're writing a library and have an `ActorSystem` that may be separate from the application's) you may want to separate the configuration for each system.

如果你有多个`ActorSystem`（或者你正在编写一个库，并且有`ActorSystem`可能与应用程序分离），你可能需要分离每个系统的配置。

Given that `ConfigFactory.load()` merges all resources with matching name from the whole class path, it is easiest to utilize that functionality and differentiate actor systems within the hierarchy of the configuration:

假定`ConfigFactory.load()`从整个类路径合并具有匹配名称的所有资源，最简单的方法是使用该功能并区分出配置结构中的Actor系统：

```
myapp1 {
  akka.loglevel = "WARNING"
  my.own.setting = 43
}
myapp2 {
  akka.loglevel = "ERROR"
  app2.setting = "appname"
}
my.own.setting = 42
my.other.setting = "hello"
```

```scala
val config = ConfigFactory.load()
val app1 = ActorSystem("MyApp1", config.getConfig("myapp1").withFallback(config))
val app2 = ActorSystem("MyApp2",
  config.getConfig("myapp2").withOnlyPath("akka").withFallback(config))
```

These two samples demonstrate different variations of the “lift-a-subtree” trick: in the first case, the configuration accessible from within the actor system is this

这两个示例演示了“lift-a-subtree”技巧的不同变体：在第一种情况下，可以从Actor系统内访问的配置如下

```ruby
akka.loglevel = "WARNING"
my.own.setting = 43
my.other.setting = "hello"
// plus myapp1 and myapp2 subtrees
```

while in the second one, only the “akka” subtree is lifted, with the following result

而在第二种情况下，只有满足“akka”路径下的属性被选中，结果如下

```ruby
akka.loglevel = "ERROR"
my.own.setting = 42
my.other.setting = "hello"
// plus myapp1 and myapp2 subtrees
```

@@@ note

The configuration library is really powerful, explaining all features exceeds the scope affordable here. In particular not covered are how to include other configuration files within other files (see a small example at [Including files](#including-files)) and copying parts of the configuration tree by way of path substitutions.

配置库非常强大，能解释的功能远超于此。特别是未介绍的如何包含其他文件中的配置文件（请参阅[包含文件](https://doc.akka.io/docs/akka/current/general/configuration.html#including-files)中的小示例），并通过路径替换的方式复制配置树中的某些部分。

@@@

You may also specify and parse the configuration programmatically in other ways when instantiating the `ActorSystem`.

你还可以通过其他编程的方式在实例化`ActorSystem`时指定分析配置信息。

@@snip [ConfigDocSpec.scala]($code$/scala/docs/config/ConfigDocSpec.scala) { #imports #custom-config }

## 从自定义位置读取配置

You can replace or supplement `application.conf` either in code or using system properties.

你可以在代码中或使用系统属性来对`application.conf`进行替换或补充。

If you're using `ConfigFactory.load()` (which Akka does by default) you can replace `application.conf` by defining `-Dconfig.resource=whatever`, `-Dconfig.file=whatever`, or `-Dconfig.url=whatever`.

如果你使用`ConfigFactory.load()`（Akka默认操作），你可以通过定义`-Dconfig.resource=whatever`，`-Dconfig.file=whatever`或`-Dconfig.url=whatever`来替换`application.conf`。

From inside your replacement file specified with `-Dconfig.resource` and friends, you can `include
"application"` if you still want to use `application.{conf,json,properties}` as well.  Settings
specified before `include "application"` would be overridden by the included file, while those after would override the included file.

在你指定的`-Dconfig.resource`替换文件中，如果你仍想使用`application.{conf,json,properties}`你仍然可以`include "application"`。在`include "application"`之前配置的信息将被包含的文件覆盖，而后面配置的信息将覆盖包含的文件。

In code, there are many customization options.

在代码中，有很多自定义选项。

There are several overloads of `ConfigFactory.load()`; these allow you to specify something to be sandwiched between system properties (which override) and the defaults (from `reference.conf`), replacing the usual `application.{conf,json,properties}` and replacing `-Dconfig.file` and friends.

`ConfigFactory.load()`有几项重载; 允许你指定要夹在系统属性（将覆盖之）和默认值（来自`reference.conf`）之间的内容，它们会替换常规的`application.{conf,json,properties}`和`-Dconfig.file`等。

The simplest variant of `ConfigFactory.load()` takes a resource basename (instead of `application`); `myname.conf`,`myname.json`, and `myname.properties` would then be used instead of `application.{conf,json,properties}`.

`ConfigFactory.load()`最简单的方式是采用资源基本名称（不是`application`）; `myname.conf`，`myname.json`和`myname.properties`将替代使用`application.{conf,json,properties}`。

The most flexible variant takes a `Config` object, which you can load using any method in `ConfigFactory`.  For example you could put a config string in code using `ConfigFactory.parseString()` or you could make a map and `ConfigFactory.parseMap()`, or you could load a file.

最灵活的方式是一个`Config`对象，你可以使用任何`ConfigFactory`中的方法加载它。例如，你可以在代码中利用`ConfigFactory.parseString()`方法使用配置字符串，或者你可以利用`ConfigFactory.parseMap()`方法创建一个Map，或者你可以加载一个文件。

You can also combine your custom config with the usual config, that might look like:

你也可以将你自定义的配置与常规配置相结合，可能如下所示：

@@snip [ConfigDoc.java]($code$/java/jdocs/config/ConfigDoc.java) { #java-custom-config }

When working with `Config` objects, keep in mind that there are three "layers" in the cake:

在你使用`Config`对象时，请记住有三个层次：

 * `ConfigFactory.defaultOverrides()` (system properties)
 * `ConfigFactory.defaultOverrides()` （系统属性）
 * the app's settings
 * 应用程序的设置
 * `ConfigFactory.defaultReference()` (reference.conf)
 * `ConfigFactory.defaultReference()` （reference.conf）

The normal goal is to customize the middle layer while leaving the other two alone.

通常是定制中间层，而不动另外两层。

 * `ConfigFactory.load()` loads the whole stack 
 * `ConfigFactory.load()` 加载整个栈
 * the overloads of `ConfigFactory.load()` let you specify a different middle layer
 * 重载`ConfigFactory.load()`可以让你指定一个不同的中间层
 * the `ConfigFactory.parse()` variations load single files or resources
 * `ConfigFactory.parse()`这样的方式会加载单个文件或资源

To stack two layers, use `override.withFallback(fallback)`; try to keep system props (`defaultOverrides()`) on top and `reference.conf` (`defaultReference()`) on the bottom.

要加入其他两层，请使用`override.withFallback(fallback)`方法; 将系统属性（`defaultOverrides()`）保留在顶部，并将`reference.conf`（`defaultReference()`）放在底部。

Do keep in mind, you can often just add another `include ` statement in `application.conf` rather than writing code. Includes at the top of `application.conf` will be overridden by the rest of  `application.conf`, while those at the bottom will override the earlier stuff.

请记住，你可以在`application.conf`中添加其他`include`语句而不用编写代码。包含在顶部的`application.conf`将被`application.conf`其余部分覆盖，而底部的include将覆盖较前的内容。

## Actor部署配置

Deployment settings for specific actors can be defined in the `akka.actor.deployment` section of the configuration. In the deployment section it is possible to define things like dispatcher, mailbox, router settings, and remote deployment. Configuration of these features are described in the chapters detailing corresponding topics. An example may look like this:

特定Actor的部署设置可以在`akka.actor.deployment`部分中定义。在部署部分中，可以定义诸如调度程序，邮箱，路由器设置和远程部署之类的内容。这些功能的配置会在相应的主题中详细说明。一个例子可能如下所示：

@@snip [ConfigDocSpec.scala]($code$/scala/docs/config/ConfigDocSpec.scala) { #deployment-section }

@@@ note

The deployment section for a specific actor is identified by the path of the actor relative to `/user`.

特定Actor的部署部分按照Actor相对`/user`的路径来定义。

@@@

You can use asterisks as wildcard matches for the actor path sections, so you could specify:
`/*/sampleActor` and that would match all `sampleActor` on that level in the hierarchy. In addition, please note:

你可以使用星号作为通配符匹配Actor路径部分，因此你可以指定：`/*/sampleActor`，它能够匹配在层次结构中目标层级的所有`sampleActor`匹配项。另外请注意：



 * you can also use wildcards in the last position to match all actors at a certain level: `/someParent/*`
 * 你也可以在最后的位置使用通配符来匹配某个级别的所有Actor： `/someParent/*`
 * you can use double-wildcards in the last position to match all child actors and their children recursively: `/someParent/**`
 * 你可以在最后一个位置使用双通配符来递归匹配所有的子Actor以及他们的子级(**确定是从子级开始，所有子级和后代么？？**)： `/someParent/**`
 * non-wildcard matches always have higher priority to match than wildcards, and single wildcard matches have higher priority than double-wildcards, so: `/foo/bar` is considered **more specific** than `/foo/*`, which is considered **more specific** than `/foo/**`. Only the highest priority match is used
 * 匹配优先级为非通配符匹配 > 单通配符匹配 > 双通配符匹配，这样：`/foo/bar`  > `/foo/*`  > `/foo/**`。匹配只按最高优先级匹配。
 * wildcards **cannot** be used to partially match section, like this: `/foo*/bar`, `/f*o/bar` etc.
 * 通配符**不能**用于部分匹配，如：`/foo*/bar`，`/f*o/bar`等

@@@ note

Double-wildcards can only be placed in the last position.

双通配符只能放在最后一个位置。

@@@

## Listing of the Reference Configuration

Each Akka module has a reference configuration file with the default values.

每个Akka模块都有一个具有默认值的参考配置文件。

<a id="config-akka-actor"></a>
### akka-actor

@@snip [reference.conf]($akka$/akka-actor/src/main/resources/reference.conf)

<a id="config-akka-agent"></a>
### akka-agent

@@snip [reference.conf]($akka$/akka-agent/src/main/resources/reference.conf)

<a id="config-akka-camel"></a>
### akka-camel

@@snip [reference.conf]($akka$/akka-camel/src/main/resources/reference.conf)

<a id="config-akka-cluster"></a>
### akka-cluster

@@snip [reference.conf]($akka$/akka-cluster/src/main/resources/reference.conf)

<a id="config-akka-multi-node-testkit"></a>
### akka-multi-node-testkit

@@snip [reference.conf]($akka$/akka-multi-node-testkit/src/main/resources/reference.conf)

<a id="config-akka-persistence"></a>
### akka-persistence

@@snip [reference.conf]($akka$/akka-persistence/src/main/resources/reference.conf)

<a id="config-akka-remote"></a>
### akka-remote

@@snip [reference.conf]($akka$/akka-remote/src/main/resources/reference.conf) { #shared #classic type=none }

<a id="config-akka-remote-artery"></a>
### akka-remote (artery)

@@snip [reference.conf]($akka$/akka-remote/src/main/resources/reference.conf) { #shared #artery type=none }

<a id="config-akka-testkit"></a>
### akka-testkit

@@snip [reference.conf]($akka$/akka-testkit/src/main/resources/reference.conf)

<a id="config-cluster-metrics"></a>
### akka-cluster-metrics

@@snip [reference.conf]($akka$/akka-cluster-metrics/src/main/resources/reference.conf)

<a id="config-cluster-tools"></a>
### akka-cluster-tools

@@snip [reference.conf]($akka$/akka-cluster-tools/src/main/resources/reference.conf)

<a id="config-cluster-sharding"></a>
### akka-cluster-sharding

@@snip [reference.conf]($akka$/akka-cluster-sharding/src/main/resources/reference.conf)

<a id="config-distributed-data"></a>
### akka-distributed-data

@@snip [reference.conf]($akka$/akka-distributed-data/src/main/resources/reference.conf)
