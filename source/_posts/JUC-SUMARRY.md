---
title: '聊聊java并发编程[汇总篇]'
tags:
  - java
  - 并发编程
  - juc
date: 2020-07-07 09:19:49
---


java并发文章汇总篇，从原理到应用层梳理。

<!-- more -->

#### 概念篇
- [并发编程](https://lasbun.github.io/2019/12/09/%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B/)
	- 什么是并发编程
	- 并发编程有哪些挑战
	- 怎么解决这些挑战
- [聊聊java并发编程 [核心问题以及java解决方案]](https://lasbun.github.io/2020/04/10/JUC-1/)
	- 并发编程核心问题(线程间的通信与同步)
	- 主要涉及到三个点
	  - 原子性
	  - 有序性
	  - 内存可见性。
	- java采用的解决方案(共享内存，另外一种方式是消息传递)
- [聊聊java并发编程[java实现之JMM]](https://lasbun.github.io/2020/04/17/JUC-2/)
	- 对上面java通过`共享内存方式`，进行的补充(解决思想)
	- 主要是介绍了什么是JMM，JMM又是为什么能够解决并发带来的问题
- [聊聊java并发编程 [java实现的基石]](https://lasbun.github.io/2020/04/11/java/java-concurrent/)
	- java并发体系是由哪些构建起来的

 #### 原理篇

- [聊聊java并发编程[synchronized vs volatile]](https://lasbun.github.io/2020/04/19/JUC-3/)
	- 介绍synchronized和volatile 是如何解决`原子性`，`有序性`，`内存可见性`。
	- synchronized 和volatile关键字的原理
	
- [聊聊java并发编程[JAVA-AQS]](https://lasbun.github.io/2020/04/09/JAVA-AQS/)
	- JUC核心部分，锁，同步队列，都是基于此 
	- 这篇文章介绍了什么是AQS, AQS的原理，独占式以及共享式状态如何获取与释放
	
- [聊聊java并发编程[JUC中的原子操作类]](https://lasbun.github.io/2020/04/29/JUC-4/)
	
	- 介绍了JUC中的原子类为什么是线程安全的以及其原理
	
- [聊聊java并发编程[JUC中的锁]](https://lasbun.github.io/2020/05/03/JUC-5/)
	
	- 介绍了常见的锁，以及部分锁的实现
	
- [聊聊java并发编程[JUC中的Condition]](https://lasbun.github.io/2020/05/03/JUC-6/)
	- 介绍了java并发Condition的原理以及源码剖析
	- 等待/通知模型的一种实现 
	
####  应用篇

- [聊聊java并发编程[HashMap && ConcurrenctHashMap]](https://lasbun.github.io/2020/05/12/JUC-7/)
	
	- 源码分析了jdk8的中的HashMap与ConcurrentHashMap的实现原理
	
- [聊聊java并发编程[JUC中的Executor]](https://lasbun.github.io/2020/05/15/JUC-8/)
	- 源码分析了Executor的架构，核心源码剖析

....持续更新中
