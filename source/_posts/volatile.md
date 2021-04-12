---
title: 并发关键字 volatile 详解
date: 2020-06-07 21:58:04
categories: java 并发
tags:
	- java
	- 并发
---

# 概念

​	增加了实例变量在多个线程之间的可见性，但不具备同步性也就不具备原子性。

​	volatile关键字常被称为轻量级锁 ，其作用与锁的作用有相同的地方：**保证可见性和有序性**。并不保证操作的原子性。

​	被volatile修饰的变量能够保证每个线程能够获取该变量的最新值，从而避免出现数据脏读的现象。

<!--more-->

# 作用

## 防止内存重排

### 指令重排概念

指令重排序是JVM为了优化指令，提高程序运行效率，在不影响单线程程序执行结果的前提下，尽可能地提高并行度。编译器、处理器也遵循这样一个目标。注意是单线程。多线程的情况下指令重排序就会给程序员带来问题。

不同的指令间可能存在数据依赖。比如下面计算圆的面积的语句：

```
double r = 2.3d;//a

double pi =3.1415926; //b

double area = pi* r * r; //c
```

area的计算依赖于r与pi两个变量的赋值指令。而r与pi无依赖关系。

as-if-serial语义是指：不管如何重排序（编译器与处理器为了提高并行度），（单线程）程序的结果不能被改变。这是编译器、Runtime、处理器必须遵守的语义。

虽然，

a - happensbefore -> b,

b - happens before -> c，

但是计算顺序 **a-> b-> c** 与 **b-> a-> c ** 对于r、pi、area变量的结果并无区别。编译器、Runtime在优化时可以根据情况重排序 a 与 b，而丝毫不影响程序的结果。

指令重排序包括编译器重排序和运行时重排序。

### 指令重排带来的问题

首先来看一个例子：单例模式-懒汉式

```java
public class Singleton4 {
	
	private static Singleton4 INSTANCE;
	
	private Singleton4() {};
	
	public void doSomething() {
		
	}
	
	public static Singleton4 getInstance() {
		if (INSTANCE == null) {//此处会存在竞态条件，会导致 INSTANCE 会被多次赋值
			
			INSTANCE = new Singleton4();
		}
		return INSTANCE;
	}

}
```

为了防止这种情况，因此有了DCL（Double Check Lock，双重检查锁）机制，使得大部分请求都不会进入阻塞代码块。

```java
public class Singleton6 {

	private static Singleton6 INSTANCE;
	
	private Singleton6() {};
	
	public void doSomething() {
		
	}
	
	public static Singleton6 getInstance() {
		if (INSTANCE == null) {//当instance不为null时，仍可能指向一个“被部分初始化的对象”
			synchronized (Singleton6.class) {
				if (INSTANCE == null) {
					INSTANCE = new Singleton6();
				}
			}
		}
		return INSTANCE;
	}
}
```

这里会再存在一个问题。当instance不为null时，仍可能指向一个“被部分初始化的对象”。因为

```java
INSTANCE = new Singleton6();
```

此处并不是一个原子操作。有可能会进行指令重排。

实例化一个对象可以分为三个步骤

- 分配内存空间。
- 初始化对象。
- 将内存空间的地址赋值给对应的引用。

但是由于操作系统可以 **对指令进行重排序**，所以上面的过程也可能会变成如下过程：

- 分配内存空间。
- 将内存空间的地址赋值给对应的引用。
- 初始化对象

如果是这个流程，多线程环境下就可能将一个未初始化的对象引用暴露出来，从而导致不可预料的结果。为了解决这个为题，只需要加上volatile

```java
	private static volatile Singleton6 INSTANCE;// volatile 防止内存重排
```

### volatile 如何防止指令重排

volatile关键字通过 **内存屏障** 来防止指令被重排序。

#### 内存屏障

JMM(java 内存模型) 在不改变程序执行结果的前提下，尽可能的支持处理器的重排序。通过禁止特定特定类型的编译器重排序和处理器重排序，为开发者提供一致的内存可见性保证，如 volatile、final。

Java编译器在生成指令的时候会在适当位置插入内存屏障来进制特定类型的处理器排序。

内存屏障说的通俗一点就是一个栏杆，在两个指令之间插入栏杆，后面的指令就不能越过栏杆先执行。

> A memory barrier, also known as a membar, memory fence or fence instruction, is a type of barrier instruction that causes a CPU or compiler to enforce an ordering constraint on memory operations issued before and after the barrier instruction. This typically means that operations issued prior to the barrier are guaranteed to be performed before operations issued after the barrier.

#### 内存屏障分类

**LoadLoad屏障**：

抽象场景：Load1; LoadLoad; Load2

Load1 和 Load2 代表两条读取指令。在Load2要读取的数据被访问前，保证Load1要读取的数据被读取完毕。

**StoreStore屏障：**

抽象场景：Store1; StoreStore; Store2

Store1 和 Store2代表两条写入指令。在Store2写入执行前，保证Store1的写入操作对其它处理器可见

**LoadStore屏障：**

抽象场景：Load1; LoadStore; Store2

在Store2被写入前，保证Load1要读取的数据被读取完毕。

**StoreLoad屏障：**

抽象场景：Store1; StoreLoad; Load2

在Load2读取操作执行前，保证Store1的写入对所有处理器可见。StoreLoad屏障的开销是四种屏障中最大的。

#### volatile 处理

- 在每个volatile写操作的前面插入一个StoreStore屏障。
- 在每个volatile写操作的后面插入一个StoreLoad屏障。
- 在每个volatile读操作的后面插入一个LoadLoad屏障。
- 在每个volatile读操作的后面插入一个LoadStore屏障。

volatile 写是在前面和后面分别插入内存屏障，而 volatile 读操作是在后面插入两个内存屏障。

| 内存屏障        | 说明                                                        |
| --------------- | ----------------------------------------------------------- |
| StoreStore 屏障 | 禁止上面的普通写和下面的 volatile 写重排序。                |
| StoreLoad 屏障  | 防止上面的 volatile 写与下面可能有的 volatile 读/写重排序。 |
| LoadLoad 屏障   | 禁止下面所有的普通读操作和上面的 volatile 读重排序。        |
| LoadStore 屏障  | 禁止下面所有的普通写操作和上面的 volatile 读重排序。        |

![volatile-1](volatile-1.png)

![volatile-2](volatile-2.png)

## 保证内存可见性

### 概念

可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

Java 内存模型规定，对于多个线程共享的变量，存储在主内存当中，每个线程都有自己独立的工作内存，并且线程只能访问自己的工作内存，不可以访问其它线程的工作内存。工作内存中保存了主内存中共享变量的副本，线程要操作这些共享变量，只能通过操作工作内存中的副本来实现，操作完毕之后再同步回到主内存当中，其 JVM 模型大致如下图。

![neicun](neicun.jpeg)

### 原子操作

Java通过8种原子操作完成**工作内存**和**主内存**的交互：

+ lock：作用于主内存，把变量标识为线程独占状态。

+ unlock：作用于主内存，解除独占状态。

+ read：作用主内存，把一个变量的值从主内存传输到线程的工作内存。

+ load：作用于工作内存，把read操作传过来的变量值放入工作内存的变量副本中。

+ use：作用工作内存，把工作内存当中的一个变量值传给执行引擎。

+ assign：作用工作内存，把一个从执行引擎接收到的值赋值给工作内存的变量。

+ store：作用于工作内存的变量，把工作内存的一个变量的值传送到主内存中。

+ write：作用于主内存的变量，把store操作传来的变量的值放入主内存的变量中。

其中lock和unlock定义了一个线程访问一次共享内存的界限。

### volatile 处理

- read、load、use动作必须连续出现。
- assign、store、write动作必须连续出现。

所以，使用volatile变量能够保证:

- 每次读取前必须先从主内存刷新最新的值。
- 每次写入后必须立即同步回主内存当中。

# 使用场景

+ 只有一个线程写，多个线程读
+ 对变量的写入操作不依赖变量的当前值，或者你能确保只有单个线程更新变量的值。
+ 该变量没有包含在具有其他变量的不变式中。

