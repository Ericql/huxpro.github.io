---
layout:       post
title:        "Java线程池"
subtitle:     "ThreadPoolExecutor"
date:         2018-08-23 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - 多线程
    - ThreadPoolExecutor
---

# 线程池概述 #

 - 什么是线程池
    就是将多个线程放在一个池子里面(所谓池化技术),然后需要线程的时候不是创建一个线程,而是从线程池里面获取一个可用的线程,然后执行我们的任务.
 - 线程池的优势
    - 降低资源消耗,通过重复利用已创建的线程降低线程创建和消耗
    - 提供响应速度,当任务到达时,任务可以不需要等到线程创建就立即执行
    - 提高线程的可管理性,线程是稀缺资源,如果无限制的创建,不仅会消耗系统资源,还会降低系统的稳定性,使用线程池可以进行统一的分配、调优和监控.
# 创建一个线程池并提交线程任务 #
Java线程池最核心的类是ThreadPoolExecutor,查看ThreadPoolExecutor类关系继承图如下:
![ThreadPoolExecutor类关系图][1]


查看Executor接口可以通过execute方法进行提交任务
查看ExecutorService接口可以通过submit进行提交任务
所以ThreadPoolExecutor可以使用上述两种方式提交任务
## ThreadPoolExecutor源码解析 ##
### 类的结构 ###
![ThreadPoolExecutor类结构图][2]


ThreadPoolExecutor的核心内部类为Worker,其对资源进行了复用,减少了创建线程的开销,而其他的AbortPolicy等则是RejectedExecutionHandler接口的各种拒绝策略类
![ThreadPoolExecutor类结构图1][3]
  [1]: /img/bVbfCZs
  [2]: /img/bVbfEXu
  [3]: /img/bVbfE5E
当使用线程池并且使用有界队列的时候,如果队列满了,任务添加到线程池就会有问题,针对这个问题Java线程池提供了以下拒绝策略:

 1. AbortPolicy:使用该策略时,如果线程池队列满了,丢掉这个任务并且抛出RejectedExecutionException异常
 2. DiscardPolicy: 如果线程池队列满了,会直接丢掉这个任务并且不会有任何异常
 3. DiscardOldestPolicy: 如果线程池队列满了,会将最老的(即最早进入队列的)任务删除掉并腾出队列空间,再尝试将任务加入队列
 4. CallerRunsPolicy:如果任务添加到线程池失败,那么主线程会自己去执行该任务,不会去等待线程池的任务去执行
 5. 自定义:如果以上策略不符合业务场景,那么可以自己定义拒绝策略,只要实现RejectedExecutionHandler接口,并且实现rejectedExecution方法就可以了
由于核心内部类是worker,而且worker简易,先解析worker:

### Worker类源码解析 ###

#### 类继承关系 ####
```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable
```
可知:Worker类继承了AQS抽象类,实现了Runnable接口,重写了AQS的一些方法,对应的Runnable接口可以创建线程的动作
#### 类属性 ####

```
private final class Worker
	extends AbstractQueuedSynchronizer
	implements Runnable
{
	/**
	 * This class will never be serialized, but we provide a
	 * serialVersionUID to suppress a javac warning.
	 */
	// 版本号 
	private static final long serialVersionUID = 6138294804551838833L;

	/** Thread this worker is running in.  Null if factory fails. */
	// worker 所对应的线程
	final Thread thread;
	/** Initial task to run.  Possibly null. */
	// worker初始化任务,默认第一个任务
	Runnable firstTask;
	/** Per-thread task counter */
	// 每个线程任务计数器,记录已完成任务数量
	volatile long completedTasks;
```
说明:
    1.Thread类型的thread属性用来封装worker,对应形成一个线程
    2.Runnable类型的firstTask其表示该worker包含的runnable对象,即用户自定义的Runnable
    3.volatile修饰的long类型的completedTasks表示已完成的任务数量
#### 类构造函数 ####

```
Worker(Runnable firstTask) {
	// AQS的状态设置为-1,进行抑制中断直到 runWorker
	setState(-1); // inhibit interrupts until runWorker
	// 初始化第一个任务
	this.firstTask = firstTask;
	// 根据当前worker,初始化线程
	this.thread = getThreadFactory().newThread(this);
}
```
进行构造worker对象,初始化对应的属性
#### worker核心函数分析 ####

```
/** Delegates main run loop to outer runWorker  */
// 重写Runnable的run方法,并将run方法交给外部的runWorker
public void run() {
	runWorker(this);
}

// Lock methods
//
// The value 0 represents the unlocked state.
// The value 1 represents the locked state.
// 是否被独占,0表示未被独占,1表示被独占
protected boolean isHeldExclusively() {
	return getState() != 0;
}

// 尝试获取方法
protected boolean tryAcquire(int unused) {
	// CAS方法设置State状态值
	if (compareAndSetState(0, 1)) {
		// 设置独占线程
		setExclusiveOwnerThread(Thread.currentThread());
		return true;
	}
	return false;
}

// 尝试释放
protected boolean tryRelease(int unused) {
	// 设置独占线程为null
	setExclusiveOwnerThread(null);
	// 设置状态为0
	setState(0);
	return true;
}

// 获取锁
public void lock()        { acquire(1); }
// 尝试获取锁
public boolean tryLock()  { return tryAcquire(1); }
// 是否锁
public void unlock()      { release(1); }
// 是否被独占
public boolean isLocked() { return isHeldExclusively(); }

// 中断线程操作
void interruptIfStarted() {
	Thread t;
	// 当AQS状态>=0并且worker对象的线程不为null并且该线程没有被中断
	if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
		try {
			// 中断线程
			t.interrupt();
		} catch (SecurityException ignore) {
		}
	}
}
```
### ThreadPoolExecutor类的属性 ###

```
public class ThreadPoolExecutor extends AbstractExecutorService {
	// 线程池的控制状态(用来表示线程池的运行状态--高3位和运行的worker数量--低29位) 
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
	// 29位的偏移量
    private static final int COUNT_BITS = Integer.SIZE - 3;
	// 最大容量 2^29-1
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
	// 线程运行状态,总共5种状态,高3位表示
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // 对ctl进行装箱和拆箱动作
	// 拆分运行状态
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
	// 拆分线程数量
    private static int workerCountOf(int c)  { return c & CAPACITY; }
	// 运行状态和线程数量组合
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * Bit field accessors that don't require unpacking ctl.
     * These depend on the bit layout and on workerCount being never negative.
     */
	// 判断当前的运行状态是否在s这个标准状态之下
    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }
	// 判断当前的运行状态是否在s这个标准状态之上
    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }
	// 判断是否为运行状态
    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

    /**
     * Attempts to CAS-increment the workerCount field of ctl.
	 * 尝试以CAS方式增加ctl里的workerCount字段
     */
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    /**
     * Attempts to CAS-decrement the workerCount field of ctl.
	 * 尝试以CAS方式递减ctl里的workerCount字段
     */
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    /**
     * 递减ctl的workcount字段,仅仅在线程突然终止时才调用(具体见processWorkerExit)
     * 其他递减在getTask内执行
     */
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

    /**
     * 阻塞队列:用于保存任务和移交任务给工作线程
     * 不要求workQueue执行poll()方法返回null去判断workQueue的isEmpty()
     */
    private final BlockingQueue<Runnable> workQueue;

    /**
     * 可重入锁:持有锁才可以访问workers集合和相关的记录
     * 虽然可以使用并行集,但是通常最好使用锁;原因是序列化
     * interruptIdleWorkers需避免不需要的interrupt storms,特别是shutdown期间
     * 否则退出线程将同时中断那些尚未中断的.
     */
    private final ReentrantLock mainLock = new ReentrantLock();

    /**
	 * 存放工作线程集合
     * Set集合包含线程池中所有线程,当持有mainLock就可以被访问
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
	 * 终止条件
     */
    private final Condition termination = mainLock.newCondition();

    /**
	 * 最大线程池容量(仅在mainLock下可以访问)
     */
    private int largestPoolSize;

    /**
     * 已完成任务数量.(仅在工作线程终止时更新,并且持有mainLock)
     */
    private long completedTaskCount;

    /*
     * 下方的所有用户控制参数都被声明为volatile,以致于操作于最新的值
     * 但是不需要锁定,因为没有内部变量依赖它们在其他操作上同步修改
     */

    /**
	 * 线程工厂:所有线程都是通过工厂创建(通过addworker)
     * 所有调用必须准备好addworker失败情况(如限制线程数量的策略时候),
     */
    private volatile ThreadFactory threadFactory;

    /**
     * 在失败时(执行饱和或关机)调用的处理程序
     */
    private volatile RejectedExecutionHandler handler;

    /**
     * 线程没有任务执行时最多保持多久时间会终止
     * 线程在存在corePoolSize或allowCoreThreadTimeOut时使用此超时
     */
    private volatile long keepAliveTime;

    /**
     * 是否运行核心线程超时机制
     */
    private volatile boolean allowCoreThreadTimeOut;

    /**
     * 线程池大小
     */
    private volatile int corePoolSize;

    /**
     * 最大线程池大小(受限于容量)
     */
    private volatile int maximumPoolSize;

    /**
     * 默认拒绝执行策略
     */
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

    /**
     * shutdown和shutdownNow调用时所需的权限
     */
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");

    /* 执行finalizer时要使用的上下文 */
    private final AccessControlContext acc;
```
着重讲解下线程池的运行状态：
1.RUNNING:接受新任务并且处理已经进入阻塞队列的任务
2.SHUTDOWN:不接受新任务,但是处理已经进入阻塞队列的任务
3.STOP:不接受新任务,不处理已经进入阻塞队列的任务并且中断正在运行任务
4.TIDYING:所有任务都已经终止,workerCount为0,线程转化为TIDYING状态并且调用terminated钩子函数
5.terminated钩子函数已经运行完成
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;
runState单调增加，不一定要命中每个状态：
    RUNNING -> SHUTDOWN：调用SHUTDOWN()时,可能隐式在最后调用finalize()
    (RUNNING or SHUTDOWN) -> STOP：调用shutdownNow()
    SHUTDOWN -> TIDYING：当队列和线程池都为空时
    STOP -> TIDYING：当线程池为空时
    TIDYING -> TERMINATED：当terminated()钩子方法已经完成
### ThreadPoolExecutor类的构造函数 ###
ThreadPoolExecutor类总共有四个构造函数,但是前面三个都是特例最终调的都是最后一个,咱先解析每个构造函数再统一分析好它每一个参数的意思
1.ThreadPoolExecutor(int, int, long, TimeUnit, BlockingQueue<Runnable>)

```
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue) {
	this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
}
```
说明:该构造函数默认的线程工厂及拒绝执行策略去创建ThreadPoolExecutor
2.ThreadPoolExecutor(int, int, long, TimeUnit, BlockingQueue<Runnable>, ThreadFactory)

```
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue,
						  ThreadFactory threadFactory) {
	this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, defaultHandler);
}
```
说明:该构造函数只给出默认的拒绝执行策略
3.ThreadPoolExecutor(int, int, long, TimeUnit, BlockingQueue<Runnable>, RejectedExecutionHandler)

```
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue,
						  RejectedExecutionHandler handler) {
	this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), handler);
}
```
说明:该构造函数只给出默认的线程工厂
4.ThreadPoolExecutor(int, int, long, TimeUnit, BlockingQueue<Runnable>, ThreadFactory, RejectedExecutionHandler)

```
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue,
						  ThreadFactory threadFactory,
						  RejectedExecutionHandler handler) {
	// 线程池大小不能小于0 || 最大容量不能小于0 || 最大容量不能小于线程池大小 || keepAliveTime不能小于0					  
	if (corePoolSize < 0 || maximumPoolSize <= 0 || maximumPoolSize < corePoolSize || keepAliveTime < 0)
		throw new IllegalArgumentException();
	if (workQueue == null || threadFactory == null || handler == null)
		throw new NullPointerException();
	// 初始化相应的属性数据	
	this.acc = System.getSecurityManager() == null ? null : AccessController.getContext();
	this.corePoolSize = corePoolSize;
	this.maximumPoolSize = maximumPoolSize;
	this.workQueue = workQueue;
	this.keepAliveTime = unit.toNanos(keepAliveTime);
	this.threadFactory = threadFactory;
	this.handler = handler;
}
```

 - corePoolSize:线程池大小,在创建线程池后,默认情况下线程池中并没有任何线程,而是等到有任务到来后才创建线程去执行任务,除非调用了prestartAllCoreThreads()或者prestartCoreThread()方法,就会预创建线程,即在没有任务到来之前就创建corePoolSize个线程或一个线程.默认情况下,在创建了线程池后,线程池中的线程数为0,当有任务来之后,就会创建一个线程去执行任务,当线程池中的线程数目达到corePoolSize后,就会把到达的任务放到缓存队列当中
 - maximumPoolSize:线程池最大线程数,表示线程池中最多创建多少个线程
 - keepAliveTime:表示线程没有任务执行时最多保持多久时间会终止.默认情况下只有当线程池中的线程数大于corePoolSize时,KeepAliveTime才会起作用,直到线程池中的线程数不大于corePoolSize,即当线程池中的线程数大于CorePoolSize时,如果一个线程空闲的时间达到keepAliveTime则会终止,直到线程池中的线程数不超过corePoolSize.但是如果调用了allowCoreThreadTimeOut(boolean)方法,在线程池中的线程数不大于corePoolSize时,keepAliveTime参数也会起作用,直到线程池中的线程数为0
 - unit: 参数keepAliveTime的时间单位,有7种取值,默认为纳秒
        TimeUnit.DAYS;                //天
        TimeUnit.HOURS;              //小时
        TimeUnit.MINUTES;           //分钟
        TimeUnit.SECONDS;           //秒
        TimeUnit.MILLISECONDS;      //毫秒
        TimeUnit.MICROSECONDS;      //微妙
        TimeUnit.NANOSECONDS;       //纳秒
 - workQueue: 一个阻塞队列,用来存储等待执行的任务,一般有以下几种选择:ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQueue
 - threadFactory:线程工厂,主要用来创建线程
 - handler:拒绝执行策略
### ThreadPoolExecutor类的核心函数分析 ###
#### 任务提交过程 ####
1.execute方法
```
public void execute(Runnable command) {
	if (command == null)
		throw new NullPointerException();
	/*
	 * 进行下面三步:
	 *
	 * 1. 如果运行的线程小于corePoolSize,则尝试使用用户定义的Runnable对象创建一个新的线程
	 * 	调用addWorker函数会原子性的检查runState和workCount,通过返回false来防止在不应该添加
	 * 	线程时添加了线程
	 *
	 * 2. 如果一个任务能够成功入队列,在添加一个线程时仍需进行双重检查(因为前一次检查后该线程
	 * 	可能死亡了或进入到此方法时线程池已经shutdown了,所以需要再次检查状态);如有必要当停止时
	 * 	还需要回滚入队列操作,或当线程池没有线程时需要创建一个新线程
	 *
	 * 3. 如果无法入队列,那么需要增加一个新线程,如果此操作失败,那么就意味着线程池已经shutdown
	 * 	或者已经饱和了,所以拒绝任务
	 */
	// 获取线程池控制状态 
	int c = ctl.get();
	// worker数量小于corePoolSize
	if (workerCountOf(c) < corePoolSize) {
		// 添加worker成功则返回,不成功则再次获取线程池控制状态
		if (addWorker(command, true))
			return;
		c = ctl.get();
	}
	// 线程池处于RUNNING状态,将用户自定义的Runnable对象添加进Queue队列
	if (isRunning(c) && workQueue.offer(command)) {
		// 再次检查获取线程池控制状态
		int recheck = ctl.get();
		// 若此时线程池不处于RUNNING状态,将自定义任务从workQueue队列中移除
		if (! isRunning(recheck) && remove(command))
			reject(command); // 拒绝执行命令
		// worker数量等于0,添加worker
		else if (workerCountOf(recheck) == 0)
			addWorker(null, false);
	}
	// 添加worker失败则拒绝执行命令
	else if (!addWorker(command, false))
		reject(command);
}
```
说明:当客户端调用submit时,之后会间接调用execute函数,其在将来某个时间执行给定任务,execute并不会直接运行给定任务,它主要调用addWorker方法
2.addWorker方法
addWorker主要是完成以下任务：
 - 原子性增加workerCount
 - 将用户给定的任务封装成一个worker,并将此worker添加进workers集合
 - 启动worker对应的线程,并启动该线程运行worker的run方法
 - 回滚worker的创建动作,即将worker从workers集合中删除并原子性的减少workerCount

```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {// 外层无限循环
        // 获取线程池控制状态
        int c = ctl.get();
        // 获取状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&// 状态大于等于SHUTDOWN,初始的ctl为RUNNING,小于SHUTDOWN
            ! (rs == SHUTDOWN &&// 状态为SHUTDOWN
               firstTask == null &&// 第一个任务为null
               ! workQueue.isEmpty()))// worker队列不为空
            // 返回
            return false;

        for (;;) {
            // worker数量
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||   // worker数量大于等于最大容量
                wc >= (core ? corePoolSize : maximumPoolSize))// worker数量大于等于核心线程池大小或者最大线程池大小
                return false;
            if (compareAndIncrementWorkerCount(c))// 比较并增加worker的数量
                // 跳出外层循环
                break retry;
            // 获取线程池控制状态
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)// 此次的状态与上次获取的状态不相同
                // 跳过剩余部分,继续循环
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // worker开始标志
    boolean workerStarted = false;
    // worker被添加标志
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 初始化worker
        w = new Worker(firstTask);
        // 获取worker对应的线程
        final Thread t = w.thread;
        if (t != null) {// 线程不为null
            // 线程池锁
            final ReentrantLock mainLock = this.mainLock;
            // 获取锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                // 线程池运行状态
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||    // 小于SHUTDOWN
                    (rs == SHUTDOWN && firstTask == null)) {// 等于SHUTDOWN并且firstTask为null
                    if (t.isAlive()) // precheck that t is startable;线程刚添加进来,还未启动就存活
                        // 抛出线程状态异常
                        throw new IllegalThreadStateException();
                    // worker添加到workers集合
                    workers.add(w);
                    // 获取集合大小
                    int s = workers.size();
                    if (s > largestPoolSize)// 队列大小大于largestPoolSize
                        // 重新设置largestPoolSize
                        largestPoolSize = s;
                    // 设置worker已被添加标志
                    workerAdded = true;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            if (workerAdded) {// worker被添加
                // 开始执行worker的run方法
                t.start();
                // 设置worker已开始标志
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)// worker没有开始
            // 添加worker失败
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
#### 任务执行过程 ####
1、runworker方法
runWorker函数中会实际执行给定任务(即调用用户重写的run方法),并且当给定任务完成后,会继续从阻塞队列中取任务,直到阻塞队列为空(即任务全部完成).在执行给定任务时会调用钩子函数利用钩子函数可以完成用户自定义的一些逻辑,在runWorker中会调用getTask函数和processWorkerExit钩子函数
```
final void runWorker(Worker w) {
    // 获取当前线程
    Thread wt = Thread.currentThread();
    // 获取w的firstTask
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 释放锁(设置state为0,允许中断)
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            // 获取锁
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||// 线程池运行状态至少应该高于STOP
                 (Thread.interrupted() &&// 线程被中断
                  runStateAtLeast(ctl.get(), STOP))) &&// 再次检查,线程池的运行状态至少应该高于STOP
                !wt.isInterrupted())// wt线程(当前线程)没有被中断
                wt.interrupt();// 中断wt线程(当前线程)
            try {
                // 在执行之前调用钩子函数
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 运行给定的任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 执行完后调用钩子函数
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 增加给worker完成的任务数量
                w.completedTasks++;
                // 释放锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        // 处理完成后,调用钩子函数
        processWorkerExit(w, completedAbruptly);
    }
}
```
2.getTask方法
getTask函数用于从workerQueue阻塞队列中获取Runnable对象,由于是阻塞队列,所以支持有限时间等待poll和无限时间等待take.在该函数中还会相应shutdown和shutDownNow函数的操作,若检测到线程池处于SHUTDOWN或STOP状态,则会返回null,而不再返回阻塞队列中的Runnable对象

```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {// 无限循环,确保操作成功
        // 获取线程池控制状态
        int c = ctl.get();
        // 运行状态
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {// 大于等于SHUTDOWN(表示调用了shutDown)并且-->大于等于STOP(调用shutDownNow或者worker阻塞队列为空)
            // 减少worker数量
            decrementWorkerCount();
            // 返回null,不执行任务
            return null;
        }
        // 获取worker数量
        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;// 是否允许coreThread超时或workerCount大于核心大小

        if ((wc > maximumPoolSize || (timed && timedOut))// worker数量大于maxinumPoolSize
            && (wc > 1 || workQueue.isEmpty())) {// workerCount大于1或worker阻塞队列为空(在阻塞队列不为空时,需要保证至少有一个wc)
            if (compareAndDecrementWorkerCount(c))// 比较并减少workerCount
                // 返回null,不执行任务,该worker会退出
                return null;
            // 跳过剩余部分,继续循环
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :// 等待指定时间
                workQueue.take();// 一直等待,直到有元素
            if (r != null)
                return r;
            // 等待指定时间后没有获取元素则超时
            timedOut = true;
        } catch (InterruptedException retry) {
            // 抛出了被中断异常,重试没有超时
            timedOut = false;
        }
    }
}
```
3.processWorkerExit方法
processWorkerExit函数是在worker退出时调用到的钩子函数,而引起worker退出的主要因素如下:
1.阻塞队列已经为空,即没有任务可以运行了
2.调用了shutDown或shutDownNow函数
此函数会根据是否中断了空闲线程来确定是否减少workerCount的值,并且将worker从workers集合中移除并且会尝试终止线程池

```
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    if (completedAbruptly) // 如果被中断,则需要减少workCount
        decrementWorkerCount();
    // 获取可重入锁
    final ReentrantLock mainLock = this.mainLock;
    // 获取锁
    mainLock.lock();
    try {
        // 将worker完成的任务添加到总的完成任务中
        completedTaskCount += w.completedTasks;
        // 从workers集合中移除该worker
        workers.remove(w);
    } finally {
        // 释放锁
        mainLock.unlock();
    }
    // 尝试终止
    tryTerminate();
    // 获取线程池控制状态
    int c = ctl.get();
    if (runStateLessThan(c, STOP)) {// 小于STOP的运行状态
        if (!completedAbruptly) {
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            if (min == 0 && ! workQueue.isEmpty())// 允许核心超时并且workQueue阻塞队列不为空
                min = 1;
            if (workerCountOf(c) >= min)// workerCount大于等于min
                return; // replacement not needed
        }
        // 添加worker
        addWorker(null, false);
    }
}
```
#### 任务关闭过程 ####
1.shutdown方法
shutdown会按过去执行已提交任务的顺序发起一个有序的关闭,但是不接受新任务.首先检查是否具有shutdown的权限,然后设置线程池的控制为SHUTDOWN,之后中断空闲的worker,最后尝试终止线程池.

```
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 检查shutdown权限
        checkShutdownAccess();
        // 设置线程控制状态为SHUTDOWN
        advanceRunState(SHUTDOWN);
        // 中断空闲worker
        interruptIdleWorkers();
        // 调用shutdown钩子函数
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    // 尝试终止
    tryTerminate();
}
```
2.tryTerminate方法

```
final void tryTerminate() {
    for (;;) {// 无限循环,确保操作成功
        // 获取线程池控制状态
        int c = ctl.get();
        if (isRunning(c) ||// 线程池的运行状态为RUNNING
            runStateAtLeast(c, TIDYING) ||// 线程池的运行状态最大要大于TIDYING
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))// 线程池运行状态为SHUTDOWN并且workQueue队列不为null
            // 不能终止,直接返回
            return;
        if (workerCountOf(c) != 0) { // 线程池正在运行的worker数量不为0
            // 仅仅中断一个空闲的worker
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        // 获取线程池的锁
        final ReentrantLock mainLock = this.mainLock;
        // 获取锁
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {// 比较并设置线程池控制状态为TIDYING
                try {
                    // 终止,钩子函数
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            // 释放锁
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```
3.interruptIdleWorkers方法

```

private void interruptIdleWorkers(boolean onlyOne) {
    // 线程池的锁
    final ReentrantLock mainLock = this.mainLock;
    // 获取锁
    mainLock.lock();
    try {
        for (Worker w : workers) {// 遍历workers队列
            // worker对应的线程
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {// 线程未被中断并且成功获得锁
                try {
                    // 中断线程
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    // 释放锁
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```
