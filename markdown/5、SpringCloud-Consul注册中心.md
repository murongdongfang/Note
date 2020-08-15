

# 简介

[Consul](https://learn.hashicorp.com/consul/getting-started/install.html)是一 开源的分布式服务发现和配置管理系统,由HashiCorp公司用Go语言开发。提供了微服务系统中的服务治理、配置中心、控制总线等功能。这些功能中的每一 个都可以根据需要单独使用，也可以一起使用以构建全方位的服务网格，总之Consu提供了一种完整的服务网格解决方案。它具有很多优点。基于raft协议，比较简洁，支持健康检查， 同时支持HTTP和DNS协议支持跨数据中心的WAN集群提供图形界面跨
平台，持Linux、 Mac、 Windows

[下载地址](https://www.consul.io/downloads.html)

[SpringCloud官网关于Consul的使用](https://www.springcloud.cc/spring-cloud-consul.html)



# 安装启动Consul注册中心

`consul agent -dev`：以开发模式启动consul

`consul --version`：查看consul的版本

`8500`：默认访问端口号

# 将Provider注册进注册中心

+ 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```



+ 配置Consul

```yml
server:
  # consul服务端口
  port: 8006
spring:
  application:
    name: cloud-providerconsul-payment8006
  cloud:
    consul:
      # consul注册中心地址
      host: localhost
      port: 8500
      discovery:
        hostname: 127.0.0.1
        service-name: ${spring.application.name}
```

+ 主启动类上标注`@EnableDiscovery`

# 将Consumer注册进注册中心



+ 引入依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```



+ 配置Consul

```yml
server:
  # consul服务端口
  port: 80
spring:
  application:
    name: cloud-consumerconsul-order80
  cloud:
    consul:
      # consul注册中心地址
      host: localhost
      port: 8500
      discovery:
        hostname: 127.0.0.1
        service-name: ${spring.application.name}
```

+ 消费者远程调用生产者

```java
@Configuration
public class WebConfig {

  @Bean
  @LoadBalanced
  public RestTemplate restTemplate(){
    return new RestTemplate();
  }
}


@RestController
public class ConsulOrderController {

  private static final String APPLICATION_NAME = "http://cloud-providerconsul-payment8006/";
  @Autowired
  private RestTemplate restTemplate;
  @GetMapping("/consul/consumer")
  public CommonResult<String> getOrder(){
    CommonResult res = restTemplate.getForObject(APPLICATION_NAME+"/consul/payment", CommonResult.class);
    return res;
  }
}
```



+ 主启动类上标注`@EnableDiscovery`



# 三种注册中心的比较

|   组件    | 开发语言 | CAP  | 服务健康检查 | 对外暴露接口 | SpringCloud集成 |
| :-------: | :------: | :--: | :----------: | :----------: | :-------------: |
|  Eureaka  |   Java   |  AP  |   可配支持   |     HTTP     |     已集成      |
|  Consul   |    Go    |  CP  |     支持     |   HTTP/DNS   |     已集成      |
| Zookeeper |   Java   |  CP  |     支持     |    客户端    |     已集成      |

![image-20200527200203864](https://gitee.com/little_broken_child_9527/images/raw/master/20200527200212.png)
CAP理论的核心是: 一个分布式系统不可能同时很好的满足一致性，可用性和分区容错性这三个需求,最织能同时较好的满足两个。|因此，根据CAP原理将NoSQL数据库分成了满足CA则、满足CP则和满足AP则三类:

+ CA - 单集群，满足一致性，可用性的系统，通常在可扩展性坏太强大。
+ CP - 满足一致性，分区容忍必的系统，通常性能不是特别高。

![image-20200527201331467](https://gitee.com/little_broken_child_9527/images/raw/master/20200527201333.png)

+ AP -满足可用性，分区容忍性的系统，通常可能对一致性要求低一些。当网络分区出现后，为了保证一致性,就必须拒接请求,否则无法保证一致性，违背了可用性A的要求，只满足致性和分区错，即CP

![image-20200527201143954](https://gitee.com/little_broken_child_9527/images/raw/master/20200527201145.png)





