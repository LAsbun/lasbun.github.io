---
title: thread-state
date: 2019-11-25 20:36:07
tags:  java, Thread-State
---

本文章介绍 Java Thread State各种状态.通过本篇文章你可以了解到：
1 Java 线程中都有哪些状态
2 这些状态之间的联系是怎样的
3 使用jstack查看运行中的线程状态

<!-- more -->

Java 线程中的状态
---
其实只要看下Thread.State源码就可以知道有几种类型了

```
public enum State {
        /**
         * 初始状态。线程刚创建，还没有调用start()
         */
        NEW,

        /**
         * 正在运行的状态（java 将系统中的Ready和运行都笼统的成为运行中）。正在运行的线程有可能处于等待状态。例如等待系统io
         */
        RUNNABLE,

        /**
         * 阻塞状态。等待锁的释放
         */
        BLOCKED,

        /**
         * 等待状态
         * 线程变为等待状态，分为下面三种情况
         * 1 Object.wait() 并且没有timeout参数
         * 2 Thread.join 没有timeout参数
         * 3 LockSupport.park()
				 * 例如：
         *  一个线程调用了Object.wait(). 直到另外的线程调用了该Object.notify()或者是Object.notify()方法才会解除waiting状态。
         */
        WAITING,

        /**
         * 超时等待状态。等待一定时间后，自动释放该状态
         * 例如：调用了下面的函数并有时间相关参数
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * 线程终止状态
         */
        TERMINATED;
    }
```
这些状态之间的联系是怎样的
---
具体见下图（图源《Java 并发编程艺术》4.1.4节）
{% asset_img image-20191125232222723.png [title] %}

从上图可以到

1 初始化线程状态表示为New. 调用Thread.start()转换状态为RUNNABLE

2 调用Object.wait()/Object.join()/LockSupport.park会由RUNNABLE转换为WAITING转态。调用Object.notify()/Object.notifyAll()/LockSupport.unpark()才会由，WAITING转为RUNABLE

3 调用Thread.sleep(long)/Object.wait(long)/Thread.join(long)/LockSupport.pardNanos(long)/LockSupport.parkUntil(long) 由 RUNNABLE-> TIMED_WAITING

反之通过Object.notify()/Object.notifyAll()/LockSupport.unpark()才会由，TIMED_WAITING转为RUNABLE

4 在调用synchronized函数时候，未获取到锁，会变为BLOCKED。 获取到之后即变为RUNNABLE

5 执行完成之后转换为TERMINATED

使用jstack查看运行中的线程状态
----
- NEW
  ```
  @Test
  public void testNew() {

    Thread thread = new Thread();
    System.out.println(thread.getState()); // NEW

  }
  ```

- RUNNABEL && WAITING
```
  @Test
  public void testRunnable() throws InterruptedException {

    Thread testRunnable = new Thread(new Runnable() {
      @Override
      public void run() {
        for (int i = 0; i < Integer.MAX_VALUE; i++) {
          System.out.println(i);
        }
      }
    }, "testRunnable");
    testRunnable.start();
    testRunnable.join();
  }

```

testRunnable 线程是在运行
{% asset_img image-20191125234826763.png [title] %}

main此时状态为WAITING.  因为调用了thread.join函数
{% asset_img image-20191125235543225.png [title] %}

BLOCKED && WAITING

```
  public static void main(String[] args){

    final Object lock = new Object();

    Thread waitingA = new Thread(new Runnable() {
      @Override
      public void run() {
        synchronized (lock) {
          try {
            Thread.sleep(20000);
            System.out.println("waiting");
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      }

    }, "waitingA");

    Thread waitingB = new Thread(new Runnable() {
      @Override
      public void run() {
        synchronized (lock) {
          try {
            Thread.sleep(20000);
            System.out.println("waiting");
          } catch (InterruptedException e) {
            e.printStackTrace();
          }
        }
      }

    }, "waitingB");


    waitingA.start();
    waitingB.start();

  }
```
可以看下图中 A TIMED_WAITING B因为A获取到了lock，而为BLOCKED状态
{% asset_img image-20191126001024171.png [title] %}

