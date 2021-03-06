---
title: 初识NIO
date: 2021-01-16 11:38:38
tags:
- NIO
categories:
- Java
- NIO
---

### 前言
    
   最近做技术分享，选择的主题是NIO。将分享内容记录一下，方便以后回顾。
<!-- more -->

### 不同的IO处理模型

   首先来了解下集中常用的IO处理模型。

#### BIO
   BIO是同步阻塞IO，他的特点是读取数据必须在读到数据前提下才会返回，所以会发生阻塞。
   而要解决BIO阻塞带来的效率问题，就需要使用到多线程。
   建立一个线程专门用于读取数据，读到数据返回后，就开辟新的线程去处理数据，而原有线程继续阻塞式的读取数据。
   在连接数没那么多的时候，这种处理模式确实能充分利用CPU，但是当连接数达到一定级别时，这种
   模式在线程切换，创建销毁上的性能，资源损耗就会成倍增加。
   这时候就需要新的IO处理模型。
#### NIO
   NIO作为同步非阻塞IO，他与BIO的不同之处在于读取数据的操作不会阻塞，不管有没有读到数据都会立刻返回。
   而且NIO的相关操作都会返回操作结果，告诉我们是否读取到数据等，这就方便我们进行操作。而不会像操作BIO一样，只能傻等着。

#### AIO
   AIO在NIO的基础上更进一步，不仅读取数据是非阻塞的，而且在进行I/O操作时，采用的是异步方式，也是非阻塞的。
   因此，AIO被叫做异步非阻塞IO。

### NIO的操作
   在了解了几种IO处理模型的工作方式与不同点后，回到正题，来看看如何使用NIO。
   首先，NIO有三大模块：
   1. Buffer：缓冲区，用于存储数据
   2. Channel：通道，用于传输数据
   3. Selector：选择器，会监听通道的事件，借此可实现对读写操作的自定义处理


  下面来看看示例代码

{% codeblock lang:java %}
// 服务端代码
public void testNioServer() throws IOException {
    // 创建服务端连接并设置为非阻塞
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    serverChannel.configureBlocking(false);
    // 创建选择器
    Selector selector = Selector.open();
    // 注册channel到选择器
    SelectionKey key = serverChannel.register(selector, 0, serverChannel);
    // 设置关注事件
    key.interestOps(SelectionKey.OP_ACCEPT);
    // 绑定
    serverChannel.socket().bind(new InetSocketAddress(8888));
    logger.info("服务端启动");
    while (true) {
        // 查询事件，直到有时间通知才会返回
        selector.select();
        // 获取返回的事件
        Set<SelectionKey> keySet = selector.selectedKeys();
        Iterator<SelectionKey> it = keySet.iterator();
        while (it.hasNext()) {
            SelectionKey currentKey = it.next();
            // 从选择器集合中移除 避免重复处理
            it.remove();
            // 判断连接状态
            if (currentKey.isAcceptable()) {
                // 获取当前的channel
                ServerSocketChannel server = (ServerSocketChannel) currentKey.attachment();
                // 接受客户端连接
                SocketChannel clientChannel = server.accept();
                clientChannel.configureBlocking(false);
                // 注册selector 读操作
                clientChannel.register(selector, SelectionKey.OP_READ, clientChannel);
                logger.info("新连接：{}", clientChannel.getRemoteAddress());
            }
            // 判断读取状态
            if (currentKey.isReadable()) {
                SocketChannel currentChannel = (SocketChannel) currentKey.attachment();
                // 建立缓冲区
                ByteBuffer readBuffer = ByteBuffer.allocate(1024);
                // 读操作
                int count = 0;
                while (currentChannel.isOpen() && currentChannel.read(readBuffer) != -1) {
                    // 读取完成 跳出循环
                    if (readBuffer.position() == count) {
                        break;
                    }
                    count = readBuffer.position();
                }
                if (readBuffer.position() == 0) {
                    continue;
                }
                // 重置指针
                readBuffer.flip();
                // limit表示缓冲区内能读到的最大大小
                byte[] content = new byte[readBuffer.limit()];
                readBuffer.get(content);
                logger.info("读取内容：{}", new String(content
                // 响应
                String msg = "收到了：" + currentChannel.getRemoteAddress();
                ByteBuffer responseBuffer = ByteBuffer.wrap(msg.getBytes());
                // 判断是否还有数据未读
                while (responseBuffer.hasRemaining()) {
                    // 写操作
                    currentChannel.write(responseBuffer);
                }
            }
        }
    }
}
// 客户端代码
public void testNioClient() throws IOException {
    SocketChannel clientChannel = SocketChannel.open();
    clientChannel.configureBlocking(false);
    clientChannel.connect(new InetSocketAddress("127.0.0.1", 8888));
    // 判断连接是否完成
    while (!clientChannel.finishConnect()) {
        Thread.yield();
    }
    logger.info("客户端启动");
    String msg = "测试NIO:12345689你好呀@";
    ByteBuffer writeBuffer = ByteBuffer.wrap(msg.getBytes());
    while (writeBuffer.hasRemaining()) {
        clientChannel.write(writeBuffer);
    }
    logger.info("收取响应");
    ByteBuffer responseBuffer = ByteBuffer.allocate(1024);
    int count = 0;
    while (clientChannel.isOpen() && clientChannel.read(responseBuffer) != -1) {
        // 读取完成 跳出循环
        if (responseBuffer.position() > 0 && responseBuffer.position() == count) {
            break;
        }
        count = responseBuffer.position();
    }
    responseBuffer.flip();
    byte[] content = new byte[responseBuffer.limit()];
    responseBuffer.get(content);
    logger.info("响应内容：{}", new String(content));
    clientChannel.close();
}
{% endcodeblock%}

### 总结
      本章简单介绍了几种IO模型，并初步了解NIO的特点与用法。
      相比于BIO极度依赖多线程，NIO只用单线程就可以高效的完成IO操作。使其在高数量的连接场景
    下依旧有很好性能保证。

### 参考
>[Java NIO浅析](https://tech.meituan.com/2016/11/04/nio.html)

### 推荐阅读
>[Scalable IO in Java](http://gee.cs.oswego.edu/dl/cpjslides/nio.pdf)
