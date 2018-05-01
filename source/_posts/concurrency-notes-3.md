---
title: 并发编程中的Condition使用
date: 2018-05-01 09:51:28
tags: java
---

在使用ReentrantLock类实现细粒度的同步控制之外, 还提供了其他很多有用的特性,
比如Condition类

<!-- more -->

条件变量Condition
-----------------

> 条件变量很大一个程度上是为了解决Object.wait/notify/notifyAll难以使用的问题。

Condition提供的主要属性与 `Object.wait` 相似, 即以原子方式释放相关的锁

Condition中的 `await` 与 `signal` 方法分别对于 `Object.wait` 与 `Object.notify`,
另外, 由于它也是 `Object` 的子类, 所以要注意的是它自身也有 `wait/notify` 方法, 不要混淆

每个Lock可以对应多个 `Condition` 对象, 表示在不同情况下阻塞线程

例子: 利用Lock 和 Condition实现的阻塞队列

```java
public class ConditionQueue<T> implements Iterable<T> {

  private final T[] items;

  private final Lock lock = new ReentrantLock();

  /**
   * 队列头尾
   */
  private int head, tail, count;

  /**
   * 锁条件, 队列满
   */
  private Condition full = lock.newCondition();

  /**
   * 锁条件, 队列为空
   */
  private Condition empty = lock.newCondition();

  @SuppressWarnings("unchecked")
  public ConditionQueue(int size) {
    this.items = (T[]) new Object[size];
  }

  public ConditionQueue() {
    this(10);
  }

  /**
   * 增加元素
   *
   * @param item 队列元素
   * @throws InterruptedException 中断异常
   */
  public void put(@NotNull T item) throws InterruptedException {
    lock.lock();
    try {
      while (count == items.length) {
        full.await();
      }
      items[tail++] = item;
      if (tail == items.length) {
        tail = 0;
      }
      count++;
      // 通知所有因为队列空而等待的线程继续
      empty.signalAll();
    } finally {
      lock.unlock();
    }
  }

  /**
   * 从队列头部获取
   *
   * @return item
   * @throws InterruptedException 队列为空时取消
   */
  public T get() throws InterruptedException {
    lock.lock();
    try {
      while (count == 0) {
        // 队列为空, 阻塞线程
        empty.await();
      }
      T item = items[head];
      items[head++] = null;
      if (head == items.length) {
        // 重置头指针, 如果队列满了后再下次put则会从头开始放
        head = 0;
      }
      count--;
      // 取走了item后通知所有因为队列满而阻塞的线程继续
      full.signalAll();
      return item;
    } finally {
      lock.unlock();
    }
  }

  public int size() {
    lock.lock();
    try {
      return count;
    } finally {
      lock.unlock();
    }
  }

  @Override
  public Iterator<T> iterator() {
    return new Iterator<T>() {
      @Override
      public boolean hasNext() {
        return size() > 0;
      }

      @Override
      public T next() {
        try {
          return get();
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
        return null;
      }
    };
  }
}
```

