---
title: ThinkingInJava-3
date: 2020-06-07 15:37:47
tags:
- ThinkingInJava
categories:
- 读书笔记
- ThinkingInJava
---

### 前言
  
  来到本书第五，六章，在介绍完Java操作符及一些关键字后，开始介绍一些机制。如：类的初始化，垃圾回收清理，访问权限等。这些都是有助于我们更好了解代码执行过程的知识点。
<!-- more -->

### 初始化与清理
  
  本书的第五章，主要介绍了构造器，方法的重载，类初始化的顺序及对象的回收等。关于对象的回收想放到Java虚拟机的总结中，所以这里没有记录。主要就方法重载和类初始化顺序记录一下。

#### 涉及基本类型的重载
  
  方法的重载是多个方法名一样，参数列表不一样的方法。参数列表不一样包括类型和顺序，例如

{% codeblock lang:java %}
void method(int a, String b) {
 // ·····
}
int method(String a, int b) {
 // ·····
}
{% endcodeblock %}

  而如果涉及基本类型的重载，例如：

{% codeblock lang:java %}
  method(5);// method1
  method(5f);// method1
  method(5d);// method2
  method('1');// method1
  // method1
  public void method(float f) {
      System.out.println("float");
  }
  // method2
  public void method(double d) {
      System.out.println("double");
  }
{% endcodeblock %}
  
      当入参为基本类型时，没指定具体类型的情况下，常量数值会被当作int处理。而如果这时没有对应int类型入
    参的方法，就会提升数据类型(例如这里的method(5)就提升为float类型)。而char类型则是当没有对应类型的
    方法时，被当作int处理。

#### 初始化的顺序
  
  第二章中有提到static关键字。而在一个类中，如果有静态代码块等，其初始化顺序是怎样的呢？

{% codeblock lang:java %}
public class ClassParent {
    private static int a;

    static {
        a = 1;
        System.out.println("静态代码块");
    }

    {
        System.out.println("代码块");
    }

    public static void method() {
        System.out.println("静态方法");
    }

    ClassParent() {
        System.out.println("构造方法");
    }
}

public class ClassChild extends ClassParent{
    private static int b;

    static{
        b = 1;
        System.out.println("child-静态代码块");
    }

    {
        System.out.println("child-代码块");
    }
 
    public static void function() {
        System.out.println("child-静态方法");
    }

    ClassChild() {
        System.out.println("child-构造方法");
    }
}
{% endcodeblock %}

   如上两个类，当创建类的实例或者直接调用类的静态方法，他的初始化顺序如下：

{% codeblock lang:java %}
ClassChild cc = new ClassChild();
// 输出为：
//静态代码块
//child-静态代码块
//代码块
//构造方法
//child-代码块
//child-构造方法

ClassChild.function();
// 输出为：
//静态代码块
//child-静态代码块
//child-静态方法
{% endcodeblock %}

      可以看出，静态代码块总是优先加载，在有父类的情况下，也是按照父类->子类的顺序先把静态代码块加载
    完。然后才是加载各自的代码块和构造方法。而直接调用静态方法，也会先初始化父类及子类的静态代码块。但
    因为没有涉及类的实例化，所以不会有代码块和构造方法的初始化。

### 访问权限控制
  
  本书第六章，主要介绍Java访问权限的相关知识。

#### 访问权限的类型
  
    1. public：接口访问权限，对public修饰的变量，方法的访问没有限制
    2. 默认访问：包访问权限，不加任何修饰词的话，默认是同一个包下才有访问权限
    3. protected：继承访问权限，首先会提供包访问权限，其次，如果子类继承了某个类，那么可以不在同
                  一个包下也能访问父类中用protected修饰的资源
    4. private：类访问权限，只能在本类中被访问

{% img  /image/ThinkingInJava/ThinkingInJava3-1.png  '"访问权限"' %}

### 小结

      这两章主要介绍了Java类相关的知识点。类的初始化，构造方法的一些注意点。基础类型参数的重载方法
    是以前没注意到的，所以记录了下。类的初始化顺序，也是相对重要的知识点，静态代码块的优先级是最高
    的，所以也常用作一个默认值设置等初始化操作。

> 下一章 [\<ThinkingInJava-4>](https://rel-fly.com/2020/06/09/ThinkingInJava4/)