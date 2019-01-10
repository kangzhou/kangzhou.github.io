---
title: Glide、Fresco、Picasso的对比
date: 2017-12-09
categories: "Android"
tags: "笔记"
---
# 概述
在Android应用中，体现给用户最多的就是图片，所以选择一种适合自己的图片加载框架是我开发中需要慎重考虑的事情，今天来对比一下当下最火的三款图片加载框架的区别。它们分别是：Glide、Fasco和Picasso。目前个人对于Glide比较熟悉，因为在过往的项目中运用的最多的就是Glide，写过《{% post_link Glide源码分析（一） %}》这篇文章帮助自己更好的运用这个框架。而今天这篇文章的目的就是更好的理解这些图片加载框架之间的优缺点。以便日后的开发工作。
<!-- more -->
# Glide
基于Picasso封装的
Glide.with(),括号里面可以传入上下文,activity实例,FragmentActivity实例,Fragement.
Glide采用的是RGB-565
Glide的缓存的更ImageView的尺寸相同
Glide比Picasso需要更多的空间来缓存
Glide可以加载gif图（重要）
Glide会为每种大小不一致的ImageView都缓存一次.
有内存缓存和磁盘文件缓存

# Fasco
包较大（2~3M）
需要初始化Fresco.initialize(this);
三级缓存，分别是 Bitmap缓存，未解码图片缓存， 文件缓存
加载的是全图
适用于需要高性能加载大量图片的场景
能及时地释放内存和空间占用
还要在布局使用SimpleDraweeView控件加载图片。其他两个框架可直接在ImageView中加载

# Picasso
Picasso.with()，括号里面只能传入上下文，局限性   
Picasso采用的ARGB-8888 
Picasso不能加载gif图片
Picasso缓存的是全尺寸,只缓存一次。加载的是全图，因为加载的采样率过高,导致,出现OOM异常的概率要比Glide要大很多.
不支持磁盘缓存
```java
```
# 总结
