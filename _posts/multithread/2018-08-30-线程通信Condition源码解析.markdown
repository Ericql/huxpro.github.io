---
layout:       post
title:        "Condition源码分析"
subtitle:     "Condition源码分析"
date:         2018-08-30 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - 多线程
---
# 线程通信Condition源码分析
## Object类提供的线程通信--wait、notify
&emsp;&emsp;使用Object类的线程通信模拟生产消费者模型,具体代码如下:
```
/**
 * 生产消费者模型--商店
 */
public class Store {
    // 持有商品数量
    private int count;
    // 最大持有商品数量
    public final int MAX_COUNT = 10;

    public synchronized void producte() {
        while (count >= MAX_COUNT) {
            try {
                System.out.println(Thread.currentThread().getName() + "库存已满,生产者停止生产");
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
        count ++;
        System.out.println(Thread.currentThread().getName() + "正在生产中,当前库存为" + count);
        notifyAll(); // 唤醒消费者进行消费
    }

    public synchronized void consume() {
        while (count <= 0) {
            try {
                System.out.println(Thread.currentThread().getName() + "库存为0,消费者停止消费");
                wait();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        count --;
        System.out.println(Thread.currentThread().getName() + "消费者正在消费,当前库存为" + count);
        notifyAll();    // 唤醒生产者进行生产
    }

    public static void main(String[] args) {
        Store store = new Store();
        Producter producter = new Producter(store);
        Consumer consumer = new Consumer(store);

        // 四个线程去生产
        new Thread(producter).start();
        //new Thread(producter).start();
        //new Thread(producter).start();
        //new Thread(producter).start();

        // 一个线程消费
        new Thread(consumer).start();
    }
}

public class Producter implements Runnable {
    private Store store;

    public Producter(Store store) {
        this.store = store;
    }

    @Override
    public void run() {
        while (true) {
            store.producte();
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class Consumer implements Runnable {
    private Store store;

    public Consumer(Store store) {
        this.store = store;
    }

    @Override
    public void run() {
        while (true) {
            store.consume();
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
&emsp;&emsp;wait和notify必须放在同步代码块中执行,否则报错;wait和notify必须是持有锁对象的  
&emsp;&emsp;Object的监视器方法需要结合synchronized关键字一起使用可以实现等待/通知模式;如果使用了显示锁lock,上述的线程通信方式就不能用了,所以显示锁要提供自己的等待/通知模式,这就是Condition
## 显式锁提供的线程通信--Condition使用
&emsp;&emsp;使用Condition方式的线程通信模拟生产消费者模型,具体代码如下:
```
public class Store {
    // 持有商品数量
    private int count;
    // 最大持有商品数量
    public final int MAX_COUNT = 10;

    private Lock lock = new ReentrantLock();
    private Condition full = lock.newCondition();
    private Condition empty = lock.newCondition();

    public void producte() {
        lock.lock();

        try {
            while (count >= MAX_COUNT) {
                System.out.println(Thread.currentThread().getName() + "库存已满,生产者停止生产");
                full.await();
            }
            count ++;
            System.out.println(Thread.currentThread().getName() + "正在生产中,当前库存为" + count);
            empty.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void consume() {
        lock.lock();

        try {
            while (count <= 0) {
                System.out.println(Thread.currentThread().getName() + "库存为0,消费者停止消费");
                empty.await();
            }
            count --;
            System.out.println(Thread.currentThread().getName() + "消费者正在消费,当前库存为" + count);
            full.signal();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        Store store = new Store();
        Producter producter = new Producter(store);
        Consumer consumer = new Consumer(store);

        new Thread(consumer).start();
        new Thread(producter).start();
    }
}

public class Consumer implements Runnable {
    private Store store;

    public Consumer(Store store) {
        this.store = store;
    }

    @Override
    public void run() {
        while (true) {
            store.consume();
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}

public class Producter implements Runnable {
    private Store store;

    public Producter(Store store) {
        this.store = store;
    }

    @Override
    public void run() {
        while (true) {
            store.producte();
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
notify是随机唤醒一个线程,但是Condition则可以指定线程唤醒;
## Condition简介
&emsp;&emsp;Condition主要是为了在JUC框架中提供和Java线程通信的wait、notify、notifyAll方法类似的功能;即通过设置一个条件,在合适的时候通过调用await使一个线程沉睡并释放锁,当其他线程调用singal方法时会唤醒那个线程.condition通常视为多线程之间通信的工具  
&emsp;&emsp;Condition自己也维护了一个队列,该队列的作用是维护一个等待signal信号的队列,两个队列的作用是不同,每个线程也仅仅会同时存在以上两个队列中的一个  
示例过程:  
线程1:  
&emsp;&emsp;- 线程1调用reentrantLock.lock时,持有锁  
&emsp;&emsp;- 线程1调用await方法,进入"条件等待队列",同时释放锁  
&emsp;&emsp;- 线程1获取到线程2 Signal信号,从"条件等待队列"进入到"同步等待队列"    
线程2:  
&emsp;&emsp;- 线程2调用reentrantLock.lock时,由于锁被线程1持有,进入"同步等待队列"  
&emsp;&emsp;- 由于线程1释放锁,线程2从"同步等待队列"移除,获取到锁.线程2调用signal方法,导致线程1被唤醒  
&emsp;&emsp;- 线程2调用reentrantLock.unlock,线程1获取锁,继续循环   
条件等待队列是Condition内部自己维护的一个队列,具有以下特点  
* 要加入"条件等待队列"的节点,不能在"同步等待队列"
* 从"条件等待队列"移除的节点,会进入"同步等待队列"
* 一个锁对象只能有一个"同步等待队列",但可以有多个"条件等待队列"

| 对比项      | Object监视器 | Condition |
| --------- | --------- | --------- |
| 前置条件   | 获取对象的锁 |调用Lock.locl获取锁,调用Lock.newCondition获取Condition对象|
| 调用方式   | 直接调用,如:object.notify() | 直接调用,如condition.await() |
| 等待队列个数 | 一个 | 多个 |
| 当前线程释放锁进入等待状态 | 支持 | 支持 |
| 当前线程进入等待状态,在等待状态不断响应中断 | 不支持 | 支持
| 当前线程释放锁进入超时等待状态 | 支持 | 支持 |
| 当前线程释放锁并进入等待状态知道将来某个时间 | 不支持 | 支持 |
| 唤醒等待队列的一个线程 | 支持 | 支持 |
| 唤醒等待队列的所有线程 | 支持 | 支持 |

&emsp;&emsp;Condition是一个接口,其内部接口方法如下:
```
public interface Condition {
	// 当前线程进入等待状态,直到被通知(signal)或者被中断时,当前线程进入运行状态,从await()返回
	void await() throws InterruptedException;
	// 进入等待状态,并加超时响应
	// 自定义超时时间单位;在time之前被唤醒,返回true,超时返回false
	boolean await(long time, TimeUnit unit) throws InterruptedException;
	// 线程进入等待状态加超时响应,并返回剩余时间
	// 在nanosTimeout之前被唤醒,返回值 = nanosTimeout - 实际消耗的时间
	// 返回值 <= 0表示超时
	long awaitNanos(long nanosTimeout) throws InterruptedException;
	// 当前线程进入等待状态,直到被通知,对中断不做响应
	void awaitUninterruptibly();
	// 当前线程进入等待状态直到将来的指定时间被通知
	// 如果没有到指定时间被通知返回true,否则到达指定时间返回false
	boolean awaitUntil(Date deadline) throws InterruptedException;
	// 唤醒一个等待在Condition上的线程
	void signal();
	// 唤醒等待在Condition上所有的线程
    void signalAll();
}
```
## Condition源码分析
&emsp;&emsp;我们在上述使用condition时都是通过lock.newCondition()去创建的,咱先看看lock的这个方法,如下:
```
public Condition newCondition() {
    return sync.newCondition();
}

final ConditionObject newCondition() {
    return new ConditionObject();
}
```
&emsp;&emsp;由上可知:Condition是通过sync同步类去创建的,而sync内部是直接new了ConditionObject();所以分析Condition源码就是分析ConditionObject,定位发现ConditionObject是AQS的内部类  
&emsp;&emsp;我们在使用的时候主要是使用ConditionObject的await和signal方法  
&emsp;&emsp;ConditionObject的等待队列是一个FIFO队列,队列的每个节点都是等待在Condition对象上线程的引用;在调用await方法,线程释放锁,将其构造成Node节点放入条件等待队列.  
Condition队列的结构如下:  
![Condition结构图](/img/in-post/condition/Condition结构.jpg)
> AQS实质上拥有一个同步队列和多个等待队列，具体对应关系如下图所示:  
![AQS结构图](/img/in-post/condition/AQS同步队列与等待队列示意图.jpg)

### await方法解析
```
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程添加到等待队列中
    Node node = addConditionWaiter();
    // 该节点加入condition队列中等待,await则需要释放掉当前线程占有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断该节点是否在CLH队列中
    while (!isOnSyncQueue(node)) {
        // 不在则阻塞该节点
        LockSupport.park(this);
        // 阻塞过程中发生中断
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 出了while循环,代表线程被唤醒,并且已经将该node从condition队列transfer到了CLH队列中, acquireQueued在队列中尝试获取锁,会阻塞当前线程,并且在上面while循环等待的过程中没有发生异常,则修改interruptMode为REINTERRUPT
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //该节点调用transferAfterCancelledWait添加到CLH队列中的,此时该节点的nextWaiter不为null,需要调用unlinkCancelledWaiters将该节点从CONDITION队列中删除,该节点的状态为0
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    //如果interruptMode不为0,则代表该线程在上面过程中发生了中断或者抛出了异常,则调用reportInterruptAfterWait方法在此处抛出异常
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
await具体流程如下:
![await流程图](/img/in-post/condition/await流程.png)
#### 1.入队操作
&emsp;&emsp;Condition的入队操作表示将节点添加进"条件等待队列",通过AQS的ConditionObject的addConditionWaiter方法来完成
```
/**
 * 添加一个新的waiter到等待队列
 * @return its new wait node
 */
private Node addConditionWaiter() {
	Node t = lastWaiter;
	// 判断"条件等待队列"是否为空
	if (t != null && t.waitStatus != Node.CONDITION) {
		// 循环清空所有状态不为CONDITION的节点(主要是那些因超时/中断被取消的节点)
		unlinkCancelledWaiters();
		t = lastWaiter;
	}
	// 创建新条件节点并在末尾添加进条件等待队列
	Node node = new Node(Thread.currentThread(), Node.CONDITION);
	if (t == null)
		firstWaiter = node;
	else
		t.nextWaiter = node;
	lastWaiter = node;
	return node;
}

private void unlinkCancelledWaiters() {
	Node t = firstWaiter;
	Node trail = null;
	// 条件等待队列不为空,从头开始清除
	while (t != null) {
		Node next = t.nextWaiter;
		// 判断出当前节点不为CONDITION
		if (t.waitStatus != Node.CONDITION) {
			t.nextWaiter = null;
			// 前面没有节点时,trail不为空
			if (trail == null)
				firstWaiter = next;
			else
				trail.nextWaiter = next;
			if (next == null)
				lastWaiter = trail;
		}
		else
			// trail保存住当前条件节点
			trail = t;
		// 开始对下一节点进行判断处理	
		t = next;
	}
}
```
#### 2.fullyRelease释放锁操作
&emsp;&emsp;在加入队列之后,休眠之前,保存此时的同步状态,然后将持有的锁释放,保存同步状态用于被通知后恢复同步状态
```
final long fullyRelease(Node node) {
	boolean failed = true;
	try {
		// 获取同步状态的当前值
		long savedState = getState();
		// 最终还是调用tryRelease释放
		if (release(savedState)) {
			failed = false;
			return savedState;
		} else {
			throw new IllegalMonitorStateException();
		}
	} finally {
		if (failed)
			node.waitStatus = Node.CANCELLED;
	}
}

public final boolean release(long arg) {
	// 尝试释放资源
	if (tryRelease(arg)) {
		Node h = head;
		if (h != null && h.waitStatus != 0)
			// 唤醒等待队列的下一个线程
			unparkSuccessor(h);
		return true;
	}
	return false;
}
```
#### 3.节点是否在同步队列
&emsp;&emsp;isOnSyncQueue判断的是线程是否从等待队列移到同步队列.即等待队列在等待通知,通知线程将唤醒的节点移到AQS同步队列(具体参见signal方法解析);若不满足则进入休眠,线程进入自旋
```
/**
 * 判断节点是否在同步队列中
 */
final boolean isOnSyncQueue(Node node) {
	// 如果状态为Node.CONDITION,即还在Condiiton队列中,不在同步队列
	// 如果状态不为Node.CONDITION,前置节点为null,也是不在同步队列
	if (node.waitStatus == Node.CONDITION || node.prev == null)
		return false;
	// 如果状态不为Node.CONDITION,有前置节点也有后继节点,那么一定在同步队列
	if (node.next != null
		return true;
	/*
	 * 其前置节点为非null,但是不在同步队列也是可能的,因为CAS将其加入队列失败
	 * 所以需要从尾部遍历确保其在队列中
	 */
	return findNodeFromTail(node);
}

/**
 * 从尾部查找node节点
 */
private boolean findNodeFromTail(Node node) {
	Node t = tail;
	for (;;) {
		if (t == node)
			return true;
		if (t == null)
			return false;
		t = t.prev;
	}
}
```
#### 4.获取同步状态
&emsp;&emsp;使线程在等待队列中获取资源,一直获取到资源后才返回(自旋获取同步状态).如果在等待过程中被中断,则返回true,否则返回false
```
final boolean acquireQueued(final Node node, long arg) {
	// 标记是否成功拿到资源
	boolean failed = true;
	try {
		// 标记等待过程中是否被中断过
		boolean interrupted = false;
		// 进行"自旋"
		for (;;) {
			// 拿到前驱节点
			final Node p = node.predecessor();
			// 如果前驱节点是head,则当前节点有资格尝试去获取资源
			if (p == head && tryAcquire(arg)) {
				// 获取到资源后,将head指向当前节点
				setHead(node);
				// 便于GC,即在当前节点之前拿到资源的节点进行出队了
				p.next = null; 
				failed = false;
				// 拿到资源操作已经成功再看是否被中断过
				return interrupted;
			}
			// 获取失败后进行park,进入等待直到被unparking
			if (shouldParkAfterFailedAcquire(p, node) &&
				// park和检查中断,如果在等待过程中产生中断,就标记为true
				parkAndCheckInterrupt())
				interrupted = true;
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

/*
 * 状态检查,检测自己是否可以真正休息
 */
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	// 拿到前驱的状态
	int ws = pred.waitStatus;
	/*
	 * 如果前驱已经设置状态为 释放后通知自己,则可以安心park
	 */
	if (ws == Node.SIGNAL)
		return true;
	/*
	 * 如果前驱节点状态为已取消,那就一直往前找,直到找到一个正常等待的,并排在后边
	 */	
	if (ws > 0) {
		do {
			node.prev = pred = pred.prev;
		} while (pred.waitStatus > 0);
		pred.next = node;
	/*
	 * 前驱状态为0或PROPAGATE.就把前驱状态设置为SIGNAL
	 * 告诉前驱拿完后通知自己,有可能失败
	 */	
	} else {
		compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
	}
	return false;
}

/**
 * 线程休息进入等待状态
 */
private final boolean parkAndCheckInterrupt() {
	// 调用park()使线程进入waiting状态
	LockSupport.park(this);
	// 如果被唤醒查看自己是不是被中断的
	return Thread.interrupted();
}
```
### signal方法解析
#### 出队操作
