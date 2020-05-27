

# 简介

Spring Cloud Ribbon是基于[Netflix Ribbon](https://github.com/Netflix/ribbon/wiki/Getting-Started)实现的一套**客户端**负载均衡的工具。简单的说，Ribbon是Netflix发布的开源项目 ,要功能是提供客户端的软件负载均衡算法和服务调用。Ribbon客户端组件提供一系列完善的配置项如连接超时，试等。简单的说，就是在配置文件中列出Load Balancer (简称LB)后面所有的机器，Ribbon会自动的帮助你基于某种规则(如简单轮询,随机连接等)去连接这些机器。我们很容易使用Ribbon实现自定义的负载均衡算法。



Ribbon工作步骤

1. 先选择EurekaServer ,它优先选择在同一个区域内负载较少的server.
2. 第二步：根据用户指定的策略，在从server取到的服务注册列表中选择一 个地址。其中Ribbon提供了多种策略:比如轮询、随机和根据响应时间加权。



![image-20200527203633745](https://gitee.com/little_broken_child_9527/images/raw/master/20200527203635.png)

**LB负载均衡(Load Balance)是什么？**

+ 简单的说就是将用户的请求平摊的分配到多个服务上,从而达到系统的HA (高可用)。常见的负载均衡有软件Nginx, LVS, 硬件F5等。

**负载均衡分类？**

+ 进程内LB ，将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用,然后自己再从这些地址中选择出一个合适的服务器。Ribbon就属于进程内LB,它只是一个库,集成于消费方进程,消费方通过它来获取到服务提供方的地址。
+ 集中式LB，即在服务的消费方和提供方之间使用独立的LB设施(可以是硬件，如F5, 也可以是软件,如nginx), 由该设施负责把访问请求通过某种策略转发至服务的提供方;

**Ribbon本地负载均衡客户端 VS Nginx服务端负载均衡区别？**

+ Nginx是服务负载均衡，客户端所有请求都会交给nginx,然后由nginx实现转发请求。即负载均衡是由服务端实现的。
+ Ribbon本地负载均衡，在调用微服务接口时候，会在注册中心上获取注册信息服务列表之后缓存到JVM本地，从而在本地实现RPC远程服务调用技术。





在新版本eureka-client-starter依赖中已经自动集成了Ribbon

```xml
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
ribbon 单独依赖
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

![image-20200527204108251](https://gitee.com/little_broken_child_9527/images/raw/master/20200527204109.png)



**Ribbon+RestTemplate**

**RestTemplate的getForEntity和getForEntity方法的区别**

```java
@GetMapping("/consumer/order/{id}")
  public CommonResult<Payment> getPayment(@PathVariable Long id){
    //远程调用payment接口
    CommonResult res = restTemplate.getForObject(PAYMENT_URL + "/payment/get/"+id, CommonResult.class);

    return res;
  }

  @GetMapping("/consumer/order/entity/{id}")
  public CommonResult<Payment> getEntity(@PathVariable Long id){
    //getForEntity方法返回的是ResponseEntity，ResponseEntity里面除了Provider响应数据之外还封装了Consumer一些请求相关的参数
    //getForObject方法返回的就是Provider响应的业务数据
    ResponseEntity<CommonResult> entity = restTemplate.getForEntity(PAYMENT_URL + "/payment/get/" + id, CommonResult.class);
    if(entity.getStatusCode().is2xxSuccessful()){
      return entity.getBody();
    }else {
      return CommonResult.error("查询失败");
    }
  }
```



#  Ribbon负载均衡算法

IRule:根据特定算法从服务列表中选取一个要访问的服务

![image-20200527210002978](https://gitee.com/little_broken_child_9527/images/raw/master/20200527210006.png)



# 切换默认负载均衡算法







# 自定义负载均衡算法

