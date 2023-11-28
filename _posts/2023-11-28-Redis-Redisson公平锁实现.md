---
layout: post
title: "Redis-Redisson公平锁实现"
author: "Inela"
---

Redisson公平锁源码(Redisson 3.24.0版本)，由以下几个数据结构组成：

- Redis Hash 数据结构：存放当前锁，Redis Key 就是锁，Hash 的 field 是加锁线程，Hash 的 value 是 重入次数；
- Redis List 数据结构：充当线程等待队列，新的等待线程会使用 rpush 命令放在队列右边，List的name是redisson_lock_queue:{锁key名称}
- Redis sorted set 有序集合数据结构：存放等待线程的顺序，分数 score 用来是等待线程的超时时间戳，zset的name是redisson_lock_timeout:{锁key名称}

​	zset和list可以对比Java的AQS CLH队列，Java中公平锁也是使用CLH队列来实现的。

```
RLock myFairLock = redissonClient.getFairLock("MyFairLock");
myFairLock.lock(10,TimeUnit.MILLISECONDS);

if (myFairLock.isHeldByCurrentThread()) {
	myFairLock.unlock();
}
```

如下getFairLock创建的是RedissonFairLock

```
public RLock getFairLock(String name) {
    return new RedissonFairLock(this.commandExecutor, name);
}
...
public class RedissonFairLock extends RedissonLock implements RLock {
    private final long threadWaitTime;
    private final CommandAsyncExecutor commandExecutor;
    private final String threadsQueueName;
    private final String timeoutSetName;

    public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name) {
        this(commandExecutor, name, 300000L);
    }

    public RedissonFairLock(CommandAsyncExecutor commandExecutor, String name, long threadWaitTime) {
        super(commandExecutor, name);
        this.commandExecutor = commandExecutor;
        this.threadWaitTime = threadWaitTime;
        this.threadsQueueName = prefixName("redisson_lock_queue", name);
        this.timeoutSetName = prefixName("redisson_lock_timeout", name);
    }
...
```

RedissonFairLock继承了RedissonLock，覆写了几个核心的实现，核心在tryLockInnerAsync的Lua脚本

![RedissonFairLock](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-11-28/RedissonFairLock.png)

方便起见，重新贴一下 Lua 脚本，以及脚本的参数含义。

1. KEYS[1]：加锁的名字，`anyLock`；
2. KEYS[2]：加锁等待队列，`redisson_lock_queue:{anyLock}`；
3. KEYS[3]：等待队列中线程锁时间的 set 集合，`redisson_lock_timeout:{anyLock}`，是按照锁的时间戳存放到集合中的；
4. ARGV[1]：锁超时时间 30000
5. ARGV[2]：UUID:ThreadId 组合 `a3da2c83-b084-425c-a70f-5d9a08b37f31:1`
6. ARGV[3]：threadWaitTime 默认 300000
7. ARGV[4]：currentTime 当前时间戳

![RedissonFairLock](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-11-28/redissonFairLock的Lua.webp)

第一部分，while 循环：
从等待队列 redisson_lock_queue:{anyLock} 中获取第一个等待线程；
从等待线程超时集合 redisson_lock_timeout:{anyLock} 中获取第一个等待线程的分数；
没有超时，直接结束，超时了，则直接移除。
第二部分，当前锁存在，直接跳过。
第三部分，当前锁不是持锁线程，直接跳过。

![redisson公平锁](https://github.com/MingJunDuan/mingjunduan.github.io/raw/main/images/mjduan/2023-11-28/redisson公平锁的实现.webp)

​	当线程获取锁失败，进入到等待队列时，ttl != null，在 Java 代码中会不断尝试获取锁。进入等待队列后，redisson客户端会等待一段时间，如果有线程释放锁了，则会给channel发消息通知客户端，客户端从等待中被唤醒过来，然后请求远程redis服务器，看List等待队列的头h，再获取zset中该h对应的score(时间戳)是否小于当前时间，小于则从队列头移除、从zset中移除。
​	当锁不存在且当前线程是在等待队列头时，直接获得锁。这个排队的过程就是公平锁的提现。



参考：

1. https://xie.infoq.cn/article/e7395020f902c38e939a71d3c