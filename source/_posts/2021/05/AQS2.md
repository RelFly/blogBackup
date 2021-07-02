---
title: AQS独占共享模式
date: 2021-05-10 21:15:52
tags:
- JUC
categories:
- Java
- JUC
---

### 前言
   
   本章针对AQS的独占共享两种模式，通过源码分析其具体的实现逻辑。
<!-- more -->

### 独占模式

   独占模式即同一时间，只能允许一个线程获取锁资源。
   这里结合ReentrantLock分析:
{% codeblock lang:java %}
// AQS中响应中断的独占模式获取方法
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        // 获取锁失败
        doAcquireInterruptibly(arg);
}
// ReentrantLock中tryAcquire的具体实现(非公平锁)
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}

// AQS获取锁失败后的方法
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    // 新建独占节点 使用尾插法插入队列尾部
    // Node.EXCLUSIVE作为新节点nextWaiter属性的值 用于标识独占节点
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
        	// 前置节点
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 当前置节点为头节点且获取锁成功 表明锁被释放
                // 将当前节点设置为头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
{% endcodeblock %}
{% img  /image/xxxx/xxxxx.png  '"xxxx"' %}