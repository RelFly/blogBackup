---
title: http请求过程分析
date: 2021-05-16 20:44:46
tags:
- 计算机网络
categories:
- 计算机网络
---

### 前言

  本章想通过一次http请求过程，分析总结下相关的知识点。
<!-- more -->

### 请求过程

  先概括一下整个请求过程：
  1. 当输入一个网址后，浏览器会进行域名解析，找到对应的IP地址
  2. 浏览器通过IP找到对应服务器请求建立连接
  3. 建立连接后就会发送请求给服务器
  4. 服务器返回响应结果
  5. 断开连接

### DNS解析

  DNS解析会把输入的域名映射到对应的IP地址上，然后通过IP地址找到服务器。
  其目的就是为了方便大众使用，有规律的域名更方便用户记忆。

> 域名系统（英文：Domain Name System，缩写：DNS）是互联网的一项服务。
  它作为将域名和IP地址相互映射的一个分布式数据库，能够使人更方便地访问互联网。
  DNS使用UDP端口53。当前，对于每一级域名长度的限制是63个字符，域名总长度则不能超过253个字符。
  --- 摘自百度百科

### TCP三次握手

  在得到服务器IP地址后，客户端就需要发起请求连接，通过TCP三次握手与服务器建立连接。
  
    第一次握手：客户端发送请求，
               标志位SYN=1，seq=seqC(随机数)
    第二次握手：服务端响应新建连接请求，
               标志位SYN=1，标志位ACK=1，ack=seqC+1，seq=seqS(随机数)
    第三次握手：客户端确认报文(检查ack是否正确)，确认无误后，响应请求
               标志位ACK=1，ack=seqS+1，seq=seqC+1
               服务端接收到报文后，同样会检查ack是否正确，无误后建立连接


> 标志位代表TCP连接的状态，有以下六种：
SYN：表示请求建立一个连接）
URG：紧急标志
ACK：确认数据包的接收
PSH：推送标志，提高数据传输的优先级
RST：要求对方重新建立连接
FIN：表示要关闭连接

> 为什么需要三次握手？
  第一次握手请求连接
  此时，服务端确认自己接收消息的功能没有问题
  第二次握手服务端响应第一次握手
  此时，客户端可以确认自己发送消息和接收消息的功能没有问题
  第三次客户端响应第二次握手
  此时，服务端确认自己发送消息的功能没有问题
  至此，双发互相确认了发送，接收功能的完好，可以创建连接开始通信

### 传输阶段



### TCP四次挥手
  

{% img  /image/xxxx/xxxxx.png  '"xxxx"' %}