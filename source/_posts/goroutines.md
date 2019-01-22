---
title: Goroutines学习笔记
date: 2019-01-11 23:12:31
tags:
---

参考 [Go Concurrency Patterns] 学习笔记

<!-- more -->

[Go Concurrency Patterns]: https://www.youtube.com/watch?v=f6kdp27TYZs

Channel
--------------------

### Multiplexing

Go中的协程是通过channel来实现通信的, 在视频中学习到一个技巧

*Multiplexing* 即信道的多路服用, 例子:

通过 `finIn` 方法实现一个channel代理了两个channel, 使得两个channel的结果可以并行收集

```go
func fanIn(input1, input2 chan string) chan string {
  c := make(chan string)
  // 任意通道有值后即写入新的通道
	go func () {
		for {
			c <- <-input1
		}
	}()
	go func () {
		for {
			c <- <-input2
		}
	}()
  return c
}
```

### channel的方向

在阅读代码的时候经常会看到返回值或者接收的channel都带个箭头,
像这样 `chan<-` 或者这样 `<-chan`, 尝试把箭头去掉也没有问题,
查了之后发现这个语法是声明这个channel是只读还是只写,

* `chan<-` 表示这个channel只能写入, 尝试读取数据会报错
* `<-chan` 表示这个channel只能读取, 尝试写入数据会报错

select
--------

`select` 语法像 `switch`, 只不过每个 `case` 都不等待channel就绪,

一旦有channele可以接收到值则执行相关的操作, 如果所有channelu都没有接收则调用default方法

通过selecte可以重新刚才的 `fanIn` 方法, 例子:

```go
func fanIn(input1, input2 chan string) chan string {
	c := make(chan string)
	go func() {
		for {
		  select {
		  case s := <- input1:
		  	c <- s
		  case s := <- input2:
		  	c <- s
		  }
		}
	}()
	return c
}
```
### select 用法

视频中主要展示了三种用法

1. 作为单个channel的定时器
2. 作为整个loop的定时器
3. 作为整个loop的结束信号
