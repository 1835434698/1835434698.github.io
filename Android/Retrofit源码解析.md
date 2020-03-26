# Retrofit源码解析

```java
//001
Retrofit retrofit = new Retrofit.Builder()//构建Builder
        .baseUrl(baseUrl)//给builder设置baseurl
        .addConverterFactory(ResponseConverterFactory.create())//对象序列化和反序列化的转换器工厂,ResponseConverterFactory是自己实现去处理类型转换。(详看001-1)
        .addCallAdapterFactory(RxJava2CallAdapterFactory.create())//添加用于支持服务方法返回类型的调用适配器工厂
        .client(client)//设置httpclient
        .build();//就是一个builder模式创建的Retrofit。
retrofit.create(service);//创建service对象。（调用001-2）


//001-1
//实现Converter接口复写convert方法
public T convert(ResponseBody value) throws IOException {
        String response = value.string();
        try {
            return gson.fromJson(response, type);
        } finally {
        }
    }

//001-2
 public <T> T create(final Class<T> service) {
    Utils.validateServiceInterface(service);//检查是否是Interface类型，并且检查是否有抽象类
    if (validateEagerly) {
      eagerlyValidateMethods(service);//调用001-3
    }
     //使用动态代理
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();
          private final Object[] emptyArgs = new Object[0];

          @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            return loadServiceMethod(method)//处理参数问题（调用001-6）
                .invoke(args != null ? args : emptyArgs);//处理代理问题（调用001-7）
          }
        });
  }

//001-3
  private void eagerlyValidateMethods(Class<?> service) {
    Platform platform = Platform.get();
    for (Method method : service.getDeclaredMethods()) {//获取所有方法。
      if (!platform.isDefaultMethod(method)) {
        loadServiceMethod(method);//调用001-4
      }
    }
  }

//001-4
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);//创建代理对象（调用001-5）
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
//001-5
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);

    Type returnType = method.getGenericReturnType();
    if (Utils.hasUnresolvableType(returnType)) {
      throw methodError(method,
          "Method return type must not include a type variable or wildcard: %s", returnType);
    }
    if (returnType == void.class) {
      throw methodError(method, "Service methods cannot return void.");
    }

    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);//调用001-8
  }


//001-6
  ServiceMethod<?> loadServiceMethod(Method method) {
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = ServiceMethod.parseAnnotations(this, method);
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
//001-7
  @Override ReturnT invoke(Object[] args) {
    return callAdapter.adapt(
        new OkHttpCall<>(requestFactory, args, callFactory, responseConverter));
  }
//001-8
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    CallAdapter<ResponseT, ReturnT> callAdapter = createCallAdapter(retrofit, method);
    Type responseType = callAdapter.responseType();
    if (responseType == Response.class || responseType == okhttp3.Response.class) {
      throw methodError(method, "'"
          + Utils.getRawType(responseType).getName()
          + "' is not a valid response body type. Did you mean ResponseBody?");
    }
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
      throw methodError(method, "HEAD method must use Void as response type.");
    }

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);//获取参数（调用001-9）

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    return new HttpServiceMethod<>(requestFactory, callFactory, callAdapter, responseConverter);
  }
//0010-9
  private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
      Retrofit retrofit, Method method, Type responseType) {
    Annotation[] annotations = method.getAnnotations();//获取注解参数
    try {
      return retrofit.responseBodyConverter(responseType, annotations);//注解参数传递给001中自定义类（调用001-10）
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw methodError(method, e, "Unable to create converter for %s", responseType);
    }
  }
//001-10
  public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);//（调用001-11）
  }
//001-11
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    checkNotNull(type, "type == null");//检查是否为空
    checkNotNull(annotations, "annotations == null");//检查是否为空

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);//调用 了 001 中的ResponseConverterFactory类的方法
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }

    StringBuilder builder = new StringBuilder("Could not locate ResponseBody converter for ")
        .append(type)
        .append(".\n");
    if (skipPast != null) {
      builder.append("  Skipped:");
      for (int i = 0; i < start; i++) {
        builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
      }
      builder.append('\n');
    }
    builder.append("  Tried:");
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      builder.append("\n   * ").append(converterFactories.get(i).getClass().getName());
    }
    throw new IllegalArgumentException(builder.toString());
  }
```

