---
title: 方法栈笔记
date: 2018-12-18 22:45:56
tags:
---

补习计算机原理中, 小记一下

参考学习 [Pointers as function arguments - call by reference][reference]

<!-- more -->

概念
----

### Heap, Stack - 堆栈

解释: [what-and-where-are-the-stack-and-heap](https://stackoverflow.com/questions/79923/what-and-where-are-the-stack-and-heap)

> The stack is the memory set aside as scratch space for a thread of execution. When a function is called, a block is reserved on the top of the stack for local variables and some bookkeeping data. When that function returns, the block becomes unused and can be used the next time a function is called. The stack is always reserved in a LIFO (last in first out) order; the most recently reserved block is always the next block to be freed. This makes it really simple to keep track of the stack; freeing a block from the stack is nothing more than adjusting one pointer.

> The heap is memory set aside for dynamic allocation. Unlike the stack, there's no enforced pattern to the allocation and deallocation of blocks from the heap; you can allocate a block at any time and free it at any time. This makes it much more complex to keep track of which parts of the heap are allocated or free at any given time; there are many custom heap allocators available to tune heap performance for different usage patterns.

> Each thread gets a stack, while there's typically only one heap for the application (although it isn't uncommon to have multiple heaps for different types of allocation).

#### 总结

stack(栈) 是一种后进先出的数据结构, heap(堆) 是一种先进先出的数据结构

* 每个线程创建时都会分配自己的栈空间
* 堆空间在程序启动的时候就会分配好

每个方法执行时都会在栈顶分配一块空间, 称为 stack-frame(栈帧), 方法的局部变量会存储在栈帧中, 可以解释两点

1. 在debug时可以方便的追踪的方法的调用栈信息
2. 值传递时修改方法的形参不会修改原来的值, 因为每个方法进入后都会复制一份参数到栈帧中, 每个方法的栈帧保存在不同的块中, 因此彼此不可见, 所以不可修改


堆内存区的地址分配是随机的, 只要还有剩余空间就会随机分配地址,
也就是说栈区存放的是引用, 而堆区存放的是真实的值

**栈内存比堆内存快**

主要原因是堆内存是公共资源, 所以对公共资源的访问是需要加锁的

#### 例子

```c
int foo()
{
  char *pBuffer; // 栈中初始化一个char类型的指针引用
  bool b = true; // 栈中初始化一个bool类型
  if(b)
  {
    // 还是在栈中初始化一个数据, 在方法结束后回收
    char buffer[500];

    // 在堆中创建一个数组, 并将pBuffer指向这个数组
    // 方法结束后, pBuffer指针会销毁但是堆中的数组不会,
    // 因此会造成内存泄漏
    pBuffer = new char[500];

   }
}
```

### Call Stack - 方法栈


[reference]: https://www.youtube.com/watch?v=LW8Rfh6TzGg
