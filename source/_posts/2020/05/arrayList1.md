---
title: ArrayList(jdk1.8)
date: 2020-05-26 09:46:14
tags:
- Java容器
categories:
- Java
- Collection
- List
---

### ArrayList简介

       ArrayList是java.util包下实现List接口的非线程安全的动态数组类
       优点：基于数组结构，可通过下标快速定位，查询效率快；新增涉及扩容判断，在不扩容
             的前提下，效率也较快
       缺点：删除，扩容操作基于数组的复制，代价较高
<!-- more -->
类定义如下：
{% codeblock lang:java %}
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
{% endcodeblock %}

#### 属性信息

{% codeblock lang:java %}
// 默认的初始容量(存储ArrayList数据的数组长度)
private static final int DEFAULT_CAPACITY = 10;

// 空实例的共享空数组
private static final Object[] EMPTY_ELEMENTDATA = {};

// 有默认大小的空实例的共享数组，与 EMPTY_ELEMENTDATA 区分开，是为了在第一次add的时候做判断
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

// 存储ArrayList元素的数组缓冲区
// 这里使用transient避免elementData的序列化，是为了防止序列化对ArrayList开辟
// 的预留空间处理，节省空间和时间
transient Object[] elementData;

// ArrayList包含的元素个数
private int size;

// 结构变化(新增，删除，扩容)的次数，是Fail-Fast的判断标准
protected transient int modCount = 0;

// 数组可分配的最大值
// -8 根据源码注释是由于虚拟机限制，尝试分配更大空间会容易导致
// OutOfMemoryError: Requested array size exceeds VM limit
private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
{% endcodeblock %}

#### 常用方法

1. 构造方法
{% codeblock lang:java %}
// 指定初始化容量的构造方法
// 可以看到容量为0的会将elementData指向 EMPTY_ELEMENTDATA 即实例共享的空数组
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity];
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: "+
                                           initialCapacity);
    }
}

// 无参构造方法，这里elementData初始化为 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，表示容量默认10
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 构造一个包含指定集合的list
public ArrayList(Collection<? extends E> c) {
    elementData = c.toArray();
    if ((size = elementData.length) != 0) {
        // c.toArray might (incorrectly) not return Object[] (see 6260652)
        if (elementData.getClass() != Object[].class)
            elementData = Arrays.copyOf(elementData, size, Object[].class);
    } else {
        // replace with empty array.
        this.elementData = EMPTY_ELEMENTDATA;
    }
}
{% endcodeblock %}

2. 其他方法
{% codeblock lang:java %}
// 数组容量的判断
private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 当数组为空且有默认容量时，取两者大的一个
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}


// 容量更新的判断
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;
    // overflow-conscious code
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

// 增加容量的操作
private void grow(int minCapacity) {
    // overflow-conscious code
    int oldCapacity = elementData.length;
    // 扩容为旧容量的1.5倍，x >> 1(向右位移1位) == x/2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    elementData = Arrays.copyOf(elementData, newCapacity);
}

// 容量超出最大值的处理
private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    // 这里并不禁止使用Integer.MAX_VALUE
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE :
        MAX_ARRAY_SIZE;
}

// 新增时的下标校验
private void rangeCheckForAdd(int index) {
    if (index > size || index < 0)
        throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
}
{% endcodeblock %}

3. 基本操作方法
{% codeblock lang:java %}
// 新增方法
public boolean add(E e) {
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}

// 指定下标的新增方法
public void add(int index, E element) {
    rangeCheckForAdd(index);
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    System.arraycopy(elementData, index, elementData, index + 1,
                     size - index);
    elementData[index] = element;
    size++;
}

// 元素的替换
public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue;
}

// 删除指定下标的元素
public E remove(int index) {
    // 判断index是否大于size，大于则抛出IndexOutOfBoundsException
    rangeCheck(index);
    modCount++;
    E oldValue = elementData(index);
    int numMoved = size - index - 1;
    if (numMoved > 0)
        // 这里涉及对数组的复制
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // clear to let GC do its work
    return oldValue;
}

// 删除第一次出现的指定元素
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                // 跳过边界校验的删除操作
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) {
                fastRemove(index);
                return true;
            }
    }
    return false;
}

// 清空操作
public void clear() {
    modCount++;
    // clear to let GC do its work
    for (int i = 0; i < size; i++)
        elementData[i] = null;
    size = 0;
}

// 新增一个集合的元素
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew);  // Increments modCount
    // 使用数组复制操作
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
{% endcodeblock %}

### 小结

      这里只是列出了ArrayList部分常用方法的源码。ArrayList相对比较简单容易理解，但通过对源码的解读，
    依然能有一些收获:
    1. 使用时需要注意其基于数组复制的操作，例如扩容，删除等。
    2. 数组大小默认是10，每次扩容会变为原来的1.5倍大小。建议按照实际场景提前设置默认容量，可以避免扩
       容操作
    3. 可以看到对于数组边界之类的判断，源码中做得很充分，能够有效保证代码的安全性
    4. 对无用的数组元素及时赋值null，使其能够被GC处理
    5. 方法封装的一些处理，根据不同场景的需要封装方法，而不是一味的将相同的代码封装为一个方法，使得代
       码更容易被理解