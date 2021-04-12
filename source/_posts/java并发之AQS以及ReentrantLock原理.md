---
title: java并发之AQS以及ReentrantLock原理
date: 2020-07-24 22:31:54
categories: java 并发
tags:
	- java
	- 并发
---

# AQS

AQS 全称是 AbstractQueuedSynchronizer，是一种提供了原子式管理同步状态、阻塞和唤醒线程功能以及队列模型的简单框架。ava中的大部分同步类（Lock、Semaphore、ReentrantLock等）都是基于AQS实现的。

<!--more-->

## 状态管理

用 state 属性来表示资源的状态：分独占模式（只有一个线程能执行，如ReentrantLock）和共享模式（多个线程可同时执行，如Semaphore/CountDownLatch），子类需要定义如何维护这个状态，控制如何获取
锁和释放锁

```java
 		/**
     * The synchronization state.
     */
    private volatile int state;//同步状态

    /**
     * Returns the current value of synchronization state.
     * This operation has memory semantics of a {@code volatile} read.
     * @return current state value
     */
    protected final int getState() {//获取同步状态
        return state;
    }

    /**
     * Sets the value of synchronization state.
     * This operation has memory semantics of a {@code volatile} write.
     * @param newState the new state value
     */
    protected final void setState(int newState) {//设置同步状态
        state = newState;
    }

    /**
     * Atomically sets synchronization state to the given updated
     * value if the current state value equals the expected value.
     * This operation has memory semantics of a {@code volatile} read
     * and write.
     *
     * @param expect the expected value
     * @param update the new value
     * @return {@code true} if successful. False return indicates that the actual
     *         value was not equal to the expected value.
     */
    protected final boolean compareAndSetState(int expect, int update) {//CAS操作设置同步状态
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

## 模板方法

AQS采用的模板方法模式，自定义同步器(ReentrantLock为例)需要实现以下方法。

```java
		/**
		*	独占方式。arg为获取锁的次数，尝试获取资源，成功则返回True，失败则返回False。
		*/
		protected boolean tryAcquire(int arg) {
        throw new UnsupportedOperationException();
    }

		/**
		*	独占方式。arg为释放锁的次数，尝试释放资源，成功则返回True，失败则返回False。
		*/
    protected boolean tryRelease(int arg) {
        throw new UnsupportedOperationException();
    }

		/**
		*	共享方式。arg为获取锁的次数，尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
		*/
    protected int tryAcquireShared(int arg) {
        throw new UnsupportedOperationException();
    }

		/**
		*	共享方式。arg为释放锁的次数，尝试释放资源，如果释放后允许唤醒后续等待结点返回True，否则返回False。
		*/
    protected boolean tryReleaseShared(int arg) {
        throw new UnsupportedOperationException();
    }

		/**
		*	该线程是否正在独占资源。只有用到Condition才需要去实现它。
		*/
    protected boolean isHeldExclusively() {
        throw new UnsupportedOperationException();
    }

```



## CLH队列

### FIFO

CLH(Craig, Landin, and Hagersten)锁，是自旋锁的一种。AQS中使用了CLH锁的一个变种，实现了一个虚拟双向队列（FIFO），队列头节点称作“哨兵节点”或者“哑节点”，它不与任何线程关联。其他的节点与等待线程关联，每个节点维护一个等待状态waitStatus。

![CLH](CLH.png)

### 数据结构NODE

```java
   static final class Node {
        /** 表示线程以共享的模式等待锁 */
        static final Node SHARED = new Node();
        /** 表示线程正在以独占的方式等待锁 */
        static final Node EXCLUSIVE = null;

        /** waitStatus=1 说明当前结点(即相应的线程)是因为超时或者中断取消的，进入该状态后将无法恢复 */
        static final int CANCELLED =  1;
        /** waitStatus=-1 表示后继结点在等待当前结点唤醒。后继结点入队时，会将前继结点的状态更新为							* SIGNAL。
        */
        static final int SIGNAL    = -1;
        /** waitStatus=-2 该状态是用于condition队列结点的。明结点在等待队列中，结点线程等待在Condition				 *	上，当其他线程对Condition调用了signal()方法时，会将其加入到同步队列中去。
        */
        static final int CONDITION = -2;
        /**
         * waitStatus=-3 共享模式下，前继结点不仅会唤醒其后继结点，同时也可能会唤醒后继的后继结点(无条件					 * 地向后继结点传播)。
         */
        static final int PROPAGATE = -3;

        /**
         * 等待状态。 =0，初始化的时候的默认值
         *
         */
        volatile int waitStatus;

        /**
         * 前置节点（前驱指针）
         */
        volatile Node prev;

        /**
         * 后置节点（后继指针）
         */
        volatile Node next;

        /**
         * 表示当前节点的线程
         */
        volatile Thread thread;

        /**
         * 用于记录共享模式(SHARED)。也可以用来记录CONDITION队列，指向下一个处于CONDITION状态的节点					 *（Condition Queue队列）
         */
        Node nextWaiter;

        /**
         * 通过nextWaiter的记录值判断当前结点的模式是否为共享模式
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * 返回前驱节点
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

    /**
     * 头节点
     */
    private transient volatile Node head;

    /**
     * 尾节点
     */
    private transient volatile Node tail;
```

![FIFO](FIFO.png)

## 手动实现一个简单 ReentrantLock

```java
public class MySelfLock implements Lock {

    //state 表示获取到锁  state =1获取到了锁，state=0，表示这个锁当前没有线程拿到
    private static class Sync extends AbstractQueuedSynchronizer{

        //是否独占
        protected boolean isHeldExclusively() {
            return getState() == 1;
        }

        //独占式 获取
        protected boolean tryAcquire(int arg) {
            if (compareAndSetState(0,1)){
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        //独占式 释放
        protected boolean tryRelease(int arg) {
            if (getState() ==0){
                throw new UnsupportedOperationException();
            }
            setExclusiveOwnerThread(null);

            setState(0);//不会有多个线程竞争，不需要使用原子性

            return true;
        }

        Condition newCondition(){
            return new ConditionObject();
        }
    }

    private final Sync sync = new Sync();

    @Override //加锁 不成功进入等待队列
    public void lock() {
        sync.acquire(1);
    }

    @Override //加锁 可打断
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

    @Override //尝试加锁（一次）
    public boolean tryLock() {
        return sync.tryAcquire(1);
    }

    @Override //尝试加锁 带超时的
    public boolean tryLock(long time, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireNanos(1,unit.toNanos(time));
    }

    @Override //释放锁
    public void unlock() {
        sync.release(1);
    }

    @Override
    public Condition newCondition() {
        return sync.newCondition();
    }
}
```

```java
public class TestMyLock {

    public void test(){
        final Lock lock = new MySelfLock();//new ReentrantLock();

        class Worker extends Thread{

            public void run(){
                while (true){
                    lock.lock();

                    try{
                        Thread.sleep(1000);

                        System.out.println(Thread.currentThread().getName());

                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } finally {
                        lock.unlock();
                    }

                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }

        for (int i =0;i<10;i++){
            Worker w = new Worker();
            w.setName("线程=="+i);
            w.setDaemon(true);
            w.start();
        }

        for (int i=0;i<10;i++){
            try {
                Thread.sleep(1000);
                System.out.println("*****************************");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }

    public static void main(String[] args) {
        TestMyLock testMyLock = new TestMyLock();
        testMyLock.test();
    }
}
```

```shell
线程==0
*****************************
*****************************
线程==1
*****************************
*****************************
线程==2
*****************************
*****************************
*****************************
线程==3
*****************************
线程==4
*****************************
*****************************

Process finished with exit code 0
```

# ReentrantLock 原理浅析

## 非公平锁实现原理

### 加锁

#### lock()

```java
final void lock() {
  if (compareAndSetState(0, 1))//cas 设置state的值 更新为1	
    setExclusiveOwnerThread(Thread.currentThread());//设置当前线程为锁的持有者
  else
    acquire(1);
}
```

+ 调用 **CAS** 方法去设置 state的值，如果 state 的值等于0，代表锁没有被占用，那么就将state值更新为1，代表该线程获取锁成功，然后执行**setExclusiveOwnerThread** 方法将该线程设置为锁的**持有者**。
+ 如果**CAS** 设置state的值失败，也就是 state 不等于0，此时锁正在被占用着，则要执行 **acquire** 方法

#### acquire()

```java
    public final void acquire(int arg) {
      /*
      *如果 tryAcquire 为true，这个取反就变为false了，就不会执行后面的了。如果tryAcquire 返回false ，取反就为true，会继续执
      *行后面的 acquireQueued
      *
      */
        if (!tryAcquire(arg) && 
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 
            selfInterrupt();
    }
```

首先调用 **tryAcquire** 再次进行尝试

```java
final boolean nonfairTryAcquire(int acquires) {
  final Thread current = Thread.currentThread();
  int c = getState();
  if (c == 0) {//如果 state 的值等于0（之前占有锁的线程刚好已经释放了锁）
   	// 尝试用 cas 获得, 这里体现了非公平性: 不去检查 AQS 队列
    if (compareAndSetState(0, acquires)) { //CAS 将state的值更新为1。
      setExclusiveOwnerThread(current);//设置当前线程为锁的持有者
      return true;
    }
  }
  /*
  * 查看占有锁的线程是不是刚好是自己,如果已经获得了锁, 线程还是当前线程, 表示发生了锁重入
  */
  else if (current == getExclusiveOwnerThread()) {
    int nextc = c + acquires;
    if (nextc < 0) // overflow
      throw new Error("Maximum lock count exceeded");
    setState(nextc);// 更新重入次数
    return true;
  }
  return false;
}
```

+ 调用 getState() 方法获取 state 的值，如果 state 的值等于0（之前占有锁的线程刚好已经释放了锁）

  + 则 CAS 将state的值更新为1。

  + CAS  如果成功，然后执行**setExclusiveOwnerThread** 方法将该线程设置为锁的**持有者**。

  + CAS  如果失败，则直接返回false。

+ 如果 state 的值不等于0

  + 调用 **getExclusiveOwnerThread** 方法查看占有锁的线程是不是刚好是自己。（重入）
  + 如果是，则将 state+1，然后返回true。
  + 如果不是，则直接返回 false。

#### addWaiter()

入队操作

```java
private Node addWaiter(Node mode) {
    // 当前线程构造成Node节点。mode有两种：EXCLUSIVE（独占）和SHARED（共享） 
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {//尾节点不为空
      node.prev = pred; //当前线程节点的前驱节点指向尾节点

      //并发处理 尾节点有可能已经不是之前的节点 所以需要CAS更新
      if (compareAndSetTail(pred, node)) {
        pred.next = node; //CAS更新成功 当前线程为尾节点 原先尾节点的后续节点就是当前节点
        return node;
      }
    }
    //尾节点为空，说明队列还没有初始化，需要初始化
    enq(node);
    return node;
  }
```

如果 pred == null 需要执行 enq() 方法

<font color="red">**队列初始化的时候，创建的头节点是一个哨兵节点，并不会跟任务线程相关联**</font>

```java
private Node enq(final Node node) {
    for (;;) {//自旋操作
      Node t = tail;
      if (t == null) { // Must initialize
        //尾节点为空  第一次入队  设置头尾节点一致 同步队列的初始化。注意此时初始化的头节点并不是吧当前节点设置为头节点，这个头节点可以看成是一个哨兵节点，并不会跟任何线程相关联
        if (compareAndSetHead(new Node()))
          tail = head;
      } else {
         //所有的线程节点在构造完成第一个节点后 依次加入到同步队列中
        node.prev = t;
        if (compareAndSetTail(t, node)) {
          t.next = node;
          return t;
        }
      }
    }
  }
```

![AQS_node1](AQS_node1.png)

#### acquireQueued()

挂起。会在一个死循环中不断尝试获得锁，失败后进入 park 阻塞

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
      boolean interrupted = false; //标记线程是否被中断过
      for (;;) {
        final Node p = node.predecessor(); //获取前驱节点
        if (p == head && tryAcquire(arg)) { //如果前驱节点是头节点，则再次尝试获取锁
          setHead(node); // 获取成功,将当前节点设置为head节点
          p.next = null; // help GC
          failed = false;
          return interrupted; //返回是否被中断过
        }
        /* 判断获取失败后当前线程是否可以挂起,若可以则挂起
        * 注意：
        * shouldParkAfterFailedAcquire 将前驱 node，即 head 的 waitStatus 改为 -1，这次返回 false，返回acquireQueued				*		再次尝试获取锁
        * 再次进入 shouldParkAfterFailedAcquire 时，这时因为其前驱 node 的 waitStatus 已经是 -1，这次返回 true
        */
        if (shouldParkAfterFailedAcquire(p, node) &&
            parkAndCheckInterrupt())
          interrupted = true; //返回是否被中断过
      }
    } finally {
      if (failed)
        cancelAcquire(node);
    }
  }
```

shouldParkAfterFailedAcquire()。主要当前节点的前驱节点状态是否符合要求

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus; //前驱节点的状态
        if (ws == Node.SIGNAL)
            /*
             *  前驱节点状态为signal (-1 表示后继结点在等待当前结点唤醒),返回true
             */
            return true;
        if (ws > 0) {
            /*
             * 从队尾向前寻找第一个状态不为CANCELLED的节点
             * 
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             * 将前驱节点的状态设置为SIGNAL (-1)
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

整个挂起逻辑如下：

进入acquireQueued 逻辑

+ 获取前驱节点，final Node p = node.predecessor();
+ 如果前驱节点是头节点，则再次尝试获取锁，tryAcquire(arg);
+ 尝试获取锁成功,将当前节点设置为head节点
+ 如果前驱节点不是头节点，或者尝试获取锁失败，则进行入队操作。shouldParkAfterFailedAcquire();

进入 shouldParkAfterFailedAcquire 逻辑

+ 将前驱 node，即 head 的 waitStatus 改为 -1(<font color="red">表示后继结点在等待当前结点唤醒</font>)，这次返回 <font color="red">false</font>
+ shouldParkAfterFailedAcquire 执行完毕回到 acquireQueued ，再次 tryAcquire 尝试获取锁，如果这时
  state 仍为 1，则获取锁失败。
+ 再次进入 shouldParkAfterFailedAcquire 时，这时因为其前驱 node 的 waitStatus 已经是 -1，这次返回
  <font color="red">true</font>
+ 进入 parkAndCheckInterrupt，将自己挂起。

### 解锁

```java
public void unlock() {
        sync.release(1);
    }
```

```java
public final boolean release(int arg) {
        if (tryRelease(arg)) {// 尝试释放锁，返回true代表已经全部释放成功
            Node h = head;// 获取头结点
            if (h != null && h.waitStatus != 0)
              	/*
              	* 头结点不为空并且头结点的waitStatus不是初始化节点情况，解除线程挂起状态，唤醒头结点的下个节点关联的线程
              	*/
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

```java
protected final boolean tryRelease(int releases) {
            int c = getState() - releases;// 计算减少可重入次数
            if (Thread.currentThread() != getExclusiveOwnerThread())// 判断当前线程不是持有锁的线程，抛出异常
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {// 锁被重入次数为0,表示释放成功，清空独占线程
                free = true;
                setExclusiveOwnerThread(null);
            }
  					//更新state
            setState(c);
            return free;
        }
```



**唤醒处理**：

```java
private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus; // 获取头结点waitStatus
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next; // 获取当前节点的下一个节点
  			/*
  			* 如果下个节点是null或者下个节点被cancelled，就找到队列最开始的非cancelled的节点
  			*/
        if (s == null || s.waitStatus > 0) {
            s = null;
          /*
          * 从尾部节点开始找，到队首，找到队列第一个waitStatus<0的节点。
          */
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0) //从这里可以看出，<=0的结点，都是还有效的结点
                    s = t;
        }
  			/*
  			* 如果当前节点的下个节点不为空，而且状态<=0，就把下一个节点unpark
  			*/
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

<font color="red">**为什么要从后往前节点？**</font>

主要是因为入队操作不是原子性的。

```java
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
      node.prev = pred; //还未开始执行，此时执行了 unparkSuccessor 方法的的时候
      if (compareAndSetTail(pred, node)) {
        pred.next = node; 
        return node;
      }
    }
    enq(node);
    return node;
  }
```

+ 第一个原因：入队操作不是原子性的，当pred.next = node还没执行，如果这个时候执行了unparkSuccessor方法，就没办法从前往后找了，所以需要从后往前找。
+ 第二个原因：在产生CANCELLED状态节点的时候，先断开的是Next指针，Prev指针并未断开，因此也是必须要从后往前遍历才能够遍历完全部的Node

## 可打断原理

### 不可打断模式

在此模式下，即使它被打断，仍会驻留在 AQS 队列中，一直要等到获得锁后方能得知自己被打断了

```java
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this); // 如果打断标记已经是 true, 则 park 会失效
    return Thread.interrupted(); // interrupted 会清除打断标记
}
```

```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted; //需要获得锁后, 才能返回打断状态
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true; // 如果是因为 interrupt 被唤醒, 返回打断状态为 true
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      selfInterrupt();
}
```

```java
static void selfInterrupt() {
        Thread.currentThread().interrupt(); // 重新产生一次中断
    }
```

### 可打断模式

```java
public void lockInterruptibly() throws InterruptedException {
            sync.acquireInterruptibly(1);
        }
```

```java
public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg)) // 如果没有获得到锁, 执行 doAcquireInterruptibly
            doAcquireInterruptibly(arg);
    }
```

```java
private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                  // 在 park 过程中如果被 interrupt 会进入此
									// 这时候抛出异常, 而不会再次进入 for (;;)
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

## 公平锁实现原理

### 加锁

```java
final void lock() {
    acquire(1);
  }
```

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

<font color = "red">与非公平锁主要区别在于 tryAcquire 方法的实现</font>

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
  	//判断状态state是否等于0，等于0代表锁没有被占用，不等于0则代表锁被占用着。
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //判断当前线程是否为锁的所有者，如果是，那么直接更新状态state，然后返回true。
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
  	//如果同步队列中有线程存在 且 锁的所有者不是当前线程，则返回false。
    return false;
}
```

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail; 
    Node h = head;
    Node s;
  	// h != t 时表示队列中有 Node
    return h != t &&
      	/*
      	* (s = h.next) == null 表示队列中没有第二个节点了 （第一个节点是头节点不与任何线程相关联）
      	* || s.thread != Thread.currentThread() 或者队列中第二个线程不是此线程
      	*/
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

+ 调用hasQueuedPredecessors方法判断同步队列中是否有线程在等待。
+ 如果同步队列中没有线程在等待，则当前线程成为锁的所有者。
+ 如果同步队列中有线程在等待，则继续往下执行。
+ 这个机制就是公平锁的机制，也就是先让先来的线程获取锁，后来的不能抢先获取

### 解锁

解锁与非公平锁一致。



**参考资料**

[从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)