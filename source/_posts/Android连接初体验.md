---
title: 简单Transparent shader的三种实现
date: 2016-12-5
tags:
- Android
categories: 基础操作
---

1，Error: Program “/NDK-build” not found in PATH
解决方法： [http://stackoverflow.com/questions/20200545/error-program-ndk-build-not-found-in-path](http://stackoverflow.com/questions/20200545/error-program-ndk-build-not-found-in-path)
最后试了：C/C++ Build | Tool Chain Editor and select Android Builder as current builder.

2，ADB错误“more than one device and emulator”
解决方法：adb kill-server
并参考了了博客： [http://blog.csdn.net/gaojinshan/article/details/9455193](http://blog.csdn.net/gaojinshan/article/details/9455193)