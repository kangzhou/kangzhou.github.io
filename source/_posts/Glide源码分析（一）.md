---
title: Glide源码分析（一）
date: 2018-02-24
categories: "Android"
tags: "源码"
---
# 概述
在众多的图片加载的框架中，个人觉得**Glide**是表现的最好的，尤其是它缓存策略的设计，今天来根据源码来看一下**Glide**里面是如何运作的。由于**Glide**代码十分的庞大，而且隐晦难懂。这里主要是理解其中的含义，点到为止。该系列分为两篇来记录：
{% post_link Glide源码分析（一） %}
{% post_link Glide源码分析（二） %}
# 思维导图
![](http://oxr4g4c3v.bkt.clouddn.com/Glide-1.jpg)
<!-- more -->
# 源码分析
```java
Glide.with(上下文).load("图片路径").into(ImageView);
```
### with()
在with方法中，传入的参数可以有Context，Activity，Fragment和View。这与Picasso不同的是，Picasso只能传入Context，有一定的局限性。
**Glide.class**
```java
@NonNull
  public static RequestManager with(@NonNull Context context) {
    return getRetriever(context).get(context);
  }
  
  @NonNull
  public static RequestManager with(@NonNull Activity activity) {
    return getRetriever(activity).get(activity);
  }
  
  @NonNull
  public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
  }
  
  @SuppressWarnings("deprecation")
  @Deprecated
  @NonNull
  public static RequestManager with(@NonNull android.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
  }
  
  @NonNull
  public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
  }
  
  //。。。。。
  
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // Context could be null for other reasons (ie the user passes in null), but in practice it will
    // only occur due to errors with the Fragment lifecycle.
    Preconditions.checkNotNull(
        context,
        "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
            + "returns null (which usually occurs when getActivity() is called before the Fragment "
            + "is attached or after the Fragment is destroyed).");
    return Glide.get(context).getRequestManagerRetriever();
```
可以看出都会返回**RequestManager**，而且不论传入什么都会当成转成Context处理。
**RequestManagerRetriever.class**
```java
@SuppressWarnings("deprecation")
  @NonNull
  public RequestManager get(@NonNull Activity activity) {
	
	//.......
	
      assertNotDestroyed(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(
          activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
		  
	//.......
  }
  
  @SuppressWarnings({"deprecation", "DeprecatedIsStillUsed"})
  @Deprecated
  @NonNull
  private RequestManager fragmentGet(@NonNull Context context,
      @NonNull android.app.FragmentManager fm,
      @Nullable android.app.Fragment parentHint,
      boolean isParentVisible) {
    RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```
可以看到在这两个方法做了相同的事：添加一个新的Fragment---RequestManagerFragment到当前页面;而这个Fragment的作用就是为了方便Glide控制生命周期,因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件并停止图片加载了。
至此 我们了解了Glide.with()方法背后的操作。一系列操作最终得到的是RequestManager。接下来看load()。

### load() 
**RequestManager.class**:
```java
@NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable String string) {//图片路径
    return asDrawable().load(string);
  }
  
   @NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable Bitmap bitmap) {//加载bitmap
    return asDrawable().load(bitmap);
  }
  
  @NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable Drawable drawable) {//加载本地的drawable
    return asDrawable().load(drawable);
  }
  
   @NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable Uri uri) {//Uri的形式
    return asDrawable().load(uri);
  }
  
  @NonNull
  @CheckResult
  @Override
  public RequestBuilder<Drawable> load(@Nullable File file) {
    return asDrawable().load(file);
  }
  
  @NonNull
  @CheckResult
  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
  }
  
     @CheckResult
  public RequestBuilder<GifDrawable> asGif() {
    return as(GifDrawable.class).apply(DECODE_TYPE_GIF);
  }
  
```
这里可以看出Glide支持加载各种各样的图片资源，包括网络图片、本地图片、Uri对象，Url对象，String字符串等。
其返回的是**RequestBuilder<TranscodeType>**，而**TranscodeType**包含有Flie、Drawable、gif，概括了图片的各种形式。

# 总结
总结一下，首先Glide.with(Activity)，指定了Activity，创建新的Fragment依附于Activity，其目的就是绑定了Activity的生命周期，当该Activity销毁或暂停的时候，Glide也会及时取消或者暂停加载图片。
同时**with()**返回一个RequestManager，在**load()**提供加载的路径，或者直接指定本地的Drawable，返回**RequestBuilder<TranscodeType>**，后续会在**RequestBuilder.into(ImageView)**给图片资源指定载体ImageView。
在下一篇《{% post_link Glide源码分析（二） %}》具体分析。
