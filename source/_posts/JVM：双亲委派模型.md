---
title: JVM：双亲委派模型
date: 2018-04-26
categories: "Java"
tags: "JVM"
---
# 类加载器的作用
### 实现类加载的功能（类加载需5个步骤）
将.class文件加载到内存中，并进行校验，解析和初始化等。
### 确定类的唯一性。
判断两个类是否相等的依据是：是否由同一个加载器加载。
相对于虚拟机而言：
若由同一个加载器加载，则这两个类相等。
若不是同一个类加载器加载，则两个类不相等。
相对于java程序而言：
可通过equals()，isInstance(),instanceof来判断
<!-- more -->

# 类加载器的类型
### 启动类加载器
无法被Java程序直接使用
### 拓展类加载器
可以被直接使用
### 应用程序加载器
可以被直接使用，若没有自定义类加载器的话，程序会默认使用应用程序加载器。

# 双亲委派模型
![](http://oxr4g4c3v.bkt.clouddn.com/jvm-neicun2.png)
可以看出，除了顶部的启动类加载器，其余所有的类加载器都有父加载器。
另外，类加载器的父子关系，不是继承的关系，是复用的父加载器的代码。
看下ClassLoader.class：
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {//先检查类是否被加载了
                try {
                    if (parent != null) {//没有加载的话，就让父加载器去加载
                        c = parent.loadClass(name, false);
                    } else {//如果父加载器为空，则使用默认加载器去加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {//如果还是加载不到，则用自身的加载来加载
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```
从源码中可以看出，当对一个类进行加载的时候，首先让父类加载器加载，这就是委派的概念。而父类加载器又会去委派父类加载器的父类加载器（拗口）。最终会传到顶层的启动了类加载器中。
当父类加载器无法完成加载请求的时候，子类加载器去加载。
这体现出来一种优先级的层次关系。
另外，这种模型是官方推荐使用的一种实现方式。但不是强制性的。
# 总结
