---
title: hadoop-hbase-zookeeper
date: 2019-11-02 21:00:55
tags: [hadoop, hbase, zookeeper]
---

大数据开发，hadoop,hbase 环境搭建肯定是必不可少少了。之前已经把hadoop搞清楚了，但是直到今天才把Hbase搭建成功（虽然中间还是有点瑕疵），记录下相关的操作.
<!-- more -->



#### 系统相关配置

- 三台机器centos7(master(10.211.55.20), slave1(10.211.55.21), slave2(10.211.55.22))  
- java 1.8
- hadoop 2.7.2
- hbase 1.4.9
- zookeeper 3.4.14

#### 各节点功能

| 节点/hostname       | 服务                                                         |
| ------------------- | ------------------------------------------------------------ |
| master/10.211.55.20 | DataNode<br/> JournalNode<br/>NodeManager<br/>QuorumPeerMain<br/>HRegionServer<br/>NameNode<br/>Jps<br/>HMaster |
| slave1/10.211.55.21 | JournalNode<br/> ResourceManager<br/> NodeManager<br/> HRegionServer<br/>QuorumPeerMain<br/> NameNode |
| slave2/10.211.55.22 | JournalNode<br/>HRegionServer<br/>NodeManager<br/>DataNode<br/>QuorumPeerMain |



#### 预先配置

- 用户(hadoop)

  - 在三台机器上创建hadoop用户（之后的操作都是在用户hadoop下操作的）

    ```shell
    useradd hadoop
    ```

    

- 开启免密登录

  ```shell
  # 在三台机器上分别执行
  ssh-copy-id 10.211.55.20
  ssh-copy-id 10.211.55.21
  ssh-copy-id 10.211.55.22
  ```

  

- 下载相关安装包到/home/hadoop（我们的安装目录）下面，解压，此时的目录下为

  ```
  hadoop@master [04:47:44 PM] [~/hadoop-2.7.2]
  -> % ls
  bin      lib          logs        sbin
  etc      libexec      NOTICE.txt  share
  include  LICENSE.txt  README.txt  tmp
  ```

  配置java文件

  - 将下面的几句话添加到.bashrc中(因为我用的是zsh,所以是.zshrc), 三台机器都需要配置

  ```
  # jdk1.8
  export JAVA_HOME=/home/hadoop/jdk1.8
  export PATH=$JAVA_HOME/bin:$PATH
  export CLASS_PATH=$JAVA_HOME/lib
  ```

  - 添加完之后，记得source .bashrc 或者是source .zshrc

- 关闭三台机器的防火墙,分别登录到三台机器上执行

  ```
  # 关闭防火墙
  sudo systemctl stop firewalld.service
  # 开机关闭防火墙
  sudo systemctl disable firewalld.service
  ## 验证是否关闭成功
  sudo systemctl status firewalld.service 
  ## 执行上面的命令后，出现下面的日志即表示已经关闭防火墙了
  ● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
       Docs: man:firewalld(1)
  
  Nov 01 06:54:04 slave1 systemd[1]: Starting fir...
  Nov 01 06:54:05 slave1 systemd[1]: Started fire...
  Nov 01 07:38:19 slave1 systemd[1]: Stopping fir...
  Nov 01 07:38:20 slave1 systemd[1]: Stopped fire...
  Hint: Some lines were ellipsized, use -l to show in full.
  ```

  

#### zookeeper

- 在 master(10.211.55.20上)

  ```
  cd /home/hadoop/zookeeper-3.4.14/conf
  mv zoo_sample.cfg zoo.cfg
  ```

  - 将下面的东西复制进zoo.cfg 删除原来的配置

    ```
    # The number of milliseconds of each tick
    tickTime=2000
    # The number of ticks that the initial
    # synchronization phase can take
    initLimit=10
    # The number of ticks that can pass between
    # sending a request and getting an acknowledgement
    syncLimit=5
    # the directory where the snapshot is stored.
    # do not use /tmp for storage, /tmp here is just
    # example sakes.
    dataDir=/home/hadoop/zookeeper-3.4.14/data/zkdata
    # the port at which the clients will connect
    clientPort=2181
    # the maximum number of client connections.
    # increase this if you need to handle more clients
    #maxClientCnxns=60
    #
    # Be sure to read the maintenance section of the
    # administrator guide before turning on autopurge.
    #
    # http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
    #
    # The number of snapshots to retain in dataDir
    #autopurge.snapRetainCount=3
    # Purge task interval in hours
    # Set to "0" to disable auto purge feature
    #autopurge.purgeInterval=1
    server.1=master:2888:3888
    server.2=slave1:2888:3888
    server.3=slave2:2888:3888
    ```

    

  - 创建zookeeper 数据保存文件夹

    ```
    mkdir -p /home/hadoop/zookeeper-3.4.14/data/zkdata
    cd /home/hadoop/zookeeper-3.4.14/data/zkdata
    echo "1" > myid
    ```

  - 将master机器上改过之后的zookeeper 文件夹发送到slave1以及slave2机器上

    ​	

    ```
    cd ~
    scp -r zookeeper-3.4.14 10.211.55.21:/home/hadoop
    scp -r zookeeper-3.4.14 10.211.55.22:/home/hadoop
    ```

  - 登录到slave1 

    ```
    echo "2" /home/hadoop/zookeeper-3.4.14/data/zkdata/myid
    ```

  - 登录到slave2

    ```
    echo "2" /home/hadoop/zookeeper-3.4.14/data/zkdata/myid
    ```

  - 分别登录上三台机器执行下面命令，启动zookeeper集群

    ```
    cd /home/hadoop/zookeeper-3.4.14
    ./bin/zkServer.sh start
    ```

    启动成功之后，检测三台机器状态(因zookeeper内部选取机制，所以下面的leader与fellower仅供参考，并不一定是下面的情况)

    ```
    hadoop@master [04:48:24 PM] [~/zookeeper-3.4.14]
    -> % ./bin/zkServer.sh status
    ZooKeeper JMX enabled by default
    Using config: /home/hadoop/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Mode: follower
    
    hadoop@slave1 [04:49:18 PM] [~/zookeeper-3.4.14]
    -> % ./bin/zkServer.sh status
    ZooKeeper JMX enabled by default
    Using config: /home/hadoop/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Mode: leader
    
    hadoop@slave2 [04:49:24 PM] [~/zookeeper-3.4.14]
    -> % ./bin/zkServer.sh status
    ZooKeeper JMX enabled by default
    Using config: /home/hadoop/zookeeper-3.4.14/bin/../conf/zoo.cfg
    Mode: follower
    ```
    
    

#### hadoop

- 修改配置(下面的操作都是在/home/hadoop/hadoop-2.7.2/etc/hadoop目录下面进行修改)

  - 修改hadoop-env.sh

  - ```
      # 找到JAVA_HOME 修改为
      export JAVA_HOME=/home/hadoop/jdk1.8
    ```

  - vi core-site.xml

    - ```
      <configuration>
        <!-- hdfs地址，ha模式中是连接到nameservice  -->
        <property>
          <name>fs.defaultFS</name>
          <value>hdfs://ns1</value>
        </property>
        <!-- 这里的路径默认是NameNode、DataNode、JournalNode等存放数据的公共目录，也可以单独指定 -->
        <property>
          <name>hadoop.tmp.dir</name>
          <value>/home/hadoop/hadoop-2.7.2/tmp</value>
        </property>
      
        <!-- 指定ZooKeeper集群的地址和端口。注意，数量一定是奇数，且不少于三个节点-->
        <property>
          <name>ha.zookeeper.quorum</name>
          <value>master:2181,slave1:2181,slave2:2181</value>
        </property>
      
      </configuration>
      ```

  - vi hdfs-site.xml

    - ```
      <configuration>
        <!-- 指定副本数，不能超过机器节点数  -->
        <property>
          <name>dfs.replication</name>
          <value>3</value>
        </property>
      
        <!-- 为namenode集群定义一个services name -->
        <property>
          <name>dfs.nameservices</name>
          <value>ns1</value>
        </property>
      
        <!-- nameservice 包含哪些namenode，为各个namenode起名 -->
        <property>
          <name>dfs.ha.namenodes.ns1</name>
          <value>master,slave1</value>
        </property>
      
        <!-- 名为master的namenode的rpc地址和端口号，rpc用来和datanode通讯 -->
        <property>
          <name>dfs.namenode.rpc-address.ns1.master</name>
          <value>master:9000</value>
        </property>
      
        <!-- 名为slave1的namenode的rpc地址和端口号，rpc用来和datanode通讯 -->
        <property>
          <name>dfs.namenode.rpc-address.ns1.slave1</name>
          <value>slave1:9000</value>
        </property>
      
        <!--名为master的namenode的http地址和端口号，用来和web客户端通讯 -->
        <property>
          <name>dfs.namenode.http-address.ns1.master</name>
          <value>master:50070</value>
        </property>
      
        <!-- 名为slave1的namenode的http地址和端口号，用来和web客户端通讯 -->
        <property>
          <name>dfs.namenode.http-address.ns1.slave1</name>
          <value>slave1:50070</value>
        </property>
      
        <!-- namenode间用于共享编辑日志的journal节点列表 -->
        <property>
          <name>dfs.namenode.shared.edits.dir</name>
          <value>qjournal://master:8485;slave1:8485;slave2:8485/ns1</value>
        </property>
      
        <!-- 指定该集群出现故障时，是否自动切换到另一台namenode -->
        <property>
          <name>dfs.ha.automatic-failover.enabled.ns1</name>
          <value>true</value>
        </property>
      
        <!-- journalnode 上用于存放edits日志的目录 -->
        <property>
          <name>dfs.journalnode.edits.dir</name>
          <value>/home/hadoop/hadoop-2.7.2/tmp/data/dfs/journalnode</value>
        </property>
      
        <!-- 客户端连接可用状态的NameNode所用的代理类 -->
        <property>
          <name>dfs.client.failover.proxy.provider.ns1</name>
          <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
        </property>
      
        <!-- 一旦需要NameNode切换，使用ssh方式进行操作 -->
        <property>
          <name>dfs.ha.fencing.methods</name>
          <value>sshfence</value>
        </property>
      
        <!-- 如果使用ssh进行故障切换，使用ssh通信时用的密钥存储的位置 -->
        <property>
          <name>dfs.ha.fencing.ssh.private-key-files</name>
          <value>/home/hadoop/.ssh/id_rsa</value>
        </property>
      
        <!-- connect-timeout超时时间 -->
        <property>
          <name>dfs.ha.fencing.ssh.connect-timeout</name>
          <value>30000</value>
        </property>
      </configuration>
      ```

  - vi mapred-site.xml

    - ```
      <!-- 采用yarn作为mapreduce的资源调度框架 -->
      <configuration>
        <property>
          <name>mapreduce.framework.name</name>
          <value>yarn</value>
        </property>
      </configuration>
      ```

  - vi yarn-site.xml

    - ```
      <configuration>
      
        <!-- 启用HA高可用性 -->
        <property>
          <name>yarn.resourcemanager.ha.enabled</name>
          <value>true</value>
        </property>
      
        <!-- 指定resourcemanager的名字 -->
        <property>
          <name>yarn.resourcemanager.cluster-id</name>
          <value>yrc</value>
        </property>
      
        <!-- 使用了2个resourcemanager,分别指定Resourcemanager的地址 -->
        <property>
          <name>yarn.resourcemanager.ha.rm-ids</name>
          <value>rm1,rm2</value>
        </property>
      
        <!-- 指定rm1的地址 -->
        <property>
          <name>yarn.resourcemanager.hostname.rm1</name>
          <value>master</value>
        </property>
      
        <!-- 指定rm2的地址  -->
        <property>
          <name>yarn.resourcemanager.hostname.rm2</name>
          <value>slave1</value>
        </property>
      
        <!-- 指定当前机器master作为rm1 -->
        <property>
          <name>yarn.resourcemanager.ha.id</name>
          <value>rm1</value>
        </property>
      
        <!-- 指定zookeeper集群机器 -->
        <property>
          <name>yarn.resourcemanager.zk-address</name>
          <value>master:2181,slave1:2181,slave2:2181</value>
        </property>
      
        <!-- NodeManager上运行的附属服务，默认是mapreduce_shuffle -->
        <property>
          <name>yarn.nodemanager.aux-services</name>
          <value>mapreduce_shuffle</value>
        </property>
      
      </configuration>
      ```

  - vi slaves(删除localhost)

    - ```
      master
      slave1
      slave2
      ```

  - 将修改后的文件夹发送到其他两台机器

    - ```
      cd ~
      scp -r hadoop-2.7.2 10.211.55.21:/home/hadoop
      scp -r hadoop-2.7.2 10.211.55.22:/home/hadoop
      ```

  - 登录到slave1 修改/etc/hadoop/yarn-site.xml

    - ```
        <property>
          <name>yarn.resourcemanager.ha.id</name>
          <value>rm2</value>
        </property>
      ```

  - 登录到slave2 。 对/etc/hadoop/yarn-site.xml删除上面的键值对

- 初始化

  - 在三台机器上使用命令启动journalnode

    - ```
      cd /home/hadoop/hadoop-2.7.2
      ./sbin/hadoop-daemon.sh start journalnode
      ```

  - 在master机器上执行

    - ```
      cd /home/hadoop/hadoop-2.7.2
      ./bin/hdfs namenode -format
      ./hdfs zkfc -formatZK
      ```

      这个时候目录在目录上会出现一个tmp目录(/home/hadoop/hadoop-2.7.2/tmp)

      将该目录发送到slave1,/home/hadoop/hadoop-2.7.2文件夹中

      ```
      scp -r tmp 10.211.55.21:/home/hadoop/hadoop-2.7.2
      ```

  - 在slave1 机器上执行

    - ```
      cd /home/hadoop/hadoop-2.7.2
      ./bin/hdfs namenode -bootstrapStanby
      ```

      

- 在maser机器上启动hadoop集群

  - ```
    cd /home/hadoop/hadoop-2.7.2
    ./sbin/start-dfs.sh
    ./sbin/start-yarn.sh
    ```

    如果正常启动成功，那么在master执行jps,会出现

    ```
    24624 DataNode
    24179 JournalNode
    25075 NodeManager
    25977 NameNode
    1147 Jps
    24974 ResourceManager
    ```

#### hbase

- 登录到master,进入目录hbase配置目录

  - ```
    cd /home/hadoop/hbase-1.4.9/conf
    ```

- vim hbase-env.sh

  - ```
    //配置JDK
    export JAVA_HOME=/opt/jdk
    
    //配置hbase 启动进程id
    export HBASE_PID_DIR=/home/hadoop/hbase-1.4.9/pids
    
    //不用自带的zk
    export HBASE_MANAGES_ZK=false
    ```

- vi hbase-site.xml

  - ```
    <configuration>
      <!-- 设置HRegionServers共享目录，请加上端口号 -->
      <property>
        <name>hbase.rootdir</name>
        <value>hdfs://master:9000/hbase</value>
      </property>
    
      <!-- 指定HMaster主机 -->
      <property>
        <name>hbase.master</name>
        <value>hdfs://master:60000</value>
      </property>
    
      <!-- 启用分布式模式 -->
      <property>
        <name>hbase.cluster.distributed</name>
        <value>true</value>
      </property>
    
      <!-- 指定Zookeeper集群位置 -->
      <property>
        <name>hbase.zookeeper.quorum</name>
        <value>master:2181,slave1:2181,slave2:2181</value>
      </property>
    
      <!-- 指定独立Zookeeper安装路径 -->
      <property>
        <name>hbase.zookeeper.property.dataDir</name>
        <value>/home/hadoop/zookeeper-3.4.14</value>
      </property>
    
      <!-- 指定ZooKeeper集群端口 -->
      <property>
        <name>hbase.zookeeper.property.clientPort</name>
        <value>2181</value>
      </property>
    </configuration>
    ```

- vi regionservers

  ```
  master
  slave1
  slave2
  ```

  

- 创建pids

  ```
  mkdir -p /home/hadoop/hbase-1.4.9/pids
  ```

- 将hbase发送到其他机器上

  ```
  cd ~
  scp -r hbase-1.4.9 10.211.55.21:/home/hadoop
  scp -r hbase-1.4.9 10.211.55.22:/home/hadoop
  ```

- 启动hbase

  ```
  cd /home/hadoop/hbase-1.4.9
  ./bin/start-hbase.sh
  ```

  如果启动成功，执行hbase shell 出现下面提示，即表示成功.

  ```
  -> % ./bin/hbase shell
  2019-11-01 16:37:41,780 WARN  [main] util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
  HBase Shell
  Use "help" to get list of supported commands.
  Use "exit" to quit this interactive shell.
  Version 1.4.9, rd625b212e46d01cb17db9ac2e9e927fdb201afa1, Wed Dec  5 11:54:10 PST 2018
  
  hbase(main):001:0>
  ```

#### 注意

- 如果hadoop namenode 全部是standy,需要手动指定active （master机器）

  ```
  cd  /home/hadoop/hadoop-2.7.2
  ./bin/hdfs haadmin -transitionToActive --forcemanual master
  ```



#### 参考（都是大佬啊）

[Hadoop HA高可用集群搭建（Hadoop+Zookeeper+HBase）]: https://segmentfault.com/a/1190000012486628
[一脸懵逼学习Hadoop分布式集群HA模式部署（七台机器跑集群）]: https://cloud.tencent.com/developer/article/1010794
[HADOOP HA 踩坑 - 所有 namenode 都是standby]: https://www.cnblogs.com/PigeonNoir/p/9118003.html
[部署hadoop2.7.2 集群 基于zookeeper配置HDFS HA+Federation]: https://blog.csdn.net/gao634209276/article/details/51453456



