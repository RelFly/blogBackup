---
title: synchronized的实现原理
date: 2020-08-15 20:49:14
tags: 
- 多线程
categories:
- Java
- 多线程
---

### 前言
  
  之前面试中有被问到Java锁机制相关的问题，本章就借助synchronized的实现原理来总结一下相关知识点。
<!-- more -->

### 对象头

#### 对象头的结构
  synchronized锁住的目标是Java中的对象，那么他到底依赖于什么控制的呢？
  答案就是----对象头。
  所以先来了解下对象头是啥样的：
> 一般对象的对象头由两部分组成
> 1.klass pointer(指针，指向其类元数据的信息),
> 2.Mark word(存储对象的运行时数据，包括哈希码，锁状态等)
> 如果是数组对象则还存有array length信息

#### Mark word
  对象头的三个结构中，与synchronized关联紧密的就是第二部分 Mark word。
  因为他维护着对象的锁标记信息。

{% img  /image/xxxx/xxxxx.png  '"xxxx"' %}
{% img  /image/synchronized/synchronized1.png  '"锁状态标记"' %}

### 锁升级

  对象头中记录了对象不同的锁状态，当触发了对应条件时就会引起其状态的变化，这个过程被称作锁升级。

  1. 无锁状态
     锁的初始状态
  2. 偏向锁
     偏向锁与无锁状态的lock标记一样都是01，可以理解偏向锁是一种特殊的无锁状态，是无锁到有锁的过渡。
     当有线程获取到锁资源时就会升级为偏向锁，并在对象头中记录偏向的线程标识，也就是获取到锁资源的线程。
     >引入偏向锁是因为在一个线程多次获取同一个锁的场景下，如果每次都按照竞争锁的方式去操作未免会造成平白的消耗。
     >而偏向锁记录了当前拥有锁的线程的标识信息，当同一线程多次去获取该锁时可以直接依据该标识判断，从而减小获取锁的消耗。
  3. 轻量级锁
     偏向锁是无竞争场景下的，假设这样一种情况：
     thread1先获取到了某个对象的锁，thread2慢了一步，获取失败。
     那么此时如何处理thread2？
     这里的设计是按照乐观的设想，thread1会很快释放锁资源，所以将偏向锁升级为轻量级锁，让thread2自旋不断尝试获取锁资源。
  4. 重量级锁
     轻量级锁是在乐观设想的前提下让获取失败的线程自旋等待。
     但如果考虑到最坏的情况，thread1因为某些原因占用时间超出了可接受的范围，如果让thread2一直自旋，甚至后面又来了N个
     线程也在自旋，这样就浪费了大量的CPU资源，显然是不合理的。
     所以当线程自旋一定次数还未获得锁资源时，轻量级锁就会升级为重量级锁，让这些尝试获取锁资源的线程通通阻塞，等待锁被释放后再唤醒他们。

#### 锁升级的具体过程
  
  锁升级的逻辑其实比较好理解，其目的就是为了优化及提高加锁解锁的性能。
  但这个过程具体是如何操作的呢？
  前文说到，Mark word中存储了对象的锁标记，线程占用锁资源的本质就是修改了对象头Mark word的相关信息。
  而为了保证多个线程去竞争同一个锁的并发场景的安全，这里的修改都采用了CAS操作。
  
  无锁升级为偏向锁存在于无竞争场景，这里就不再多做赘述。
         

#### jol打印对象头

{% codeblock lang:java %}
// 偏向锁升级为轻量级锁
public void testJol() throws InterruptedException {
    String object = "object";
    // 此时是无锁状态
    System.out.println("master1:" + ClassLayout.parseInstance(object).toPrintable());
    threadPoolExecutor.execute(() -> {
        // 开辟新线程，仍是无锁状态
        System.out.println("thread1-before:" + ClassLayout.parseInstance(object).toPrintable());
        synchronized (object) {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 升级为轻量级锁
            System.out.println("thread1:" + ClassLayout.parseInstance(object).toPrintable());
        }
    });
    Thread.sleep(3000);
    // thread1释放锁，还原到无锁状态
    System.out.println("master2:" + ClassLayout.parseInstance(object).toPrintable());
    Thread.sleep(10000);
}
{% endcodeblock%}

### 小结