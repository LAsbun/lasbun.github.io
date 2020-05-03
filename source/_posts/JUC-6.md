---
title: '聊聊java并发编程[JUC中的Condition]'
tags:
  - java
  - condition
date: 2020-05-03 18:14:19
---


本文聊下JUC中的Condition使用，以及源码实现

<!-- more -->

#### 什么是Condition
- Condition 实际是等待/通知模型的一种实现。
- 聊synchronized原理的时候，我们有聊到java中每个对象都有一个`monitor`, 被synchronized修饰的代码块/方法，在执行的时候，会先尝试获取到对应的`monitor`，如果没有获取到，则使用`wait`阻塞(重量级锁)，待其他获取到`monitor`的线程释放掉`monitor`，调用`notify`再尝试重新获取`signal`。`wait`/`notify`方法就是所谓的等待/通知模型。
- synchronized 做为隐式的，有对应的显示实现Lock。
- `monitor`的等待/通知 ，同样有其对应的实现，即Condition接口。

#### Condition vs  `monitor`

| 对比名称                       | monitor            | Condition                                                   |
| ------------------------------ | ------------------ | ----------------------------------------------------------- |
| 前置条件                       | 获取到对象的锁     | 调用Lock.lock()获取到锁，Lock.newCondition创建Condition对象 |
| 调用方式                       | object.wait/notify | condition.await/signal                                      |
| 等待队列个数                   | 1个                | 多个                                                        |
| 是否支持等待中，不响应中断     | 不支持             | 支持                                                        |
| 是否支持等待到某个具体的时间点 | 不支持             | 支持(awaitUtile接口)                                        |

#### demo
构建一个有界队列
创建两个条件:notEmpty（非空） notFull(非满)
创建两个线程：一个生成，一个消费
```
public class ConditionTest {

    class BoundedQueue<T> {
        private Object[] items;
        private int addIndex, removeIndex, count;
        private Lock lock = new ReentrantLock();
        private Condition notEmpty = lock.newCondition();
        private Condition notFull = lock.newCondition();

        public BoundedQueue(int size) {
            this.items = new Object[size];
        }

        public void add(T t) throws InterruptedException {
            lock.lock();
            try {
                while (count == items.length) {
                    // 表示满了
                    System.out.println(Thread.currentThread().getName() + " is waiting empty");
                    notFull.await();
                }
                items[addIndex] = t;
                if (++addIndex == items.length) {
                    addIndex = 0;
                }
                ++count;
                // 表示非空
                notEmpty.signal();
            } finally {
                lock.unlock();
            }
        }

        @SuppressWarnings("unchecked")
        public T pop() throws InterruptedException {
            lock.lock();
            try {

                while (count == 0) {
                    System.out.println(Thread.currentThread().getName() + " is waiting pop");
                    notEmpty.await();
                }

                Object t = items[removeIndex];

                if (++removeIndex == items.length) {
                    removeIndex = 0;
                }
                --count;
                notFull.signal();
                return (T) t;

            } finally {
                lock.unlock();
            }
        }
    }


    @Test
    public void testCondition() throws InterruptedException {

        BoundedQueue<String> boundedQueue = new BoundedQueue<>(5);

        ProductJob productJob = new ProductJob(boundedQueue);
        ConsumeJob consumeJob = new ConsumeJob(boundedQueue);
        productJob.start();
        consumeJob.start();

        productJob.join();
        consumeJob.join();
    }


    class ProductJob extends Thread {
        private BoundedQueue boundedQueue;

        public ProductJob(BoundedQueue boundedQueue) {
            this.boundedQueue = boundedQueue;
        }

        public void run() {
            while (true) {

                try {
                    boundedQueue.add("11");
                    // 每2s 生产一个
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }
    }

    class ConsumeJob extends Thread {
        private BoundedQueue boundedQueue;

        public ConsumeJob(BoundedQueue boundedQueue) {
            this.boundedQueue = boundedQueue;
        }

        public void run() {
            while (true) {

                try {
                    System.out.println(boundedQueue.pop());
                    // 每2s 生产一个
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

            }
        }
    }

}
```

#### 原理剖析

##### Condition 接口
```
public interface Condition {
		//等待获取到锁，或者是被中断(被其他线程)。 从await返回一定是当前线程获取到锁了
    void await() throws InterruptedException;
		// 等待 不响应中断
    void awaitUninterruptibly();
		// 等待/中断超时,与下面的区别是返回的是举例等待的时间有多少
    long awaitNanos(long var1) throws InterruptedException;
		// 返回是否等待/中断超时
    boolean await(long var1, TimeUnit var3) throws InterruptedException;

		// 等待/中断到某个确切的时间点
    boolean awaitUntil(Date var1) throws InterruptedException;
		// 通知下一个
    void signal();
		// 通知所有的
    void signalAll();
}
```
- Condition接口在JUC中的实现是AQS中的ConditionObject

##### ConditionObject
ConditionObject 实现Condition主要有三个点： 等待队列，等待，通知
###### 等待队列
- 一个Condition 包含一个等待队列。
- 等待队列的节点是复用的AQS.Node
- 下面是ConditionObject 的属性 
```java
        /** 首节点 */
        private transient Node firstWaiter;
        /** 尾节点 */
        private transient Node lastWaiter;

```
- 等待队列
	{% asset_img image-20200503172612131.png %}

###### 等待队列与AQS的关系
- 具体见下图
    {% asset_img image-20200503174511454.png %}

###### 等待(await)
- 源码
```java
public final void await() throws InterruptedException {

            if (Thread.interrupted())
                throw new InterruptedException();
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // interrupt 是为了判断是中断异常，还是人为中断
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
```
- 流程图
	{% asset_img image-20200503174040896.png %}
- 注意
	- 因为wait是在获取到锁之后才被调用的，所以没有用CAS 

###### 通知(signal)
- 源码
```
	public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }
  private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
```
- 流程图
	{% asset_img image-20200503180715548.png %}

#### 参考大佬
- 《java并发编程的艺术》方腾飞　魏鹏　程晓明　著