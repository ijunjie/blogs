---
date: 2018-06-03
title: "浅谈 JMM 和 volatile"
draft: false
tags:
  - JMM
  - volatile
categories:
  - arch
comment: true
---

并发世界有两大阵营，共享内存和消息通信。前者以 JMM 为代表，后者以 Actor 模型为代表。本篇将整理总结 JMM 相关知识并全面梳理 volatile；关于 Actor 模型，后续会探讨。

## JMM

Java 从 1.5 开始采用新的 JSR-133 JMM. JSR-133 主要修复了一些 bug, 增强了 volatile 和 final 等关键字的内存语义。

JMM 主要有以下内容：

- 逻辑划分：Main Memory 和 Local Memory
- 三大特性：Atomicity, Visibility, Ordering
- happens-bofore 原则

### Main Memory & Local Memory

Java 的并发是通过共享内存实现的，确切地说这里的内存就是主内存，每个线程有自己的工作内存(Local Memory)，**Local Memory 是逻辑上的概念，并不是真实存在的，对应于 CPU 高速缓存或者寄存器，保存了该线程使用的变量的主内存副本拷贝**。CPU 高速缓存之间的一致性的问题是通过 MESI 协议解决的，另外还有 cpu false-sharing 问题，请参考笔者另一篇博文: [CPU 伪共享问题](https://ijunjie.github.io/post/arch/cpu-false-sharing/)。

线程只能直接操作工作内存中的变量，不同线程之间的变量值传递需要通过主内存来完成。

Java 内存模型定义了 8 个操作来完成主内存和工作内存的交互操作。

- read：把一个变量的值从主内存传输到工作内存中
- load：在 read 之后执行，把 read 得到的值放入工作内存的变量副本中
- use：把工作内存中一个变量的值传递给执行引擎
- assign：把一个从执行引擎接收到的值赋给工作内存的变量
- store：把工作内存的一个变量的值传送到主内存中
- write：在 store 之后执行，把 store 得到的值放入主内存的变量中
- lock：作用于主内存的变量
- unlock

### 三大特性：Atomicity, Visibility, Ordering

**Atomicity**

Java 内存模型保证了 read、load、use、assign、store、write、lock 和 unlock 操作具有原子性，例如对一个 int 类型的变量执行 assign 赋值操作，这个操作就是原子性的。但是 Java 内存模型允许虚拟机将没有被 volatile 修饰的 64 位数据（long，double）的读写操作划分为两次 32 位的操作来进行，即 load、store、read 和 write 操作可以不具备原子性。

可以使用原子类和 synchronized 来保证原子性。

**Visibility**

可见性指当一个线程修改了共享变量的值，其它线程能够立即得知这个修改。Java 内存模型是通过在变量修改后将新值同步回主内存，在变量读取前从主内存刷新变量值来实现可见性的。

- volatile 可保证可见性。关于 volatile 后文会详述。
- synchronized 也能够保证可见性，对一个变量执行 unlock 操作之前，必须把变量值同步回主内存。
- final 关键字也能保证可见性：被 final 关键字修饰的字段在构造器中一旦初始化完成，并且没有发生 this 逃逸（其它线程可以通过 this 引用访问到初始化了一半的对象），那么其它线程就能看见 final 字段的值。

**Ordering**

有序性是指：在本线程内观察，所有操作都是有序的。在一个线程观察另一个线程，所有操作都是无序的，无序是因为发生了指令重排序。

在 Java 内存模型中，允许编译器和处理器对指令进行重排序，重排序过程不会影响到单线程程序的执行，却会影响到多线程并发执行的正确性。

**volatile 关键字通过添加内存屏障的方式来禁止指令重排**，即重排序时不能把后面的指令放到内存屏障之前。

也可以通过 synchronized 来保证有序性，它保证每个时刻只有一个线程执行同步代码，相当于是让线程顺序执行同步代码。

### happens-before 原则

一般认为这个原则是针对 JMM 的有序性，但也可以理解为针对可见性。因为可见性和有序性是紧密相连的。

happens-before 原则可以看作是 JVM 规范对开发者的承诺，用于增强开发者编写并发程序的信心。凡是不符合 happens-before 原则的，都有可能发生指令重排。

共有 8 条原则，为了便于理解，笔者将其分为两组，线程和锁相关占了 5 条，剩余为对象终结原则、传递性和 volatile 等几个原则。

第一组：

- Single Thread rule，在一个线程内，在程序前面的操作先行发生于后面的操作。
- Thread Start Rule，Thread 对象的 start() 方法调用先行发生于此线程的每一个动作。
- Thread Join Rule，join() 方法返回先行发生于 Thread 对象的结束。
- Thread Interruption Rule，对线程 interrupt() 方法的调用先行发生于被中断线程的代码检测到中断事件的发生，可以通过 Thread.interrupted() 方法检测到是否有中断发生。
- Monitor Lock Rule，一个 unlock 操作先行发生于后面对同一个锁的 lock 操作。

第二组：

- Finalizer Rule，一个对象的初始化完成（构造函数执行结束）先行发生于它的 finalize() 方法的开始。
- Transitivity, 如果操作 A 先行发生于操作 B，操作 B 先行发生于操作 C，那么操作 A 先行发生于操作 C。
- Volatile Variable Rule，对一个 volatile 变量的写操作先行发生于后面对这个变量的读操作。


## volatile

关于 volatile 有如下内容：

- volatile 的两个意义：可见性和有序性
- volatile 底层是通过内存屏障实现的
- 应用场景：状态判断和 DCL
- volatile 的使用优化：cpu false-sharing 问题，字节填充解决方案等
- JSR-133 对 volatile 的语义增强
- volatile 与锁

以上几个方面应该是比较全面的关于 volatile 的知识了。下面是每一点的简要讲解。

### volatile 的意义

上文已提到 JMM 有三个特性：原子性、可见性、有序性。volatile 首先可以保证可见性。一个线程对变量的修改会回写到主存中，被另一个线程看到。其次，volatile 可以保证有序性，禁止指令重排，屏蔽 JVM 的优化。

### volatile 是通过内存屏障实现的

volatile 关键字通过添加内存屏障的方式来禁止指令重排，即重排序时不能把后面的指令放到内存屏障之前。

### 应用场景

主要用于状态判断和 DCL.

DCL 之所以用 volatile 实际与单例模式没有实质关系。如果没有 volatile 且单例对象还有其他属性，则初始化的可见性得不到保证。一个线程对单例的初始化可能不会被另一个线程看到。

### volatile 使用优化

使用 volatile 时要注意 cpu false-sharing 问题。多个 CPU 核心的高速缓存之间通过 MESI 协议保证一致性。高速缓存以行为单位。当多个不同线程分别独立操作不同变量时，这些不同变量可能位于同一个高速缓存行中，一个变量的修改导致一个缓存行失效，影响另一个变量。这就是 cpu false-sharing 问题。这个问题是编程中真实存在的问题。

解决方法有两个：一是通过填充字节确保 volatile 变量占用整个缓存行；二是使用 Java 8 的 Contended 注解，不要忘记使用 JVM 参数配合之。

### JSR-133 对 volatile 的增强

旧的模型允许 volatile 和普通变量重排序，JSR-133 做了限制。

### volatile 与锁

功能上，锁比 volatile 更强大；可伸缩性和执行性能上，volatile 更有优势。
