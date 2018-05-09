---
title: Spring-Cloud微服务架构（三）断路器
date: 2018-05-09 16:51:51
tags: Hystrix
categories: Spring Cloud
---

在微服务架构中，我们把整个系统拆分成一个个的子服务，每个服务之间通过注册与订阅的方式进行相互依赖。每个服务在不同的进程中运行，通过远程调用的方式进行访问。这样就有可能因为网络原因或是依赖服务自身问题出现调用故障或延迟，而这些问题会直接导致调用方的对外服务也出现延迟，若此时调用方的请求不断增加，最后就会出现因等待出现故障的依赖方响应而形成任务积压，最终导致自身服务的瘫痪。形成服务器雪崩

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hystrix1.jpg?raw=true)
如上图所示：A为服务提供者，B为A的服务调用者，C和D是B的服务调用者。当A的不可用，引起B的不可用，并将不可用逐渐放大C和D时，服务雪崩就形成了。

### Netflix Hystrix断路器

在Spring Cloud中使用了[Hystrix](https://github.com/Netflix/Hystrix) 来实现断路器的功能。Hystrix是Netflix开源的微服务框架套件之一，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能。

继续之前的的两个服务来做断路器测试。

### 1、启动Eureka Server、Pay Server、Provider Server

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hystrix2.png?raw=true)
访问`http://localhost:9200/api/pay/testProviderRibbon`，调用`Provider Server`的服务，可以看到我们的测试返回 testProvider success。

然后我们关闭 `Provider Server`，在浏览器再次访问`http://localhost:9200/api/pay/testProviderRibbon`，我们得到报错信息。

```
Whitelabel Error Page

This application has no explicit mapping for /error, so you are seeing this as a fallback.

Wed May 09 23:06:29 CST 2018
There was an unexpected error (type=Internal Server Error, status=500).
I/O error on POST request for "http://provider-server/api/provider/testProvider": Connection refused: connect; nested exception is java.net.ConnectException: Connection refused: connect
```

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hystrix3.png?raw=true)

开启`Pay Server`服务的断路器功能，在住应用类`PayServerApplication.java`上加上`@EnableCircuitBreaker`，代码如下：

```java
@EnableEurekaClient
@SpringCloudApplication
@EnableFeignClients     //Feign支持
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

可以看到我们，没有加上这个注解。因为我们用的是`@SpringCloudApplication`这个注解，它里面包含了`@EnableCircuitBreaker`这个注解

```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public @interface SpringCloudApplication {
}
```
### Ribbon中使用Hystrix

改造`Pay Server` 的`TestPayController.java` 类，在接口`testProviderRibbon`加入注解`@HystrixCommand`，代码如下：

```java
@RestController
@RequestMapping("/api/pay")
public class TestPayController {

    private final static Logger log = LoggerFactory.getLogger(TestPayController.class);

    @Autowired
    private DiscoveryClient client;

    @Autowired
    private ProviderFeginCustomClient feignClient;

    @Autowired
    private RestTemplate restTemplate;

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @RequestMapping("/testPay")
    public String testPay(){
        List<ServiceInstance> instances = client.getInstances("pay-server");
        ServiceInstance instance = instances.get(0);
        log.info("/testPay, host:" + instance.getHost() + ", service_id:" + instance.getServiceId());
        return "test";
    }

    @RequestMapping("/testProviderFeign")
    public String testProviderFeign(){
        ServiceInstance instance = this.loadBalancerClient.choose("provider-server");
        log.info("testProviderFeign >>>>>" + " " + instance.getServiceId() + ":" + instance.getHost() + ":" + instance.getPort());
        return feignClient.testProvider();
    }

    @RequestMapping("/testProviderRibbon")
    @HystrixCommand(fallbackMethod = "testProviderRibbonFallback")
    public String testProviderRibbon(){
        ServiceInstance instance = this.loadBalancerClient.choose("provider-server");
        log.info(">>>>>" + " " + instance.getServiceId() + ":" + instance.getHost() + ":" + instance.getPort());
        return restTemplate.postForObject("http://provider-server/api/provider/testProvider", null, String.class);
    }

    private String testProviderRibbonFallback(){
        return "error";
    }

}
```

可以看到我们现在已经在 `testProviderRibbon`方法上加上了注解，并且添加了一个回调方法。

现在，我们重新启动`Pay Server`，也就是我们只启动`Eureka Server`和`Pay Server`两个服务，再次访问`http://localhost:9200/api/pay/testProviderRibbon`，我们可以看到页面显示：error

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hystrix4.png?raw=true)

### Feign中使用Hystrix

1、Feign 是默认关闭 Hystrix的，所以我们要在配置里打开它，修改`Pay Server`的`application.yml`

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
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RoundRobinRule #轮调
    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.RandomRule #随机分配
#    NFLoadBalancerRuleClassName: com.netflix.loadbalancer.WeightedResponseTimeRule

feign:
  hystrix:
    enabled: true

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
```

也就是添加了feign.hystrix.enabled=true 这个属性。

2、创建类`ProviderFeginCustomClientFallback.java`，并且实现`ProviderFeginCustomClient.java`

```java
@Component
public class ProviderFeginCustomClientFallback implements ProviderFeginCustomClient {

    @Override
    public String testProvider() {
        return "errorFeign";
    }
}
```

3、修改`ProviderFeginCustomClient.java`，在`@FeignClient`中添加属性 `fallback`，并配置为`ProviderFeginCustomClientFallback`。

```java
@FeignClient(name = "provider-server",
        fallback = ProviderFeginCustomClientFallback.class)
public interface ProviderFeginCustomClient {

    @RequestMapping(value = "/api/provider/testProvider", method = RequestMethod.GET)
    public String testProvider();

}
```

现在，我们重新启动`Pay Server`，也就是我们只启动`Eureka Server`和`Pay Server`两个服务，在浏览器访问`http://localhost:9200/api/pay/testProviderFeign`，我们可以看到页面显示：errorFeign

![](https://github.com/lazyymans/lazyymans.github.io/blob/hexo/source/img/hystrix5.png?raw=true)