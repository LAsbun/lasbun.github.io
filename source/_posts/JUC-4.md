---
title: '聊聊java并发编程[JUC中的原子操作类]'
tags:
  - java
  - atomic
date: 2020-04-29 00:05:27
---


通过本片文章，可以了解到JUC中的原子类的不同种类，其原理。

<!-- more -->

#### 为什么要有原子操作类
- 前面我们讲过，java并发编程主要解决三个问题，可见性，有序性以及原子性。
- 对于可见性以及有序性，都可以通过volatile来解决。 原子性可以通过sychronized来修饰保证代码块的原子性，但是如果是使用volatile修饰的变量，其原子性无法保证。
- 证明
	```
public class ValotileAtomicTest {

    static volatile int a = 0;
//    AtomicInteger a = new AtomicInteger(0);

    @Test
    public void testVolatileUnSafeAtomic() throws InterruptedException {
        Thread[] threads = new Thread[50];
        for (int i = 0; i < 50; i++) {
            threads[i] = new Thread(new Runnable() {
                @Override
                public void run() {
                    for (int j = 0; j < 1000; j++) {
                        ++a;
//                        a.incrementAndGet();
//                        System.out.println(++a);
                    }
                }
            });
        }

        for (int i = 0; i < 50; i++) {
            threads[i].start();

        }
        for (int i = 0; i < 50; i++) {
            threads[i].join();

        }

        System.out.println(a);
    }
}
// 最后的结果 是 48618(机器不一样，可能结果就不一样，可以多跑几次，如果机器性能很好不一定能够看出来不是原子性的)
	```
- 看到上面的结果是不是很奇怪。正常来讲最后的结果应该是50000。
	- 实际上是因为 `++a` 看似是一个操作，但是实际上并不是一个操作
		- 先从主内存中读取a的值
		- 再把a的值+1
		- 再刷新到主存中
	- 如果是多线程，就会有下面的场景出现
		- 1 线程1先读取了a的值，然后被阻塞
		- 2 线程2也先读取了a的值，但是没有被阻塞，然后+1，刷新到内存
		- 3 线程1 解除阻塞，+1，刷新到主内
		- 上述3个流程，实际是两个自增，但是只增加了1次。
- 原子操作类就是为了解决这种问题的。有兴趣的同学，可以把上面的AtomicInteger 取消注释，测试下，每次的结果都是理想中的50000.

#### 原子操作类的分类(按照具体的场景)

##### 基本类型
主要是更新基础类型
- AtomicInteger 整型
- AtomicLong 长整型原子类
- AtomicBoolean 布尔类型

##### 数组类型
通过原子的方式更新数组中的某个元素
- AtomicIntergeArray 整型数组
- AtomicLongArray  长整形数组
- AtomicReferenceArray 引用类型数组原子类

##### 原子更新引用类型
通过原子性的更新多个变量
- AtomicReference 原子更新引用类型
- AtomicReferenceFieldUpdater 原子更新引用类型里的字段
- AtomicMarkableReference 原子更新带标记为的引用类型

##### 原子更新字段类
更新某个类的某个字段
- AtomicIntegerFileUpdater 原子更新整型字段的更新器
- AtomicLongFieldUpdater 原子更新长整型字段的更新器
- AtomicStampedReference 原子更新带有版本号的引用类型。解决ABA

#### 原理
其实翻看下源码，getAndSet是利用CAS + Unsafe.compareAndSwapXXX() 算法。

CAS 历史文章有聊过这里不在赘述。
##### Unsafe
看下Unsafe的源码，有三个compareAndSwap,分别是:
```
	public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

    public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);

```
都是引用的native方法。

需要注意的是：
- 如果是AtomicBoolean 会先转换成int，然后使用compareAndSwapInt. 其他的char也可以利用此方法。 如果是其他的类可以可以使用AtomicReference进行原子性更改。

- Unsafe 之所以是 Unsafe 是因为其直接是从内存的地址读取offset，并进行修改。

#### 参考大佬
- 《java并发编程的艺术》方腾飞　魏鹏　程晓明　著