---
title: java 基础学习之int 与 Integer 区别
tags: [java]
---

int 与 Integer 区别
<!-- more -->

int 与 Integer 区别
----
- 基本概念
  - java里面有基础类型和引用数据类型
    - (1）基本数据类型，分为boolean、byte、int、char、long、short、double、float； 
    - (2）引用数据类型 ，分为数组、类、接口。
    - 为什么只提到了 int和Integer 因为Integer内部装箱做了一个额外的处理(用了缓存).

- 基本对比
  - Integer 是 int 的包装类，int 是基本数据类型.
  - Integer 必须实例化才可以使用，int 可以直接赋值
  - Integer 实际是个类，所以是对象的引用，int是数据的存储
  - Integer 默认值为null, int 默认值为0. 并且int声明为0;
- 深入对比
  - Integer实际是对象，所以两个new的对象  是不等的(因为是new 内存地址不一样)。
    - ```
      Integer i = new Integer(100);
      Integer j = new Integer(100);
      System.out.println(i == j); //false
      
      // ①注意：如果是-128到127直接的变量赋值，则 nteger b = 5; // 与 int b = 5; 一样
      Integer a = 5;
      Integer b = 5; // 与 int b = 5; 一样
      int c = 5;
      System.out.println(a == c); // true
      System.out.println(a == b); // true
      System.out.println(a.equals(b));  // true

      // ②如果是超过这个范围 那么结果就不一样了

      Integer a = 599999999;
      Integer b = 599999999;
      int c = 599999999;
      System.out.println(a == c); // true
      System.out.println(a == b); // false  这个是false.
      System.out.println(a.equals(b));  // true

      // 出现①和②是因为 
       java在编译Integer a = 5 ;时，会翻译成为Integer a = Integer.valueOf(5)。而java API中对Integer类型的valueOf的定义如下，对于-128到127之间的数，会进行缓存，Integer a = 5 b = 5时，就会直接从缓存中取，就不会new了。

      见源码
          public static Integer valueOf(int i) {
            if (i >= IntegerCache.low && i <= IntegerCache.high)
                return IntegerCache.cache[i + (-IntegerCache.low)];
            return new Integer(i);
           }

    ```
