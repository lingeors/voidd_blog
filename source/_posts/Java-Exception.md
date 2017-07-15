---
title: Java 异常相关问题
date: 2017-05-15 20:32:21
tags:
  - Java
  - 异常

---

> 記得之前看過某個大佬的文章，他說Java不是他主要工作語言，但他從Java中學到的東西是最多的。我想我正在慢慢理解他的這句話。

<!-- more -->

## 前言
很多语言都有异常处理机制，之前我对异常的理解可能和大部分的新手一样，知道异常是什么，平时会使用到异常，通过相关的书籍或博客对异常也有相应的了解。但是，总觉得缺了点什么，这也导致了3月份面某厂的 Java 岗，被问到异常时答的不是很好（当然当时学习Java也没多久）。后来又再次看了相关的书籍，做了一些总结，产出这篇文章。主要是对一些以前没理解好的概念做一些梳理

## 概述
异常，就是不正常的情况。这里我们所说的异常更多指的是异常处理机制。
如果我们使用过不支持异常的语言，比如 C 语言，我们应该会有所体会：当我们要告诉客户端程序员被调用程序发生了哪些问题时，通常会通过返回某个特殊值或者设置特殊标识，然后调用者对它们进行相应的判断和处理。如下代码展示了一种简单的情形
```c
// api
int doSomething() {
    doSomething
    if (condition1) return 1;
    if (condition2) return 2;
    return 0;
}
// client
int flag = doSomething();
if (flag == 1) do...
else if (flag == 2) do...
```
通过上述例子，我们可以发现
1. 错误处理机制完全由程序员自定义，难以阅读
2. 客户端程序很可能会忽略这些错误
3. 程序逻辑代码和错误处理代码耦合

针对传统错误处理机制的不足，我们可以这样介绍异常机制：语言本身支持的对程序发生的非正常情况的处理机制。即当发生某些不正常情况时，异常处理机制会跳出当前的执行流，并通过异常的方式（抛出一个异常对象）将问题反馈给我们，我们要做到只是捕获到程序告诉我们的异常然后做恰当的处理。

## 异常的好处
看下述例子
```java
// api
int doSomething() throws SomeException {
    throw new SomeException();
    doOtherThing;
}
// client
try {
    doSomething();
    doOtherThing;
} catch (SomeException e) {
    handle the exception
}
```
其中，包裹在try中的代码被称为监控区域，它可能会产生异常。而包裹在catch中的代码则对前面可能产生的 `SomeException` 异常进行了处理。
首先我们会执行 `doSomething()`，当这段代码发生 `SomeException` 异常时，catch 中的程序将会被执行。
和上面的 c 代码做一个比较，我们可以得出以下结论
1. 其中 api 部分专心处理它的业务，错误处理程序被集中到了 catch 语句块中。
2. 代码更加易读，我们清楚什么时候可能会出现什么问题，并决定怎么处理这些问题
3. 如果产生问题，程序将无法沿着正常的路径执行下去，这样处理显然更符合逻辑，发生异常的程序正常执行下去本身就是错误

## 异常执行过程
同样是上面的例子，当发生 `SomeException`异常时，会先创建异常对象，然后终止当前的执行路径（这意味者api和client中的doSomething都不会被执行），异常处理程序接管程序，执行异常处理对象。具体到上面的例子，在 api 中，当异常发生时，它并不做任何处理，而是通过 throws 关键字将异常交给更高层的代码（这里就是client）。client 捕获了这个异常，然后通过 catch 语句进行处理，这里 catch 中的语句就是异常处理程序。当然，我们也可以在 api 中捕获异常并进行处理。

## Java 中异常
首先我们需要明确一点，在 Java 中，我们说的异常其实就是一些 Java 类。所有异常的基类为 `Throwable`，继承它的类有两个，分别是 `Error` 和 `Exception`。而 `Exception` 的子类又可分为两大类，一类继承自 `RuntimeException`，另一类我们暂时称做“非运行时异常”。

关于 `Error` 及其子类，表示的是编译时和系统错误，这部分异常我们无法控制和避免，也无需捕获，因此我们并不关心
关于 `Exception` 及其子列，表示可以被抛出的基本类型。程序员主要关系的是这个类及其子类。

通过阅读源码，会发现其实这些类都非常简单。基类 `Throwable` 基本实现了所有的功能，而它的子类几乎都只是对基类的构造器做了一个简单的重载。也就是说，Java 异常庞大的继承体系所存储的信息其实并不多（类内部几乎不存储信息），我个人把它们分为三部分
1. `Throwable` 本身，因为它实现了异常类的几乎所有功能
2. 继承体系本身，通过继承体系我们可以了解子类的特点以及作用
3. 类名，类名基本是自解释的，可以说，子类的大部分信息就在类名中

了解了这几个异常间的关系后再去稍微看一下源码，会对整个异常体系有很好的了解。比较推荐。

## 运行时异常
`Exception` 分为两大类，一类为 `RuntimeException`，继承自它的类我们称为运行时异常。另一类我们暂且称为非运行时异常。
区分它们很简单，如果是编程错误，则是前者，这种问题应该由程序员负责，比如 `IndexOutOfBoundsException`，它表示数组访问越界问题，再如 `NullPointerException` 它表示访问了未初始化的对象，这些情况多加注意一般都可以解决。而如果是非编程错误，我们将它归为第二类。在Java中，这种异常被强制处理。称为受检异常，如 `IOException`，在IO操作中，这种异常可能会发生，比如所打开的文件不存在，这时它显然不是编程错误，即使它可以通过编程方式来比较优雅地处理。

## 受检异常
在 Java 中，`Error` 类异常是不需要程序员处理的，`RuntimeException` 类异常属于编程错误，是需要用户避免的（也就是说，良好的编程实践中，用户不会捕获这类异常）。这两类异常也称为 “非受检异常”，这里的“受检”表示受用户检查，意思为 “此类错误用户不需要做异常处理”，如果用户疏忽导致 `RuntimeException` ，此类异常会被 JVM 捕获而不是编译器。而另一类异常则是强制处理的，这类异常称之为 “受检异常”，即必须检查的异常（检查的主语为程序员，必须指如果不处理，则无法通过编译）。同样使用上面的例子：
```java
// api
int doSomething() throws IOException {
    throw new IOException();
    doOtherThing;
}
// client
try {
    doSomething();
    doOtherThing;
} catch (IOException e) {
    handle the exception
}
```
`IOException` 是一个受检异常，因此在异常发生时，我们要么使用 `throws` 关键字将它交给高层代码处理，要么使用 try-catch 进行捕获。否则将会发生编译错误。

## 异常链
> 了解这个概念前建议先看下 `Throwable` 的源码。

先看以下代码
```java
try {
    access the database
} catch (SQLException e) {
    // 方式一
    Throwable se = new ServletException("database error");
    se.initCase(e);
    throw se;
    // 方式二
    Throwable se = new ServletException("database error", e);
    throw se;
}
// 捕获上述异常的原始异常
Throwable e = se.getCause();
```
上述代码捕获了 `SQLException` 异常，然后重新抛出了 `ServletException` 异常。但需要注意的是，`new ServletException("database error", e);` 的第二个参数，它是前一个异常对象，它保存在 Throwable 类的 `cause` 属性中，它表示抛出 `ServletException` 的原因是捕获到了 `SQLException` 异常。
然后我们可以通过 `getCause()`方法来获得前一个异常。它的作用是保存原始异常（即当前异常的前一个异常对象）的信息。称之为原型链。

## [良好实践][1]
1. 只针对异常情况使用异常
2. 对可恢复的情况使用受检异常，对编程错误使用运行时异常
3. 避免不必要地使用受检异常
4. 优先使用标准的异常（即通常情况下避免自定义异常）
5. 抛出与抽象相对应的异常
6. 每个方法抛出的异常都要有文档
7. 努力是失败保持原子性（一般而言，失败的方法调用应该使对象保持在调用之前的状态，具有这种属性的方法称为具有失败原子性）
8. 不要忽略异常

## 总结
搞清楚 Java 的异常，个人实践如下：了解异常概念；了解 Java异常关键字，语句；了解Java标准类的继承体系；大致查看Java标准异常类源码；了解运行时异常、非运行时异常区别；了解受检异常、非受检异常的概念和区别；了解相关异常最佳实践

## 参考
[Java 编程思想][1]
[Effective Java][2]

[1]: https://book.douban.com/subject/2130190/?from=0&typed=8
[2]: https://book.douban.com/subject/3360807/?from=1&typed=6
