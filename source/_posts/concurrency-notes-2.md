---
title: 并发编程笔记 (二)
date: 2018-02-19 19:10:50
tags: [Java, Concurrency]
---

《Java并发编程实战》阅读笔记, 主要包括死锁, 显式锁等高级主题

<!-- more -->

死锁
-----

**如果线程以固定顺序获得锁那么程序中就不会出现死锁**

但是有可能因为参数的动态传递而导致原本以为顺序的锁变成非顺序

**避免锁嵌套**, 如果要使用则要确保嵌套锁的执行时序, 或者通过 `System.identityHashCode`
动态决定锁的顺序

需要警惕在持有锁的情况下调用外部方法, 因为该方法可能也是同步的, 会获得两个锁而导致可能的死锁

### 开放调用

调用某个方法时不需要持有锁则被成为开放调用

也就是尽量缩小同步代码块, 使得锁只保护需要保护的区域

活锁
-----

多个相互协作的线程都对彼此作出响应而修改自己的状态, 导致任何线程都无法继续执行

降低锁竞争的方式
----------------

* 减少锁的持有时间
* 降低锁的请求频率
* 使用带有协调机制的独占锁

锁分段
-------

如果一个锁需要保护多个相互独立的状态变量, 则可以将这个锁分解为多个锁, 并且每个锁只保护一个变量

显式锁
------

显式锁可以解决一些内置锁无法解决的问题, 如

* 中断一个正在等待锁的线程
* 请求锁时无限等待

Lock接口的标准使用形式:

```java

Lock lock = new ReentrantLock();
// ...
lock.lock();
try {
  // ...
} finally {
  lock.unlock();
}

```

### 定时锁与轮询锁

通过 `lock.tryLock` 来获取锁, 可以防止死锁的发生 (如果在指定时间内无法获得锁将返回一个失败状态)

### 读写锁

  允许多个读操作同时执行, 但每次只允许一个写操作

```java

public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading
     */
    Lock readLock();

    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing
     */
    Lock writeLock();
}

```

例子: 用读写锁包装的map

```java

public class ReadWriteLockMap<K, V> implements Map<K, V> {

  private final Map<K, V> innerMap;
  private final ReadWriteLock lock = new ReentrantReadWriteLock();
  private final Lock rLock = lock.readLock();
  private final Lock wLock = lock.writeLock();

  public ReadWriteLockMap(Map<K, V> innerMap) {
    this.innerMap = innerMap;
  }

  @Override
  public int size() {
    rLock.lock();
    try {
      return innerMap.size();
    } finally {
      rLock.unlock();
    }
  }

  @Override
  public V get(Object key) {
    rLock.lock();
    try {
      return innerMap.get(key);
    } finally {
      rLock.unlock();
    }
  }

  @Override
  public V put(K key, V value) {
    wLock.lock();
    try {
      return innerMap.put(key, value);
    } finally {
      wLock.unlock();
    }
  }

  // ........

}

```



