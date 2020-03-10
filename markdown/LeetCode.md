https://github.com/CoderMJLee

https://cnblogs.com/mjios





博客系统分为以下四个部分：

| 项目名称     | 简介                                                         |
| :----------- | :----------------------------------------------------------- |
| blog         | 提供整个系统的服务，采用 [Spring Boot](https://spring.io/) 开发 |
| blog-Admin   | 负责后台管理的渲染，采用 [Vue](https://vuejs.org/) 开发，已集成在 Halo 运行包内，无需独立部署。 |
| blog-comment | 评论插件，采用 [Vue](https://vuejs.org/) 开发，在主题中运行方式引入构建好的 `Javascript` 文件即可 |
| halo-theme   | 主题项目集，采用 [Freemarker](https://freemarker.apache.org/) 模板引擎编写，需要包含一些特殊的配置才能够被 halo 所使用 |

