---
title: ThinkingInJava-8
date: 2020-06-17 10:33:35
tags:
- ThinkingInJava
categories:
- 读书笔记
- ThinkingInJava
---

### 前言

  String应该是Java中较为特殊的对象。一方面他的使用场景非常多，其丰富的方法使得String类型的数据非常容易处理。
  另一方面，他的不可变的特性也让其区别于其他对象。怎么理解String的不可变，以及他与StringBuilder和StringBuffer的区别等问题也是在面试中会问到的问题。
<!-- more -->


### 字符串
   
   本书第十三章，围绕String，介绍了String的不可变特性，StringBuilder对某些场景下String操作的优化，及正则表达式等内容。

#### 不可变的String

  作为我们用到的最多的一个对象之一，他的不可变让其显得有点特殊。对于参数的类型可能会这样区分：基本类型，String类型，集合类型，其他对象类型。
  那么如何理解String的不可变呢？

  首先从源码上看：
{% codeblock lang:java %}
public final class String 
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // 存放字符串数据的char数组.
    private final char value[];
{% endcodeblock %}
  
  可以看到，String类从以下三个方面保证了不可变：

  1. 类不可继承，这就防止了子类的修改
  2. 数组value用final修饰，表示指向数组的引用值不可变(但是数组本身是可变的)
  3. 数组value用private修饰，避免外部访问修改数组，而String类本身也没有修改数组的操作，这就保证了数组的不变


  综上三点是保证String不可变的实现逻辑。
  然后我们看看，String的不可变为我们带来什么：
{% codeblock lang:java %}
public static void main(String[] args) {
    int i = 1;
    String s = "aaa";
    ClassChild cc = new ClassChild(i, s);
    stringTest(i, s, cc);
    logger.info("result1:{},{},{}", i, s, cc);
    stringTest2(i, s, cc);
    logger.info("result2:{},{},{}", i, s, cc);
}

public void stringTest(int i, String param, ClassChild object) {
    i = 2;
    param = "bbb";
    object.a = 2;
    object.b = "bbb";
}

public void stringTest2(int i, String param, ClassChild object) {
    i = 3;
    String newStr = "ccc";
    param = newStr;
    ClassChild cc = new ClassChild(3, "ccc");
    object = cc;
}
// output:
// result1:1,aaa,{"a":2,"b":"bbb"}
// result2:1,aaa,{"a":2,"b":"bbb"}
{% endcodeblock %}
  
  可以看到，同样是对象，传参的是引用值，String类型的原对象不变，而普通对象的原值变了。
  对于给String类型重新赋值的操作,其实可以理解为新建了一个String对象然后将引用改为指向这个新对象。就如方法*stringTest2()* 的操作一样。
  
  另一点，已经定义过的字符串都会放在字符串常量池里。如果新建一个String对象，会首先去池里找有没有现成的，有则直接指向他，没有才会创建并放入常量池，这节省了大量的内存空间。

#### StringBuilder和StringBuffer

  StringBuilder和StringBuffer都继承了抽象类AbstractStringBuilder。两者是为了在某些场景下替换String的使用，作为一种优化方案。
  他们的不同之处只在于StringBuffer的方法都用synchronized修饰，所以是线程安全的。
{% codeblock lang:java %}
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    // 存储字符的数组
    char[] value;
{% endcodeblock%}

  可以看到存放字符串的数组并不是private final，这就是StringBuiler，StringBuffer与String的最大区别，前者是可变的。
{% codeblock lang:java %}
public static void main(String[] args) {
    StringBuilder sb = new StringBuilder();
    sb.append(1);
    logger.info("result:{}", sb);
    sbTest(sb);
    logger.info("result:{}", sb);
}
public void sbTest(StringBuilder sb) {
    sb.append(2);
}
// output:
// result:1
// result:12
{% endcodeblock %}

#### 正则表达式

  正则表达式是过滤字符串和格式校验等方面极好用高效的工具。
  正则表达式通过通配符构建匹配规则，然后针对目标字符串进行匹配处理。这里就不详细叙述他的规则了，简单介绍下Java中的用法。
{% codeblock lang:java %}
String regText = "^[1-5]{1,3}$";
String checkText1 = "1111";
String checkText2 = "33";
String checkText3 = "888";
logger.info("result1:{}", Pattern.matches(regText, checkText1));
logger.info("result2:{}", Pattern.matches(regText, checkText2));
logger.info("result3:{}", Pattern.matches(regText, checkText3));
// output:
// result1:false
// result2:true
// result3:false
{% endcodeblock %}

  上面就是一个很简单使用正则校验字符串的例子。
  当我们对一些复杂的正则表达式很难快速理解的时候，可以借助正则表达式的可视化工具，这里推荐*regexper*。

### 小结
  
      对于String，我们用得很多，但不一定理解的很深。String最重要的就是理解他的不可变，会有什么效
    果，能有什么好处，什么时候用StringBuilder和StringBuffer等。
      本章只是着重记录了对String不可变的理解，其实String的源码包括StringBuilder及StringBuffer
    的源码都很值得一看。而且阅读源码才能够深入理解String。所以这里就简单的记录下，更多内容放在之后
    对其源码解读上吧。

> 下一章 [\<ThinkingInJava-9>](https://rel-fly.com/2020/06/19/ThinkingInJava9/)