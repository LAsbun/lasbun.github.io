---
title: '聊聊java并发编程[JUC中的Executor]'
tags:
  - java
  - executor
  - ThreadPoolExecutor
date: 2020-05-15 13:08:11
---

本文主要深入了解下 线程池，并剖析下JUC中的ThreadPoolExecutor 源码

<!-- more -->

#### demo
- 创建一个线程池, 然后执行size个任务
```
public class ExecutorTest {


    final ExecutorService executorService = new ThreadPoolExecutor(4, 50, 3, TimeUnit.SECONDS, new LinkedBlockingQueue<>());


    @Test
    public void testEx() throws InterruptedException {
        int size = 10;

        for (int i = 0; i < size; i++) {
            executorService.submit(new Job());
        }

        executorService.awaitTermination(30, TimeUnit.HOURS);

    }

    class Job implements Runnable {

        @Override
        public void run() {
            System.out.println(Thread.currentThread().getName() + " start");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + " end");
        }

    }
}
```
#### Executors 整体结构
- 分为三部分
	- 任务(Thread)
	- 执行任务(ExecuteService)
	- 任务结果(Future) 
{% asset_img image-20200514213103252.png %}

- 从上面的图我们可以观察到，核心还是ThreadPoolExecutor 以及其子类 ScheduledThreadPoolExecutor. 
- 我们主要从ThreadPoolExecutor 执行流程剖析下

#### ThreadPoolExecutor核心源码剖析
- 从下面几个点读源码
	- ThreadPoolExecutor 关键性属性
	- ThreadPoolExecutor 构造方法
	- ThreadPoolExecutor 的submit 方法
	- ThreadPoolExecutor 的 execute方法

##### ThreadPoolExecutor 关键性属性
```
		// 核心状态控制字段。 线程安全。 高三位表示线程状态，后29位表示工作的线程数。
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

		// 保存等待执行的阻塞队列.
    private final BlockingQueue<Runnable> workQueue;

		// 锁  保证操作安全
    private final ReentrantLock mainLock = new ReentrantLock();

		// 存储线程池中的Worker。只有持有mainLock 才可以操作
    private final HashSet<Worker> workers = new HashSet<Worker>();

		// 线程工厂，线程池创建一个线程时，调用线程工厂创建
    private volatile ThreadFactory threadFactory;

		// 线程池的拒绝策略
    private volatile RejectedExecutionHandler handler;

		// 核心线程池
    private volatile int corePoolSize;
		
		// 最大线程池数目
    private volatile int maximumPoolSize;

		// 默认线程池拒绝策略
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```
##### ctl 原理解析
- `ctl` int 32位。高三位表示线程状态，后29位表示工作的线程数。
- 涉及状态计算如下
	- RUNNING: ob11100000_00000000_00000000_00000000  // 只有这个最高位为1，所以只需要判断是否小于0 即可判断是否在运行
		- 线程池在运行中，可接受也可执行队列中的任务
	-  SHUTDOWN: ob00000000_00000000_00000000_00000000
		- 关闭状态。不再接受新任务，但是可以继续处理阻塞队列中的任务。线程池在RUNNING时调用shutdown() 就会进入此状态
	- STOP: 0b00100000_00000000_00000000_00000000
		- 停止状态，不接受任务，也不执行队列中的任务。调用shutdownNow 会进入此状态
	- TIDYING: 0b01000000_00000000_00000000_00000000
		- 如果所有任务都已经终止了，workCount(有效线程数)为0，线程池会调用terminated() 进入TERMINATED 状态  
	- TERMINATED: 0b01100000_00000000_00000000_00000000
		- 在terminated() 方法执行完后进入该状态，默认terminated()方法中什么也没有做
- 下面是状态流转图
	{% asset_img image-20200515142519394.png %}

##### ThreadPoolExecutor 构造方法
- 主要的都是调用下面的构造方法
```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```
- 核心参数解释
	-  **corePoolSize** 核心线程数。默认情况下创建线程池后，线程池并没有线程。直到后面有任务才会创建线程。可以通过prestartCoreThread()或prestartAllCoreThread()进行初始化。
	-  **maximumPoolSize** 线程池中最大的线程数。
	-  **keepAliveTime** 存活时间。默认情况下，线程池中线程数大于`corePoolSize`,线程空闲达到`keepAliveTime`就会被终止掉，直到线程池中的线程数不超过`corePoolSize`。如果设置了`allowCoreThreaTimeOut`,线程池中空闲时间达到`keepAliveTime`就会被终止掉，直到线程池中的线程数为0.
	- unit  `keepAliveTime` 的时间单位
	- workQueue: 保存等待执行的任务阻塞队列。当线程池中的线程数大于corePoolSizde时，会把待执行的任务封装成`Work`对象放进队列。不同的BlockingQueue有不同的线程池排队策略。
		- LinkedBlockingQueue 基于链表的FIFO的队列。默认队列大小为Integer.MAX_VALUE.所以是无界队列。当活跃线程等于	`corePoolSize`时，新建的任务都会被放进队列中等待。因为maximumPoolSize就是无效的。所以无界队列适用于某一时间端的高并发(就是核心线程不变，所有任务排队执行)
		- ArrayBlockingQueue： 有界队列。适用于防止资源耗尽
		- SynchronousQueue： 无界缓存值为1的等待队列。其特性就是每次添加一个元素后，必须等待其他线程取走后才需要添加。如果添加后没有线程在等待，并且线程池中的线程数不大于maximumPoolSize,就会创建一个新的线程。否则则会根据饱和策略拒绝掉这个任务。

	- threadFactory 线程工厂。线程池每需要创建一个线程时，都会调用线程工厂来完成。默认的线程工厂`DefaultThreadFactory` 会创建一个新的，非守护的仙鹤草呢个，不包含任何的配置信息。通常情况下都是要自己定义线程工厂方法,便于排查问题：
		- 新建线程提供名字，后续排查方便
		- 为线程指定UncaughtExceptionHandler来处理线程执行过程中未被捕获的异常。
		- 修改线程优先级等

	- handler: RejectedExecutionHandler 线程池的拒绝策略.如果阻塞队列满了且线程数达到了maximum，此时继续提交任务就会触发拒绝策略。JUC默认提供了四种不同的策略：
		- CallerRunPolicy 由调用方线程执行(阻塞主线程)
		- AbortPolicy 直接丢出RejectedExecutionException. 默认
		- DiscardPolicy 直接丢弃任务。
		- DiscardOldPolicy 利用FIFO 丢弃队列中最靠前的任务，并尝试再次执行。

##### 源码
- 具体逻辑见源码注释。
- 需要注意：
	- 一个Worker 就是线程池中的一个线程. Worker 集成了AQS, 其内部是独占锁的实现
	- 创建新Worker 可以使用new Worker(null) 
	- `worker.thread.start` 会调用`worker.run()` 实际是`runWorker()`方法
	- worker 轮询队列执行
    - 执行完之后删除worker，并根据实际的最小线程池数，创建新worker

###### ThreadPoolExecutor.execute 方法
- submit 本质上也是调用的这个方法
```
public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        int c = ctl.get();
        // 判断是否小于核心线程数。注意 如果是非运行中的其他状态，都是大于corePoolSize 的。具体可以看下ctl 的构成，以及设计原理。
        if (workerCountOf(c) < corePoolSize) {
        		// 如果添加，运行成功，返回
            if (addWorker(command, true))
                return;
            // 到这里肯定是没有添加成功
            c = ctl.get();
        }
        // 判断是否在运行，以及能否入队
        if (isRunning(c) && workQueue.offer(command)) {
        		// 这里double check 是因为中间线程池可能会 被关闭
            int recheck = ctl.get();
            // 如果没有运行 并且从队列中移除了job
            if (! isRunning(recheck) && remove(command))
            		// 调用拒绝策略
                reject(command);
            // 运行中的数量为0 
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        // 前置就表示队列已经满了 尝试直接添加进线程池中。 注意addWorker 就是一个CAS。所以在SynchronizedQueue 情况下，如果所有线程池中的线程都运行很久，就会导致主线程卡在第一个阻塞的代码块处。
        else if (!addWorker(command, false))
            reject(command);
    }
    
```
 ###### ThreadPoolExecutor.addWorker方法
 ```
 // 添加一个Worker (一个worker就是一个线程池中可执行的线程)
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:  
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // >= SHUTDOWN 表示不是在运行, 不接受任何新增任务
            if (rs >= SHUTDOWN &&
            		! (SHUTDOWN状态，并且(task 是null 并且队列非空))
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                // 超过最大线程池数了
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // CAS 添加WorkerCount. 成功 跳出双重死循环
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                // 如果运行的数量跟实际读取的不一致，重新计算下数量。调到第一层死循环
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

				// 到这里 肯定是标识新增workerCount 成功。 （还没有真正创建worker）
				
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
        		// new Worker 会在内存使用线程工厂创建一个Thread
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
										
										// rs < SHUTDOWN 标识在运行
                    if (rs < SHUTDOWN ||
                    		// 下面这个条件标识可以新建一个新的线程 
                        (rs == SHUTDOWN && firstTask == null)) {
                        // 如果新建的线程在运行，肯定是不对的
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        // 添加worker 成功
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                		// 运行线程
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
            		// 失败处理
                addWorkerFailed(w);
        }
        return workerStarted;
    }
 ```
###### ThreadPoolExecutor.runWorker 方法
- while 一直从队列中获取任务. 使用getTask() 方法获取
- 如果线程池正在停止，保证该线程是中断状态，否则就保证是非中断状态。
- beforeExecute && afterExecute 是扩展方法
- 队列执行完之后调用`processWorkerExit` 对该Worker 进行销毁等其他操作
 ```
 final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 如果线程池正在停止，保证该线程是中断状态。否则就保证是非中断状态。
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                		// before 扩展函数
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                    		// after 扩展函数
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
        		// 到这里就表示阻塞队列全空了，需要做其他的策略
            processWorkerExit(w, completedAbruptly);
        }
    }
 ```

 ###### ThreadPoolExecutor.getTask()
 - 主要目的是为了获取队列中的任务
 ```
 		private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // 队列空了就减少个worker
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
						
						// 如果运行的worker 数目大于最大值，或者是超时了
            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
            		// 不同的策略，如果允许获取超时，就是使用poll 超时获取，否则就是通过take 方法获取阻塞队列数据
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
               	// 到这里就表示超时了
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
 ```
 ###### ThreadPoolExecutor.processWorkerExit
 - 销毁当前线程
 - 如果当前线程池数目小于allowCoreThreadTimeOut ? 0(允许corePoolThread 超时) : corePoolSize。则新创建一个线程。
 ```
 private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
 ```
 ###### 总结
{% asset_img image-20200515113516329.png %}

#### 一个线上案例
- 线上现状
	- 线上疯狂FullGc 




- 原因
  - 原来定义的代码是下面的
    ```
      private ThreadPoolExecutor executorService = 
      new ThreadPoolExecutor(0, 10, 60, TimeUnit.SECONDS, new LinkedBlockingQueue<>());
  	```

	-  corePoolSize 设置为0 就会创建一个线程池。后续任务会放进到队列
	-  但是LinkedBlockingQueue  是一个无界的阻塞队列。就导致后续的队列都是直接放进队列，并没有创建新的线程，所以就把内存撑爆了。
- 解决
	- 将无界队列改为有界队列 new ArrayBlockingQueue
	- corePoolSize 设置大一点

#### 参考
《java并发编程的艺术》方腾飞　魏鹏　程晓明　著