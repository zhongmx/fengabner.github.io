---
title: java并发基础
date: 2020-06-24 16:58:23
categories: java 并发
tags:
	- java
	- 并发
---

# 进程线程的概念

## 进程

程序由指令和数据组成，但这些指令要运行，数据要读写，就必须将指令加载至 CPU，数据加载至内存。在指令运行过程中还需要用到磁盘、网络等设备。进程就是用来加载指令、管理内存、管理 IO 的 。

当一个程序被运行，从磁盘加载这个程序的代码至内存，这时就开启了一个进程。 进程就可以视为程序的一个实例。

<!--more-->

## 线程

一个进程之内可以分为一到多个线程。 

一个线程就是一个指令流，将指令流中的一条条指令以一定的顺序交给 CPU 执行

Java 中，线程作为最小调度单位，进程作为资源分配的最小单位。 在 windows 中进程是不活动的，只是作 为线程的容器

## 两者对比

+ 进程基本上相互独立的，而线程存在于进程内，是进程的一个子集 

+ 进程拥有共享的资源，如内存空间等，供其内部的线程共享

+ 进程间通信较为复杂
  + 同一台计算机的进程通信称为 IPC(Inter-process communication) 
  + 不同计算机之间的进程通信，需要通过网络，并遵守共同的协议，例如 HTTP

+ 线程通信相对简单，因为它们共享进程内的内存，一个例子是多个线程可以访问同一个共享变量

+ 线程更轻量，线程上下文切换成本一般上要比进程上下文切换低

# 并发并行的概念

**并行:** 指两个或多个事件在同一时刻点发生

**并发:** 指两个或多个事件在同一时间段内发生

# Java 创建线程方式

```java
@Slf4j
public class ThreadStartMethod {

    private static class UseThread extends Thread{

        @Override
        public void run(){
            log.info("我是通过继承Thread方式启动的");
        }
    }

    private static class UseRunnable implements Runnable{

        @Override
        public void run() {
            log.info("我是通过实现Runnable方式启动的");
        }
    }

    private static class UseCallable implements Callable{

        @Override
        public String call() throws Exception {
            log.info("我是通过实现Callable方式启动的,具备返回值");
            return "this is callable";
        }
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        UseThread useThread = new UseThread();
        new Thread(useThread).start();

        UseRunnable useRunnable = new UseRunnable();
        new Thread(useRunnable).start();

        UseCallable useCallable = new UseCallable();
        FutureTask<String> futureTask = new FutureTask<String>(useCallable);
        new Thread(futureTask).start();


        Thread.sleep(3000);

        //get 方法阻塞
        log.info(futureTask.get());

    }
}
```

```
16:59:16.970 [Thread-3] INFO com.fengabner.business.ThreadStartMethod - 我是通过实现Callable方式启动的,具备返回值
16:59:16.970 [Thread-2] INFO com.fengabner.business.ThreadStartMethod - 我是通过实现Runnable方式启动的
16:59:16.970 [Thread-1] INFO com.fengabner.business.ThreadStartMethod - 我是通过继承Thread方式启动的
16:59:19.970 [main] INFO com.fengabner.business.ThreadStartMethod - this is callable

Process finished with exit code 0
```



# Java 线程状态

![image-20200620231936167](image-20200620231936167.png)

代码测试

```java
public class ThreadState {
    public static void main(String[] args) throws IOException {
        Thread t1 = new Thread("t1") {
            @Override
            public void run() {
                log.debug("running...");
            }
        };

        Thread t2 = new Thread("t2") {
            @Override
            public void run() {
                while(true) { 

                }
            }
        };
        t2.start();

        Thread t3 = new Thread("t3") {
            @Override
            public void run() {
                log.debug("running...");
            }
        };
        t3.start();

        Thread t4 = new Thread("t4") {
            @Override
            public void run() {
                synchronized (ThreadState.class) {
                    try {
                        Thread.sleep(1000000); 
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t4.start();

        Thread t5 = new Thread("t5") {
            @Override
            public void run() {
                try {
                    t2.join(); 
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        };
        t5.start();

        Thread t6 = new Thread("t6") {
            @Override
            public void run() {
                synchronized (ThreadState.class) { 
                    try {
                        Thread.sleep(1000000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        };
        t6.start();

        try {
            Thread.sleep(500);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("t1 state {}", t1.getState());
        log.debug("t2 state {}", t2.getState());
        log.debug("t3 state {}", t3.getState());
        log.debug("t4 state {}", t4.getState());
        log.debug("t5 state {}", t5.getState());
        log.debug("t6 state {}", t6.getState());
    }
}
```

运行结果

```
23:11:18.172 [thread3] DEBUG com.fengabner.business.ThreadState - running...
23:11:18.808 [main] DEBUG com.fengabner.business.ThreadState - t1 state NEW
23:11:18.811 [main] DEBUG com.fengabner.business.ThreadState - t2 state RUNNABLE
23:11:18.811 [main] DEBUG com.fengabner.business.ThreadState - t3 state TERMINATED
23:11:18.811 [main] DEBUG com.fengabner.business.ThreadState - t4 state TIMED_WAITING
23:11:18.811 [main] DEBUG com.fengabner.business.ThreadState - t5 state WAITING
23:11:18.811 [main] DEBUG com.fengabner.business.ThreadState - t6 state BLOCKED

```



+ **初始(NEW)**：新创建了一个线程对象，但还没有调用start()方法。
+ **运行(RUNNABLE)**：Java线程中将就绪（ready）和运行中（running）两种状态笼统的称为“运行”。
  线程对象创建后，其他线程(比如main线程）调用了该对象的start()方法。该状态的线程位于可运行线程池中，等待被线程调度选中，获取CPU的使用权，此时处于就绪状态（ready）。就绪状态的线程在获得CPU时间片后变为运行中状态（running）。
+ **阻塞(BLOCKED)**：表示线程阻塞于锁。
+ **等待(WAITING)**：进入该状态的线程需要等待其他线程做出一些特定动作（通知或中断）。
+ **超时等待(TIMED_WAITING)**：该状态不同于WAITING，它可以在指定的时间后自行返回。
+ **终止(TERMINATED)**：表示该线程已经执行完毕。



**重新理解**

1、**NEW–>RUNABLE**

+ 当调用 t.start()方法时，由 NEW --> RUNNABLE

2、**RUNABL<–>WATING**

t 线程用 synchronized(obj) 获取了对象锁后

+ 调用 obj.wait()方法时，t 线程从RUNNABLE --> WAITING
+ 调用 obj.notify()，obj.notifyAll()，t.interrupt()时
  + 如果竞争失败，t 线程从 WAITING --> RUNNABLE，线程进入Monitor中EntryList
  + 如果竞争成功，t 线程从 WAITING --> BLOCKED，Monitor中Owner指向该线程

3、**RUNNABLE <–> WAITING**

+ 当前线程调用t.join()方法时，当前线程从 RUNNABLE --> WAITING
  + 注意是当前线程在t 线程对象的监视器上等待
+ t 线程运行结束，或调用了当前线程的 interrupt() 时，当前线程从 WAITING --> RUNNABLE

4、**RUNNABLE <–> WAITING**

+ 当前线程调用LockSupport.park()方法会让当前线程从 RUNNABLE --> WAITING
+ 调用 LockSupport.unpark(目标线程)或调用了线程 的interrupt()，会让目标线程从 WAITING --> RUNNABLE

5、**RUNNABLE <–> TIMED_WAITING**

t 线程用 synchronized(obj)获取了对象锁后

+ 调用 obj.wait(long n)方法时，t 线程从 RUNNABLE --> TIMED_WAITING
+ t 线程等待时间超过了 n 毫秒，或调用 obj.notify()， obj.notifyAll()， t.interrupt()时
  + 竞争锁成功，t 线程从 TIMED_WAITING --> RUNNABLE
  + 竞争锁失败，t 线程从 TIMED_WAITING --> BLOCKED

6、 **RUNNABLE <–> TIMED_WAITING**

+ 当前线程调用t.join(long n)方法时，当前线程从 RUNNABLE --> TIMED_WAITING
  + 注意是当前线程在t 线程对象的监视器上等待
+ 当前线程等待时间超过了 n 毫秒，或t 线程运行结束，或调用了当前线程的interrupt() 时，当前线程从 TIMED_WAITING --> RUNNABLE

7、**RUNNABLE <–> TIMED_WAITING**

+ 当前线程调用 Thread.sleep(long n) ，当前线程从 RUNNABLE --> TIMED_WAITING

+ 当前线程等待时间超过了 n 毫秒，当前线程从 TIMED_WAITING --> RUNNABLE

8、**RUNNABLE <–> TIMED_WAITING**

+ 当前线程调用 LockSupport.parkNanos(long nanos)或 LockSupport.parkUntil(long millis)时，当前线程从 RUNNABLE --> TIMED_WAITING
+ 调用LockSupport.unpark(目标线程)或调用了线程 的 interrupt()，或是等待超时，会让目标线程从 TIMED_WAITING--> RUNNABLE

9、 **RUNNABLE <–> BLOCKED**

+ t 线程用 synchronized(obj) 获取了对象锁时如果竞争失败，从 RUNNABLE --> BLOCKED
+ 持 obj 锁线程的同步代码块执行完毕，会唤醒该对象上所有 BLOCKED 的线程重新竞争，如果其中 t 线程竞争 成功，从 BLOCKED --> RUNNABLE，其它失败的线程仍然 BLOCKED

10、**RUNNABLE <–> TERMINATED**

+ 当前线程所有代码运行完毕，进入 TERMINATED



# 常用方法

##  sleep

Sleep 不会释放锁

> java并发编程之美

Thread 类中有一个静态的 sleep 方法，当一个执行中的线程调用了 Thread 的 sleep 方法后，调用线程会暂时让出指定时间的执行权，也就是在这期间不参与 CPU 的调度，但是该线程所拥有的监视器资源，比如锁还是持有不让出的。指定的睡眠时间到了后该函数就会正常返回，线程就处于就绪状态，然后参与 CPU 调度，获取到 CPU 资源后就可以继续执行了。如果在睡眠期间其他线程调用了该线程的 interrupt() 方法中断了该线程，则该线程会在调用 sleep方法的地方抛出 InterruptedException() 异常而返回。

线程在睡眠时拥有的监视器资源不会被释放：

```java
@Slf4j
public class SleepTest {
    /**
     * 创建一个独占锁
     */
    private static final Lock lock = new ReentrantLock();

    public static void main(String[] args) {
        //创建线程A
        Thread threadA = new Thread(() -> {
            //获取独占锁
            lock.lock();
            try {
                log.info("child threadA is in sleep......");
                Thread.sleep(1000);
                log.info("child threadA is in awaked......");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                //释放锁
                lock.unlock();
            }
        });

        //创建线程 B
        Thread threadB = new Thread(() -> {
            //获取独占锁
            lock.lock();
            try {
                log.info("child threadB is in sleep......");
                Thread.sleep(1000);
                log.info("child threadB is in awaked......");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                //释放锁
                lock.unlock();
            }
        });

        //启动线程
        threadA.start();
        threadB.start();
    }
}
```

```
17:20:59.202 [Thread-0] INFO com.fengabner.business.SleepTest - child threadA is in sleep......
17:21:00.205 [Thread-0] INFO com.fengabner.business.SleepTest - child threadA is in awaked......
17:21:00.205 [Thread-1] INFO com.fengabner.business.SleepTest - child threadB is in sleep......
17:21:01.205 [Thread-1] INFO com.fengabner.business.SleepTest - child threadB is in awaked......

Process finished with exit code 0
```

当一个线程处于睡眠状态时，如果另外一个线程中断了它，会在调用 sleep 方法出抛出异常。

```java
@Slf4j
public class SleepInterruptTest {

    public static void main(String[] args) throws InterruptedException {
        //创建线程
        Thread thread = new Thread(() -> {
            try {
                log.info("child thread is in sleep......");
                Thread.sleep(10000);
                log.info("child thread is in awaked......");
            } catch (Exception e) {
                e.printStackTrace();
            }
        });

        //启动线程
        thread.start();

        //主线程休眠2s
        Thread.sleep(2000);

        //主线程中断子线程
        thread.interrupt();
    }
}
```

```
17:24:03.522 [Thread-0] INFO com.fengabner.business.SleepInterruptTest - child thread is in sleep......
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at com.fengabner.business.SleepInterruptTest.lambda$main$0(SleepInterruptTest.java:18)
	at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 0
```



## yield

> java并发编程之美

当一个线程调用 yield 方法时，当前线程会让出CPU使用权，然后处于 **就绪状态**，线程调度器会从线程就绪队列里面获取一个线程优先级最高的线程，当然也有可能会调度到刚刚让出 CPU 的那个线程来获取 CPU 的执行权。

```java
@Slf4j
public class YieldTest implements Runnable{

    YieldTest(String name){
        //创建并启动线程
        Thread t = new Thread(this);
        t.setName(name);
        t.start();
    }

    public void run() {
        for (int i = 0; i < 5; i++) {
            //当 i=0 是让出 CPU 执行权，放弃时间片，并进行下一轮调度
            if (i % 5 == 0){
                log.debug(Thread.currentThread().getName() + " yield cpu ...");

                //当前线程让出 cpu 执行权，放弃时间片，进行下一轮调度
                Thread.yield();
            }
        }

        log.debug(Thread.currentThread().getName() + " is over...");
    }

    public static void main(String[] args) {
        new YieldTest("t1");
        new YieldTest("t2");
        new YieldTest("t3");
    }
}
```

```
16:55:01.648 [t1] DEBUG com.fengabner.business.YieldTest - t1 yield cpu ...
16:55:01.648 [t3] DEBUG com.fengabner.business.YieldTest - t3 yield cpu ...
16:55:01.650 [t1] DEBUG com.fengabner.business.YieldTest - t1 is over...
16:55:01.648 [t2] DEBUG com.fengabner.business.YieldTest - t2 yield cpu ...
16:55:01.651 [t2] DEBUG com.fengabner.business.YieldTest - t2 is over...
16:55:01.651 [t3] DEBUG com.fengabner.business.YieldTest - t3 is over...
```

### sleep与yield的区别

- 当线程调用 sleep 方法时，调用线程会被阻塞挂起指定时间，在这期间线程调度器不会调度该线程。
- 调用 yield 方法时，线程只会让出自己剩余的时间片，并没有被阻塞挂起，而是处于就绪状态，线程调度器下一次调度时就有可能调度到当前线程执行。

## join

join() 功能说明：等待线程运行结束。（白话：谁调用join，就等待谁结束再进行运行）

备注：join底层实现原理就是wait

```java
@Slf4j
public class JoinOneTest {

    static class MyThread extends Thread{

        @Override
        public void run(){

            int secondValue = (int) (Math.random() * 10000);

            log.info(Thread.currentThread().getName()+" 计算的结果值为 ===>"+secondValue);

            try {
                Thread.sleep(secondValue);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {

        MyThread thread = new MyThread();
        thread.setName("计算线程");
        thread.start();
        try {
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.info("当前线程"+Thread.currentThread().getName()+"===>我想等 计算线程 执行完毕后我再执行");
    }
}
```

```java
21:48:26.324 [计算线程] INFO com.fengabner.business.JoinOneTest - 计算线程 计算的结果值为 ===>3380
21:48:29.719 [main] INFO com.fengabner.business.JoinOneTest - 当前线程main===>我想等 计算线程 执行完毕后我再执行
```



## interrupt

**interrupt()** ：中断线程。

+ 本线程中断自身是被允许的，且"中断标记"设置为true

+ 其它线程调用本线程的interrupt()方法时，会通过 checkAccess() 检查权限。这有可能抛出 SecurityException 异常。
  + 如果线程在阻塞状态**（调用线程的wait(), wait(long)或wait(long, int)会让它进入等待(阻塞)状态，或者调用线程的join(), join(long), join(long, int), sleep(long), sleep(long, int)也会让它进入阻塞状态）**，调用了它的interrupt()方法，那么它的“中断状态”会被清除并且会收到一个InterruptedException异常
    + 例如：线程通过wait()进入阻塞状态，此时通过interrupt()中断该线程；调用interrupt()会立即将线程的中断标记设为“true”，但是由于线程处于阻塞状态，所以该“中断标记”会立即被清除为“false”，同时，会产生一个InterruptedException的异常。
  + 如果线程被阻塞在一个**Selector**选择器中，那么通过interrupt()中断它时；线程的中断标记会被设置为true，并且它会立即从选择操作中返回。
  + 如果线程**正在运行中**，那么通过interrupt()中断线程时，它的中断标记会被设置为“true”。中断一个“已终止的线程”不会产生任何操作，这通常使用于使用isInterrupted做开关判断的死循环中。

> java并发编程之美解释：
>
> 当线程 A 运行时，线程 B 可以调用钱程 A的 interrupt（） 方法来设置线程 A 的中断标志为 true 并立即返回。设置标志仅仅是设置标志 ， 线程 A 实际并没有被中断， 它会继续往下执行 。 如果线程 A 因为调用了wait 系列函数、 join 方法或者 sleep 方法而被阻塞挂起，这时候若线程 B 调用线程A 的 interrupt（） 方法，线程 A 会在调用这些方法 的地方抛 出 InterruptedException 异常而返回。

```java
public void interrupt() {
        if (this != Thread.currentThread())
            checkAccess();

        synchronized (blockerLock) {
            Interruptible b = blocker;
            if (b != null) {
                interrupt0();           // Just to set the interrupt flag
                b.interrupt(this);
                return;
            }
        }
        interrupt0();
    }
```

**isInterrupted()** : 检查**此线程**是否被中断，不会清除线程的中断状态

**interrupted()** ：判断的是**当前线程**是否处于中断状态。是类的静态方法，同时会清除线程的中断状态。

线程正在运行中 根据中断标志位判断线程是否终止例子

```java
public class InterruptTest3 {

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(()->{
            while (!Thread.currentThread().isInterrupted()) {
                System.out.println(Thread.currentThread().getName() +"未终止...");
            }
            System.out.println(Thread.currentThread().getName() +"已终止");
        });
        t.setName("t");
        t.start();

        TimeUnit.SECONDS.sleep(1);

        System.out.println("main thread interrupt t");
        t.interrupt();

        t.join();

        TimeUnit.SECONDS.sleep(5);
        System.out.println("main is over");

    }
}
```

```
t未终止...
t未终止...
t未终止...
t未终止...
t未终止...
t未终止...
main thread interrupt t
t未终止...
t未终止...
t未终止...
t已终止
main is over
```



线程处于阻塞时，中断线程例子

```java
public class InterruptTest4 {

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(()->{
            try {
                System.out.println(Thread.currentThread().getName() +" begin sleep for 20 seconds");
                TimeUnit.SECONDS.sleep(20);
                System.out.println(Thread.currentThread().getName() +" awaking");
            } catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() +" is interrupted while sleeping");
                e.printStackTrace();
            }
        });

        t.setName("t");
        t.start();

        TimeUnit.SECONDS.sleep(1);

        System.out.println("main thread interrupt t");
        t.interrupt();

        t.join();

        System.out.println("main is over");
    }
}
```

```
t begin sleep for 20 seconds
main thread interrupt t
t is interrupted while sleeping
main is over
java.lang.InterruptedException: sleep interrupted
	at java.lang.Thread.sleep(Native Method)
	at java.lang.Thread.sleep(Thread.java:340)
	at java.util.concurrent.TimeUnit.sleep(TimeUnit.java:386)
	at com.fengabner.business.InterruptTest4.lambda$main$0(InterruptTest4.java:16)
	at java.lang.Thread.run(Thread.java:748)

Process finished with exit code 0
```

**isInterrupted()**  和  **interrupt()** 不同之处

例子：

```java
public class InterruptTest1 {

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            for (;;) {

            }
        });
        t.start();

        t.interrupt();

        System.out.println("线程是否被中断:" + t.isInterrupted());//A

        System.out.println("线程是否被中断:" + t.interrupted());//B

        System.out.println("线程是否被中断:" + Thread.interrupted());//C

        System.out.println("线程是否被中断:" + t.isInterrupted());//D

        t.join();

        System.out.println("main thread is over");
    }
}
```

```
线程是否被中断:true
线程是否被中断:false
线程是否被中断:false
线程是否被中断:true
```

从例子可以看出，A 和 D 处都会输出为 true。也说明 isInterrupted（）方法并不会清除中断状态。

在这里可能会有一个疑惑，为什么 B 和 C 处输出的不是一个 true 一个 false。其实这里有一个 **巨大的坑**，注意 **interrupted()** 判断的是 **当前线程** 是否处于中断状态， **当前线程**  **当前线程**  **当前线程** 。这里当前线程是main线程，而 t.interrupt(）中断的是thread线程，这里的此线程就是 t 线程。所以当前线程main从未被中断过，尽管interrupted\() 方法是以 t.interrupted() 的形式被调用，但它检测的仍然是main线程而不是检测 t 线程，所以 t.interrupted() 在这里相当于main.interrupted()。

改一下代码测试一下：

```java
public class InterruptTest1 {

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            for (;;) {

            }
        });
        t.start();

        t.interrupt();

        //中断当前线程（mian 线程）
        Thread.currentThread().interrupt();

        System.out.println("线程是否被中断:" + t.isInterrupted());//A

        System.out.println("线程是否被中断:" + t.interrupted());//B

        System.out.println("线程是否被中断:" + Thread.interrupted());//C

        System.out.println("线程是否被中断:" + t.isInterrupted());//D

        t.join();

        System.out.println("main thread is over");
    }
}
```

```
线程是否被中断:true
线程是否被中断:true
线程是否被中断:false
线程是否被中断:true
```

