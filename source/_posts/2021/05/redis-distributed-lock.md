---
title: reids-分布式锁
date: 2021-05-01 10:09:09
tags:
- 分布式锁
- redis
categories:
- redis
---

### 前言

	本章来谈谈如何使用redis实现分布式锁
<!-- more -->

### 常规方案

#### 方案一
    
    最简单的方案就是利用redis的set NX命令去获取锁，用delete命令释放锁。
    这种实现简单直接，但其有以下弊端：
    1. setnx命令后需要执行expire设置锁的过期时间，否则，持有锁的线程down掉，锁永远无法释放
       而这两个命令是非原子性的，无法保证一起成功
    2. delete释放锁，在持有锁的线程还未结束，但是已超过锁的过期时间，这时第二个获取到锁的线程
       正准备执行，第一个线程执行完毕执行delete，释放的就是第二个线程持有的锁
    3. 无法实现重入

#### 方案二

    有问题自然就要解决，针对方案一的问题1和2，有以下解决方案：
    1. 使用redis set命令的参数EX PX，可以同时设置过期时间，而spring的redisremplate也封装了
       setIfAbsent(K key, V value, long timeout, TimeUnit unit) 方法实现该功能
    2. 为了防止释放锁时误删不属于自己的锁，可以对缓存的value设置一个自定义值，在释放时进行
       判断，如果属于自己才执行delete操作
    3. 为保证释放锁操作的原子性，可以借用lua脚本提交释放锁的命令

    相对方案一，方案二仍然存在几个问题
    1. 在服务挂掉时，只能依赖锁超时来释放锁，在一些场景下并不友好
    2. 若redis有多个节点，可能存在主节点挂掉，锁信息还未同步到从节点导致锁丢失的情况
    3. 依然不能重入

### Redlock算法

    常规方案只适合于redis单点部署的情况，就如方案二中提到的，若redis存在多个节点就可能因为主节点挂掉
    且锁信息未及时复制到从节点而导致锁失效。
    所幸，redis提供了解决这种问题的方案，那就是Redlock算法

#### 算法思路

    总的看来，Redlock的思路是将锁存在N个主节点中，这样减小主节点挂掉对锁的影响。
    而为了较为安全搞笑实现这一过程，他的具体思路如下：
    1. 获取当前毫秒级的时间
    2. 依次用相同的key和随机值去每个redis实例中获取锁
       设置一个超时时间，某个实例获取锁的时间超过就放弃，去下个实例
    3. 当超过一半(至少达到N/2+1个)的锁获取成功，且获取锁所花费的时间小于设置的锁有效时间，则视为
       获取锁成功
    4. 获取锁成功后，线程能够持有的时间=最初有效时长-获取锁消耗的时间
    5. 若最终获取失败(获取的实例个数小于一半，或最终有效时间无意义)，则会解除所有实例的锁

### redisson

    redisson是基于redis实现的一个Java驻内存数据网格。
    其根据redlock实现了分布式锁。

#### RLock类

{% codeblock lang:java %}
// 分布式锁接口
public interface RLock extends Lock, RExpirable, RLockAsync {
    // 获取锁的三个方式，leaseTime表示获取到锁后的有效时间
    void lockInterruptibly(long leaseTime, TimeUnit unit) throws InterruptedException;
    boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException;
    void lock(long leaseTime, TimeUnit unit);

    void forceUnlock();

    boolean isLocked();

    boolean isHeldByCurrentThread();
    // 重入次数的获取
    int getHoldCount();

}
{% endcodeblock %}
   
#### RedissonLock类

{% codeblock lang:java %}
// 具体的实现类
public class RedissonLock extends RedissonExpirable implements RLock {
	
}
// 核心方法
// 加锁逻辑 这里的threadId为当前线程id
<T> RFuture<T> tryLockInnerAsync(long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
    internalLockLeaseTime = unit.toMillis(leaseTime);
    // 通过lua提交redis命令保证原子性
    // KEYS[1] 指 getName()的值也就是创建锁对象时设置的key
    // ARGV[1] 指 internalLockLeaseTime 也就是锁的有效时间
    // ARGV[2] 指 threadId 当前线程id
    // 第一段lua命令是 如果key不存在就设置一个hash类型缓存，
    //                hash中元素的key为当前线程id，value为1
    // 第二段命令则是重入锁判断 用当前线程id去hash里判断是否存在
    //                        若存在，则将value+1
    // 最后返回此时锁的剩余有效时间
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, command,
              "if (redis.call('exists', KEYS[1]) == 0) then " +
                  "redis.call('hset', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                  "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                  "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                  "return nil; " +
              "end; " +
              "return redis.call('pttl', KEYS[1]);",
                Collections.<Object>singletonList(getName()), internalLockLeaseTime, getLockName(threadId));
}
// 锁释放的逻辑
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
    return commandExecutor.evalWriteAsync(getName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
    		// 锁不存在，发布unlockMessage消息
            "if (redis.call('exists', KEYS[1]) == 0) then " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; " +
            "end;" +
            // 锁存在，但hash获取不到对应key的值，表示锁被其他线程占用，直接返回
            "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                "return nil;" +
            "end; " +
            // 当前线程释放锁，锁重入-1
            "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
            // 还未完全释放，重置有效时间
            "if (counter > 0) then " +
                "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                "return 0; " +
            // 释放锁 并发布消息
            "else " +
                "redis.call('del', KEYS[1]); " +
                "redis.call('publish', KEYS[2], ARGV[1]); " +
                "return 1; "+
            "end; " +
            "return nil;",
            Arrays.<Object>asList(getName(), getChannelName()), LockPubSub.unlockMessage, internalLockLeaseTime, getLockName(threadId));
}
{% endcodeblock %}

#### RedissonRedLock类

{% codeblock lang:java %}
// redlock的实现，将多个锁当作一个锁管理
public class RedissonRedLock extends RedissonMultiLock {
    // 构造方法是传入多个RLock对象，表示多个redis节点获取锁	
    public RedissonRedLock(RLock... locks) {
        super(locks);
    }
    // 允许获取失败的最大节点数
    @Override
    protected int failedLocksLimit() {
        return locks.size() - minLocksAmount(locks);
    }
    // 获取锁成功要求的最小节点数
    protected int minLocksAmount(final List<RLock> locks) {
        return locks.size()/2 + 1;
    }
    @Override
    public void unlock() {
        unlockInner(locks);
    }
    @Override
    protected boolean isLockFailed(Future<Boolean> future) {
        return false;
    }
    @Override
    protected boolean isAllLocksAcquired(AtomicReference<RLock> lockedLockHolder, AtomicReference<Throwable> failed, Queue<RLock> lockedLocks) {
        return (lockedLockHolder.get() == null && failed.get() == null) || lockedLocks.size() >= minLocksAmount(locks);
    }
}
{% endcodeblock %}

#### RedissonMultiLock


{% codeblock lang:java %}
// 将多个锁作为一个锁管理
public class RedissonMultiLock implements Lock {
    // 锁集合
    final List<RLock> locks = new ArrayList<RLock>();
    // 构造方法
    public RedissonMultiLock(RLock... locks) {
        if (locks.length == 0) {
            throw new IllegalArgumentException("Lock objects are not defined");
        }
        this.locks.addAll(Arrays.asList(locks));
    }
    // other code
}
// 获取锁逻辑
public boolean tryLock(long waitTime, long leaseTime, TimeUnit unit) throws InterruptedException {
    long newLeaseTime = -1;
    if (leaseTime != -1) {
        // 将锁的有效时长设置为2倍的等待时长
        // 真正传入的有效时长最后设置
        newLeaseTime = waitTime * 2;
    }
    // 当前时间点
    long time = System.currentTimeMillis();
    long remainTime = -1;
    if (waitTime != -1) {
        remainTime = unit.toMillis(waitTime);
    }
    // 允许获取失败的最大节点数
    int failedLocksLimit = failedLocksLimit();
    // 成功获取到锁的集合
    List<RLock> lockedLocks = new ArrayList<RLock>(locks.size());
    // 遍历锁对象
    for (ListIterator<RLock> iterator = locks.listIterator(); iterator.hasNext();) {
        RLock lock = iterator.next();
        boolean lockAcquired;
        try {
            if (waitTime == -1 && leaseTime == -1) {
                // 无有效时间要求的锁获取
                lockAcquired = lock.tryLock();
            } else {
                // 设置等待时间
                long awaitTime = unit.convert(remainTime, TimeUnit.MILLISECONDS);
                // 实际调用RedissonLock的tryLock方法
                // 当获取锁后的时间点与初识时间点的差小于awaitTime 便视作获取锁失败
                lockAcquired = lock.tryLock(awaitTime, newLeaseTime, unit);
            }
        } catch (Exception e) {
            lockAcquired = false;
        }
        if (lockAcquired) {
            // 获取锁成功则加入集合
            lockedLocks.add(lock);
        } else {
            // 获取锁失败
            // 获取成功的节点已达到要求，不再尝试
            if (locks.size() - lockedLocks.size() == failedLocksLimit()) {
                break;
            }
            if (failedLocksLimit == 0) {
                // 允许获取失败的节点为0，则此时整个获取锁的操作失败
                unlockInner(lockedLocks);
                if (waitTime == -1 && leaseTime == -1) {
                    return false;
                }
                failedLocksLimit = failedLocksLimit();
                lockedLocks.clear();
                // reset iterator
                while (iterator.hasPrevious()) {
                    iterator.previous();
                }
            } else {
                // 一个节点失败，允许的失败节点数减1
                failedLocksLimit--;
            }
        }
        if (remainTime != -1) {
            // 计算剩余可等待时长
            remainTime -= (System.currentTimeMillis() - time);
            // 重置开始时间点
            time = System.currentTimeMillis();
            // 超出可等待时长
            if (remainTime <= 0) {
                unlockInner(lockedLocks);
                return false;
            }
        }
    }
    // 要求的锁有效时间不为永久
    if (leaseTime != -1) {
        List<RFuture<Boolean>> futures = new ArrayList<RFuture<Boolean>>(lockedLocks.size());
        // 遍历设置锁的有效时间
        for (RLock rLock : lockedLocks) {
            RFuture<Boolean> future = rLock.expireAsync(unit.toMillis(leaseTime), TimeUnit.MILLISECONDS);
            futures.add(future);
        }
        for (RFuture<Boolean> rFuture : futures) {
            rFuture.syncUninterruptibly();
        }
    }
    return true;
}
{% endcodeblock %}

### 总结
    
    可以看到，redisson不仅实现了常规方案的分布式锁，还根据redlock算法实现了进一步更有效安
    全的分布式锁。所以想用redis实现分布式锁，其实直接用redisson就行了。
    当然，redlock算法其实也有争议，主要是因为多个节点获取锁虽然能够避免单节点服务挂掉及主从
    复制导致的锁信息丢失，但也有可能出现节点间锁信息不一致的问题。具体可以看看参考链接。

### 参考

> [Distributed locks with Redis](https://redis.io/topics/distlock)
> [How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
> [Is Redlock safe?](http://antirez.com/news/101)
> [Redlock：Redis分布式锁最牛逼的实现](https://www.jianshu.com/p/7e47a4503b87)


