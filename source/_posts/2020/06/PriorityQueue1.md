---
title: PriorityQueue(jdk1.8)
date: 2020-06-29 15:18:40
tags:
- Java容器
categories:
- Java
- Collection
- Queue
---

### PriorityQueue简介

       PriorityQueue是java.util包下实现Queue接口的非线程安全的优先级队列
       也是由数组实现，类似ArrayList通过复制数组达到扩容的操作
       其特点是可以按照自定义的元素比较器的规则输出队列元素，默认是按小到大的输出顺序
       该队列不允许插入null或不可比较的对象(没有实现Comparable接口的对象)
<!-- more -->
类定义如下：
{% codeblock lang:java %}
public class PriorityQueue<E> extends AbstractQueue<E>
    implements java.io.Serializable
{% endcodeblock %}

### 属性信息

{% codeblock lang:java %}
// 默认数组长度
private static final int DEFAULT_INITIAL_CAPACITY = 11;

// 存储元素的数组
transient Object[] queue;

// 数组长度
private int size = 0;

// 元素比较器
private final Comparator<? super E> comparator;

{% endcodeblock %}


### 构造方法
  
{% codeblock lang:java %}
// 无参构造方法
public PriorityQueue() {
    this(DEFAULT_INITIAL_CAPACITY, null);
}

// 指定长度的构造方法
public PriorityQueue(int initialCapacity) {
    this(initialCapacity, null);
}

// 指定元素比较器的构造方法
public PriorityQueue(Comparator<? super E> comparator) {
    this(DEFAULT_INITIAL_CAPACITY, comparator);
}

// 指定元素比较器和数组长度的构造方法
public PriorityQueue(int initialCapacity,
                     Comparator<? super E> comparator) {
    // Note: This restriction of at least one is not actually needed,
    // but continues for 1.5 compatibility
    if (initialCapacity < 1)
        throw new IllegalArgumentException();
    this.queue = new Object[initialCapacity];
    this.comparator = comparator;
}

// 指定集合的构造
public PriorityQueue(Collection<? extends E> c) {
// 判断集合入参的类型，初始化元素比较器
    if (c instanceof SortedSet<?>) {
        SortedSet<? extends E> ss = (SortedSet<? extends E>) c;
        this.comparator = (Comparator<? super E>) ss.comparator();
        initElementsFromCollection(ss);
    }
    else if (c instanceof PriorityQueue<?>) {
        PriorityQueue<? extends E> pq = (PriorityQueue<? extends E>) c;
        this.comparator = (Comparator<? super E>) pq.comparator();
        initFromPriorityQueue(pq);
    }
    else {
        this.comparator = null;
        initFromCollection(c);
    }
}
{% endcodeblock %}

### 核心方法

{% codeblock lang:java %}
// 元素排序算法，在初始化队列元素时，调用该方法保证队列的顺序符合规则
private void heapify() {
    for (int i = (size >>> 1) - 1; i >= 0; i--)
        siftDown(i, (E) queue[i]);
}

// 扩容方法
private void grow(int minCapacity) {
    int oldCapacity = queue.length;
    // 数组小于64则按old+1的两倍扩容，反之，old的1.5倍扩容
    int newCapacity = oldCapacity + ((oldCapacity < 64) ?
                                     (oldCapacity + 2) :
                                     (oldCapacity >> 1));
    // 数组长度控制，不超过Integer.MAX_VALUE
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    queue = Arrays.copyOf(queue, newCapacity);
}

// 元素下沉
private void siftDown(int k, E x) {
    if (comparator != null)
        siftDownUsingComparator(k, x);
    else
        siftDownComparable(k, x);
}

// 实现逻辑
private void siftDownUsingComparator(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        // 计算子节点下标
        int child = (k << 1) + 1;
        Object c = queue[child];
        int right = child + 1;
        // 取子节点较小的一个
        if (right < size &&
            comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right];
        // 与子节点比较直到小于等于子节点
        if (comparator.compare(x, (E) c) <= 0)
            break;
        queue[k] = c;
        k = child;
    }
    queue[k] = x;
}

// 元素上浮
private void siftUp(int k, E x) {
    if (comparator != null)
        siftUpUsingComparator(k, x);
    else
        siftUpComparable(k, x);
}

// 实现逻辑
private void siftUpUsingComparator(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;// 计算父节点下标
        Object e = queue[parent];
        // 每次和父节点比较，直到大于等于父节点
        if (comparator.compare(x, (E) e) >= 0)
            break;
        queue[k] = e;
        k = parent;
    }
    queue[k] = x;
}
{% endcodeblock%}

### 基本操作

{% codeblock lang:java %}
// 新增元素
public boolean offer(E e) {
    if (e == null)
        throw new NullPointerException();
    modCount++;
    int i = size;
    if (i >= queue.length)
        grow(i + 1);
    size = i + 1;
    if (i == 0)
        queue[0] = e;
    else
        siftUp(i, e);// 对新增元素做上浮操作
    return true;
}

// 弹出队首元素
public E poll() {
    if (size == 0)
        return null;
    int s = --size;
    modCount++;
    E result = (E) queue[0];// 弹出队首元素
    E x = (E) queue[s];
    queue[s] = null;// 队尾置空
    // 此时队首元素还没变
    if (s != 0)
        siftDown(0, x);// 对队尾元素做下沉操作
    return result;
}

// 删除
private E removeAt(int i) {
    // assert i >= 0 && i < size;
    modCount++;
    int s = --size;
    if (s == i) // removed last element
        queue[i] = null;
    else {
        E moved = (E) queue[s];
        queue[s] = null;
        siftDown(i, moved);// 对队尾元素做下沉操作 
        if (queue[i] == moved) {// 判断moved元素没有改变
            siftUp(i, moved);// 进行一次上浮操作
            if (queue[i] != moved)
                return moved;
        }
    }
    return null;
}
{% endcodeblock%}

  通过siftDown()和siftUp()方法可以看出，优先队列是借助数组实现了一个二叉堆。
> 二叉堆是一种特殊的堆，二叉堆是完全二元树（二叉树）或者是近似完全二元树（二叉树）。
> 二叉堆有两种：最大堆和最小堆。最大堆：父结点的键值总是大于或等于任何一个子节点的键值；最小堆：父结点的键值总是小于或等于任何一个子节点的键值。
> 二叉堆一般用数组来表示。如果根节点在数组中下标为0，下标n的元素子节点下标为2n+1和2n+2。


{% codeblock lang:java %}
PriorityQueue<Integer> queue = Queues.newPriorityQueue();
queue.add(5);
queue.add(3);
queue.add(10);
queue.add(6);
queue.add(2);
queue.add(2);
queue.add(1);
queue.add(6);
for(Integer i:queue){
    System.out.print(i);
    System.out.print(",");
}
System.out.println();
while(queue.size()>0){
    System.out.print(queue.poll());
    System.out.print(",");
}
// output:
// 1,3,2,6,5,10,2,6,
// 1,2,2,3,5,6,6,10,
{% endcodeblock %}

  上述的代码示例可以看到PriorityQueue每次弹出时都会重新整理元素顺序。

### 小结

      优先级队列的结构其实比较简单，是一个用数组实现的二叉堆来保证元素的弹出顺序。