---
title: Java字节码和类加载
date: 2018-04-19
categories: "Java"
tags: "JVM"
---
# 类加载
将描述类的数据 从Class文件加载到内存 & 对数据进行校验、转换解析 和 初始化，最终形成：可被虚拟机直接使用的Java使用类型

# 类加载过程
分为五个步骤：加载 -> 验证 -> 准备 -> 解析 -> 初始化
1. 加载：将外部.class文件加载到虚拟机
2. 验证：确保加载进来的.class文件包含的信息符合虚拟机的规范
3. 准备：为类变量分配内存，设置类变量的初始值
4. 解析：将常量池内的符号引用转成直接引用
5. 初始化：初始化类变量

# Java字节码
![](http://oxr4g4c3v.bkt.clouddn.com/jvm-neicun2.png)
Java的产生是由javac将.java文件编译成.class文件。.class文件中存放的就是.java文件里的内容。例如：
```java
public class Demo{
	private int m;
	public int inc(){
		reture m+1;
	}
}
```
将上面定义的Demo.java（源文件）通过javac编译成.class文件，然后用16进制文本打开.class文件，文本里显示的是十六进制的符号内容。这段符号的组成是遵守着虚拟机的规范的。
```
C38AC3BEC2BAC2BE2020203320130A20
04200F09200320100720110720120120
016D012001490120063C696E69743E01
2003282956012004436F646501200F4C
696E654E756D6265725461626C650120
03696E6301200328294901200A536F75
72636546696C6501200944656D6F2E6A
6176610C200720080C20052006012004
44656D6F0120106A6176612F6C616E67
2F4F626A656374202120032004202020
01200220052006202020022001200720
08200120092020201D20012001202020
052AC2B72001C2B120202001200A2020
20062001202020012001200B200C2001
20092020201F20022001202020072AC2
B420020460C2AC20202001200A202020
062001202020042001200D2020200220
0E
```
但是这些十六进制符号内容代表的是什么意思呢？
如果我们再用javap -verbose来反编译查看一下
<!-- more -->

```java
public class Demo
  SourceFile: "Demo.java"
  minor version: 0
  major version: 51
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#15         //  java/lang/Object."<init>":()V
   #2 = Fieldref           #3.#16         //  Demo.m:I
   #3 = Class              #17            //  Demo
   #4 = Class              #18            //  java/lang/Object
   #5 = Utf8               m
   #6 = Utf8               I
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               inc
  #12 = Utf8               ()I
  #13 = Utf8               SourceFile
  #14 = Utf8               Demo.java
  #15 = NameAndType        #7:#8          //  "<init>":()V
  #16 = NameAndType        #5:#6          //  m:I
  #17 = Utf8               Demo
  #18 = Utf8               java/lang/Object
{
  public Demo();
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>
":()V
         4: return
      LineNumberTable:
        line 1: 0

  public int inc();
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: getfield      #2                  // Field m:I
         4: iconst_1
         5: iadd
         6: ireturn
      LineNumberTable:
        line 4: 0
}
```
我们可以看到，.class 文件中主要有常量池、字段表、方法表和属性表等内容
# 总结
