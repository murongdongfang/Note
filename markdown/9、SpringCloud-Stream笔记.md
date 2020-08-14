# 简介

**什么是SpringCloudStream？**
官方定义Spring Cloud Stream是一个构建消息驱动微服务的框架。应用程序通过inputs（消息生产者）或者outputs（消息消费者）来和Spring Cloud Stream中binder对象交互。通过我们配置来binding(绑定)，而Spring Cloud Stream的binder对象负责与消息中间件交互。所以，我们只需要搞清楚如何与Spring Cloud Stream交互就可以方便使用消息驱动的方式。通过使用Spring Integration来连接消息代理中间件以实现消息事件驱动。Spring Cloud Stream为一些供应商的消息中间件产品提供了个性化的自动化配置实现，引用了发布-订阅、消费组、分区的三个核心概念。**目前仅支持RabbitMQ、Kafka。一句话Spring Cloud Stream就是用来蔽底层消息中间件的差异，降低切换版本，统一消息的编程模型。相当于SpringData JPA就是用来屏蔽底层数据库的差异，提供统一的数据库访问抽象模型一样**





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





![image-20200814225614888](https://gitee.com/little_broken_child_9527/images/raw/master/20200814225615.png)





# 重要概念

绑定器：很方便的连接中间件，屏蔽差异

Channel：通道，是队列Queue的一种抽象，在消息通讯系统中就是实现存储和转发的媒介，通过对Channel对队列进行配置

Source和Sink：简单的可理解为参照对象是Spring Cloud Stream自身，从Stream发布消息就是输出，接受消息就是输入



![image-20200814225656644](https://gitee.com/little_broken_child_9527/images/raw/master/20200814225658.png)



![image-20200814230137386](https://gitee.com/little_broken_child_9527/images/raw/master/20200814230138.png)























































