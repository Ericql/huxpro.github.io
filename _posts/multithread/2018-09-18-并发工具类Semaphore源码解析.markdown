# 并发工具类Semaphore信号量源码解析
## 简介
&emsp;&emsp;Semaphore是信号量,在Java并发编程中,信号量控制的是线程并发的数量.它允许n个任务同时访问某个资源,主要是通过信号量大小控制并发数量
## 源码解析
### 内部类
内部类图示
&emsp;&emsp;由上图可知:Semaphore有三个内部类,三个内部类的关系如下:  
![内部类关系图](/img/in-post/JUC/Semaphore/内部类示意图.jpg)
> 内部存在Sync、NonfairSync、FairSync等三个内部类,与ReentrantLock类似,分别实现了公平与非公平方法

#### Sync源码解析
```
/**
 * 信号量的同步实现  
 * 使用AQS的状态来表示许可
 * 子类分别实现了公平和非公平的两个版本
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
	private static final long serialVersionUID = 1192457210091910933L;
	// 构造函数设置了AQS的状态值
	Sync(int permits) {
		setState(permits);
	}
	// 获取AQS的状态值
	final int getPermits() {
		return getState();
	}
	// 共享模式下以非公平方式获取
	final int nonfairTryAcquireShared(int acquires) {
		for (;;) {
			int available = getState();
			// 设置获取后的剩余通行量
			int remaining = available - acquires;
			// remaining<0时,放到队列中阻塞;remaining>0时,则可以通行
			if (remaining < 0 ||
				compareAndSetState(available, remaining))
				return remaining;
		}
	}
	// 共享模式下进行释放
	protected final boolean tryReleaseShared(int releases) {
		for (;;) {
			int current = getState();
            // 释放通行量,使得可用的通行量增加,别的线程就可以使用
			int next = current + releases;
			if (next < current) // 超出最大允许释放数
				throw new Error("Maximum permit count exceeded");
			// 设置释放后的剩余通行量,返回释放成功,表示需要通知下一个线程通行	
			if (compareAndSetState(current, next)) // 剩下的通行数
				return true;
		}
	}
	// 根据指定的减少量减小可用通行数
	final void reducePermits(int reductions) {
		for (;;) { // 自旋操作
			int current = getState(); // 获取通行数
			int next = current - reductions; // 剩余的通行数
			if (next > current) // underflow
				throw new Error("Permit count underflow");
			// 设置缩减后的通行数	
			if (compareAndSetState(current, next))
				return;
		}
	}

	final int drainPermits() {
		for (;;) {
			int current = getState();
			// 可用通行数设置为0
			if (current == 0 || compareAndSetState(current, 0))
				return current;
		}
	}
}
```
#### NonfairSync源码解析
```
static final class NonfairSync extends Sync {
	private static final long serialVersionUID = -2694183684443567898L;

	NonfairSync(int permits) {
		super(permits);
	}
	// 共享模式下获取
	protected int tryAcquireShared(int acquires) {
		return nonfairTryAcquireShared(acquires);
	}
}
```
&emsp;&emsp;由上可知:NonfairSync继承了Sync类,非公平方式获取资源,tryAcquireShared还是调用了父类的nonfairTryAcquireShared去实现非公平策略
#### FairSync类
```
static final class FairSync extends Sync {
	private static final long serialVersionUID = 2014338818796000944L;

	FairSync(int permits) {
		super(permits);
	}

	protected int tryAcquireShared(int acquires) {
		for (;;) {
			// 如果当前线程不位于队列头部,则阻塞
			if (hasQueuedPredecessors())
				return -1;
			int available = getState();
			int remaining = available - acquires;
			// 设置获取后的剩余通行量,当剩余量小0时,则阻塞,否则通行
			if (remaining < 0 ||
				compareAndSetState(available, remaining))
				return remaining;
		}
	}
}
```
&emsp;&emsp;由上可知:FairSync继承了Sync类,公平方式获取资源,tryAcquireShared此处没有在父类中重写,直接子类实现了公平策略
### 属性
```
/** 所有操作都是通过AbstractQueuedSynchronizer子类去实现 */
private final Sync sync;
```
### 构造方法
```
/**
 * 通过指定的permits参数设置通行量和默认使用非公平方式
 * permits可能为负值,在这种情况下,必须在任何操作前进行释放
 */
public Semaphore(int permits) {
	sync = new NonfairSync(permits);
}

/**
 * 通过指定的permits参数设置通行量和参数来决定公平还是非公平锁
 */
public Semaphore(int permits, boolean fair) {
	sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```
### 核心方法
#### acquire
```
/**
 * 可中断的方式从信号量获取一次通行
 * 如果能够获取到通行量,则立即返回,并相应减少一次通行
 * 如果没有可用的通行,则当前线程因线程调度而被禁用,并且在发生以下两种情况之一时处于休眠:
 * 1.其他线程调用信号量的release方法,并且将当前线程指定为下一个被许可的
 * 2.其他线程中断当前线程
 */
public void acquire() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}
```
&emsp;&emsp;假设使用默认的非公平策略,该方法将调用Sync的nonfairTryAcquireShared以非公平方式获取,底层还是调用AQS的doReleaseShared执行的,具体会在示例中按流程分析
#### release
```
/**
 * 释放一次通行,将其返回给信号量,会增加通行的数量.
 */
public void release() {
	sync.releaseShared(1);
}
```
## 示例分析
&emsp;&emsp;使用信号量控制资源的访问,每次最多10个线程访问,具体代码如下:
```
public class SemaphoreDemo {
    public void method(Semaphore semaphore) {
        try {
            System.out.println(Thread.currentThread().getName() + " tryAcquire ..");
            semaphore.acquire(5);
            System.out.println(Thread.currentThread().getName() + " acquire success ..");
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release(5);
            System.out.println(Thread.currentThread().getName() + " release..");
        }
    }

    public static void main(String[] args) {
        SemaphoreDemo demo = new SemaphoreDemo();
        Semaphore semaphore = new Semaphore(12);

        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    demo.method(semaphore);
                    try {
                        Thread.sleep(100);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }).start();
        }
    }
}
```
&emsp;&emsp;由上述代码可知:初始化了12个资源的线程,当线程Thread0进行获取的时候它是可以成功获取的,之后的thread1也是可以成功获取,但是thread2已经没有足够的通行数量,会获取失败;它会阻塞等待直到前面0和1进行一次释放,去增加通行数量,满足thread2的需求后就可以获取成功  
按照流程逐步分析  
1.Thread0的acquire流程如下:  
![t0acquire流程图](/img/in-post/JUC/Semaphore/thread0的acquire.jpg)  
&emsp;&emsp;初始的state值为12,当执行acquire后state成为7,此时还是remaining是大于0的,可以通行的
2.Thread1的acquire流程如下:  
![t1acquire流程图](/img/in-post/JUC/Semaphore/thread1的acquire.jpg)  
&emsp;&emsp;初始的state值为7,当执行acquire后state成为2,此时还是remaining是大于0的,可以通行的
3.Thread2的acquire流程如下:  
![t2acquirepark流程图](/img/in-post/JUC/Semaphore/thread2的acquirepark.jpg)  
&emsp;&emsp;初始的state值为2,此时remaining值是小于0的,不能通行,会进入队列进行等待,会直接park
4.thread0的release流程  
![t0release流程图](/img/in-post/JUC/Semaphore/thread0的release.jpg)  
&emsp;&emsp;当进行release时,初始的state值为2,会将thread的state返回增加到其上,release后的state值为7,此时执行doReleaseShared进行unpark,thread2会unpark后再次tryAcquireShared,这次会成功,state值又会从7到2  
5.thread2的release  
![t2release流程图](/img/in-post/JUC/Semaphore/thread2的release.jpg)  
6.thread1的release  
![t1release流程图](/img/in-post/JUC/Semaphore/thread1的release.jpg)  
此处还有unpark一次,但这次的unpark不会对程序有影响.