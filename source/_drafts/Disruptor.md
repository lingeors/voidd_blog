---
title: Disruptor 简介
date: 2017-05-09 12:58:33
tags:
    - Disruptor
    - Java
    - 消费者-生产者模式
    - 并发
---
> 此文章主要介绍 Disruptor 框架的基本使用，读者需具备 Java 并发的基本知识

<!-- more -->

## 介绍
`Disruptor` 是 `LMAX` 公司开发的一个高性能的线程间通信的库。具有并发、高性能、无阻塞等特点。具体见 [`disruptor-wiki`][1]。现在我们主要用它来实现生产者-消费者模式。

## 安装
下载或使用 `maven` 安装 [`disruptor jar`][2]，添加到  `classpath` 路径下即可。本人使用的是最新版本 `3.3.6`

## 快速使用
> 这里说一句题外话，面对一个新东西，本人倾向于让它快速运行，再去探讨具体细节和原理（当然一些基本的前置知识是必须要有的），下面也将采用这样的模式介绍。

1. 首先是整个编码过程的步骤，让我们心里有底
    - 创建待消费数据结构，它将被生产者生产，被消费者消费
    - 创建上述数据结构的工厂方法，用来生成数据结构对象。
    - 创建生产者，用来向上述数据结构填充具体数据。
    - 创建消费者。
    - 创建客户端程序，协调生产者、消费者
        + 创建 `Disruptor` 对象
        + 设置消费者线程
        + 启动并初始化 `Disruptor`
        + 通过生产者生成并存入数据
    - 下述代码参考 [实战Java高并发程序设计][3]

2. 生产-消费者模式，主要是用来进行线程间通信，或者说，主要是用来在线程间传递数据，传递的数据结构我们称之为 `任务`。因此我们的第一步就是要建立这样一个数据结构。代码如下：
```java
package top.voidd.concurrency.disruptordemo;

/**
 * Created by coder on 5/9/2017.
 * task data
 */
public class Data {
    private String value;
    public void setValue(String value) {
        this.value = value;
    }

    public String getValue() {
        return value;
    }
}
```
3. 创建任务工厂方法，这一步是框架的要求，到后面我们就可以看到它的用途。它需要实现 `disruptor` 框架中的 `EventFactory` 接口
```java
package top.voidd.concurrency.disruptordemo;

import com.lmax.disruptor.EventFactory;

/**
 * Created by coder on 5/9/2017.
 */
public class DataFactory implements EventFactory<Data> {
    /**
     * 这个工厂类会在 Disruptor 系统初始化时，构造所有的缓冲区中的对象实例(即预先分配空间)
     * @return
     */
    @Override
    public Data newInstance() {
        return new Data();
    }
}
```

4. 创建生产者。由于框架的封装，我们需要做的事情非常简单，往 `RingBuffer` 类推入任务（即上述数据结构类）即可
```java
package top.voidd.concurrency.disruptordemo;

import com.lmax.disruptor.RingBuffer;

/**
 * Created by coder on 5/9/2017.
 */
public class Producer {
    private final RingBuffer<Data> ringBuffer;

    public Producer(RingBuffer<Data> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }

    /**
     * 将数据推入缓冲区
     * @param value
     */
    public void pushData(String value) {
        long sequence = ringBuffer.next();
        try {
            // 获取空闲位置并设置具体数据
            Data event = ringBuffer.get(sequence);
            event.setValue(value);
        } finally {
            // 数据发布，可以被更新
            ringBuffer.publish(sequence);
        }
    }
}
```
5. 创建消费者。如下
```java
package top.voidd.concurrency.disruptordemo;

import com.lmax.disruptor.WorkHandler;

/**
 * Created by coder on 5/9/2017.
 */
public class Consumer implements WorkHandler<Data> {
    /**
     * onEvent 为框架的回调方法，我们需要做的只是简单地进行数据的处理
     * @param event
     * @throws Exception
     */
    @Override
    public void onEvent(Data event)
            throws Exception
    {
        System.out.println(Thread.currentThread().getName() + ":Event: consumer data: " + event.getValue());
    }
}
```
6. 创建主程序。
```java
package top.voidd.concurrency.disruptordemo;

import com.lmax.disruptor.RingBuffer;
import com.lmax.disruptor.dsl.Disruptor;

import java.util.concurrent.Executor;
import java.util.concurrent.Executors;

/**
 * Created by coder on 5/9/2017.
 */
public class Main {
    public static void main(String[] args)
            throws Exception
    {
        // 用来创建消费者线程
        Executor executor = Executors.newCachedThreadPool();
        DataFactory factory = new DataFactory();
        int bufferSize = 1024; // RaingBuffer大小，必须是2的整数次幂
        Disruptor<Data> disruptor = new Disruptor<>( factory, bufferSize, executor);

        // 连接消费者线程
        disruptor.handleEventsWithWorkerPool(
                new Consumer(),
                new Consumer()
        );
        // 启动框架
        disruptor.start();
        // 由一个生产者不断像缓冲区存入数据
        RingBuffer<Data> ringBuffer = disruptor.getRingBuffer();
        Producer producer = new Producer(ringBuffer);
        for (long l = 0; l < 1000; l++) {
            System.out.println("add data: data " + l);
            producer.pushData("data " + l);
            Thread.sleep(1000);
        }
    }

}
```
7. 执行结果如下
```java
add data: data 0
pool-1-thread-2:Event: consumer data: data 0
add data: data 1
pool-1-thread-1:Event: consumer data: data 1
add data: data 2
pool-1-thread-2:Event: consumer data: data 2
add data: data 3
pool-1-thread-1:Event: consumer data: data 3
// more ...
```

## Disruptor 相关概念
> 有了上述能跑起来的demo，下面我们针对demo解释一些相关概念

**RingBuffer**: 一般情况下，我们使用 `Java` 写的 生产-消费者 都是使用阻塞队列，如 `BlockingQueue, ConcurrentLinkedQueue` 等，明显的，它们都属于线性队列，而 `Disruptor` 框架使用的是一个数组实现的环形队列，这个队列就是 `RingBuffer`，它作为线程通信的缓冲区。

**RingBuffer 容量**：`RingBuffer` 使用环形队列，则其大小无法动态扩展（当然复制新队列除外），因此我们需要实现给定该队列的大小。如果你使用数组实现过环形队列，那对 `seq % queueSize` 应该非常熟悉，我们使用它来找出 seq 在队列中的位置。而 `Disruptor` 使用 `seq & (queueSize-1)` 的方式来定位具体位置。这种方式更高效，因此也带来了额外的要求，`queueSize` 必须是 2 的整数次幂。这种情况下 `queueSize-1` 必然是全1，从而保证了定位的准确性。

**高性能**: `Disruptor` 之所以高效，基于它采用的并发策略 `CAS`(比较交换)。通常情况下，我们会使用锁或阻塞的方式来实现同步，从而构造线程安全的类，如使用 `BlockingQueue`，但线程同步往往意味着低并发。于是我们更进一步，使用 JDK 封装的 `ConcurrentLinkedQueue`，这是一个线程安全的类，并且拥有优秀的并发性能，而秘诀就在于它内部大量使用了 `CAS` 并发策略。而 `Disruptor` 同样使用了 CAS 策略，并且，拥有更多且完善的功能。

**CAS(compare and swap)**: 


[1]: https://github.com/LMAX-Exchange/disruptor/wiki
[2]: https://mvnrepository.com/artifact/com.lmax/disruptor/3.3.6
[3]: https://book.douban.com/subject/26663605/