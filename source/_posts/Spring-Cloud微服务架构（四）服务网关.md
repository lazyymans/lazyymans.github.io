---
title: Spring-Cloud微服务架构（四）服务网关
date: 2018-05-09 23:49:32
tags: API GateWay
categories: Spring Cloud
---
之前，我们已经初步完成了一个简易版的微服务分布式系统了。我们使用Spring Cloud Netflix中的Eureka实现了服务注册中心以及服务注册与发现；而服务间通过Ribbon或Feign实现服务的消费以及均衡负载；为了使得服务集群更为健壮，使用Hystrix的融断机制来避免在微服务架构中个别服务出现异常时引起的故障蔓延。

在该架构中，我们的服务集群包含：内部服务`Pay Server`和`Provider Server`，他们都会注册与订阅服务至`Eureka Server`，这里还有个对外提供的一个服务，这个服务通过负载均衡提供调用服务`Pay Server`和服务`Provider Server`的方法，本文我们把焦点聚集在对外服务这块，这样的实现是否合理，或者是否有更好的实现方式呢？

先来说说这样架构需要做的一些事儿以及存在的不足：

- 首先，破坏了服务无状态特点。为了保证对外服务的安全性，我们需要实现对服务访问的权限控制，而开放服务的权限控制机制将会贯穿并污染整个开放服务的业务逻辑，这会带来的最直接问题是，破坏了服务集群中REST API无状态的特点。从具体开发和测试的角度来说，在工作中除了要考虑实际的业务逻辑之外，还需要额外可续对接口访问的控制处理。
- 其次，无法直接复用既有接口。当我们需要对一个即有的集群内访问接口，实现外部服务访问时，我们不得不通过在原有接口上增加校验逻辑，或增加一个代理调用来实现权限控制，无法直接复用原有的接口。

面对类似上面的问题，我们要如何解决呢？下面进入本文的正题：服务网关！

为了解决上面这些问题，我们需要将权限控制这样的东西从我们的服务单元中抽离出去，而最适合这些逻辑的地方就是处于对外访问最前端的地方，我们需要一个更强大一些的均衡负载器，它就是本文将来介绍的：服务网关。

服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud Netflix中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

下面我们通过实例例子来使用一下Zuul来作为服务的路有功能。

### 开始使用Zuul

1、创建一个`Web Gateway`服务，引入pom.xml 所需要的依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.blog</groupId>
    <artifactId>web-gateway</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>web-gateway</name>
    <description>web-gateway</description>

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

        <!-- zuul -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
        </dependency>

        <!-- 服务发现 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>

        <!-- 服务消费者 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
        </dependency>

        <!-- 断路器 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
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

2、在主应用类上开启zuul，加入注解`@EnableZuulProxy`

```java
@EnableZuulProxy
@EnableEurekaClient
@SpringCloudApplication
public class WebGatewayApplication {

    @Bean
    @LoadBalanced   //开启负载均衡
    RestTemplate restTemplate(){
        SimpleClientHttpRequestFactory simpleClientHttpRequestFactory  = new SimpleClientHttpRequestFactory ();
        simpleClientHttpRequestFactory.setConnectTimeout(10000);
        simpleClientHttpRequestFactory.setReadTimeout(10000);
        return new RestTemplate(simpleClientHttpRequestFactory);
    }

    public static void main(String[] args) {
        SpringApplication.run(WebGatewayApplication.class, args);
    }
}
```

3、在`application.yml`中配置zuul

```yaml
server:
  port: 9400
eureka:
  client:
    service-url:
      defaultZone: http://peer1:9100/eureka/
  instance:
    instanceId: ${spring.application.name}:${server.port}
    prefer-ip-address: false
spring:
  application:
    name: web-gateway

pay-server:
  ribbon:
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #轮调
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机分配
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule

provider-server:
  ribbon:
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #轮调
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机分配
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule

##断路器 将 hystrix 的超时时间设置成 60000 毫秒（60秒）
hystrix:
  command:
    default:
      execution:
        timeout:
          enabled: false
        isolation:
          thread:
            timeoutInMilliseconds: 60000
# zuul
zuul:
  host:
    socket-timeout-millis: 60000
    connect-timeout-millis: 60000
  routes:
    api-pay:
      path: /pay/**
#      url: http://localhost:9200/
      serviceId: pay-server
    api-provider:
      path: /provider/**
      serviceId: provider-server
```

zuul 配置介绍

#### 服务路由

通过服务路由的功能，我们在对外提供服务的时候，只需要通过暴露Zuul中配置的调用地址就可以让调用方统一的来访问我们的服务，而不需要了解具体提供服务的主机信息了。

在Zuul中提供了两种映射方式：

- 通过url直接映射，我们可以如下配置：（实际项目中不推荐使用）

  ```yaml
  # zuul
  zuul:
    host:
      socket-timeout-millis: 60000
      connect-timeout-millis: 60000
    routes:
      api-pay:
        path: /pay/**
        url: http://localhost:9200/
      api-provider:
        path: /provider/**
        url: http://localhost:9300/
  ```

  该配置中将所有的/pay/**的访问映射到`http://localhost:9200/`上，也就是说当我们访问`http://localhost:9400/pay/api/pay/testPay`的时候，Zuul会将该请求路由到：`http://localhost:9200/api/pay/testPay`上。

  其中，配置属性zuul.routes.api-pay.path中的api-pay部分为路由的名字，可以任意定义，但是一组映射关系的path和url要相同，下面讲serviceId时候也是如此。


- 通过serviceId映射（推荐配置）

  通过url映射的方式对于Zuul来说，并不是特别友好，Zuul需要知道我们所有为服务的地址，才能完成所有的映射配置。而实际上，我们在实现微服务架构是，服务名与服务实例地址的关系在`Eureka Server`中已经存在了，所以只需要将Zuul注册到`Eureka Server`上去发现其他服务，我们就可以实现对serviceId的映射。例如，我们可以如下配置：

  ```yaml
  # zuul
  zuul:
    host:
      socket-timeout-millis: 60000
      connect-timeout-millis: 60000
    routes:
      api-pay:
        path: /pay/**
        serviceId: pay-server
      api-provider:
        path: /provider/**
        serviceId: provider-server
  ```

  这里我们说明下为什么推荐使用这种配置。当我们整个系统运行起来的时候，肯定会把我们一个个的服务集群化，这样我们使用直接的url映射，将无法进行负载均衡。也就容易出现某一台服务器的负载过高，而其他服务器空闲，资源浪费。

  下面，我们就来演示一下效果，分别启动`Eureka Server`、`Pay Server`、`Provider Server`、`Web Gateway`四个服务。

  ![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/gateway1.png?raw=true)
  然后我们同过服务网关来访问`Pay Server`、`Provider Server`，根据配置的映射关系，分别访问下面的url

  `http://localhost:9400/pay/api/pay/testPay`：通过serviceId映射访问`Pay Server`中的testPay服务

  `http://localhost:9400/provider/api/provider/testProvider`：通过serviceId映射访问`Provider Server`中的testProvider服务

  访问结果

  ![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/gateway2.png?raw=true)


#### 服务过滤

在完成了服务路由之后，我们对外开放服务还需要一些安全措施来保护客户端只能访问它应该访问到的资源。所以我们需要利用Zuul的过滤器来实现我们对外服务的安全控制。

在服务网关中定义过滤器只需要继承`ZuulFilter`抽象类实现其定义的四个抽象函数就可对请求进行拦截与过滤。

比如下面的例子，定义了一个Zuul过滤器，实现了在请求被路由之前检查请求中是否有`accessToken`参数，若有就进行路由，若没有就拒绝访问，返回`401 Unauthorized`错误。

```java
public class ZuulPreFilter extends ZuulFilter {

    private static Logger log = LoggerFactory.getLogger(ZuulPreFilter.class);

    /**
     * ZuulFilter 的类型
     * @return
     */
    @Override
    public String filterType() {
        return "pre";
    }

    /**
     * 执行顺序控制
     * @return
     */
    @Override
    public int filterOrder() {
        return 0;
    }

    /**
     * 返回一个boolean类型来判断该过滤器是否要执行，所以通过此函数可实现过滤器的开关。我们直接返回true，所以该过滤器总是生效。
     * @return
     */
    @Override
    public boolean shouldFilter() {
        return true;
    }

    /**
     * 过滤器的具体逻辑。这里我们通过ctx.setSendZuulResponse(false)令zuul过滤该请求，不对其进行路由，
     * 然后通过ctx.setResponseStatusCode(401)设置了其返回的错误码，
     * 当然我们也可以进一步优化我们的返回，比如，通过ctx.setResponseBody(body)对返回body内容进行编辑等。
     */
    @Override
    public Object run() throws ZuulException {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        log.info(String.format("%s request to %s", request.getMethod(), request.getRequestURL().toString()));

        Object accessToken = request.getParameter("accessToken");
        if(accessToken == null) {
            log.warn("access token is empty");
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(401);
          	ctx.setResponseBody("access token is empty");
            return null;
        }
        log.info("access token ok");
        return null;
    }
}
```

实现了自定义过滤器之后，向容器中注入`ZuulPreFilter`

```java
@EnableZuulProxy
@EnableEurekaClient
@SpringCloudApplication
public class WebGatewayApplication {

    @Bean
    @LoadBalanced
        //开启负载均衡
    RestTemplate restTemplate() {
        SimpleClientHttpRequestFactory simpleClientHttpRequestFactory = new SimpleClientHttpRequestFactory();
        simpleClientHttpRequestFactory.setConnectTimeout(10000);
        simpleClientHttpRequestFactory.setReadTimeout(10000);
        return new RestTemplate(simpleClientHttpRequestFactory);
    }

  	/**
  	 * 加入ZuulPreFilter
  	 */
    @Bean
    public ZuulPreFilter zuulPreFilter() {
        return new ZuulPreFilter();
    }

    public static void main(String[] args) {
        SpringApplication.run(WebGatewayApplication.class, args);
    }
}
```

重新启动我们的`Web Gateway`服务，访问`http://localhost:9400/pay/api/pay/testPay`：状态码401，响应体access token is empty
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/gateway3.png?raw=true)
访问`http://localhost:9400/pay/api/pay/testPay?accessToken=xxx`：状态码200，响应体test，访问成功
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/gateway4.png?raw=true)
对于其他一些过滤类型，这里就不一一展开了，根据之前对filterType生命周期介绍，可以参考下图去理解，并根据自己的需要在不同的生命周期中去实现不同类型的过滤器。
![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/gateway5.png?raw=true)
最后，总结一下为什么服务网关是微服务架构的重要部分，是我们必须要去做的原因：

- 不仅仅实现了路由功能来屏蔽诸多服务细节，更实现了服务级别、均衡负载的路由。
- 实现了接口权限校验与微服务业务逻辑的解耦。通过服务网关中的过滤器，在各生命周期中去校验请求的内容，将原本在对外服务层做的校验前移，保证了微服务的无状态性，同时降低了微服务的测试难度，让服务本身更集中关注业务逻辑的处理。
- 实现了断路器，不会因为具体微服务的故障而导致服务网关的阻塞，依然可以对外服务。