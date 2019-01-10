---
title: JVM：内存结构
date: 2018-04-03
categories: "Java"
tags: "JVM"
---
# 概述
Java虚拟机在运行时的区域称为Java运行时数据区，同时会分成不同的区域各司其职，Java虚拟机运行数据区结构如图显示：
![](http://oxr4g4c3v.bkt.clouddn.com/jvm-neicun2.png)
<!-- more -->
# 作用
- 方法区
所有线程共享，存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。
异常：OutOfMemoryError
- Java栈
线程私有，存储Java方法中的局部变量，还包括对象的引用
异常：OutOfMemoryError、StackOverFlowError
- Java堆
所有线程共享，存储实例对象。
异常：OutOfMemoryError
该区域内存最大，是GC主要管理的区域，也称为GC堆
- 本地方法栈
与Java栈类似，但是服务对象是Netive方法。
- 程序计数器
线程私有，用于异常处理，线程恢复等基础功能
异常：在JVM中唯一没有OutOfMemoryError的内存区域。

# 总结

![内存结构功能](http://oxr4g4c3v.bkt.clouddn.com/jvm-neicun1.png)