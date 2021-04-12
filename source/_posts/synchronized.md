---
title: 并发关键字 synchronized 详解
date: 2020-05-29 23:45:06
categories: java 并发
tags:
	- java
	- 并发
---

# 特性

**原子性**

原子性是指一个操作是不可中断的，要全部执行完成，要不就都不执行。

<!-- more -->

**可见性**

可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

volatile 也具备此特性

**有序性**

有序性是指程序执行的顺序按照代码的先后顺序执行。

volatile 也具备此特性

**可重入性**

可重入性是指同一个线程外层函数获取到锁之后，内层函数可以直接使用该锁

# 使用方式

+ 修饰实例方法（当前实例加锁）

+ 修饰静态方法（类对象加锁）

  >注意：如果一个线程A调用一个实例对象的非静态 synchronized方法，而线程B需要调用这个实例对象所属类的静态 synchronized方法，是**允许**的，不会发生互斥现象，因为访问**静态 synchronized 方法占用的锁是当前类的class对象，而访问非静态 synchronized 方法占用的锁是当前实例对象锁**。

+ 修饰代码块（指定一个加锁对象，对所指定的对象加锁）

**synchronized不能被继承，不能使用Synchronized关键字修饰接口方法；构造方法也不能用Synchronized。**

# synchronized 底层实现原理

## sync 方法和代码块的底层实现原理

先看简单demo：

```java
public class SynchronizedDemo {

    public synchronized void syncMethod() {
        System.out.println("do something...");
    }

    public void syncBlock() {
        synchronized (this) {
            System.out.println("do something...");
        }
    }
}
```

使用 javap -v SynchronizedDemo.class 反编译得到字节码如下：

```java
{
  public SynchronizedDemo();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 6: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LSynchronizedDemo;

  //修饰实例方法
  public synchronized void syncMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String do something...
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 9: 0
        line 10: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   LSynchronizedDemo;

  //修饰代码块方法
  public void syncBlock();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter //进入同步方法 将this对象的 MarkWord 置为 Monitor 指针
         4: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         7: ldc           #3                  // String do something...
         9: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
        12: aload_1
        13: monitorexit //退出同步方法 将this对象 MarkWord 重置，唤醒 EntryList
        14: goto          22
        17: astore_2
        18: aload_1
        19: monitorexit //退出同步方法 将this对象 MarkWord 重置，唤醒 EntryList
        20: aload_2
        21: athrow
        22: return
      Exception table://异常检测，如果发现14-17中有异常出现，就从17行处继续执行，这也就是为什么会有两个monitorexit的原因
         from    to  target type
             4    14    17   any
            17    20    17   any
      LineNumberTable:
        line 13: 0
        line 14: 4
        line 15: 12
        line 16: 22
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      23     0  this   LSynchronizedDemo;
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 17
          locals = [ class SynchronizedDemo, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4
}
```

从字节码可以看出

1、同步方法是利用 **ACC_SYNCHRONIZED** 这个修饰符来实现的

>同步方法是通过隐式的方式来实现的。用ACC_SYNCHRONIZED来标识。
>
>意思就是：当方法调用时，调用指令将会去检查这个方法的 ACC_SYNCHRONIZED 访问标志是否被设置，如果设置了，执行线程将先获取monitor，获取成功之后才能执行方法体，方法执行完后再释放monitor（注意这个完成包括非正常的完成比如异常）。在方法执行期间，其他的线程都将无法再获得同一个monitor对象。

2、同步代码块是利用 **monitorenter** 和 **monitorexit** 这2个指令来实现的

> monitorenter指令指向同步代码块的开始位置，monitorexit指令指向同步代码块的结束位置。
>
> monitorenter：
>
> 每个对象有一个监视器锁（monitor）。当monitor被占用时就会处于锁定状态，线程执行monitorenter指令时尝试获取monitor的所有权，过程如下：
>
> 1、如果monitor的进入数为0，则该线程进入monitor，然后将进入数设置为1，该线程即为monitor的所有者。
>
> 2、如果线程已经占有该monitor，只是重新进入，则进入monitor的进入数加1.
>
> 3.如果其他线程已经占用了monitor，则该线程进入阻塞状态，直到monitor的进入数为0，再重新尝试获取monitor的所有权。
>
> monitorexit：
>
> 执行monitorexit的线程必须是objectref所对应的monitor的所有者。
>
> 指令执行时，monitor的进入数减1，如果减1后进入数为0，那线程退出monitor，不再是这个monitor的所有者。其他被这个monitor阻塞的线程可以尝试去获取这个 monitor 的所有权。 

3、同步代码块的字节码可发现 有两个 monitorexit，为何会有两个？

>主要是防止在同步代码块中线程因异常退出，而锁没有得到释放，这必然会造成死锁（等待的线程永远获取不到锁）。因此最后一个monitorexit是保证在异常情况下，锁也可以得到释放，避免死锁。

## java对象头

synchronized使用的锁对象是存储在Java对象头里的。如果对象是数组类型，则虚拟机用3个Word（字宽）存储对象头，如果对象是非数组类型，则用2字宽存储对象头。在32位虚拟机中，一字宽等于四字节，即32bit。

| 长度     | 内容                   | 说明                             |
| -------- | ---------------------- | -------------------------------- |
| 32/64bit | Mark Word              | 存储对象的hashCode或锁信息等。   |
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针         |
| 32/64bit | Array length           | 数组的长度（如果当前对象是数组） |

Java对象头里的Mark Word里默认存储对象的HashCode，分代年龄和锁标记位。32位JVM的Mark Word的默认存储结构如下：

|          | 25 bit         | 4bit         | 1bit是否是偏向锁 | 2bit锁标志位 |
| -------- | -------------- | ------------ | ---------------- | ------------ |
| 无锁状态 | 对象的hashCode | 对象分代年龄 | 0                | 01           |

在运行期间Mark Word里存储的数据会随着锁标志位的变化而变化。Mark Word可能变化为存储以下4种数据： 

| 锁状态   | 25 bit                       | 4bit         | 1bit         | 2bit |      |
| -------- | ---------------------------- | ------------ | ------------ | ---- | ---- |
| 23bit    | 2bit                         | 是否是偏向锁 | 锁标志位     |      |      |
| 轻量级锁 | 指向栈中锁记录的指针         | 00           |              |      |      |
| 重量级锁 | 指向互斥量（重量级锁）的指针 | 10           |              |      |      |
| GC标记   | 空                           | 11           |              |      |      |
| 偏向锁   | 线程ID                       | Epoch        | 对象分代年龄 | 1    | 01   |

上表里面的GC标记，为11的话，推断应该是准备GC的意思。

在64位虚拟机下，Mark Word是64bit大小的，其存储结构如下： 

| 锁状态 | 25bit                       | 31bit    | 1bit     | 4bit   | 1bit     | 2bit |
| ------ | --------------------------- | -------- | -------- | ------ | -------- | ---- |
|        |                             | cms_free | 分代年龄 | 偏向锁 | 锁标志位 |      |
| 无锁   | unused                      | hashCode |          |        | 0        | 01   |
| 偏向锁 | ThreadID(54bit) Epoch(2bit) |          |          | 1      | 01       |      |

> 以上java对象头的表格概括引用：
>
> https://www.cnblogs.com/charlesblc/p/5994162.html

上面看到的**重量级锁也就是通常说synchronized的对象锁**，锁标识位为**10**，其中指针指向的是**monitor对象的起始地址**。每个对象都存在着一个 monitor 与之关联，对象与其 monitor 之间的关系有存在多种实现方式，如**monitor可以与对象一起创建销毁或当线程试图获取对象锁时自动生成**，但当一个 monitor 被某个线程持有后，它便处于锁定状态。

在Java虚拟机(HotSpot)中，**monitor**是由ObjectMonitor实现的

```shell
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数 记录该线程获取锁的次数
    _waiters      = 0, //等待线程数
    _recursions   = 0; //锁的冲入次数
    _object       = NULL;
    _owner        = NULL;  //_owner指向持有ObjectMonitor对象的线程
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ; //监视器前一个拥有者线程的ID
  }
```

当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSet集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)

> 此解释引用：https://blog.csdn.net/javazejian/article/details/72828483



## 为什么Java的任意对象都可以作为锁？

在Java对象头中，存在一个monitor对象，每个对象自创建之后在对象头中就含有monitor对象，monitor是线程私有的，不同的对象monitor自然也是不同的，因此对象作为锁的本质是对象头中的monitor对象作为了锁。这便是为什么Java的任意对象都可以作为锁的原因。



## Monitor的机制

![image-20200528153829565](image-20200528153829565.png)

当一个线程需要获取 Object 的锁时，会被放入 **EntrySet** 中进行等待，如果该线程获取到了锁，成为当前锁的 **owner**。如果根据程序逻辑，一个已经获得了锁的线程缺少某些外部条件，而无法继续进行下去（例如生产者发现队列已满或者消费者发现队列为空），那么该线程可以通过调用 wait 方法将锁释放，进入 **wait set** 中阻塞进行等待，其它线程在这个时候有机会获得锁，去干其它的事情，从而使得之前不成立的外部条件成立，这样先前被阻塞的线程就可以重新进入 **EntrySet** 去竞争锁。这个外部条件在 **monitor** 机制中称为条件变量。

**普通对象没有 Monitor**

# Jvm优化

Java 的锁分为两种：

1、一种是内部锁，它用 **Synchronized** 关键字来修饰，由 JVM 负责管理，并且不会出现锁泄漏的情况。

2、另外一种是显示锁。

​	内部锁的优化方式由 Java 内部机制完成，虽然不需要程序员直接参与，但了解它对会更容易让我们理解多线程。

​	JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。 

　**锁主要存在四中状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。**他们会随着竞争的激烈而逐渐升级。但是锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

## 锁膨胀

**锁有四种状态，并且会因实际情况进行膨胀升级，其膨胀方向是：无锁—>偏向锁—>轻量级锁—>重量级锁，并且膨胀方向不可逆。**

### 偏向锁

在大多数情况下，锁不存在多线程竞争，总是由同一线程多次获得，那么此时就是偏向锁。

1、首先检测是否为可偏向状态（锁标识是否设置成1，锁标志位是否为01）.
2、如果处于可偏向状态，测试Mark Word中的线程ID是否指向自己，如果是，不需要再次获取锁，直接执行同步代码。
3、如果线程Id，不是自己的线程Id，通过CAS获取锁，获取成功表明当前偏向锁不存在竞争，获取失败，则说明当前偏向锁存在锁竞争，偏向锁膨胀为 **轻量级锁**。

![image-20200529173233623](image-20200529173233623.png)

### 轻量级锁

轻量级锁是由偏向锁升级而来，当存在第二个线程申请同一个锁对象时，偏向锁就会立即升级为轻量级锁。注意这里的第二个线程只是申请锁，不存在两个线程同时竞争锁，可以是一前一后地交替执行同步块。

**偏向锁考虑的是不存在多个线程竞争同一把锁，而轻量级锁考虑的是，多个线程不会在同一时刻来竞争同一把锁。**

加锁：

1、首先会先去检查Mark Word中锁标记是否为01。如果是JVM虚拟机就会在该线程的栈帧中创建一个Lock Record（锁记录）内存空间，用于存储Mark Word的拷贝。

2、将Mark Word拷贝到锁记录中。官方称为Displaced Mark Word。

3、尝试将锁对象中的标记字段替换成一个指针，该指针指向该线程的锁记录空间，并将锁记录中owner指针指向当前线程的对象头。如果更新成功执行步骤4，否则执行步骤5.

4、更新成功表明该线程拥有了该对象锁，并且锁记录变为00

5、更新失败，先检查Mark Word中的指针是否指向当前线程的栈帧，如果是，表明线程已经获取了该对象锁。否则表明，存在多个线程竞争，这时膨胀为重量级锁，锁记录变为10，后面等待的线程进入到阻塞状态，并且当前线程自旋尝试获取锁。

解锁：

1、通过CAS操作尝试将线程栈帧中的锁记录替换掉堆中Mark Word

2、成功，锁释放成功

3、失败，表明此时锁已经升级为重量级锁，在释放锁的同时需要唤醒被挂起的线程。

![image-20200529225908860](image-20200529225908860.png)

### 重量级锁

**重量级锁描述同一时刻有多个线程竞争同一把锁。**

重量级锁是由轻量级锁升级而来，当同一时间有多个线程竞争锁时，锁就会被升级成重量级锁，此时其申请锁带来的开销也就变大。

重量级锁一般使用场景会在追求吞吐量，同步块或者同步方法执行时间较长的场景。

## 锁消除

在JIT编译时(可以简单理解为当某段代码即将第一次被执行时进行编译，又称即时编译)，对运行上下文进行扫描，去除不可能存在竞争的锁  举例：在一个方法中存在被StringBuffer修饰的成员变量，由于该变量只会在该方法中使用，不能被其他线程锁共享，所以JVM会自动消除其内部的锁。

## 锁粗化

锁粗化是虚拟机对另一种极端情况的优化处理，通过扩大锁的范围，避免反复加锁和释放锁。也就是将多次连接在一起的加锁、解锁操作合并为一次，将多个连续的锁扩展成一个范围更大的锁

如：

```java
public class StringBufferTest {
    StringBuffer stringBuffer = new StringBuffer();
 
    public void append(){
        stringBuffer.append("a");
        stringBuffer.append("b");
        stringBuffer.append("c");
        stringBuffer.append("d");
        stringBuffer.append("e");
        stringBuffer.append("f");
        stringBuffer.append("g");
        stringBuffer.append("h");
        stringBuffer.append("i");
    }

```

每次调用stringBuffer.append方法都需要加锁和解锁，如果虚拟机检测到有一系列连串的对同一个对象加锁和解锁操作，就会将其合并成一次范围更大的加锁和解锁操作，即在第一次append方法时进行加锁，最后一次append方法结束后进行解锁。

## 自旋锁与自适应自旋锁

### 自旋锁

轻量级锁失败后，虚拟机为了避免线程真实地在操作系统层面挂起，还会进行一项称为自旋锁的优化手段。

**所谓自旋，不是获取不到锁就会阻塞，而是在原地等待一会儿，再次尝试（次数或时长有限），他是以牺牲CPU为代价来换取内核状态切换带来的开销。**

缺点：若线程占用锁时间过长，导致CPU资源白白浪费。

解决方式：当尝试次数达到每个值得时候，线程挂起。

### 自适应自旋锁

这种相当于是对 **自旋锁** 优化方式的进一步优化，它的自旋的次数不再固定，其自旋的次数由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定，这就解决了自旋锁带来的缺点。

## 总结：

> 图片引用：https://jacobchang.cn/media/v2-9db4211af1be81785f6cc51a58ae6054_r.jpg

![v2-9db4211af1be81785f6cc51a58ae6054_r](v2-9db4211af1be81785f6cc51a58ae6054_r-1590764485134.jpg)

