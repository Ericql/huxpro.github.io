---
layout:       post
title:        "ReentrantLock源码分析"
subtitle:     "ReentrantLock源码分析"
date:         2018-09-10 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
	- 多线程
---
# ReentrantLock源码分析
## 介绍
&emsp;&emsp;我们在上一篇已经分析了AbstractQueuedSynchronier的源码,接下来就分析ReentrantLock的源码
## 内部类分析
&emsp;&emsp;ReentrantLock内部总共有三个内部类,并且FairSync和NonfairSync都是Sync的具体实现
### Sync
```
/**
 * lock锁的基础同步控制类;子类实现了公平和非公平两个版本
 * 使用AQS的状态值来表示锁的持有数
 */
abstract static class Sync extends AbstractQueuedSynchronizer {
	private static final long serialVersionUID = -5179523762034025860L;

	/**
	 * 获取锁方法
	 */
	abstract void lock();

	/**
	 * 非公平方式获取锁;tryAcquire由子类去实现
	 */
	final boolean nonfairTryAcquire(int acquires) {
		// 当前线程
		final Thread current = Thread.currentThread();
		// 获取状态
		int c = getState();
		if (c == 0) { // 当前锁处于空闲状态
			if (compareAndSetState(0, acquires)) { 	// 当设置成功后,
				setExclusiveOwnerThread(current);	// 设置当前线程拥有锁
				return true;
			}
		}
		// 本身就是当前线程拥有锁(重入)
		else if (current == getExclusiveOwnerThread()) {
			// 增加一次重入数
			int nextc = c + acquires;
			if (nextc < 0) // overflow
				throw new Error("Maximum lock count exceeded");
			setState(nextc); // 设置状态
			return true;
		}
		return false;
	}

	// 进行资源的释放,此处考虑重入问题
	protected final boolean tryRelease(int releases) {
		int c = getState() - releases;
		// 当前线程不是拥有锁的线程则抛出异常
		if (Thread.currentThread() != getExclusiveOwnerThread())
			throw new IllegalMonitorStateException();
		boolean free = false;
		// 如果重入的次数已经完全释放,状态已经空闲则free为true,锁拥有的线程为null
		if (c == 0) {
			free = true;
			setExclusiveOwnerThread(null);
		}
		// 设置状态值
		setState(c);
		return free;
	}

	protected final boolean isHeldExclusively() {
		// 我们不需要检查当前线程是否为所有者,因为我们通常在所有者之前读取状态
		return getExclusiveOwnerThread() == Thread.currentThread();
	}

	// 创建一个condition通信类(包含等待队列)
	final ConditionObject newCondition() {
		return new ConditionObject();
	}

	// 返回资源占用线程
	final Thread getOwner() {
		return getState() == 0 ? null : getExclusiveOwnerThread();
	}

	// 获取当前线程的重入值
	final int getHoldCount() {
		return isHeldExclusively() ? getState() : 0;
	}

	// 获取资源是否被锁
	final boolean isLocked() {
		return getState() != 0;
	}

	/**
	 * 反序列化操作
	 */
	private void readObject(java.io.ObjectInputStream s)
		throws java.io.IOException, ClassNotFoundException {
		s.defaultReadObject();
		setState(0); // reset to unlocked state
	}
}
```
&emsp;&emsp;Sync是抽象同步器,它继承了AQS,实现了部分非公平和公平的方法,后续的FairSync和NonfairSync都继承Sync并实现了lock和tryAcquire方法
### FairSync
```
/**
 * 公平锁的同步器
 */
static final class FairSync extends Sync {
	private static final long serialVersionUID = -3000897897090466540L;

	// 以独占方式获取对象,期间忽略中断
	final void lock() {
		acquire(1);
	}

	/**
	 * tryAcquire的公平方式.  
	 */
	protected final boolean tryAcquire(int acquires) {
		// 获取当前线程
		final Thread current = Thread.currentThread();
		// 获取独占锁的状态
		int c = getState();
		// 为0意味着锁没有被任何线程所拥有
		if (c == 0) {
			// 若锁没有被任何线程所拥有,则判断当前线程是否是CLH队列中的第一个线程,
			// 若是的话,则获取该锁,设置锁的状态,并设置锁的拥有者为当前线程
			if (!hasQueuedPredecessors() &&
				compareAndSetState(0, acquires)) {
				setExclusiveOwnerThread(current);
				return true;
			}
		}
		else if (current == getExclusiveOwnerThread()) {
			// 若独占锁拥有者已经为当前线程—重入,则更新锁的状态
			int nextc = c + acquires;
			if (nextc < 0)
				throw new Error("Maximum lock count exceeded");
			setState(nextc);
			return true;
		}
		return false;
	}
}
```
### NonFairSync
```
/**
 * 非公平锁的同步器
 */
static final class NonfairSync extends Sync {
	private static final long serialVersionUID = 7316153563782823691L;

	/**
	 * 执行锁定.正常以独占模式获取资源,Try immediate barge, backing up to normal
	 * acquire on failure.
	 */
	final void lock() {
		if (compareAndSetState(0, 1)) // 开始是状态0的设置可以成功,则将当前线程设置独占了锁
			setExclusiveOwnerThread(Thread.currentThread());
		else // 锁已经被占用,尝试获取锁
			acquire(1);
	}

	protected final boolean tryAcquire(int acquires) {
		return nonfairTryAcquire(acquires);
	}
}
```
&emsp;&emsp;由上可知:公平锁和非公平锁在获取资源时的唯一区别是:公平锁会使用hasQueuedPredecessors方法去判断当前线程节点是否为CLH队列的第一个,如果是第一个才会往下获取,否则false
## 构造函数
```
/**
 * 创建ReentrantLock实例,默认是非公平锁
 */
public ReentrantLock() {
	sync = new NonfairSync();
}

/**
 * 创建ReentrantLock实例,以fair去选择公平还是非公平
 */
public ReentrantLock(boolean fair) {
	sync = fair ? new FairSync() : new NonfairSync();
}
```
## ReentrantLock使用示例(示例来自于源码)
```
class X {
  ReentrantLock lock = new ReentrantLock();
  // ...
  public void m() {
    assert lock.getHoldCount() == 0;
    lock.lock();
    try {
      // ... method body
    } finally {
      lock.unlock();
    }
  }
}
```
上述为ReentrantLock的最基本用法,主要使用到的是ReentrantLock的lock方法和unlock方法,下面我们就主要解析这两个方法:
## lock
```
public void lock() {
    sync.lock();
}
```
即内部是调用了同步类sync的方法,再去看sync的lock方法:发现其是抽象方法,具体实现由FairSync公平同步器和NonfairSync非公平同步器
FairSync的lock:
```
final void lock() {
	acquire(1);
}
```
NonfairSync的lock:
```
final void lock() {
	if (compareAndSetState(0, 1))
		setExclusiveOwnerThread(Thread.currentThread());
	else
		acquire(1);
}
```
&emsp;&emsp;由上可知:公平锁与非公平锁在lock的时候的区别是:compareAndSetState可以将资源状态由0设置为1,即将资源占有线程设置为当前线程,而公平锁不会这样,它会acquire从CLH队列中获取第一个
&emsp;&emsp;acquire方法则是AQS类中的方法,已经在上一篇AQS中解析了,具体可以看上一篇
## unlock
```
public void unlock() {
	sync.release(1);
}
```
内部还是调用了同步类的release方法,在这里是直接使用了AQS的release方法
```
/**
 * 独占模式下进行资源释放  
 * true则释放一个或多个线程来实现
 */
public final boolean release(int arg) {
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```
tryRelease方法则是Sync来实现的
```
protected final boolean tryRelease(int releases) {
	// 状态值-1
    int c = getState() - releases;
	// 当前线程与锁持有的线程不一致则抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
	// c==0则所有的锁都unlock,将free置为true,独占锁置为null
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
## 示例分析
```
public class MyThread extends Thread {
    private Lock lock;

    public MyThread(String name, Lock lock) {
        super(name);
        this.lock = lock;
    }

    @Override
    public void run() {
        lock.lock();

        try {
            System.out.println(Thread.currentThread() + " running");
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }
}

public class ReentrantLockDemo {
    public static void main(String[] args) {
        Lock lock = new ReentrantLock(true);

        //while (true) {
            MyThread t1 = new MyThread("t1", lock);
            MyThread t2 = new MyThread("t2", lock);
            MyThread t3 = new MyThread("t3", lock);

            t1.start();
            t2.start();
            t3.start();
        //}
    }
}
```
