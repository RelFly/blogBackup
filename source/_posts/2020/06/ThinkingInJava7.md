---
title: ThinkingInJava-7
date: 2020-06-15 14:16:51
tags:
- ThinkingInJava
categories:
- 读书笔记
- ThinkingInJava
---

### 前言

  错误是代码难以避免的问题。编译器的错误可以直接修改代码，运行期的错误就需要将其传递到其他能处理的地方，这时候就要用到Java的异常机制。

<!-- more -->

### 通过异常处理错误

  本书第十二章，主要讲述了Java异常及异常处理的相关内容。

#### Java异常
  
  Java异常都继承自Throwable，然后其下分为Exception和Error两个子类

1. Error(错误)
   Error指一些无法恢复或不能捕捉的严重错误，此类错误会导致程序中断。

2. Exception(异常)
   Exception又分为运行时异常和非运行时异常

   * 运行时异常
     继承自Exception子类RuntimeException的异常类，可以被捕捉处理，也可以不处理。
     常见的包括：NullPointerException，IndexOutOfBoundsException等

   * 非运行时异常
     继承自Exception的非RuntimeException的异常类，统称为非运行时异常。
     这种异常必须要捕捉处理，否则编译无法通过。
     常见的如：IOException

#### 异常的处理

1. 抛出异常
   异常抛出的关键字有两个throw和throws，他们的区别很大：
   throw：强调的是抛出异常的这个动作，只在方法体中使用
   throws：是对方法可能抛出异常的声明，相当于告诉其他开发者这个方法可能抛出什么异常，需要处理
{% codeblock lang:java %}
// 手动抛出异常，程序中止
throw new MyException("10000","error");
// 声明方法可能抛出的异常，可以声明多个以逗号分隔
void method(int i) throws MyException, IOException;
{% endcodeblock %}

2. 捕获处理异常

{% codeblock lang:java %}
try {// 执行逻辑
    test.method(1);
}catch(MyException e){// 异常捕捉
    e.printStackTrace();
}catch(IOException ioe){// 方法抛出的所有异常都要捕捉
    System.out.println("不同的处理");
    ioe.printStackTrace();
}finally {
    // 最后执行的代码块，无论是否发生异常都会执行，通常用来执行一些资源的关闭逻辑
    System.out.println("最后执行");
}


try {
    test.method(1);
}catch(MyException | IOException e){// 不同异常的处理逻辑相同则可以简写
    e.printStackTrace();
} finally {
    System.out.println("最后执行");
}


// try-with-resource语法，声明的资源会自动关闭，无需在finally手动执行
try (BufferedReader br = new BufferedReader(new FileReader(""))) {
     br.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
{% endcodeblock %}

  用catch捕捉异常可以通过捕捉对应异常的父类异常，例如上述例子中可以这么写
   
{% codeblock lang:java %}
try {// 执行逻辑
    test.method(1);
}catch(Exception e){// 异常捕捉
    e.printStackTrace();
}finally {
    // 最后执行的代码块，无论是否发生异常都会执行，通常用来执行一些资源的关闭逻辑
    System.out.println("最后执行");
}
{% endcodeblock %}    

  因为Exception是其他异常的父类，所以可以捕捉到方法抛出的两种异常。当然，虽然这样看上去更加
  简洁，但是异常信息就不够明确，所以一般情况不建议这样做。
  因此，catch异常的顺序也需要特别注意，如果把大的异常(即父类异常)放到前面，由于对异常的匹配
  寻找是按顺序来的，后面的小异常可能永远不会派上用场。

#### 自定义异常

  除了Java已经定义好的异常类，我们也可以根据自己的需求自定义异常类

{% codeblock lang:java %}
// 自定义异常需要继承Exception或其子类
public class MyException extends Exception {
    private String code;
    private String msg;

    public MyException() {

    }
    // Exception提供了不同的有参构造方法
    public MyException(String code, String msg) {
        super(msg);
        this.code = code;
        this.msg = msg;
    }
    // 重写getMessage()方法以实现定制化的信息输出
    @Override
    public String getMessage() {
        return "MyException detail:" + code + "-" + msg;
    }
}
{% endcodeblock %}

#### 异常的栈轨迹
  
  通常我们用*e.printStackTrace();*输出异常信息的时候可以看到一行行的方法地址。这就是异常的栈轨迹，表示异常一步步抛出到最外面的轨迹。

{% codeblock lang:java %}
  public void method(float f) {
    InterfaceTest test = new Product();
    try {
        throw new MyException();
    }catch(MyException e){
        e.printStackTrace();
        for(StackTraceElement ste:e.getStackTrace()){
            System.out.println(ste.getMethodName());
        }
    }
}
public void numberTwo(){method(1);}
public void numberThree(){numberTwo();}
{% endcodeblock %}

  比如上面这样层层调用，通过*e.getStackTrace()*获取栈轨迹的数组，然后逐个输出，就能看到如下结果

     method
     numberTwo
     numberThree
     main
  
  能够看到每一步调用的方法。
  
#### 异常链
  
  如果捕捉异常后，抛出新的异常，那么会丢失原异常的信息。
  这时可以通过Exception的构造方法保留原异常的信息：

{% codeblock lang:java %}
public Exception(String message, Throwable cause) {
    super(message, cause);
}
{% endcodeblock %}

  cause参数用来表示原始异常，可以通过这种方法将不同的异常信息串联起来，也被称作异常链。

### 小结

      异常不可避免，但是通过抛出/捕捉处理我们可以自由的控制异常。将他抛到我们需要处理的地方，或者直接
    捕捉进行处理。这使得代码逻辑能够按照我们所设想的逻辑执行，而不会因为突如其来的异常导致程序中止，偏
    离了我们的预想。
      另一方面，对异常的信息输出也很关键了。良好的输出代表着代码有效的反馈，能够帮助开发者快速的定位问
    题，然后决定如何修改优化。

> 下一章 [\<ThinkingInJava-8>](https://rel-fly.com/2020/06/17/ThinkingInJava8/)