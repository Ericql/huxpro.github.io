---
layout:       post
title:        "可重入读写锁ReentrantReadWriteLock源码解析"
subtitle:     "可重入读写锁ReentrantReadWriteLock源码解析"
date:         2018-09-19 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - 多线程
---
# 可重入读写锁ReentrantReadWriteLock源码解析
## 简介
&emsp;&emsp;在之前的synchronized、自定义的Lock、ReentrantLock都是一种排他锁,而读写锁既是排他锁也是共享锁,对于读锁是共享锁,对于写锁则是排他锁  
&emsp;&emsp;排他锁:就是多个线程在同一个时刻只允许一个线程访问;其他线程处于自旋状态,并不能访问锁内部代码块  
&emsp;&emsp;共享锁:在同一个时刻可以允许多个线程访问,共享锁与共享锁之间是可以同时访问的;  
读读不互斥、读写互斥(等所有读锁完成写锁才会进行)、写写互斥  
&emsp;&emsp;ReadWriteLock维护了一对相关的锁,一个用于只读操作,另一个用于写入操作.只要没有writer,读取锁可以由多个reader线程同时保持.写入锁是独占的;ReadWriteLock读取操作通常不会改变共享资源,但执行写入操作时,必须独占方式来获取锁.对于读取操作占多数的情况.ReadWriteLock就能够提供比独占锁更高的并发性.  
ReentrantReadWriteLock支持以下功能:  
1.支持公平和非公平获取锁方式  
2.支持可重入.读线程在获取了读锁后还可以获取读锁;写线程在获取了写锁之后既可以再次获取写锁又可以获取读锁  
3.还允许从写入锁降级为读取锁,但是从读取锁升级到写入锁是不允许的  
4.读取锁和写入锁都支持锁获取期间的中断  
5.Condition支持.仅写入锁提供了一个Condition实现;读取锁不支持Condiiton,readLock().newCondition()会抛出UnSupportedOperationException
## 父接口ReadWriteLock
ReadWriteLock也是一个接口,在它里面只定义了两个方法:
```
public interface ReadWriteLock {
    /**
     * 返回读锁
     */
    Lock readLock();
 
    /**
     * 返回写锁
     */
    Lock writeLock();
}
```
&emsp;&emsp;一个用来获取读锁,一个用来获取写锁.也就是说将文件的读写操作分开,分成2个锁来分配给线程,从而使得多个线程可以同时进行读操作.ReentrantReadWriteLock实现了ReadWriteLock接口
## 内部类
内部类总体结构示意图:  
![内部类示意图](/img/in-post/JUC/ReentrantReadWriteLock/内部类示意图.jpg)
### Sync
#### Sync的内部类
&emsp;&emsp;Sync类内部有两个内部类,分别是HoldCounter和ThreadLocalHoldCounter,HoldCounter主要是与读锁配套使用,其源码如下:
```
/**
 * 每个线程读取计数器的计数
 * 维持在ThreadLocal或缓存在cachedHoldCounter中
 */
static final class HoldCounter {
	int count = 0;
	// 使用id而不是引用,以避免垃圾保留
	final long tid = getThreadId(Thread.currentThread());
}
```
> 说明:HoldCounter内有两个属性:count表示读线程重入的次数,tid则表示当前线程唯一标志

ThreadLocalHoldCounter源码如下:
```
/**
 * ThreadLocal的子类. 本地线程计数器
 */
static final class ThreadLocalHoldCounter
	extends ThreadLocal<HoldCounter> {
	// 重写初始化方法
	public HoldCounter initialValue() {
		return new HoldCounter();
	}
}
```
> 说明:ThreadLocalHoldCounter重写了ThreadLocl的initialValue方法,初始化值是HoldCounter

#### Sync类属性
```
/*
* 读取和写入时计数所需常量
* 从逻辑上锁状态被分为两个无符号short:
* 低位表示写锁计数,高位表示读锁计数
*/
// 共享计数偏移量
static final int SHARED_SHIFT   = 16;
// 读锁偏移单位
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 读锁计数最大量
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 写锁计数最大量
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

/**
 * 当前线程持有的重入读锁的数目
 * 只能在构造函数和readObject方法中被初始化
 * 当计数降至0时被删除
 */
private transient ThreadLocalHoldCounter readHolds;

/**
 * 缓存的线程计数
 * 获取readLock的最后一个线程的保留计数
 * 通常情况下保存在ThreadLocal里
 */
private transient HoldCounter cachedHoldCounter;

/**
 * 获取读锁定的第一个线程
 * 更确切地说,firstReader是上次将共享计数从0更改为1的唯一线程
 */
private transient Thread firstReader = null;
// 第一个读线程的计数
private transient int firstReaderHoldCount;
```
#### 构造函数
```
/**
 * 设置本地线程计数器和State的状态
 */
Sync() {
	readHolds = new ThreadLocalHoldCounter();
	setState(getState()); // ensures visibility of readHolds
}
```
#### 主要方法
1.读写锁线程占有的数量
```
// 获取到读锁占有的线程数量
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

// 写锁占有的线程数量
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```
2.写锁的获取和释放  
2.1写锁获取
```
// 重写的tryAcquire方法
protected final boolean tryAcquire(int acquires) {
	/*
	 * Walkthrough:
	 * 1.如果读取计数非零或写入计数非零,并且所有者非零,则会失败
	 * 2.如果计数饱和也会失败(仅当count已为非零时才会发生)
	 * 3.否则,如果该线程是可重入的获取或队列策略允许的,则它有资格
	 *   获得锁定,如果是,则更新状态和设置所有者
	 */
	Thread current = Thread.currentThread();
	int c = getState();
	int w = exclusiveCount(c); // 获取到写锁状态值
	if (c != 0) {
		// (Note: if c != 0 and w == 0 then shared count != 0)
		// 写线程数量为0(写锁还没有被获取)或当前线程不是独占资源的线程
		if (w == 0 || current != getExclusiveOwnerThread())
			return false;
		// 判断是否超过最高写线程数量	
		if (w + exclusiveCount(acquires) > MAX_COUNT)
			throw new Error("Maximum lock count exceeded");
		// Reentrant acquire
		// 设置state值,支持重入
		setState(c + acquires);
		return true;
	}
	// 此时c==0,空闲读锁和写锁都没用被获取
	if (writerShouldBlock() ||
		!compareAndSetState(c, c + acquires))
		return false;
	setExclusiveOwnerThread(current);
	return true;
}
```
写锁获取流程图  
![写锁获取流程图](/img/in-post/JUC/ReentrantReadWriteLock/写锁获取.jpg)  
2.2写锁释放
```
/*
 * tryAcquire和tryRelease都可以被Conditions调用
 */
// 重写的tryReleases方法
protected final boolean tryRelease(int releases) {
	// 当前线程是否是独占资源的
	if (!isHeldExclusively())
		throw new IllegalMonitorStateException();
	// state值减去releases	
	int nextc = getState() - releases;
	// 判断写锁状态是否为0,如果为0就完全释放,否则就false
	boolean free = exclusiveCount(nextc) == 0;
	if (free)
		setExclusiveOwnerThread(null);
	setState(nextc);
	return free;
}
```
写锁释放流程图  
![写锁释放流程图](/img/in-post/JUC/ReentrantReadWriteLock/写锁释放.jpg)  
3.读锁的获取和释放  
3.1读锁的获取
```
// 共享模式下获取资源,参数是unused,因为读锁的重入技术是内部维护的
protected final int tryAcquireShared(int unused) {
	Thread current = Thread.currentThread();
	int c = getState();
	// 先获取低16位写锁,存在写锁并且当前线程不是获取写锁的线程,返回-1即获取失败
	if (exclusiveCount(c) != 0 &&
		getExclusiveOwnerThread() != current)
		return -1;
	// 再去获取高16位的读锁	
	int r = sharedCount(c);
	/**
	 * 先判断当前线程是否应该被阻塞
	 * MAX_COUNT为获取读锁的最大数量,16位的最大值,即判断读锁占有数量是否小于MAX_COUNT
	 * 在上述两个都为true的情况下,CAS设置c高位+1
	 */
	if (!readerShouldBlock() &&
		r < MAX_COUNT &&
		compareAndSetState(c, c + SHARED_UNIT)) {
		// r为0表示是firstReader,计数不会放到readHolds,这样在读锁只有一个的情况下就避免了查找readHolds
		if (r == 0) {
			firstReader = current;
			firstReaderHoldCount = 1;
		// firstReader重入	
		} else if (firstReader == current) {
			firstReaderHoldCount++;
		// 非firstReader读锁重入计数更新	
		} else {
			// 读锁重入计数缓存,基于ThreadLocal实现
			HoldCounter rh = cachedHoldCounter;
			// 计数缓存为null或其tid不是当前线程
			if (rh == null || rh.tid != getThreadId(current))
				// 获取当前线程对应的计数器
				cachedHoldCounter = rh = readHolds.get();
			else if (rh.count == 0)
				// 计数为0则设置
				readHolds.set(rh);
			rh.count++;
		}
		return 1;
	}
	/**
	 * 第一次获取读锁失败有两种情况:
	 * 1).没有写锁被占用时,尝试通过一次CAS去获取时,更新失败(说明有其他读锁在申请)
	 * 2).当前线程占有写锁,并且有其他写锁在当前线程的下一个节点等待获取写锁,除非
	 * 当前线程的下一个节点被取消,否则fullTryAcquireShared也获取不到读锁
	 */
	return fullTryAcquireShared(current);
}
```
读锁获取流程图  
![读锁获取流程图](/img/in-post/JUC/ReentrantReadWriteLock/读锁获取.jpg)  
3.2读取的释放
```
protected final boolean tryReleaseShared(int unused) {
    // 获取当前线程
    Thread current = Thread.currentThread();、
    // 当前线程为第一读线程
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        // 第一个读线程占用资源数为1,表示释放掉此线程即可
        if (firstReaderHoldCount == 1)
            firstReader = null;
        else // 否则得减少第一个读线程重入个数
            firstReaderHoldCount--;
    } else { // 当前线程不是第一个读线程
        // 获取缓存计数器
        HoldCounter rh = cachedHoldCounter;
        // 计数值为空或计数器的tid不为当前线程的tid
        if (rh == null || rh.tid != getThreadId(current))
            // 获取到当前线程的计数器
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 计数<=1直接移除
            readHolds.remove();
            // 此时计数<=0抛出异常
            if (count <= 0)
                throw unmatchedUnlockException();
        }
        // 减少计数
        --rh.count;
    }
    for (;;) { // 自旋CAS
        // 获取状态
        int c = getState();
        // 减去1<<16
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            // Releasing the read lock has no effect on readers,
            // but it may allow waiting writers to proceed if
            // both read and write locks are now free.
            return nextc == 0;
    }
}
```
读锁释放流程图  
![读锁获取流程图](/img/in-post/JUC/ReentrantReadWriteLock/读锁获取.jpg)
### FairSync
```
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```
> 说明:公平锁与非公平锁的区别就是在获取时会进行是否为队列前节点的判断,此处就是公平锁的所需要的是否还有前驱节点判断,有则false,否则返回true

### NonfairSync
```
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    // 直接返回false,可插队
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}

//当head节点不为null且head节点的下一个节点s不为null且s是独占模式(写线程)且s的线程不为null时,返回true.
//目的是不应该让写锁始终等待.作为一个启发式方法用于避免可能的写线程饥饿,这只是一种概率性的作用,因为如果有一个等待的写线程在其他尚未从队列中出队的读线程后面等待,那么新的读线程将不会被阻塞
 final boolean apparentlyFirstQueuedIsExclusive() {
     Node h, s;
     return (h = head) != null &&
         (s = h.next)  != null &&
         !s.isShared()         &&
         s.thread != null;
}
```
### ReadLock
```
/**
 * ReadLock内部类
 */
public static class ReadLock implements Lock, java.io.Serializable {
	private static final long serialVersionUID = -5992448646407690164L;
	private final Sync sync;

	/**
	 * 子类构造方法
	 */
	protected ReadLock(ReentrantReadWriteLock lock) {
		sync = lock.sync;
	}

	/**
	 * 尝试以共享方式获取读锁(具体AQS中的实现)
	 * 如果写锁没有被另外线程占有,则可以获取读锁
	 */
	public void lock() {
		sync.acquireShared(1);
	}

	/**
	 * 尝试以共享方式获取读锁,但是可以中断的
	 */
	public void lockInterruptibly() throws InterruptedException {
		sync.acquireSharedInterruptibly(1);
	}

	/**
	 * 尝试获取读锁(Sync类的获取读锁)
	 */
	public boolean tryLock() {
		return sync.tryReadLock();
	}

	/**
	 * 如果写锁在给定的时间内没有被另一个线程锁定,则读锁可以获取
	 * 如果锁以公平方式来获取,则其他线程在等待时当前获取的则被锁定
	 */
	public boolean tryLock(long timeout, TimeUnit unit)
			throws InterruptedException {
		return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
	}

	/**
	 * 尝试以共享方式去释放锁
	 */
	public void unlock() {
		sync.releaseShared(1);
	}

	/**
	 * 不支持创建条件等待队列
	 */
	public Condition newCondition() {
		throw new UnsupportedOperationException();
	}

	/**
	 * 返回锁的字符串描述和锁状态值
	 */
	public String toString() {
		int r = sync.getReadLockCount();
		return super.toString() +
			"[Read locks = " + r + "]";
	}
}
```
### WriteLock
```
/**
 * WriteLock内部类
 */
public static class WriteLock implements Lock, java.io.Serializable {
	private static final long serialVersionUID = -4992448646407690164L;
	private final Sync sync;

	/**
	 * 子类内部构造方法
	 */
	protected WriteLock(ReentrantReadWriteLock lock) {
		sync = lock.sync;
	}

	/**
	 * 尝试以独占方式获取写锁
	 * 因为读锁和写锁互斥,只有在读锁和写锁都未被锁定时,则获取成功并且
	 * state值为1
	 */
	public void lock() {
		sync.acquire(1);
	}

	/**
	 * 尝试以独占方式获取写锁,并且是可中断的
	 */
	public void lockInterruptibly() throws InterruptedException {
		sync.acquireInterruptibly(1);
	}

	/**
	 * 尝试获取写锁(Sync的获取写锁方法)
	 */
	public boolean tryLock( ) {
		return sync.tryWriteLock();
	}

	/**
	 * 在给定时间内当前线程没有被中断,如果写锁没有被另一个线程持有
	 * 则获取写锁
	 */
	public boolean tryLock(long timeout, TimeUnit unit)
			throws InterruptedException {
		return sync.tryAcquireNanos(1, unit.toNanos(timeout));
	}

	/**
	 * 尝试释放写锁
	 */
	public void unlock() {
		sync.release(1);
	}

	/**
	 * 返回实例的条件等待队列
	 */
	public Condition newCondition() {
		return sync.newCondition();
	}

	/**
	 * 返回写锁的字符串及其锁定状态值
	 */
	public String toString() {
		Thread o = sync.getOwner();
		return super.toString() + ((o == null) ?
								   "[Unlocked]" :
								   "[Locked by thread " + o.getName() + "]");
	}

	/**
	 * 查询当前线程是否持有写锁定
	 */
	public boolean isHeldByCurrentThread() {
		return sync.isHeldExclusively();
	}

	/**
	 * 通过当前查询写锁的持有数
	 * @return 当前线程对锁的持有数或 如果当前线程为保持锁定则返回零
	 */
	public int getHoldCount() {
		return sync.getWriteHoldCount();
	}
}
```
## 属性
```
private static final long serialVersionUID = -6992448646407690164L;
/** 内部类提供的读锁 */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类提供的写锁 */
private final ReentrantReadWriteLock.WriteLock writerLock;
/** 同步队列,下方所有同步机制都是由此执行 */
final Sync sync;

// Unsafe机制
private static final sun.misc.Unsafe UNSAFE;
// 线程ID的偏移地址
private static final long TID_OFFSET;
static {
	try {
		UNSAFE = sun.misc.Unsafe.getUnsafe();
		Class<?> tk = Thread.class;
		// 获取线程tid字段的内存地址
		TID_OFFSET = UNSAFE.objectFieldOffset
			(tk.getDeclaredField("tid"));
	} catch (Exception e) {
		throw new Error(e);
	}
}
```
## 构造方法
```
/**
 * 默认以非公平方式创建可重入的读写锁
 */
public ReentrantReadWriteLock() {
	this(false);
}

/**
 * 通过指定参数创建可重入的读写锁
 */
public ReentrantReadWriteLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
	readerLock = new ReadLock(this);
	writerLock = new WriteLock(this);
}
```
## 核心方法
&emsp;&emsp;ReentrantReadWriteLock的核心方法(只要是Public)的全部都转化为了Sync的操作,而Sync的方法已经在上面内部类全部介绍了
## 示例分析
```
public class ReadWrite {
    private Map<String, Object> map = new HashMap<>();
    private ReadWriteLock rwl = new ReentrantReadWriteLock();

    private Lock r = rwl.readLock();
    private Lock w = rwl.writeLock();

    public Object get(String key) {
        r.lock();
        System.out.println(Thread.currentThread().getName()+" 读操作正在执行...");
        try {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return map.get(key);
        } finally {
            r.unlock();
            System.out.println(Thread.currentThread().getName()+" 读操作执行完毕...");
        }
    }

    public void put(String key, Object value) {
        w.lock();
        try {
            System.out.println(Thread.currentThread().getName()+" 写操作正在执行...");
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            map.put(key, value);
        } finally {
            w.unlock();
            System.out.println(Thread.currentThread().getName()+" 写操作执行完毕...");
        }
    }
}

public class ReadWriteTest {
    public static void main(String[] args) {
        final ReadWrite readWrite = new ReadWrite();

        new Thread(new Runnable() {
            @Override
            public void run() {
                readWrite.put("key1", "value1");
            }
        }, "t0").start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                //readWrite.put("key2", "value2");
                System.out.println(readWrite.get("key1"));
            }
        }, "t1").start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                readWrite.put("key3", "value3");
            }
        }, "t2").start();

    }
}
```
> 说明:此处以非公平锁为例,新建了三个线程,分别写、读、写的方式对读写锁进行操作,可能出现的时序:t0去执行写操作,此时t1进来读,会失败进行等待,当t0执行完毕后,t1读线程执行,此时t2进来也会等待t1执行完毕

1.t0线程执行w.lock时,流程如下:  
![t0lock流程图](/img/in-post/JUC/ReentrantReadWriteLock/t0lock.jpg)  
2.t1线程执行r.lock时,由于w写线程在执行,会被阻塞,流程如下:  
![t1lock流程图](/img/in-post/JUC/ReentrantReadWriteLock/t1lockpark.jpg)  
3.t0线程执行w.unlock时,会在后面进行唤醒t1线程,流程如下:  
![t0unlock流程图](/img/in-post/JUC/ReentrantReadWriteLock/t0unlock.jpg)  
4.t2线程在执行w.lock时,因读写互斥,会被阻塞,流程如下:  
![t2lock流程图](/img/in-post/JUC/ReentrantReadWriteLock/t2lock.jpg)  
5.t1线程在执行r.unlock时,会唤醒上述的写线程,流程如下:  
![t1unlock流程图](/img/in-post/JUC/ReentrantReadWriteLock/t1unlockt2unpark.jpg)  
6.t2线程unlock,其流程类似于上述的写线程unlock,不再赘述