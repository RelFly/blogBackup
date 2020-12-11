---
title: final修饰入参的问题
date: 2020-11-11 22:49:56
tags:
- final
categories:
- Java
- 基础
---

### 前言

  最近在使用itext导出pdf的时候遇到一个关于final修饰入参的问题，这里记录一下。
<!-- more -->

### final修饰入参

  首先看看正确例子
{% img  /image/final/final1.png  '"正确例子"' %}

  错误例子
{% img  /image/final/final2.png  '"错误例子"' %}


  两个例子一个是分别判断去new，一个是先new再调用set方法。
  为什么说第一个例子是错误例子呢？
  先看看这里的setFont()方法：
{% img  /image/final/final3.png  '"构造方法"' %}

  这里展示的是Paragraph类的父类Phrase的构造方法，当调用*new Paragraph(value)*时，其底层就是调用的这个方法。
  可以看到这里用final修饰入参font，表示在第一次传入参数后，就会永远指向这个对象不再改变。
  所以在错误例子中，当调用Paragraph构造方法时，font属性的值已被“锁定”，后续调用set方法也无法改变他的值。

### 总结

  这个关于final的使用虽然是一个很基础的知识点，但是当时却还是花了较多时间去定位问题。
  因为极少会碰到用final修饰入参的情况，一开始也没注意这里用final修饰。
  后续也思考了下这里为啥要这么写，没有一个确定的答案。
  只能从效果推测应该是不希望二次修改属性值。


