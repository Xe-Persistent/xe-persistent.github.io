---
title: Java函数式编程学习笔记（二）
date: 2022-05-27 16:25:45
tags:
    - 后端
    - Java
    - Stream
categories: 学习笔记
keywords: "Java, Stream"
---
# 前言
上次我们谈到，在Java代码中使用`Lambda`可以显著减少代码量，提高开发效率。那么在Java项目开发过程中，还有一个非常好用的接口：`Stream Api`，我们叫它`Stream`流。

> 注意：这个`Stream`不同于`java.io`的`InputStream`和`OutputStream`，它代表的是若干个任意Java对象元素的序列，这一点特性和Collection有点相似，但是`Stream`并不会真正存储这些元素，而是根据需要来实时计算和存储，真正的计算通常发生在最后结果的获取，是一种`惰性计算`。

`Stream`使用一种类似用`SQL`语句从数据库查询数据的直观方式来提高Java集合运算逻辑的编码效率，让我们能够写出高效率、干净、简洁的代码。在使用它的时候，我感受到了前所未有的便利。

# Stream流

