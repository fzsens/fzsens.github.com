---
layout: post
title: Netty ByteBuf 
date: 2017-01-05
categories: netty
tags: [netty]
description: 介绍Netty ByteBuf设计
---

> 关于ByteBuf的部分主要翻译的Netty官方提供的文档

在Java的网络编程体系中，时常会提到一点重要的区别，堵塞式的`IO`，以`Socket`为例是面向`Stream`构建的，而非堵塞式的`NIO`，以`SocketChannel`为代表，则是基于`Buffer`作为基础。两者之间的区别在与，`Stream`类似连接在自来水管道上，只能顺序从`Stream`中读取或者写入数据，并且`Stream`只能是单向的，`InputStream`或者`OutputStream` 两者有严格的区分，必须按照顺序获取流中的所有数据，在需要跳过部分数据的时候，也需要将数据全部读取，再进行额外的处理；而`NIO`模式则是将数据先读/写到`Buffer`中，`Buffer`本身只是一个缓存区，`SocketChannel`是一个双向通道，在进行读写的时候再从`Buffer`中获取想要的数据，并可以通过下表索引来控制获取全部或者部分数据。

在`NIO`中提供了多种`Buffer`支持，其中使用最多的为`ByteBuffer`，用来表示低级别的二进制和文本消息的数据结构，一个电信的`ByteBuffer`的使用代码如下

````java
ByteBuffer buffer = ByteBuffer.allocate(1024);
while(true) {
  Set<SelectionKey> selectionKeys = selector.selectedKeys();
  Iterator<SelectionKey> keyIterator = selectionKeys.iterator();
  while(keyIterator.hasNext()) {
    SelectionKey key = keyIterator.next();
    if(key.isAcceptable()) {
      ServerSocketChannel sChannel = (ServerSocketChannel) key.channel();
      SocketChannel channel = sChannel.accept();
      channel.configureBlocking(false);
      //写入
      channel.write(ByteBuffer.wrap("from server to client".getBytes()));
      channel.register(selector,SelectionKey.OP_READ);
    } else if (key.isReadable()) {
      SocketChannel channel = (SocketChannel)key.channel();
      //充值position为0，limit为capacity，避免重复读取之前遗留的buffer数据
      buffer.clear();
      //读取，
      channel.read(buffer);
      //设置limit为position，为下一次从buffer中读取做准备
      buffer.flip();
      channel.write(buffer);
    }
    keyIterator.remove();
  }
}
````

整体而言，对`ByteBuffer`的使用还是略显繁琐，特别需要手工管理`mark`，`position`，`limit`，`capacity`这四个下标索引，容易出现遗漏调用`flip()`和`clear()`导致一些异常情况的出现。

正是因为`ByteBuffer`的一些缺点，以及在`NIO`变成中的重要性，`Netty`使用独有的`Buffer API`代替`NIO`中`ByteBuffer`来表示字节序列。这种方式相比使用`ByteBuffer`具有明显的优势。`Netty`的新`Buffer`类型，`ByteBuf`（在Netty3中为`ChannelBuffer`），被设计用来彻底解决`ByteBuffer`存在的问题，并满足网络应用开发人员常见的需求。下面是一些很酷的特性：

- 允许自定义缓存类型
- 复合缓存类型中，内建透明的`zero copy`实现
- 开箱即用的动态Buffer类型，提供类似`StringBuffer`的按需拓展能力
- 不再需要手工调用`flip()`
- 一般情况下会比`ByteBuffer`更快

####  可拓展性

`ByteBuf`有一组丰富并经过优化的操作，由于快速实现自定义协议。举一个例子，`ByteBuf`提供多种操作和访问无符号值，字符串并在缓冲中搜索一定的`byte`序列。你可以通过拓展或者包装 已经存在的`buffer`类型来提供便捷的访问。自定义的`Buffer`类型依然实现了`ByteBuf`的接口，而不是引入一个不兼容的类型。

#### 透明的Zero Copy

为了让网络应用的性能提高到极致，你需要减少内存拷贝操作的次数。你可能有一组`Buffer`需要经过切分，组合形成完整的消息，`Netty`提供一个组合`Buffer`，允许你通过任意数量的既有`Buffer`创建一个新的`buffer`,并且不需要进行内存拷贝。举个例子，一个消息由两部分组成，消息头和消息体，在一个模块化的应用，这两部分的数据可能分别由两个模块产生并在消息发送之前进行组合。

如果使用`ByteBuffer`，你必须创建一个大的`Buffer`，并把header和body两部分拷贝到新的`buffer`。或者，你可以使用`NIO`来执行收集写操作，但是这限制你需要将组合后的`Buffer`表示为一个数组，而不是单一的`Buffer`。并且，这种方法破坏了原本的抽象，并引入复杂的状态管理，更重要的是，如果你不准备使用`NIO`的`Channel`来进行读写，这个方法是不可用的。

````java
//组合对象不兼容组件对象
ByteBuffer[] message = new ByteBuffer[] {header,body};
````

作为比较，`ByteBuf`并没有类型`ByteBuffer`这么多的限制，`ByteBuf`是完全可拓展的，并且拥有一个内建的组合`Buffer`类型。

````java
//组合对象和组件对象都是ByteBuf
ByteBuf message = Unpooled.wrappedBuffer(header,body);

//因此你也可以把组合后的ByteBuf当做一个普通的ByteBuf对象和其他原生的ByteBuf类型进行组合
ByteBuf messageWithFooter = Unpooled.wrappedBuffer(message,footer);

//因为组合对象依然是一个ByteBuf，你可以便捷地访问他的内容，并且访问方法也表现的像一个简单的Buffer，你甚至可以跨越原来的组件长度，进行跨多个组件对象的操作。
messageWithFooter.getunsignedInt(messageWithFooter.readableBytes() - footer.readableBytes() -1);
````

#### **容量自动扩展**

许多协议定义可变长度的消息，这代表着你没有办法确定消息的长度，直到你完成消息的构建，如果消息没有完全构建完毕，则很难有便捷的方法获取精确的消息长度。就像你创建一个`String`，经常需要估算最后的`String`长度，让`StringBuffer`按照自身的需求拓展，

````java
//一个动态Buffer 创建，在内部，实际Buffer的创建是延迟创建的，用于避免潜在的内存空间浪费
ByteBuf b = Unpooled.buffer(4);
//当第一次尝试写入的时候，内部的Buffer使用指定的初始化容量
b.writeByte('1');
b.writeByte('2');
b.writeByte('3');
b.writeByte('4');
//当写入的字节操作了容量之后，内部buffer会自动进行容量的拓展
b.writeByte('5');
````

#### 更好的性能

大部分高频率使用的`Buffer`实现，`ByteBuf`是一个 `byte` 数组 ( `byte[]` ) 非常轻量级的封装，不像`ByteBuffer`，`ByteBuf`没有错综复杂的边界检查和索引补偿，因此JVM可以更容易地优化`buffer`访问，更复杂的`Buffer`实现，仅用于拆分和组合`buffer`，`ByteBuf`的性能表现和`ByteBuffer`一致。















