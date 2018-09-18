---
layout:       post
title:        "synchronized原理"
subtitle:     "synchronized原理"
date:         2018-08-27 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - Java
    - 多线程
---
# synchronized原理 #
## 理论上保证线程安全原理
* 内置锁: java中每个对象都可以作为内置(同步)锁
* 锁的互斥(互斥锁)  
修饰普通方法:内置锁就是当前类的实例  
修饰静态方法:内置锁是当前Class字节码对象Sequence.class  
修饰代码块:指定的锁对象  
```  
private static int value;

/**
 * synchronized放在普通方法上,内置锁就是当前类的实例
 * @return
 */
public synchronized int getNext() {
    return value ++;
}

/**
 * 修饰静态方法:内置锁是当前Class字节码对象Sequence.class
 * @return
 */
public static synchronized int getPrevious() {
    return value --;
}

public int xx() {
    synchronized (Sequence.class) {
        if (value > 0) {
            return 1;
        } else {
            return -1;
        }
    }
}
```
## JVM上保证线程安全原理
基于进入与退出Mintor对象来实现的,JVM的指令分别是mintorenter和mintorexit  
任何对象都可以作为锁,那么锁信息又存在对象的什么地方?  
&emsp;&emsp;存在对象头中  
对象头中的信息:  
* Mark Word:存的是对象的hash值,锁信息等  
&emsp;&emsp;虚拟机团队为了提高空间利用率,定义了几个锁的标志位,根据不同的锁标志位存储不同的状态信息,即MarkWord空间利用率是非常高的
* Class Metadata Address
* Array Length
JDK1.6之前就synchronized这种重量级的锁,在JDK1.6之后引入了偏向锁、轻量级锁、重量级锁
## 偏向锁
&emsp;&emsp;很多情况下,竞争锁不是由多个线程竞争,而是由一个线程在使用.相当于单线程在访问,每次像多线程那样获取资源会浪费资源,于是就有了偏向锁  
示例如下:  
&emsp;&emsp;当第一个线程过来了,就找对象头查一下看是不是偏向锁,偏向锁的对象头会记录(线程id、Epoch、对象的分代年龄信息、是否是偏向锁、锁标志位等);再查看线程id,第一次需要获取锁对象后不释放,后面再有线程过来会查看id,id一致就直接进,没有锁的获取和释放过程;锁毕竟不是单个线程在运行,肯定是多个线程的情况下运行,偏向锁获取了之后就一直没有撤销  
&emsp;&emsp;偏向锁使用了一种等到竞争出现才释放锁的机制,所以当其他锁尝试竞争偏向锁的时候有偏向锁的线程才回去释放偏向锁  
&emsp;&emsp;偏向锁的使用场景:只有一个线程访问同步代码块  
## 轻量级锁
&emsp;&emsp;轻量级锁是如何加锁的?在线程执行同步代码块前,java虚拟机会先在当前线程的栈帧中创建用于锁记录的空间,并将对象头中的MarkWord复制到锁记录空间,然后开始竞争锁,当竞争成功后,MarkWord就会把锁标志位改成轻量级锁,接着执行同步体  
&emsp;&emsp;比如多个线程一块访问,一个获取了轻量级锁,另外一个需要等待执行,它也要复制MarkWord到虚拟机栈中,接着她要修改MarkWord,发现被别的线程获得了锁,它修改不成功,就不停的去修改,不停的失败,直到前面的线程将锁给释放了,于是它可以获得了,上述的过程就是自旋锁  
&emsp;&emsp;当第一个线程执行完毕后,第二个线程获取到之后就把锁升级成重量级锁,线程就会阻塞,当第一个线程执行完毕释放锁并唤醒第二个线程,接着第二个线程开始继续执行,这就是轻量级锁的执行过程  
```
public class Demo3 {
    private Object obj1 = new Object();
    private Object obj2 = new Object();

    public void a() {
        synchronized (obj1) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (obj2) {
                System.out.println("a");
            }
        }
    }

    public void b() {
        synchronized (obj2) {
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            synchronized (obj1) {
                System.out.println("b");
            }
        }
    }

    public static void main(String[] args) {
        Demo3 demo3 = new Demo3();

        new Thread(new Runnable() {
            @Override
            public void run() {
                demo3.a();
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                demo3.b();
            }
        }).start();
    }
}
```
