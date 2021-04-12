---
title: java并发之ReentrantLock简单介绍
date: 2020-07-17 22:31:54
categories: java 并发
tags:
	- java
	- 并发
---

# ReentrantLock 简单介绍

ReentrantLock是实现了Lock接口。

## 特点

+ 支持可重入
+ 可中断
+ 支持公平锁和非公平锁
+ 支持多个条件变量设置
+ 可以设置超时时间

<!--more-->

## 基本语法

ReentrantLock 中的 lock 和 unlock 是需要成对出现的。

```java
private Lock lock = new ReentrantLock() ;
public void await() {
    try {
         lock.lock();//加锁
    }  finally {
        lock.unlock();//释放锁
    }
}
```

# Lock api

![image-20200804165753248](image-20200804165753248.png)

```java
public interface Lock {

     // 获取锁，若当前lock已经被其他线程获取时,则当前线程会被阻塞一直到lock被释放。采用lock,必须主动去释放锁，并且在发生异常时，不会自动释放锁
    void lock();

    // 获取锁，若当前锁不可用,则当前线程会被阻塞,等待获取锁，在等待过程中这个线程能够响应中断，即中断线程的等待状态,会抛出异常
    void lockInterruptibly() throws InterruptedException;

    // 尝试获取锁，如果获取成功，则返回true,如果获取失败（即锁已被其他线程获取），则返回false。这个方法无论如何都会立即返回
    boolean tryLock();

    // 如果锁在给定等待时间内没有被另一个线程持有，并且当前线程未被中断，则获取该锁
    // 等待过程中，可以被中断
    // 超过时间，依然获取不到，则返回false；否则返回true
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    // 释放锁
    void unlock();

    // 返回一个绑定该lock的Condtion对象
    // 在Condition await()之前，锁会被该线程持有
    // Condition await()之后 会自动释放锁，在wait返回之后，会自动获取锁
    Condition newCondition();
}
```

# ReentrantLock api

![image-20200730144344679](image-20200730144344679.png)

```java
public ReentrantLock()// 默认非公平锁

public ReentrantLock(boolean fair)// fair为true表示是公平锁，fair为false表示是非公平锁

public void lock()// 获取锁

public void lockInterruptibly()// 中断等待获取锁的线程,如果当前线程未被中断,则去获取锁,已被中断则抛异常

public boolean tryLock()// 在调用时,锁如果没有被另一个线程持有的情况下，才获取该锁

public boolean tryLock(long timeout, TimeUnit unit)// 如果锁在给定等待时间内没有被另一个线程持有，并且当前线程未被中断，则获取该锁

public void unlock()// 释放锁

public Condition newCondition()// 返回用来与此 Lock 实例一起使用的 Condition 实例

public int getHoldCount()// 查询当前线程保持锁定的个数，也就是调用lock()方法的个数
  
public boolean isHeldByCurrentThread()// 查询当前线程是否持有此锁
  
public boolean isLocked()// 查询此锁是否由任意线程所持有
  
public final boolean isFair()// 判断是否是公平锁,如果是公平锁返回true，否则返回false

protected Thread getOwner()// 返回当前拥有此锁的线程，如果此锁不被任何线程拥有，则返回 null
  
public final boolean hasQueuedThreads()// 查询是否有线程正在等待获取此锁
  
public final boolean hasQueuedThread(Thread thread)// 查询指定线程是否正在等待获取此锁
  
public final int getQueueLength()// 返回正在等待获取此锁的线程估计数
  
protected Collection<Thread> getQueuedThreads()// 返回一个 collection，它包含可能正等待获取此锁的线程
  
public boolean hasWaiters(Condition condition)// 查询是否有线程正在等待与此锁有关的指定条件
  
public int getWaitQueueLength(Condition condition)// 返回正在等待与此锁相关的指定条件的线程估计数
  
protected Collection<Thread> getWaitingThreads(Condition condition)// 返回一个 collection，它包含可能正在等待与此锁相关指定条件的哪些线程
```



# ReentrantLock 简单使用

## lock() 和 unlock()

```java
public class ReentrantLockOneTest {


    static class MyService{
        private Lock lock = new ReentrantLock();

        void testMethod(){
            lock.lock();//获取锁

            System.out.println("线程 【"+Thread.currentThread().getName()+"】 进来了 获取到锁 开始");
            for (int i=0;i<4;i++){
                System.out.println(Thread.currentThread().getName() +(i+1));
            }

            System.out.println("线程 【"+Thread.currentThread().getName()+"】 释放锁 结束");
            lock.unlock();//释放锁
        }
    }

    static class MyThread extends Thread{

        private MyService myService;

        MyThread(MyService myService){
            super();
            this.myService = myService;
        }

        @Override
        public void run(){
            myService.testMethod();
        }
    }

    public static void main(String[] args) {
        MyService service = new MyService();

        MyThread a1 = new MyThread(service);
        MyThread a2 = new MyThread(service);
        MyThread a3 = new MyThread(service);
        MyThread a4 = new MyThread(service);
        MyThread a5 = new MyThread(service);

        a1.setName("a1==>");
        a1.start();

        a2.setName("a2==>");
        a2.start();

        a3.setName("a3==>");
        a3.start();

        a4.setName("a4==>");
        a4.start();

        a5.setName("a5==>");
        a5.start();
    }
}
```

```
线程 【a1==>】 进来了 获取到锁 开始
a1==>1
a1==>2
a1==>3
a1==>4
线程 【a1==>】 释放锁 结束
线程 【a2==>】 进来了 获取到锁 开始
a2==>1
a2==>2
a2==>3
a2==>4
线程 【a2==>】 释放锁 结束
线程 【a3==>】 进来了 获取到锁 开始
a3==>1
a3==>2
a3==>3
a3==>4
线程 【a3==>】 释放锁 结束
线程 【a4==>】 进来了 获取到锁 开始
a4==>1
a4==>2
a4==>3
a4==>4
线程 【a4==>】 释放锁 结束
线程 【a5==>】 进来了 获取到锁 开始
a5==>1
a5==>2
a5==>3
a5==>4
线程 【a5==>】 释放锁 结束

Process finished with exit code 0

```

## tryLock()方法

> 在调用时,锁如果没有被另一个线程持有的情况下，才获取该锁

```java
@Slf4j
public class ReentrantLockTryLock {

    public ReentrantLock lock = new ReentrantLock();

    public void serviceHandle(){
        try {
            if (lock.tryLock()) {
                log.info(Thread.currentThread().getName() + "获取到锁 （🔒） ");
                Thread.sleep(4000);
            } else {
                log.info(Thread.currentThread().getName() + "未获取到锁（🔒）");
            }
        } catch (Exception e) {
            e.printStackTrace();
        }finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) {
        ReentrantLockTryLock reentrantLockTryLock = new ReentrantLockTryLock();

        new Thread(reentrantLockTryLock::serviceHandle,"t1").start();

        new Thread(reentrantLockTryLock::serviceHandle,"t2").start();
    }
}
```

```shell
00:33:58.045 [t2] INFO com.fengabner.lock.ReentrantLockTryLock - t2未获取到锁（🔒）
00:33:58.045 [t1] INFO com.fengabner.lock.ReentrantLockTryLock - t1获取到锁 （🔒） 

Process finished with exit code 0
```

## lockInterruptibly()方法

> 中断等待获取锁的线程,如果当前线程未被中断,则去获取锁,已被中断则抛异常。
>
> 当两个线程同时通过lock.lockInterruptibly()获取某个锁时，假若此时线程A获取到了锁，而线程B只有等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

```java
@Slf4j
public class ReentrantLockInterruptibly {

    public ReentrantLock lock = new ReentrantLock();

    public void serviceHandle(){
        try {
            log.info("thread {} get lock",Thread.currentThread().getName());
            lock.lockInterruptibly();

            log.info("thread {} has lock",Thread.currentThread().getName());

            for (int i=0;i<5;i++) {
                Thread.sleep(1000);
                log.info("Thread {} do something...{}",Thread.currentThread().getName(),i);
            }
            log.info("thread {} lock end",Thread.currentThread().getName());
        } catch (InterruptedException e) {
            log.error("thread {} 被中断",Thread.currentThread().getName());
            e.printStackTrace();
        }finally {
            if (lock.isHeldByCurrentThread()) {// 查询当前线程是否持有此锁
                lock.unlock();
                log.info("thread {} 释放锁",Thread.currentThread().getName());
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ReentrantLockInterruptibly reentrantLockInterruptibly = new ReentrantLockInterruptibly();
        new Thread(reentrantLockInterruptibly::serviceHandle,"t1").start();

        Thread t2 = new Thread(reentrantLockInterruptibly::serviceHandle,"t2");


        Thread.sleep(100);

        t2.start();

        Thread.sleep(100);
        // 如果线程t2没有得到锁，中断t2的等待
        t2.interrupt();
        log.info("main end ...");
    }
}
```

```shell
23:44:02.148 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - thread t1 get lock
23:44:02.156 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - thread t1 has lock
23:44:02.249 [t2] INFO com.fengabner.lock.ReentrantLockInterruptibly - thread t2 get lock
23:44:02.349 [main] INFO com.fengabner.lock.ReentrantLockInterruptibly - main end ...
23:44:02.349 [t2] ERROR com.fengabner.lock.ReentrantLockInterruptibly - thread t2 被中断
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
	at com.fengabner.lock.ReentrantLockInterruptibly.serviceHandle(ReentrantLockInterruptibly.java:21)
	at java.lang.Thread.run(Thread.java:748)
23:44:03.158 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - Thread t1 do something...0
23:44:04.161 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - Thread t1 do something...1
23:44:05.163 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - Thread t1 do something...2
23:44:06.167 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - Thread t1 do something...3
23:44:07.170 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - Thread t1 do something...4
23:44:07.170 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - thread t1 lock end
23:44:07.170 [t1] INFO com.fengabner.lock.ReentrantLockInterruptibly - thread t1 释放锁

Process finished with exit code 0
```

## getHoldCount()

> 查询当前线程保持锁定的个数，也就是调用lock()方法的个数

```java
@Slf4j
public class ReentrantLockGetHoldCount {

    private ReentrantLock lock = new ReentrantLock();

    public void serviceHandle1(){
        try {
            lock.lock();

            log.info("{} getHoldCount : {}",Thread.currentThread().getName(),lock.getHoldCount());

            this.serviceHandle2();
        }finally {
            lock.unlock();
        }
    }

    public void serviceHandle2(){
        try {
            lock.lock();
            log.info("{} getHoldCount : {}",Thread.currentThread().getName(),lock.getHoldCount());
        }finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        ReentrantLockGetHoldCount reentrantLockGetHoldCount = new ReentrantLockGetHoldCount();
        new Thread(reentrantLockGetHoldCount::serviceHandle1,"t1").start();
    }
}
```

```shell
00:40:33.573 [t1] INFO com.fengabner.lock.ReentrantLockGetHoldCount - t1 getHoldCount : 1
00:40:33.581 [t1] INFO com.fengabner.lock.ReentrantLockGetHoldCount - t1 getHoldCount : 2

Process finished with exit code 0
```

## isHeldByCurrentThread()

> 查询当前线程是否持有此锁

```java
@Slf4j
public class ReentrantLockIsHeldByCurrentThread {

    ReentrantLock lock = new ReentrantLock();

    private void serviceHandle() {
        try {
            log.info("当前线程{}是否持有锁 {}", Thread.currentThread().getName(), lock.isHeldByCurrentThread());
            lock.lock();
            log.info("当前线程{}是否持有锁 {}", Thread.currentThread().getName(), lock.isHeldByCurrentThread());
        } finally {
            lock.unlock();
            log.info("当前线程{}释放了锁", Thread.currentThread().getName());
        }
    }

    public static void main(String[] args) {
        ReentrantLockIsHeldByCurrentThread reentrantLockIsHeldByCurrentThread = new ReentrantLockIsHeldByCurrentThread();
        new Thread(reentrantLockIsHeldByCurrentThread::serviceHandle, "t1").start();
    }
}
```

```shell
11:45:24.377 [t1] INFO com.fengabner.lock.ReentrantLockIsHeldByCurrentThread - 当前线程t1是否持有锁 false
11:45:24.384 [t1] INFO com.fengabner.lock.ReentrantLockIsHeldByCurrentThread - 当前线程t1是否持有锁 true
11:45:24.384 [t1] INFO com.fengabner.lock.ReentrantLockIsHeldByCurrentThread - 当前线程t1释放了锁

Process finished with exit code 0
```

## getWaitQueueLength(Condition condition)

> 返回正在等待与此锁相关的指定条件的线程估计数

```java
@Slf4j
public class ReentrantLockGetWaitQueueLength {

    ReentrantLock lock = new ReentrantLock();
    public Condition condition = lock.newCondition();

    public void serviceWaitHandle() {
        try {
            lock.lock();
            condition.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void serviceSignalHandle() {
        try {
            lock.lock();
            log.info("当前有{}个正在等待 condition", lock.getWaitQueueLength(condition));
            condition.signal();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ReentrantLockGetWaitQueueLength lockGetWaitQueueLength = new ReentrantLockGetWaitQueueLength();
        for (int i = 0; i < 10; i++) {
            new Thread(lockGetWaitQueueLength::serviceWaitHandle).start();
        }

        Thread.sleep(1000);

        for (int j = 0; j < 10; j++) {
            new Thread(lockGetWaitQueueLength::serviceSignalHandle).start();
        }
    }
}
```

```shell
15:30:30.508 [Thread-10] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有10个正在等待 condition
15:30:30.525 [Thread-11] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有9个正在等待 condition
15:30:30.527 [Thread-12] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有8个正在等待 condition
15:30:30.527 [Thread-13] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有7个正在等待 condition
15:30:30.528 [Thread-14] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有6个正在等待 condition
15:30:30.529 [Thread-15] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有5个正在等待 condition
15:30:30.529 [Thread-16] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有4个正在等待 condition
15:30:30.529 [Thread-17] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有3个正在等待 condition
15:30:30.530 [Thread-18] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有2个正在等待 condition
15:30:30.530 [Thread-19] INFO com.fengabner.lock.ReentrantLockGetWaitQueueLength - 当前有1个正在等待 condition

Process finished with exit code 0
```

