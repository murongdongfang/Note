# 简介

## 服务雪崩
多个微服务之间调用的时候，假设微服务A调用微服务B和微服务C，微服务B和微服务C又调用其它的微服务，这就是所谓的扇出” 。如果链路上某个微服务的调用响应时间过长或者不可用，对微服务A的调用就会占用越来越多的系统资源,进而弓|系统崩溃。所谓的"雪崩效应”，对于高流量的应用来说，单一的后端依赖可能会导致所有服务器上的所有资源都在几秒钟内饱和。比失败更糟糕的是,这些应用程序还可能导致服务之间的延迟增加，备份队列，线程和其他系统资源紧张，导致整个系统发生更多的级联故障。这些都表示需要对故障和延迟进行隔离和管理，以便单个依赖关系的失败,不能取消整个应用程序或系统。所以，通常当你发现一 个模块下的某个实例失败后,这时候这个模块依然还会接收流量,然后这个有问题的模块还调用了其他的模块，这样就会发生级联故障，或者叫雪崩。



## Hystrix作用

Hystrix是一个用于 处理分布式系统的**延迟**和**容错**的开源库,在分布式系统里，许多依赖不可避免的会调用失败,比如超时、异常等。**Hystrix能够保证在一个依赖出问题的情况下， 不会导致整体服务失败,避免级联故障,以提高分布式系统的弹性。**

”断路器”身是-种开关装置, 当某个服务单元发生故障之后，通过断路器的故障监控(类似熔断保险丝) ，**向调用方返回一个符合预期的、可处理的备选响应(FallBack) ，而不是长时间的等待或者抛出调用方无法处理的异常**，这样就保证了服务调用方的线程不会被长时间、不必要地占用，从而避免了故障在分布式系统中的蔓延，乃至雪崩。

[github地址](https://github.com/Netflix/hystrix/wiki)

[马丁富勒熔断器理论](https://martinfowler.com/bliki/CircuitBreaker.html)



## 重要概念

### 服务熔断(break)

类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示



### 服务降级(fall back)

服务器忙,请稍后再试，不让客户端等待并立刻返回一个友好提示,fallback

以下几种情况可能出现服务降级

+ 程序运行异常
+ 服务熔断触发服务降级
+ 线程池/信号量也会导致服务降级
+ 超时

### 服务限流(flowlimit)

秒杀高并发等操作,严禁一窝蜂的过来拥挤,大家排队,一秒钟N个,有序进行



Hystrix既可以放在生产者也可以放到消费者，OpenFeign只能放到消费者。

# Hystrix服务降级

## 生产者服务降级

+ 引入相关依赖

```xml
  <!--hystrix-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```



+ 相应的service层配置服务降级，如果访问当前生产者的相应方法出现异常或者是调用超时配置相应的容错处理。

```java
@Service
public class PaymentServiceImpl implements PaymentService {
  /**
   * 超时访问
   * 如果paymentInfo_TimeOut方法访问出现异常或者超过了@HystrixProperty配置的时间(3秒)
   * 就会调用fallbackMethod配置的paymentInfo_TimeOutHandler方法进行容错处理
   */
  @Override
  @HystrixCommand(fallbackMethod = "paymentInfo_TimeOutHandler",commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000")
  })
  public String paymentInfo_TimeOut(Integer id) {
    int timeNumber = id;
//    int i = 1/0;
    try {
      // 暂停3秒钟
      TimeUnit.SECONDS.sleep(timeNumber);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
    return "线程池:" + Thread.currentThread().getName() + " paymentInfo_TimeOut,id:" + id + "\t" +
      "O(∩_∩)O哈哈~  耗时(秒)" + timeNumber;
  }

  //对paymentInfo_TimeOut进行容错处理，paymentInfo_TimeOut超时或者异常就会回调这个方法
  public String paymentInfo_TimeOutHandler(Integer id){
    String res = "线程池:" + Thread.currentThread().getName() + " paymentInfo_TimeOutHandler提示：系统异常或者超时，请稍后再试";
    return res;
  }
}

```

+ 主启动类上标注`@EnableCircuitBreaker`开启服务降级



## 消费者服务降级

+ 在消费者的Controller中配置服务降级，配置和生产者服务降级类似。此时如果远程调用就不会再向以前那样报`read time out`错误，而是会执行fallbackMethod配置的回调方法。

```java
@RestController
@RequestMapping("/hystrix")
public class HystrixOrderController {
  @Autowired
  private PaymentRemoteService paymentRemoteService;
    
  @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
  })
  @GetMapping("/order/timeout/{id}")
  public String paymentInfo_TimeOut(@PathVariable("id") Integer id){
//    int i = 1/0;
    String res = paymentRemoteService.paymentInfo_TimeOut(id);
    return res;

  }
  //对paymentInfo_TimeOut进行容错处理，paymentInfo_TimeOut超时或者异常就会回调这个方法
  public String paymentTimeOutFallbackMethod(Integer id){
    String res = "消费者Order80，线程池:" + Thread.currentThread().getName() + " paymentInfo_TimeOutHandler提示：系统异常或者超时，请稍后再试";
    return res;
  }


}
```

+ 消费者使用feign调用生产者接口，需要在配置文件的feign中开启Hystrix

```yml
server:
  port: 80

spring:
  application:
    name: consumer-feign-hystrix-order80
eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka,http://eureka7002.com:7002/eureka
feign:
  hystrix:
    enabled: true

```



+ 消费者的主启动类标注`@EnableCircuitBreaker`



**使用Hystrix按照上述的配置在生产者和消费者两端进行服务降级有两个缺点**

+ **业务代码和fallbackMethod方法耦合在一起**
+ **每一个方法都需要单独配置一个fallbackMethod太过麻烦。**

为了解决每一个方法都需要单独配置一个fallbackMethod太过麻烦的问题，可以使用全局服务降级

## 全局服务降级

+ 在消费者的Controller类上面标注``@DefaultProperties`配置全局的降级处理方法

```java
@RestController
@RequestMapping("/hystrix")
@DefaultProperties(defaultFallback = "globalFallMethod",
  commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "2000")
})
public class HystrixOrderController {
  @Autowired
  private PaymentRemoteService paymentRemoteService;

  @GetMapping("/order/ok/{id}")
  @HystrixCommand
  //标注了@HystrixCommand却没有标注属性的方法就会执行@DefaultProperties配置的属性
  public String paymentInfo_Ok(@PathVariable("id") Integer id){
//    int i = 1/0;
    //线程休眠
    try{ TimeUnit.SECONDS.sleep(id); } catch (Exception e) { e.printStackTrace(); }
    
    String res = paymentRemoteService.paymentInfo_Ok(id);
    return res;
  }

  public String globalFallMethod(){
    String res = "全局服务降级配置，线程池:" + Thread.currentThread().getName() + " globalFallMethod：系统异常或者超时，请稍后再试";
    return res;
  }
    
    
  //@HystrixCommand里边配置了属性的就会使用自定义降级处理，不会使用全局的降级配置
  @HystrixCommand(fallbackMethod = "paymentTimeOutFallbackMethod",commandProperties = {
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "1500")
  })
  @GetMapping("/order/timeout/{id}")
  public String paymentInfo_TimeOut(@PathVariable("id") Integer id){
//    int i = 1/0;
    String res = paymentRemoteService.paymentInfo_TimeOut(id);
    return res;

  }
  //对paymentInfo_TimeOut进行容错处理，paymentInfo_TimeOut超时或者异常就会回调这个方法
  public String paymentTimeOutFallbackMethod(Integer id){
    String res = "消费者Order80，线程池:" + Thread.currentThread().getName() + " paymentInfo_TimeOutHandler提示：系统异常或者超时，请稍后再试";
    return res;
  }

}
```

+ 消费者的配置文件feign'远程调用开启hystrix

```yml
feign:
  hystrix:
    enabled: true
```

+ 消费者的主启动类标注`@EnableCircuitBreaker`

 生产者的全局降级配置方法类似，只不过不需要配置feign中开启hystrix服务降级罢了



## 服务降级终极解决方案

需要注意的是全局服务降级配置虽然解决了单独为每一个方法配置一个服务降级的问题，但是还是没有解决服务降级的回调方法和业务方法耦合在一起的问题。**上边的配置无论是在生产者还是消费者中都可以配置服务降级，相当于微服务下的异常处理。如果只是专注于消费者远程调用的服务降级（异常处理），需要做到的做到的是异常处理和业务不能耦合。此时需要在远程调用接口中做文章，具体做法就是为远程调用接口提供一个实现类，实现类的重写方法相当于相应远程接口方法出现异常的回调，每当远程调用的接口方法出现异常或者超时的时候就会执行这个远程接口实现类中相应的重写方法。**

+ 在远程调用接口`@FeignClient`中配置方法出现异常或者超时时候的容错处理方法，其中fallback属性指定的类必须要实现当前远程调用接口

```java
@FeignClient(value = "hystrix-payment",fallback = PaymentRemoteFallback.class )
public interface PaymentRemoteService {
  @GetMapping("/hystrix/ok/{id}")
  public String paymentInfo_Ok(@PathVariable("id") Integer id);
  @GetMapping("/hystrix/timeout/{id}")
  public String paymentInfo_TimeOut(@PathVariable("id") Integer id);
}
```

+ 容错处理类必须实现远程调用接口，并加入IOC容器。如果远程调用的接口中的方法出现错误就会回调接口实现类的重写方法。例如PaymentRemoteService接口中的paymentInfo_Ok出现异常。就会回调PaymentRemoteFallback中的paymentInfo_Ok方法

```java
@Component
public class PaymentRemoteFallback implements PaymentRemoteService {

  @Override
  public String paymentInfo_Ok(Integer id) {
    String res = "PaymentRemoteFallback服务熔断终极解决方案， paymentInfo_Ok";
    return res;
  }

  @Override
  public String paymentInfo_TimeOut(Integer id) {
    String res = "PaymentRemoteFallback服务熔断终极解决方案， paymentInfo_TimeOut";
    return res;
  }
}
```

+ 消费者的配置文件feign'远程调用开启hystrix

```yml
feign:
  hystrix:
    enabled: true
```

+ 消费者的主启动类标注`@EnableCircuitBreaker`



# Hystrix熔断

熔断机制是应对雪崩效应的一种微服务链路保护机制。当扇出链路的某个微服务出错不可用或者响应时间太长时，会进行服务的降级，进而熔断该节点微服务的调用,快速返回错误的响应信息。**当检测到该节点微服务调用响应正常后，恢复调用链路。**在Spring Cloud框架里,熔断机制通过Hystrix实现。Hystrix会监控微服务间调用的状况,当失败的调倒-定阈值,缺省是5秒内20次调用失败,就会启动熔断机制。**熔断机制的注解是@HystrixCommand**

类比保险丝达到最大服务访问后，直接拒绝访问，拉闸限电，然后调用服务降级的方法并返回友好提示。服务的降级->进而熔断->恢复调用链路

##  Hystrix断路器配置



![image-20200530104907926](https://gitee.com/little_broken_child_9527/images/raw/master/20200530104917.png)





当输入的数值负数的时候就进行降级，输入的正数正常运行。

下面配置的意思就是开启熔断`enabled`，请求次数`requestVolumeThreshold`10次中如果失败率达到`errorThresholdPercentage`60%，就开启熔断。也就是10次请求中如果有60%的请求都进行了服务降级，此时就会熔断。熔断之后下次即使输入的是正数也会进行降级处理，直到服务慢慢恢复正常关闭熔断，链路正常时候输入正数才会进行正常响应



+ 在service方法上进行服务熔断相关的配置` @HystrixCommand`里边的详细配置项可以查看`HystrixCommandProperties`，下面配置的意思就是最近10S内10个请求中有六个请求服务降级就开启熔断

```java

@Service
public class PaymentServiceImpl implements PaymentService {

  //服务熔断，里边的具体配置查看HystrixCommandProperties类
  @Override
  @HystrixCommand(
    fallbackMethod = "paymentCircuitBreak_fallback",
      //最近10S内10个请求中有六个请求服务降级就开启熔断
    commandProperties = {
      @HystrixProperty(name="circuitBreaker.enabled",value = "true"),//是否开启断路器
      @HystrixProperty(name="circuitBreaker.requestVolumeThreshold",value = "10"),//请求次数
      @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds",value = "10000"),//时间窗口期
      @HystrixProperty(name="circuitBreaker.errorThresholdPercentage",value = "60"),//失败率达到多少后跳闸
    }
  )
  public String paymentCircuitBreak(@PathVariable("id") Integer id) {
    if (id < 0) {
      throw new RuntimeException("******id  不能为负数");
    }
    String serialNumber = IdUtil.simpleUUID();
    return Thread.currentThread().getName() + "\t" + "调用成功，流水号：" + serialNumber;
  }

  public String paymentCircuitBreak_fallback(@PathVariable("id") Integer id){

    return "id 不能负数，稍后再试，id："+id;
  }
}

```

+ 主启动类标注`@EnableCircuitBreaker`

**Hystrix熔断配置涉及到三个重要参数:快照时间窗、请求总数阀值、错误百分比阀值。**

+ `circuitBreaker.sleepWindowInMilliseconds`快照时间窗：断路器确定否打开需要统计一些请求和错误数据， 而统计的时间范围就是快照时间窗，默认为最近的10秒。
+ `circuitBreaker.requestVolumeThreshold`请求总数阀值：在快照时间窗内，必须满足请求总数阀值才有资格熔断。默认为20, 意味着在10秒内,如果该hystrix命令的调用次数不足20次，即使所有的请求都超时或其他原因失败，断路器都不会打开。
+ `circuitBreaker.errorThresholdPercentage`错误百分比阀值：当请求总数在快照时间窗内超过了阀值,比如发生了30次调用,如果在这30次调用中,有15次发生了超时异常，也就是超过50%的错误百分比，在默认设定50%阀值情况下,这时候就会将断路器打开。



## 断路器工作流程

[Hystrix官网工作流程](https://github.com/Netflix/Hystrix/wiki/How-it-Works)

**熔断器的三种状态**

熔断打开：进入熔断之后，有请求调用的时候，将不会调用主逻辑，而是直接调用降级fallback通过断路器，实现了自动地发现错误并将降级逻辑切换为主逻辑，减少响应延迟的效果。

熔断关闭：熔断关闭后不会对服务进行熔断

熔断半开：请求不再调用当前服务，内部设置一般为MTTR(平均故障处理时间)，当打开长达导所设时钟则进入半熔断状态。部分请求根据规则调用当前服务，如果请求成功且符合规则则认为当前服务恢复正常，关闭熔断。

![image-20200530114255111](https://gitee.com/little_broken_child_9527/images/raw/master/20200530114256.png)

**工作步骤**

1. 当满足一定的阈值的时候(默认10秒钟超过20个请求次数)
2. 当失败率达到一定的时候(默认10秒内超过50%的请求次数)
3. 到达以上阈值,断路器将会开启
4. 当开启的时候，所有请求都不会进行转发
5. 一段时间之后(默认5秒),这个时候断路器是半开状态,会让其他一个请求进行转发. 如果成功,断路器会关闭,若失败,继续开启.重复4和5

![image-20200530115309757](https://gitee.com/little_broken_child_9527/images/raw/master/20200530115311.png)



**原来的主逻辑要如何恢复呢？**
对于这一问题，hystrix也为我们实现了自动恢复功能。当断路器打开,对主逻辑进行熔断之后，hystrix会启动一个休眠时间窗,在这个时间窗内，降级逻辑是临时的成为主逻辑，当休眠时间窗到期，断路器将进入半开状态，释放一次请求到原来的主逻辑 上,如果此次请求正常返回，那么断路器将继续闭合，主逻辑恢复，如果这次请求依然有问题，断路器继续进入打开状态,休眠时间窗重新计时。Spring Cloud Hystrix Dashboard的底层原理是间隔一定时间去“Ping”目标服务，返回的结果是最新的监控数据，最后将数据显示出来。必须具备两个条件。第一，服务本身有对Hystrix做监控统计(spring-cloud-starter-hystrix启动依赖)；第二，暴露hystrix.stream端口(spring-boot-starter-actuator启动依赖)。所以需要确保目标服务引入如下相应的依赖。





# 服务熔断和服务降级的区别

服务熔断：在微服务架构中，微服务之间的数据交互通过远程调用完成，微服务A调用微服务B和微服务C，微服务B和微服务C又调用其它的微服务，此时如果链路上某个微服务的调用响应时间过长或者不可用，那么对微服务A的调用就会占用越来越多的系统资源，进而引起系统崩溃，导致“雪崩效应”。服务熔断是应对雪崩效应的一种微服务链路保护机制。例如在高压电路中，如果某个地方的电压过高，熔断器就会熔断，对电路进行保护。同样，在微服务架构中，熔断机制也是起着类似的作用。当调用链路的某个微服务不可用或者响应时间太长时，会进行服务熔断，不再有该节点微服务的调用，快速返回错误的响应信息。当检测到该节点微服务调用响应正常后，恢复调用链路。



服务降级：当服务器压力剧增的情况下，根据实际业务情况及流量，对一些服务和页面有策略的不处理或换种简单的方式处理，从而释放服务器资源以保证核心业务正常运作或高效运作。说白了，就是尽可能的把系统资源让给优先级高的服务。资源有限，而请求是无限的。如果在并发高峰期，不做服务降级处理，一方面肯定会影响整体服务的性能，严重的话可能会导致宕机某些重要的服务不可用。所以，一般在高峰期，为了保证核心功能服务的可用性，都要对某些服务降级处理。比如当双11活动时，把交易无关的服务统统降级，如查看蚂蚁深林，查看历史订单等等。服务降级主要用于什么场景呢？当整个微服务架构整体的负载超出了预设的上限阈值或即将到来的流量预计将会超过预设的阈值时，为了保证重要或基本的服务能正常运行，可以将一些 不重要 或 不紧急 的服务或任务进行服务的 延迟使用 或 暂停使用。降级的方式可以根据业务来，可以延迟服务，比如延迟给用户增加积分，只是放到一个缓存中，等服务平稳之后再执行 ；或者在粒度范围内关闭服务，比如关闭相关文章的推荐。

**区别**

+ 触发原因不太一样，服务熔断一般是某个服务（下游服务）故障引起，而服务降级一般是从整体负荷考虑；
+ 管理目标的层次不太一样，熔断其实是一个框架级的处理，每个微服务都需要（无层级之分），而降级一般需要对业务有层级之分（比如降级一般是从最外围服务开始）
+ 实现方式不太一样，服务降级具有代码侵入性(由控制器完成/或自动降级)，熔断一般称为自我熔断。

# HystrixDashboard服务监控

除了隔离依赖服务的调用以外，Hystrix还提供了准实时的调用监控(Hystrix Dashboard)，Hystrix会持续地记录所有通过Hystrix发起的请求的执行信息，以统计报表和图形的形式展示给用户，包括每秒执行多少请求多少成功，多少失败等。Netflix通过hystrix-metrics-event-stream项目实现了对以上指标的监控。Spring Cloud也提供了Hystrix Dashboard的整合,对监控内容转化成可视化界面。



+ HystrixDashboard监控微服务引入依赖，不止是Hystrix，**所有的组件监控功能都需要依赖SpringBoot的`spring-boot-starter-actuator`**

```xml
  <!--hystrix dashboard-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!---->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

+ 主启动上标注注解`@EnableHystrixDashboard`

```java
@SpringBootApplication
@EnableHystrixDashboard
public class App_Hystrix_Dashbord9001 {
  public static void main(String[] args) {
    SpringApplication.run(App_Hystrix_Dashbord9001.class,args);
  }
}
```



+ **SpringCloud升级之后有个坑，必须要在被监控的微服务中注册一个Servlet这个微服务才能被监控**，否则出现`Unable to connect to Command Metric Stream.`，注意这个被Hystrix监控的微服务必须开启`@EnableCircuitBreaker`

```java
@SpringBootApplication
@EnableEurekaClient
@EnableDiscoveryClient
@EnableCircuitBreaker
public class App_hystrix_payment8001 {
  public static void main(String[] args) {
    SpringApplication.run(App_hystrix_payment8001.class,args);
  }

  /**
   * 此配置是为了服务监控而配置，与服务容错本身无观，springCloud 升级之后的坑
   * ServletRegistrationBean因为springboot的默认路径不是/hystrix.stream
   * 只要在自己的项目中配置上下面的servlet即可
   * @return
   */
  @Bean
  public ServletRegistrationBean getServlet(){
    HystrixMetricsStreamServlet streamServlet = new HystrixMetricsStreamServlet();
    ServletRegistrationBean<HystrixMetricsStreamServlet> registrationBean = new ServletRegistrationBean<>(streamServlet);
    registrationBean.setLoadOnStartup(1);
    registrationBean.addUrlMappings("/hystrix.stream");
    registrationBean.setName("HystrixMetricsStreamServlet");
    return registrationBean;
  }
}
```



+ Dashboard微服务的配置文件中无需其余的配置，只需要配置以下端口号即可



访问http://localhost:9001/hystrix

配置需要被监控的微服务地址：http://localhost:8001/hystrix.stream

配置需要被监控的微服务

![image-20200530121651733](https://gitee.com/little_broken_child_9527/images/raw/master/20200530121653.png)

![image-20200530130608206](https://gitee.com/little_broken_child_9527/images/raw/master/20200530130609.png)



![image-20200530124704283](https://gitee.com/little_broken_child_9527/images/raw/master/20200530124705.png)



![markdown-img-paste-20190616181619375.cd4abf54](https://gitee.com/little_broken_child_9527/images/raw/master/20200530125002.png)

仪表盘说明：

+ 七种颜色：每一种颜色代表一种错误
+ 实心圆：共有两种含义。它通过颜色的变化代表了实例的健康程度，它的健康度从绿色<艴<橙艴<红绝递减。
  该实心圆除了颜色的变化之外,它的大小也会根据实例的请求流量发生变化，流量越大该实心圆就越大。所以通过该实心圆的展示，就
  可以在大量的实例中快速的发现故障实例和高压力实例。
+ 曲线：用来记录2分钟内流量的相对变化，可以通过它来观察到流量的升和下降趋势。







