# 简介

Spring Cloud Bus能管理和传播分布式系统间的消息,就像-个分布式执行器， 可用于广播状态更改、事件推送等,也可以当作微服务间的通信通道。Spring Cloud Bus配合Spring Cloud Config使用可以实现配置的动态刷新。Spring Cloud Bus是用来将分布式系统的节点与轻量级消息系统链接起来的框架，它整合了Java的事件处理机制和消息中间件的功能。Spring Clud Bus目前支持RabbitMQ和Kafka。**Bus支持两种消息代理：RabbitMQ和Kafka**







什么是总线？
在微服务架构的系统中，通常会使用轻量级的消息代理来构建一个共用的消息主 题，让系统中所有微服务实例都连接上来。于该注题中产生的消息会被所有实例监听和消费,所以称它为消息总线。在总线上的各个实例，都可以方便地广播-些要让其他连接在该主题上的实例都知道的消息。


https://www.bilbili.com/video/av55976700?from=search&seid=15010075915728605208





# SpringCloud  Bus动态刷新方式



## 方式一

利用消息总线触发一个客户端/bus/refresh，而刷新所有客户端的配置





![image-20200604174501522](https://gitee.com/little_broken_child_9527/images/raw/master/20200604174509.png)

![image-20200812235502056](https://gitee.com/little_broken_child_9527/images/raw/master/20200812235503.png)

## 方式二

利用消息总线出发一个客户端Config Server的bus/refresh端点，而刷新所有客户端的配置

![image-20200604174756470](https://gitee.com/little_broken_child_9527/images/raw/master/20200812232506.png)

方式二的工作方式比方式一更加适合，原因如下：

+ 方式一打破了微服务的职责单一性,因为微服务本身是业务模块,它本不应该承担配置刷新的职责。
+ 方式一破坏了微服务各节点的对等性。
+ 方式一有一定的局限性。例如，微服务在迁移时，它的网络地址常常会发生变化，此时如果想要做到自动刷新,那就会增加更多的修改



# 方式二配置

## 配置中心+消息总线server端

+ 在spring config配置中心的server端增加bus消息总线依赖

```xml
<!--        配置中心server端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>com.whpu</groupId>
    <artifactId>cloud-api-common</artifactId>
    <version>${project.version}</version>
</dependency>
<!-- 消息总线配置-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<!--监控-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

+ **application.yml配置文件新增配置**

```yml
server:
  port: 3344

spring:
  application:
    name: cloud-config-center
  cloud:
    config:
      server:
        git:
          #github仓库地址，为什么使用SSH无法成功？
#          uri: git@github.com:murongdongfang/SpringCloud-Config.git
          uri: https://github.com/murongdongfang/SpringCloud-Config.git
          search-paths:
            - SpringCloud-Config  #github仓库名
#          skip-ssl-validation: true
      label: master
  #增加消息总线配置
  rabbitmq:
    host: localhost
    port: 5672 #客户端通过浏览器访问的端口是15672，rabbit运行在5672端口
    username: guest
    password: guest

#增加消息总线配置
#暴露配置端点，使得可以通知client或者server动态刷新
management:
  endpoints:
    web:
      exposure:
#运维发送响应的刷新命令的时候server就会刷新配置，并且通过消息总线让client也从github更新配置信息
        include: 'bus-refresh' 


eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:7001/eureka


```

+ spring config的server端启动类

```java
@SpringBootApplication
@EnableConfigServer
@EnableEurekaClient
public class App_config_3344 {
  public static void main(String[] args) {
    SpringApplication.run(App_config_3344.class,args);
  }
}
```



## 配置中心+消息总线client端

+ pom依赖

```xml
<!--springcloud配置中心客户端-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<!--        消息总线-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bus-amqp</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!--springcloud监控支持-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

+ **`bootstrap.yml`配置文件，配置中心client配置文件，配置中心server端配置文件是application.yml**

```yml
server:
  port: 3355
spring:
  application:
    name: config-client
  cloud:
    config:
      #分支名
      label: master
      name: config #配置文件名称
      #配置中心客户端配置信息。读取远程仓库github配置信息
      #配置文件后缀名 读取的是远程仓库中master分支的config-dev配置文件
      #实际上uri+label+name+profile合起来就是config client访问config server，http://localhost:3344/master/config-dev.yml来读取config-dev配置文件
      profile: dev
      #配置
      uri: http://localhost:3344
    #消息总线配置，server更新配置信息之后会通过消息总线通知client也更新配置信息
    rabbitmq:
      host: 127.0.0.1
      port: 5672 #客户端通过浏览器访问的端口是15672，rabbit运行在5672端口
      username: guest
      password: guest
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka
# 暴露监控端点，做到客户端config的动态刷新
management:
  endpoints:
    web:
      exposure:
        include: "*"

```

+ client 启动类

```java
@SpringBootApplication
@EnableEurekaClient
public class App_config_client_3355 {
  public static void main(String[] args) {
    SpringApplication.run(App_config_client_3355.class,args);
  }
}
```

+ controller信息

```java
@RestController
//要想动态刷新远程仓库的配置信息必须要增加这个类
@RefreshScope
public class ConfigController {
  @Value("${config.info}")
  private String configInfo;

  @GetMapping("/info")
  public String getConfigInfo(){
    return this.configInfo;
  }
}
```



+ 在github上面修改最新的配置文件信息，版本要修改为5

![image-20200813234109146](https://gitee.com/little_broken_child_9527/images/raw/master/20200813234118.png)



+ 查看配置中心server端信息

![image-20200813234252593](https://gitee.com/little_broken_child_9527/images/raw/master/20200813234511.png)

+ 查看配置中心client端信息

![image-20200813234457502](https://gitee.com/little_broken_child_9527/images/raw/master/20200813234512.png)



**给配置中心的server端发送刷新请求，配置中心server端更新github仓库上的配置之后通过消息总线通知其他的client端也都自动刷新配置，做到了一次刷新，处处生效。注意：必须要给配置中心的server端发送post请求，浏览器无法模拟post请求，只能通过工具或者命令**

`curl -X POST "http://localhost:3344/actuator/bus-refresh" `



发送post刷新命令给配置中心server端后再次查看配置中心client端，无需重启，发现此时配置已经更新了

![image-20200813235215304](https://gitee.com/little_broken_child_9527/images/raw/master/20200813235217.png)



# 基本原理
ConfigClient实例都监听MQ中同一个topic(默认是springCloudBus)。 当一个服务刷新数据的时候,它会把这个信息放入到Topic中，这样其它监听
同一Topic的服务就能得到通知，然后便新自身的配置。

![image-20200814000230957](https://gitee.com/little_broken_child_9527/images/raw/master/20200814000232.png)





# SpringCloud Bus动态刷新定点通知



发送post刷新请求与给配置中心的server的时候，指定具体某一个实例生效而不是全部。不想全部通知，只想定点通知，例如只通知3355，不通知3366。

公式：http://localhost:配置中心的端口号/actuator/bus-refresh/{destination}【destination = client端在注册中心服务名+端口号】

`curl -X POST "http://localhost:3344/actuator/bus-refresh/config-client:3355`

此时配置中心的client端只有3355这个端点能够动态刷新远程仓库的配置，3366这个client端不能动态刷新





























