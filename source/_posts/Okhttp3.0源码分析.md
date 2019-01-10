---
title: OkHttp3.0源码分析
date: 2018-03-01
categories: "Android"
tags: "源码"
---
# 概述
项目开发开发中使用的最多就是OkHttp，今天尝试根据OkHttp的源码分析一下OkHttp。

# 基本使用
```java
OkHttpClient okHttpClient = new OkHttpClient();
Request request = new Request.Builder().url("http://api.douban.com/v2/movie/top250").build();
okHttpClient.newCall(request).enqueue(new Callback() {
	@Override
	public void onFailure(Call call, IOException e) {//请求失败

	}

	@Override
	public void onResponse(Call call, Response response) throws IOException {//请求成功
		Log.e("Tag","response:"+response.body().string());//返回数据
	}
});
```
<!-- more -->
# 源码分析
首先是创建一个OkHttpClient
```java
OkHttpClient okHttpClient = new OkHttpClient();
```
**OkHttpClient.class:**
```java
  public OkHttpClient() {//构造方法
    this(new Builder());
  }
  
  public Builder() {//前期的基本配置
      dispatcher = new Dispatcher();
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;
      writeTimeout = 10_000;
      pingInterval = 0;
    }

  OkHttpClient(Builder builder) {
    this.dispatcher = builder.dispatcher;
    this.proxy = builder.proxy;
    this.protocols = builder.protocols;
    this.connectionSpecs = builder.connectionSpecs;
    this.interceptors = Util.immutableList(builder.interceptors);
    this.networkInterceptors = Util.immutableList(builder.networkInterceptors);
    this.eventListenerFactory = builder.eventListenerFactory;
    this.proxySelector = builder.proxySelector;
    this.cookieJar = builder.cookieJar;
    this.cache = builder.cache;
    this.internalCache = builder.internalCache;
    this.socketFactory = builder.socketFactory;

    boolean isTLS = false;
	
    // ... ...

    this.hostnameVerifier = builder.hostnameVerifier;
    this.certificatePinner = builder.certificatePinner.withCertificateChainCleaner(
        certificateChainCleaner);
    this.proxyAuthenticator = builder.proxyAuthenticator;
    this.authenticator = builder.authenticator;
    this.connectionPool = builder.connectionPool;
    this.dns = builder.dns;
    this.followSslRedirects = builder.followSslRedirects;
    this.followRedirects = builder.followRedirects;
    this.retryOnConnectionFailure = builder.retryOnConnectionFailure;
    this.connectTimeout = builder.connectTimeout;
    this.readTimeout = builder.readTimeout;
    this.writeTimeout = builder.writeTimeout;
    this.pingInterval = builder.pingInterval;

    // ... ...
	
  }
```
然后是创建一个Request
```java
Request request = new Request.Builder().url("http://api.douban.com/v2/movie/top250").build();
```
**Request.class:**

```java
public Builder() {//默认是get请求
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
	
	public Builder url(String url) {//Build模式
      if (url == null) throw new NullPointerException("url == null");

      // Silently replace web socket URLs with HTTP URLs.
      if (url.regionMatches(true, 0, "ws:", 0, 3)) {
        url = "http:" + url.substring(3);
      } else if (url.regionMatches(true, 0, "wss:", 0, 4)) {
        url = "https:" + url.substring(4);
      }

      HttpUrl parsed = HttpUrl.parse(url);
      if (parsed == null) throw new IllegalArgumentException("unexpected url: " + url);
      return url(parsed);//url字符串转成web的URL
    }
	
	public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);//创建一个Request返回
    }
```
OkHttpClient和Request创建的部分都非常的简单。接下来看看请求部分
```java
okHttpClient.newCall(request).enqueue(new Callback() {
	@Override
	public void onFailure(Call call, IOException e) {//请求失败

	}

	@Override
	public void onResponse(Call call, Response response) throws IOException {//请求成功
		Log.e("Tag","response:"+response.body().string());//返回数据
	}
```
**OkHttpClient.class:**
```java
@Override
public Call newCall(Request request) {//okhttp实现了call.Factory接口，会创建一个Call的实现类RealCall
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
**RealCall.class:**
```java
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    RealCall call = new RealCall(client, originalRequest, forWebSocket);
    return call;
  }
  
  @Override 
  public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");//如果已经执行，就没有必要再重复
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```
dispatcher()方法在OkHttpClient中，返回的是Dispatcher，我们看看Dispatcher的enqueue方法
**Dispatcher.class:**
```java
private int maxRequests = 64;//请求队列中的Call的最大值
 /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();//准备异步的Call队列

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();//直接异步的Call队列

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();//直接执行同步的Call队列
  
synchronized void enqueue(AsyncCall call) {//注意此时的AsyncCall并不是Call的实现类,而是Runnable的实现类，后面会讲
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {//当直接异步的队列中的call小于最大值得时候
      runningAsyncCalls.add(call);//将Call放进直接异步的队列中
      executorService().execute(call);//执行这个call
    } else {//当直接异步的队列中的call大于最大值得时候
      readyAsyncCalls.add(call);//将call放进准备异步的队列中
    }
  }
  
  public synchronized ExecutorService executorService() {//在这里我们可以看到单例生成了一个线程池
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
从上面可以看出将call放进线程池中执行，我们就可以推断出AsyncCall是个Runnable任务
**AsyncCall.class:**
```java
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    AsyncCall(Callback responseCallback) {//将CallBack传进来
      super("OkHttp %s", redactedUrl());
      this.responseCallback = responseCallback;
    }
	
	// ... ...

    @Override 
	protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();//获取后台接口返回的数据封装成Response
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));//失败的回调
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);//成功的回调
        }
      } catch (IOException e) {
          eventListener.callFailed(RealCall.this, e);//失败的回调
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```
到这里为止除了Response的部分之外，OkhttpClient算是简单的走了一遍
# 小结
![](http://oxr4g4c3v.bkt.clouddn.com/okhttp-2.webp)
每个一个Request都会封装成一个Call（RealCall），代表着一个请求，这个Call会在Dispatcher（管理线程池）将任务根据设置缓存进一个队列里面，并且执行，执行结果（失败或者成功）有CallBack回调回去。

# getResponseWithInterceptorChain()
接下来分析一下Response的怎么获取，他是整个Okhttp的重点，包括其缓存策略
**RealCall.class:**
```Java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```
在这里可以看出除了我们自定义的拦截器之外，OkHttp也自带许多拦截器，构成了一个拦截器链（List），请求在这个链上传递，包括请求重试，缓存，建立连接等，链上的每个节点都单独负责的部分。先逐个看下拦截器的源码分析其作用。
![](http://oxr4g4c3v.bkt.clouddn.com/okhttp-1.png)
**RetryAndFollowUpInterceptor.class:**
首先下游拦截器在处理网络请求过程如抛出异常，则通过一定的机制判断一下当前链接是否可恢复的（例如，异常是不是致命的、有没有更多的线路可以尝试等），如果可恢复则重试，否则跳出循环。
如果没什么异常则校验下返回状态、代理鉴权、重定向等，如果需要重定向则继续，否则直接跳出循环返回结果。
如果重定向，则要判断下是否已经达到最大可重定向次数， 达到则抛出异常，跳出循环。
```Java
@Override 
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Call call = realChain.call();
    EventListener eventListener = realChain.eventListener();

    StreamAllocation streamAllocation = new StreamAllocation(client.connectionPool(),
        createAddress(request.url()), call, eventListener, callStackTrace);
    this.streamAllocation = streamAllocation;

    int followUpCount = 0;
    Response priorResponse = null;
    while (true) {
      if (canceled) {
        streamAllocation.release();
        throw new IOException("Canceled");
      }

      Response response;
      boolean releaseConnection = true;
      try {//把请求向下传递
        response = realChain.proceed(request, streamAllocation, null, null);
        releaseConnection = false;
      } catch (RouteException e) {
        // 出现异常时，判断是否能恢复，可以的话继续循环重试
        if (!recover(e.getLastConnectException(), streamAllocation, false, request)) {
          throw e.getLastConnectException();
        }
        releaseConnection = false;
        continue;
      } catch (IOException e) {
        // 出现异常时，判断是否能恢复，可以的话继续循环重试
        boolean requestSendStarted = !(e instanceof ConnectionShutdownException);
        if (!recover(e, streamAllocation, requestSendStarted, request)) throw e;
        releaseConnection = false;
        continue;
      } finally {
        // We're throwing an unchecked exception. Release any resources.
        if (releaseConnection) {
          streamAllocation.streamFailed(null);
          streamAllocation.release();
        }
      }

      // Attach the prior response if it exists. Such responses never have a body.
      if (priorResponse != null) {
        response = response.newBuilder()
            .priorResponse(priorResponse.newBuilder()
                    .body(null)
                    .build())
            .build();
      }
	// 重定向
      Request followUp = followUpRequest(response, streamAllocation.route());

      if (followUp == null) {
        if (!forWebSocket) {
          streamAllocation.release();
        }
		// 不需要重定向，返回response
        return response;
      }

      closeQuietly(response.body());

      if (++followUpCount > MAX_FOLLOW_UPS) {// 达到上限次数
        streamAllocation.release();
        throw new ProtocolException("Too many follow-up requests: " + followUpCount);
      }

      if (followUp.body() instanceof UnrepeatableRequestBody) {
        streamAllocation.release();
        throw new HttpRetryException("Cannot retry streamed HTTP body", response.code());
      }

      if (!sameConnection(response, followUp.url())) {
        streamAllocation.release();
        streamAllocation = new StreamAllocation(client.connectionPool(),
            createAddress(followUp.url()), call, eventListener, callStackTrace);
        this.streamAllocation = streamAllocation;
      } else if (streamAllocation.codec() != null) {
        throw new IllegalStateException("Closing the body of " + response
            + " didn't close its backing stream. Bad interceptor?");
      }

      request = followUp;
      priorResponse = response;
    }
  }
```
**BridgeInterceptor.class:**
 桥接拦截器，用于完善请求头，比如Content-Type、Content-Length、Host、Connection、Accept-Encoding、User-Agent等等，这些请求头不用用户一一设置，如果用户没有设置该库会检查并自动完善。此外，这里会进行加载和回调cookie。
**CacheInterceptor.class:**
本地缓存由DiskLruCache实现，另外OkHttp是不支持POST缓存的
```Java
@Override 
public Response intercept(Chain chain) throws IOException {
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;

    long now = System.currentTimeMillis();

    CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
	//缓存策略，网络+本地
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;

    if (cache != null) {
      cache.trackResponse(strategy);
    }

    if (cacheCandidate != null && cacheResponse == null) {
      closeQuietly(cacheCandidate.body()); // The cache candidate wasn't applicable. Close it.
    }

    // If we're forbidden from using the network and the cache is insufficient, fail.
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          .request(chain.request())
          .protocol(Protocol.HTTP_1_1)
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)
          .sentRequestAtMillis(-1L)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();
    }

    // 走缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    Response networkResponse = null;
    try {//走网络
      networkResponse = chain.proceed(networkRequest);
    } finally {
      // If we're crashing on I/O or otherwise, don't leak the cache body.
      if (networkResponse == null && cacheCandidate != null) {
        closeQuietly(cacheCandidate.body());
      }
    }

    // If we have a cache response too, then we're doing a conditional get.
    if (cacheResponse != null) {//走本地
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        Response response = cacheResponse.newBuilder()
            .headers(combine(cacheResponse.headers(), networkResponse.headers()))
            .sentRequestAtMillis(networkResponse.sentRequestAtMillis())
            .receivedResponseAtMillis(networkResponse.receivedResponseAtMillis())
            .cacheResponse(stripBody(cacheResponse))
            .networkResponse(stripBody(networkResponse))
            .build();
        networkResponse.body().close();

        // Update the cache after combining headers but before stripping the
        // Content-Encoding header (as performed by initContentStream()).
        cache.trackConditionalCacheHit();
        cache.update(cacheResponse, response);
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    if (cache != null) {//储存缓存
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {
        // Offer this request to the cache.
        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }

      if (HttpMethod.invalidatesCache(networkRequest.method())) {
        try {
          cache.remove(networkRequest);
        } catch (IOException ignored) {
          // The cache cannot be written.
        }
      }
    }

    return response;
  }
```
看下cache.put(response)
**CaChe.class:**
```Java
@Nullable 
CacheRequest put(Response response) {
    String requestMethod = response.request().method();

    if (HttpMethod.invalidatesCache(response.request().method())) {
      try {
        remove(response.request());
      } catch (IOException ignored) {
        // The cache cannot be written.
      }
      return null;
    }
    if (!requestMethod.equals("GET")) {
      // Don't cache non-GET responses. We're technically allowed to cache
      // HEAD requests and some POST requests, but the complexity of doing
      // so is high and the benefit is low.
      return null;
    }

   // ... ...
  }
```
可以看出不支持POST请求的缓存。其实可以想象一下，POST一般都是要提交参数与后台交互，并不是一个查看的动作，是改的动作，更改后台数据肯定即时的反馈新数据，而不是缓存的旧数据。
另外客户端通过 cacheControl 指定了无缓存，不走缓存，也可以指定了缓存，则看缓存过期时间，符合要求走缓存。
**ConnectInterceptor.class:**
```Java
@Override 
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    StreamAllocation streamAllocation = realChain.streamAllocation();

    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
    RealConnection connection = streamAllocation.connection();

    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```
调用服务拦截器，拦截链中的最后一个拦截器，通过网络与调用服务器。通过HttpStream依次次进行写请求头、请求头（可选）、读响应头、读响应体。实际上利用的是 Okio，而 Okio 实际上还是用的Socket。
**RealInterceptorChain.class:**
```Java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {

     ... 
   //  1.  获取获截器链中的第一个拦截器
   //  2.  通过index + 1，去掉拦截器链中的第一个拦截器获得新的拦截器链
   //  3.  调用原拦截器链中第一个拦截器的intercept()方法，并传入新的拦截器链

     RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ... 
    return response;
  }
```
按顺序添加不同的拦截器，用于分级处理Request和Response，最后创建了一个RealInterceptorChain对象，用于顺序执行每个拦截器中的intercept()方法。
```Java
@Override public Response intercept(Chain chain) throws IOException {

      Request request = chain.request();

      ...   //  加工处理网络请求体

      response = realChain.proceed(request, streamAllocation, null, null);  // 将请求传递给下一个拦截器

      ...  //   加工处理响应体

     return response; 
 }
```
可以看到每个拦截器做的事无非是加工请求对象，将请求交由下一个拦截器处理，当然最后一个拦截器就不需要下交请求，而是直接向服务器发送网络请求，最后对响应加工处理并返回。
