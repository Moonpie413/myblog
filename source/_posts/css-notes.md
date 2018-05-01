---
title: css笔记
date: 2018-03-31 21:31:06
tags: css
---

恶补前端ing...

<!-- more -->

相对单位
--------

> em是相对长度单位。相对于当前对象内文本的字体尺寸。如当前对行内文本的字体尺寸未被人为设置
> 则相对于浏览器的默认字体尺寸。(引自CSS2.0手册)

浏览器的默认字体高度是16px, 1em=16px, 所以在使用相对单位时可以在body中定义 `font-size=62.5%`, 即 16px * 62.5% = 10px
让10px=1em, 这样就可以方面运算

## rem与em的区别

rem即root em, rem在为元素定义字体大小是仍然是用相对大小, 即相对于root元素的大小, 这样就可以修改root元素
的字体大小就成比例的调整大小

em是指当前节点与父级节点之间的相对大小, 因此如果在副元素重新定义了大小则又要重新计算, 所以引入了rem, 只需要在html元素中
定义 `font-size=62.5%` 则可以统一所有元素都相对根元素调整


display:inline 与 display:block的区别
-------------------------------------

总体来说就是inline的元素不会换行且无法设置其宽高, 而block的元素则会换行并可设置宽高,
具体参考 [what-is-the-difference-between-display-inline-and-display-inline-block](https://stackoverflow.com/questions/8969381/what-is-the-difference-between-display-inline-and-display-inline-block)

inline-block 同时具备两中特性, 即元素会在同一行显示, 且可以设置宽高

## 处理空格

将样式设置为`display: inline-block` 后会在两个元素之间留下一个间隙

如下例子:

```html
<nav>
  <a href="#">One</a>
  <a href="#">Two</a>
  <a href="#">Three</a>
</nav>
```

```css
nav a {
  display: inline-block;
  padding: 5px;
  background: red;
}
```

会产生以下效果:

![inline-block](/images/inline_block.png)

这些空隙是html中换行导致的, 所以一种解决方案就是想办法写成一行, 或者不要在不想要空格的地方换行, 如:

```html
<ul>
  <li>
   one</li><li>
   two</li><li>
   three</li>
</ul><Paste>
```

或

```
<ul>
  <li>one</li
  ><li>two</li
  ><li>three</li>
</ul>
```

或...

```
<ul>
  <li>one</li><!--
  --><li>two</li><!--
  --><li>three</li>
</ul>
```

但这种花式写法终究不是好的解决方案

### 解决方案

1. 由于这个间隙其实是一个文字, 所以可以将出现间隙的父级元素字体设置为0,
然后为子元素单独设置字体大小, 这样就可以去掉间隙

2. 使用float或者flex等其他布局方式解决

## 伪类与伪元素

CSS3中规定伪类用 `selector:pseudo-class { property: value; }` 定义, 而伪元素用 `element::after { style properties }` 定义

定位
-----

## 绝对定位

默认状态下绝对定位是想对于html元素的位置, 如果父节点中有设置了
`position: absolute;` 的相对元素, 则变为相对该元素

## fixed 与 absolute的区别

> 绝对定位固定元素是相对于 <html> 元素或其最近的定位祖先，而固定定位固定元素则是相对于浏览器视口本身

flex布局
---------

flex主要可以解决如下问题

- 垂直居中父内容的里一块内容。
- 使容器的所有子项占用等量的可用宽度/高度，而不管有多少宽度/高度可用。
- 使多列布局中的所有列采用相同的高度，即使它们包含的内容量不同。

当子元素宽\高超过父元素时, 可以使用`flex-wrap: wrap` 属性将溢出的元素自动排列

## 调整子元素的占用比例

通过 `flex` 属性为子元素设置比例值, 元素的百分比会根据该比例值调整

例子:

```css
article {
  flex: 1 200px;
}

article:nth-of-type(3) {
  flex: 2 200px;
}
```

> 这表示“每个flex 项将首先给出200px的可用空间，然后，剩余的可用空间将根据分配的比例共享“
