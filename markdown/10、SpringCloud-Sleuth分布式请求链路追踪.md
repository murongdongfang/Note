# 简介



在微服务框架中，一个由客户端发起的请求在后端系统中会经过多个不同的的服务节点调用来协同产生最后的请求结果，每一个前段请求都会形成一复杂的分布式服务调用链路,链路中的任何一环出现高延时或错误都会引|起整个请求最后的失败。**Spring Cloud Sleuth提供了一套完整的服务跟踪的解决方案。在分布式系统中提供追踪解决方案并且兼容支持了zipkin**



Sleuth github地址：https://github.com/spring-cloud/spring-cloud-sleuth

# 安装访问zipkin

zipkin是sleuth服务追踪图形化显示支持

SpringCloud从F版起已不需要自己构建Zipkin server了，只需要调用jar包即可

zipkin jar包下载地址：https://dl.bintray.com/openzipkin/maven/io/zipkin/java/zipkin-server/

运行zipkin`java -jar zipkin-server-2.12.9-exec.jar`

![image-20200815233347588](https://gitee.com/little_broken_child_9527/images/raw/master/20200815233349.png)

浏览器访问：`localhost:9411/zipkin`

![image-20200815234344630](https://gitee.com/little_broken_child_9527/images/raw/master/20200815234346.png)

**Trace:类似于树结构的Span集合，表示一条调用链路，存在唯一标识。**

**span:表示调用链路来源，通俗的理解span就是一次请求信息**

如图表示一请求链路，一条链路通过Trace ld唯一标识， Span标识发起的请求信息，各span通过parent id关联起来

<img src="https://gitee.com/little_broken_child_9527/images/raw/master/20200815234012.png" alt="image-20200815234000790" style="zoom:150%;" />





![image-20200815234223055](https://gitee.com/little_broken_child_9527/images/raw/master/20200815234224.png)

![image-20200815234234541](https://gitee.com/little_broken_child_9527/images/raw/master/20200815234235.png)

# 编码

使用消费者order80调用生产者payment8001

## 生产者payment8001

+ pom依赖

```xml
<!--spring cloud链路追踪组件，包含了sleuth+zipkin-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

+ yml配置文件配置zipkin

```yml
server:
  port: 8001
spring:
  application:
    name: cloud-payment-provider-8001
  zipkin:
    # 监控的信息最终展示到http://localhost:9411这个地址
    base-url: http://localhost:9411
    sleuth:
      sampler:
        #采样率介于0-1之间，1表示全部采集，一般用0.5即可，采样率越高越耗费性能
        probability: 1


eureka:
  client:
    #是否注册进微服务，除了Eureka Server其他都需要注册进去
    register-with-eureka: true
    #是否从EurekaServer抓取已有的注册信息，默认为true，单节点无所谓，集群必须设置为true才能配合Ribbon
    fetch-registry: true
    #提供者访问注册中心，讲服务注册到注册中心
    service-url:
      #注册中心向外暴露的地址，如果Eureka Server是集群需要把所有集群暴露的地址写下，多个地址用逗号分隔
      defaultZone: http://localhost:7001/eureka/
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



+ 生产者被调用

```JAVA
@RestController
@RequestMapping("/payment")
@Slf4j
public class PaymentController {

  @GetMapping("/payment/zipkin")
  public String paymentZipkin()
  {
    return "hi ,i'am paymentzipkin server fall back，welcome to atguigu，O(∩_∩)O哈哈~";
  }

}
```

## 消费者order80

+ pom依赖，依赖通生产者8001

+ yml配置文件

```yml
server:
  port: 80
spring:
  application:
    name: consumer-order-consumer-80
  zipkin:
    # 监控的信息最终展示到http://localhost:9411这个地址
    base-url: http://localhost:9411
    sleuth:
      sampler:
        #采样率介于0-1之间，1表示全部采集，一般用0.5即可，采样率越高越耗费性能
        probability: 1


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

+ 然后使用RestTemplate远程调用

```java
@RestController
public class OrderContrller {

  @Autowired
  private RestTemplate restTemplate;

//  private static final String PAYMENT_URL = "http://localhost:8001";
  private static final String PAYMENT_URL = "http://CLOUD-PAYMENT-PROVIDER-8001";

  // ====================> zipkin+sleuth
  @GetMapping("/consumer/payment/zipkin")
  public String paymentZipkin()
  {
    String result = restTemplate.getForObject(PAYMENT_URL+"/payment/zipkin/", String.class);
    return result;
  }
}
```

+ zikpin图形化展示微服务调用线路

![image-20200816005329019](https://gitee.com/little_broken_child_9527/images/raw/master/20200816005330.png)



















































