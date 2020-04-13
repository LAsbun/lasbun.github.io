---
title: java 基础学习之final finally finalize 区别
tags: [java]
date: 2019-12-14
---

final finally finalize 区别
<!-- more -->

- final finally finalize 区别
    - final (可以修饰 类 方法 变量)
      - 修饰类 
        - 表示不允许继承
          - final类中所有的成员方法都会隐式的定义为final方法。
      - 修饰方法
        - 表示子类不能重写此方法
          - 提高效率
          - 锁定方法
      - 修饰变量
        - final成员变量表示常量，只能被赋值一次，赋值后其值不再改变
    - finally(跟try...except连用)
      - 不关有没有异常都会执行。
        - 注意：
          - 只有当执行的时候，并且没有发生 线程被kill,断电，退出虚拟机等情况(这些情况不会执行finally)，finally才会执行.
        - 特殊
          - ```  
            private int testFinallyString(String str) {

              try {
                return str.charAt(0) - '0';
              } catch (NullPointerException e) {
                return 1;
              } catch (StringIndexOutOfBoundsException e) {
                return 2;
              } finally {
                return 6;
              }

            }
            
            // 调用函数
            System.out.println(testFinallyString(null) + " " + testFinallyString("0") + " " + testFinallyString(""));
            // 结果将会输出 6 6 6
            // 原因是因为 finally 特殊，会撤销之前的return.
          ```
    - finalize(在java.lang.Object里定义的)
      - 这个方法在gc启动，该对象被回收的时候被调用
      - 跟析构函数不一样.
      - 注意：
        - 这个跟c++ 中的析构函数是不一样的。 c++ 调用delete 时候，对象就会被删除掉。
        - java 里面的调用gc，也不一定会及时删除。而是根据下一个删除动作才会删除