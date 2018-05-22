---
title: Tomcat 服务器发布启动缓慢
date: 2018-05-23 05:13:13
tags: 大坑系列
categories: 我踩过的大坑
---

## Tomcat 服务器发布启动缓慢

楼主的开发经历中，第一次遇到这个问题。Tomcat 服务器发布启动缓慢

本地项目启动多次，没有任何报错，正常启动，接口正常调用，一切正常！！！然后作者将其发布到`linux`服务器上（自己编写的脚本），启动无任何报错。在然后调用服务器接口，奇怪的一幕发生了，接口调用失败，`nginx 504`，就觉得自己写的脚本有问题，经过一行行的查看，确定无误，在此发布。还是`nginx 504`，这时作者恼火了，就一直调用接口，5分钟过去了，突然发现接口可以正常返回数据。再调用其它接口，也都正确返回，这时楼主再重新执行脚本发布，立马调用接口，`nginx 504`又再次出现了，x 分钟过去了，有突然好了。然后就把服务器日志清理了，重新发布。查看日志，发现tomcat 卡在Spring 启动哪里。

```
Initializing Spring FrameworkServlet 'spring'
```

第一次遇到，所以直接百度Tomcat 卡在 Initializing Spring FrameworkServlet 'spring'，发现好多都是更改版本都是说更换jdk，修改数据库连接池之类的。哇，全是瓜皮。此刻我心里就一万句mmp啊。问了一些之前的同事朋友，也说没有遇到过这种问题。那我想，这个问题被我遇到了，肯定是有原因的，肯定其他人也遇到过，楼主不断的找，终于不小心掉到一片 Tomcat 8熵池阻塞变慢。我就按着文章来走，修改服务器的一些环境配置。重新启动脚本发布成功。立马调用接口，成功了！！！



#### Tomcat 8熵池阻塞变慢原因

Tomcat 7/8都使用org.apache.catalina.util.SessionIdGeneratorBase.createSecureRandom类产生安全随机类SecureRandom的实例作为会话ID，接近6分钟。

SHA1PRNG算法是基于SHA-1算法实现且保密性较强的伪随机数生成器。 

在SHA1PRNG中，有一个种子产生器，它根据配置执行各种操作。

1）如果Java.security.egd属性或securerandom.source属性指定的是”file:/dev/random”或”file:/dev/urandom”，那么JVM会使用本地种子产生器NativeSeedGenerator，它会调用super()方法，即调用SeedGenerator.URLSeedGenerator(/dev/random)方法进行初始化。

2）如果java.security.egd属性或securerandom.source属性指定的是其它已存在的URL，那么会调用SeedGenerator.URLSeedGenerator(url)方法进行初始化。

这就是为什么我们设置值为”file:///dev/urandom”或者值为”file:/./dev/random”都会起作用的原因。

在这个实现中，产生器会评估熵池（entropy pool）中的噪声数量。随机数是从熵池中进行创建的。当读操作时，/dev/random设备会只返回熵池中噪声的随机字节。/dev/random非常适合那些需要非常高质量随机性的场景，比如一次性的支付或生成密钥的场景。

当熵池为空时，来自/dev/random的读操作将被阻塞，直到熵池收集到足够的环境噪声数据。这么做的目的是成为一个密码安全的伪随机数发生器，熵池要有尽可能大的输出。对于生成高质量的加密密钥或者是需要长期保护的场景，一定要这么做。

那么什么是环境噪声？

随机数产生器会手机来自设备驱动器和其它源的环境噪声数据，并放入熵池中。产生器会评估熵池中的噪声数据的数量。当熵池为空时，这个噪声数据的收集是比较花时间的。这就意味着，Tomcat在生产环境中使用熵池时，会被阻塞较长的时间。



#### 有两种解决办法：

1）在Tomcat环境中解决
可以通过配置JRE使用非阻塞的Entropy Source。
在catalina.sh中加入这么一行：-Djava.security.egd=file:/dev/./urandom 即可。
加入后再启动Tomcat。

2）在JVM环境中解决
打开$JAVA_PATH/jre/lib/security/java.security这个文件，找到下面的内容：

```
securerandom.source=file:/dev/urandom
替换成
securerandom.source=file:/dev/./urandom
```

