获取SpringBoot和SpringCloud适配关系

https://start.spring.io/actuator/info



# RestTemplate远程调用

RestTemplate使用步骤

1. 将RestTemplate注入容器中

2. 远程调用接口

   

```java
@RestController
public class OrderContrller {

  @Autowired
  private RestTemplate restTemplate;

  private static final String PAYMENT_URL = "http://localhost:8001";

   /*
   		注意：restTemplate.getForObject()
   		方法和restTemplate.postForObject()二者参数的区别
   
   */
  @GetMapping("/consumer/order/{id}")
  public CommonResult<Payment> getPayment(@PathVariable Long id){
    //远程调用payment接口，get调用参数拼接的在URL后面
    CommonResult res = restTemplate.getForObject(PAYMENT_URL + "/payment/get/"+id, CommonResult.class);

    return res;
  }

  @GetMapping("/consumer/add")
  public CommonResult addPayment(Payment payment){
     //Post提交的参数在第二个参数而非URL后面
    CommonResult res = restTemplate.postForObject(PAYMENT_URL + "/payment/add", payment, CommonResult.class);
    return res;
  }
}
```



RestTemplate是基于Rest风格的Http远程调用，也就是说Producer的Controller必须是Rest风格的。此时Producer的Controller需要注意两点

1. **Controller必须是RestController，如果是普通Controller方法上面必须加@ResponseBody注解**
2. **如果时Post请求，提交参数上必须加@RequestBody**



# Eureka注册中心

Spring Cloud 封装了 Netflix 公司开发的 Eureka 模块来实现服务注册和发现。Eureka 采用了 C-S 的设计架构。Eureka Server 作为服务注册功能的服务器，它是服务注册中心。而系统中的其他微服务，使用 Eureka 的客户端连接到 Eureka Server，并维持心跳连接。这样系统的维护人员就可以通过 Eureka Server 来监控系统中各个微服务是否正常运行。Spring Cloud 的一些其他模块（比如Zuul）就可以通过 Eureka Server 来发现系统中的其他微服务，并执行相关的逻辑。

Eureka由两个组件组成：Eureka服务器和Eureka客户端。Eureka服务器用作服务注册服务器。Eureka客户端是一个java客户端，用来简化与服务器的交互、作为轮询负载均衡器，并提供服务的故障切换支持。Netflix在其生产环境中使用的是另外的客户端，它提供基于流量、资源利用率以及出错状态的加权负载均衡。

![](http://favorites.ren/assets/images/2017/springcloud/eureka-architecture-overview.png)

## 新旧依赖区别

```xml
旧版：client和server的依赖是在一块的
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>

新版：将client和server的依赖分开分开
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



## Eureka Server

+ 引入依赖

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```



+ yml配置

```yml
server:
  port: 7001
spring:
  application:
    name: cloud-eureka-server-7001
eureka:
  instance:
    hostname: localhost
  client:
    #自己就是注册中心，不需要注册自己
    register-with-eureka: false
    #自己就是注册中心，所以不需要从注册中心取回自己的信息
    fetch-registry: false
    #Eureka Server向外暴露的地址，客户端(consumer,provider)向注册中心注册的时候使用地址
    service-url:
      #单机版配置
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/

```

+ Server的主启动类上标注`@EnableEurekaServer`

+ 访问http://localhost:7001/ eureka server所在的ip+端口号

## Eureka Client

将服务消费者和服务提供者都注册到Eureka Server，注册的方式相同

+ 引入依赖

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

+ 配置yml

```yml
eureka:
  client:
    #是否注册进微服务，除了Eureka Server其他都需要注册进去
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true，单节点无所谓，集群必须设置为true才能配合Ribbon
    fetch-registry: true
    #提供者访问注册中心，讲服务注册到注册中心
    service-url:
      defaultZone: http://localhost:7001/eureka/   #注册中心向外暴露的地址
```

+ 启动上标注`@EnableEurekaClient`



## Eureka Server集群配置

在一台机器上部署Eureka Server集群需要注意：

+ 由于每一个Eureka Server都需要配置hostname，这个hostname就是Eureka Server所在IP地址，hostname的值不能重复，因此需要在本机上面修改host做DNS映射
+ **在单机模式中defaultZone需要写自己的IP地址和端口，而在集群模式中需要写的是其他Eureka Server的IP地址，多个地址用逗号分隔**

![image-20200523162029576](https://gitee.com/little_broken_child_9527/images/raw/master/20200523162031.png)



单机版的defaultZone配置的是自己的IP和端口，集群配置的是其他Eureka Server的Ip和端口

```yml
server:
  port: 7001
spring:
  application:
    name: cloud-eureka-server-7001
eureka:
  instance:
  	#配置当前主机IP，由于在一台机器上部署集群，需要修改host文件做DNS映射
    hostname: eureka7001.com
  client:
    #自己就是注册中心，不需要注册自己
    register-with-eureka: false
    #自己就是注册中心，所以不需要从注册中心取回自己的信息
    fetch-registry: false
    #eureka server集群配置，当前server节点配置的是其他eureka server的地址，多个地址使用","分割
    service-url:
      defaultZone: http://eureka7002.com:7002/eureka/

=======================================================
server:
  port: 7002

spring:
  application:
    name: eureka-server-7002

eureka:
  instance:
    hostname: eureka7002.com
  client:
    #自己就是注册中心，不需要注册自己
    register-with-eureka: false
    #自己就是注册中心，所以不需要从注册中心取回自己的信息
    fetch-registry: false
      #eureka server集群配置，当前server节点配置的是其他eureka server的地址，多个地址使用","分割
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/


```



## 微服务注册到Eureka Server集群

+ 服务提供者Provider注册到Eureka Server集群，**defalutZone需要配置所有Eureka Server向外暴露的地址** 
+ **如果Provider是集群配置那么所有的provider的spring.application.name都需要一样**，在使用Ribbon或者Feign远程调用的时候可以实现负载均衡

```yml
server:
  port: 8001
spring:
  application:
   #如果是Provider集群，所有的Provider的name必须一样
    name: cloud-payment-provider-8001

eureka:
  client:
    #是否注册进微服务，除了Eureka Server其他都需要注册进去
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true，单节点无所谓，集群必须设置为true才能配合Ribbon
    fetch-registry: true
    #提供者访问注册中心，讲服务注册到注册中心
    service-url:
      #注册中心向外暴露的地址，如果Eureka Server是集群需要把所有集群暴露的地址写下，多个地址用逗号分隔
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002:7002.com/eureka
```



![image-20200523165820446](https://gitee.com/little_broken_child_9527/images/raw/master/20200523165821.png)



## Consumer调用Provider集群

有了注册中心之后，Consumer远程调用Provider就可以不用使用IP+端口的方式调用，而是使用注册中心服务名调用Provider。如果该服务名下是Provider集群，必须在RestTemplate上开启负载均衡功能，默认使用轮询的负载均衡算法

+ 开启RestTemplate的负载均衡功能，只需要在将RestTemplate注入容器的时候加上`@LoadBalanced`注解即可，如果Provider部署的是集群配置，RestTemplate没有开启负载均衡会报错

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(){
    return new RestTemplate();
}

```

+ Consumer的时候Provider的时候使用注册中心的服务名调用

```java
@RestController
public class OrderContrller {
  @Autowired
  private RestTemplate restTemplate;
//Provider集群注册到注册中心的时候，Consumer使用微服务名+负载均衡调用 
  private static final String PAYMENT_URL = "http://CLOUD-PAYMENT-PROVIDER-8001";
  @GetMapping("/consumer/order/{id}")
  public CommonResult<Payment> getPayment(@PathVariable Long id){
    CommonResult res = restTemplate.getForObject(PAYMENT_URL + "/payment/get/"+id, CommonResult.class);
    return res;
  }
}
```

## actuator微服务信息完善

actuator是SpringBoot自带的服务监控组件，通过这个组件可以查看系统的运行信息。通常导入web-starter同时也要导入actuator-starer依赖，只有导入了actuator-starer微服务信息完善才能生效。导入了这个依赖之后输入http://localhost:8001/actuator/health等可以查看当前微服务健康状态

```xml
 <dependency>
     <groupId>org.springframework.boot</groupId>
     <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--监控-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```



完善微服务注册信息，没有完善信息前如图所示

存在两个问题：没法显示具体微服务的ip地址，集群配置的情况下，每个具体微服务默认是按照微服务名+端口显示不直接

![image-20200523174157502](https://gitee.com/little_broken_child_9527/images/raw/master/20200523174200.png)

```yml
server:
  port: 8002
spring:
  application:
    name: cloud-payment-provider-8001
eureka:
  client:
    #是否注册进微服务，除了Eureka Server其他都需要注册进去
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true，单节点无所谓，集群必须设置为true才能配合Ribbon
    fetch-registry: true
    #提供者访问注册中心，讲服务注册到注册中心
    service-url:
      #注册中心向外暴露的地址，如果Eureka Server是集群需要把所有集群暴露的地址写下，多个地址用逗号分隔
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002:7002.com/eureka
      #单机版
      #defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
     
  instance:  #actuor信息完善
    # 如果一个服务名下面是一个集群配置，具体微服务将显示instance-id配置的名
    instance-id: payment-8002
    #鼠标悬停具体微服务会显示这个微服务ip地址
    prefer-ip-address: true

```



完善信息之后如下图所示

![image-20200523174142070](https://gitee.com/little_broken_child_9527/images/raw/master/20200523174144.png)



## 服务发现DiscoveryClient

对于注册eureka里面的微服务，可以通过服务发现来获得该服务的信息。可以通过`DiscoverClient`获取该微服务所注册在的注册中心所有的微服务信息

1. 将DiscoverClient注册到容器中，通过该组件获取这个相应注册中心所有微服务信息

```java
 @Autowired
  private DiscoveryClient discoveryClient;

  @GetMapping("/discovery")
  public CommonResult<DiscoveryClient> getDiscoveryClient(){
    //获得注册到注册中心的微服务名
    List<String> services = discoveryClient.getServices();
    for (String service : services) {
      log.error("service===>"+service);
      //获取该微服务下面具体实例，因为可能该微服务名下面部署的是一个集群配置
      List<ServiceInstance> instances = discoveryClient.getInstances(service);
      for (ServiceInstance instance : instances) {
          //可以获取相应微服务的端口，地址，id等所有信息
        log.error("instance"+instance.getHost()+instance.getMetadata()+instance.getInstanceId());
      }

    }

    return CommonResult.ok(this.discoveryClient);
  }
```

2. 在该微服务的启动类上标注注解`@EnableDiscoveryClient`

## Eureka自我保护机制

<img src="https://gitee.com/little_broken_child_9527/images/raw/master/20200525233835.png" alt="image-20200525233833711"  />



默认情况下，如果EurekaServer在一定时间内没有接收到某 个微服务实例的心跳，EurekaServer将会注销该实例(默认90秒)。但是当网络分区故障发生(延时、卡顿、 拥挤)时，微服务与EurekaServer之间无法正常通信，以上行能变得非常危险了，因为微服务本身其实是健康的，此时本不应该注销这个微服务。Eureka通过" 自我保护模式”来解决这个问题。当EurekaServer节 点在短时间内丢失过多客户端时(可能发生了网络分区故障)，那么这个节点就会进入 自我保护模式。在自我保护模式中，Eureka Server会保护服务注册表中的信息，不再注销任何服务实例。
它的设计哲学就是宁可保留错误的服务注册信息，也不盲目注销任何可能健康的服务实例。通俗来讲就是好死不如赖活着综上，自我保护模式是一种应对网络异常的安全保护措施。 它的架构哲学是宁可同时保留所有微服务(健康的微服务和不健康的微服务都会保留)也不盲目注销任何健康的微服务。使用自我保护模式，可以让Eureka集群更加的健壮、 稳定。总之一句话：某时刻 一个微服务不可用了，Eureka不会立刻清理，依旧会对该服务的信息进行保存，自我保护设计理念是基于CAP理论中的AP分支。

为什么产生Eureka自我保护机制?
为了防止EurekaClient可以正常运行，但是与EurekaServer网络不通情况下，EurekaServer不会 立刻将EurekaClient服务剔除



**配置禁止自我保护**

EurekaServer使用`eureka.server.enable-self-preservation=false` 可以禁用自我保护模式

```yml
server:
  port: 7002

spring:
  application:
    name: eureka-server-7002

eureka:
  instance:
    hostname: eureka7002.com
  server:
    #关闭Eureka的自我保护机制
    enable-self-preservation: false
    #eureka server清理无效节点的时间间隔，默认60000毫秒，即60秒
    eviction-interval-timer-in-ms: 2000
  client:
    #自己就是注册中心，不需要注册自己
    register-with-eureka: false
    #自己就是注册中心，所以不需要从注册中心取回自己的信息
    fetch-registry: false
    #客户端(consumer,provider)访问Eureka时候使用的地址
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/

```

![image-20200526000438107](https://gitee.com/little_broken_child_9527/images/raw/master/20200526000439.png)

EurekaClient配置禁止自我保护机制

```yml
eureka:
  client:
    #是否注册进微服务，除了Eureka Server其他都需要注册进去
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true，单节点无所谓，集群必须设置为true才能配合Ribbon
    fetch-registry: true
    #提供者访问注册中心，讲服务注册到注册中心
    service-url:
      #注册中心向外暴露的地址，如果Eureka Server是集群需要把所有集群暴露的地址写下，多个地址用逗号分隔
      defaultZone: http://eureka7001.com:7001/eureka/,http://eureka7002:7002.com/eureka
      #单机版
      #defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
  instance:
    #配置健康检查 输入http://localhost:8001/actuator/health可以查看当前微服务健康状态
    # 如果一个服务名下面是一个集群配置，具体微服务将显示instance-id配置的名
    instance-id: payment-8001
    #鼠标悬停具体微服务会显示这个微服务ip地址
    prefer-ip-address: true
    #Eureka服务端在收到最后一次心跳后等待时间上限 ,单位为秒(默认是90秒),超时剔除服务
    lease-expiration-duration-in-seconds: 1
    #Eureka客户端向服务端发送心跳的时间间隔,单位为秒(默认是30秒)
    lease-renewal-interval-in-seconds: 2
```



`eureka.client.registry-fetch-interval-seconds`

表示eureka client间隔多久去拉取服务注册信息，默认为30秒，对于api-gateway，如果要迅速获取服务注册状态，可以缩小该值，比如5秒

`eureka.instance.lease-expiration-duration-in-seconds` 

表示eureka server至上一次收到client的心跳之后，等待下一次心跳的超时时间，在这个时间内若没收到下一次心跳，则将移除该instance。

- 默认为90秒
- 如果该值太大，则很可能将流量转发过去的时候，该instance已经不存活了。
- 如果该值设置太小了，则instance则很可能因为临时的网络抖动而被摘除掉。
- 该值至少应该大于leaseRenewalIntervalInSeconds



`eureka.instance.lease-renewal-interval-in-seconds`

leaseRenewalIntervalInSeconds，表示eureka client发送心跳给server端的频率。如果在leaseExpirationDurationInSeconds后，server端没有收到client的心跳，则将摘除该instance。除此之外，如果该instance实现了HealthCheckCallback，并决定让自己unavailable的话，则该instance也不会接收到流量。

- 默认30秒

`eureka.server.enable-self-preservation`

是否开启自我保护模式，默认为true。

默认情况下，如果Eureka Server在一定时间内没有接收到某个微服务实例的心跳，Eureka Server将会注销该实例（默认90秒）。但是当网络分区故障发生时，微服务与Eureka Server之间无法正常通信，以上行为可能变得非常危险了——因为微服务本身其实是健康的，此时本不应该注销这个微服务。

Eureka通过“自我保护模式”来解决这个问题——当Eureka Server节点在短时间内丢失过多客户端时（可能发生了网络分区故障），那么这个节点就会进入自我保护模式。一旦进入该模式，Eureka Server就会保护服务注册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。当网络故障恢复后，该Eureka Server节点会自动退出自我保护模式。

综上，自我保护模式是一种应对网络异常的安全保护措施。它的架构哲学是宁可同时保留所有微服务（健康的微服务和不健康的微服务都会保留），也不盲目注销任何健康的微服务。使用自我保护模式，可以让Eureka集群更加的健壮、稳定。

`eureka.server.eviction-interval-timer-in-ms`

eureka server清理无效节点的时间间隔，默认60000毫秒，即60秒





























