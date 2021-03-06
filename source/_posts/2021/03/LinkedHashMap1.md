---
title: LinkedHashMap(jdk1.8)
date: 2021-03-04 20:26:54
tags:
- Java容器
categories:
- Java
- Map
---

### LinkedHashMap简介
   
     LinkedHashMap是HashMap的子类，基于HashMap的数据结构，维护了一个双向链表。
     所以相对HashMap来说，LinkedHashMap可以保证取数的顺序。

<!-- more -->

类定义如下
{% codeblock lang:java %}
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{% endcodeblock%}

### 基础属性
{% codeblock lang:java %}
// 节点对象
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
// 链表的头指针和尾指针
transient LinkedHashMap.Entry<K,V> head;
transient LinkedHashMap.Entry<K,V> tail;
// 排序方式的设置 true表示访问顺序 false表示插入顺序
final boolean accessOrder;
{% endcodeblock%}

### put操作
  
  LinkedHashMap继承了HashMap，他的put方法没有重写，直接使用的HashMap的put方法。
  但是，LinkedHashMap需要维护一个链表结构，所以在重写newNode()方法。

{% codeblock lang:java %}
// 创建新节点
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p = 
    	new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p);
    return p;
}
// 维护链表关系
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
{% endcodeblock%}
   
   LinkedHashMap的put操作逻辑与HashMap完全一样，只是在新建节点时通过重写方法，增加了链表维护的逻辑。

### get操作
   
   get操作与put操作不同，LinkedHashMap直接重写了父类的方法，并在其中增加了取数顺序的判断及处理逻辑。

{% codeblock lang:java %}
public V get(Object key) {
    Node<K,V> e;
    // 使用HasMap的getNode方法获取节点对象
    if ((e = getNode(hash(key), key)) == null)
        return null;
    // 访问顺序的处理
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
// 将节点移动到尾部
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    // 判断入参节点是否尾部
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e, 
        b = p.before, 
        a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
{% endcodeblock %}

### 示例
{% codeblock lang:java %}
// 按照插入顺序访问
public void linkedHashMapTest() {
    LinkedHashMap<Integer, Integer> linkedHashMap = new LinkedHashMap<>();
    Set<Integer> set = linkedHashMap.keySet();
    linkedHashMap.put(1, 1);
    linkedHashMap.put(2, 2);
    linkedHashMap.put(22, 22);
    for (Integer key : set) {
        logger.info("key:{}", key);
    }
    linkedHashMap.get(2);
    for (Integer key1 : set) {
        logger.info("key1:{}", key);
    }
}
{% endcodeblock %}
结果输出：
{% img  /image/LinkedHashMap/LinkedHashMap1.png  '"结果展示"' %}

  get方法并不会影响取值的顺序，永远按照插入顺序取出数据。

{% codeblock lang:java %}
// 按照访问顺序
public void linkedHashMapTest() {
    LinkedHashMap<Integer, Integer> linkedHashMap1 = new LinkedHashMap<>(16, (float) 0.75, true);
    linkedHashMap.put(1, 1);
    linkedHashMap.put(2, 2);
    linkedHashMap.put(22, 22);
    for (Integer key : set) {
        logger.info("key1:{}", key);
    }
    linkedHashMap1.get(2);
    for (Integer key : set) {
        logger.info("key2:{}", key);
    }
    linkedHashMap1.get(22);
    for (Integer key : set) {
        logger.info("key3:{}", key);
    }
}
{% endcodeblock %}
结果输出：
{% img  /image/LinkedHashMap/LinkedHashMap2.png  '"结果展示"' %}

  当通过构造方法将accessOrder设置为true时，get方法除了查到节点，还会将所查到的节点放入到链表队尾，实现访问顺序的维护。


### 总结

       LinkedHashMap大部分内容都与HashMap一样，其通过重写部分方法，在HashMap的操作逻辑基础上，穿插了
     链表的操作逻辑。
       另外，LinkedHashMap还提供了访问顺序的维护，逻辑很简单，每一次get取数时，就将这个数据所属节点放
     到链表队尾，这样遍历数据时，就是按照最近最少使用的顺序输出。也是一种LRU算法的实现思路。





