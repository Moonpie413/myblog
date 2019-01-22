---
title: effective go
date: 2019-01-12 23:16:16
tags:
---

Effective Go
=============

插一句因为网络原因, vscode无法自动下载很多依赖工具, 所以使用如下命令自己通过代理安装

```sh
export http_proxy=http://127.0.0.1:8123
git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'

go get -v github.com/cweill/gotests/...
go get -u -v github.com/uudashr/gopkgs/cmd/gopkgs
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v github.com/mdempsky/gocode
go get -u -v github.com/rogpeppe/godef
go get -u -v github.com/golang/lint/golint
go get -u -v github.com/ramya-rao-a/go-outline
go get -u -v sourcegraph.com/sqs/goreturns
go get -u -v golang.org/x/tools/cmd/gorename
go get -u -v github.com/tpng/gopkgs
go get -u -v github.com/newhook/go-symbols
go get -u -v golang.org/x/tools/cmd/guru
```


<!-- more -->

方法接收者
------------

> The rule about pointers vs. values for receivers is that value methods can be invoked on pointers and values, but pointer methods can only be invoked on pointers.

方法接收者的范围是, 定义了value接收者, 则指针也可以使用,
如果定义的是指针接收者, 则只能由指针使用

原因:

> This rule arises because pointer methods can modify the receiver; invoking them on a value would cause the method to receive a copy of the value, so any modifications would be discarded. The language therefore disallows this mistake. There is a handy exception, though. When the value is addressable, the language takes care of the common case of invoking a pointer method on a value by inserting the address operator automatically. In our example, the variable b is addressable, so we can call its Write method with just b.Write. The compiler will rewrite that to (&b).Write for us.

指针接收者定义的方法可能会修改接收者的值, 所以如果由值接收者来调用该方法, 则只会传递一个值的拷贝, 在方法中对该值做修改都会失效, 而绑定到值接收者上的方法, 本身的修改就是无效的, 因此可以传递指针来调用

接口设计原则
------------

参考 [Understanding Go Interfaces]

> Be consservative in what you do, be liberal
> in what you accept from others

这句话原文出自 TCP协议 ,个人理解就是返回值的范围要尽可能的小, 而接收值的范围要尽可能的大, 这跟之前学习的泛型设计思维是一致的

> Return concrete types, receive interfaces as parameters

视频中作者举了个例子, 一个 `Stack` 的设计

```go
type Stack interface {
  Push(v interface{}) Stack
  Pop() Stack
  Empty() bool
}
```

[Understanding Go Interfaces]: https://www.youtube.com/watch?v=F4wUrj6pmSI


panic 和 recover
----------------

go中错误处理的最佳实践

> 在包内部，总是应该从 panic 中 recover：不允许显式的超出包范围的 panic()
> 向包的调用者返回错误值（而不是 panic）

基于这个原则我们在使用第三方包时可以只处理包返回的err, 而不用担心会发生panic

在自己写程序的时候也许要遵循这个原则, 标准库中 `recover` 的使用例子:

```go
// Parse parses the space-separated words in in put as integers.
func Parse(input string) (numbers []int, err error) {
    defer func() {
        if r := recover(); r != nil {
            var ok bool
            err, ok = r.(error)
            if !ok {
                // 将panic转化为错误返回
                err = fmt.Errorf("pkg: %v", r)
            }
        }
    }()

    fields := strings.Fields(input)
    // 该方法可能引发panic
    numbers = fields2numbers(fields)
    return
}
```

通过闭包实现的错误处理方式:

[一种用闭包处理错误的模式](https://go.fdos.me/13.5.html)