---
title: List、Set、Map数据结构
date: 2018-03-22
categories: "Java"
tags: "数据结构"
---
# 概述
选择合适的数据结构来对日常的数据进行增删改查是十分必要的，不同的结构在不同的情况下会表现出不同的效果，今天记录一下List、Set、Map的一些特点。
List、Set、Map在Java中被定义为借口，都有增删改查的方法。Collection也是接口，是所有集合类的接口。继承实现关系如下：
```
Collection<–List<–ArrayList

Collection<–List<–Vector

Collection<–List<–LinkedList

Collection<–Set<–HashSet

Collection<–Set<–HashSet<–LinkedHashSet

Collection<–Set<–SortedSet<–TreeSet

Map<–HashMap<-LinkedHashMap

Map<–SortedMap<–TreeMap
```
<!-- more -->
# List
是有序的Collection，能够精确控制元素插入的位置。与数组类似
- ArrayList
基于**数组**（Array）的List，ArrayList其实是对数组的动态扩充，底层的数据结构使用的是数组结构
特点：有序、可重复、增删快（少量数据）、查找快，线程不安全

- Vector
基于**数组**（Array）的List，Vector其实是对数组的动态扩充，底层的数据结构使用的是数组结构
特点：线程安全，增删改查效率都不如ArrayList
- LinkedList
LinkedList不同于前面两个，是基于链表实现的**双向链表**数据结构，元素的增删不用移动数据。
特点：增删快，查找慢

# Map
关联键和值，存放关联的键值对，根据键来获取值，所以必须保证键的唯一性
- HashMap
最常用的Map，基于**数组和链表**实现，根据键算出一个hash值，确定一个存放的index，允许null。具体可参看《{% post_link 关于1.7和1.8的HashMap分析 %}》
- LinkedHashMap
LinkedHashMap继承自HashMap，特点是内部存入数据是有顺序的，增加了记住元素插入或者访问顺序的功能，这个是通过内部一个**双向的循环链表**实现的。
- TreeMap
基于**红黑二叉树**的NavigableMap的实现，线程非安全，不允许null，key不可以重复
- HashTable
**线程安全**，其他均与HashMap相同
- ArrayMap
它不是一个适应大数据的数据结构，相比传统的HashMap速度要慢，因为查找方法是二分法，并且当你删除或者添加数据时，会对空间重新调整，在使用大量数据时，效率并不明显，低于50%。
所以ArrayMap是牺牲了时间换区空间。在写手机app时，适时的使用ArrayMap，会给内存使用带来可观的提升。

# Set
Set是一种不包含重复的元素的无序Collection。基于HashMap实现
- HashSet
HashSet是根据hashCode的计算位置的，数据无序的，线程不安全且不允许元素重复：
**HashSet.class:**
```java
public boolean add(E e) {
        return map.put(e, PRESENT)==null;//元素作为键值
    }
```
可以看出元素作为键值不能重复。同时作为键的话也要重写hashcode方法。
- LinkedHashSet
LinkedHashSet是集成HashSet的，也是线程不安全。区别是LinkedHashSet的取出顺序跟存放顺序一样的
- TreeSet
实现了SortedSet接口，所以是有序的，基于TreeMap（二叉树）实现的，同样不能重复，但是不能为空
```java
```
```java
```
# 总结
大多数情况下，从性能上来说ArrayList最好

当集合内的元素需要频繁插入、删除时LinkedList会有比较好的表现

它们三个性能都比不上数组，另外Vector是线程同步的

能用数组的时候(元素类型固定，数组长度固定)，请尽量使用数组来代替List

没有频繁的删除插入操作，又不用考虑多线程问题，优先选择ArrayList

在多线程条件下使用，可以考虑Vector

需要频繁地删除插入，LinkedList就有了用武之地

如果你什么都不知道，用ArrayList没错。
