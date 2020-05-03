---
title: '聊聊java并发编程[JUC中的锁]'
tags:
  - java
  - lock
  - ReentrantLock
  - ReentrantReadWriteLock
date: 2020-05-03 10:56:35
---


早些时候找了下[java中锁分类](https://lasbun.github.io/2019/12/23/java/JAVA中的锁/), 今天我们聊下java中的锁

<!-- more -->

#### 场景
在计算机科学中，锁是在执行多线程时用于强行限制资源访问的同步机制，即用于在并发控制中保证对互斥要求的满足。---> 来自wikipedia

#### sychronized vs Lock

| 类型                  | sychronized              | Lock                     |
| --------------------- | ------------------------ | ------------------------ |
| 解决的问题            | 对访问资源进行限制       | 同sychronized            |
| 实现                  | 内置关键字               | 基于AQS，实现接口        |
| 锁的获取与释放 可见性 | 隐式(手动加锁，隐式释放) | 显式(需要手动获取与释放) |
| 是否支持中断          | 不支持                   | 支持                     |

#### demo
实现一个独占式锁
```
public class Mutex implements Lock {

    private static class Sync extends AbstractQueuedSynchronizer {

        @Override
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        public boolean tryAcquire(int acquire) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        public boolean tryRelease(int release) {
            if (getState() == 0) throw new IllegalMonitorStateException();
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        Condition newCondition() {
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    @Override
    public void lock() {
        sync.acquire(1);
    }

    @Override
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(time));
    }

    @Override
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }


    public static void main(String[] args) throws InstantiationException, IllegalAccessException, InterruptedException {

        Mutex mutex = new Mutex();


        Thread x = ObjectLock.class.newInstance();
        x.start();

        Thread t1 = mutex.new FuncLock2(false, mutex);
        Thread t2 = mutex.new FuncLock2(true, mutex);
        Thread t3 = mutex.new FuncLock2(true, mutex);


        t1.start();
        t2.start();
        t3.start();
        t1.join();
        t2.join();
        t3.join();

        System.out.println(mutex.testFinally());


    }


    public boolean testFinally(){
        try{
            return true;
        }
        finally {
            System.out.println("dsadsa");
        }

    }

    public class FuncLock2 extends Thread {

        private Mutex mutex;

        private boolean boo;

        public FuncLock2(boolean b, Mutex mutex) {
            this.boo = b;
            this.mutex = mutex;
        }

        @Override
        public synchronized void run() {
            if (this.boo) {
                this.read();
            } else {
                this.write();
            }

        }

        private void read() {
            mutex.lock();
            try {
                System.out.println(Thread.currentThread().getName() + " read lock ing...");
                for (int i = 0; i < 5; i++) {
                    System.out.println(mutex.sync.getQueuedThreads());
                    Thread.sleep(1000);
                }
            } catch (Exception e) {

            } finally {
                System.out.println(Thread.currentThread().getName() + " read lock end...");

                mutex.unlock();
            }

        }

        private void write() {
            mutex.lock();

            try {
                System.out.println(Thread.currentThread().getName() + " write lock ing..");
                for (int i = 0; i < 5; i++) {
                    System.out.println(mutex.sync.getQueuedThreads());
                    Thread.sleep(1000);
                }

            } catch (Exception e) {

            } finally {
                System.out.println(Thread.currentThread().getName() + " write end ing..");

                mutex.unlock();
            }
        }
    }


}
```

#### 原理
其实就是基于AQS 内置的一部分参数，然后实现tryAcquire，acquire等接口。 具体的可以看下[这篇关于AQS的总结](https://lasbun.github.io/2020/04/09/JAVA-AQS/). 理解AQS原理其实就基本把JUC中的锁摸的差不多了。
接下来我们主要的聊下公平锁，非公平锁，重入锁，重入读写锁。

#### 公平锁 vs 非公平锁
这两个是两种实现方式，一般锁内置的实现都是会有实现这两种。比如重入锁，重入读写锁内部都有公平与非公平方式的实现

“公平性与否是针对获取锁而言的，如果一个锁是公平的，那么锁的获取顺序就应该符合请求的绝对时间顺序，也就是FIFO。”

#### 重入锁
顾名思义就是一个线程在获取到锁之后，再次获取锁。
sychronized 也是可重入锁。
重入锁其实是独占式状态共享
##### demo
- 打印当前等待的线程
- 主要接口就是lock() && unlock
```
public class ReenTrantLockTest {

    static ReentrantLock reentrantLock = new ReentrantLock2(true);

    static ReentrantLock reentrantLockUnfair = new ReentrantLock2(false);

    @Test
    public void testReenTrantLockUnFairAndFair() throws InterruptedException {

        Thread[] threads = new Thread[5];

        for (int i = 0; i < 5; i++) {
            threads[i] = new Thread(new Job(reentrantLock));
            threads[i].start();
        }

        for (int i = 0; i < 5; i++) {
            threads[i].join();
        }

    }

    private static class Job extends Thread {
        private Lock lock;

        public Job(Lock lock) {
            this.lock = lock;
        }

        public void run() {
            for (int i = 0; i < 2; i++) {
                lock.lock();

                try {

                    System.out.print("获取锁的当前线程[" + Thread.currentThread().getName() + "]");
                    System.out.print("等待队列中的线程[");
                    ((ReentrantLock2) lock).getQueueThreads();
                    System.out.println("]");
                    reenDoSth();
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    lock.unlock();
                }
            }

        }

        private void reenDoSth() {
            lock.lock();
            try {
                System.out.println("reenDoSth[" + Thread.currentThread().getName() + "]");
            } catch (Exception e) {

            } finally {
                lock.unlock();
            }
        }
    }

    private static class ReentrantLock2 extends ReentrantLock {
        public ReentrantLock2(boolean fair) {
            super(fair);
        }

        public void getQueueThreads() {
            ArrayList<Thread> threads = new ArrayList<>(super.getQueuedThreads());
            threads.forEach(thread -> System.out.print(thread.getName()));
        }
    }
}
```
##### 源码分析
- 重入锁是一个典型的独占式锁

###### 非公平锁

####### lock
######## NonfairSync.lock 
```
        /**
         * 获取到锁
         */
        final void lock() {
        		// 如果可以直接获取到锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
            		// 没有获取到锁， acquir会掉用下面的tryAcquire。实际是会调用 Sync.nonfairTryAcquire 方法
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
```

######## Sync.nonfairTryAcquire
```
/**
         * 非公平锁的获取到锁
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 如果当前的状态是0, 则表示是可获取状态
            if (c == 0) {
            		// 在多线程情况下，有可能其他线程已经获取到锁，所以要用CAS尝试赋值
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果当前锁已经被占用了，尝试判断是否是当前线程，如果是当前线程，则表示可以再次获取到锁. 这里就是重入锁了。
            // 仔细想一下这里其实就是非公平锁的非公平，如果某个线程获取到锁了，与此同时如果再有新的(下面会将为什么一定是新的线程)个线程尝试获取锁，就有可能直接获取到锁，继续进行下去。
            // 为什么一定是新的线程：因为AQS某个线程(节点)获取到锁的前提是，其前置节点为head 并且tryAcquire获取到锁，才会继续执行下去。具体可以看前置的AQS源码梳理
            else if (current == getExclusiveOwnerThread()) {
            		// 这里为什么是c+acquires: 因为有可能在该线程获取到的时候，已经存在其他线程获取到重入锁了
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

####### lock
######## NonfairSync.unlock 
- 实际调用的是Sync.release
```
	public final boolean release(int arg) {
				// 释放锁。
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
            		// 唤醒后继节点
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
 
 protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // 这里c == 0 才表示该线程所有获取到锁的都已经释放掉了
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```




###### 公平锁
####### lock
######## FairSync.lock 
- 公平锁的获取稍微有点不一样
```
				protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
            		// 比非公平锁多了一个一个  !hasQueuedPredecessors()  其他都一样
            		// 就是判断是否有前置节点，就是公平锁的本质，一定是按照顺序来的
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```
####### unlock
公平锁的释放与非公平锁的释放共用的一套代码，这里就不赘述了

##### 重入锁总结
- 重入锁其实主要是依赖来AQS的内部实现。
- 公平锁的释放与非公平锁的释放是一样的
- 公平锁的获取与非公平锁的获取唯一的不同在与公平锁在判断是否可以获取到锁的时候，需要判断**是否有前置节点**

#### 读写锁
##### demo
- 读写操作夹杂，然后看当前等待的读写线程
```
public class ReenTrantReadWriteLockTest {

    static Map<String, Integer> cache = new HashMap<>();

    static ReentrantReadWriteLock reentrantReadWriteLockUnFair = new ReentrantReadWriteLock2(false);
    static ReentrantReadWriteLock reentrantReadWriteLockFair = new ReentrantReadWriteLock2(true);

    @Test
    public void testWRLock() throws InterruptedException {

        int index = 0;

        int length = 14;

        Thread[] threads = new Thread[length];
        for (int i = 0; i < 10; i++) {

            threads[index++] = new ReadJob(reentrantReadWriteLockFair);
            if (i % 3 == 0) {
                threads[index++] = new WriteJob(reentrantReadWriteLockFair);
            }

        }

        for (int i = 0; i < length; i++) {
            threads[i].start();
        }

//        reentrantReadWriteLockFair.getQueueLength()

        for (int i = 0; i < length; i++) {
            threads[i].join();
        }

    }


    static class WriteJob extends Thread {

        ReadWriteLock readWriteLock;

        public WriteJob(ReadWriteLock readWriteLock) {
            this.readWriteLock = readWriteLock;
        }

        public void run() {
            readWriteLock.writeLock().lock();
            try {
                Thread.sleep(1000);
                String s = Thread.currentThread().getId() + "-" + cache.get(Thread.currentThread().getName());

                cache.put(Thread.currentThread().getName(), ((ReentrantReadWriteLock2) readWriteLock).getQueueLength());

                System.out.println("write[" + Thread.currentThread().getName() + "] " + s);
                System.out.println("[" + Thread.currentThread().getName() + "] read " + s);
                System.out.print("等待队列中的线程[");
                ((ReentrantReadWriteLock2) readWriteLock).getWaitQueue();
                System.out.println("]");

                read();

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                readWriteLock.writeLock().unlock();
            }
        }

        private void read() {

            readWriteLock.readLock().lock();
            try {

                Integer s = cache.get(Thread.currentThread().getName());
                System.out.println("[" + Thread.currentThread().getName() + "] read " + ((ReentrantReadWriteLock2) readWriteLock).getQueueLength());
//                System.out.print("等待队列中的线程[");
//                ((ReentrantReadWriteLock2) readWriteLock).getWaitQueue();
//                System.out.println("]");

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                readWriteLock.readLock().unlock();
            }
        }

    }

    static class ReadJob extends WriteJob {

        public ReadJob(ReadWriteLock readWriteLock) {
            super(readWriteLock);
        }

        public void run() {
            super.read();
        }
    }


    private static class ReentrantReadWriteLock2 extends ReentrantReadWriteLock {
        public ReentrantReadWriteLock2(boolean fair) {
            super(fair);
        }

        public void getWaitQueue() {
        		// 打印等待写的线程
            super.getQueuedWriterThreads().forEach(thread -> System.out.print(thread.getName()));
            System.out.println();
            // 打印等待读的线程
            super.getQueuedReaderThreads().forEach(thread -> System.out.print(thread.getName()));
        }
    }


```
#### 读写锁
重写读写锁 是共享式状态共享(读) + 独占式状态共享(写)
- 举例A(R)->B(w)->C(R)->D(R)->E(W)->F(R)
	- 正确的获取应该是
		- A获取到锁  其他线程等待
		- B获取到锁(CDEF等待)
		- C，D获取到锁(EF等待)
		- E获取到锁(F等待)
		- F获取到锁 
##### demo
- 下面的demo 是读写夹杂，打印等待读写的线程
```
public class ReenTrantReadWriteLockTest {

    static Map<String, Integer> cache = new HashMap<>();

    static ReentrantReadWriteLock reentrantReadWriteLockUnFair = new ReentrantReadWriteLock2(false);
    static ReentrantReadWriteLock reentrantReadWriteLockFair = new ReentrantReadWriteLock2(true);

    @Test
    public void testWRLock() throws InterruptedException {

        int index = 0;

        int length = 14;

        Thread[] threads = new Thread[length];
        for (int i = 0; i < 10; i++) {

            threads[index++] = new ReadJob(reentrantReadWriteLockFair);
            if (i % 3 == 0) {
                threads[index++] = new WriteJob(reentrantReadWriteLockFair);
            }

        }

        for (int i = 0; i < length; i++) {
            threads[i].start();
        }

//        reentrantReadWriteLockFair.getQueueLength()

        for (int i = 0; i < length; i++) {
            threads[i].join();
        }

    }


    static class WriteJob extends Thread {

        ReadWriteLock readWriteLock;

        public WriteJob(ReadWriteLock readWriteLock) {
            this.readWriteLock = readWriteLock;
        }

        public void run() {
            readWriteLock.writeLock().lock();
            try {
                Thread.sleep(1000);
                String s = Thread.currentThread().getId() + "-" + cache.get(Thread.currentThread().getName());

                cache.put(Thread.currentThread().getName(), ((ReentrantReadWriteLock2) readWriteLock).getQueueLength());

                System.out.println("write[" + Thread.currentThread().getName() + "] " + s);
                System.out.println("[" + Thread.currentThread().getName() + "] read " + s);
                System.out.print("等待队列中的线程[");
                ((ReentrantReadWriteLock2) readWriteLock).getWaitQueue();
                System.out.println("]");

                read();

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                readWriteLock.writeLock().unlock();
            }
        }

        private void read() {

            readWriteLock.readLock().lock();
            try {

                Integer s = cache.get(Thread.currentThread().getName());
                System.out.println("[" + Thread.currentThread().getName() + "] read " + ((ReentrantReadWriteLock2) readWriteLock).getQueueLength());
//                System.out.print("等待队列中的线程[");
//                ((ReentrantReadWriteLock2) readWriteLock).getWaitQueue();
//                System.out.println("]");

            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                readWriteLock.readLock().unlock();
            }
        }

    }

    static class ReadJob extends WriteJob {

        public ReadJob(ReadWriteLock readWriteLock) {
            super(readWriteLock);
        }

        public void run() {
            super.read();
        }
    }


    private static class ReentrantReadWriteLock2 extends ReentrantReadWriteLock {
        public ReentrantReadWriteLock2(boolean fair) {
            super(fair);
        }

        public void getWaitQueue() {
            super.getQueuedWriterThreads().forEach(thread -> System.out.print(thread.getName()));
            System.out.println();
            super.getQueuedReaderThreads().forEach(thread -> System.out.print(thread.getName()));
        }
    }

}
```
##### 原理剖析
详细的剖析可以看[这篇文章](https://www.cnblogs.com/xiaoxi/p/9140541.html)
我这边主要聊几个关键点

- 读写锁是可重入锁

- 状态的设计
	- 一共32位。 高16位为读锁的状态 ，底16位为写锁的状态。
	- 所以判断读写锁，根据位运算计算即可。
- 写锁
	- 排它锁
	- 获取的时候需要判断是否有读锁在获取，如果有，则获取失败.（如果可以获取成功，就是锁升级，ReentrantWriteReadLock 不支持）

- 读锁
	- 共享锁
	- 获取的时候，如果存在写锁，并且是当前线程，则是可以获取到锁的（锁降级） 。
- HoldCounter
	- 共享锁从某一角度来讲，就是一个计数器。这个计数器是于当前线程绑定的。 而HoldCounter可以理解为是这个计数器(原理是ThreadLocalCache)

#### 参考大佬
- [ReentrantReadWriteLock读写锁详解](https://www.cnblogs.com/xiaoxi/p/9140541.html)
- 《java并发编程的艺术》方腾飞　魏鹏　程晓明　著
