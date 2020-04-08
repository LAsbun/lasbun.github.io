---
title: 聊聊dubbo与zookeeper
date: 2019-07-16 13:53:02
tags: [java, dubbo, zookeeper]
---

### dubbo
简单来讲就是一个远程调用RPC协议框架。本质上还是个协议

### zookeeper
是一个注册中心。本质上还是一个软件(software)

### dubbo 分组 vs zk分组
dubbo 分组本意上是当应用有多种版本的时候，可以用group。 （从这点来看用来做环境弱隔离也是可以的）
zk分组实际上是根据物理机ip分组。