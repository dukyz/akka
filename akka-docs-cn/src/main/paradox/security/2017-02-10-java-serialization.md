# Java序列化，在Akka 2.4.17中修复

### Date

10 Feburary 2017

### Description of Vulnerability

### 漏洞描述

An attacker that can connect to an `ActorSystem` exposed via Akka Remote over TCP can gain remote code execution capabilities in the context of the JVM process that runs the ActorSystem if:

在以下情况，攻击者可以借助Akka Remote机制，通过TCP连接到ActorSystem，并在JVM上下文中获取远程代码的执行能力。

 * `JavaSerializer` is enabled (default in Akka 2.4.x)
 * `JavaSerializer` 已启用（默认在Akka 2.4.x中）
 * and TLS is disabled *or* TLS is enabled with `akka.remote.netty.ssl.security.require-mutual-authentication = false`
    (which is still the default in Akka 2.4.x)
 * 并且TLS被禁用*或*启用TLS `akka.remote.netty.ssl.security.require-mutual-authentication = false`（这仍然是Akka 2.4.x中的默认设置）
 * or if TLS is enabled with mutual authentication and the authentication keys of a host that is allowed to connect have been compromised, an attacker gained access to a valid certificate (e.g. by compromising a node with certificates issued by the same internal PKI tree to get access of the certificate)
 * 或者如果使用相互身份验证启用了TLS，并且允许连接的主机的身份验证密钥已遭到入侵，则攻击者可以访问有效的证书（例如通过使用同一内部PKI树颁发的证书对节点进行攻击以获取访问权限的证书）
 * regardless of whether `untrusted` mode is enabled or not
 * 不管`untrusted`模式是否启用

Java deserialization is [known to be vulnerable](https://community.hpe.com/t5/Security-Research/The-perils-of-Java-deserialization/ba-p/6838995) to attacks when attacker can provide arbitrary types.

Java反序列化是[已知的漏洞](https://community.hpe.com/t5/Security-Research/The-perils-of-Java-deserialization/ba-p/6838995) ，攻击者可以提供任意类型。

Akka Remoting uses Java serialiser as default configuration which makes it vulnerable in its default form. The documentation of how to disable Java serializer was not complete. The documentation of how to enable mutual authentication was missing (only described in reference.conf).

Akka Remoting使用Java串行器作为默认配置时，使其以默认形式存在漏洞。有关如何禁用Java序列化程序的文档尚未完成。缺少如何启用相互认证的文档（只在reference.conf中描述）。

To protect against such attacks the system should be updated to Akka *2.4.17* or later and be configured with 
@ref:[disabled Java serializer](../remoting.md#disable-java-serializer). Additional protection can be achieved when running in an 
untrusted network by enabling @ref:[TLS with mutual authentication](../remoting.md#remote-tls).

为防止此类攻击，系统应更新至Akka *2.4.17*或更高版本，并配置[禁用的Java序列化程序](https://doc.akka.io/docs/akka/current/remoting.html#disable-java-serializer)。通过启用[TLS和相互身份验证](https://doc.akka.io/docs/akka/current/remoting.html#remote-tls)，可在不受信任的网络中运行时实现额外的保护。

Please subscribe to the [akka-security](https://groups.google.com/forum/#!forum/akka-security) mailing list to be notified promptly about future security issues.

请订阅[akka-security](https://groups.google.com/forum/#!forum/akka-security)邮件，及时获取相关的安全问题。

### Severity

### 严重

The [CVSS](https://en.wikipedia.org/wiki/CVSS) score of this vulnerability is 6.8 (Medium), based on vector 

该漏洞的[CVSS](https://en.wikipedia.org/wiki/CVSS)得分为6.8（中），基于矢量

[AV:A/AC:M/Au:N/C:C/I:C/A:C/E:F/RL:TF/RC:C](https://nvd.nist.gov/cvss.cfm?calculator&version=2&vector=\(AV:A/AC:M/Au:N/C:C/I:C/A:C/E:F/RL:TF/RC:C\)).

Rationale for the score:

评分理由:

 * AV:A - Best practice is that Akka remoting nodes should only be accessible from the adjacent network, so in good setups, this will be adjacent.
 * AV:A - 最佳做法是只能从相邻网络访问Akka远程节点，因此在良好的设置中，这将相邻。
 * AC:M - Any one in the adjacent network can launch the attack with non-special access privileges.
 * AC:M - 邻近网络中的任何一个人都可以以非特殊访问权限发起攻击。
 * C:C, I:C, A:C - Remote Code Execution vulnerabilities are by definition CIA:C.
 * C:C，I:C，A:C - 远程执行代码漏洞按照定义CIA:C。

### Affected Versions

### 受影响的版本

 * Akka *2.4.16* and prior
 * Akka *2.5-M1* (milestone not intended for production)

### Fixed Versions

### 修正版本

We have prepared patches for the affected versions, and have released the following versions which resolve the issue: 

我们已经为受影响的版本准备了补丁程序，并发布了解决此问题的以下版本：

 * Akka *2.4.17* (Scala 2.11, 2.12)

Binary and source compatibility has been maintained for the patched releases so the upgrade procedure is as simple as changing the library dependency.

二进制和源代码兼容性已针对补丁版本进行维护，因此升级过程与更改库依赖关系一样简单。

It will also be fixed in 2.5-M2 or 2.5.0-RC1.

它也将固定在2.5-M2或2.5.0-RC1中。

### Acknowledgements

### 致谢

We would like to thank Alvaro Munoz at Hewlett Packard Enterprise Security & Adrian Bravo at Workday for their thorough investigation and bringing this issue to our attention.

我们要感谢Workday的Hewlett Packard Enterprise Security和Adrian Bravo的Alvaro Munoz对该问题的彻底调查，并使得大家能够正视这个问题。