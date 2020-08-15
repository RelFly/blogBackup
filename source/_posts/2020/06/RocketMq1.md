---
title: RocketMq-架构
date: 2020-06-23 12:41:37
tags:
- RocketMQ
categories:
- MQ
- RocketMQ
---

### 前言

  虽然用过RocketMQ，但是对他的架构及底层原理都不甚了解，所以阅读github的文档增加一些了解，这里做一个记录。
<!-- more -->

### 架构

  RocketMQ的架构设计主要是四部分
  1. NameServer：路由注册中心，主要功能是管理Broker和路由信息
  2. BrokerServer：负责消息的存储，投递，查询及高可用保证，是RocketMQ中最复杂的部分
     为实现以上功能，他包含以下子模块：
     * Remoting Module：整个Broker的实体，负责处理来自clients端的请求
     * Client Manager：负责管理客户端(Producer/Consumer)和维护Consumer的Topic订阅信息
     * Store Service：提供API接口处理消息存储到物理硬盘和查询功能
     * HA Service：高可用服务，提供Master Broker 和 Slave Broker之间的数据同步功能
     * Index Service：根据特定的Message key对投递到Broker的消息进行索引服务，以提供消息的快速查询
  3. Producer：消息的发布者
  4. Consumer：消息的消费者
  
#### NameServer对Broker的管理
  
  NameServer接受Broker集群的注册信息并保留作为路由信息的数据，并通过心跳检测检查Broker的存活情况。
  心跳包由BrokerServer发出，NameServer负责接收及更新信息，如上报时间等。
  {% codeblock lang:java %}
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

      @Override
      public void run() {
          try {
              BrokerController.this.registerBrokerAll(true, false, brokerConfig.isForceRegister());
          } catch (Throwable e) {
              log.error("registerBrokerAll Exception", e);
          }
      }
  }, 1000 * 10, 
  Math.max(10000, Math.min(brokerConfig.getRegisterNameServerPeriod(), 60000)), 
  TimeUnit.MILLISECONDS);

  // command：执行线程
  // initialDelay：初始化延时
  // period：两次开始执行最小间隔时间
  // unit：计时单位
  public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay,
				                                long period, TimeUnit unit);
  {% endcodeblock %}

  使用ScheduledExecutorService新建一个定时任务，间隔时间范围在10s~60s之间。

  然后NameServer会接收这个心跳包并保存，源码暂时没太理解，这里就不贴代码了。
  除了接受心跳包，NameServer还会启动一个定时任务检测BrokerServer的状态：
  {% codeblock lang:java %}
  // 每10秒检测一次，对应心跳包发送的最小时间间隔
  this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

      @Override
      public void run() {
          NamesrvController.this.routeInfoManager.scanNotActiveBroker();
      }
  }, 5, 10, TimeUnit.SECONDS);

  // 检测的具体逻辑
  // private final static long BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2;
  public void scanNotActiveBroker() {
      Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
      while (it.hasNext()) {
          Entry<String, BrokerLiveInfo> next = it.next();
          // 超过2分钟没有收到心跳则删除
          long last = next.getValue().getLastUpdateTimestamp();
          if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
              RemotingUtil.closeChannel(next.getValue().getChannel());
              it.remove();
              log.warn("The broker channel expired, {} {}ms", 
              next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
              this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
          }
      }
  }
  {% endcodeblock %}

#### Broker的消息存储
  
  消息存储主要和以下三个文件有关：
  1. CommitLog：消息主体以及元数据的存储主体，存储Producer端写入的消息主体内容,消息内容不是定长的
>    单个文件大小默认1G ，文件名长度为20位，左边补零，剩余为起始偏移量，
>    比如00000000000000000000代表了第一个文件，起始偏移量为0，文件大小为1G=1073741824；
>    当第一个文件写满了，第二个文件为00000000001073741824，起始偏移量为1073741824，以此类推。消息主要是顺序写入日志文件，当文件满了，写入下一个文件。

  2. ConsumeQueue：消息消费队列，引入的目的主要是提高消息消费的性能，可以理解为toic的索引文件，能够提高基于topic对消息的查询效率
>    ConsumeQueue（逻辑消费队列）作为消费消息的索引，保存了指定Topic下的队列消息在CommitLog中的起始物理偏移量offset，消息大小size和消息Tag的HashCode值。
>    consumequeue文件可以看成是基于topic的commitlog索引文件，故consumequeue文件夹的组织方式如下：topic/queue/file三层组织结构，
>    具体存储路径为：$HOME/store/consumequeue/{topic}/{queueId}/{fileName}。
>    同样consumequeue文件采取定长设计，每一个条目共20个字节，分别为8字节的commitlog物理偏移量、4字节的消息长度、8字节tag hashcode，单个文件由30W个条目组成，可以像数组一样随机访问每一个条目，每个ConsumeQueue文件大小约5.72M。
  3. IndexFile：提供了一种可以通过key或时间区间来查询消息的方法
>    Index文件的存储位置是：$HOME \store\index${fileName}，
>    文件名fileName是以创建时的时间戳命名的，固定的单个IndexFile文件大小约为400M，一个IndexFile可以保存 2000W个索引，IndexFile的底层存储设计为在文件系统中实现HashMap结构，故rocketmq的索引文件其底层实现为hash索引。

  CommitLog是数据实际的存储和数据持久化的方式，保证了消息不会丢失。而ConsumeQueue和IndexFile则是基于CommitLog的数据建立的索引来优化消息消费和查询的效率。
  针对三种文件，org.apache.rocketmq.store包下也有对应的同名类进行配置管理，就不在本章详细解析了。

### 部署架构

* NameServer：几乎无状态节点，可集群部署，节点之间无任何信息同步
* BrokerServer：多主从结构，Master与Slave 的对应关系通过指定相同的BrokerName，不同的BrokerId 来定义，BrokerId为0表示Master，非0表示Slave。每个Broker与NameServer集群中的所有节点建立长连接，定时注册Topic信息到所有NameServer
* Producer：完全无状态，可集群部署；与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic 服务的Master建立长连接，且定时向Master发送心跳
* Consumer：与NameServer集群中的其中一个节点（随机选择）建立长连接，定期从NameServer获取Topic路由信息，并向提供Topic服务的Master、Slave建立长连接，且定时向Master、Slave发送心跳

### 小结
  
      本章内容是阅读RocketMQ文档后总结的一些概念和知识点。简单的看了一下NameServer和Broker的部分
    源码。源码内容较复杂，只能一个个模块拆开慢慢看了。
      经过本章的总结，算是对RocketMQ有了基础的认识，之后再详细的了解吧。

> 下一章 [\<RocketMQ-设计>](https://rel-fly.com/2020/06/24/RocketMq2/)