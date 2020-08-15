---
title: ThinkingInJava-5
date: 2020-06-11 09:16:22
tags:
- ThinkingInJava
categories:
- 读书笔记
- ThinkingInJava
---

### 前言

  在介绍完基本的类和相关设计方法如继承，组合后，就开始扩展类的定义了。
  抽象类和接口是普通类向抽象化进一步延申的类型，两者的出现使得继承这种关系更加灵活且更容易扩展。
  而内部类则是与接口一起，解决了Java无法多继承的问题。变相实现了Java的多继承。
<!-- more -->

### 接口
  
  本书第九章，主要介绍了抽象类和接口的使用。

#### 抽象类

    定义：使用abstract关键字修饰的类，其特点是无法被实例化，所以只能用来被继承。
    抽象方法：1.抽象类中可以没有抽象方法，但有抽象方法的一定是抽象类
             2.抽象方法没有方法体，需要由抽象类的非抽象类子类重写
             3.抽象方法不能定义为private，因为无法被继承，与第二点相悖

  抽象类的出现使得开发者不必关心父类中方法的实现细节，只需要定义某一类型的行为。然后在各个子类中重写方法，定义具体的实现逻辑。

#### 接口
    
    定义：使用interface关键字修饰，其特点是：
          1. 无法实例化，没有构造方法
          2. 接口中的方法都被隐式的指定为public abstract 且无法修改，同样必须在其子类中被重写
          3. 接口的属性都被隐式的指定为public static final 且无法修改
          4. 接口中不能含有代码块及静态方法
          5. 用implements表示实现接口，并且可以实现多个接口，用逗号分隔

  可以看到，相比抽象类，接口是一种更抽象化的类型。
  抽象类还保留有类的一些结构，而接口则可以说是完全的抽象。而且不受单继承的限制，这使得对类结构的设计更加灵活，可扩展。

#### 同名方法的冲突

  书中提到了一种情况，实际开发中因为开发规范等没有遇到过，所以自己试了下。
  问题就是，一个类继承的父类和实现的接口有同名的方法，会怎么样？
  这里就不展示代码了，直接说结果：

    1. 同名且参数列表不同
       没有冲突，可以各自重写
    2. 同名，参数列表相同，返回值也相同
       可以理解为当作了一个方法。如果父该方法是抽象方法，只用也只能重写一次
       (要不然两个一样的方法妥妥的冲突，这点很好理解)。如果不是抽象方法，可不用重写，因为父类已经实现
       了方法逻辑
    3. 同名，参数列表相同，返回值不同
       会有冲突，你对其中一个方法的重写也被视作对另一个方法的重写，但是返回类型不一样就会报错。两个都
       重写，不符合重载方法的规则，也会报错。

  这个问题，日常开发应该不会遇到，而且也应该避免这种同名方法，很容易带来混淆。


### 内部类
  
  本书第十章，在认识了抽象类，接口后，开始介绍另一种类-内部类。

#### 不同的内部类

1. 成员内部类
   最常见的内部类定义，相当于外部类的成员，拥有外部类所有资源的访问权限。

2. 局域内部类
   定义于类方法或作用域中的内部类，只在包含他的方法和作用域中有效。属于方法而不是类的一部分。

3. 匿名内部类
   应该是比较常用的一种内部类定义。主要是在父类或接口存在的情况下，创建一个继承该父类或实现接口的子类对象，并实现对应方法。
   这种方式可以避免单独创建类，如果不考虑类的复用，那么这种写法能节省很多代码量。

4. 静态内部类(嵌套类)
   如果不需要内部类对象与其外围类有联系，则可以声明为static。
   静态内部类的创建不需要外围类的对象，且不能从静态内部类的对象中访问非静态的外围类对象。

代码示例：

{% codeblock lang:java %}
public class ExternalClass {
    private int a;
    private static int b;

    private void method1() {
        // 外围类对内部类的创建及访问
        InsideClass ic = new InsideClass();
        ic.show();
        InsideClass2 ic2 = new InsideClass2();
        ic2.show();
        InsideClass2.show2();

        // 局域内部类
        class InsideClass3 {
            private int i;

            private void method() {
                // 可以访问外围类成员
                System.out.println(a);
            }
        }
        // 只能在方法内访问到
        InsideClass3 ic3 = new InsideClass3();
        ic3.method();
        // 匿名内部类
        // 新建一个实现接口InterfaceTest的子类的对象
        new InterfaceTest() {

            @Override
            public void method(int i) {
                System.out.println(i);
            }
        };
    }

    private static void method2() {
        System.out.println("2");
        class InsideClass3 {
            private int i;

            private void method() {
                System.out.println("```");
            }
        }
        InsideClass3 ic = new InsideClass3();
        ic.method();
    }

    // 成员内部类 不能创建static属性或方法
    public class InsideClass {
        private int i;

        private void show() {
            // 可以访问外部类的所有资源
            a = 1;
            b = 2;
            method1();
            method2();
            ExternalClass.this.method1();
        }
    }

    // 静态内部类
    public static class InsideClass2 {
        private int i;
        private static int j;

        private void show() {
            // 只能访问外部类的静态资源
            b = 2;
            method2();
            System.out.println("InsideClass2");
        }

        private static void show2() {
            b = 2;
            method2();
            System.out.println("InsideClass2");
        }
    }
}
{% endcodeblock%}
{% codeblock lang:java %}
// 成员内部类的对象新建方法
// 需要先取得外围类的对象再新建其内部类对象
ExternalClass.InsideClass ic = new ExternalClass().new InsideClass();
// 静态内部类的对象新建方法： 
ExternalClass.InsideClass2 ic2 = new ExternalClass.InsideClass2();
{% endcodeblock%}

### 小结

        接口，抽象类，内部类，这三者为类的设计提供了更多的可能和思路。特别是接口和内部类，变相实现了
    Java的多继承。
        内部类感觉要比接口和抽象类复杂一些。内部类与外围类相互独立，一个类可以建立多个内部类来继承不
    同的类或实现不同的接口已突破单一继承的限制。另一方面，内部类也是可以被继承的，这在增加了更多可能
    的同时，也会增加类关系的复杂度。所以还是要依据实际业务场景去设计(HashMap和LinkedHashMap应该算
    是个不错的参考对象)。

> 下一章 [\<ThinkingInJava-6>](https://rel-fly.com/2020/06/14/ThinkingInJava6/)