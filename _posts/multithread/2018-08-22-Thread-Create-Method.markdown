---
layout:       post  
title:        "Java多线程"  
subtitle:     "Thread Create Method"  
date:         2018-08-22 12:00:00  
author:       "Eric"  
header-img:   ""  
header-mask:  0.3  
catalog:      true  
multilingual: false  
tags:  
&#8194;&#8194;- 后端开发  
&#8194;&#8194;- Java  
&#8194;&#8194;- 多线程
  
---
# 实现多线程的方式 #
## 1.继承Thread类 ##
### 线程初始化 ###
Thread类总共有9个构造方法,统一看内部都是调用了init方法进行初始化的,例如无参构造:  

<pre name="code" class="java">public Thread() {    
	init(null, null, "Thread-" + nextThreadNum(), 0);  
}</pre>

线程默认名称由Thread-与nextThreadNum()从0开始++组成,也可由Thread构造方法初始指定  
构造初始化主要是init方法,具体如下:  
<pre name="code" class="java">private void init(ThreadGroup g, Runnable target, String name, long stackSize) {  
	init(g, target, name, stackSize, null, true);  
}
</pre>  
ThreadGroup:是一种树状结构,底下可以放线程组也可以放线程  
Runnable:可以指定的线程任务  
stackSize:线程所需栈大小,未指定默认为0  
### daemon ###
thread.setDaemon(true):设置守护线程
作用在程序中后台调用,做支持性工作,比如垃圾回收线程,扔在后台默默做些事
随着主线程的退出后默默在后台运行
### 线程中断与销毁 ###
thread1.stop(); --- 很好用,但是已经过期了,使用这种方式在停止的时候线程所获取的锁及资源都没释放掉,只是让线程在无限期的等待下去,所以这种方式是非常不好的,不推荐使用  
thread1.interrupt(); --- 推荐使用interrupt,但直接使用并没有让线程停止掉,这需要我们手动去处理,代码如下:  
<pre name="code" class="java">public class ExtendsThread extends Thread {
    public ExtendsThread(String name) {
        super(name);
    }
    @Override
    public void run() {
        while (!interrupted()) {
            System.out.println(getName()+"线程执行了");
            try {
                Thread.sleep(200);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public static void main(String[] args) {
        ExtendsThread thread1 = new ExtendsThread("first-thread");
        ExtendsThread thread2 = new ExtendsThread("second-thread");
        thread1.start();
        thread2.start();

        thread1.interrupt();// 会把中断标志改为是中断
    }
}  
</pre>




