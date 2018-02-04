---
title: VIM中的Operator-Pending
date: 2018-02-04 16:37:17
tags: vim
---

关于VIM中的Operator-Pending映射
=============================

VIM已经用了有一年了, 但是这一年来基本上只是用到了最基本的功能, 并没有提高很多自己的码字效率,
插件倒是装了一堆却并没有理解为什么,
使用着强大的工具却不了解其运作原理总归让人有点不安, 所以趁这两天准备把VIM的基本运作方式搞搞清楚, 顺便给自己的 _.vimrc_ 瘦个身。

首先, 看的是在线版本的 [Learn Vim The Hard Way][bookurl]

前几章关于 _基本设置_ , _文件类型相关设置_ , _按键映射_ 方面的章节都还好, 到 [operator-padding][operator-padding]
这一章就有点看不懂了, 主要是因为 _text-objects_ 有些不明白, 因此又看了另一篇 [博客][text-objects],
搞懂了 _text-objects_ 之后再来看就简单多了。

Text-Objects
-------------



[bookurl]: 'http://learnvimscriptthehardway.stevelosh.com/'
[operator-padding]: 'http://learnvimscriptthehardway.stevelosh.com/chapters/15.html'
[text-objects]: 'https://blog.carbonfive.com/2011/10/17/vim-text-objects-the-definitive-guide/'
