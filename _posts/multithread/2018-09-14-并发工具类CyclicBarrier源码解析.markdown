---
layout:       post
title:        "并发工具类CyclicBarrier源码解析"
subtitle:     "并发工具类CyclicBarrier源码解析"
date:         2018-09-14 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - 多线程
---
# 并发工具类CyclicBarrier源码解析
## 简介
&emsp;&emsp;CyclicBarrier也是一个同步工具类,字面上意思为可循环的屏障,查看JDK的描述:它允许一组线程相互等待直到所有线程都到达一个公共的屏障点;而这个屏障是可循环的,即在所有的线程释放后这个屏障是可以重新使用的  
&emsp;&emsp;与CountDownLatch非常相似;不同点在于CountDownLatch维护了一个锁存器计数,当计数达到0后处于wait状态的线程可以执行,而对于CyclicBarrier相当于一个屏障(类似于开会,当所有人都赶到会议室后就开始开会,先来的什么也不干wait),允许一组线程相互等待直到达到某个公共屏障点
## 源码解析
### 内部类
```
/**
 * 每一次使用的CyclicBarrier都可以看做Generation的实例
 * 当屏障被触发或重置时,Generation类都会发生改变(主要broken值改变)
 * 可能存在很多线程通过使用barrier从而与Generation类关联
 * 如果发生中断并后续没有复位,则不会有一个处于active状态的Generation
 */
private static class Generation {
	boolean broken = false;
}
```
> 说明：此内部类只有一个broken属性,用来表示屏障是否被损坏状态

### 属性
```
/** 用于保护屏障的可重入锁 */
private final ReentrantLock lock = new ReentrantLock();
/** 设置条件去等待直到屏障失效 */
private final Condition trip = lock.newCondition();
/** 参与线程数量 */
private final int parties;
/* 屏障失效时执行的操作 */
private final Runnable barrierCommand;
/** generation类 */
private Generation generation = new Generation();

/**
 * 正在等待进入屏障的数量
 * 在每一个Generation进行count down从parties到0
 * 当屏障打破后又0恢复到parties
 */
private int count;
```
### 构造函数
```
/**
 * 创建一个新的CyclicBarrier
 * 当给定数量parties线程都在等待时屏障就会失效
 * 当屏障失效后,就会执行给定的动作,由进入屏障的最后一个线程执行
 */
public CyclicBarrier(int parties, Runnable barrierAction) {
	if (parties <= 0) throw new IllegalArgumentException();
	this.parties = parties;
	this.count = parties;
	this.barrierCommand = barrierAction;
}

/**
 * 创建一个新的CyclicBarrier
 * 此处只能指定线程数量,没有指定预先定义的执行动作
 */
public CyclicBarrier(int parties) {
	this(parties, null);
}
```
### breakBarrier
```
/**
 * 设置当前屏障被打破并且唤醒每一个线程
 * 只能被持有锁的线程所调用
 */
private void breakBarrier() {
	generation.broken = true;	// 设置线程状态
	count = parties;			// 恢复初始的等待进入屏障线程数量
	trip.signalAll();			// 唤醒所有线程
}
```
### dowait
```
/**
 * 主屏障代码,内部含有各种策略
 */
private int dowait(boolean timed, long nanos)
	throws InterruptedException, BrokenBarrierException,
		   TimeoutException {
	// 获取锁对象,final表示可用但不可更改(里面还用到Condition)	   
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		// 获取Generation对象
		final Generation g = generation;
		// 屏障已经破坏,则抛出异常
		if (g.broken)
			throw new BrokenBarrierException();
		// 线程被中断
		if (Thread.interrupted()) {
			// 损坏当前屏障,并且唤醒所有线程,只有拥有锁线程才可以调用
			breakBarrier();
			throw new InterruptedException();
		}
		// 等待进入屏障的线程数量递减(dowait使得当前线程进入屏障点)
		int index = --count;
		if (index == 0) {  // 等待进入屏障的线程数量为0,即都已经进入屏障
			// 全部进入后的动作执行标志
			boolean ranAction = false;
			try {
				// 具体需要执行的操作
				final Runnable command = barrierCommand;
				if (command != null)
					command.run(); 	// 执行的动作
				ranAction = true;	// 设置动作执行标志
				// 更新屏障状态并唤醒所有的线程
				nextGeneration();
				return 0;
			} finally {
				// 出现异常等各种情况,导致ranAction没有运行,则直接打破屏障
				if (!ranAction)
					breakBarrier();
			}
		}

		// loop until tripped, broken, interrupted, or timed out
		// 进行自旋操作
		for (;;) {
			try {
				// 没有设置等待时间
				if (!timed)
					// 等待
					trip.await();
				else if (nanos > 0L) // 设置了等待时间并且大于0
					// 则指定等待时长
					nanos = trip.awaitNanos(nanos);
			} catch (InterruptedException ie) {
				// 如果出现中断异常
				// generation没有改变并且屏障没有打破
				if (g == generation && ! g.broken) {
					// 则打破屏障并抛出异常
					breakBarrier();
					throw ie;
				} else {
					// generation已经改变或者屏障已经被打破,则中断线程
					Thread.currentThread().interrupt();
				}
			}
			// 屏障被损坏,抛出异常
			if (g.broken)
				throw new BrokenBarrierException();
			// 不等于generation,则抛出异常
			if (g != generation)
				// 返回线程数量
				return index;
			// 设置了等待时间,并且等待时间小于0
			if (timed && nanos <= 0L) {
				// 损坏屏障抛出异常
				breakBarrier();
				throw new TimeoutException();
			}
		}
	} finally {
		// 释放锁
		lock.unlock();
	}
}
```
### getNumberWaiting
```
/**
 * 返回在屏障上等待的线程数量
 * 此方法主要用于调试和断言
 */
public int getNumberWaiting() {
	final ReentrantLock lock = this.lock;
	lock.lock();
	try {
		return parties - count;
	} finally {
		lock.unlock();
	}
}
```
### nextGeneration
```
/**
 * 更新屏障打破后的状态并唤醒所有线程
 * 只能被持有资源的线程唤醒
 */
private void nextGeneration() {
	// 设置当前的generation完成
	trip.signalAll();
	// 重置count
	count = parties;
	generation = new Generation();
}
```
## 示例分析
&emsp;&emsp;以同事聚集开会为例,具体代码如下:
```
public class CyclicBarrierDemo {
    Random random = new Random();

    public void meeting(CyclicBarrier barrier) {
        try {
            Thread.sleep(random.nextInt(4000));
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName()+" 到达会议室,等待开会.");

        try {
            barrier.await();
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println(Thread.currentThread().getName() + "发言");
    }

    public static void main(String[] args) {
        CyclicBarrierDemo demo = new CyclicBarrierDemo();
        CyclicBarrier barrier = new CyclicBarrier(3, new Runnable() {
            @Override
            public void run() {
                System.out.println("我们开始开会...");
            }
        });

        for (int i = 0; i < 3; i++) {
            new Thread(new Runnable() {
                @Override
                public void run() {
                    demo.meeting(barrier);
                }
            }).start();
        }
    }
}
```
&emsp;&emsp;运行结果可知:当Thread0、1、2全部到达会议室后(即调用await)后就会执行CyclicBarrier的barrierAction;之后会继续执行方法,之后执行各线程自己的方法  
1.当Thread0执行await的流程图如下:  
![t0await流程图](/img/in-post/JUC/CyclicBarrier/Thread0的await.jpg)
> 说明:ReentrantLock默认使用非公平策略,在dowait调用了NonfairSync.lock;因为一开始的AQS状态为0,所以Thread0可以设置为占用线程,trip.await调用使得线程被添加到条件等待队列上,并被park释放掉资源

2.Thread1执行await的流程图如下:  
![t1await流程图](/img/in-post/JUC/CyclicBarrier/Thread1的await.jpg)
> Thread1也被添加到条件等待队列的尾部,并且被park;

3.Thread2执行await的流程图如下:  
![t2await流程图](/img/in-post/JUC/CyclicBarrier/Thread2的await.jpg)
> 当Thread2进来后人员到齐,此时count为0,就会执行barrier中的操作;同时也会把条件等待队列的节点全部signalAll到同步队列中;Thread0线程也会被unpark(),获取到CPU资源继续运行

Thread0获取到资源后,在AQS的conditionObject方法中就会跳出while循环,执行下放的acquireQueued方法,具体流程图如下:  
![unpark流程图](/img/in-post/JUC/CyclicBarrier/unpark后的操作.jpg)
