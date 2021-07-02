---
title: 浅析AQS(jdk1.8)
date: 2020-07-14 16:52:35
tags:
- JUC
categories:
- Java
- JUC
---

### 前言

  作为JUC并法包的核心组件，AQS是学习JUC必不可少的一步，本章就来看看AQS是怎样实现同步需求的。
<!-- more -->

### state
  
  state是一个计数值，用来表示同步状态，state>0表示占用了锁，等于0则释放锁，像CountDownLatch中就是用state的值来判断是否释放锁唤醒等待线程。
{% codeblock lang:java %}
// state属性
private volatile int state;

// 返回当前的state值，因为state是volatile修饰的，所以每次都是读的主存
protected final int getState() {
    return state;
}

// 设置state值，同样基于volatile是直接写的主存
protected final void setState(int newState) {
    state = newState;
}

// 使用CAS方式修改state
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
{% endcodeblock %}

### FIFO队列

  除了使用state表示同步状态，AQS还维护了一个FIFO队列来存储等待的线程信息。

{% codeblock lang:java %}
// 队列节点Node类
static final class Node {
    // 共享标记
    static final Node SHARED = new Node();
    // 独占标记
    static final Node EXCLUSIVE = null;

    // waitStatus值，表示取消状态，线程不会再参与竞争
    static final int CANCELLED =  1;
    // waitStatus值，表示后继线程需要出列
    static final int SIGNAL    = -1;
    // waitStatus值，当前线程正在等待
    static final int CONDITION = -2;
    // waitStatus值，表示下一次的共享式同步状态的获取应无条件传播
    static final int PROPAGATE = -3;

    // 等待状态
    volatile int waitStatus;

    // 上一节点
    volatile Node prev;
    // 下一节点
    volatile Node next;

    // 节点所属线程
    volatile Thread thread;

    // 维护条件队列的节点或者存独占/共享模式的标识
    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    // 返回上一个节点，为空则抛出空指针
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    // Used to establish initial head or SHARED marker
    Node() {   
    }
    Node(Thread thread, Node mode) {     // Used by addWaiter
        // 独占/共享失败时调用addWaiter()，通过入参设置独占/共享标识
        this.nextWaiter = mode;
        this.thread = thread;
    }
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
{% endcodeblock %}

  从Node类可以看到，每一个节点都保存了对应线程信息并维护了一个waitStatus信息来判断如何操作。
  并且每一个节点都指向了他的前一节点和后一节点，可以看出这个队列是一个双向队列。

  下面再来看看AQS中是如何操作这个队列的：

{% codeblock lang:java %}
// 队列头节点 头节点如果存在，其waitStatus不会被设置为CANCELLED
private transient volatile Node head;

// 队列尾节点
private transient volatile Node tail;

// 节点插入队列
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        // 尾节点为空则初始化
        if (t == null) { // Must initialize
            // 使用CAS保证线程安全
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // 尾部插入节点
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}

// 设置头节点
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}

// 当前线程加入队列尾部
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
{% endcodeblock %}

  从节点的定义可以看到，这是一个双向队列，AQS维护了指向这个队列头尾的两个指针，每次新增的节点都是从尾部插入。

### 获取释放操作

  在看完了AQS队列结构后，接下来就来看看AQS为实现同步需求提供的一些操作方法。

#### "抽象"方法

  AQS中有几个方法并没有具体实现逻辑，这些方法都需要子类自己去实现，AQS只是定义了方法的含义。

{% codeblock lang:java %}
// 独占模式的获取
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}

// 独占模式的释放
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}

// 共享模式的获取
// 返回结果：
// 1. 负值表示失败
// 2. 0表示成功，但无法再执行获取操作
// 3. 整数表示成功，后续的获取操作也能成功，但需要检查可用性
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}

// 共享模式的释放
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}

// 
protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
}
{% endcodeblock %}


#### 模板方法

  AQS还提供了一些实现了具体逻辑的获取释放操作的模板方法。

##### 独占模式
{% codeblock lang:java %}
// 独占模式的获取
public final void acquire(int arg) {
    // 1. 调用tryAcquire()进行独占模式的获取操作
    // 2. tryAcquire() 失败，则调用addWaiter将节点加入队列
    // 3. 调用acquireQueue() 自旋等待
    // 4. 若acquireQueue() 返回结果中断过，中断当前线程
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

// 自旋等待资源释放
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋不停的请求获取操作
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;// 返回是否中断过
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                // 方法中断，记录结果，节点依然在队列中等待
                interrupted = true;
        }
    } finally {
        if (failed)
            // 失败，取消正在尝试获取的操作
            cancelAcquire(node);
    }
}

// 响应中断的独占模式的获取
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    // 线程中断，直接抛异常
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        // 逻辑与acquireQueue()类似，但线程中断会抛异常
        doAcquireInterruptibly(arg);
}

// 设置等待时间的独占模式的获取
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    // 依然会对中断响应
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
        doAcquireNanos(arg, nanosTimeout);
}

// 等待时间的独占模式获取
private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    // 根据入参计算到期时间戳
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
            // 每一次自旋判断时间是否超时
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 独占模式的释放
public final boolean release(int arg) {
    // 调用子类实现的tryRelease()方法
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            // 唤醒头节点的下一节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}
{% endcodeblock %}

##### 共享模式

{% codeblock lang:java %}
// 共享模式的获取
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        // 获取失败，进入队列自旋等待
        doAcquireShared(arg);
}

// 加入队列，自旋等待，不响应中断
private void doAcquireShared(int arg) {
    // 节点加入队列，设置为共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        // 自旋不断尝试获取
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

// 共享模式的释放
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

// 释放逻辑
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
{% endcodeblock %}

  共享模式的获取同样有响应中断和超时等待两种，逻辑大致与独占式类似就不一一列举。

### 条件队列

  除了Node内部类，AQS中还有个ConditionObject内部类，其内部维护了针对条件队列的方法。
  条件队列的元素同样是Node对象，但借助其nextWaiter属性维护。

#### 类定义

{% codeblock lang:java %}
// 实现了Condition接口
public class ConditionObject implements Condition, java.io.Serializable
{% endcodeblock %}

#### 主要方法

{% codeblock lang:java %}
// 从条件队列移除，进入等待队列
public final void signal() {
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);
}

// 条件等待
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    // 进入条件队列
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断是否处于同步队列中
    while (!isOnSyncQueue(node)) {
        // 阻塞当前线程
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
{% endcodeblock %}

### 小结

      AQS实现同步需求的基础就是依赖一个FIFO的队列。当获取操作失败时，就会将当前线程加入队
    列，用自旋的方式不断尝试获取直到成功。
      本章只是从AQS的数据结构及获取释放操作来了解他是如何实现同步需求的，其实还有许多底层
    实现逻辑值得分析，这里因为篇幅原因就不一一分析了。
      AQS的设计确实非常全面，考虑到了各种可能，不过也因此，单看他的源码会觉得难以联系具体
    场景去深入理解。个人认为最好结合JUC中依赖AQS实现的各种工具类来理解会更透彻些。