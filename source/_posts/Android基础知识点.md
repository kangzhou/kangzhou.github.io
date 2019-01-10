---
title: Android基础知识点
date: 2016-01-18
categories: "Android"
tags: "笔记"
---
# Android基础
1. 四大组件是什么
活动（Activity） 服务（Server） 广播（BroadcastReceiver） 内容提供者（ContentProvider）

2. 四大组件的生命周期
Activity:onCreate()、onStart()、onPuase()、onResume()、onstop()、onDestoty()、onRestart()
Server:onCreate()、onBind()（首次启动会调用前面这两个方法，再次启动就不会调用了）、onUnbind()、onDestoty()

3. Activity之间的通信方式
Intent
借助类的静态变量
借助全局变量/Application
借助外部工具 :
– 借助SharedPreference
– 使用Android数据库SQLite
– 赤裸裸的使用File – Android剪切板
借助Service
<!-- more -->
4. Activity各种情况下的生命周期

5. 横竖屏切换的时候，Activity 各种情况下的生命周期
5.1 不设置Activity的android:configChanges时，切屏会重新调用各个生命周期，切横屏时会执行一次，切竖屏时会执行两次
5.2 设置Activity的android:configChanges=”orientation”时，切屏还是会重新调用各个生命周期，切横、竖屏时只会执行一次
5.3 设置Activity的android:configChanges=”orientation|keyboardHidden”时，切屏不会重新调用各个生命周期，只会执行onConfigurationChanged方法

6. Activity与Fragment之间生命周期比较
Fragment:onAttach() onCreate() onCreateView() onActivityCreate() onstart() onResume() onPause() onStop() onDestoryView() onDestory() onDetach()

7. Activity上有Dialog的时候按Home键时的生命周期
onSaveInstanceState –> onPause –> onStop onRestart –>onStart—>onResume

8. 两个Activity 之间跳转时必然会执行的是哪几个方法？
一般情况下比如说有两个activity,分别叫A,B。
当在A 里面激活B 组件的时候, A会调用onPause()方法,然后B调用onCreate() ,onStart(), onResume()。
这个时候B覆盖了A的窗体, A会调用onStop()方法。
如果B是个透明的窗口,或者是对话框的样式, 就不会调用A的onStop()方法。
如果B已经存在于Activity栈中，B就不会调用onCreate()方法。

9. Activity的四种启动模式对比
standard:每次启动一个Activity都会重写创建一个新的实例，不管这个实例存不存在
singleTop:判断新的activity已经位于栈顶，那么这个Activity不会被重写创建，同时它的onNewIntent方法会被调用，通过此方法的参数我们可以去除当前请求的信息。如果栈顶不存在该Activity的实例，则情况与standard模式相同
singleTask：如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且会回调该实例的onNewIntent方法
singleInstance :该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例，由于栈内复用的特性，后续的请求均不会创建新的Activity实例，除非这个特殊的任务栈被销毁了。以singleInstance模式启动的Activity在整个系统中是单例的，如果在启动这样的Activiyt时，已经存在了一个实例，那么会把它所在的任务调度到前台，重用这个实例。

10. Activity状态保存如何恢复

11. service和activity怎么进行数据交互？
Activity与Service之间的交互

12. 谈谈你对ContentProvider的理解

13. 说说ContentProvider、ContentResolver、ContentObserver 之间的关系

14. 请描述一下广播BroadcastReceiver的理解

15. 在manifest 和代码中如何注册和使用BroadcastReceiver?
代码中使用广播（动态注册）：
15.1 实现一个广播接收器，继承BroadcastReceiver,实现onReceive方法，在其中实现主要的逻辑。
15.2 使用IntentFilter()添加Action，注册广播。让广播接收器和IntentFilter绑定。registerReceiver
15.3 通过Intent发送一个广播消息

16. 本地广播和全局广播有什么差别？
BroadcastReceiver是针对应用间、应用与系统间、应用内部进行通信的一种方式
LocalBroadcastReceiver仅在自己的应用内发送接收广播，也就是只有自己的应用能收到，数据更加安全广播只在这个程序里，而且效率更高。LocalBroadcastReceiver不能静态注册，只能采用动态注册的方式。

17. AlertDialog,popupWindow,Activity区别

18. Application 和 Activity 的 Context 对象的区别

19. Android属性动画特性

20. 写一个回调demo
```java
/** 
 * 回调接口 
 * 
 */  
public interface Callback {  
    public void execute(String s);  
}

public class B {  
    public void Test(int n ,Callback callback) {  
        callback.execute("int转成String"+n);//进行回调  
    }  
}

protected void onCreate(Bundle savedInstanceState) {  
    super.onCreate(savedInstanceState);  
    setContentView(R.layout.activity_main);  
    B b=new B();  
    b.Test(100 ,new Callback(){  
  
        @Override  
        public void execute(String s) {  
            // TODO Auto-generated method stub  
            System.out.println("实现回调");
            System.out.println(s);  				
        }});  
}
```


21. RecycleView的使用
RecyclerView 是一个增强版的ListView，不仅可以实现和ListView同样的效果，还优化了ListView中存在的各种不足之处
copy:
① RecyclerView封装了viewholder的回收复用，也就是说RecyclerView标准化了ViewHolder，编写Adapter面向的是ViewHolder而不再是View了，复用的逻辑被封装了，写起来更加简单。
② 提供了一种插拔式的体验，高度的解耦，异常的灵活，针对一个Item的显示RecyclerView专门抽取出了相应的类，来控制Item的显示，使其的扩展性非常强。例如：你想控制横向或者纵向滑动列表效果可以通过LinearLayoutManager这个类来进行控制(与GridView效果对应的是GridLayoutManager,与瀑布流对应的还StaggeredGridLayoutManager等)，也就是说RecyclerView不再拘泥于ListView的线性展示方式，它也可以实现GridView的效果等多种效果。你想控制Item的分隔线，可以通过继承RecyclerView的ItemDecoration这个类，然后针对自己的业务需求去抒写代码。
③ 可以控制Item增删的动画，可以通过ItemAnimator这个类进行控制，当然针对增删的动画，RecyclerView有其自己默认的实现。

22. 序列化的作用，以及Android两种序列化的区别
Serializable接口：
Parcelable接口：相比于Seriablizable具有更好的性能。实现Parcelable接口的对象就可以实现序列化并可以通过Intent和Binder传递。

23. 属性动画
差值器和估值器

24. Android的权限管理机制

25. SDK版本升级的兼容问题

26. Dalvik、Art虚拟机与JVM