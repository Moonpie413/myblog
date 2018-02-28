---
title: netty笔记 (二)
date: 2018-02-27 20:36:51
tags:
---

Netty笔记第二部分

<!-- more -->

ByteBuf与ByteBuffer的不同点
---------------------------

Netty提供了自己的ByteBuf实现, 跟JDK中ByteBuffer主要有如下的不同点

1. ByteBuf支持动态扩容 (若不指定长度则默认长度为Integer.MAX_VALUE)
2. 读写分离, 不需要 `clear()` 及 `flip()` 方法来改变读写模式
3. 支持引用计数
4. 支持池化

通过 `hasArray()` 方法区分直接缓冲区还是堆缓冲区

直接缓冲区内部空间将驻留在会被垃圾回收的堆内存之外, 
而堆缓冲区实现底层数据保存在一个数组中, 如果 `hasArray()` 返回 `true` 则可以获取数组遍历数据

事件传播路径
------------

调用channelHandlerContext中 `fire` 开头的方法时会调用channelPipeLine中的下一个handler

### 调用channelHandlerContext与调用channelPipeLine的不同

在channelHandlerContext中调用方法(如write)将会让事件当前handler开始传播,
而调用channelPipeLine中的方法会让事件从头部开始流动

EventLoop
---------


