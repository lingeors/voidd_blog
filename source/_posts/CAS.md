---
title: 高效并发策略：CAS
date: 2017-05-09 14:53:43
tags:
    - 并发
    - 并发策略
    - Java
    - CAS
---
> 此文章主要介绍并发策略：CAS。读者需具备并发的基本知识

<!-- more -->

## 基础
> 这里只是简单介绍一些概念，其它基础读者需具备或自行查阅

**线程同步**：在编写多线程程序时，最基本也是最关键的一点是要保证线程是安全的，编写线程安全的类有多种方式，一般情况会使用加锁的方式进行线程同步，Java 中的内置锁、重入锁都是基于这个目，当然还有更加复杂的读写锁等。一般的锁机制最重要的特点是一次只允许一个线程进入临界区，其他线程被阻塞，导致在高并发环境下性能较差

**阻塞**：通过加锁的方式实现线程同步，往往会使获取不到锁的线程被阻塞，而阻塞的线程唯一能做的事情就是等待（或被中断）

**原语**：指由若干条指令组成、用来实现某个特定操作的一个过程。原语的执行具有原子性。原语常驻内存，且在系统态下执行。

## CAS 简介
CAS 全名 *Compare-and-swap*。它是用来在多线程中实现同步机制的**原子指令**。是 CPU 的硬件同步原语。大部分的现代处理器都支持 CAS 指令

整个 **CAS 过程**如下：CAS 算法会先读取内存位置 V，并保存 V 的值 A，然后基于 A 计算得到 B，接着尝试更新 B 到 V 位置，更新时会先比较 A 和 V 位置的值是否相同，如果相同则将 B 写入 V 位置。如果不相同则表示该位置的值被其他线程修改过，此时更新失败，同时 CAS 会将该结果返回给调用者（通过 boolean 值或返回此时 V 位置的值），然后由调用者决定后续策略（是重试还是回退或直接放弃更新）

可以看到，由于 CAS 的原子性，整个操作过程是无需加锁的，即 CAS 实现并发是无锁、无阻塞的。（感觉有点类似 MySQL 数据库 InnoDB 存储引擎采用的 MVCC 策略，尽管后者是在应用层面实现的）

下面我们通过 Java 代码来模拟这个过程（注意只是模拟语义，CAS 的实现和 Java 无关）
```java
package top.voidd.cas;

import net.jcip.annotations.GuardedBy;
import net.jcip.annotations.ThreadSafe;

/**
 * Created by coder on 5/9/2017.
 * 模拟 CAS 语义。两种结果通知策略
 * from 《Java并发编程实战》
 */
@ThreadSafe
public class SimulatedCAS {
    @GuardedBy("this") private int value;

    public synchronized int get() {
        return value;
    }

    public synchronized int compareAndSwap(int expectedValue, int newValue) {
        int oldValue = value;
        if (oldValue == expectedValue)
            value = newValue;
        return oldValue;
    }

    public synchronized boolean compareAndSet(int expectedValue, int newValue) {
        return (expectedValue == compareAndSwap(expectedValue, newValue));
    }
}
```
上述程序通过加锁机制模拟了 CAS 的过程，并通过两个 `compare` 方法模拟结果通知的两个不同策略。应该说整个过程是比较清晰的。

## CAS 优劣

**优势**：不难看出，通过 CAS，线程间不存在阻塞操作，失败后的处理更加灵活。另外，对 Java 而言，JVM 处理锁操作过程很复杂，可能会导致操作系统级的锁定、线程挂起以及上下文切换，开销很大。而在内部执行 CAS 是不需要执行 JVM 代码、系统调用或线程调度，开销很小。

**缺点**：使用 CAS 时需要自行处理竞争问题。

## Java 中的使用
1. JDK 5.0 中引入了对 CAS 底层的支持，JVM 将它们编译为底层硬件提供的方法
2. `java.util.concurrent.atomit` 中的相关类大量使用了 CAS 操作。（通过源码可以看到类 `sum.misc.Unsafe`，它提供了相关的 CAS 操作，google一下，好像挺有意思）
3. `java.util.concurrent` 中的大多数类在实现时直接或间接的使用了上述的原子变量类。这也是如 `ConcurrentBlockingQueue` 类比线程安全的 `BlockingQueue` 高效的原因

## CAS 使用
CAS 拥有很好的性能，但因为其复杂性，使用成本较高，要使用好则更加困难。因此在应用层我们更多时候是使用一些封装比较完善的库或相关扩展，比如上述提到的 JDK 对 CAS 的支持，以及，后面我们会介绍的 一个高性能的线程间通信的库 [Disruptor][3]

## 总结
1. 必须记住的是: CAS 是系统原语，是原子性的，操作系统支持的
2. CAS 作为一种并发策略，可以实现线程安全，但同时是无锁、非阻塞的类。性能很好
3. JVM 对 CAS 做了底层封装，`java.util.concurrent` 中的多数类依靠 JVM 提供的高效 CAS 操作实现的。

## 参考
[Compare-and-swap -- wikipedia][1]
[Java并发编程实战][2]

[1]: https://en.wikipedia.org/wiki/Compare-and-swap
[2]: https://book.douban.com/subject/10484692/
[3]: https://mvnrepository.com/artifact/com.lmax/disruptor/3.3.6