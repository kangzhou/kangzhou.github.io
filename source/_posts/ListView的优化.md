---
title: ListView的优化
date: 2016-03-25
categories: "Android"
---
# 概述
ListView是我最初学Android时最常用的几个控件之一了，那时候根本不知道什么是自定义View，什么是优化。后来看了别人写的代码才发现在getView里面还能有这样的操作呢！ListView的优化是我每次去面试必问的几道面试题之一。当然内容也很简单，今天不细讲，就大概汇总一下。

# ListView的优化方案
1. 复用converView
目的不用每次都findViewbyid
2. 定义静态内部类ViewHolder
为了避免对外部类（外部类很可能是Activity）对象的引用，那么最好将内部类声明为static的（非必须的）。
3. 尽可能减少在getView中的逻辑判断
4. Item布局优化
减少布局镶嵌，减少背景覆盖（重复渲染）
5. 滑动时停止加载图片
6. 使用分页
7. 数据类型考虑使用弱引用
8. 缓存
<!-- more -->