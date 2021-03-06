



### 1、Glide加载图片用法。

```java
Glide.with(this).load(imageUrl)
        .apply(new RequestOptions().placeholder(R.drawable.ic_launcher))
        .into(imageView1);
```

1.其中with()方法传递上线文，重载了多种方式。目的是方便在activity、fragment、view等直接地方调用获取RequestManager。

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
public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
}

@NonNull
public static RequestManager with(@NonNull Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull androidx.fragment.app.Fragment fragment) {
    return getRetriever(fragment.getActivity()).get(fragment);
}

@NonNull
public static RequestManager with(@NonNull View view) {
    return getRetriever(view.getContext()).get(view);
}
```

```java

private static RequestManagerRetriever getRetriever(@Nullable Context context) {
  // Context could be null for other reasons (ie the user passes in null), but in practice it will
  // only occur due to errors with the Fragment lifecycle.
    //检查context是否为空
  Preconditions.checkNotNull(
      context,
      "You cannot start a load on a not yet attached View or a Fragment where getActivity() "
          + "returns null (which usually occurs when getActivity() is called before the Fragment "
          + "is attached or after the Fragment is destroyed).");
  return Glide.get(context).getRequestManagerRetriever();
}

public static Glide get(@NonNull Context context) {
    //单例双检查,并且glide使用了volatile修饰。保证一定唯一
  if (glide == null) {
    synchronized (Glide.class) {
      if (glide == null) {
        checkAndInitializeGlide(context);//初始化
      }
    }
  }
  return glide;
}

private static void checkAndInitializeGlide(@NonNull Context context) {
  // In the thread running initGlide(), one or more classes may call Glide.get(context).
  // Without this check, those calls could trigger infinite recursion.
  if (isInitializing) {//判断是否正在初始化，如果正在初始化报异常。再次保证一定唯一
    throw new IllegalStateException("You cannot call Glide.get() in registerComponents(),"
        + " use the provided Glide instance instead");
  }
  isInitializing = true;
  initializeGlide(context);
  isInitializing = false;//初始化完成
}

 private static void initializeGlide(@NonNull Context context) {
    initializeGlide(context, new GlideBuilder());//实例一个GlideBuilder
 }


private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();//得到一个GeneratedAppGlideModuleImpl
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();//得到一个空的list
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
        //取设置的配置内容。
      manifestModules = new ManifestParser(applicationContext).parse();
    }
//遍历配置，设置。
    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
      Set<Class<?>> excludedModuleClasses =
          annotationGeneratedModule.getExcludedModuleClasses();
      Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
      while (iterator.hasNext()) {
        com.bumptech.glide.module.GlideModule current = iterator.next();
        if (!excludedModuleClasses.contains(current.getClass())) {
          continue;
        }
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
        }
        iterator.remove();
      }
    }

    if (Log.isLoggable(TAG, Log.DEBUG)) {
      for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
        Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
      }
    }

    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
            ? annotationGeneratedModule.getRequestManagerFactory() : null;
    builder.setRequestManagerFactory(factory);//给builder设置RequestManagerFactory构造Glide时候需要用到
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.applyOptions(applicationContext, builder);//取得用户配置应用设置。
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    Glide glide = builder.build(applicationContext);//调用构造方法构造glide
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
      module.registerComponents(applicationContext, glide, glide.registry);
    }
    if (annotationGeneratedModule != null) {
      annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide = glide;
  }

/**
*通过反射得到一个GeneratedAppGlideModuleImpl实例
*/
 private static GeneratedAppGlideModule getAnnotationGeneratedGlideModules() {
    GeneratedAppGlideModule result = null;
    try {
      Class<GeneratedAppGlideModule> clazz =
          (Class<GeneratedAppGlideModule>)
              Class.forName("com.bumptech.glide.GeneratedAppGlideModuleImpl");
      result = clazz.getDeclaredConstructor().newInstance();
    } catch (ClassNotFoundException e) {
      if (Log.isLoggable(TAG, Log.WARN)) {
        Log.w(TAG, "Failed to find GeneratedAppGlideModule. You should include an"
            + " annotationProcessor compile dependency on com.github.bumptech.glide:compiler"
            + " in your application and a @GlideModule annotated AppGlideModule implementation or"
            + " LibraryGlideModules will be silently ignored");
      }
    // These exceptions can't be squashed across all versions of Android.
    } catch (InstantiationException e) {
      throwIncorrectGlideModule(e);
    } catch (IllegalAccessException e) {
      throwIncorrectGlideModule(e);
    } catch (NoSuchMethodException e) {
      throwIncorrectGlideModule(e);
    } catch (InvocationTargetException e) {
      throwIncorrectGlideModule(e);
    }
    return result;
  }


public Glide build(@NonNull Context context) {
    if (sourceExecutor == null) {//网络图片加载线程池
      sourceExecutor = GlideExecutor.newSourceExecutor();
    }

    if (diskCacheExecutor == null) {//本地硬盘图片加载线程池
      diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    }

    if (animationExecutor == null) {//动画加载线程池
      animationExecutor = GlideExecutor.newAnimationExecutor();
    }

    if (memorySizeCalculator == null) {//内存大小计算器（全屏显示一张图片所占内存，缓存多少张图片等）
      memorySizeCalculator = new MemorySizeCalculator.Builder(context).build();
         //包含bitmapPoolSize、 memoryCacheSize、int arrayPoolSize;

    }

    if (connectivityMonitorFactory == null) {//连接器工厂
      connectivityMonitorFactory = new DefaultConnectivityMonitorFactory();
    }

    if (bitmapPool == null) {//bitmap池
      int size = memorySizeCalculator.getBitmapPoolSize();//得到bitmap可使用的最大大小。如果获取不到，责走统一默认缓冲池
      if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
      } else {
        bitmapPool = new BitmapPoolAdapter();
      }
    }

    if (arrayPool == null) {//array池
      arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
    }

    if (memoryCache == null) {//memory池
      memoryCache = new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {//存储到SD卡的存储工厂
      diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    if (engine == null) {
      engine =
          new Engine(//构成engine传入上面生成的配置。
              memoryCache,
              diskCacheFactory,
              diskCacheExecutor,
              sourceExecutor,
              GlideExecutor.newUnlimitedSourceExecutor(),//一个没有限制的网络资源线程池
              GlideExecutor.newAnimationExecutor(),//一个线程池
              isActiveResourceRetentionAllowed);
    }

    //终于初始化了RequestManagerRetriever。其中成员有主线程的Handler，用来发送消息给主线程。
    RequestManagerRetriever requestManagerRetriever =
        new RequestManagerRetriever(requestManagerFactory);
//终于初始化了Glide，并且锁定了配置。
    return new Glide(
        context,
        engine,
        memoryCache,
        bitmapPool,
        arrayPool,
        requestManagerRetriever,
        connectivityMonitorFactory,
        logLevel,
        defaultRequestOptions.lock(),
        defaultTransitionOptions);
  }


```

```java
//拿到了RequestManagerRetriever开始getRetriever(context).get(context);
public RequestManager get(@NonNull Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
      //不是application责调用 对应的方法
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }
  return getApplicationManager(context);
}

 public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
        //或者通过此方法得到RequestManager
      return supportFragmentGet(activity, fm, null /*parentHint*/);
    }
  }

  public RequestManager get(@NonNull Fragment fragment) {
    Preconditions.checkNotNull(fragment.getActivity(),
          "You cannot start a load on a fragment before it is attached or after it is destroyed");
    if (Util.isOnBackgroundThread()) {
      return get(fragment.getActivity().getApplicationContext());
    } else {
      FragmentManager fm = fragment.getChildFragmentManager();
      return supportFragmentGet(fragment.getActivity(), fm, fragment);
    }
  }

  public RequestManager get(@NonNull Activity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      android.app.FragmentManager fm = activity.getFragmentManager();
      return fragmentGet(activity, fm, null /*parentHint*/);
    }
  }

  public RequestManager get(@NonNull View view) {
    if (Util.isOnBackgroundThread()) {
      return get(view.getContext().getApplicationContext());
    }

    Preconditions.checkNotNull(view);
    Preconditions.checkNotNull(view.getContext(),
        "Unable to obtain a request manager for a view without a Context");
    Activity activity = findActivity(view.getContext());
    // The view might be somewhere else, like a service.
    if (activity == null) {
      return get(view.getContext().getApplicationContext());
    }

    // Support Fragments.
    // Although the user might have non-support Fragments attached to FragmentActivity, searching
    // for non-support Fragments is so expensive pre O and that should be rare enough that we
    // prefer to just fall back to the Activity directly.
    if (activity instanceof FragmentActivity) {
      Fragment fragment = findSupportFragment(view, (FragmentActivity) activity);
      return fragment != null ? get(fragment) : get(activity);
    }

    // Standard Fragments.
    android.app.Fragment fragment = findFragment(view, activity);
    if (fragment == null) {
      return get(activity);
    }
    return get(fragment);
  }

private RequestManager supportFragmentGet(@NonNull Context context, @NonNull FragmentManager fm,
      @Nullable Fragment parentHint) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
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

private RequestManager getApplicationManager(@NonNull Context context) {
  // Either an application context or we're on a background thread.
  if (applicationManager == null) {
    synchronized (this) {
      if (applicationManager == null) {
        // Normally pause/resume is taken care of by the fragment we add to the fragment or
        // activity. However, in this case since the manager attached to the application will not
        // receive lifecycle events, we must force the manager to start resumed using
        // ApplicationLifecycle.

        // TODO(b/27524013): Factor out this Glide.get() call.
          //这个是初始化Glide，稍后会讲到。
        Glide glide = Glide.get(context.getApplicationContext());
          //设置一个Application级别的RequestManager来管理Glide
        applicationManager =
            factory.build(
                glide,
                new ApplicationLifecycle(),//绑定生命周期
                new EmptyRequestManagerTreeNode(),
                context.getApplicationContext());
      }
    }
  }
  return applicationManager;
}
//至此Glide初始化完成。 
Glide只有一个单例，但是RequestManager有多个，一个Application的、一个Activity的
```

2.load(url);//做配置设置地址等

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
  return asDrawable().load(string);
}

public RequestBuilder<Drawable> asDrawable() {
  return as(Drawable.class);//传入Drawable类
}

public <ResourceType> RequestBuilder<ResourceType> as(
    @NonNull Class<ResourceType> resourceClass) {
    //传入glide对象，drawable类以及上下文生成 RequestBuilder
    return new RequestBuilder<>(glide, this, resourceClass, context);
}
protected RequestBuilder(Glide glide, RequestManager requestManager,
      Class<TranscodeType> transcodeClass, Context context) {
    this.glide = glide;
    this.requestManager = requestManager;
    this.transcodeClass = transcodeClass;
    this.defaultRequestOptions = requestManager.getDefaultRequestOptions();//默认请求的配置
    this.context = context;
    this.transitionOptions = requestManager.getDefaultTransitionOptions(transcodeClass);//动画配置
    this.requestOptions = defaultRequestOptions;//请求的配置
    this.glideContext = glide.getGlideContext();
  }

public RequestBuilder<TranscodeType> load(@Nullable Bitmap bitmap) {
    //不保存到SD卡
  return loadGeneric(bitmap).apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
}
public RequestBuilder<TranscodeType> load(@Nullable Drawable drawable) {
    //不保存到SD卡
  return loadGeneric(drawable).apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
}
public RequestBuilder<TranscodeType> load(@RawRes @DrawableRes @Nullable Integer resourceId) {
	return loadGeneric(resourceId).apply(signatureOf(ApplicationVersionSignature.obtain(context)));
}
public RequestBuilder<TranscodeType> load(@Nullable byte[] model) {
  RequestBuilder<TranscodeType> result = loadGeneric(model);   
  if (!result.requestOptions.isDiskCacheStrategySet()) {
     result = result.apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
  }
  if (!result.requestOptions.isSkipMemoryCacheSet()) {
    result = result.apply(skipMemoryCacheOf(true /*skipMemoryCache*/));
  }
  return result;
}
//获取到了RequestBuilder，该执行 return asDrawable().load(string);
public RequestBuilder<TranscodeType> load(@Nullable String string) {
  return loadGeneric(string);
}
public RequestBuilder<TranscodeType> load(@Nullable byte[] model) {
    RequestBuilder<TranscodeType> result = loadGeneric(model);
    if (!result.requestOptions.isDiskCacheStrategySet()) {
        result = result.apply(diskCacheStrategyOf(DiskCacheStrategy.NONE));
    }
    if (!result.requestOptions.isSkipMemoryCacheSet()) {
      result = result.apply(skipMemoryCacheOf(true /*skipMemoryCache*/));
    }
    return result;
  }

private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
  this.model = model; 
  isModelSet = true;    
  return this;
}
//新建RequestOptions
public static RequestOptions diskCacheStrategyOf(@NonNull DiskCacheStrategy diskCacheStrategy) {
  return new RequestOptions().diskCacheStrategy(diskCacheStrategy);
}
//使配置生效
public RequestBuilder<TranscodeType> apply(@NonNull RequestOptions requestOptions) {
  Preconditions.checkNotNull(requestOptions);
  this.requestOptions = getMutableOptions().apply(requestOptions);
  return this;
}


```

3.into(imageView1)；

```java
public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
	return into(target, /*targetListener=*/ null);
}

@Synthetic <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener) {
  return into(target, targetListener, getMutableOptions());
}
//获取默认配置
protected RequestOptions getMutableOptions() {
    return defaultRequestOptions == this.requestOptions? this.requestOptions.clone() :this.requestOptions;
}

private <Y extends Target<TranscodeType>> Y into(
    @NonNull Y target,
    @Nullable RequestListener<TranscodeType> targetListener,
    @NonNull RequestOptions options) {
  Util.assertMainThread();//判断是否是主线程，如果不是抛出异常。
  Preconditions.checkNotNull(target);//检查view是否为空，为空抛出异常。
  if (!isModelSet) {//判断是否设置了model即图片地址或者url等。
    throw new IllegalArgumentException("You must call #load() before calling #into()");
  }

  options = options.autoClone();//锁定
  Request request = buildRequest(target, targetListener, options);//开始构建

  Request previous = target.getRequest();//获取请求对象
    
  if (request.isEquivalentTo(previous)//判断请求是否重复
      //判断是否检查内存，并且是否同一个请求是否下载完成。
      && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
    request.recycle();//回收请求
    // If the request is completed, beginning again will ensure the result is re-delivered,
    // triggering RequestListeners and Targets. If the request is failed, beginning again will
    // restart the request, giving it another chance to complete. If the request is already
    // running, we can let it continue running without interruption.
    if (!Preconditions.checkNotNull(previous).isRunning()) {//如果没有在加载中
      // Use the previous request rather than the new one to allow for optimizations like skipping
      // setting placeholders, tracking and un-tracking Targets, and obtaining View dimensions
      // that are done in the individual Request.
      previous.begin();//开始加载
    }
    return target;
  }

  requestManager.clear(target);
  target.setRequest(request);//设置请求对象
  requestManager.track(target, request);// 开始加载

  return target;
}


private Request buildRequest(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      RequestOptions requestOptions) {
    return buildRequestRecursive(
        target,
        targetListener,
        /*parentCoordinator=*/ null,
        transitionOptions,
        requestOptions.getPriority(),
        requestOptions.getOverrideWidth(),
        requestOptions.getOverrideHeight(),
        requestOptions);
  }

  private Request buildRequestRecursive(
      Target<TranscodeType> target,
      @Nullable RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      RequestOptions requestOptions) {

    // Build the ErrorRequestCoordinator first if necessary so we can update parentCoordinator.
    ErrorRequestCoordinator errorRequestCoordinator = null;
    if (errorBuilder != null) {
      errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
      parentCoordinator = errorRequestCoordinator;
    }
//开始封装网络请求
    Request mainRequest =
        buildThumbnailRequestRecursive(
            target,
            targetListener,
            parentCoordinator,
            transitionOptions,
            priority,
            overrideWidth,
            overrideHeight,
            requestOptions);

    if (errorRequestCoordinator == null) {
      return mainRequest;
    }

    int errorOverrideWidth = errorBuilder.requestOptions.getOverrideWidth();
    int errorOverrideHeight = errorBuilder.requestOptions.getOverrideHeight();
    if (Util.isValidDimensions(overrideWidth, overrideHeight)
        && !errorBuilder.requestOptions.isValidOverride()) {
      errorOverrideWidth = requestOptions.getOverrideWidth();
      errorOverrideHeight = requestOptions.getOverrideHeight();
    }

    Request errorRequest = errorBuilder.buildRequestRecursive(
        target,
        targetListener,
        errorRequestCoordinator,
        errorBuilder.transitionOptions,
        errorBuilder.requestOptions.getPriority(),
        errorOverrideWidth,
        errorOverrideHeight,
        errorBuilder.requestOptions);
    errorRequestCoordinator.setRequests(mainRequest, errorRequest);
    return errorRequestCoordinator;
  }


 private Request buildThumbnailRequestRecursive(
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      @Nullable RequestCoordinator parentCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight,
      RequestOptions requestOptions) {
    if (thumbnailBuilder != null) {
      // Recursive case: contains a potentially recursive thumbnail request builder.
      if (isThumbnailBuilt) {
        throw new IllegalStateException("You cannot use a request as both the main request and a "
            + "thumbnail, consider using clone() on the request(s) passed to thumbnail()");
      }

      TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions =
          thumbnailBuilder.transitionOptions;

      // Apply our transition by default to thumbnail requests but avoid overriding custom options
      // that may have been applied on the thumbnail request explicitly.
      if (thumbnailBuilder.isDefaultTransitionOptionsSet) {
        thumbTransitionOptions = transitionOptions;
      }

      Priority thumbPriority = thumbnailBuilder.requestOptions.isPrioritySet()
          ? thumbnailBuilder.requestOptions.getPriority() : getThumbnailPriority(priority);

      int thumbOverrideWidth = thumbnailBuilder.requestOptions.getOverrideWidth();
      int thumbOverrideHeight = thumbnailBuilder.requestOptions.getOverrideHeight();
      if (Util.isValidDimensions(overrideWidth, overrideHeight)
          && !thumbnailBuilder.requestOptions.isValidOverride()) {
        thumbOverrideWidth = requestOptions.getOverrideWidth();
        thumbOverrideHeight = requestOptions.getOverrideHeight();
      }

      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
      Request fullRequest =
          obtainRequest(
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight);
      isThumbnailBuilt = true;
      // Recursively generate thumbnail requests.
      Request thumbRequest =
          thumbnailBuilder.buildRequestRecursive(
              target,
              targetListener,
              coordinator,
              thumbTransitionOptions,
              thumbPriority,
              thumbOverrideWidth,
              thumbOverrideHeight,
              thumbnailBuilder.requestOptions);
      isThumbnailBuilt = false;
      coordinator.setRequests(fullRequest, thumbRequest);
      return coordinator;
    } else if (thumbSizeMultiplier != null) {
      // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
      ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
        //得到request实例
      Request fullRequest =
          obtainRequest(
              target,
              targetListener,
              requestOptions,
              coordinator,
              transitionOptions,
              priority,
              overrideWidth,
              overrideHeight);
      RequestOptions thumbnailOptions = requestOptions.clone()
          .sizeMultiplier(thumbSizeMultiplier);
//得到request实例
      Request thumbnailRequest =
          obtainRequest(
              target,
              targetListener,
              thumbnailOptions,
              coordinator,
              transitionOptions,
              getThumbnailPriority(priority),
              overrideWidth,
              overrideHeight);

      coordinator.setRequests(fullRequest, thumbnailRequest);//设置请求
      return coordinator;
    } else {
      // Base case: no thumbnail.
      return obtainRequest(
          target,
          targetListener,
          requestOptions,
          parentCoordinator,
          transitionOptions,
          priority,
          overrideWidth,
          overrideHeight);
    }
  }
//得到request
private Request obtainRequest(
      Target<TranscodeType> target,
      RequestListener<TranscodeType> targetListener,
      RequestOptions requestOptions,
      RequestCoordinator requestCoordinator,
      TransitionOptions<?, ? super TranscodeType> transitionOptions,
      Priority priority,
      int overrideWidth,
      int overrideHeight) {
    return SingleRequest.obtain(
        context,
        glideContext,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListener,
        requestCoordinator,
      glideContext.getEngine(),
      transitionOptions.getTransitionFactory());
}
//得到request
public static <R> SingleRequest<R> obtain(
      Context context,
      GlideContext glideContext,
      Object model,
      Class<R> transcodeClass,
      RequestOptions requestOptions,
      int overrideWidth,
      int overrideHeight,
      Priority priority,
      Target<R> target,
      RequestListener<R> targetListener,
      RequestListener<R> requestListener,
      RequestCoordinator requestCoordinator,
      Engine engine,
      TransitionFactory<? super R> animationFactory) {
    @SuppressWarnings("unchecked") SingleRequest<R> request =
        (SingleRequest<R>) POOL.acquire();
    if (request == null) {
      request = new SingleRequest<>();
    }
    //初始化request，设置参数
    request.init(
        context,
        glideContext,
        model,
        transcodeClass,
        requestOptions,
        overrideWidth,
        overrideHeight,
        priority,
        target,
        targetListener,
        requestListener,
        requestCoordinator,
        engine,
        animationFactory);
    return request;
  }
//初始化request，设置参数
private void init(
      Context context,
      GlideContext glideContext,
      Object model,
      Class<R> transcodeClass,
      RequestOptions requestOptions,
      int overrideWidth,
      int overrideHeight,
      Priority priority,
      Target<R> target,
      RequestListener<R> targetListener,
      RequestListener<R> requestListener,
      RequestCoordinator requestCoordinator,
      Engine engine,
      TransitionFactory<? super R> animationFactory) {
    this.context = context;
    this.glideContext = glideContext;
    this.model = model;
    this.transcodeClass = transcodeClass;
    this.requestOptions = requestOptions;
    this.overrideWidth = overrideWidth;
    this.overrideHeight = overrideHeight;
    this.priority = priority;
    this.target = target;
    this.targetListener = targetListener;
    this.requestListener = requestListener;
    this.requestCoordinator = requestCoordinator;
    this.engine = engine;
    this.animationFactory = animationFactory;
    status = Status.PENDING;
  }


 @Override
  public boolean isEquivalentTo(Request o) {
    if (o instanceof ThumbnailRequestCoordinator) {
      ThumbnailRequestCoordinator that = (ThumbnailRequestCoordinator) o;
      return (full == null ? that.full == null : full.isEquivalentTo(that.full))
          && (thumb == null ? that.thumb == null : thumb.isEquivalentTo(that.thumb));
    }
    return false;
  }


//开始加载
public void track(Target<?> target) {
    targets.add(target);
  }
public void runRequest(Request request) {
    requests.add(request);
    if (!isPaused) {
      request.begin();
    } else {
      pendingRequests.add(request);
    }
  }

public void begin() {
    assertNotCallingCallbacks();
    stateVerifier.throwIfRecycled();//检查对象是否是可回收的。
    startTime = LogTime.getLogTime();
    if (model == null) {//如果被加载对象是null，则走failed
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        width = overrideWidth;
        height = overrideHeight;
      }
      // Only log at more verbose log levels if the user has set a fallback drawable, because
      // fallback Drawables indicate the user expects null models occasionally.
      int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      onLoadFailed(new GlideException("Received null model"), logLevel);
      return;
    }

    if (status == Status.RUNNING) {//如果正在加载中，则抛出异常
      throw new IllegalArgumentException("Cannot restart a running request");
    }

    // If we're restarted after we're complete (usually via something like a notifyDataSetChanged
    // that starts an identical request into the same Target or View), we can simply use the
    // resource and size we retrieved the last time around and skip obtaining a new size, starting a
    // new load etc. This does mean that users who want to restart a load because they expect that
    // the view size has changed will need to explicitly clear the View or Target before starting
    // the new load.
    if (status == Status.COMPLETE) {//如果加载完成。则开始回显
      onResourceReady(resource, DataSource.MEMORY_CACHE);
      return;
    }

    // Restarts for requests that are neither complete nor running can be treated as new requests
    // and can run again from the beginning.

    status = Status.WAITING_FOR_SIZE;//切换到等待设置尺寸状态
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
      onSizeReady(overrideWidth, overrideHeight);
    } else {
      target.getSize(this);
    }

    if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
      target.onLoadStarted(getPlaceholderDrawable());//开始加载
    }
    if (IS_VERBOSE_LOGGABLE) {
      logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
  }

public void onResourceReady(Resource<?> resource, DataSource dataSource) {
    stateVerifier.throwIfRecycled();
    loadStatus = null;
    if (resource == null) {
      GlideException exception = new GlideException("Expected to receive a Resource<R> with an "
          + "object of " + transcodeClass + " inside, but instead got null.");
      onLoadFailed(exception);
      return;
    }

    Object received = resource.get();
    if (received == null || !transcodeClass.isAssignableFrom(received.getClass())) {
      releaseResource(resource);//释放资源
      GlideException exception = new GlideException("Expected to receive an object of "
          + transcodeClass + " but instead" + " got "
          + (received != null ? received.getClass() : "") + "{" + received + "} inside" + " "
          + "Resource{" + resource + "}."
          + (received != null ? "" : " " + "To indicate failure return a null Resource "
          + "object, rather than a Resource object containing null data."));
      onLoadFailed(exception);
      return;
    }

    if (!canSetResource()) {
      releaseResource(resource);//释放资源
      // We can't put the status to complete before asking canSetResource().
      status = Status.COMPLETE;
      return;
    }

    onResourceReady((Resource<R>) resource, (R) received, dataSource);
  }

private void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    // We must call isFirstReadyResource before setting status.
    boolean isFirstResource = isFirstReadyResource();
    status = Status.COMPLETE;
    this.resource = resource;

    if (glideContext.getLogLevel() <= Log.DEBUG) {
      Log.d(GLIDE_TAG, "Finished loading " + result.getClass().getSimpleName() + " from "
          + dataSource + " for " + model + " with size [" + width + "x" + height + "] in "
          + LogTime.getElapsedMillis(startTime) + " ms");
    }

    isCallingCallbacks = true;
    try {
      if ((requestListener == null
          || !requestListener.onResourceReady(result, model, target, dataSource, isFirstResource))
          && (targetListener == null
          || !targetListener.onResourceReady(result, model, target, dataSource, isFirstResource))) {
        Transition<? super R> animation =
            animationFactory.build(dataSource, isFirstResource);//加载动画
        target.onResourceReady(result, animation);//开始读数据给imageview
      }
    } finally {
      isCallingCallbacks = false;
    }

    notifyLoadSuccess();//通知结果
  }
public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    if (IS_VERBOSE_LOGGABLE) {
      logV("Got onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
    if (status != Status.WAITING_FOR_SIZE) {
      return;
    }
    status = Status.RUNNING;

    float sizeMultiplier = requestOptions.getSizeMultiplier();
    this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
    this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

    if (IS_VERBOSE_LOGGABLE) {
      logV("finished setup for calling load in " + LogTime.getElapsedMillis(startTime));
    }
    loadStatus = engine.load(
        glideContext,
        model,
        requestOptions.getSignature(),
        this.width,
        this.height,
        requestOptions.getResourceClass(),
        transcodeClass,
        priority,
        requestOptions.getDiskCacheStrategy(),
        requestOptions.getTransformations(),
        requestOptions.isTransformationRequired(),
        requestOptions.isScaleOnlyOrNoTransform(),
        requestOptions.getOptions(),
        requestOptions.isMemoryCacheable(),
        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
        requestOptions.getUseAnimationPool(),
        requestOptions.getOnlyRetrieveFromCache(),
        this);

    // This is a hack that's only useful for testing right now where loads complete synchronously
    // even though under any executor running on any thread but the main thread, the load would
    // have completed asynchronously.
    if (status != Status.RUNNING) {
      loadStatus = null;
    }
    if (IS_VERBOSE_LOGGABLE) {
      logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
  }

public <R> LoadStatus load(
      GlideContext glideContext,
      Object model,
      Key signature,
      int width,
      int height,
      Class<?> resourceClass,
      Class<R> transcodeClass,
      Priority priority,
      DiskCacheStrategy diskCacheStrategy,
      Map<Class<?>, Transformation<?>> transformations,
      boolean isTransformationRequired,
      boolean isScaleOnlyOrNoTransform,
      Options options,
      boolean isMemoryCacheable,
      boolean useUnlimitedSourceExecutorPool,
      boolean useAnimationPool,
      boolean onlyRetrieveFromCache,
      ResourceCallback cb) {
    Util.assertMainThread();
    long startTime = LogTime.getLogTime();
//生成图片对应的key，判断缓存用
    EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);
//从激活的资源文件中加载
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
      cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      return null;
    }
//从MemoryCache加载
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
      cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      return null;
    }
//从Jobs map中加载
    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Added to existing load", startTime, key);
      }
      return new LoadStatus(cb, current);
    }

    EngineJob<R> engineJob =
        engineJobFactory.build(
            key,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache);

    DecodeJob<R> decodeJob =
        decodeJobFactory.build(
            glideContext,
            model,
            key,
            signature,
            width,
            height,
            resourceClass,
            transcodeClass,
            priority,
            diskCacheStrategy,
            transformations,
            isTransformationRequired,
            isScaleOnlyOrNoTransform,
            onlyRetrieveFromCache,
            options,
            engineJob);

    jobs.put(key, engineJob);

    engineJob.addCallback(cb);
    engineJob.start(decodeJob);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
  }

class EngineKeyFactory {
//生成图片对应的key，判断缓存用
  @SuppressWarnings("rawtypes")
  EngineKey buildKey(Object model, Key signature, int width, int height,
      Map<Class<?>, Transformation<?>> transformations, Class<?> resourceClass,
      Class<?> transcodeClass, Options options) {
    return new EngineKey(model, signature, width, height, transformations, resourceClass,
        transcodeClass, options);
  }
}

//ImageViewTarget类 
public void onLoadStarted(@Nullable Drawable placeholder) {
    super.onLoadStarted(placeholder);
    setResourceInternal(null);
    setDrawable(placeholder);
  }





```

