BeanPostProcessor：动态增强的Bean功能的接口，AOP的底层就是通过这个接口实现。

BeanFactory：bean创建的方式的抽象接口。

BeanFactoryPostProcessor：动态修改创建bean的factory工厂信息的接口

BeanDefinition：保存Bean信息的对象



SpringMVC和struts2的区别有哪些?

1. springmvc的入口是一个servlet即前端控制器（DispatchServlet），而struts2入口是一个filter过虑器（StrutsPrepareAndExecuteFilter）。

2. springmvc是基于方法开发(一个url对应一个方法)，请求参数传递到方法的形参，可以设计为单例或多例(建议单例)，struts2是基于类开发，传递参数是通过类的属性，只能设计为多例。

3. Struts采用值栈存储请求和响应的数据，通过OGNL存取数据，springmvc通过参数解析器是将request请求内容解析，并给方法形参赋值，将数据和视图封装成ModelAndView对象，最后又将ModelAndView中的模型数据通过reques域传输到页面。Jsp视图解析器默认使用jstl。
   

博客系统为什么要选择Springboot，SpringBoot的优点？

1. SpringBoot使用嵌入式Tomcat、Jetty、Undertow等丰富web容器，无需手动部署war包，支持热部署。

2. 可以使用SpringBoot创建独立独立的应用程序，方便分布式，微服务应用程序的开发。

3. 摒弃了XML繁杂配置文件，引入Ymal，Properties和注解进行配置。相比于XML更加简洁，清晰。

4. 提供生产就绪型功能，如指标、健康检查和外部配置等功能；