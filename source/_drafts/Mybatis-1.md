---
title: 聊聊Mybatis[整体概览]
tags: [java,mybatis,mysql]
---

<!-- more -->

#### 核心模块(类)
核心模块流转关系见下图

![image-20200730170645454](Mybatis-1/image-20200730170645454.png)

{% asset-img image-20200730170645454.png %}

##### SqlSessionFactoryBuilder
描述:每一个MyBatis的应用程序的入口是SqlSessionFactoryBuilder。通过读取配置文件
作用: 生成SqlSessionFactory

