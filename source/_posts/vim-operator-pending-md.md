---
title: VIM中的Operator-Pending
date: 2018-02-04 16:37:17
tags: vim
---

VIM已经用了有一年了, 但是这一年来基本上只是用到了最基本的功能, 并没有提高很多自己的码字效率,
插件倒是装了一堆却并没有理解为什么,
使用着强大的工具却不了解其运作原理总归让人有点不安, 所以趁这两天准备把VIM的基本运作方式搞搞清楚, 顺便给自己的 _.vimrc_ 瘦个身。

<!-- more -->

首先, 看的是在线版本的 [Learn Vim The Hard Way][bookurl]

前几章关于 _基本设置_ , _文件类型相关设置_ , _按键映射_ 方面的章节都还好, 到 [operator-padding][operator-padding]
这一章就有点看不懂了, 主要是因为 _text-objects_ 有些不明白, 因此又看了另一篇 [博客][text-objects],
搞懂了 _text-objects_ 之后再来看就简单多了。
Text-Objects
-------------

VIM中的 _text-objects_ 是更高效得操作单词、句子、段落的一个作用域对象, 称为 __文本对象__

首先需要了解VIM的命令结构

VIM中的命令结构如下

```viml
<number><command><text object or motion>
```

* __number__ 是指作用的文本对象数量, 它是可选的, 并且可以在之前或者之后出现

* __commond__ 对文本对象的操作, 如删除、复制等, 它也是可选的, 但如果不指定则该命令就
失去了编辑能力

* __text objects or motion__ 指文本对象的作用域, 如一个单词(w), 一句话(s), 或一个段落(p)

总的来说 一个命令由以上几部分组成起来, 就可以将基本的修改、移动操作变得更加强大

### 文本对象类型

以下为一些常用的文本对象类型

* iw inner word
* aw a word
* iW inner WORD
* aW a WORD
* is inner sentence
* as a sentence
* ip inner paragraph
* ap a paragraph
* it inner tag
* at a tag
* i  ( or i) inner block
* a  ( or a) a block
* i  < or i> inner block
* a  < or a> a block
* i  { or i} inner block
* a  { or a} a block
* i  [ or i] inner block
* a  [ or a] a block
* i  inner block
* a  a block
* i\` inner block
* a\` a block

从列表中可以看出, _text objects_ 不仅可以匹配单词句子, 还可以匹配到括号等变成中经常会用到的
符号, 数量使用的话会相当方便

### 举例

#### 普通文本

`diw` 或 `daw` 删除当前选中的单词

其中 _d_ 指 __commond__ , 而 _iw_ 则是一个单词类型的文本对象

注意 `diw` 只会删除单词, 而 `daw` 会同时删除单词旁边的空格, 这是因为

_iw_ 表示 _inner word_ 而 _aw_ 表示 _a word_ , 它不仅会选中单词, 还会选中旁边的空格

#### 程序文本

如下 _html_ 代码中, 使用 `cit` 命令可以快速修改两个tag之间的内容

```html
<div>
  <p>hello world!</p>
</div>
```

Operator-Pending
----------------

_Operator-Pending_ 中的Operaotr就是上文中的 __commond__,
配上 _text-object_ 或者一个 _movement_ 就构成了一个完成的命令

### Movement 映射

VIM 允许将命令中的 _movement_ 映射到其他按键, 如

```viml
:onoremap p i(
```

将 `i('` 映射到按键p上, 也就是说 `dp` 等价于 `di(`, 即删除括号中的内容

第二个例子:

```viml
:onoremap b /return<cr>
```

执行 `db`, VIM会删除所有下一个 `return` 之间的内容

第三个例子:

```viml
:onoremap in( :<c-u>normal! f(vi(<cr>
```

执行 `din(` 会删除当前光标所在的后一对 () 中的内容, 其中用到了 _normal!_
来模拟命令, 关键点就是 `f(` 这个命令移动到最近的 `(` 然后通过 `vi(` 选中下一个 `)`
之间的内容

[bookurl]: 'http://learnvimscriptthehardway.stevelosh.com/'
[operator-padding]: 'http://learnvimscriptthehardway.stevelosh.com/chapters/15.html'
[text-objects]: 'https://blog.carbonfive.com/2011/10/17/vim-text-objects-the-definitive-guide/'
