---
title: JUC-Lock(jdk1.8)
date: 2020-09-12 16:43:23
tags:
- JUC
categories:
- Java
- JUC
---

### 前言

   Lock是JUC包下定义了锁相关方法的接口，相对于synchronized，其提供了更多的锁功能，如响应中断，超时锁等。
   本章就基于Lock及Condition的使用和实现原理做一个学习总结。
<!-- more -->

### Lock与synchronized

#### synchronized

1. 基于JVM实现的非同步锁，锁释放由虚拟机完成，死锁可能性较低
2. 使用wait/notify控制线程的等待和唤醒
3. 提供锁升级方案提升性能
4. 无法提供响应中断，超时锁等功能

#### Lock

1. 利用CAS和代码逻辑实现
2. 提供了synchronized没有的很多功能
3. 锁的控制全由开发者自己操作，容易因没有释放锁而导致死锁	 

### 四种获取锁的方法

1. void lock()
   *lock()*方法是最常用的获取锁的方法，使用该方法会一直尝试获取锁，直到获取成功。
{% codeblock lang:java %}
public void testLock() throws InterruptedException {
    Lock lock = new ReentrantLock();
    lock.lock();
    threadPoolExecutor.execute(()->{
        logger.info("尝试获取锁");
        lock.lock();
        logger.info("获取到锁");
        lock.unlock();
    });
    Thread.sleep(5000);
    logger.info("过去了5S");
    lock.unlock();
}
{% endcodeblock %}
{% img  /image/lock/lock1.png  '"lock()方法"' %}

2. void lockInterruptibly() throws InterruptedException
   响应线程中断，当线程中断时，会停止获取锁的尝试并抛出中断异常，如果线程没有发生中断则和*lock()*方法一样。
{% codeblock lang:java %}
public void testLock() throws InterruptedException {
    Lock lock = new ReentrantLock();
    lock.lock();
    Thread thread = new Thread(() -> {
        logger.info("尝试获取锁···");
        try {
            lock.lockInterruptibly();
        } catch (InterruptedException e) {
            logger.info("中断");
        }
        lock.unlock();
    });
    thread.start();
    Thread.sleep(5000);
    logger.info("过去了5S");
    thread.interrupt();
    lock.unlock();
}
{% endcodeblock %}
{% img  /image/lock/lock2.png  '"lockInterruptibly()方法"' %}

3. boolean tryLock()
   *tryLock()*与*lock()*不一样，他只会尝试一次，如果成功则占用锁，失败则不再尝试。
{% codeblock lang:java %}
public void testLock() throws InterruptedException {
    Lock lock = new ReentrantLock();
    lock.lock();
    threadPoolExecutor.execute(() -> {
        logger.info("1尝试获取锁···");
        boolean result = lock.tryLock();
        logger.info("1获取结果：{}", result);
        if (result) {
            logger.info("1获取成功");
            lock.unlock();
        }
    });
    threadPoolExecutor.execute(() -> {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info("2尝试获取锁···");
        boolean result = lock.tryLock();
        logger.info("2获取结果：{}", result);
        if (result) {
            logger.info("2获取成功");
            lock.unlock();
        }
    });
    Thread.sleep(5000);
    logger.info("过去了5S");
    lock.unlock();
}
{% endcodeblock %}
{% img  /image/lock/lock3.png  '"tryLock()方法"' %}

4. boolean tryLock(long time, TimeUnit unit) throws InterruptedException
   本方法在*tryLock()*的基础上设置了一个等待时间，在限定时间内会一直等待锁释放，超过了就不再尝试获取锁。另外，本方法也是和*lockInterruptibly()*一样响应线程中断的。
{% codeblock lang:java %}
public void testLock() throws InterruptedException {
    Lock lock = new ReentrantLock();
    lock.lock();
    Thread thread1 = new Thread(() -> {
        logger.info("1尝试获取锁···");
        boolean result = false;
        try {
            result = lock.tryLock(6, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info("1获取结果：{}", result);
        if (result) {
            logger.info("1获取成功");
            lock.unlock();
        }
    });
    Thread thread2 = new Thread(() -> {
        logger.info("2尝试获取锁···");
        boolean result = false;
        try {
            result = lock.tryLock(7, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        logger.info("2获取结果：{}", result);
        if (result) {
            logger.info("2获取成功");
            lock.unlock();
        }
    });
    thread1.start();
    thread2.start();
    Thread.sleep(5000);
    thread1.interrupt();
    logger.info("过去了5S");
    lock.unlock();
}
{% endcodeblock %}
{% img  /image/lock/lock4.png  '"tryLock(long time, TimeUnit unit)方法"' %}

### Condition

  正如synchronized能够通过wait/notify控制线程等待/唤醒一样，Lock也有配套的Condition配合控制线程。
  当抢占到lock锁资源时，便可使用lock中的Condition的方法控制线程。就像进入synchronized同步块后便可使用wait/notify方法一样。

#### 方法介绍

1. void await()
   最常用的等待方法，线程会一直等待直到有另一个线程调用*signal()*唤醒他，如果线程中断则抛出异常。

2. void awaitUninterruptibly()
   与await()相似，不同的是当检查到当前线程中断，会进入一个安全的中断方法而不是抛出异常。

3. long awaitNanos(long nanosTimeout) throws InterruptedException
   与await()的不同是会设定一个等待时间，只在这个时间段内等待。

4. boolean awaitUntil(Date deadline) throws InterruptedException
   与awaitNanos(long n)类似，不过其设定的是一个时间点。

5. void signal()
   唤醒方法，唤醒队列的头节点

6. void signalAll()
   唤醒方法，唤醒队列中所有节点

#### Condition使用

1. 线程等待唤醒
{% codeblock lang:java %}
public void testCondition() throws InterruptedException {
    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();
    threadPoolExecutor.execute(()->{
        logger.info("尝试获取锁···");
        lock.lock();
        logger.info("获取成功");
        try {
            condition.await();
            logger.info("苏醒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.unlock();
    });
    Thread.sleep(5000);
    logger.info("过了5s···");
    lock.lock();
    logger.info("master获取成功");
    condition.signal();
    logger.info("唤醒等待线程···");
    lock.unlock();
    Thread.sleep(5000);
}
{% endcodeblock %}
{% img  /image/lock/lock5.png  '"Condition演示1"' %}

2. 多个Condition
{% codeblock lang:java %}
public void testCondition() throws InterruptedException {
    Lock lock = new ReentrantLock();
    Condition conditionOne = lock.newCondition();
    Condition conditionTwo = lock.newCondition();
    threadPoolExecutor.execute(() -> {
        logger.info("1尝试获取锁···");
        lock.lock();
        logger.info("1获取成功");
        try {
            conditionOne.await();
            logger.info("1苏醒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.unlock();
    });
    threadPoolExecutor.execute(() -> {
        logger.info("2尝试获取锁···");
        lock.lock();
        logger.info("2获取成功");
        try {
            conditionTwo.await();
            logger.info("2苏醒");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        lock.unlock();
    });
    Thread.sleep(5000);
    logger.info("过了5s···");
    lock.lock();
    logger.info("master获取成功");
    conditionOne.signal();
    logger.info("唤醒等待线程1···");
    lock.unlock();
    //---------------------------------
    Thread.sleep(5000);
    logger.info("又过了5s···");
    lock.lock();
    logger.info("master获取成功");
    conditionTwo.signal();
    logger.info("唤醒等待线程2···");
    lock.unlock();
    Thread.sleep(5000);
}
{% endcodeblock %}
{% img  /image/lock/lock6.png  '"Condition演示2"' %}

  可以看到，相对synchronized和wait，Lock和Condition的组合显然更加灵活和强大。
  一个Lock可以通过*newCondition()*同时拥有多个等待队列，互不干涉影响。阻塞队列正是利用这种方式实现放入/弹出的控制。


