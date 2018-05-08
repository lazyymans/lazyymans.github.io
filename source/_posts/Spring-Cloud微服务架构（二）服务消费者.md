---
title: Spring Cloud微服务架构（二）服务消费者
date: 2018-05-07 19:36:11
categories: Spring Cloud
tags: [Ribbon,Feign]
---

在上一篇中[《Spring Cloud微服务架构（一）高可用服务注册与发现》](https://lazyymans.github.io/2018/05/07/Spring-Cloud%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%EF%BC%88%E4%B8%80%EF%BC%89%E9%AB%98%E5%8F%AF%E7%94%A8%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/)中，我们已经创建了`服务注册中心`，并且注册了一个`服务提供者 pay-server`，那么接下来我们要怎么去消费提供者提供的接口呢？

### 首先创建 Provider Server 服务提供方

创建一个基本的Provider Server 项目，在pom.xml 引入以下依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.blog</groupId>
    <artifactId>provider-server</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>provider-server</name>
    <description>provider-server</description>

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
        <spring-cloud.version>Finchley.RC1</spring-cloud.version>
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

        <!-- 断路器 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
        </dependency>

        <!-- 服务消费者 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
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

创建好Provider Server 项目后，将`@SpringBootApplication`修改为`SpringCloudApplication`注解，将此服务注册到`Eureka Server`中，代码如下：

```java
@EnableEurekaClient
@SpringCloudApplication
public class ProviderServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProviderServerApplication.class, args);
    }
}
```

配置`application.yml`

```yaml
---
server:
  port: 9300
eureka:
  client:
    service-url:
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
    name: provider-server
  profiles: peer1

---
server:
  port: 9301
eureka:
  client:
    service-url:
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
    name: provider-server
  profiles: peer2

---
server:
  port: 9302
eureka:
  client:
    service-url:
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
    name: provider-server
  profiles: peer3
```

这里有三个配置，用来测试我们的负载均衡，服务调用方具体是访问的是那个端口下的服务。

接下来编写一个提供其他服务调用的接口，这里也就是我们的`Pay Server`来调用，代码如下：

```java
@RestController
@RequestMapping("/api/provider")
public class TestProviderController {

    private final static Logger log = LoggerFactory.getLogger(TestProviderController.class);

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @RequestMapping("testProvider")
    public String testProvider(){
        ServiceInstance instance = this.loadBalancerClient.choose("provider-server");
        log.info(">>>>>" + " " + instance.getServiceId() + ":" + instance.getHost() + ":" + instance.getPort());
        return "testProvider success";
    }

}
```

到此，服务提供方就已经做好了。

### Ribbon

`Ribbon` 是一个客户端负载均衡器，可以让您对HTTP和TCP客户端的行为有很多控制权。Feign已经使用Ribbon，因此，如果您使用`@FeignClient`，此部分也适用。原理我们这里就不做过多说明，直接进入实战演练。

下面我们通过实例看看如何使用Ribbon来调用服务，并实现客户端的均衡负载。

上面，我们已经创建好了`Provider Server`服务提供方，在`Pay Server`中使用`Ribbon `来进行消费。

#### 1、修改`Pay Server`主应用类：

```java
@EnableEurekaClient
@SpringCloudApplication
public class PayServerApplication {

    @Bean
    @LoadBalanced   //开启负载均衡
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

    public static void main(String[] args) {
        SpringApplication.run(PayServerApplication.class, args);
    }
}
```

#### 2、创建TestPayController 来消费`Provider Server`的 testProvider 服务，通过直接RestTemplate来调用服务。

```java
@RestController
@RequestMapping("/api/pay")
public class TestPayController {

    private final static Logger log = LoggerFactory.getLogger(TestPayController.class);

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @RequestMapping("/testProviderRibbon")
    public String testProviderRibbon(){
        ServiceInstance instance = this.loadBalancerClient.choose("provider-server");
        log.info(">>>>>" + " " + instance.getServiceId() + ":" + instance.getHost() + ":" + instance.getPort());
        return restTemplate.postForObject("http://provider-server/api/provider/testProvider", null, String.class);
    }

}
```

#### 3、修改`application.yml`

```yaml
---
server:
  port: 9200
eureka:
  client:
    service-url:
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
    name: pay-server
  profiles: peer1

provider-server:
  ribbon:
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #轮询
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机分配
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule	#加权响应时间

#断路器 将 hystrix 的超时时间设置成 60000 毫秒（60秒）
hystrix:
  command:
    default:
      execution:
        isolation:
          thread:
            timeoutInMilliseconds: 60000
```

以上工作做完之后，我们就可以启动`Pay Server`和`Provider Server`了。将`Provider Server`分别根据配置启动三次（Spring Cloud微服务架构（一）高可用服务注册与发现 有说明如何根据不同配置启动）。启动后如下图
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/ribbon1.png?raw=true)
接下来我们访问`http://localhost:9200/api/pay/testProviderRibbon`（也就是我们`Pay Server`的接口），查看日志和访问结果。
调用日志：
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/ribbon2.png?raw=true)
访问结果：
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/ribbon3.png?raw=true)
我们可以看到，按顺序访问的端口服务器，这是因为我们在配置负载均衡的时候使用的是轮询模式。修改负载均衡模式（这次我们使用随机分配模式），我们在次调用。
调用日志：
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/ribbon4.png?raw=true)

可以发现，现在调用的端口服务器是随机的。
这里，通过`Ribbon`的方式来消费服务的方式就已经做好了。

### Feign

