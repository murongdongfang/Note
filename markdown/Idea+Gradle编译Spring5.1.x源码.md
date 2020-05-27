# 准备工作

IDEA2020+jdk1.8+[Gradle-5.6.4](https://gradle.org/releases/)+[spectj-1.9.5](https://www.eclipse.org/aspectj/downloads.php)+[spring5.1.x](https://github.com/spring-projects/spring-framework/tree/5.1.x)

## gradle安装

方法类似Tomcat，没有什么好说的。配置两个环境变量

![image-20200508121244062](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508121244062.png)



Gradle下载依赖jar包的本地路径

![image-20200508121416134](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508121416134.png)

Path路径配置

![image-20200508121605496](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508121605496.png)

## 安装aspectj

找到aspect.jar包路径，cmd窗口运行命令`java -jar aspectj-1.9.5.jar`。然后选择安装路径安装即可

![image-20200508121841210](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508121841210.png)





# 构建Spring源码

将Spring5源码解压，进入目录输入`gradlew.bat`

![image-20200508122548393](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508122437644.png)

![image-20200508122437644](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508122811970.png)

如果上面这个步骤出现错误比如网络错误，多试几次就行。接下来把源码导入IDEA中

![image-20200508122749807](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508122548393.png)



![image-20200508122811970](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508122925854.png)



要选择自己本地装的gradle和jdk，勾选自动导入依赖

![image-20200508122925854](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508123641996.png)

接下来就是构建了，构建时间要看个人的网速和电脑的配置。如果看到原来的文件夹图标变了，而且spring的gradle界面能够展开了就代表构建成功。接下来就是解决jar包依赖问题

![image-20200508123056116](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508123056116.png)



## spring-core模块报错

首先我们看到spring-core报错，这是因为在idea中导入`spring5`源码构建时，`spring-core`模块报错，缺失`cglib`相关的jar包依赖。

> 为了避免第三方class的冲突，Spring把最新的`cglib`和`objenesis`给重新打包（repack）了，它并没有在源码里提供这部分的代码，而是直接将其放在jar包当中，这也就导致了我们拉取代码后出现编译错误。那么为了成功编译，我们要把缺失的jar补回来

![image-20200508123641996](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508124348380.png)

运行完这两个命令后发现项目依赖里边多了一些jar包

![image-20200508171356876](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508171356876.png)

## spring-oxm模块报错

解决完spring-core模块报错之后继续来解决spring-oxm模块报错，运行这两个命令即可。

![image-20200508124348380](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508122749807.png)



# 编译源码

## 编译前准备

编译源码之前需要做两个准备，如果需要完整编译源码需要进行下面两个设置

设置虚拟机参数，防止发生OOM，`xml -XX:MaxPermSize=2048m -Xmx2048m -XX:MaxHeapSize=2048m`

![image-20200508172453028](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508172453028.png)

设置阿里云镜像`maven { url "http://maven.aliyun.com/nexus/content/groups/public/"}`

![image-20200508172634476](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508172634476.png)



编译译整个源码之前要先编译spring-core和spring-oxm模块

> `spring-core` and `spring-oxm` should be pre-compiled due to repackaged dependencies. 

编译的这两个模块的方法就是运行相应gradle文件下面的compileTestJava任务

> 1. Precompile `spring-oxm` with `./gradlew :spring-oxm:compileTestJava`





## 编译aspect模块

编译aspect模块的时候发现出错，由于aspect模块需要引入aspect的支持。稍显麻烦，因此干脆一不做二不休直接把这个模块移除。

在源码自带的`import-into-idea.md`中也对这个问题有说明

> `spring-aspects` does not compile due to references to aspect types unknown to IntelliJ IDEA. See https://youtrack.jetbrains.com/issue/IDEA-64446 for details. In the meantime, the 'spring-aspects' can be excluded from the project to avoid compilation errors.



<img src="https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508222659693.png" alt="image-20200508222659693" style="zoom:80%;" />



![image-20200508170550534](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508124108967.png)



介绍完上面那种暴力的方法接下来介绍完整编译aspect模块了。编译aspect模块我们最开始安装aspect就起到作用了。至于aspectj和aop可以谷歌一下，这里不做详细解释



设置编译器为Ajc，同时设置Ajc的安装目录。这一步有两点注意事项

+ 选择刚才安装的aspectj目录到aspectjtools.jar包。
+ 要勾选`Delegate to javac`，==作用只编译AspectJ的Facets项目（也就是下面设置的项目），而其他使用JDK代理。如果不勾选则全部使用Ajc编译就会导致编译出错==

![image-20200508124108967](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508170550534.png)





![image-20200508171826473](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508171826473.png)



![image-20200508123838853](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508123838853.png)

同样的方法为aop设置facets属性

![image-20200508174053587](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508174053587.png)







![image-20200508123937432](https://picgo-markdown.oss-cn-beijing.aliyuncs.com/img/image-20200508123937432.png)





https://gitee.com/zhong96/spring-framework-analysis

https://juejin.im/post/5d75a8e56fb9a06ade113673#heading-0