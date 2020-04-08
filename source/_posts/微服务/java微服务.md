---
title: java微服务
tags: [java, 微服务]
---

java微服务 的一些对比
<!-- more -->

java微服务

----

- 微服务

  - 概念
    - 一种架构模式，提倡将单一应用划分为一组小的服务，服务之间相互协调，相互配合
  - 特点
    - 独立部署 不需要因单一服务重新部署全部服务
    - 技术选型灵活 去中心化
    - 容错 会隔离到单独的服务
    - 扩展 每个小服务都可以增加额外的扩展
    - 复杂度可控 每个微服务功能可以单一

- 在分布式环境下，REST方式的服务依赖要比RPC方式的依赖更为灵活

- Dubbo

  - 弊端
    - 调用方与提供方依赖性太强

- Spring-boot.

  - 弊端
    - rest比rpc 更轻，不存在各种强依赖。但是要统一管理好文档
    - 可以通过整合swagger 使每个服务代码与文档一体化，解决上面的问题

  [比较参考链接]: https://www.jianshu.com/p/997ad63bd11d

  | 比较           | Dubbo                | Spring-boot                                                  |
  | -------------- | -------------------- | ------------------------------------------------------------ |
  | 交互方式       | 定义DTO              | json                                                         |
  | 调用方式       | rpc                  | http                                                         |
  | 代码入侵       | 配置xml,无代码入侵   | 注解配置有代码入侵                                           |
  | 依赖情况       | 调用方与提供方强依赖 | 无依赖，可跨平台                                             |
  | 版本管理       | 要有完善的版本管理   | 省略了版本管理的问题，但是具体字段约定要统一管理             |
  |                |                      |                                                              |
  | 服务注册中心   | Zookeeper redis      | Netflix Eureka                                               |
  | 服务网关       | 无                   | Netflix Zuul                                                 |
  | 断路（熔断器） | 暂不完善             | Netflix Hystrix                                              |
  | 配置中心       | 无                   | Spring-cloud config                                          |
  | 调用链追踪     | 无                   | Spring Cloud Sleuth                                          |
  | 消息总线       | 无                   | spring-cloud                                                 |
  | 数据流         | 无                   | Spring Cloud Stream 封装了与Redis,Rabbit、Kafka等发送接收消息 |
  | 批量任务       | 无                   | Spring Cloud Task                                            |

  

- Dubbo只是实现了服务治理
- SpringCloud子项目分别覆盖了微服务架构体系下的方方面面，服务治理只是其中的一个方面
- Dubbo额外提供了Filter扩展，对于上述“暂无”的部分，都可以通过扩展Filter来完善