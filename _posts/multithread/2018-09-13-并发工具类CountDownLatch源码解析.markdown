---
layout:       post
title:        "CountDownLatch源码分析"
subtitle:     "CountDownLatch源码分析"
date:         2018-09-13 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
	- 多线程
---
# 并发工具类CountDownLatch源码解析
## 简介
&emsp;&emsp;CountDownLatch是一个同步工具类,用来协调多个线程之间的同步,比如我有很多数据需要进行分类计算,再进行汇总.为了更快的解析,则每一种类型计算都用一个线程去处理,等到所有类型计算完毕再进行汇总,这时我们就可以CountDownLatch
&emsp;&emsp;CountDownLatch能够使一个线程在等待另外一些线程完成各自工作后再继续执行.其内部使用计数器实现,其初始值为线程的数量.当每一个线程完成后计数器值就会减一,当计数器值为0时,表示所有的线程都已经完成任务了,则CountDownLatch上等待线程就可以继续运行了
## 使用场景
1.**开始执行前等待n个线程完成各自任务**：主线程在开始运行前等待n个线程执行完毕.将CountDownLatch的计数器初始化为n(new CountDownLatch(n)),每当一个任务线程执行完毕,就将计数器减1(cdl.countDown()),当计数器值变为0时,在CountDownLatch上await()的线程就会被唤醒.典型的就是启动应用程序之前,程序的必备组件/监听等都已经准备就绪了.
2.**实现多个线程开始执行任务的最大并行性**：初始化一个共享的CountDownLatch(1),将其计数器初始化为1.多个线程在开始运行前首先cdl.await(),当主线程调用countDown时,计数器变为0,多个线程就同时被唤醒.
3.**检测死锁**：使用n个线程访问共享资源,在每次测试阶段线程数量不同,查看什么情况会产生死锁
## 源码解析
### 内部类
```
/**
 * 为CountDownLatch服务的同步控制器
 * 使用AQS状态表示计数
 */
private static final class Sync extends AbstractQueuedSynchronizer {
	private static final long serialVersionUID = 4982264981922014374L;

	// 初始化count进state
	Sync(int count) {
		setState(count);
	}

	int getCount() {
		return getState();
	}

	// 尝试获取同步状态
	protected int tryAcquireShared(int acquires) {
		return (getState() == 0) ? 1 : -1;
	}

	// 自旋+CAS的方式释放同步状态
	protected boolean tryReleaseShared(int releases) {
		// 递减计数直到count值为0
		for (;;) {
			int c = getState();
			if (c == 0)
				return false;
			int nextc = c-1;
			if (compareAndSetState(c, nextc))
				return nextc == 0;
		}
	}
}
```
### 构造函数
```
/**
 * 用给定的count值去构造初始化
 * 内部是给到Sync的AQS的状态值上
 */
public CountDownLatch(int count) {
	if (count < 0) throw new IllegalArgumentException("count < 0");
	this.sync = new Sync(count);
}
```
> 注:计数值count实际上是闭锁需要等待的线程数量.这个值只能在这个地方设置,没有其他地方可以设置,而且CountDownLatch也没有提供任何方法去重新设置这个值

### await
```
/**
 * 使线程处于等待状态直到等待线程数量减到0
 * 除了线程是可中断的
 */
public void await() throws InterruptedException {
	sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg)
		throws InterruptedException {
	if (Thread.interrupted()) // 如果中断标志为true则响应中断
		throw new InterruptedException();
	// tryAcquireShared返回1则不执行,-1则线程加入等待队列中;(state一开始设置为线程数即!=0,则此处返回为-1)	
	if (tryAcquireShared(arg) < 0)
		doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}

/**
 * 在共享中断模式下获取资源
 */
private void doAcquireSharedInterruptibly(int arg)
	throws InterruptedException {
	// 添加节点至等待队列
	final Node node = addWaiter(Node.SHARED);
	boolean failed = true;
	try {
		for (;;) { // 自旋
			// 获取node的前驱节点
			final Node p = node.predecessor();
			if (p == head) {
				// 前驱是头节点,则当前节点尝试在共享模式下获取资源
				int r = tryAcquireShared(arg);
				if (r >= 0) {
					// 获取成功后则当前节点设置为头节点并进行传播
					setHeadAndPropagate(node, r);
					p.next = null; // help GC
					failed = false;
					return;
				}
			}
			// 获取失败后park线程并检查中断
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())
				throw new InterruptedException();
		}
	} finally {
		if (failed)
			cancelAcquire(node);
	}
}

/**
 * 设置队列的头节点,并检查后继节点是否可以在共享模式下等待
 * 如果满足下方条件就会被释放
 */
private void setHeadAndPropagate(Node node, int propagate) {
	Node h = head; // 记录下头节点,便于下方检查
	setHead(node); // 设置头节点
	/*
	 * 如果还有剩余量或h为null或h被取消/被中断则唤醒下一个线程
	 */
	if (propagate > 0 || h == null || h.waitStatus < 0 ||
		(h = head) == null || h.waitStatus < 0) {
		// 获取节点的后继
		Node s = node.next;
		if (s == null || s.isShared())
			// 以共享模式进行释放
			doReleaseShared();
	}
}
```
await流程图如下:

### countDown
```
public void countDown() {
	sync.releaseShared(1);// 共享释放,对应状态值减1
}

// 注释放资源与arg参数无关,tryReleaseShared内部进行自旋,每次减1
public final boolean releaseShared(int arg) {
	if (tryReleaseShared(arg)) {
		doReleaseShared();
		return true;
	}
	return false;
}

protected boolean tryReleaseShared(int releases) {
	// 递减计数,当递减至0时signal被触发
	for (;;) { // 自旋
		int c = getState(); // 获取状态
		if (c == 0) // 线程空闲
			return false;
		// state状态释放一次	
		int nextc = c-1;
		if (compareAndSetState(c, nextc))
			return nextc == 0;
	}
}

/**
 * 共享模式下的释放动作 -- 唤醒后继和确保传递
 * 说明: 在独占模式下,如果Signal则释放仅仅是调用unparkSuccessor
 */
private void doReleaseShared() {
	for (;;) { // 自旋
		Node h = head; // 获取头节点
		// 头节点不为null或不是尾节点
		if (h != null && h != tail) {
			// 获取头节点的waitStatus
			int ws = h.waitStatus;
			if (ws == Node.SIGNAL) {
				// 不成功就继续
				if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
					continue;            // loop to recheck cases
				// 释放后继节点	
				unparkSuccessor(h);
			}
			else if (ws == 0 &&
					 // 状态为0并且不成功则继续
					 !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
				continue;                // loop on failed CAS
		}
		if (h == head)                   // loop if head changed
			break;
	}
}
```
countDown流程图如下:
### getCount
```
/**
 * 返回当前的count值
 */
public long getCount() {
	return sync.getCount();
}

int getCount() {
	return getState();
}
```
## 使用示例分析
&emsp;&emsp;对文本中每一行使用一个线程来计算,等每一行都计算完毕后就进行汇总线程的执行
```
public class DownLatchDemo2 {
    private int[] nums;

    public DownLatchDemo2(int line) {
        this.nums = new int[line];
    }

    public void calc(String line,int index, CountDownLatch latch) {
        String[] nus = line.split(",");
        int total = 0;
        for (String num : nus) {
            total += Integer.parseInt(num);
        }
        nums[index] = total;
        System.out.println(Thread.currentThread().getName()+" 执行计算任务"+line+"结果为:"+total);
        latch.countDown();
    }

    public void sum() {
        System.out.println("汇总线程开始执行");
        int total = 0;
        for (int i = 0; i < nums.length; i++) {
            total += nums[i];
        }
        System.out.println("最终的结果为:"+total);
    }

    public static void main(String[] args) {
        List<String> contents = readFile();
        int lineCount = contents.size();

        CountDownLatch latch = new CountDownLatch(lineCount);

        DownLatchDemo2 demo = new DownLatchDemo2(lineCount);
        for (int i = 0; i < lineCount; i++) {
            final int j = i;
            new Thread(new Runnable() {
                @Override
                public void run() {
                    demo.calc(contents.get(j), j, latch);
                }
            }).start();
        }

        try {
            latch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        demo.sum();
    }

    private static List<String> readFile() {
        List<String> contents = new ArrayList<>();
        String line = null;
        BufferedReader reader = null;

        try {
            reader = new BufferedReader(new FileReader("C:\\work\\nums.txt"));
            while ((line = reader.readLine()) != null) {
                contents.add(line);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (reader != null) {
                try {
                    reader.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return contents;
    }
}
```
&emsp;&emsp;首先main线程会创建出nums.txt中对应行数的线程,运行后main会执行await操作,main线程会被阻塞,对应子线程会处理自己计算后执行countDown操作,当state值递减到0时main线程会被唤醒继续执行