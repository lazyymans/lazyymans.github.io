---
title: Spring Cloud微服务架构（一）高可用服务注册与发现
date: 2018-05-07 17:02:25
categories: Spring Cloud
tags: Eureka
---

### 创建“服务注册中心”

创建一个基础的Spring Boot EurekaServer项目，在pom.xml 中引入需要的依赖

​	注意：在创建Spring Boot 项目时，版本的选择。本次系列文章均采用最新版本的2.0.1[版本对应关系](https://blog.csdn.net/stloven5/article/details/78285541)

​		为什么要在这里说明一下，因为不同的版本，你写的pom依赖可能不同。如下面的 服务注册依赖，

​		在Spring Boot 1.5.x 版本中  spring-cloud-starter-eureka-server 是这个，如何去找准确的依赖，要去

​		看官方文档，去里面找。如[Spring Boot 2.0.1 Eureka Server](http://cloud.spring.io/spring-cloud-static/Finchley.RC1/multi/multi_spring-cloud-eureka-server.html)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>com.geeur.demo.cloud</groupId>
	<artifactId>eureka-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>eureka-server</name>
	<description>eureka-server</description>

	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.0.1.RELEASE</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
		<project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
		<java.version>1.8</java.version>
		<spring-cloud.version>Finchley.M9</spring-cloud.version>
	</properties>

	<dependencies>
		<!-- 服务注册 -->
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<dependencyManagement>
		<dependencies>
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>${spring-cloud.version}</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>
		</dependencies>
	</dependencyManagement>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

	<repositories>
		<repository>
			<id>spring-snapshots</id>
			<name>Spring Snapshots</name>
			<url>https://repo.spring.io/snapshot</url>
			<snapshots>
				<enabled>true</enabled>
			</snapshots>
		</repository>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>


</project>
```

创建好EurekaServer 项目后，通过`@EnableEurekaServer`注解启动一个服务注册中心提供给其他应用进行对话。这一步只需要在你的Spring Boot 应用中添加这个注解，就可以开启此功能。代码如下：

```java
@EnableEurekaServer
@SpringCloudApplication
public class EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaServerApplication.class, args);
	}
}
```

这里使用更改了创建项目时的`@SpringBootApplication`注解，变为`@SpringCloudApplication`，其实就是一个注解的整合，这里不做过多说明。

接下来就是`application.yml` 的配置(默认创建项目时生成的是`application.properties` 文件)

```yaml
# 测试环境 peer1
---
# 服务端口
server:
  port: 9100

eureka:
  server:
  	# 正式环境不推荐加入此配置，以下server 配置为 Eureka Server的自我保护机制，在我们测试时候加入，是为了看效果
    eviction-interval-timer-in-ms: 5000	
    enable-self-preservation: false
  client:
  	# 是否向服务注册中心注册自己
    register-with-eureka: true
    # 是否检索服务
    fetch-registry: true
    serviceUrl:
      defaultZone: http://peer1:9100/eureka/,http://peer2:9101/eureka/,http://peer3:9102/eureka/
  instance:
    instanceId: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: false
    hostname: peer1
    appname: ${spring.application.name}
    # 以下两个配置正式环境推荐使用默认值，不建议修改
    # 表示eureka client间隔多久去拉取服务注册信息，默认为30秒，对于api-gateway，如果要迅速获取服务注册状态，可以缩小该值，比如5秒
    lease-renewal-interval-in-seconds: 5
    # 表示eureka server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则将移除该instance
    lease-expiration-duration-in-seconds: 10

spring:
  application:
  	# 微服务名称，后续在调用的时候只需要使用该名称就可以进行服务的访问
    name: eureka-server
  profiles: peer1

# 测试环境 peer2
---
server:
  port: 9101

eureka:
  server:
    eviction-interval-timer-in-ms: 5000
    enable-self-preservation: false
  client:
    register-with-eureka: true
    fetch-registry: true
    serviceUrl:
      defaultZone: http://peer1:9100/eureka/,http://peer2:9101/eureka/,http://peer3:9102/eureka/
  instance:
    instanceId: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: false
    hostname: peer2
    appname: ${spring.application.name}
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 10

spring:
  application:
    name: eureka-server
  profiles: peer2

# 测试环境 peer3
---
server:
  port: 9102

eureka:
  server:
    eviction-interval-timer-in-ms: 5000
    enable-self-preservation: false
  client:
    register-with-eureka: true
    fetch-registry: true
    serviceUrl:
      defaultZone: http://peer1:9100/eureka/,http://peer2:9101/eureka/,http://peer3:9102/eureka/
  instance:
    instanceId: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: false
    hostname: peer3
    appname: ${spring.application.name}
    lease-renewal-interval-in-seconds: 5
    lease-expiration-duration-in-seconds: 10

spring:
  application:
    name: eureka-server
  profiles: peer3

#生产环境推荐配置
---
server:
  port: 9103

eureka:
  client:
    serviceUrl:
      defaultZone: http://peer1:9100/eureka/,http://peer2:9101/eureka/,http://peer3:9102/eureka/,http://prod:9103/eureka/
  instance:
    instanceId: ${eureka.instance.hostname}:${server.port}
    prefer-ip-address: false
    hostname: prod
    appname: ${spring.application.name}

spring:
  application:
    name: eureka-server
  profiles: prod
```

注意：这里需要在你们的 hosts  文件中加入如下配置

```
127.0.0.1 peer1
127.0.0.1 peer2
127.0.0.1 peer3
```

因为我们要做Eureka Server 的集群，所以这个在我的`application.yml` 配置中，分别有三个配置

peer1、peer2、peer3。然后我们启动Eureka Server 三次，根据不同的配置，在IDEA 中如何启动这三个配置。

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/eureka1.png?raw=true)

修改Program arguments: --spring.profiles.active=peer1、 --spring.profiles.active=peer2、--spring.profiles.active=peer3，启动三次即可。任意访问其中一个：

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/eureka2.png?raw=true)

可以看到我们的集群已经搭建好了。这里我们为什么要注册自己（register-with-eureka: true和

fetch-registry: true这两个配置），我们可以看到下面那个红框中的数据，registered-replicas（注册副本）、

unavailable-replicas（不可用副本）、available-replicas（可用副本）。当我们其中某一个Eureka Server 死掉

的时候，unavailable-replicas就会有死掉的是哪一个Eureka Server，根据这个，我们就可以快速的重启对应的

Eureka Server。

通常情况下是，启动两个注册中心c1和c2，但是在c1注册中心的available-replicas项中没有c2存在，反而是unavailable-replicas中有。

加入的配置为：

```yaml
# 1、默认即可，源码中，这两个配置均为  true
register-with-eureka: true
fetch-registry: true
# 2、eureka.client.serviceUrl.defaultZone配置项的地址，不能使用localhost，要使用service-center-1之类的域名，通过host映射到127.0.0.1；这里使用的是peer1、peer2、peer3
# 3、spring.application.name或eureka.instance.appname必须一致；
```

现在我们将peer2 的Eureka Server 死掉，再看情况

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/eureka3.png?raw=true)

我们可以看到 `http://peer2:9101/eureka/` 出现在 unavailable-replicas，在我们的注册区EUREKA-SERVER 中也

只有两个Eureka Server。

### 创建“服务提供方”

创建提供支付服务的为服务模块，并向服务注册中心注册

创建一个名叫Pay Server 的Spring Boot 服务模块，在pom.xml 引入如下配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.geeur</groupId>
    <artifactId>pay-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>pay-server</name>
    <description>pay-server</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.0.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
        <spring-cloud.version>Finchley.M9</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 服务发现 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <build>
        <finalName>weixin-server</finalName>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <encoding>UTF-8</encoding>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <repositories>
        <repository>
            <id>spring-milestones</id>
            <name>Spring Milestones</name>
            <url>https://repo.spring.io/milestone</url>
            <snapshots>
                <enabled>false</enabled>
            </snapshots>
        </repository>
    </repositories>

</project>
```

然后在主应用类上加上`@EnableDiscoveryClient`注解，该注解能激活Eureka中的`DiscoveryClient`实现，才能实

现Controller中对服务信息的输出。

```java
/**
 * 把 @SpringBootApplication 修改为 @SpringCloudApplication
 * 此注解中已经包含了 @EnableDiscoveryClient 注解
 */
@SpringCloudApplication
public class PayServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(PayServerApplication.class, args);
    }
}
```

接下来就是`application.yml` 的配置(默认创建项目时生成的是`application.properties` 文件)

```yaml
server:
  port: 9200
eureka:
  client:
    service-url:
      # 指定服务注册中心的位置（这里指定集群中的一个就可以）
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
  	# 微服务名称，后续在调用的时候只需要使用该名称就可以进行服务的访问
    name: pay-server
```

启动Pay Server 项目，访问我们的`http://peer1:9100`，显示如下图：

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/eureka4.png?raw=true)

我们可以看到我们的服务中心已经注册了Pay Server，现在我们将 peer1的Eureka Server杀掉，为什么杀掉它，

因为我们的Pay Server 中，`defaultZone`是向`http://peer1:9100/eureka/`注册的。显示如下图：

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/eureka5.png?raw=true)

可以看到，我们的`unavailable-replicas` 中出现了`http://peer1:9100/eureka/`，但是我们的注册区任然是存在

Pay Server 这个服务的。