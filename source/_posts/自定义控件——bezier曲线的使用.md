---
title: 自定义控件——bezier曲线的使用
date: 2017-12-02
categories: "Android"
tags: "自定义控件"
---
![](http://oxr4g4c3v.bkt.clouddn.com/bezierAn.png)
# 概述
关于自定义View的话，这里先给自己标注一下，用于以后方便查看，介绍一个技术博主的技术博客[HenCoder](http://hencoder.com/)
今天尝试用贝瑟尔曲线自定义一个控件，效果如下：
<!--more-->
![](http://oxr4g4c3v.bkt.clouddn.com/view1.gif)
是不是有种似曾相识的感觉，没错啦，就是QQ消息拖拽的效果。
如果还有人不太了解贝瑟尔曲线，可以[拼命戳我](http://www.html-js.com/article/1628),里面介绍的很详细。

# 具体实现：
首先看下这个图：
![](http://oxr4g4c3v.bkt.clouddn.com/bezierAn.png)
可以看出这里涉及到一些三角函数方面的数学知识，我们都可以用代码的形式把它表述出来。以下：


1. 先有一个拖动大圆c1的圆心，和一个固定小圆c0圆心。我点击屏幕的地方就是两个圆的圆心，获取圆心：
```java
@Override
   public boolean onTouchEvent(MotionEvent event) {
       switch (event.getAction()){
           case MotionEvent.ACTION_DOWN:
               iniPoint(event.getX(),event.getY());//获取圆心
               break;
           case MotionEvent.ACTION_MOVE:
               upPoint(event.getX(),event.getY());//移动的时候拖动圆一直在动，固定圆不动
               break;
           case MotionEvent.ACTION_UP:
               break;
       }
       invalidate();
       return true;
   }
 /**
    * 按下便初始化该点
    * @param x
    * @param y
    */
   private void iniPoint(float x, float y) {
       PressPoint = new PointF(x,y);
       unPressPoint = new PointF(x,y);
   }
/**
    * 移动该点
    * @param x
    * @param y
    */
   private void upPoint(float x, float y) {//移动的时候拖动圆一直在动，固定圆不动
       PressPoint.x = x;
       PressPoint.y = y;
   }
```
2. 拖拽大圆的半径不变的，我们可以写死，不动小圆的半径时根据两圆心的距离的变化而变化的，然后在onDrew()方法里绘制这两个圆
```java
@Override
    protected void onDraw(Canvas canvas) {
        if(PressPoint==null&&unPressPoint==null){
            return;
        }
        //绘制拖拽圆
        canvas.drawCircle(PressPoint.x,PressPoint.y,mPressRadius,mPaint);//mPressRadius是写死的常量
        //计算两圆心c1和c0的距离，在坐标系中计算两个点的距离，公式是x的平方加上y的平方之和，然后开根号，表达式如下：
        int distence = (int) Math.sqrt((PressPoint.x-unPressPoint.x)*(PressPoint.x-unPressPoint.x)+(PressPoint.y-unPressPoint.y)*(PressPoint.y-unPressPoint.y));
		//固定小圆的半径变化形式，两圆越远，小圆半径越小
        mUnPressRadius = MIX_UNPRESSRADIUS-distence/20;
		
        Path bazier = getBazierPath();//路径
        //绘制固定圆和贝瑟尔曲线
        if(bazier!=null){
            canvas.drawCircle(unPressPoint.x,unPressPoint.y,mUnPressRadius,mPaint);//固定小圆
            canvas.drawPath(bazier,mPaint);//曲线
        }
    }
```
3. 接下来最重要的一点是获取四个点（p0,p1,p2,p3）的坐标,先求出a的角度，小圆半径有了，然后就能得到x，y的值了，我们知道x，y和角度a的关系是：tan a = x/y。

求角度a
```java
//角度a，对着图来看这个公式
float a = (float) Math.atan((PressPoint.y-unPressPoint.y)/(PressPoint.x-unPressPoint.x));
//x = mUnPressRadius*Math.sin(a),半径乘以sin(a)
//y = mUnPressRadius*Math.cos(a)
```
求p0(x,y)，这里需要注意一下的是，屏幕的坐标系x轴向右是正方向，但是y轴的是向下才是正的。所以，
```java
p0x = c0x+x
p0y = c0y-y
```
同理我们就能求出p1,p2,p3点了
```java
//四个点的位置
      float p0x = (float) (mUnPressRadius*Math.sin(a)+unPressPoint.x);
      float p0y = (float) (unPressPoint.y-mUnPressRadius*Math.cos(a));
      float p1x = (float) (mPressRadius*Math.sin(a)+PressPoint.x);
      float p1y = (float) (PressPoint.y-mPressRadius*Math.cos(a));
      float p2x = (float) (PressPoint.x-mPressRadius*Math.sin(a));
      float p2y = (float) (PressPoint.y+mPressRadius*Math.cos(a));
      float p3x = (float) (unPressPoint.x-mUnPressRadius*Math.sin(a));
      float p3y = (float) (unPressPoint.y+mUnPressRadius*Math.cos(a));
```
4. 控制点
```java
//获取控制点，取两个圆心的中点为控制点
PointF contrlPoint = new PointF((PressPoint.x+unPressPoint.x)/2,(PressPoint.y+unPressPoint.y)/2);
```
5. 所有东西都准备好就可以把路径画出来了，就是把p0,p1,p2,p3围起来。
```java
bazier.moveTo(p0x,p0y);//移动到p0点
      bazier.quadTo(contrlPoint.x,contrlPoint.y,p1x,p1y);//曲线路径
      bazier.lineTo(p2x,p2y);
      bazier.quadTo(contrlPoint.x,contrlPoint.y,p3x,p3y);
```
6. 最后我们在布局文件中使用
```java
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
   xmlns:app="http://schemas.android.com/apk/res-auto"
   xmlns:tools="http://schemas.android.com/tools"
   android:id="@+id/activity_main"
   android:orientation="vertical"
   android:layout_width="match_parent"
   android:layout_height="match_parent"
   tools:context="acase.zhoukang.com.studydemo.MainActivity">
<acase.zhoukang.com.studydemo.view.BezierView
       android:layout_width="match_parent"
       android:layout_height="match_parent" />
</RelativeLayout>
```
# 总结
关键点是在于求p0，p1，p2，p3四个点的位置，[源码戳我](https://github.com/Kanging/StudyDemo/blob/master/app/src/main/java/acase/zhoukang/com/studydemo/view/BezierView.java)
