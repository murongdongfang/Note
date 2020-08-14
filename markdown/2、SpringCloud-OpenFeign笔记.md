# 简介



Feign是一个声明式WebService客户端。 使用Feign能让编写Web Service客户端更加简单。它的使用方法是定义一个服务接口然后在上面添加注解。Feign也支持可拔插式的编码器和解码器。Spring Cloud对Feign进行了封装使其支持了Spring MVC标准注解和HttpMessageConverters，Feign可以与Eureka和Ribbon组合使用以支持负载均衡。一句话Feign就是在Ribbon的基础上封装了父接口+注解方式调用的整合



## Feign作用
Feign旨在使编写Java Http客户端变得更容易。面在使用Ribbon+ RestTemplate时，利用RestTemplate对http请求的封装处理 ,形成了一套模版化的调用方法。但是在实际开发中，于对服务依赖的调可能不止一处，往往一个接口会被多处调用，所以通常都会针对每个微服务自行封装一些客户端类来包装这些依赖服务的调用。所以，Feign在呲基础 上做了进一步封装,由他来帮助我们定义和实现依赖服务接口的定义。**在Feign的实现下，我们只需创建一个接口并使用注解的方式来配置它**(以前是Dao接口上面标注Mapper注解现在是一个微服务接口上面标注一个Feign注解即可)，即可完成对服务提供方的接口绑定，简化了使佣Spring cloud Ribbon时，自动封装服务调用客户端的开发量。Feign集成了Ribbon，利用Ribbon维护了Payment的服务列表信息，通过轮询实现了客户端的负载均衡。而与Ribbon不同的是，通过feign只需要定义服务绑定接口且以声明式的方法，优雅而简单的实现了服务调用。

https://github.com/spring-cloud/spring-cloud-openfeign



## Feign和OpenFeign的区别

| Feign                                                        | OpenFeign                                                    |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| Feign是Spring Cloud组件中的一个轻量级HTTP服务客户端，Feign内置了Ribbon，用来做客户端负载均衡，去调用服务注册中心的服务。Feign的使用方式是：使用Feign的注解定义接口，调用这个接口，就可以实现服务注册中心的功能<br /><dependency><br/>	<groupId>org.springframework.cloud</groupId><br/>    <artifactId>spring-cloud-starter-feign</artifactId><br/></dependency> | OpenFeign是SpringCloud在Feign的基础上支持了SpringMVC的注解，如@RequestMapping等。OpenFeign的@FeignClient可以解析SpringMVC的@ReqeustMapping注解下的接口，并通过动态代理的方式产生实现类，实现类中负载均衡并调用其他服务<br /><dependency><br/>	<groupId>org.springframework.cloud</groupId><br/>    <artifactId>spring-cloud-starter-openfeign</artifactId><br/></dependency> |



```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-feign</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```



# OpenFeign使用

+ 引入依赖

```xml
<!--openfeign-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

+ YML配置

```yml
server:
  port: 80

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
spring:
  application:
    name: feign-consumer-order80
```



+ 启动类上面标注`@EnableFeignClients`开启OpenFengin
+ 消费者微服务中编写远程调用接口

```java
//里面值填写的远程调用生产者的微服务名，
//可以不用写@Component注解将组件加入容器，@FeignClient自动将接口动态代理生成子类加入容器中
@FeignClient("cloud-payment-provider-8001")
public interface PaymentServiceRemote {

  /**
   * 要求和远程调用生产者的controller响应的控制器方法签名一致，注意@RequestMapping的路径写全
   * 在SpringMVC中@PathVariable，@RequestParam等注解的值符合条件可以省略，在feign中一律不能省略
   */
  @GetMapping("/payment/get/{id}")
  public CommonResult<Payment> queryPayment(@PathVariable("id") Long id);
}
```

+ 消费者的Controller调用远程接口

```java
@RestController
public class FeignOrderController {
  @Autowired
  private PaymentServiceRemote paymentServiceRemote;

  @GetMapping("/feign/consumer/{id}")
  public CommonResult<Payment> getPayment(@PathVariable("id") Long id){
    return paymentServiceRemote.queryPayment(id);
  }
}
```





# OpenFeign超时控制

当消费者Feign客户端调用生产者接口，默认Feign客户端只等待一秒钟， 但是如果服务端处理需要超过默认时间1秒钟，就会导致Feign客户端不想等待了，直接返回报错。为了避免这样的情况，有时候我们需要设置消费者Feign客户端的超时控制。

![image-20200528182918022](https://gitee.com/little_broken_child_9527/images/raw/master/20200528182919.png)

```yml
server:
  port: 80

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
spring:
  application:
    name: feign-consumer-order80

ribbon:
  # 指的是建立连接所用的时间,适用于网络状态正常的情况下,两端连接所用的时间
  ReadTimeout: 5000
  # 指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000
```



# OpenFeign日志打印功能

Feign提供了日志打印功能，我们可以通过配置来调整日志级别，从而了解Feign中的http请求的细节。说白的就是对Feign接口的调用情况进行监控和输出。

NONE：默认的，不示任何志; 
BASIC：仅记录请求方法、URL、 响应状态码及执行时间;
HEADERS：除了BASIC定义的信息之外,还有请求和响应的头信息;
FULL：除了HEADERS中定义的信息之外，还有请求和响应的正文及元数据。



配置步骤

+ 配置文件开启日志打印，配置日志框架监控消费者远程调用接口

```yml
server:
  port: 80

eureka:
  client:
    register-with-eureka: true
    fetch-registry: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
spring:
  application:
    name: feign-consumer-order80

ribbon:
  # 指的是建立连接所用的时间,适用于网络状态正常的情况下,两端连接所用的时间
  ReadTimeout: 5000
  # 指的是建立连接后从服务器读取到可用资源所用的时间
  ConnectTimeout: 5000

logging:
  level:
    com.whpu.springcloud.client: debug
 
```



+ 编写Feign日志打印配置类

```java
@Configuration
public class FeignConfig {

    /**
     * feignClient配置日志级别
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        // 请求和响应的头信息,请求和响应的正文及元数据
        return Logger.Level.FULL;
    }
}
```

+ 控制台输出的Feign远程调用详细过程

![image-20200528183423799](https://gitee.com/little_broken_child_9527/images/raw/master/20200528183425.png)



























