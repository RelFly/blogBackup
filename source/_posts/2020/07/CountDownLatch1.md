---
title: CountDownLatch(jdk1.8)
date: 2020-07-13 22:09:40
tags:
- JUC
categories:
- Java
- JUC
---

### 前言

  本章对CountDownLatch的原理及应用场景总结一下。
<!-- more -->

### 源码解读

#### 内部类Sync

{% codeblock lang:java %}
// 继承了AQS的内部类，使用AQS的state作为计数
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    Sync(int count) {
        setState(count);
    }
    int getCount() {
        return getState();
    }
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
    // countDown操作
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            // 使用CAS更新state
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
{% endcodeblock %}

#### 操作方法

{% codeblock lang:java %}
// 构造方法，初始化count，可以调用await()使之休眠直到count归零
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

// 使当前线程等待，直到计数count归零才会唤醒
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// 使当前线程等待，直到计数count归零 或 等待时间==timeout
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

// 计数器递减
// count>0 -1 若-1操作的结果为零则唤醒所有等待线程
// count==0 不做操作
public void countDown() {
    sync.releaseShared(1);
}
{% endcodeblock %}

#### 调用的AQS方法
{% codeblock lang:java %}
// 等待阻塞
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // tryAcquireShared(arg)走Sync重写的逻辑判断计数
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

// 释放
public final boolean releaseShared(int arg) {
    // 这里的tryReleaseShared(arg)走的使Sync重写的逻辑
    if (tryReleaseShared(arg)) {
        // 确认释放，走AQS的释放方法
        doReleaseShared();
        return true;
    }
    return false;
}
{% endcodeblock %}

  从源码中可以看到，CountDownLatch的实现依赖于继承了AQS的内部类Sync。
  他的操作方法不错，主要就是初始化定义计数阈值，然后通过await()方法阻塞线程直到计数归零，同归countDown()递减控制计数。

### 使用示例

{% codeblock lang:java %}
public void mian(String[] args){
    CountDownLatch masterFlag = new CountDownLatch(1);
    CountDownLatch slaveFlag = new CountDownLatch(3);
    new Thread(() -> {
        try {
            masterFlag.await(1, TimeUnit.MILLISECONDS);
            logger.info("thread run not waiting：{}", Thread.currentThread().getName());
            slaveFlag.countDown();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }).start();
    for (int i = 0; i < 2; i++) {
        new Thread(() -> {
            try {
                masterFlag.await();
                logger.info("start sleep:{}", Thread.currentThread().getName());
                Thread.sleep(10000);
                logger.info("thread run：{}", Thread.currentThread().getName());
                slaveFlag.countDown();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();
    }
    Thread.sleep(10000);
    masterFlag.countDown();
    logger.info("master countDown");
    slaveFlag.await();
    logger.info("slave countDown finish");
}
{% endcodeblock %}

  输出结果

{% img /image/CountDownLatch/CountDownLatch1.png '"示例结果"'%}

  从结果可以看到，Thread-4因为使用的是*await(long, timeUnit)* 方法，其等待时间比较短，所以在主线程休眠时就会被唤醒。
  而其他线程都是等到主线程休眠计数将*masterFlag* 计数归零后才被唤醒。

### 小结

      总的来看CountDownLatch其实很好理解，借助AQS实现一个计数器，以达到控制线程等待唤醒的目
    的。不过想要理解他的底层实现原理就得了解AQS的实现了，关于AQS的内容就不在本章赘述。
      在实际应用中，CountDownLatch应该是控制一批线程相互等待并让其同时被唤醒这种场景。例如批
    量下载，要求全部下载完之后发送一个成功的提醒。

> [浅析AQS(jdk1.8)](https://rel-fly.com/2020/07/14/AQS1/)