---
title: ThreadLocal(jdk1.8)
date: 2020-07-02 15:57:56
tags:
- 多线程
categories:
- Java
- 多线程
---

### 前言

  ThreadLocal，也被称作线程本地变量，他为每一个线程创建了变量的副本，使得线程能够访问各自的变量副本，互不影响。
<!-- more -->

### 属性信息

{% codeblock lang:java %}
// 用于关联线程的线性哈希
private final int threadLocalHashCode = nextHashCode();

// 下一个哈希值，使用原子更新
private static AtomicInteger nextHashCode = new AtomicInteger();

// 连续hash计算的增长量
private static final int HASH_INCREMENT = 0x61c88647;

// 计算哈希的方法
private static int nextHashCode() {
    // 使用原子操作计算
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
{% endcodeblock %}

### 核心方法

{% codeblock lang:java %}
// 赋值方法
public void set(T value) {
    // 获取当前线程
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap实例
    ThreadLocalMap map = getMap(t);
    // 判断非空 进行初始化及赋值操作
    if (map != null)
        // 这里的this是当前ThreadLocal实例
        map.set(this, value);
    else
        createMap(t, value);
}

// 取值方法
public T get() {
    // 获取当前线程对象
    Thread t = Thread.currentThread();
    // 获取当前线程的ThreadLocalMap实例
    ThreadLocalMap map = getMap(t);
    // 判断map非空，取值
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    // map为空 进行初始化
    return setInitialValue();
}

// 获取指定线程对象的ThreadLocalMap
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

// ThreadLocalMap的初始化及赋值
void createMap(Thread t, T firstValue) {
    // 这里的this是当前ThreadLocal实例
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

// 删除
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        // 调用ThreadLocalMap的remove清理
        m.remove(this);
}
{% endcodeblock %}

  可以从set()看到，这里是将ThreadLocal实例作为map的key存储对应的值。因为ThreadLocalMap属于每个线程私有的变量，通过不同的ThreadLocal实例区分不同的变量。
  
### ThreadLocalMap
  
  ThreadLocalMap是ThreadLocal内部定义的一个key-value结构的内部类，用于存储线程内部的变量值。

{% codeblock lang:java %}
// 构造方法
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    // 这里使用ThreadLocal的哈希值计算数组下标，与HashMap的逻辑相似
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

// 通过key(ThreadLocal)获取value
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

// 清理value
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    // 手动置空要清除的项
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;
    // Rehash until we encounter null
    Entry e;
    int i;
    // 数组长度修改了，重新计算下标
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                // Unlike Knuth 6.4 Algorithm R, we must scan until
                // null because multiple entries could have been stale.
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
{% endcodeblock %}

### 原理分析

  从上述源码可以看出，每一个Thread都定义了一个ThreadLocalMap的属性用来存储自己的局部变量，map的key是ThreadLocal，通过其哈希值计算数组下标。
  不同的变量可以通过定义新的ThreadLocal存储在Thread中。
  而不同的Thread访问的永远是自己的map，互不影响。

{% img  /image/ThreadLocal/ThreadLocal1.png  '"ThreadLocal结构"' %}


### 内存溢出问题

  ThreadLocalMap的key为弱引用，但value仍然是强引用。
{% codeblock lang:java %}
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;
    // 这里的k 用的弱引用
    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
{% endcodeblock%}

  Entry的key是弱引用指向ThreadLocal实例，这样做的目的是能够及时释放ThreadLocal防止内存溢
  出。带来的问题是，value是强引用，需要手动清理，否则累积过多会导致内存溢出。
  每次使用完后，需要记得调用ThreadLocal的remove()方法，这样就能避免内存溢出的问题。

