---
title: Firefox Developer's Edition 使用小记
date: 2018-02-17 18:53:17
tags:
---

小记一下火狐开发者版本使用中遇到的问题
=====================================

修改DPI适应高分屏
------------------

_about:config_ 中修改 `layout.css.devPixelsPerPx`

参考 [How to get Firefox looking right on a high DPI display and Fedora](https://fedoramagazine.org/how-to-get-firefox-looking-right-on-a-high-dpi-display-and-fedora/)

解决画面撕裂
------------

_about:config_ 中修改 `layers.acceleration.force-enabled` 开启硬件加速

参考 [Enable hardware acceleration in Firefox](https://cialu.net/enable-hardware-acceleration-firefox-make-faster/)

