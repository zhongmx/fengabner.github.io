---
title: java并发之park和unpark
date: 2020-07-02 00:02:43
categories: java 并发
tags:
	- java
	- 并发
---

# 基本使用

## 方法介绍

+ LockSupport.park(); 暂停当前线程
+ LockSupport.unpark(); 恢复某个线程的运行

<!--more-->

## 简单例子

先park再unpark

```java
@Slf4j
public class ParkUnparkOneTest {

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            log.debug("start...");
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("park...");
            LockSupport.park();
            log.debug("resume...");
        }, "t1");
        t1.start();

        try {
            TimeUnit.SECONDS.sleep(2);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        log.debug("unpark...");
        LockSupport.unpark(t1);
    }
}
```

```
23:51:57.941 [t1] DEBUG com.fengabner.business.ParkUnparkOneTest - start...
23:51:58.950 [t1] DEBUG com.fengabner.business.ParkUnparkOneTest - park...
23:51:59.943 [main] DEBUG com.fengabner.business.ParkUnparkOneTest - unpark...
23:51:59.943 [t1] DEBUG com.fengabner.business.ParkUnparkOneTest - resume...

Process finished with exit code 0
```

先unpark再park

```java
@Slf4j
public class ParkUnparkTwoTest {

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(() -> {
            log.debug("start...");
            try {
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.debug("park...");
            LockSupport.park();
            log.debug("resume...");
        }, "t1");
        t1.start();

        TimeUnit.SECONDS.sleep(1);
        log.debug("unpark...");
        LockSupport.unpark(t1);
    }
}
```

```
23:56:50.996 [t1] DEBUG com.fengabner.business.ParkUnparkTwoTest - start...
23:56:51.999 [main] DEBUG com.fengabner.business.ParkUnparkTwoTest - unpark...
23:56:53.004 [t1] DEBUG com.fengabner.business.ParkUnparkTwoTest - park...
23:56:53.004 [t1] DEBUG com.fengabner.business.ParkUnparkTwoTest - resume...

Process finished with exit code 0
```

# 原理分析

每个java线程都有一个Parker实例，Parker类定义如下：

```c++
class Parker : public os::PlatformParker {
private:
  volatile int _counter ;
  ...
public:
  void park(bool isAbsolute, jlong time);
  void unpark();
  ...
}
class PlatformParker : public CHeapObj<mtInternal> {
  protected:
    pthread_mutex_t _mutex [1] ;
    pthread_cond_t  _cond  [1] ;
    ...
}
```

Parker 对象，由三部分组成 _counter ， _cond 和 _mutex。Parker类里的 _counter字段，就是用来记录所谓的“许可”的。

**先调用park**

![image-20200630003257340](image-20200630003257340.png)

当调用park时，先去尝试能否直接拿到“许可”，也就时 _counter>0，如果成功，则把 _counter 设置为0，并返回。如果不成功，则把  _counter设置为0，获得 _mutex 互斥锁。

**再调用 unpark**

1、设置 _counter为1

2、唤醒 _cond 条件变量中的 Thread_0

3、Thread_0 恢复运行

4、设置 _counter 为 0

**先调用 unpark 再调用 park**

![image-20200630004037812](image-20200630004037812.png)

1、调用 unpark ,设置_counter 为 1

2、当前线程调用 park，检查 _counter，此时 _counter 为1，这是线程无需阻塞，继续运行

3、设置 _counter 为0

# 对比 wait/notify

+ wait，notify 和 notifyAll 必须配合 Object Monitor 一起使用（必须使用synchronized加锁），而 park，unpark 不必
+ park & unpark 是以线程为单位来【阻塞】和【唤醒】线程，而 notify 只能随机唤醒一个等待线程，notifyAll 是唤醒所有等待线程，就不那么【精确】
+ park & unpark 可以先 unpark，而 wait & notify 不能先 notify

# 固定顺序执行

比如，必须先 2 后 1 打印

```java
@Slf4j
public class ParkUnparkThreeTest {

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            LockSupport.park(); //暂停等待'
            log.debug("1");
        }, "t1");
        t1.start();


        new Thread(() -> {
            log.debug("2");
            LockSupport.unpark(t1); //唤醒t1
        }, "t2").start();
    }
}
```

```
23:46:46.438 [t2] DEBUG com.fengabner.business.ParkUnparkThreeTest - 2
23:46:46.443 [t1] DEBUG com.fengabner.business.ParkUnparkThreeTest - 1

Process finished with exit code 0
```

# 交替输出

例子：交替打印 abc

```java
public class ParkUnparkFourTest {

    private final int loopNums;

    static Thread t1, t2, t3;

    public ParkUnparkFourTest(int loopNums) {
        this.loopNums = loopNums;
    }

    public void print(String message, Thread nextThread) {
        for (int i = 0; i < loopNums; i++) {

            //等待
            LockSupport.park();

            //打印输出并唤醒下一个线程
            System.out.print(message);
            LockSupport.unpark(nextThread);
        }
    }

    public static void main(String[] args) {

        ParkUnparkFourTest parkUnparkFourTest = new ParkUnparkFourTest(10);
        t1 = new Thread(() -> parkUnparkFourTest.print("a", t2));
        t2 = new Thread(() -> parkUnparkFourTest.print("b", t3));
        t3 = new Thread(() -> parkUnparkFourTest.print("c", t1));

        t1.start();
        t2.start();
        t3.start();

        LockSupport.unpark(t1);
    }
}

```

```
abcabcabcabcabcabcabcabcabcabc
Process finished with exit code 0
```



**参考资料**

[全面深入了解并发编程](https://www.bilibili.com/video/BV16J411h7Rd?p=109)

