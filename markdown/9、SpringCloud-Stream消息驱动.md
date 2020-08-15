# 简介

**什么是SpringCloudStream？**
官方定义**Spring Cloud Stream是一个构建消息驱动微服务的框架**。应用程序通过inputs（消息生产者）或者outputs（消息消费者）来和Spring Cloud Stream中binder对象交互。通过我们配置来binding(绑定)，而Spring Cloud Stream的binder对象负责与消息中间件交互。所以，我们只需要搞清楚如何与Spring Cloud Stream交互就可以方便使用消息驱动的方式。通过使用Spring Integration来连接消息代理中间件以实现消息事件驱动。Spring Cloud Stream为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅、消费组、分区的三个核心概念。**目前仅支持RabbitMQ、Kafka。一句话Spring Cloud Stream就是用来蔽底层消息中间件的差异，降低切换版本，统一消息的编程模型。相当于SpringData JPA就是用来屏蔽底层数据库的差异，提供统一的数据库访问抽象模型一样**





**官网地址**

https://spring.io/projects/spring-cloud-stream#overview

https://cloud.spring.io/spring-cloud-static/spring-cloud-stream/3.0.1.RELEASE/reference/html/

https://m.wang1314.com/doc/webapp/topic/20971999.html



**举个栗子**

比方说我们用到了RabbitMQ和Kafka，由于这两个消息中间件的架构上的不同，像RabbitMQ有exchange, kafka有Topic和Partitions分区。这些中间件的差异性导致我们实际项目开发给我们造成了-定的困扰，我们如果用了两个消息队列的其中-种, 后面的业务需求,我想往另外-种消息队列进行迁移,这时候无疑就是一个灾难性的， 一大堆东西都要重新推倒重新做,因为它跟我们的系统耦合了，这时候springcloud Stream给我们提供了一种解耦合的方式。由于各消息中间件构建的初衷不同，它们的实现细节上会有较大的差异性，通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。通过向应用程序暴露统-的Channel通道， 使得应用程序不需要再考虑各种不同的消息中间件实现。
通过定义绑定器Binder作为中间层，实现了应用程序与消息中间件细节之间的隔离。

![image-20200814224358904](https://gitee.com/little_broken_child_9527/images/raw/master/20200814224407.png)

# 设计架构



在没有绑定器（binder）这个概念的情况下，我们的SpringBoot应用要直接 与消息中间件进行信息交互的时候，由于各消息中间件构建的初衷不同，它们的实现细节上会有较大的差异性，通过定义绑定器作为中间层，完美地实现了应用程序与消息中间件细节之间的隔离。Stream对消息中间件的进一 步封装，可以做到代码层面对中间件的无感知，甚至于动态的切换中间件(rabbitmq切换为kafka)，使得微服务开发的高度解耦，服务可以关注更多自己的业务流程。**通过定义绑定器Binder作为中间层，实现了应用程序与消息中间件细节之间的隔离。**



![image-20200814225341391](https://gitee.com/little_broken_child_9527/images/raw/master/20200814225342.png)











# 重要概念

绑定器：很方便的连接中间件，屏蔽差异

Channel：通道，是队列Queue的一种抽象，在消息通讯系统中就是实现存储和转发的媒介，通过对Channel对队列进行配置

Source和Sink：简单的可理解为参照对象是Spring Cloud Stream自身，从Stream发布消息就是输出，接受消息就是输入



![image-20200814225656644](https://gitee.com/little_broken_child_9527/images/raw/master/20200814225658.png)



![image-20200814230137386](https://gitee.com/little_broken_child_9527/images/raw/master/20200814230138.png)



# 编码

![image-20200814225614888](https://gitee.com/little_broken_child_9527/images/raw/master/20200814225615.png)

## 生产者

+ 引入依赖，如果使用的消息中间件是kafka此时需要引入`spring-cloud-starter-stream-kafka`

```xml
<!--springcloud消息驱动组件-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
<!--eureka服务注册client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>com.whpu</groupId>
    <artifactId>cloud-api-common</artifactId>
    <version>${project.version}</version>
</dependency>
<!--        web启动器-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--监控组件-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

+ 配置文件，生产者的binders里边配置output

```yml
server:
  port: 8801

spring:
  application:
    name: cloud-stream-provider
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        output: # output消息生产者，这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义,此时在rabbitmq中有一个exchange为studyExchange
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: defaultRabbit  # 设置要绑定的消息服务的具体设置

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: send-8801.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址

```

+ 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class App_stream_provider_8801 {
  public static void main(String[] args) {
    SpringApplication.run(App_stream_provider_8801.class, args);
  }
}
```

+ controller

```java
//浏览器发送请求，消息生产者向mq推送消息
@RestController
public class SendMsgController {

  @Autowired
  private SendMsgService sendMsgService;
  @GetMapping("/send/msg")
  public String sendMsg(){
    return sendMsgService.sendMsg();
  }
}
```

+ service

```java
public interface SendMsgService {
  public String sendMsg();
}

//定义消息的推送管道
@EnableBinding(Source.class)
public class SendMsgServiceImpl implements SendMsgService {

  /**
   * MessageChannel这个接口下面有多个实现类bean被注入容器中
   * 要使用的是Output这个bean
   * 也可以使用
   * @Autowired或者@Resource无需加 @Qualifier("output")指定name
   * MessageChannel output
   */
  @Autowired
  @Qualifier("output")
  // 消息发送管道
  private MessageChannel messageChannel;
  @Override
  public String sendMsg() {
    String uuid = UUID.randomUUID().toString();
    //org.springframework.integration.support.MessageBuilder
    messageChannel.send(MessageBuilder.withPayload(uuid).build());

    return uuid;
  }
}
```



## 消费者

+ 引入依赖，如果消息中间件用的是kafka，要引入的消息驱动组件是`spring-cloud-starter-stream-kafka`

```xml
<!--springcloud的消息驱动组件，-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
<!--注册中心-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<!--web启动器-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--监控组件-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!--热部署-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

+ 配置文件，消费者的binding配置成input

```yml
server:
  port: 8802

spring:
  application:
    name: cloud-stream-consumer
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        input: # 这个一个服务的消费者，名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: defaultRabbit  # 设置要绑定的消息服务的具体设置

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: receive-8802.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址

```

+ 启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class App_stream_consumer_8802 {
  public static void main(String[] args) {
    SpringApplication.run(App_stream_consumer_8802.class, args);
  }
}
```

+ 消费者监听消息队列，一旦有消息到达消息队列就获取消息

```java
//监听消息队列，一旦消息到达消息队列就获取消费
@Component
@EnableBinding(Sink.class)
public class ConsumeMsgListener {

  @Value("${server.port}")
  private Integer serverPort;

  @StreamListener(Sink.INPUT)
  public void consumeMsg(Message<String> message){
    System.out.println("消费者一号，----->接受的消息: "+message.getPayload()+" port:"+serverPort);
  }
}

```

和上面新建8802消费者一样，新建8003消费者，配置文件如下。如果生产者发送一个消息，消费者8802和8803都会消费这同一条消息。就会下面的重复消费问题

```yml
server:
  port: 8803

spring:
  application:
    name: cloud-stream-consumer  #和8002一样，消费者集群部署，也可以不一样。也会有重复消费的问题
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        input: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: defaultRabbit  # 设置要绑定的消息服务的具体设置

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: receive-8803.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址

```





# 分组消费和消息持久化

比如在如下场景中，订单系统我们做集群部署，都会从RabbitMQ中获取订单信息，那如果一个订单同时被两个服务获取到,那么就会造成数据错误，我们得避免这种情况。这时我们就可以使用Stream中的消息分组来解决。**注意在Stream中处于同一个group中的多个消费者是竞争关系，就能够保证消息只会被其中一个应用消费一次。不同组是可以全面消费的(重复消费)。**

![image-20200815223132260](https://gitee.com/little_broken_child_9527/images/raw/master/20200815223133.png)

消费者的配置文件里边没有指定分组，Springcloud stream会把消费者分配到不同的组里边去，组名是随机分配的。不同的组可以消费同一条消息，相同的组里边消费者是竞争关系不会存在消息的重复消费的问题。

![image-20200815225019742](https://gitee.com/little_broken_child_9527/images/raw/master/20200815225021.png)

在8802和8803消费者的配置文件中配置，使得两个消费者在同一个组只需要配置`group: consumer  #配置8802，8803消费者在同一个组，避免重复消费问题`

```yml
server:
  port: 8802

spring:
  application:
    name: cloud-stream-consumer
  cloud:
    stream:
      binders: # 在此处配置要绑定的rabbitmq的服务信息；
        defaultRabbit: # 表示定义的名称，用于于binding整合
          type: rabbit # 消息组件类型
          environment: # 设置rabbitmq的相关的环境配置
            spring:
              rabbitmq:
                host: localhost
                port: 5672
                username: guest
                password: guest
      bindings: # 服务的整合处理
        input: # 这个名字是一个通道的名称
          destination: studyExchange # 表示要使用的Exchange名称定义
          content-type: application/json # 设置消息类型，本次为json，文本则设置“text/plain”
          binder: defaultRabbit  # 设置要绑定的消息服务的具体设置
          group: consumer  #配置8802，8803消费者在同一个组，避免重复消费问题

eureka:
  client: # 客户端进行Eureka注册的配置
    service-url:
      defaultZone: http://localhost:7001/eureka
  instance:
    lease-renewal-interval-in-seconds: 2 # 设置心跳的时间间隔（默认是30秒）
    lease-expiration-duration-in-seconds: 5 # 如果现在超过了5秒的间隔（默认是90秒）
    instance-id: receive-8802.com  # 在信息列表时显示主机名称
    prefer-ip-address: true     # 访问的路径变为IP地址


```

<img src="https://gitee.com/little_broken_child_9527/images/raw/master/20200815230924.png" alt="image-20200815230923379" style="zoom:100%;" />



消息的持久化：***消息的持久化：如果给消息消费者配置相应的分组，那么即使当消息消费者宕机时，消息提供者发送消息，在消息消费者启动应用成功之后也能接收到消息提供者发来的消息。通过group属性不仅能够解决消息的重复消费，也能解决消息的持久化问题***

1. 停止8802/8803并去除掉8802的分组group:atguiguA
2. 8801先发送4条信息到rabbitmq
3. 先启动8802，无分组属性配置，后台没有打出来消息
4. 再启动8803，有分组属性配置，后台打出来了MQ上的消息











































