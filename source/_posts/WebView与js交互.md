---
title: WebView与js交互
date: 2018-04-25
categories: "Android"
tags: "Web"
---
# 概述
现在开发一个电商类的应用，比较流行的做法就是采用混合式开发，即原生应用里面嵌入网页，这种应用称为Hybrid App。这种开发方式优点是成本低，跨平台，周期短，灵活性强。
但是也有一个致命的缺点就是体验差，毕竟是网页，加载过程会比原生慢。但这不妨碍它的流行。今天来记录一下原生Android如何与js交互的要点。其实简单来讲就是两点：
- Android调用js方法。
- js调用Android方法。


<!-- more -->
# Android调用js方法
两种方法：
1. 通过WebView的loadUrl()
2. 通过WebView的evaluateJavascript()
先新建一个项目，将一个html文件放进新项目的assets中：
```html
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>JSinAndroid</title>
    <script>
         function testJS(){
            alert("Android调用了JS的callJS方法");
         }

         function testAndroid(){
            test.hello("js调用了android中的hello方法");
         }
      </script>
</head>
<body>
<button type="button" id="button1" onclick="testAndroid()">调用Android代码</button>
</body>
</html>
```
Android的布局文件：
```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">
    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        app:layout_constraintTop_toTopOf="parent"
        android:layout_marginTop="200dp"
        android:text="调用js方法"/>

    <WebView
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</android.support.constraint.ConstraintLayout>
```
界面显示如下：
![](http://oxr4g4c3v.bkt.clouddn.com/JSinAndroid1.png)
上面是一个最简单的Web，定义了一个Button，点击了之后Web调用testAndroid方法，这个后面讲。先看Andoid调用JS方法。
首先看testJS方法，在Android端点击按钮之后调用Web的testJS方法。具体实现：
```Java
button.setOnClickListener(new View.OnClickListener() {//调用无参
            @Override
            public void onClick(View v) {
                if (Build.VERSION.SDK_INT < 18) {//SDK版本，当SDK处于4.4以下的时候采用loadUrl的方式加载网页
                    mWebView.loadUrl("javascript:testJS()");
                } else {//SDK版本，当SDK大于4.4的时候采用evaluateJavascript的方式加载网页
                    mWebView.evaluateJavascript("javascript:testJS()", new ValueCallback<String>() {
                        @Override
                        public void onReceiveValue(String value) {
                            //此处为 js 返回的结果
                        }
                    });
                }
            }
        });
		
		button2.setOnClickListener(new View.OnClickListener() {//传参
            @Override
            public void onClick(View v) {
                if(Build.VERSION.SDK_INT < 18){
                    mWebView.loadUrl("javascript:testJS(4567)");
                }else {
                    mWebView.evaluateJavascript("javascript:testJS(456)", new ValueCallback<String>() {
                        @Override
                        public void onReceiveValue(String value) {
                            //此处为 js 返回的结果
                        }
                    });
                }
            }
        });

        // 由于设置了弹窗检验调用结果,所以需要支持js对话框
        // webview只是载体，内容的渲染需要使用webviewChromClient类去实现
        // 通过设置WebChromeClient对象处理JavaScript的对话框
        //设置响应js 的Alert()函数
        mWebView.setWebChromeClient(new WebChromeClient() {
            @Override
            public boolean onJsAlert(WebView view, String url, String message, final JsResult result) {
                AlertDialog.Builder b = new AlertDialog.Builder(MainActivity.this);
                b.setTitle("来自JS的参数");
                b.setMessage(message);
                b.setPositiveButton(android.R.string.ok, new DialogInterface.OnClickListener() {
                    @Override
                    public void onClick(DialogInterface dialog, int which) {
                        result.confirm();
                    }
                });
                b.setCancelable(false);
                b.create().show();
                return true;
            }

        });
```
显示结果：
![](http://oxr4g4c3v.bkt.clouddn.com/JSinAndroid2.png)
![](http://oxr4g4c3v.bkt.clouddn.com/JSinAndroid3.png)
加载网页需要注意的是：
1. loadUrl效率较低，会使页面重新刷新
2. evaluateJavascript效率高，该方法执行时不会使页面刷新，但是只能在Android4.4以后才能使用
日常使用的时候可以安装上述写法，让两者结合起来使用

# js调用Android方法
js调用Android方法有两种方式：
1. 定义AndroidinJS映射类，添加@JavascriptInterface注解
2. 定义协议，复写WebViewClient类的shouldOverrideUrlLoading方法
具体实现如下：
```java
/**
     * JS调用Android方法
     */
    private void AndroidinJS(){
		//方式1：
        // 通过addJavascriptInterface()将Java对象映射到JS对象
        //参数1：Javascript对象名
        //参数2：Java对象名
        mWebView.addJavascriptInterface(new AndroidinJS(), "test");//AndroidtoJS类对象映射到js的test对象
        // 加载JS代码
        // 格式规定为:file:///android_asset/文件名.html
        mWebView.loadUrl("file:///android_asset/jstest.html");

        //方式2：
        // 复写WebViewClient类的shouldOverrideUrlLoading方法
        mWebView.setWebViewClient(new WebViewClient() {
                                      @Override
                                      public boolean shouldOverrideUrlLoading(WebView view, String url) {
                                          // 一般根据scheme（协议格式） & authority（协议名）判断（前两个参数）
                                          //假定传入进来的 url = "js://webview?arg1=111&arg2=222"（同时也是约定好的需要拦截的）
                                          Uri uri = Uri.parse(url);
                                          // 如果url的协议 = 预先约定的 js 协议
                                          // 就解析往下解析参数
                                          if ( uri.getScheme().equals("js")) {
                                              // 如果 authority  = 预先约定协议里的 webview，即代表都符合约定的协议
                                              // 所以拦截url,下面JS开始调用Android需要的方法
                                              if (uri.getAuthority().equals("webview")) {
                                                  Set<String> collection = uri.getQueryParameterNames();
                                                  Iterator iterator = collection.iterator();
                                                  while (iterator.hasNext()) {
                                                      String k = iterator.next().toString();
                                                      Log.e("Tag","-----"+k+" -- "+uri.getQueryParameter(k));
                                                  }
                                              }
                                              return true;
                                          }
                                          return super.shouldOverrideUrlLoading(view, url);
                                      }
                                  }
        );
```
方式1需要定义AndroidinJS类
缺点是存在严重的漏洞问题
```Java
public class AndroidinJS {
    // 定义JS需要调用的方法
    // 被JS调用的方法必须加入@JavascriptInterface注解
    @JavascriptInterface
    public void hello(String msg) {
        Log.e("Tag",msg);
    }
}
```
方式2需要添加约定协议
不存在漏洞问题，虽然有点麻烦，不过出于安全问题，推荐方式2
```html
function testAndroid1(){
            /*约定的url协议为：js://webview?arg1=111&arg2=222*/
            document.location = "js://webview?arg1=111&arg2=222";
         }
```
调用结果：
![](http://oxr4g4c3v.bkt.clouddn.com/JSinAndroid4.png)

# 总结
![](http://oxr4g4c3v.bkt.clouddn.com/JSinAndroid5.png)

# 下载
[源码下载](https://github.com/Kanging/JSinAndroid)

