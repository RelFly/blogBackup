---
title: jvm知识点梳理
date: 2020-06-28 09:22:11
tags:
- JVM
categories:
- JVM
---

### 前言

  本章主要对JVM主要的知识点进行梳理总结。
<!-- more -->

### 运行时区域

  1. 程序计数器：线程私有，是当前线程执行的字节码的行号指示器，决定命令的执行顺序
  2. Java虚拟机栈：线程私有，生命周期与线程相同，其描述的是Java方法的内存模型
     *每个方法执行时，都会创建一个栈帧，存储局部变量表，操作数栈，动态链接，方法出口等信息。*
     *方法的从执行到完成，对应着栈帧在虚拟机栈中入栈和出栈的过程。*
  3. 本地方法栈：类似于虚拟机栈，所不同的时，本地方法栈服务的对象时Native方法
  4. 堆：几乎所有的对象实例都在堆中分配内存，也是垃圾收集器管理的主要区域
  5. 方法区：存储已被虚拟机加载的类信息，常量，静态常量，即时编译器编译后的代码等数据

### 对象存活判断

  1. 引用计数法：在对象中添加一个引用计数器，每当被引用时则+1，引用失效就-1，当为0时表示对象不会再被使用，其缺陷是没法解决相互循环引用的问题
  2. 可达性分析：将一系列的对象定义为"GC Roots"，然后分析一个对象有没有直达"GC Roots"的引用链，没有则表示对象没有被使用
     可作为"GC Roots"的对象有以下几种：
     * 虚拟机栈，栈帧中局部变量表中引用的对象
     * 方法区中类静态属性引用的对象
     * 方法区中常量引用的对象
     * 本地方法栈中JNI(Native方法)引用的对象

### 引用类型
  
  1. 强引用：普遍存在类似"Object o = new Object()"这种，垃圾收集器不会回收掉强引用还存在的对象
  2. 软引用：软引用关联的对象会在系统将要发生内存溢出前进行第二次回收，可通过SoftReference实现
  3. 弱引用：弱引用关联的对象只能生存到下一次垃圾回收之前，可通过WeakReference实现
  4. 虚引用：不影响对象的生存时间，只会在其被回收时收到一个系统通知，可通过PhantomReference实现

### 垃圾收集算法

  1. 标记-清除算法
     先标记需要回收的对象，然后再执行回收操作。缺点是效率不高，且会产生大量的内存碎片，可能导致大对象进入时引发不必要的垃圾回收
  2. 复制算法
     将内存分为两块，每次只使用其中一块。当执行回收操作时，将存活对象复制到空的那块上，然后清空使用过的，效率较高，但空间浪费了
  3. 标记-整理算法
     以标记-清理算法为基础，在回收前将存活对象向一端移动，然后清理掉边界以外的内存，保证内存的连续性
  4. 分代收集算法
     在实际虚拟机中采用分代收集算法，根据对象的存活周期划分为年轻代和老年代，根据各个年代的特点采取不同的收集算法
     * 年轻代
       采用复制算法，以HotSpot为例，空间按8:1:1的比例分为Eden，及两个Survivor共三个空间。每次回收时将Eden和Survivor中存活对象复制到空着的那个Survivor中，然后对其内存清理
       如果空着的Survivor没有足够空间存放，这些对象就会进入老年代
     * 老年代
       由于老年代的对象存活率高，通常采用标记-清理或者标记-整理来进行回收

#### 对象分配及回收条件
  
  对象优先在Eden中进行分配，若Eden中没有空间可进行分配就会触发MinorGC。

  大对象直接进入老年代，可通过-XX:PretenureSizeThreshold参数设置这个大对象的判断阈值。

  长期存活的对象也会进入老年代，每经历过一次MinorGC还存活的对象，其年龄就增加一岁(虚拟机为每一个对象定义了一个年龄计数器)，当达到一定年龄时就会进入老年代(默认15)。
  年龄阈值可通过参数-XX:MaxTenuringThreshold控制。
  
  当老年代空间不足时就会触发Full GC，Full GC对性能影响较大，应尽量避免。


### JDK命令行工具
  
  1. jps:虚拟机进程状态查询，可以显示正在运行的虚拟机进程及LVMID等信息
  2. jstat：虚拟机统计信息查询，现实虚拟机中各空间的使用信息，GC的统计信息等
  3. jinfo：实时的查看和调整虚拟机各项参数
  4. jmap：Java内存映像工具，可用于生成堆转储快照
  5. jhat：针对jmap生成的快照文件进行分析，可以在浏览器中查看分析结果
  6. jstack：生成虚拟机当前时刻的线程快照