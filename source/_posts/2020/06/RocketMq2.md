---
title: RocketMq-设计
date: 2020-06-24 09:54:19
tags:
- RocketMQ
categories:
- MQ
- RocketMQ
---

### 前言
  
  上一章基本了解了RocketMQ的架构部分，知道了四个结构的功能和作用，这一章继续看下RocketMQ的一些设计细节。
<!-- more -->

### 通信机制

  在上一章中，我们可以看到NameServer，BrokerServer，Producer，Consumer之间都存在通信
  的过程。例如Broker需要注册，发送心跳包给NameServer;生产者，消费者也需要通过NameServer获取Broker的路由信息等。
  而RocketMQ中专门有一个rocketmq-remoting负责实现这些通信功能。

> rocketmq-remoting 模块是 RocketMQ消息队列中负责网络通信的模块，它几乎被其他所有需
> 要网络通信的模块（诸如rocketmq-client、rocketmq-broker、rocketmq-namesrv）所依赖和引用。为了实现客户端与服务器之间高效的数据请求与接收，RocketMQ消息队列自定义了通信协议并在Netty的基础之上扩展了通信模块。

#### RemotingCommand
  
> RemotingCommand在消息传输过程中对所有数据内容的封装，不但包含了所有的数据结构，还包含了编码解码操作。

  根据RemotingCommand的定义看看通信的数据结构
  {% codeblock lang:java %}
  // 请求操作吗|应答响应码
  private int code;
  // 实现语言，这里默认是Java
  private LanguageCode language = LanguageCode.JAVA;
  // 程序版本
  private int version = 0;
  // 相当于requestId
  private int opaque = requestId.getAndIncrement();
  // 区分是普通RPC还是onewayRPC得标志
  private int flag = 0;
  // 传输自定义文本信息
  private String remark;
  // 请求自定义扩展信息
  private HashMap<String, String> extFields;
  {% endcodeblock %}

  除了定义了数据结构，还提供了相应的方法，如：编码解码及创建请求响应的数据结构，这里就不一一看了。

### 消息过滤
  
  消息过滤主要有两种方式：
  1. Tag过滤方式：消费者指定消费消息时，除了topic还可以配置多个tag。其过程分为两步，首先
     在store层从ConsumeQueue根据消息的tag的hash值做过滤；然后，消费者拉取到消息时还会对tag做一遍比对，防止hash冲突导致拉取错误的数据。

  2. SQL92：大致过程与第一种一样，不同的是在store层过滤数据执行的sql并用BloomFilter避免了每次都去执行

>  SQL92，是数据库的一个ANSI/ISO标准。它定义了一种语言（SQL）以及数据库的行为（事务、隔离级别等）

### 消息查询

  消息查询有两种方式：
  1. 按照MessageId查询，MessageId的长度总共有16字节，其中包含了消息存储主机地址（IP地址和端口）
> Client端从MessageId中解析出Broker的地址（IP地址和端口）和Commit Log的偏移地址
> 后封装成一个RPC请求后通过Remoting通信层发送（业务请求码：VIEW_MESSAGE_BY_ID）。Broker端走的是QueryMessageProcessor，读取消息的过程用其中的 commitLog offset 和 size 去 commitLog 中找到真正的记录并解析成一个完整的消息返回。

  2. 按照MessageKey查询，基于IndexFile实现

### 小结
  
      本章主要是基于GitHub文档中设计一节的内容的记录。大概了解了RocketMQ通信的结构及实现，不过关于
    通信的一些设计是基于netty实现，源码看得不是很 理解，就不在这里记录了。
      关于负载均衡及分布式事务消息的设计，内容有点复杂，可能需要单独一章记录，就不在本章赘述了。