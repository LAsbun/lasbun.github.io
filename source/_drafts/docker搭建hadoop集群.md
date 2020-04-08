---
title: Docker 搭建Hadoop 集群.
tag: hadoop, docker, 集群
---

## 介绍
---
- 用docker 从0开始搭建一个hadoop集群.

背景
----
- todo 


搭建步骤
----
- 启动三个centos6.7镜像的容器，容器名分别是：master、slave1、slave2；
- 给master、slave1、slave2分别安装ssh服务；
- 从当前电脑通过scp命令将jdk1.8的安装包上传到三个容器，然后依次安装jdk1.8；
- 配置hostname，编辑三个容器的/etc/sysconfig/network文件，修改HOSTNAME分别是master、slave1、slave2；
- 配置host，通过ip addr命令取得三个容器的ip，然后修改每个/etc/hosts文件，都添加如下内容： 
  ```
  172.18.0.2 master 
  172.18.0.3 slave1 
  172.18.0.4 slave2
  ```
- master、slave1、slave2之间配置相互免密码登录       sshd_config、authorized_keys、id_rsa.pub文件；
- master、slave1、slave2上安装zookeeper-3.4.6集群，并启动；
- 配置java和Hadoop环境变量：/etc/profile、hadoop-env.sh、yarn-env.sh；
- 修改hadoop相关配置文件：core-site.xml、hdfs-site.xml、mapred-site.xml、mapred-site.xml；
- 修改hadoop的slave配置：etc/hadoop/slaves；
- 保证以上的hadoop配置在三个容器上是一致的；
- 启动hadoop，验证；
- 修改/etc/profile，配置hbase；
- 配置hbase-site.xml、regionservers、hbase-env.sh；
- 启动hbase，验证；



注意点：
  - 搭建zookeeper 集群时出现 Address unresolved: xxx:3888 原因是zoo.cfg server.x=A:B:C C后面不能有空格
  - 通过docker-compose extra_hosts 增加/etc/hosts修改


## 流程
----
- 设置各节点 ssh 免密
- 启动zookeeper(设置不同节点的myid)
- 启动hdfs hdfs namenode -format  && start-yarn.sh
- Hbase 配置，&& 包括JAVA_HOME
- 