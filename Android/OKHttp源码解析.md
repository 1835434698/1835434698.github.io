### OKHttp源码解析

1、首先创建 OkHttpClient.Builder，分为有参与无惨两个构造方法。

```java
//无参，使用默认参数
public Builder() {
  dispatcher = new Dispatcher();// 负责异步请求执行的类
  protocols = DEFAULT_PROTOCOLS;//http协议，判断使用哪个："http/1.0","http/1.1"，spdy/3.1，h2
  connectionSpecs = DEFAULT_CONNECTION_SPECS;//设置默认TLS配置
  eventListenerFactory = EventListener.factory(EventListener.NONE);//事件的侦听器
  proxySelector = ProxySelector.getDefault();//代理选择器
  if (proxySelector == null) {
    proxySelector = new NullProxySelector();
  }
  cookieJar = CookieJar.NO_COOKIES;//cookie
  socketFactory = SocketFactory.getDefault();//socket
  hostnameVerifier = OkHostnameVerifier.INSTANCE;//主机地址验证
  certificatePinner = CertificatePinner.DEFAULT;//证书
  proxyAuthenticator = Authenticator.NONE;
  authenticator = Authenticator.NONE;// 本地身份验证
  connectionPool = new ConnectionPool();//连接池(默认最多5个线程，最多存活5分钟)
  dns = Dns.SYSTEM;
  followSslRedirects = true; //安全套接层重定向
  followRedirects = true;//本地重定向
  retryOnConnectionFailure = true;//重试连接失败
  callTimeout = 0;
  connectTimeout = 10_000;//连接超时
  readTimeout = 10_000;//read 超时
  writeTimeout = 10_000;//write 超时 
  pingInterval = 0;
}
//okHttpClient作为参数。
Builder(OkHttpClient okHttpClient) {
  this.dispatcher = okHttpClient.dispatcher;
  this.proxy = okHttpClient.proxy;
  this.protocols = okHttpClient.protocols;
  this.connectionSpecs = okHttpClient.connectionSpecs;
  this.interceptors.addAll(okHttpClient.interceptors);
  this.networkInterceptors.addAll(okHttpClient.networkInterceptors);
  this.eventListenerFactory = okHttpClient.eventListenerFactory;
  this.proxySelector = okHttpClient.proxySelector;
  this.cookieJar = okHttpClient.cookieJar;
  this.internalCache = okHttpClient.internalCache;
  this.cache = okHttpClient.cache;
  this.socketFactory = okHttpClient.socketFactory;
  this.sslSocketFactory = okHttpClient.sslSocketFactory;
  this.certificateChainCleaner = okHttpClient.certificateChainCleaner;
  this.hostnameVerifier = okHttpClient.hostnameVerifier;
  this.certificatePinner = okHttpClient.certificatePinner;
  this.proxyAuthenticator = okHttpClient.proxyAuthenticator;
  this.authenticator = okHttpClient.authenticator;
  this.connectionPool = okHttpClient.connectionPool;
  this.dns = okHttpClient.dns;
  this.followSslRedirects = okHttpClient.followSslRedirects;
  this.followRedirects = okHttpClient.followRedirects;
  this.retryOnConnectionFailure = okHttpClient.retryOnConnectionFailure;
  this.callTimeout = okHttpClient.callTimeout;
  this.connectTimeout = okHttpClient.connectTimeout;
  this.readTimeout = okHttpClient.readTimeout;
  this.writeTimeout = okHttpClient.writeTimeout;
  this.pingInterval = okHttpClient.pingInterval;
}
```

2、builder对象可以设置一系列的配置参数包括连接时长、cookie等以及拦截器

```java
builder.connectTimeout(10, TimeUnit.SECONDS).readTimeout(20, TimeUnit.SECONDS).cookieJar(cookieJarImpl)
                .addInterceptor(new Interceptor() {
                    @Override
            public Response intercept(Chain chain) throws IOException {
                    Request request = chain.request();
                    if (!NetUtil.checkNetType(Constant.app)) {
                        int offlineCacheTime = 60;//离线的时候的缓存的过期时间
                        request = request.newBuilder()
//                        .cacheControl(new CacheControl
//                                .Builder()
//                                .maxStale(60,TimeUnit.SECONDS)
//                                .onlyIfCached()
//                                .build()
//                        ) 两种方式结果是一样的，写法不同
                                .header("Cache-Control", "public, only-if-cached, max-stale=" + offlineCacheTime)
                                .build();
                    } 17216
                    return chain.proceed(request);
            }
        }).addNetworkInterceptor(new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Request request = chain.request();
                Response response = chain.proceed(request);
                int onlineCacheTime = 30;//在线的时候的缓存过期时间，如果想要不缓存，直接时间设置为0
                return response.newBuilder()
                        .header("Cache-Control", "public, max-age="+onlineCacheTime)
                        .removeHeader("Pragma")
                        .build();
            }
        }).cache(new Cache(new File(Environment.getExternalStorageDirectory() + "/okttpcaches"), 1024 * 1024 * 20));
//                .build();
        builder.addInterceptor(new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Logger.d(TAG, "addInterceptor : "+chain.request().toString());
//                return null;
                return chain.proceed(chain.request());
            }
        })
                .addNetworkInterceptor(new Interceptor() {
            @Override
            public Response intercept(Chain chain) throws IOException {
                Logger.d(TAG, "addNetworkInterceptor : "+chain.request().toString());
                return chain.proceed(chain.request());
            }
        });
```

3、builder.build();得到OkHttpClient；

```java
public OkHttpClient build() {
  return new OkHttpClient(this);//构造 OkHttpClient
}

public OkHttpClient() {
    this(new Builder());
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
    for (ConnectionSpec spec : connectionSpecs) {
      isTLS = isTLS || spec.isTls();
    }

    if (builder.sslSocketFactory != null || !isTLS) {
      this.sslSocketFactory = builder.sslSocketFactory;
      this.certificateChainCleaner = builder.certificateChainCleaner;
    } else {
      X509TrustManager trustManager = Util.platformTrustManager();
      this.sslSocketFactory = newSslSocketFactory(trustManager);
      this.certificateChainCleaner = CertificateChainCleaner.get(trustManager);
    }

    if (sslSocketFactory != null) {
      Platform.get().configureSslSocketFactory(sslSocketFactory);
    }

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
    this.callTimeout = builder.callTimeout;
    this.connectTimeout = builder.connectTimeout;
    this.readTimeout = builder.readTimeout;
    this.writeTimeout = builder.writeTimeout;
    this.pingInterval = builder.pingInterval;

    if (interceptors.contains(null)) {
      throw new IllegalStateException("Null interceptor: " + interceptors);
    }
    if (networkInterceptors.contains(null)) {
      throw new IllegalStateException("Null network interceptor: " + networkInterceptors);
    }
  }

```

5、构建Request请求体

```java
//A:FormBody方式创建请求参数
FormBody.Builder builder = new FormBody.Builder();//首先构建FormBody，builder存放请求参数
if (httpParams != null && httpParams.length() > 0) {
    Iterator<String> sIterator = httpParams.keys();
    String key;
    String value;
    while (sIterator.hasNext()) {
        // 获得key
        key = sIterator.next();
        if (!"uri".equals(key)){
            // 根据key获得value, value也可以是JSONObject,JSONArray,使用对应的参数接收即可
            value = httpParams.optString(key);
            builder.add(key, value);//设置参数
        }
    }
}
request = new Request.Builder()//构造Request的builder设置请求类型以及创建header的builder；
    .url(url)//设置url
    .post(builder.build())//设置post方式并设置参数RequestBody
    .build();//Request构造请求体

public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
}
Builder(Request request) {
      this.url = request.url;
      this.method = request.method;
      this.body = request.body;
      this.tags = request.tags.isEmpty()
          ? Collections.<Class<?>, Object>emptyMap()
          : new LinkedHashMap<>(request.tags);
      this.headers = request.headers.newBuilder();
    }

    public Builder url(HttpUrl url) {
      if (url == null) throw new NullPointerException("url == null");
      this.url = url;
      return this;
    }
//B:MultipartBody方式创建请求参数
private Request getMultipartBody(JSONObject httpParams, HashMap<String, File> files, String url) {
        MultipartBody.Builder builder = new MultipartBody.Builder()//创建Builder
            .setType(MultipartBody.FORM);//设置类型
        Iterator<Map.Entry<String, File>> iterator = files.entrySet().iterator();

        while (iterator.hasNext()) {
            Map.Entry<String, File> entry = iterator.next();
            builder.addFormDataPart(entry.getKey(), entry.getValue().getName(), RequestBody.create(MediaType.parse("image/png"), entry.getValue()));//添加文件类型参数
            Logger.d(TAG, url + "response = {"+ entry.getKey()+":"+ entry.getValue().getName()+"}");
        }
        if (httpParams != null && httpParams.length() > 0) {
            Iterator<String> sIterator = httpParams.keys();
            String key;
            String value;
            while (sIterator.hasNext()) {
                // 获得key
                key = sIterator.next();
                if (!"uri".equals(key)){
                    // 根据key获得value, value也可以是JSONObject,JSONArray,使用对应的参数接收即可
                    value = httpParams.optString(key);
                    builder.addFormDataPart(key, value);//添加字符串类型参数
                }
            }
        }
        return new Request.Builder().url(url).post(builder.build()).build();
    }

```

4、okHttpClient.newCall(request).enqueue(new Callback() {});开始网络请求。

```java
okHttpClient.newCall(request)//设置请求参数 调用 02
    .enqueue(new Callback() {//开始网络请求并且设置监听接收返回类型 调用 03
            @Override
            public void onFailure(Call call, final IOException e) {
                Logger.d(TAG, uri + "onFailure = " + e.getLocalizedMessage());
                mDeliverHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        if (listener != null) {
                            listener.onErr(Constant.ERRORCODE, e.getMessage(), uri);
                        }
                    }
                });
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                final String strResult = response.body().string();
                Logger.d(TAG, uri + "response = " + strResult);
                mDeliverHandler.post(new Runnable() {
                    @Override
                    public void run() {
                        listener.onResp(strResult, uri);
//                        handleResult(strResult, listener, uri);
                    }
                });
            }
        });




//02
public Call newCall(Request request) {
  return RealCall.newRealCall(this, request, false /* for web socket */);
}
static RealCall newRealCall(OkHttpClient client, Request originalRequest, boolean forWebSocket) {
    // Safely publish the Call instance to the EventListener.
    RealCall call = new RealCall(client, originalRequest, forWebSocket);//构造异步执行器
    call.eventListener = client.eventListenerFactory().create(call);//设置监听
    return call;
  }
//03
 public void enqueue(Callback responseCallback) {
    synchronized (this) {//锁定
      if (executed) throw new IllegalStateException("Already Executed");//是否已经被执行
      executed = true;
    }
    captureCallStackTrace();//调用 04
    eventListener.callStart(this);
    client.dispatcher()//得到负责异步请求执行的类
        .enqueue(new AsyncCall(responseCallback));//创建返回监听并开始异步请求 调用 05
  }
//04
private void captureCallStackTrace() {
    Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");//设置状态
    retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
  }
//05
void enqueue(AsyncCall call) {
    synchronized (this) {//锁定
      readyAsyncCalls.add(call);//添加进一步任务监听列表
    }
    promoteAndExecute();//调用 06
  }
//06
private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {//排队列表
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.正在网络通讯的队列是否已经满了。
        if (runningCallsForHost(asyncCall) >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        executableCalls.add(asyncCall);//添加到执行表
        runningAsyncCalls.add(asyncCall);//添加到执行列表
      }
      isRunning = runningCallsCount() > 0;
    }

    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService()); //开始执行， 调用 07
    }

    return isRunning;
  }
void executeOn(ExecutorService executorService) {
      assert (!Thread.holdsLock(client.dispatcher()));
      boolean success = false;
      try {
        executorService.execute(this);//云行线程池
        success = true;
      } catch (RejectedExecutionException e) {
        InterruptedIOException ioException = new InterruptedIOException("executor rejected");
        ioException.initCause(e);
        eventListener.callFailed(RealCall.this, ioException);
        responseCallback.onFailure(RealCall.this, ioException);
      } finally {
        if (!success) {
          client.dispatcher().finished(this); // This call is no longer running!
        }
      }
    }
```































