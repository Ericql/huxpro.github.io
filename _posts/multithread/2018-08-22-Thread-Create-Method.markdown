---
layout:       post
title:        "Java线程创建方式"
subtitle:     "Thread Create Method"
date:         2018-08-22 12:00:00
author:       "Eric"
header-img:   ""
header-mask:  0.3
catalog:      true
multilingual: false
tags:
    - 后端开发
    - Java
    - 多线程
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
}</pre>
## 2.实现Runnable接口 ##
run方法分析:
<pre name="code" class="java">public void run() {
    if (target != null) {
        target.run();
    }
}</pre>
target就是指定的Runnable接口的,所以就是我们实现的类
<pre name="code" class="java">/**
 * @author Eric
 * @create 2018 04 11 7:15
 * 作为线程任务存在
 */
public class Runn implements Runnable {
    @Override
    public void run() {
        while (true) {
            System.out.println("thread running ...");
        }
    }
    public static void main(String[] args) {
        Thread thread1 = new Thread(new Runn());
        thread1.start();
    }
}</pre>
初学阶段一般比较上述两种方式哪种比较好?
好与坏没有绝对,而是根据业务需求来决定;但是可以在业务场景中去评论对应方式的好坏,java中推崇面向接口编程,Runnable接口实现任务与接口分离,这样让代码更解耦
## 3.匿名内部类方式 ##
<pre name="code" class="java">--子类方式
new Thread() {
    @Override
    public void run() {
        System.out.println("thread start ...");
    }
}.start();</pre>
<pre name="code" class="java">--构造方法方式
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("thread start ...");
    }
}).start();</pre>
<pre name="code" class="java">--两种都有执行子类的,子类run方法覆盖父类的,直接执行run
new Thread(new Runnable() {
    @Override
    public void run() {
        System.out.println("runnable start...");
    }
}) {
    @Override
    public void run() {
        System.out.println("sub start....");
    }
}.start();</pre>
## 4.实现Callable接口--带返回值并抛出异常 ##
FutureTask implements RunnableFuture
RunnableFuture<V> extends Runnable, Future
即FutureTask是对线程任务的封装,可将thread4封装成一个FutureTask
将Task任务放到Thread中去执行
```
public class Thread4 implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        System.out.println("正在进行紧张的计算...");
        Thread.sleep(3000);
        return 1;
    }
    public static void main(String[] args) throws Exception {
        Thread4 thread4 = new Thread4();
        FutureTask<Integer> task = new FutureTask<>(thread4);
        // task包装到Thread中去执行
        Thread thread = new Thread(task);
        thread.start();
        System.out.println("我先干点别的");
        // 取出返回结果
        Integer result = task.get();
        System.out.println("线程执行的结果为:"+result);
    }
}
```
## 5.定时器 ##
除了JDK给我们提供的Timer类外,还有第三方的定时任务的框架Quartz
利用Timer的schedule去定时执行任务,TimerTask也是继承Runnable接口实现run方法的线程任务
<pre name="code" class="java">public class Thread5 {
    public static void main(String[] args) {
        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                // 实现定时任务
                System.out.println("timertask is run");
            }
        }, 0, 1000);
    }
}</pre>
实现起来比较简单,相对比较灵活,但是有一个非常大的问题是不可控,控制起来比较麻烦,当任务没有执行完毕或每次执行不同任务时没法对它定制化;如果是quartz框架则都帮我们实现好了
## 6.使用线程池实现多线程 ##
<pre name="code" class="java">public class Thread6 {
    public static void main(String[] args) {
        //ExecutorService threadPool = Executors.newFixedThreadPool(10);
        ExecutorService threadPool = Executors.newCachedThreadPool();
        // 向线程池提交任务去运行
        for (int i = 0; i < 10; i++) {
            threadPool.execute(new Runnable() {
                @Override
                public void run() {
                    System.out.println(Thread.currentThread().getName());
                }
            });
        }
        threadPool.shutdown();
    }
}</pre>
## 7.Spring实现多线程 ##
Spring有对并发的支持,Spring的异步任务
<pre name="code" class="java">@Configuration
@ComponentScan("com.eric.thread.createthreadmode")
@EnableAsync
public class Config {
}</pre>
<pre name="code" class="java">@Service
public class DemoService {
    @Async
    public void a() {
        while (true) {
            System.out.println("a");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    @Async
    public void b() {
        while (true) {
            System.out.println("b");
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}</pre>
<pre name="code" class="java">public class Main {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ac = new AnnotationConfigApplicationContext(Config.class);
        DemoService ds = ac.getBean(DemoService.class);
        ds.a();
        ds.b();
    }
}</pre>
## 8.使用lambda实现多线程 ##
<pre name="code" class="java">public class Thread7 {
    public int add(List<Integer> values) {
        //values.parallelStream().forEach(System.out::println);//排序不一定
        values.parallelStream().forEachOrdered(System.out::println);
        return values.parallelStream().mapToInt(a -> a).sum();
    }
    public static void main(String[] args) {
        List<Integer> values = Arrays.asList(10,20,30,40);
        int result = new Thread7().add(values);
        System.out.println(result);
    }
}</pre>
