---
title: "CPU 伪共享问题"
date: 2018-05-19
draft: false
tags:
- arch
categories:
- programming
---

不知是谁最初把 false sharing 翻译成“伪共享”，翻译得莫名其妙，不知所谓。实际意思应该是“错误的共享”。

最近在读的几本书都提到了 CPU 伪共享问题。这里尝试做一下总结。这几本书是：

- 《架构解密：从分布式到微服务》，leader-us 著
- 《java并发编程的艺术》, 方腾飞、魏鹏、程晓明 著
- 《实战Java高并发程序设计》，葛一鸣、郭超 著

其中，《架构解密》一书中的讲解比较详细。

## CPU 伪共享

当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。伪共享是上层编程层面的问题，底层对应的是 CPU Cache 一致性问题。

## CPU 通过缓存操作内存

CPU 和内存的处理速度差距在 **100 倍**左右，因此需要高速缓存。CPU 的高速缓存采用SRAM (Static Random-Access Memory, 静态随机存取存储器), 频率与 CPU 基本一致，速度快，造价高；而内存使用 DRAM (Dynamic Random-Access Memory), DRAM 直到 DDR4 才接近 CPU 的速度。

CPU 缓存一般采用多级缓存机制。如 Intel 的一款 CPU：

- L1 Cache 容量为 64 KB, 访问速度 1ns；
- L2 Cache 容量扩大 4 倍，256 KB，访问速度 3ns 左右；
- L3 Cache 容量扩大 512 倍，达到 32 MB，访问速度 12 ns 左右

CPU 要访问 DRAM 的数据，要经过 L3，L2，L1 才能最终到达 CPU.

## Cache 一致性问题和 MESI 协议

每个 CPU 都有自己的 Cache；多个 CPU 核心的 Cache 应该是一致的。来自 Intel 的 MESI 协议是业界公认的 Cache 一致性问题最佳解决方案。

I/O 操作从来不以字节为单位，而是以块为单位。CPU 对 DRAM 的读取也是类似 I/O 块的方式。最小单元为 Cache Line. Cache 从内存中加载数据时，一次加载一条 Line.

Cache Line 的头部有两个 bit 记录自身状态：

- M (Modified): 修改状态，其他 CPU 无数据副本，在本 CPU 上被修改过，必然引发系统总线的写指令，将 Cache Line 中的数据写回内存中。
- E (Exclusive): 独占状态，当期 Cache Line 中的数据与内存中一致，并且其他 CPU 无副本
- S (Shared): 共享状态，与内存一致，至少在其他某个 CPU 有副本
- I (Invalid): 无效状态，当前 Cache Line 无数据或数据已经失效；加载数据时优先选择 I 状态的 Cache Line

一些细节：

(1) 同时读取。CPU A 读取 -- Invalid 状态；加载后 -- Exclusive 状态。此时 CPU B 的读取请求会被 A 嗅探到，CPU A 在总线上复制一份作为应答，并将自身 Cache Line 改为 Shared 状态。CPU B 收到后也改为 Shared.

(2) A 写 B 读。CPU A 写入内存前 -- Modified，此时其他 CPU 如果有读请求，则 A 嗅探到后先将自己的缓存写入内存，然后复制给其他 CPU，状态改为 Shared.

(3) A 写 B 写。A 的 Modified 写入内存前， B 也发起写请求，则 A 优先写入，完成后设置为 Invalid. 然后导致 B 请求无响应，B 只能再次请求。

(4) 如果多个 Shared 状态，有一个 CPU 进行写操作，会导致其他都变为 Invalid.

总结：存在多个处理器时，对共享变量的操作会涉及多个 CPU 之间的协调问题及 Cache 失效问题。

## 伪共享问题的解决

Java 对象是在堆内存上分配空间的，成员变量是在内存空间上是连续的。它们完全有可能被加载到同一个 Cache Line 中。如果有两个线程运行在不同的 CPU 上，它们分别修改不同的成员变量，则会造成一个线程对变量的修改导致另一个线程所在的 CPU 的 Cache Line 失效，被迫重新加载 —— 这仅仅是因为不同变量处于同一个 Cache Line.

解决思路很简单，将变量分到不同的 Cache Line.

一种方式是用一些无用的字段填充。这种方式很可能会被 java 7 的编译器优化掉。

JDK 8 的解决方案是用 @Contended 确保一个 Object 或 Class 的某个属性与其他属性
不在一个 Cache Line 中。

下面的 VolatileLong 的多个实例之间就不会产生 Cache 伪共享问题。

```java
@Contended
class VolatileLong {
    public volatile long value = 0L;
}
```

由于填充字节的方案操作较为复杂，推荐使用 Java 8 的 @Contended 注解。**务必记得要加 jvm 参数 `-XX:-RestrictContended`**.


## 实验

这个例子来源于 [http://ifeve.com/falsesharing/](http://ifeve.com/falsesharing/) 但做了修改。



```java
import sun.misc.Contended;

public class FalseSharing {


    public static void main(final String[] args) throws Exception {
        long start = System.nanoTime();

        final int CPU_NUM = 4;

        final VolatileLong number1 = new VolatileLong();
        final VolatileLong number2 = new VolatileLong();
        final VolatileLong number3 = new VolatileLong();
        final VolatileLong number4 = new VolatileLong();

        long times = 500L * 1000L * 1000L;

        Thread[] threads = new Thread[]{
                new Thread(new MyRunnable(number1, times)),
                new Thread(new MyRunnable(number2, times)),
                new Thread(new MyRunnable(number3, times)),
                new Thread(new MyRunnable(number4, times))
        };

        for (Thread t : threads) {
            t.start();
        }

        for (Thread t : threads) {
            t.join();
        }

        System.out.println("duration = " + (System.nanoTime() - start));
    }

}


class VolatileLong {
    volatile long value;
    public long p1, p2, p3, p4, p5; // comment out, 原文中到 p6
}

class MyRunnable implements Runnable {
    final long times;
    final VolatileLong data;

    MyRunnable(VolatileLong data, long times) {
        this.data = data;
        this.times = times;
    }

    public void run() {
        long i = times + 1;
        while (0 != --i) {
            data.value = i;
        }
    }
}
```


缓存行一般为 64 字节。

```java
class VolatileLong {
    volatile long value;
    public long p1, p2, p3, p4, p5; // comment out
}

```

这段代码中，p1 到 p5 占用 40 个字节，value 占用 8 个字节；**对象头在 64 位 jvm 下占用 16 个字节**，共 64 字节。但[原文](https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html)中填充到 p6, 引发了一些讨论。还有一篇博文继续深入讨论了 false-sharing 问题：[http://mechanical-sympathy.blogspot.co.uk/2011/08/false-sharing-java-7.html](http://mechanical-sympathy.blogspot.co.uk/2011/08/false-sharing-java-7.html)

如果使用 @Contended, 务必记得要加 jvm 参数 `-XX:-RestrictContended`.

## 参考资料

- [https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html](https://mechanical-sympathy.blogspot.com/2011/07/false-sharing.html)
- [http://mechanical-sympathy.blogspot.co.uk/2011/08/false-sharing-java-7.html](http://mechanical-sympathy.blogspot.co.uk/2011/08/false-sharing-java-7.html)
- [http://www.cnblogs.com/Binhua-Liu/p/5620339.html](http://www.cnblogs.com/Binhua-Liu/p/5620339.html)
- [https://www.jianshu.com/p/a9b1d32403ea](https://www.jianshu.com/p/a9b1d32403ea)
- [http://ifeve.com/falsesharing/](http://ifeve.com/falsesharing/)