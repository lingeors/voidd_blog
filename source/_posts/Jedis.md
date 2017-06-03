---
title: Jedis 基本使用
date: 2017-05-06 12:01:26
tags: 
    - Redis
    - Jedis
    - Java
---
> 此文章主要介绍 Java 下 Redis 的基本使用。读者需具备 Redis 的基本操作知识

<!-- more -->

## 安装
下载或使用 `maven` 安装 [`jedis jar`][1] 以及 [`Apache Commons Pool jar`][2]，添加到  `classpath` 路径下即可
## 测试demo
1. 获取 `Jedis` 实例。`Jedis jedis = new Jedis(host, port)`;
2. 如果 `redis` 有设置密码，则需要进行验证 `jedis.auth(password)`
3. 接下来就是执行 `redis` 的各种操作。通过查看文档或源码，可以发现 `jedis` 的各种命令基本和在 `redis` 原始函数同名
4. 实例代码如下
    ```java
    package top.voidd.redis;

    import redis.clients.jedis.Jedis;

    /**
     * Created by coder on 5/5/2017.
     */
    public class JedisDemo {

        public static void main(String[] argv) {
            Jedis jedis = new Jedis("localhost", 6379);
            jedis.auth(""); // if you set the redis password
            jedis.set("jedis.foo", "bar");
            System.out.println(jedis.get("jedis.foo"));
        }
    }
    ```
5. 关于各种函数的返回值我们可以查看 `Jedis` 的 [API 文档][3]

## 实际使用
可以看到，上面的操作非常简单，但一般情况下我们不会直接这么用，因为单个 `Jedis` 实例是线程不安全的，因此我们一般通过`线程池`的方式来使用 `Jedis`。

首先我们需要初始化一个资源池
`JedisPool pool = new JedisPool(new JedisPoolConfig(), host);`
注意 `JedisPool` 是 `Jedis` 包中的类，而它依赖于 `JedisPoolConfig`类，这个配置类是 `Commons Pool` 包中的类，这也是我们需要同时添加这两个 `jar` 的原因！`JedisPoolConfig` 包含了一些和 `redis` 特定连接有关的资源池配置。

代码 demo 如下
```java
package top.voidd.redis;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisPoolConfig;

/**
 * Created by coder on 5/5/2017.
 */
public class JedisPoolDemo {

    public static void main(String[] argv) {
        // 使用 JedisPoolConfig 必须添加它的依赖 Common pool2
        JedisPool pool = new JedisPool(new JedisPoolConfig(), "localhost");
        try (Jedis jedis = pool.getResource()) {
            jedis.auth("");
            System.out.println(jedis.set("foo", "bar"));
        }
        // pool.destroy(); // 不使用 try 关闭资源模式则需要使用 destroy 销毁资源池
    }
}
```
整个过程如下：
1. 创建资源池
2. 从资源池获取 `Jedis` 实例
3. 使用 `Jedis` 实例
4. 资源池使用完毕，调用 `destroy` 方法进行销毁

## 实际使用 demo
> 下面是我在一个小项目中使用 Jedis 的基本 demo

1. 首先使用饿汉单例模式封装资源池，并对每一次请求都返回一个新的 `Jedis instance`
```java
package Redis;

// import ...

/**
 * Created by coder on 5/5/2017.
 */
public class Redis {

    private static JedisPool poll = null;
    private static final String HOST = "localhost";
    private static final String PORT = "3679";
    private static final String PASSWORD = "";

    private Redis() {}

    static {
        poll = new JedisPool(new JedisPoolConfig(), HOST);
    }


    /**
     * get a new jedis instance
     * note that single jedis instance is not thread-safe!
     * so that we are supposed to return a new instance
     * @return
     */
    public static Jedis getJedisInstance() {
        Jedis jedis = poll.getResource();
        jedis.auth(PASSWORD);
        return jedis;
    }
}
```
2. 封装操作接口
```java
package Redis;

/**
 * Created by coder on 5/5/2017.
 * maybe the jedis is not the best choice, i guess.
 */
public interface RedisOperateInterface {

    /**
     * return the number of elements of the list
     * if the key is not the nome of a list, error return
     * @param key
     * @param strings
     * @return
     */
    public Long lpush(String key, String... strings) ;

    /**
     * if the list is not empty, the value is returned
     * or the value 'nil' is returned
     * @param key
     * @return
     */
    public String rpop(String key) ;
}
```
3. 具体实现
```java
package Redis;

import redis.clients.jedis.Jedis;

/**
 * Created by coder on 5/5/2017.
 * jedis implements of RedisOperateInterface
 */
public class JedisImpl implements RedisOperateInterface {
    private Jedis jedis = null;

    public JedisImpl() {
        jedis = Redis.getJedisInstance();
    }

    public Long lpush(String key, String... strings) {
        return jedis.lpush(key, strings);
    }

    public String rpop(String key) {
        return jedis.rpop(key);
    }
}
```
4. 测试(这里需要注意的一点：除了添加 `junit jar` 还需要添加 `amcrest` 依赖)
```java
package test.Reids;

import Redis.JedisImpl;
import Redis.RedisOperateInterface;
import org.junit.Test;

import static org.junit.Assert.assertEquals;

/**
 * Created by coder on 5/5/2017.
 */
public class JedisImplTest {

    @Test
    public void operateTest() {
        RedisOperateInterface ji = new JedisImpl();
        assertEquals(ji.lpush("operate.key", "value").toString(), "1");
        assertEquals(ji.rpop("operate.key"), "value");
    }
}
```

## 参考
[jedis-wiki][4]

[1]:https://mvnrepository.com/artifact/redis.clients/jedis/2.4.2
[2]:https://commons.apache.org/proper/commons-pool/download_pool.cgi
[3]: http://javadox.com/redis.clients/jedis/2.8.0/redis/clients/jedis/package-summary.html
[4]: https://github.com/xetorthio/jedis/wiki/Getting-started