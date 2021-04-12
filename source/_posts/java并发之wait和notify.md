---
title: java并发之wait和notify
date: 2020-06-27 23:03:58
categories: java 并发
tags:
	- java
	- 并发
---

# 方法介绍

+ obj.wait() 让进入 object 监视器的线程到 waitSet 等待，注意必须是获得对象锁的像锁的线程才能调用。wait方法会释放对象的锁，进入 WaitSet 等待区，从而让其他线程就机会获取对象的锁。无限制等待，直到 notify 为止。
+ obj.wait(long n) 有时限的等待, 到 n 毫秒后结束等待，或是被 notify。
+ obj.notify() 在 object 上正在 waitSet 等待的线程中挑一个唤醒。
+ obj.notifyAll() 让 object 上正在 waitSet 等待的线程全部唤醒。

<!--more-->

**简单示例**

```java
@Slf4j
public class WaitNotifyTwoTest {
    final static Object obj = new Object();

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            synchronized (obj) {
                log.debug("执行....");
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("do something....");
            }
        },"t1").start();
        new Thread(() -> {
            synchronized (obj) {
                log.debug("执行....");
                try {
                    obj.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                log.debug("do something....");
            }
        },"t2").start();

        TimeUnit.SECONDS.sleep(2);
        log.debug("唤醒 obj 上其它等待线程");
        synchronized (obj) {
            obj.notify(); // 随机唤醒obj上的一个等待线程
        }
    }
}
```

使用 notify()

```
10:38:24.631 [t1] DEBUG com.fengabner.business.WaitNotifyTwoTest - 执行....
10:38:24.638 [t2] DEBUG com.fengabner.business.WaitNotifyTwoTest - 执行....
10:38:26.638 [main] DEBUG com.fengabner.business.WaitNotifyTwoTest - 唤醒 obj 上其它等待线程
10:38:26.638 [t1] DEBUG com.fengabner.business.WaitNotifyTwoTest - do something....
```

使用 notifyAll()

```
10:39:09.904 [t1] DEBUG com.fengabner.business.WaitNotifyTwoTest - 执行....
10:39:09.911 [t2] DEBUG com.fengabner.business.WaitNotifyTwoTest - 执行....
10:39:11.912 [main] DEBUG com.fengabner.business.WaitNotifyTwoTest - 唤醒 obj 上其它等待线程
10:39:11.912 [t2] DEBUG com.fengabner.business.WaitNotifyTwoTest - do something....
10:39:11.912 [t1] DEBUG com.fengabner.business.WaitNotifyTwoTest - do something....
```

**注意事项**

+ wait()、notify/notifyAll() 方法是Object的本地final方法，无法被重写。
+ wait()、notify/notifyAll() 需要配合 synchronized 关键字使用
+ 当线程执行wait()方法时候，会释放当前的锁，然后让出CPU，进入等待状态。
+ notify/notifyAll() 的执行只是唤醒沉睡的线程，而不会立即释放锁，锁的释放要看代码块的具体执行情况。
+ notify方法只唤醒一个等待（对象的）线程并使该线程开始执行，notifyAll 会唤醒所有等待(对象的)线程。
+ 使用if 还是while
  + notify唤醒沉睡的线程后，线程会接着上次的执行继续往下执行。所以在进行条件判断时候，可以先把 wait 语句忽略不计来进行考虑；显然，要确保程序一定要执行，并且要保证程序直到满足一定的条件再执行，要使用while进行等待，直到满足条件才继续往下执行。

 # wait/notify原理

![image-20200627105711596](image-20200627105711596.png)

+ Owner 线程发现条件不满足，调用 wait 方法，即可进入 WaitSet 变为 WAITING 状态
+ WAITING 线程会在 Owner 线程调用 notify 或 notifyAll 时唤醒，但唤醒后并不意味者立刻获得锁，仍需进入 EntryList 重新竞争
+ BLOCKED 和 WAITING 的线程都处于阻塞状态，不占用 CPU 时间片
+ BLOCKED 线程会在 Owner 线程释放锁时唤醒

# wait和sleep比较

+ 它们 状态都是 TIMED_WAITING。
+ sleep 是 Thread 方法，而 wait 是 Object 的方法。
+ sleep 不需要强制和 synchronized 配合使用，但 wait 需要 和 synchronized 一起用。
+ sleep 在睡眠的同时，不会释放对象锁的，但 wait 在等待的时候会释放对象锁。

# 同步设计模式之保护性暂停

## 概念

![image-20200627111642813](image-20200627111642813.png)

保护性暂停即 Guarded Suspension，用在一个线程等待另一个线程的执行结果。

+ 有一个结果需要从一个线程传递到另一个线程，让他们关联同一个 GuardedObject
+ 如果有结果不断从一个线程到另一个线程那么可以使用消息队列（见生产者/消费者）
+ JDK 中，join 的实现、Future 的实现，采用的就是此模式
+ 因为要等待另一方的结果，因此属于同步模式

## 简单实现

```java
public class GuardedObject {

    private Object response;

    public Object get(){
        synchronized (this) {
            while (response == null) { //防止虚假唤醒
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return response;
        }
    }

    public void complete(Object response){
        synchronized (this) {
            this.response = response;

            this.notifyAll();
        }
    }
}
```

测试

```java
public class Downloader {

    public static List<String> download() throws IOException {
        HttpURLConnection connection = (HttpURLConnection) new URL("https://www.baidu.com/").openConnection();
        List<String> lines = new ArrayList<>();
        try(BufferedReader reader =
                    new BufferedReader(new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8))) {
            String line;
            while ((line =reader.readLine()) != null) {
                lines.add(line);
            }
        }
        return lines;
    }
}
```

```java
@Slf4j
public class Test {

    public static void main(String[] args) {
        GuardedObject guardedObject = new GuardedObject();
        new Thread(() -> {
            log.debug("开始获取数据");
            List<String> list = (List<String>) guardedObject.get();

            log.debug("数据的大小为：{}", list.size());
        }, "t1").start();

        new Thread(() -> {
            log.debug("下载数据");

            try {
                List<String> list = Downloader.download();
                guardedObject.complete(list);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }, "t2").start();
    }
}
```

```
12:39:41.456 [t1] DEBUG com.fengabner.guarded.Test - 开始获取数据
12:39:41.465 [t2] DEBUG com.fengabner.guarded.Test - 下载数据
12:39:42.624 [t1] DEBUG com.fengabner.guarded.Test - 数据的大小为：3

Process finished with exit code 0
```

## 带有超时效果的保护性暂停

```java
public class GuardedObject {

    private Object response;

    public Object get(long timeout){
        synchronized (this){
            //开始时间
            long begin = System.currentTimeMillis();
            //所经历的时间
            long passedTime = 0;
            while (response == null){
                //这一轮循环需要等待的时间
                long waitTime = timeout - passedTime;
                if(waitTime <= 0)
                    break;
                try {
                    //等待唤醒
                    this.wait(waitTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //真实经历的时间
                passedTime = System.currentTimeMillis() - begin;
            }
        }
        return response;
    }

    public void complete(Object response){
        synchronized (this) {
            this.response = response;

            this.notifyAll();
        }
    }
}
```

## join 原理

join 底层实现其实就是用的wait，使用了带有超时效果的保护性暂停。

join源码

```java
    public final void join() throws InterruptedException {
        join(0);//当超时时间 millis == 0时，表示一直等待，没有超时时间
    }
```

```java
public final synchronized void join(long millis)
    throws InterruptedException {
        long base = System.currentTimeMillis();
        long now = 0;

        if (millis < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

  			//如果 == 0，表示不设置超时时间，一直等待
        if (millis == 0) {
            while (isAlive()) {//判断线程是否存活
                wait(0);//一直等待
            }
        } else {
            while (isAlive()) {//判断线程是否存活
                long delay = millis - now;//计算延迟时间
                if (delay <= 0) {//如果延迟时间<=0,退出循环
                    break;
                }
                wait(delay);//等待 计算的delay时间
                now = System.currentTimeMillis() - base;//已经经过的时间
            }
        }
    }
```

# 同步设计模式之保护性暂停扩展

## 概念

**多任务版 GuardedObject**

![image-20200627151330997](image-20200627151330997.png)

**分析**：如果需要在多个类之间使用 GuardedObject 对象，作为参数传递不是很方便，因此设计一个用来解耦的中间类， 这样不仅能够解耦【结果等待者】和【结果生产者】，还能够同时支持多个任务的管理。

## 简单实现

新增 id 用来标识 GuardedObject

```java
public class GuardedObject {

    private Object response;

    //多个GuardedObject时用于标识
    private int id;

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public GuardedObject(int id) {
        this.id = id;
    }


    public Object get(){
        synchronized (this) {
            while (response == null) { //防止虚假唤醒
                try {
                    this.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            return response;
        }
    }

    public Object get(long timeout){
        synchronized (this){
            //开始时间
            long begin = System.currentTimeMillis();
            //所经历的时间
            long passedTime = 0;
            while (response == null){
                //这一轮循环需要等待的时间
                long waitTime = timeout - passedTime;
                if(waitTime <= 0)
                    break;
                try {
                    //等待唤醒
                    this.wait(waitTime);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                //真实经历的时间
                passedTime = System.currentTimeMillis() - begin;
            }
        }
        return response;
    }

    public void complete(Object response){
        synchronized (this) {
            this.response = response;

            this.notifyAll();
        }
    }
}
```

中间解耦类：使用线程安全的Map来存储GuardedObject，用于解耦。

```java
class MailBoxes {

    private static Map<Integer, GuardedObject> map = new Hashtable();

    private static int i = 1;

    //生成唯一id
    private  static synchronized int generateId(){
        return i++;
    }

    //生成GuardedObject
    public static GuardedObject createGuardedObject(){
        GuardedObject go = new GuardedObject(generateId());
        map.put(go.getId(), go);
        return go;
    }

    public static GuardedObject getGuardedObject(int i){
        return map.remove(i);
    }

    public static Set<Integer> getIds(){
        return map.keySet();
    }
}
```

业务相关，收信与送信

**收信人**

```java
@Slf4j
public class People implements Runnable{


    @Override
    public void run() {
        GuardedObject guardedObject = MailBoxes.createGuardedObject();
        log.debug("开始收信了... id:{}",guardedObject.getId());

        Object mail = guardedObject.get(5000);

        log.debug("收到信了... id:{}, 内容:{}",guardedObject.getId(),mail);
    }
}
```

**送信人**

```java
@Slf4j
public class Postman implements Runnable{

    private int id;
    private String mail;

    public Postman(int id, String mail) {
        this.id = id;
        this.mail = mail;
    }

    @Override
    public void run() {
        GuardedObject guardedObject = MailBoxes.getGuardedObject(id);
        log.debug("开始送信了... id:{}, 内容:{}",id,mail);
        guardedObject.complete(mail);
    }
}
```

测试

```java
@Slf4j
public class Test {

    public static void main(String[] args) {
        for (int i=0;i<3;i++) {
            Thread thread = new Thread(new People());
            thread.setName("p"+i);
            thread.start();
        }

        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        for (int id: MailBoxes.getIds()) {
            Thread thread = new Thread(new Postman(id,"这是内容项"+id));
            thread.setName("m"+id);
            thread.start();
        }
    }
}
```

```
15:10:48.405 [p2] DEBUG com.fengabner.guarded.People - 开始收信了... id:3
15:10:48.405 [p1] DEBUG com.fengabner.guarded.People - 开始收信了... id:1
15:10:48.405 [p0] DEBUG com.fengabner.guarded.People - 开始收信了... id:2
15:10:51.407 [m3] DEBUG com.fengabner.guarded.Postman - 开始送信了... id:3, 内容:这是内容项3
15:10:51.407 [p2] DEBUG com.fengabner.guarded.People - 收到信了... id:3, 内容:这是内容项3
15:10:51.407 [m2] DEBUG com.fengabner.guarded.Postman - 开始送信了... id:2, 内容:这是内容项2
15:10:51.408 [p0] DEBUG com.fengabner.guarded.People - 收到信了... id:2, 内容:这是内容项2
15:10:51.408 [m1] DEBUG com.fengabner.guarded.Postman - 开始送信了... id:1, 内容:这是内容项1
15:10:51.408 [p1] DEBUG com.fengabner.guarded.People - 收到信了... id:1, 内容:这是内容项1

Process finished with exit code 0
```

# 同步模式之顺序控制

## 固定顺序运行

例子

```java
@Slf4j
public class WaitNotifyThreeTest {

    //用来同步的对象
    static final Object obj = new Object();
    // t2 运行标记， 代表 t2 是否执行过
    static boolean t2Runed = false;

    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            synchronized (obj) {
                // 如果 t2 没有执行过
                while (!t2Runed) {
                    try {
                        // t1 先等一会
                        obj.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();

                    }
                }
            }
            log.debug("1");
        });

        Thread t2 = new Thread(() -> {
            log.debug("2");
            synchronized (obj) {
                //修改运行标记
                t2Runed = true;
                // 通知 obj 上等待的线程（可能有多个，因此需要用 notifyAll）
                obj.notifyAll();
            }
        });

        t1.start();
        t2.start();
    }

}
```

```
23:40:25.968 [Thread-1] DEBUG com.fengabner.business.WaitNotifyThreeTest - 2
23:40:25.973 [Thread-0] DEBUG com.fengabner.business.WaitNotifyThreeTest - 1

Process finished with exit code 0
```



缺点：

+ 需要保证先 wait 再 notify，否则 wait 线程永远得不到唤醒。因此使用了『运行标记』来判断该不该 wait。

+ 如果有些干扰线程错误地 notify 了 wait 线程，条件不满足时还要重新等待，使用了 while 循环来解决 此问题。

+ 唤醒对象上的 wait 线程需要使用 notifyAll，因为『同步对象』上的等待线程可能不止一个。

因此使用 park unpack 更加灵活。

## 交替输出

例子：交替打印 abc

```java
package com.fengabner.business;

import lombok.extern.slf4j.Slf4j;

/**
 * @Author: fengabner
 * @Date: 2020/7/1 23:32
 * @Version: 1.0
 * @Description:
 */
public class WaitNotifyFourTest {

    private int flag;

    private final int loopNums;

    public WaitNotifyFourTest(int flag, int loopNums) {
        this.flag = flag;
        this.loopNums = loopNums;
    }

    public void print(String message, int flag) {
        for (int i = 0; i < loopNums; i++) {
            synchronized (this) {
                while (this.flag != flag) {
                    try {
                        this.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                System.out.print(message);

                this.flag = (flag + 1) % 3;
                this.notifyAll();
            }
        }
    }

    public static void main(String[] args) {
        WaitNotifyFourTest waitNotifyFourTest = new WaitNotifyFourTest(0, 10);

        new Thread(() -> waitNotifyFourTest.print("a", 0)).start();
        new Thread(() -> waitNotifyFourTest.print("b", 1)).start();
        new Thread(() -> waitNotifyFourTest.print("c", 2)).start();
    }
}

```

```
abcabcabcabcabcabcabcabcabcabc
Process finished with exit code 0
```



**参考资料**

[全面深入了解并发编程](https://www.bilibili.com/video/BV16J411h7Rd)

【java并发编程之美】