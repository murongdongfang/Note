# 简介

Gateway是在Spring生态系统之上构建的API网关服务,于Spring 5, Spring Boot 2和Project Reactor等技术。Gateway旨在提供-种简单而有效的方式来对API进行路由，以及提供-些强大的过滤器功能， 例如熔断、限流重试等



SpringCloud Gateway是Spring Cloud的一个全新项目， 基于Spring 5.0+ Spring Boot 2.0和Project Reactor等技术开发的网关，它旨在为微服务架构提供一种简单有效的统-的API路由管理方式。SpringCloud Gateway作为Spring Cloud生态系统中的网关,目标是替代Zuul，在Spring Cloud 2.0以上版本中，没有对新版本的Zuul 2.0以上最新高性能版本进行集成，仍然还是使用的Zuul 1.x非Reactor模式的老版本。而为了提升网关的性能，SpringCloud Gateway是基于WebFlux框架实现的，而WebFlux框架底层则使用了高性能的Reactor模式通信框架Netty。SpringCloudGateway的目标提供统的路由方式且基盱Filter链的方式提供了网关基本的功能，例如：安全，监控/指标，和限流。

**SpringCloud Gateway使用的是Webflux中的reactor-netty响应式编程组件,底层使用了Netty通讯框架**





![image-20200530132520266](https://gitee.com/little_broken_child_9527/images/raw/master/20200530132521.png)





在SpringCloud Finchley正式版之前，Spring Cloud推荐的网关是Netflix提供的Zuul:

1. Zuul 1.x, 是一个基于阻塞I/ 0的API Gateway
2. Zuul 1.x基于Servlet 2. 5使用阻塞架构它不支持任何长连接(如WebSocket) Zuul的设计模式和Nginx较像，次I/ 0操作都是从工作线程中选择一个执行， 求线程被阻塞到工作线程完成，但是差别是Nginx用C++实现，Zuul 用Java实现，而JVM本身会有第一次加载较慢的情况， 使得Zuul的性能相对较差。
3. Zuul 2.x理念更先进,想基于Netty非阻塞和支持长连接，但SpringCloud且 前还没有整合。Zuul 2.x的性能较Zuul 1.x有较大提升。在性能面,根据官方提供的基准测试， Spring Cloud Gateway的RPS (每秒请求数)是Zul的1. 6倍。
4. Spring Cloud Gateway建立在Spring Framework 5、Project Reactor和Spring Boot2之上，使佣非阻塞API。
5. Spring Cloud Gateway还支持WebSocket,組与Spring紧密集成拥有更好的开发体验



传统的Web框架，比如说: struts2, springmvc等都是 基于Servlet API与Servlet容器基础之上运行的。但是在Servlet3.1之后有了异步非阻塞的支持。而WebFlux是-个典型非阻塞异步的框架，它的核心是基于Reactor的相关API实现的。相对于传统的web框架来说，它可以运行在诸如Netty, Undertow及 支持Servlet3.1的容器上。非阻塞式+函数式编程(Spring5必须让你使用java8)Spring WebFlux是Spring 5.0引入的新的响应式框架，区别于Spring MVC,它不需要依赖Servlet API,它是完全异步非阻塞的，并且基于Reactor来实现响应式流规范。

[gatway官网](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.1.RELEASE/reference/html/)



##  Gateway重要概念

+ Route(路由)：路由是构建网关的基本模块,它由ID,目标URI,一系列的断言和过滤器组成,如断言为true则匹配该路由
+ Predicate(断言)：参考的是Java8的java.util.function.Predicate开发人员可以匹配HTTP请求中的所有内容(例如请求头或请求参数),如果请求与断言相匹配则进行路由
+ Filter(过滤)：指的是Spring框架中GatewayFilter的实例,使用过滤器,可以在请求被路由前或者之后对请求进行修改。



web请求,通过一些配条件,定位到真正的服务节点。并在这个转发过程的前后，进行些精细化控制。predicate就是我们的匹配条件;而filter, 就可以理解为一个无所不能的拦截器。有了这两个元素，加上目标uri,就可以实现一个具体的路由了

![image-20200531185738679](https://gitee.com/little_broken_child_9527/images/raw/master/20200531185747.png)



## Gateway工作流程

客户端向Spring Cloud Gateway发出请求。然后在Gateway Handler Mapping找到与请求相匹配的路由，将其发送到Gateway Web Handler。Handler通过指定的过滤器链来将请求发送到我们实际的服务执行业务逻辑，然后返回。过滤器之间虚线分开因为过滤器可能会在发送代理请求之前( "pre" )或之后( "post" )执行业务逻辑。Filter在"pre" 型的过滤器可以做参数校验、权限校验、流量监控、日志输出、 协议转换等,
在"post"型的过滤器中可以做响应内容、响应头的修改,日志的输出，流糧监控等有着非常重要的作用。

==**简而言之就是：一个请求（url）在到达后台之前首先gateway需要通过Predict断言来判断这个请求的url是否符合某个Router路由，如果符合某个路由就把这个请求转发到相应的后台。在到达后台之前gateway需要通过Filter对这个请求进行一定的处理（例如权限校验，跨域配置等）**==

![image-20200531190001714](https://gitee.com/little_broken_child_9527/images/raw/master/20200531190004.png)

# 基本用法



+ 引入依赖，**注意引入`spring-cloud-starter-gateway`不能引入`spring-boot-starter-web`，因为gateway的视图层底层使用的是spring5的新特性Spring Webflux，而``spring-boot-starter-web``使用的是SpringMVC**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

```

+ 使用yml配置路由，如下路由配置的含义的是当访问localhost:9527的时候，如果路径中包含path中配置的/payment/get/**就把这个请求转发到配置的uri+path的微服务controller

```yml
server:
  port: 9527

spring:
  application:
    name: cloud-gateway9527
  cloud:
    gateway:
      routes:
        - id: payment_route # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
          uri: http://localhost:8001
          predicates:
            - Path=/payment/get/** # 断言，路径相匹配的进行路由
        - id: payment_route2
          uri: http://localhost:8001
          predicates:
            Path=/payment/lb/** #断言,路径相匹配的进行路由

```



**Gateway不同于我们使用以往的其他微服务组件，Gateway无需我们手动再主启动类上开启@Enablexxx，只需要引入依赖，配置好相应的yml配置文件即可**

+ 使用编码的方式配置路由

```java
@Configuration
public class GatewayConfig {
  /**
   * 配置一个id名为baidu的路由
   * 当访问gateway网关http://localhost:9527的时候如果路径中是http://localhost:9527/guonei
   * gateway会把这个请求转发到http://news.baidu.com/guonei
   */
  @Bean
  public RouteLocator customRouteLocator(RouteLocatorBuilder routeLocatorBuilder){
    RouteLocatorBuilder.Builder routes = routeLocatorBuilder.routes();
    routes.route(
          "baidu",//配置id
           r->r.path("/guonei").uri("http://news.baidu.com/guonei")//使用lambda配置路由转发规则
    );
    return routes.build();
  }
}

```



# Gateway配置负载均衡



+ 引入注册中心依赖

```xml
<!--eureka client-->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

```

+ 新增配置

Gateway将请求转发给生产者，如果该生产者是一个集群配置，**使用Gateway可以实现负载均衡功能，但是需要借助注册中心。此时uri上面配置的就不再是一个具体的IP+端口号，而是注册在注册中心的微服务名。**以下是完整配置

**注意：- Path首字母大写，-和Path有空格，Path后面接的是=而非:**

```yml
server:
  port: 9527
spring:
  application:
    name: cloud-gateway9527
  cloud:
    gateway:
#      从注册中心获取所有生产者微服务实例，以便进行负载均衡
      discovery:
        locator:
          enabled: true
      routes:
        - id: payment_route # 路由的id,没有规定规则但要求唯一,建议配合服务名
          #匹配后提供服务的路由地址
# uri: http://localhost:8001 此时不在是具体的ip+端口号，而是lb://+微服务名，这样就可以实现负载均衡
          uri: lb://CLOUD-PAYMENT-PROVIDER-8001
          predicates:
            - Path=/payment/get/** # 断言，路径相匹配的进行路由
        - id: payment_route2
#          uri: http://localhost:8001
          uri: lb://CLOUD-PAYMENT-PROVIDER-8001
          predicates:
            - Path=/payment/lb/** #断言,路径相匹配的进行路由

eureka:
  instance:
    hostname: cloud-gateway-service9527
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://eureka7001.com:7001/eureka/
```

在properties文件中配置数组

```properties
# 服务端口
server.port=8222
# 服务名
spring.application.name=service-gateway

# nacos服务地址
spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848

#使用服务发现路由
spring.cloud.gateway.discovery.locator.enabled=true
#服务路由名小写
#spring.cloud.gateway.discovery.locator.lower-case-service-id=true

#设置路由id
spring.cloud.gateway.routes[0].id=service-acl
#设置路由的uri
spring.cloud.gateway.routes[0].uri=lb://service-acl
#设置路由断言,代理servicerId为auth-service的/auth/路径
spring.cloud.gateway.routes[0].predicates= Path=/*/acl/**
```



+ 启动类标注`@EnableDiscovery`



# Gateway常用Predicate

Spring Cloud Gateway将路由匹配作为Spring WebFlux Handler Mapping基础架构的一部分。Spring Cloud Gateway包括许多内置的Route Predicate工厂。所有这些Predicate都与HTTP请求的不同属性匹配。多个Route  Predicate厂可以进行组给Spring Cloud Gateway创建Route对象时，使用RoutePredicateFactory创建Predicate对象，Predicate 对象可以赋值给Route。Spring Cloud Gateway包含许多内置的Route Predicate Factories。**一句话下面的这些就是指定配置好的路由映射只有在符合一定的条件之下才能生效**



启动Gateway微服务，下面是是控制台的输入，以下都是predicate的配置

[Predicate官网](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.3.RELEASE/reference/html/#gateway-request-predicates-factories)

![image-20200531214924799](https://gitee.com/little_broken_child_9527/images/raw/master/20200531214926.png)



1. After Route Predicate：当前配置的路由映射只有在过了After配置的这个时间之后才能生效

```yml
routes：
    - id: payment_route2
      uri: lb://CLOUD-PAYMENT-PROVIDER-8001
      predicates:
        - Path=/payment/lb/** #断言,路径相匹配的进行路由
        - After=2020-05-31T22:59:17.981+08:00[GMT+08:00] #这个路由匹配只有到了这个时间才会生效
```

2. Before Route Predicate：当前配置的路由映射只有在Before配置的时间之前才能生效
3. Between Route Predicate ：当前配置的路由映射只有在between配置的时间段之内才能生效

```yml
 routes:
      - id: between_route
        uri: https://example.org
        predicates:
        - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
```

4. Cookie Route Predicate ：需要两个参数，一个是Cookie name ,一个是正则表达式。路由规则会通过获取对应的Cookie namle值和正则表达式匹配，如果匹配上就会执行路由，如果没有匹配上则不执行。下面这段配置意思：访问当前路由的时候只有携带了cookie，并且cookie的key为username，value为xxh时候这个路由才能生效

```yml
 routes:
     - id: payment_route2
     #          uri: http://localhost:8001
     uri: lb://CLOUD-PAYMENT-PROVIDER-8001
     predicates:
        - Path=/payment/lb/** #断言,路径相匹配的进行路由
        - Cookie=username,xxh
```



使用`crul http://localhost:9527/payment/lb`不携带cookie，访问刚才配置好的路由此时会报404

![image-20200531222036932](https://gitee.com/little_broken_child_9527/images/raw/master/20200531222038.png)

**测试：使用`curl http://localhost:9527/payment/lb --cookie "username=xxh"`命令发送携带有指定cookie的请求访问刚才配置好的路由能正常访问**

![image-20200531222124213](https://gitee.com/little_broken_child_9527/images/raw/master/20200531222125.png)



5. Header Route Predicate：只有携带了Id请求头，而且这个值必须为整数当前配置的路由才能生效

```yaml
routes: 
 - id: payment_route2
      uri: lb://CLOUD-PAYMENT-PROVIDER-8001
      predicates:
          - Path=/payment/lb/** #断言,路径相匹配的进行路由
          - Header=Id, \d+
```

**测试：使用curl发送携带有指定请求头的http请求`curl http://localhost:9527/payment/lb  -H "Id:5"`**

6. Host Route Predicate：接收一组参数，一组匹配的域名列表,这个模板是- -个ant分隔的模板，用.号作为分隔符。它通过参数中的主机地址作为匹配规则。如下配置，必须要携带Host请求头，Host请求头必须以whpu.com结尾才能使得当前路由配置生效

```yml
routes: 
 - id: payment_route2
      uri: lb://CLOUD-PAYMENT-PROVIDER-8001
      predicates:
          - Path=/payment/lb/** #断言,路径相匹配的进行路由
          - Host=**.whpu.com
```



**测试：使用curl发送携带有指定Host请求头的http请求`curl http://localhost:9527/payment/lb  -H "Host:www.whpu.com"`**

7. Method Route Predicate：请求只有是指定的提交格式，当前配置的路由才能生效

```yml
routes: 
 - id: payment_route2
      uri: lb://CLOUD-PAYMENT-PROVIDER-8001
      predicates:
        - Path=/payment/lb/** #断言,路径相匹配的进行路由
        - Method=GET,POST
```

**测试：使用curl发送携带有指定请求头的http请求`curl http://localhost:9527/payment/lb  -H "Id:5"`**

8. Path Route Predicate

9. Query Route Predicate 
10. RemoteAddr Route Predicate
11. Weight Route Predicate



# Filter的使用

指的是Spring框架中GatewayFilter的实例，使用过滤器，可以在请求被路由前或者之后对请求进行修改.

[官网filter](https://cloud.spring.io/spring-cloud-static/spring-cloud-gateway/2.2.3.RELEASE/reference/html/#global-filters)



## 自定义过滤器

+ 实现`GlobalFilter,Ordered`接口

```java
@Slf4j
@Component
public class MyLogGatewayConfig implements GlobalFilter, Ordered {

  @Override
  public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    log.info("come in global filter: {}", new Date());

    //从exchange中获取reqeust对象
    ServerHttpRequest request = exchange.getRequest();
    //获取get提交的query参数
    String uname = request.getQueryParams().getFirst("uname");
    if (uname == null) {
      log.info("用户名为null，非法用户");
      exchange.getResponse().setStatusCode(HttpStatus.NOT_ACCEPTABLE);
      return exchange.getResponse().setComplete();
    }
    // 放行
    return chain.filter(exchange);
  }

  /**
   * 过滤器加载的顺序 越小,优先级别越高
   *
   * @return
   */
  @Override
  public int getOrder() {
    return 0;
  }

}

```

## 使用Filter解决跨域问题

**注意：使用Filter解决跨域问题的时候相应的Controller不能使用`@Cross`注解**

```java
@Configuration
public class CorsConfig {
    @Bean
    public CorsWebFilter corsFilter() {
        CorsConfiguration config = new CorsConfiguration();
        config.addAllowedMethod("*");
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource(new PathPatternParser());
        source.registerCorsConfiguration("/**", config);

        return new CorsWebFilter(source);
    }
}
```



## 使用Filter权限管理

使用filter拦截每个请求请求头，如果请求头包含token信息给与放行，否则不能放行

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private AntPathMatcher antPathMatcher = new AntPathMatcher();

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        //谷粒学院api接口，校验用户必须登录
        if(antPathMatcher.match("/api/**/auth/**", path)) {
            List<String> tokenList = request.getHeaders().get("token");
            if(null == tokenList) {
                ServerHttpResponse response = exchange.getResponse();
                return out(response);
            } else {
//                Boolean isCheck = JwtUtils.checkToken(tokenList.get(0));
//                if(!isCheck) {
                    ServerHttpResponse response = exchange.getResponse();
                    return out(response);
//                }
            }
        }
        //内部服务接口，不允许外部访问
        if(antPathMatcher.match("/**/inner/**", path)) {
            ServerHttpResponse response = exchange.getResponse();
            return out(response);
        }
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return 0;
    }

    private Mono<Void> out(ServerHttpResponse response) {
        JsonObject message = new JsonObject();
        message.addProperty("success", false);
        message.addProperty("code", 28004);
        message.addProperty("data", "鉴权失败");
        byte[] bits = message.toString().getBytes(StandardCharsets.UTF_8);
        DataBuffer buffer = response.bufferFactory().wrap(bits);
        //response.setStatusCode(HttpStatus.UNAUTHORIZED);
        //指定编码，否则在浏览器中会中文乱码
        response.getHeaders().add("Content-Type", "application/json;charset=UTF-8");
        return response.writeWith(Mono.just(buffer));
    }
}
```



















