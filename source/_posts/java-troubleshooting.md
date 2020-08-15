---
title: 聊聊java线上问题排查
tags:
  - java
  - 线上问题
date: 2020-08-15 19:32:21
---


因为最近有次是疑似内存溢出，特地去线上排查了下。本篇是总结性的把如何排查Java线上问题做个汇总

<!--  more -->

#### 内存溢出如何查

* 什么是内存溢出
	* 简单讲:内存溢出就是在你申请内存的时候，内存不够了。 
	* 现象是错误栈带有`OutOfMemoryError`
	* 出现原因
		* 简单讲分为2种
			* 一种是因为内存泄露(认为原因) 导致内存一直在增加 (解决办法是排查内存溢出的代码块，修复调即可)
			* 另外一种是因为设计上的原因，导致一次性加载数据过多，导致内存溢出(解决办法是有两个: 1 优化代码(不要一次性读取那么多) 2 增加jvm运行内存) 
* 什么是内存泄露
	* 简单讲:内存泄露就是申请到了内存，但是因为某些原因一直无法释放(应该是要释放)。这就是内存泄露

##### 线上问题排查步骤

######  使用`jps` 确定当前进程
{% asset_img image-20200815181334361.png %} 	
###### 使用`jstat` 查看当前gc的频率
- 使用`jstat [ option vmid [interval[s|ms] [count]] ]`
	- `jstat -gcutil 24988 1000` // 对进程pid 24988 查看gc 每1000ms打印一次gc日志
	- {% asset_img image-20200815185816670.png %}
	- 图片中的参数解释
		
		- 这台服务器的新生代Eden区（E，表示Eden）使用了16.81%（最后）的空间，两个Survivor区（S0、S1，表示Survivor0、Survivor1）分别是0和20.73%，老年代（O，表示Old）使用了43.43%。程序运行以来共发生Minor GC（YGC，表示Young GC）19283次，总耗时212.222秒，发生Full GC（FGC，表示Full GC）7次，Full GC总耗时2.741秒，总的耗时（GCT，表示GC Time）为214.963秒。
	
###### 找出疯狂FullGC的原因
使用jmap命令可以在线简单分析下，也可以dump当时内存情况然后离线分析。

####### 在线分析
* 第一种 使用jmap
  使用`./jmap -histo:live pid |head -n num` 查看当前程序中存活的对象实例数，以及持有的内存。
  {% asset_img image-20200815190649881.png %}

  上面看起来程序是正常的。 

  根据参考的大佬博文中的图
  ![img](https://user-gold-cdn.xitu.io/2018/6/22/164263a7ef8ccb83?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
  从上面的图可以看出来，HashTable 有大概5000w+的实例数，以及1.5G左右的数据，大概率是有问题的。
  
* 第二种 使用 阿里开源的arthas 。https://alibaba.github.io/arthas/ 可参考教程。arthas提供了丰富的排查命令，具体可以通过链接看下文档. 我下面只放了`dashboard` 运行后的情况,可以看到线程的执行状态，内存的使用情况都是一目了然
	{% asset_img image-20200815191319061.png %}

* 第三种 把线上的内存情况使用`jmap -dump:format=b,file=xx.dump pid` 把线上内存情况保存下来。 然后从服务器上同步到本地，本地使用`jvisualvm` 或者是eclips 的`MAT`来分析。具体可以google下使用[教程](https://blog.csdn.net/lkforce/article/details/60878295)，这里就不在细讲了。

#### cpu 飙升怎么查

##### 第一步 找到cpu飙升的线程
使用`ps p pid -L -o pcpu,pid,tid,time,tname,cmd`  第一个pid 就是java进程pid
使用完之后
{% asset_img image-20200815192602615.png %}
找到cpu占用最多的那个tid。

##### 将tid转为16进制
使用`printf "%x\n" Tid` 例如`printf "%x\n" 36444` -> "8e5c".

##### 使用jstack 找到正在运行的代码块
`jstack -l <pid> | grep <thread-hex-id> -A 10` 命令显示出错的堆栈信息
{% asset_img image-20200815193028008.png %}




#### 参考大佬
- [一个Java内存泄漏的排查案例](https://juejin.im/entry/6844903623881654280?utm_source=gold_browser_extension)
