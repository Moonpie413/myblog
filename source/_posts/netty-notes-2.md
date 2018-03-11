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

Netty线程模型
-------------

在笔记一中说明了Netty服务器启动时需要两个EventLoopGroop(通常只有一个EventLoop), 第一个Group(即BossGroup)负责客户端请求,
并打开channel, 然后将channel交给第二个workerGroup处理, 在workerGroup中存在多个EventLoop,
**每个EventLoop可以处理多个channel, 每个channel由一个EventLoop处理所以事件**, 由于一个EventLoop及对应一个线程, 所有在channel
的生命周期内对它的操作都是线程安全的

**图例: NIO分配的Eventloop模型**

![eventloop.png](/images/eventloop_thread.png)

EventLoop
---------

Netty的EventLoop继承了ScheduledExecutorService, 及实现了自定义的ThreadPoolExecutor(在ScheduledExecutorService基础上扩展)

通过channel可以获取到当前分配给该个channel的EventLoop, 由于EventLoop继承了ScheduledExecutorService, 所以
可以通过它调用ScheduledExecutorService中的所有方法, 即可以给它提交任务(或定时任务),

Eventloop被称为事件循环, 并且前面讲到了它对应一个线程, 通过查看源码找出看出这个入口在哪里

NIO中EventLoop的实现为NioEventLoop, 它是一个 *exctor* 所有它的入口应该为 `execute()` 方法, 这个方法继承自
`SingleThreadEventExecutor`, 代码为:

```java
@Override
    public void execute(Runnable task) {
        if (task == null) {
            throw new NullPointerException("task");
        }

        // 进行了判断, 如果当前线程不是eventloop线程
        // 则会调用startThread启动自身的线程, 并把任务提交到队列
        // 下次任务循环时从队列中取出, 并由自己的线程执行, 以此保证并发安全性
        boolean inEventLoop = inEventLoop();
        if (inEventLoop) {
            addTask(task);
        } else {
            startThread();
            addTask(task);
            if (isShutdown() && removeTask(task)) {
                reject();
            }
        }

        if (!addTaskWakesUp && wakesUpForTask(task)) {
            wakeup(inEventLoop);
        }
    }
```

这里的 `startThread()` 方法就是自身线程的初始化方法, 最后的工作在 `doStartThread()` 中进行

```java
private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
                thread = Thread.currentThread();
                if (interrupted) {
                    thread.interrupt();
                }

                boolean success = false;
                updateLastExecutionTime();
                try {
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                    // 清理工作, 省略...
                }
            }
        });
```

通过跟踪代码， 自身启动了一个thread, 跟前面的理论吻合, 关键点为调用了 `run()` 方法, 发现这个 `run()`
方法为抽象方法, 由子类实现, 在 `NioEventLoop` 中:

```java
@Override
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    select(wakenUp.getAndSet(false));

                    
                default:
            }
        // 省略后续操作...
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // 省略后续操作...
    }
}
```

在 `run()` 方法中通过监听事件, 在循环中 **针对IO和非IO时间做了不同的处理**
