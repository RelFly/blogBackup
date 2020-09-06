---
title: synchronized相关-加锁过程解析
date: 2020-08-27 21:17:19
tags:
- 多线程
categories:
- 多线程
---

### 前言

  [上一章](https://rel-fly.com/2020/08/15/synchronized1/)总结了synchronized相关的对象头和锁机制，这一章就锁的加锁及升级过程总结一下。
<!-- more -->

### 偏向锁
  
  首先看一下偏向锁的几个例子：
{% codeblock lang:java %}
// 未偏向线程，可偏向状态
public void testOne() throws InterruptedException {
    String object = "ob";
    System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized3.png  '"testOne()输出结果"' %}

  上例虽然并没有使用synchronized，但输出结果可以看到，锁标志仍然是'101'，不过表示偏向线程标识的位都是零说明此时并未偏向任何线程。
  这种情况可以理解为此时锁处于“可偏向但还未偏向”的状态。

{% codeblock lang:java %}
// 偏向锁偏向占用线程
public void testTwo() throws InterruptedException {
    String object = "hello";
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized4.png  '"testTwo()输出结果"' %}

  testTwo()就展示了一个常见的单一线程占用锁资源的场景，可以看到开始锁标识是“可偏向但还未偏向”的状态。
  而当进入同步代码块后，锁就偏向了当前线程。

#### 加锁过程

  1. 首次占用
    首先，线程会在自己栈帧中存储一份对象头信息，作为CAS操作的期望值，这块空间被称为*Lock Word* 。如果CAS执行成功，即当前对象是无锁状态，则直接偏向。
  2. 重入
    当尝试占用时，发现偏向线程即为当前线程，则会在线程栈帧中再添加一个*Lock Word* ，但其存的对象头信息为空，与首次存的*Lock Word* 区分开。
    这个新增的*Lock Word* 会作为重入的统计标识，每当退出一次则会清空一个*Lock Word* 表示一次释放。
  3. 竞争
    如果请求占用的锁已偏向且偏向的不是的当前线程，则会判断被偏向的线程是否存活或已释放锁：
    a)线程存活且还在占用，则进行轻量级锁的升级
    b)线程未存活或不再占用，则会先修改对象头为无锁状态，然后做轻量级锁的升级
  

  偏向锁不会主动释放，当偏向后，对象头就会一直处于偏向的状态，直到下一个来竞争的线程使其升级。
{% codeblock lang:java %}
public void test() throws InterruptedException {
    String object = "hello";
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(1000);
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread2:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(1000);
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized8.png  '"thread1占用及结束后的对象头信息"' %}

{% img  /image/synchronized/synchronized9.png  '"thread2占用及结束后的对象头信息"' %}

### 轻量级锁

  出现轻量级锁会有两种情况，一种是正常的抢占偏向锁导致的升级，第二种就是从无锁状态直接到轻量级锁状态。
{% codeblock lang:java %}
public void testThree() throws InterruptedException {
    String object = "123";
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    // thread1先抢占到锁
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(100);
    // thread2去竞争锁，此时导致偏向锁升级
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread2:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized5.png  '"testThree()输出结果"' %}

{% codeblock lang:java %}
// 无锁直接升级到轻量级锁
public void testFour() throws InterruptedException {
    String object = "Object";
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized6.png  '"testFour()输出结果"' %}

#### 加锁过程
  
  轻量级锁的加锁过程与偏向锁有点类似，也是基于*Lock Word* 去做CAS操作。
  1. 首先在*Lock Word* 中存储一个空的*Mark Word* 对象，其中会存储指向锁对象的指针
     然后将这个空的对象头作为期望值去做CAS操作

  2. CAS成功：表示此时是无锁状态，直接占用，对象头中存储指向*Lock Word* 的指针，此时两者互相指向
     
  3. CAS失败：会先判断是否重入，是则在栈帧中增加一个*Lock Word* 表示重入，否则进行锁升级


### 重量级锁

{% codeblock lang:java %}
public void testFive() throws InterruptedException {
    String object = "hello";
    System.out.println("master:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    threadPoolExecutor.execute(() -> {
        synchronized (object) {
            System.out.println("thread2:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(10000);
}
{% endcodeblock %}

{% img  /image/synchronized/synchronized7.png  '"testFive()输出结果"' %}

#### 重量级锁结构
  
  重量级锁与偏向锁，轻量级锁就有很大的差别了，当升级为重量级锁后，未占用的线程是处于阻塞状态的，他的实现是基于*Monitor* 的结构。

  *Monitor* 中维护了一个阻塞队列用来存储被阻塞的线程信息，还有一个等待队列，当线程调用wait()方法后，就会释放锁进入等待队列。
  {% img  /image/synchronized/synchronized10.png  '"重量级锁结构"' %}

### 总结

      为了清楚的理解synchronized中锁升级及加锁流程，也很是看了很久博客和文档。算是大概理出了
    加锁流程，不过还有很多细节没有弄清楚。
    1. 测试时，定义的String对象内容如果是Object类似的关键字，就会跳过偏向锁，直接升级为轻量
       级锁。(这种情况我了解的是当对象头的HashCode有值导致偏向锁偏向标识的空间被占用无法偏向
       才会出现)
    2. 偏向锁与轻量级锁都会在线程栈帧中Lock Word存储对象头信息用作CAS操作，这里我理解的是
       用无锁状态的对象头当作期望值去尝试CAS，这样当对象头被释放时就能CAS成功
    3. 理解synchronized的使用其实只要抓住他锁定的是对象就够了，关于锁升级主要是了解什么操作
       会导致重量级锁出现，以尽量避免重量级锁

> [synchronized相关-对象头&锁机制](https://rel-fly.com/2020/08/15/synchronized1/)
