---
title: java自旋锁
date: 2020-01-16 16:19:02
tags: [java,自旋锁]
---

通过本篇文章，你可以了解到
    1 什么是CAS
    2 什么是自旋锁
    3 公平自旋锁
    4 MCS自旋锁
    5 CLH 自旋锁

<!-- more -->


#### 什么是CAS
	假设一种场景，你想修改某个共享变量的值，比如对该变量进行加1的操作，因为其他线程有可能已经更改了这个变量的值。如果直接进行+1,就有可能导致线程不安全。这个时候就可以使用CAS。 
	CAS compareAndSet/compareAndSwap  比较并且替换。一般使用compareAndSet(address,expect, update).对于某一内存地址address上的值，比较是否等于expect，如果等于就更新为update的值。 大部分的主流cpu都有支持相应的原子操作命令。
	通过CAS的特性，对数据进行修改就不会出现线程安全问题了(只是大概来讲，具体的线程安全还是要结合代码来讲)。
	其中J.U.C包中的原子类，都是支持CAS操作的。
#### 什么是自旋锁
	这里假设一个简单的锁场景。两个线程A，B.都在竞争一个资源C.资源C有且仅能被一个线程同时使用。那么这个时候如果A获取到C的时候(即A已经获取到了锁)，B在此时是无法获取到C的，但是这个时候有两种解决方案，
	一种就是线程挂起，放弃CPU时间片，等待被唤醒。
	另外一种就是不放弃时间片，轮询获取C. 第二种轮询获取资源C的方式就称为自旋锁.
	从这里就可以看出来，自旋锁最适用于时间较短的场景，如果是时间比较长，就白白的浪费了系统资源.
	demo
	```java
		public class SpinLock {
	
	private AtomicReference<Thread> owner = new AtomicReference<>();
	
	private void lock() {
	    Thread curThread = Thread.currentThread();
	    // 如果owner不为null，则表示此时锁被其他线程拥有，自旋获取锁
	    while (!owner.compareAndSet(null, curThread)) ;
	}
	
	private void unlock() {
	    owner.compareAndSet(Thread.currentThread(), null);
	}
	
	public static void main(String[] args) {
	
	    SpinLock spinLock = new SpinLock();
	
	    Thread t1 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	
	            try {
	                spinLock.lock();
	                System.out.println("dsadsa");
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                spinLock.unlock();
	
	            }
	        }
	    });
	
	    Thread t2 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	
	            try {
	                spinLock.lock();
	                System.out.println("94382843902834902");
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                spinLock.unlock();
	
	            }
	        }
	    });
	
	    t1.start();
	    t2.start();
	
	}

}
```
#### 公平点的自旋锁

	从上面的自旋锁中，我们不难看到一点，上面获取锁的方式其实是不公平的。大家都在竞争，谁运气好，谁就可以获取到锁。如果做到公平呢：其实就是给每一个获取锁的线程发个排号，锁就按照顺序依次给到相应的线程
	**demo**
	```java
	public class ReenLock {
	
		// 表示当前锁给到的序号
	private AtomicInteger serviceNum = new AtomicInteger();
		// 发号员
	private AtomicInteger ticketNum = new AtomicInteger();
	
	private int lock(){
	    int myticket = ticketNum.getAndIncrement(); // 先给个号
	    while (serviceNum.get()!=myticket){} // 如果目前的叫号不是自己，就等着
	
	    return myticket;
	}
	
	private void unlock(int myticket){
	
	    int next = myticket + 1;
	    serviceNum.compareAndSet(myticket, next);//叫下一个号
	}
	
	public static void main(String[] args) {
	
	    ReenLock reenLock = new ReenLock();
	
	    Thread t1 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            int lock = reenLock.lock();
	
	            try {
	                System.out.println(lock);
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                reenLock.unlock(lock);
	
	            }
	        }
	    });
	
	    Thread t2 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            int lock = reenLock.lock();
	
	            try {
	                System.out.println(lock);
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                reenLock.unlock(lock);
	
	            }
	        }
	    });
	
	    t1.start();
	    t2.start();
	
	}

}
	```
#### MCS自旋锁
	MCS其实是发明人的名字John Mellor-Crummey和Michael Scott
	原理是基于链表，对每一个获取锁的线程都创建一个node, 获取锁的时候，查看当前链表的是否存在前驱节点，如果不存在，就表示已经获取到锁。如果存在，就表示需要等待，这个时候将前置节点的next指向自己，然后轮询自己的变量。  释放锁的时候，判断是否有下一个节点，如果不存在下一个节点，就将当前队列置为Null.如果存在下一个节点，将下一个节点的状态变量置为false(表示不需要等待了)。
	**demo**
	```
		public class MCSLock {
	
	public static class MCSNode {
	    volatile MCSNode next;
	    volatile boolean isWaiting = true;
	}
	
	volatile MCSNode queue;
	
	private static final AtomicReferenceFieldUpdater<MCSLock, MCSNode> update =
	        AtomicReferenceFieldUpdater.newUpdater(MCSLock.class, MCSNode.class, "queue");
	
	public void lock(MCSNode curThread) {
	    MCSNode predecessor = update.getAndSet(this, curThread); //setp 1
	    if (predecessor != null) {
	        predecessor.next = curThread; //step 2
	        while (curThread.isWaiting) { // step 3
	            System.out.println(Thread.currentThread().getName() + "waiting...");
	        }
	    } else {  // 只有一个线程在使用锁，没有前驱通知他，就直接标记自己已经获取到锁了
	        curThread.isWaiting = false;
	    }
	}
	
	public void unlock(MCSNode curThread) {
	    if (curThread.isWaiting) {  // 拥有者释放，才有意义
	        return;
	    }
	    if (curThread.next == null) {  // 检查是否有人排在后面
	        //cas返回true 标识真的是没有排在后面
	        if (update.compareAndSet(this, curThread, null)) {
	            return;
	        } else {
	            //就表示突然有人在后面。需要等待后面的节点
	            // 场景是 step运行了，但是step2还没有执行完
	            while (curThread.next == null) {
	
	            }
	        }
	    }
	    curThread.next.isWaiting = false;
	    curThread.next = null;
	}
	
	public static void main(String[] args) {
	    MCSLock lock = new MCSLock();
	
	    Thread t1 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            final MCSNode mcsNode = new MCSNode();
	            lock.lock(mcsNode);
	
	            try {
	                System.out.println("111");
	
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                lock.unlock(mcsNode);
	
	            }
	        }
	    });
	
	    Thread t2 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            final MCSNode mcsNode = new MCSNode();
	            lock.lock(mcsNode);
	
	            try {
	                System.out.println("2222");
	
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                lock.unlock(mcsNode);
	
	            }
	        }
	    });
	
	    t1.start();
	    t2.start();
	
	}
	}
	```
#### CLH自旋锁
	入股知道AQS，肯定知道CLH自旋锁。CLH自旋锁，原理跟MCS自旋锁差不多。CLH是检测前驱节点的运行状态，MCS是检测自身节点状态信息.获取时：如果有前驱节点，那么就等待前驱节点释放。如果没有就表示获取到了锁。释放：直接释放当前节点状态。
	相比较之下，CLH理解起来比较简单，实现起来也比较简单.
	**demo**
	```
	public class CLHLock {
	
	private final static AtomicInteger in = new AtomicInteger();
	
	public static class CLHNode {
	    private volatile boolean isWaiting = true;
	    private final int a = in.getAndIncrement();
	}
	
	private volatile CLHNode tail;
	private static final AtomicReferenceFieldUpdater<CLHLock, CLHNode> update =
	        AtomicReferenceFieldUpdater.newUpdater(CLHLock.class, CLHNode.class, "tail");
	
	public void lock(CLHNode curNode) {
	    CLHNode preNode = update.getAndSet(this, curNode);
	    if (preNode != null) {
	
	        while (preNode.isWaiting) {
	
	        }
	    }
	}
	
	public void unlock(CLHNode curNode) {
	
	    if (!update.compareAndSet(this, curNode, null)) {
	        curNode.isWaiting = false;
	    }
	}
	
	public static void main(String[] args) {
	    CLHLock lock = new CLHLock();
	
	    Thread t1 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            final CLHLock.CLHNode mcsNode = new CLHLock.CLHNode();
	            lock.lock(mcsNode);
	
	            try {
	                System.out.println("111");
	
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                System.out.println("release");
	                lock.unlock(mcsNode);
	
	            }
	        }
	    });
	
	    Thread t2 = new Thread(new Runnable() {
	        @Override
	        public void run() {
	            final CLHLock.CLHNode mcsNode = new CLHLock.CLHNode();
	            lock.lock(mcsNode);
	
	            try {
	                System.out.println("2222");
	
	                Thread.sleep(10000);
	            } catch (InterruptedException e) {
	
	            } finally {
	                lock.unlock(mcsNode);
	
	            }
	        }
	    });
	
	    t1.start();
	    t2.start();
	}

}
	```
#### 参考
[线程安全实现与CLH队列](https://www.jianshu.com/p/0f6d3530d46b)

