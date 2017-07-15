---
title: php 变量实现
date: 2017-05-11 14:01:26
tags:
    - zval
    - php
    - php-internal
---
> PHP是一门弱类型的动态语言，众所周知，它的学习成本较低，从应用场景来看是因为这门语言主要用来做Web开发，功能不需要太复杂，而从语言本身实现来看，是因为php语言的开发者做了很多包装，使得php用起来非常方便。下面我们解开php程序员每天都在使用的“php变量”神秘的面纱，看看高层语言和较底层语言是如何结合在一起的

<!-- more -->

## 前言
如果学习过 C 语言，那读者可能会对语言的基本运作原理有一定的了解。就 C 而言，过程大致如下：我们编写 c 源码，然后由编译器编译形成汇编代码，汇编代码经过汇编器后形成机器可识的机器代码（省略其它步骤）。接着机器便可按照可识别的机器代码按部就班地执行。

而对于 php 这种入门相对比较容易的解释型语言又是经过什么步骤才能被机器识别的，可能很多学习过 php 的读者就不那么关心的（毕竟语言太高级 :-)
尽管要了解 php 具体的实现不太现实，但对具体的某一部分做些许探究却并不是不可能

## php 语言如何运行
关于php的实现，可以简单概括为：**对比较底层的 c语言 的二次封装**。因此，我们编写的 php 代码实际上是要经历这样一个过程：`php 源码 --> c 代码 --> 汇编代码 --> 机器代码`。而 php 语言本身的编写者实现的主要是 php 源码到 c 代码的转换过程。如下面的 `test.php` 文件：
```php
<?php
$var = "hello world";
echo $var;
echo "\n";
```
当我们执行该代码文件 `php test.php`。大致会经历这样一个过程：
- php 内核接收 `test.php` 文件输入流，
- 对php代码进行词法分析和语法分析，词法分析将php代码分割成token，语法分析将token转化为 Zend Engine(执行引擎) 可执行的操作
- 由 `zend Engine` 执行这些操作（c语言执行）。
- 将结果返回。

其中 `Zend Engine` 可以类比 JVM（当然二者不是一回事），它们都生成了中间代码，JVM 是 CLASS 类型的字节码，而ZEND生成称为 opcode 的中间码。
`Zend Engine` 是 php 实现的核心，它提供了语言实现的基础设施，如 php 语法实现，脚本的编译环境，扩展机制以及内存管理等。

## php 变量
所谓php语言实现，也就是采用什么策略、算法来用 c 封装 php 语言特性，使它成为一门易于使用的高级语言。而php变量无疑是重要的基础实现之一。

### 变量信息
首先我们回忆一下，变量需要具备哪些信息？还是从上面的脚本看 `$var = "hello world";` 首先是变量的名称 `$var`，接着是变量的具体信息 `hello world`，另外还有弱类型语言经常被忽略的变量类型 `String`。因此变量的底层实现至少需要包括上述三部分信息。

### 变量底层实现
> note: 所有c源码都是基于 php-5.6.30

首先我们可以查看 `Zend/zend.h` 文件 334 行关于变量的定义
```c
struct _zval_struct {
    /* Variable information */
    // 存储值信息的联合体
    zvalue_value value;     /* value */
    // 技术引用。垃圾回收机制一部分
    zend_uint refcount__gc;
    // 变量类型
    zend_uchar type;    /* active type */
    // 是否为引用类型
    zend_uchar is_ref__gc;
};
```
可以看到，php 的变量在c中是通过一个结构体来表示的。
我们分别简单介绍下上述结构体中各部分的信息
- `value` 这是一个 zvalue_value 类型的联合体，用来保存变量的具体值。这个后面再说
- `refcount__gc` 初始化一个变量自然就意味着需要分配出一部分内存，因此当该变量或变量指向的资源不再被需要时，自然需要被销毁。这个值作为php垃圾回收机制的一部分。熟悉引用计数垃圾回收的读者应该很容易理解。
- `type` 此变量用来存储变量的类型。我们知道，语言分为强类型和弱类型，比如 Java 属于强类型语言，这意味着 `String var = "abc"` 中 `var` 变量将无法被赋值为其它类型，而弱类型语言如 php，py 等则没有这个限制。对php而言，在修改 `zvalue_value` 时只要相应地修改此变量的值即可。
- `is_ref__gc` 表示php中的一个类型是否是引用类型

<br>

接下来我们看下 `zvalue_value` 的具体实现，还是先看源码
```c
typedef union _zvalue_value {
    long lval;                  /* long value */
    double dval;                /* double value */
    struct {
        char *val;
        int len;
    } str;
    HashTable *ht;              /* hash table value */
    zend_object_value obj;
    zend_ast *ast;
} zvalue_value;
```
可以看到，变量的具体信息是存放在一个 c联合体中的，根据联合体的内存分配，很容易知道这样做的主要目的是为了节省内存。(php 7 对此做了很大的优化)
从联合体的各部分而言，我们可以看到php支持的各种变量的类型
- 单精度和双精度浮点型
- 字符串类型：使用字符类型指针存储具体值，并保存了字符串长度，也就是说函数 `strlen(str)` 是 `O(1)` 的。（刚开始学习C语言时的我们是不是对 `str_len()` 的 *O(n)* 感到困惑 :-)
- `ht` 用来实现php中的数组。它是一个使用分离链表法的哈希表。链表使用的是双向链表。其实现比较复杂，有兴趣的可以查看 `Zend/zend_hash.h` 和 `Zend/zend_hash.c` 文件
- `obj` 该结构体用来存储对象信息。该结构体包含一个 `handlers` 属性，用来引用对象在对象池中的位置。
- `ast` php 的抽象语法树，此处的具体作用暂时还没搞懂（zend.ast.h）

## 总结
php 作为一门高级语言，底层是对 c 语言的封装，因此对其具体运作过程往往需要在 c 层面来理解。上面我们提到的php变量也是这样，简而言之，php变量在底层用 c 结构体来表示，结构体存储了变量的基本信息，我们在php中对变量的所有操作都可以映射到底层结构体的变化中来。
通过该结构体，我也可以理解php弱类型的原理以及其他一些php变量操作的相关实现。如果我们从该结构体深入下去阅读相关源码，比如数组的实现，则可以理解php灵活的数组定义以及相关操作是基于什么数据结构实现的，以及，是如何实现的。

## 参考
[TIPI: 深入理解PHP内核][2]
[php7-internal/zval.md at master · laruence/php7-internal][1]

[1]: https://github.com/laruence/php7-internal/blob/master/zval.md
[2]: http://www.php-internals.com/book/?p=chapt03/03-01-00-variables-structure
