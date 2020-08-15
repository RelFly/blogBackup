---
title: HashMap(jdk1.8)
date: 2020-05-30 20:04:27
tags:
- Java容器
categories:
- Java
- Map
---

### HashMap简介

      HashMap是java.util包下的实现Map接口的非线程安全集合，其基于数组+链表|红黑树的结构，以键值对的
    形式存储数据
    特点：
    1. 基于key-value的数据结构可以通过key直接定位value值，查询较快，key值可为null
    2. 使用hash值确定数组下标，所以遍历顺序不是输入顺序
    3. 扩容操作涉及重新计算hash值，对性能影响较大，需尽量避免
<!-- more -->
类定义如下

{% codeblock lang:java %}
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable
{% endcodeblock %}

### 源码部分解析

#### 属性信息

{% codeblock lang:java %}
// ------------------设置的默认值-----------------------
// 数组的默认容量  (需为2的n次方，涉及扩容时的计算)
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;

// 数组的最大容量 2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认的加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 树化的阈值
static final int TREEIFY_THRESHOLD = 8;

// 取消树化的阈值
static final int UNTREEIFY_THRESHOLD = 6;

// 树化时的数组容量最小值 
// 当容量小于该值时应考虑增加数组容量而不是将链表变化为树
static final int MIN_TREEIFY_CAPACITY = 64;

// -------------------基本属性---------------------------
// 数组
transient Node<K,V>[] table;

// 键值对集合
transient Set<Map.Entry<K,V>> entrySet;

// 集合中键值对的数量
transient int size;

// 结构变化的计数
transient int modCount;

// 键值对数量的阈值  计算方式 capacity * loadFactor
int threshold;

// 加载因子
final float loadFactor;
{% endcodeblock %}

#### 构造方法

{% codeblock lang:java %}
// 初始化数组容量和加载因子的构造方法
public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    this.loadFactor = loadFactor;
    // 键值对数量的阈值 为数组容量的两倍  tableSizeFor(int cap) 返回cap的两倍
    this.threshold = tableSizeFor(initialCapacity);
}

// 初始化数组容量 
public HashMap(int initialCapacity) {
    // 使用默认的加载因子初始化
    this(initialCapacity, DEFAULT_LOAD_FACTOR);
}

// 使用默认值初始化一个空的hashmap
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}

// 用指定的map集合初始化
public HashMap(Map<? extends K, ? extends V> m) {
    this.loadFactor = DEFAULT_LOAD_FACTOR;
    putMapEntries(m, false);
}
{% endcodeblock %}

#### 数组下标的计算

{% codeblock lang:java %}
// 计算hash值
static final int hash(Object key) {
    int h;
    // key为null就取0，否则用 h 异或 h 无符号右移16位
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// 计算数组下标 n表示数组长度 & 按位与
(n - 1) & hash
{% endcodeblock %}

关于以上计算的具体分析，可以参考美团技术团队的文章<Java 8系列之重新认识HashMap>

#### 基本操作

##### get方法

{% codeblock lang:java %}
// 对外暴露的方法
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

// 内部实现逻辑
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    // 数组下标的计算 (n-1) & hash
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        // 定位数组位置后，先判断头节点是否匹配
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
        if ((e = first.next) != null) {
            // 判断如果是树结构，走树的遍历
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            // 链表的遍历
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
// ---------------------------内部类 TreeNode 的方法-----------------------------------
// 获取符合条件的树节点
final TreeNode<K,V> getTreeNode(int h, Object k) {
    // 从根节点开始找
    return ((parent != null) ? root() : this).find(h, k, null);
}

// 树节点遍历比对
final TreeNode<K,V> find(int h, Object k, Class<?> kc) {
    TreeNode<K,V> p = this;
    do {
        int ph, dir; K pk;
        TreeNode<K,V> pl = p.left, pr = p.right, q;
        // 查找节点hash值小于当前值，转向左子节点
        if ((ph = p.hash) > h)
            p = pl;
        // 反之，转向右子节点
        else if (ph < h)
            p = pr;
        // 直接匹配，返回当前节点
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        // 排除本节点且无法定位下一步的情况，看左右子节点是否有为空的
        else if (pl == null)
            p = pr;
        else if (pr == null)
            p = pl;
        // 左右节点都不为空的前提下，比较key值来确定下一步去左边还是右边
        else if ((kc != null ||
                  (kc = comparableClassFor(k)) != null) &&
                 (dir = compareComparables(kc, k, pk)) != 0)
            p = (dir < 0) ? pl : pr;
        // 前面的判断都没得出结果
        // 递归遍历右子树
        else if ((q = pr.find(h, k, kc)) != null)
            return q;
        // 右子树没找到结果，就找左子树
        else
            p = pl;
    } while (p != null);
    return null;
}
{% endcodeblock %}

##### put方法

{% codeblock lang:java %}
// 对外暴露的方法
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// 具体实现逻辑
// @param onlyIfAbsent  如果为true，不更改现有值
// @param evict 如果为false table处于创建模式
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        // 初始化table
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        // 计算下标 (n-1) & hash, 若下标对应的值为null，则创建一个Node对象赋值
        tab[i] = newNode(hash, key, value, null);
    else {
        // 首节点 p 不为null
        Node<K,V> e; K k;
        // 比较hash值和key值
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            // 如果是红黑树结构，走树的逻辑
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 链表的遍历赋值
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        // 判断链表长度，是否需要树化
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        // key存在的处理
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 扩容的判断
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}

// 树化的逻辑
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index; Node<K,V> e;
    // 如果数组为空或长度太小，不会选择树化，而是扩容
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize();
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);// 替换节点对象
            if (tl == null)
                hd = p;
            else {
                p.prev = tl;
                tl.next = p;
            }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            // 调用TreeNode的方法完成树化
            hd.treeify(tab);
    }
}

// 扩容方法
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    // 原数组不为空的前提
    if (oldCap > 0) {
        // 阈值判断
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 不超出阈值的前提下，扩容为2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        // 计算新的键值对阈值
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // table扩容 重新计算数组下标
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 树结构的处理
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

// -----------------------------内部类 TreeNode 的方法-----------------------------------------
// 树结构的put方法
final TreeNode<K,V> putTreeVal(HashMap<K,V> map, Node<K,V>[] tab, int h, K k, V v) {
    Class<?> kc = null;
    boolean searched = false; // 是否遍历过
    TreeNode<K,V> root = (parent != null) ? root() : this;
    for (TreeNode<K,V> p = root;;) {
        int dir, ph; K pk;
        // 查找key是否在树中存在
        if ((ph = p.hash) > h)
            dir = -1;
        else if (ph < h)
            dir = 1;
        else if ((pk = p.key) == k || (k != null && k.equals(pk)))
            return p;
        else if ((kc == null && (kc = comparableClassFor(k)) == null) ||
                 (dir = compareComparables(kc, k, pk)) == 0) {
            if (!searched) {
                TreeNode<K,V> q, ch;
                searched = true; // 避免后续循环再走到此逻辑中
                // 递归find方法，分别遍历左右子树
                if (((ch = p.left) != null && (q = ch.find(h, k, kc)) != null) ||
                    ((ch = p.right) != null && (q = ch.find(h, k, kc)) != null))
                    return q;
            }
            dir = tieBreakOrder(k, pk);
        }
        TreeNode<K,V> xp = p;
        // 新建树节点
        if ((p = (dir <= 0) ? p.left : p.right) == null) {
            Node<K,V> xpn = xp.next;
            TreeNode<K,V> x = map.newTreeNode(h, k, v, xpn);
            if (dir <= 0)
                xp.left = x;
            else
                xp.right = x;
            xp.next = x;
            x.parent = x.prev = xp;
            if (xpn != null)
                ((TreeNode<K,V>)xpn).prev = x;
            moveRootToFront(tab, balanceInsertion(root, x));
            return null;
        }
    }
}

// 树化具体实现
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;
        x.left = x.right = null;
        // 确定根节点 红黑树根节点为黑色
        if (root == null) {
            x.parent = null;
            x.red = false;
            root = x;
        }
        // 左右子节点的判定，与get，put中的判断类似
        else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null &&
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    dir = tieBreakOrder(k, pk);
                TreeNode<K,V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    // 平衡插入 暂时没完全看懂 这里就贴出来
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 保证树的头节点在table中
    moveRootToFront(tab, root);
}
{% endcodeblock %}

### 小结

       HashMap相对ArrayList复杂许多，涉及红黑树的操作逻辑都很值得琢磨。因为篇幅问题，只从源码中摘出
    我认为较常用的方法(get，put)和一些重要的逻辑。分析源码逻辑确实受益良多，一些想法中直接简单的逻辑
    ，为了保证代码的健壮性，实现起来往往就会复杂许多。例如涉及红黑树的一些处理，做了很多判断，遍历，
    递归之类的，一开始感觉有些重复了，思考良久之后才理解为什么。这里照旧小结一下：
    1. HashMap中涉及了较多的位运算，异或，按位与之类的。因为实际工作中较少使用，导致对这些位运算都比
    较陌生了，需要对位运算复习一下
    2. 源码中许多方法都是使用了局部变量去处理，比如红黑树的判断，都是在方法中定义变量指向树的根节点
    或左右子节点再去处理。我理解的原因第一是局部变量能在方法结束后被回收，第二应该是避免直接操作导致
    原对象的错误更改。(虽然有时候这样的处理增加了代码阅读的难度···)
    3. 红黑树的相关操作(TreeNode的一些方法)，虽然有些方法没完全看懂，但是可以作为树结构操作的一个参考
