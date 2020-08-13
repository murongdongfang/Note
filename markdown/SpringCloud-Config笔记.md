# Config简介

SpringCloud Config为微服务架构中的微服务提供集中化的外部配置支持I配置服务器为各个不同微服务应用的所有环境提供了一个中
心化的外部配置。

![image-20200602190221838](https://gitee.com/little_broken_child_9527/images/raw/master/20200602190223.png)



## 组成

**SpringCloud Config分为服务端和客户端两部分。服务端也称为分布式配置中心，它是一个独立的微服务应用**， 来连接配置服务器并为客户端提供获取配置信息，加密/解密信息等访问接口，客户端则是通过指定的配置中心来管理应用资源，以及与业务相关的配置内容,在启动的时候从配置中心获取和加载配置信息配置服务器默认采用git来存储配置信息,这样就有助于对环境配置进行版本管理,并呵以通过git客户端工具舫便的管理和访问配置内容



## 作用

1. 集中管理配置文件
2. 不同环境不同配置，动态化的配置更新,分环境部署比如dev/test/prod/beta/release
3. 运行期间动态调整配置，不再需要在每个服务部署的机器上编写配置文件，服务会向配置中心统一拉取配置自己的信息
4. 当配置发生变动时，服务不需要重启即可感知到配置的变化并应用新的配置
5. 将配置信息以REST接口的形式暴露



# 配置Config Server





+ 引入依赖

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-config-server</artifactId>
</dependency>
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

+ 新建GitHub仓库

![image-20200603234549975](https://gitee.com/little_broken_child_9527/images/raw/master/20200603234551.png)

+ 配置yml

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

eureka:
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:7001/eureka


```

+ 主启动类上标注`@EnableConfigServer`

+ **可以有以下几种访问方式**
+ /{label}/{application}-{profile}.yml：`http://localhost:3344/master/config-dev.yml`
  
+ /{application}-{profile}.yml：`http://localhost:3344/config-dev.yml` 默认访问的是配置的label分支
  + /{application}/{profile}[/{label}]：`http://localhost:3344/dev/master`

![image-20200604002619714](https://gitee.com/little_broken_child_9527/images/raw/master/20200604002620.png)

# 配置Config Client

app1icaiton. yml是用户级的资源配置项，bootstrap. ym系统级的，优先级更加高。Spring Cloud会创建一个"Bootstrap Context"，作为Spring应用的Application Context的父上下文。初始化的时候，BootstrapContext'债从外部源载配置属性并解析配置。这两个上下文共享一个从外部获取的Environment。Bootstrap属性有高优先级，默认情况下，它们不会被本地配置覆盖。Bootstrap context和Application Context有着不同的约定，所以新增了一个bootstrap.ymI文件，保证Bootstrap Context和Application Context配置的分离。要将Client模块下的application.ym|文件改为bootstrap.yml,这是很关键的，因为bootstrap.ym|是比application.yml先加载的。bootstrap.yml优先级高 于application.yml。**一句话：自己的微服务应用先加载远程配置（github）的公有配置，在加载微服务独有的配置。**



+ 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-client</artifactId>
</dependency>
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

+ 配置yml

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
      #配置文件后缀名 读取的是远程仓库中master分支的config-dev配置文件
      #本质上uri+label+name+profile合起来就是config client访问config server，http://localhost:3344/master/config-dev.yml来读取config-dev配置文件
      profile: dev
      #配置server端的地址+端口号
      uri: http://localhost:3344
eureka:
  client:
    service-url:
      defaultZone: http://localhost:7001/eureka


```

+ 启动类无需额外标注注解

+ 将远程配置信息通过Rest接口暴露

```java
@RestController
public class ConfigController {
  @Value("${config.info}")
  private String configInfo;

  @GetMapping("/info")
  public String getConfigInfo(){
    return this.configInfo;
  }
}
```

![image-20200604002657901](https://gitee.com/little_broken_child_9527/images/raw/master/20200604002659.png)





# 客户端动态刷新

1. 将GitHub仓库上面的配置文件修改

![image-20200604003102222](https://gitee.com/little_broken_child_9527/images/raw/master/20200604003103.png)

2. 不重启微服务访问springcloud-config 的server端，输入`http://localhost:3344/master/config-dev.yml`，发现输出信息也随之修改

![image-20200604003406161](https://gitee.com/little_broken_child_9527/images/raw/master/20200604003407.png)



3. 不重启微服务，访问springcloud-config的client端，输入`http://localhost:3355/info`，发现暴露的远程仓库配置信息并没有随着远程仓库配置文件的变化而变化。只有重启客户端暴露的信息才能发生更新。此时就需要配置客户端的动态刷新，让客户端暴露的配置信息随远程仓库配置信息的变化而变化

![image-20200604003727178](https://gitee.com/little_broken_child_9527/images/raw/master/20200604003728.png)

4. 引入监控`spring-boot-starter-actuator`，配置暴露监控端点

```yml
# 暴露监控端点，做到客户端config的动态刷新      
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

5. config client控制器加入`@RefreshScope`

6. 运维工程师输入`curl -X POST "http://localhost:3355/actuator/refresh"`通知config client端更新配置信息，**必须要多做这一步，否则执行步骤7 client端仍然不会刷新**

7. 再次访问config client控制器`http://localhost:3355/info`，无需重启config client微服务就能更新配置信息

   

**如果想做到无需向config client 发送Http请求也能动态刷新就需要使用使用消息总线**





b









