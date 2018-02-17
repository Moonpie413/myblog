---
title: 并发编程笔记 (一)
date: 2018-02-14 22:45:33
tags: java
---

并发容器
---------

增强了同步同步容器类, 

### ConcurrentHashMap

* 在迭代时不需要加锁
* 具有若一致性, 而不是即时失败

### CopyOnWriteArrayList

* 安全读取, 写入时复制
  - 保留一个指向底层基础数组的索引, 它不会被修改, 因此其他线程可以安全迭代

处理InterruptedException
------------------------

抛出该异常表示该方法可能长时间阻塞

* 向调用层抛出
* 调用 `Thread.currentThread().interrput()` 中断当前线程

闭锁
-----

用于控制多个线程在同一时间点做某件事情, `awati()` 方法调用后直到该 `countDownLatch` 值
为1, 否则一直阻塞

信号量
------

信号量管理一组 __许可__, 执行操作时首先通过 `acquire()` 获取许可, 执行完后再通过 `release()`
返回许可, 当没有许可时 `acquire()` 会阻塞或抛出异常

__例子: 通过信号量实现有界容器__

```java

public class BoundedSet<T> {

  private final Set<T> innerSet;
  private final Semaphore sem;

  public BoundedSet(int bound) {
    // 封装为同步容器, 多个线程会添加或删除
    this.innerSet = Collections.synchronizedSet(new HashSet<>());
    this.sem = new Semaphore(bound);
  }

  public boolean add(T t) throws InterruptedException {
    // 添加时若容器已满则阻塞
    sem.acquire();
    return innerSet.add(t);
  }

  public boolean remove(T t) {
    boolean removed = innerSet.remove(t);
    if (removed) {
      // 删除时若删除成功则返回许可
      sem.release();
    }
    return removed;
  }

  public static void main(String[] args) throws InterruptedException {
    // 测试, 先充满容器, 再添加元素时应该阻塞, 两秒后删除某个元素后
    // 应该可以继续正常添加
    int bound = 10;
    BoundedSet<Integer> boundedSet = new BoundedSet<>(bound);

    for (int i = 0; i < 10; i++) {
      boundedSet.add(i);
    }

    new Thread(() -> {
      try {
        TimeUnit.SECONDS.sleep(2);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      boundedSet.remove(8);
    }).start();

    System.out.println("start adding 11");
    boundedSet.add(10);
    System.out.println("this should appear after 2 seconds");

  }
}

```

栅栏barrier
-----------

可以确保在并行计算中, 每个线程都执行完第K步操作后才进入第K+1步

### 与闭锁的区别

所有线程必须同时到 __栅栏的位置__ 才可以继续运行
这个 __位置__ 可以由线程自己调用

Memorzer缓存实例
------------

```java

public class Memoizer<A, V> implements Computable<A, V> {

  private final ConcurrentMap<A, Future<V>> cache
      = new ConcurrentHashMap<>();

  private final Computable<A, V> c;

  public Memoizer(Computable<A, V> c) {
    this.c = c;
  }

  @Override
  public V compute(A a) throws InterruptedException {
    while (true) {
      Future<V> f = cache.get(a);
      // 缓存中没有值
      if (f == null) {
        FutureTask<V> ft = new FutureTask<>(() -> c.compute(a));
        // 这时其他线程可能已经放入了future
        f = cache.putIfAbsent(a, ft);
        // 如果已经有future了则不替换 (putIfAbsent是原子操作)
        if (f == null) {
          f = ft;
          // 否则运行future
          // 可保证future只运行一次
          ft.run();
        }
      }
      try {
        // 只要有线程在计算则会阻塞 (因为不会被替换)
        return f.get();
      } catch (ExecutionException e) {
        e.printStackTrace();
      }
    }
  }

}

```

CompletionService
-----------------

将 _Executor_ 与 _BlockingQueue_ 结合到一起, 可以通过 `take()` 异步获取已经完成的任务

线程中断
---------

调用中断不意味着立即停止, 而只是发起了停止请求， 在适当的时候执行中断, 并做好清理工作

可以通过 `Thread.currentThread.isInterrupted` 方法判断当前线程是否已经收到了中断请求

### 响应中断

有两种策略可以用于处理 `InterruptedException`

1. 抛出中断异常
2. __恢复__ 中断状态, 使得上层代码可以处理中断

关于第二中方式, 刚看书的时候并不明白什么叫 _恢复中断_,
查了一下找到了这篇[解释](https://stackoverflow.com/questions/4906799/why-invoke-thread-currentthread-interrupt-in-a-catch-interruptexception-block)

在捕获 `InterruptedException` 后会清除当前的中断状态, 所以外部线程无法看到该线程是否中断,
因此才需要调用`Thread.currentThread().interrupt()` 来重新设置线程的中断状态, 从而使外部线程可以处理

**如下面这个例子**

如果在异常捕获后没有设置当前线程的 `interrupt`,
那么在一次中断后当前线程在调用 `sleep` 或者 `take`
这样的阻塞方法时仍然会继续阻塞, 因为它们看不到当前线程的已阻塞标志

而手动调用 `Thread.currentThread.isInterrupted` 后在下次循环时 `sleep` 或者
`take` 方法可以看到该线程已经中断, 因此会直接再次抛出中断异常

```java

public class QueueInterruptTest {

  private static BlockingQueue<Integer> QUEUE
      = new ArrayBlockingQueue<>(1);

  public static void main(String[] args) throws InterruptedException {
    // 该线程会在take或sleep处阻塞直到外部调用中断
    Thread t = new Thread(() -> {
      while (true) {
        try {
          TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
          System.out.println("ignore sleep interrupt");
          // 不显式调用该方法则下次循环到该处还会继续阻塞
          // 因为sleep看不到该线程已被中断
          // Thread.currentThread().interrupt();
        }
        System.out.println("still looping...");
        try {
          QUEUE.take();
        } catch (InterruptedException e) {
          System.out.println("take interrupt");
          // Thread.currentThread().interrupt();
        }
      }
    });
    t.start();

    // 启动一个线程10秒后中断BlockingQueue的线程
    new Thread(() -> {
      try {
        TimeUnit.SECONDS.sleep(10);
      } catch (InterruptedException e) {
        e.printStackTrace();
      }
      System.out.println("interrupt t, status: " + t.isInterrupted());
      t.interrupt();
    }).start();
  }

}

```

### 处理不可中断的阻塞

以下几种阻塞操作不会抛出 `interruptexception`, 因此需要手动调用相关的方法关闭阻塞操作,
可以复写线程的 `interrput` 方法在响应中断时进行相应的处理

* 同步/异步 IO 手动调用关闭IO
* 获取某个锁而阻塞 通过 `lockInterruptibly` 中断锁等待
