---
title: Glide图片加载库源码学习
date: 2021-05-07 10:30:05
tags: 图片加载框架
---
### Overview
> Glide is a fast and efficient open source media management and image loading framework for Android that wraps media decoding, memory and disk caching, and resource pooling into a simple and easy to use interface. ——官方介绍

Glide是Bump Technologies公司开源的一款图片加载框架。Glide这个单词的意思是”滑行“。

### 示例用法
``` java
  Glide.with(this).load("http://goo.gl/gEgYUd").into(imageView);
```
**依照示例用法的调用链学习一下Glide的源码**

### 1.Glide.with(Activity activity)
``` java
  public static RequestManager with(@NonNull FragmentActivity activity) {
    return getRetriever(activity).get(activity);
  }
```

### 1.1 getRetriever(Activity activity)的实现:
``` java
  @NonNull
  private static RequestManagerRetriever getRetriever(@Nullable Context context) {
    // ……
    return Glide.get(context).getRequestManagerRetriever();
  }
```
在这个方法中调用Glide.get()完成了Glide的全局初始化，后面的getRequestManagerRetriever()方法返回Glide的requestManagerRetriever对象

### 1.2 RequestManagerRetriever.get(Activity activity)
``` java
  @NonNull
  public RequestManager get(@NonNull FragmentActivity activity) {
    if (Util.isOnBackgroundThread()) {
      return get(activity.getApplicationContext());
    } else {
      assertNotDestroyed(activity);
      frameWaiter.registerSelf(activity);
      FragmentManager fm = activity.getSupportFragmentManager();
      return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
    }
  }
```
在这里判断了一下是不是后台线程，如果是后台线程，调用到同名方法，这里的原因暂时略过不谈；如果是主线程，会调用supportFragmentGet()方法：
``` java
  @NonNull
  private RequestManager supportFragmentGet(
      @NonNull Context context,
      @NonNull FragmentManager fm,
      @Nullable Fragment parentHint,
      boolean isParentVisible) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
     // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
      if (isParentVisible) {
        requestManager.onStart();
      }
      current.setRequestManager(requestManager);
    }
    return requestManager;
  }
```
这里又调用了一次Glide.get()，是为了确保Glide有被初始化成功
关键的代码是：
``` java
requestManager =
          factory.build(
              glide, current.getGlideLifecycle(), current.getRequestManagerTreeNode(), context);
```
这里创建出了requestManager

**小结：示例用法的第一环调用Glide.with()，完成了Glide的初始化工作，返回一个RequestManager**

### 2.RequestManager.load(String string)
``` java
  public RequestBuilder<Drawable> load(@Nullable String string) {
    return asDrawable().load(string);
  }
```
调用的是asDrawble()的load()方法

### 2.1 RequestManager.asDrawable()
``` java
  public RequestBuilder<Drawable> asDrawable() {
    return as(Drawable.class);
  }
```
转换成RequestBuilder，实现是：
``` java
  public <ResourceType> RequestBuilder<ResourceType> as(
      @NonNull Class<ResourceType> resourceClass) {
    return new RequestBuilder<>(glide, this, resourceClass, context);
  }
```
这里构造RequestBuilder对象时传入的resourceClass就是Drawable.class

### 2.2 RequestBuilder.load(String string)
``` java
  public RequestBuilder<TranscodeType> load(@Nullable String string) {
    return loadGeneric(string);
  }
```
调用loadGeneric()方法来实现
``` java
  @NonNull
  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    if (isAutoCloneEnabled()) {
      return clone().loadGeneric(model);
    }
    this.model = model;
    isModelSet = true;
    return selfOrThrowIfLocked();
  }
```
往下调用selfOrThrowIfLocked()
``` java
  protected final T selfOrThrowIfLocked() {
    if (isLocked) {
      throw new IllegalStateException("You cannot modify locked T, consider clone()");
    }
    return self();
  }
```
继续看self()的实现：
``` java
  private T self() {
    return (T) this;
  }
```
如方法名所说，就真的是拿到自身对象

**小结，RequestManager.load()方法并没有完成图片加载的工作，在这一步仅仅是构建出了RequestBuilder对象**

### 3.RequestBuilder.into(ImageView view)
``` java
  public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
    Util.assertMainThread();
    Preconditions.checkNotNull(view);

    BaseRequestOptions<?> requestOptions = this;
    if (!requestOptions.isTransformationSet()
        && requestOptions.isTransformationAllowed()
        && view.getScaleType() != null) {
      // Clone in this method so that if we use this RequestBuilder to load into a View and then
      // into a different target, we don't retain the transformation applied based on the previous
      // View's scale type.
      switch (view.getScaleType()) {
        case CENTER_CROP:
          requestOptions = requestOptions.clone().optionalCenterCrop();
          break;
        case CENTER_INSIDE:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case FIT_CENTER:
        case FIT_START:
        case FIT_END:
          requestOptions = requestOptions.clone().optionalFitCenter();
          break;
        case FIT_XY:
          requestOptions = requestOptions.clone().optionalCenterInside();
          break;
        case CENTER:
        case MATRIX:
        default:
          // Do nothing.
      }
    }

    return into(
        glideContext.buildImageViewTarget(view, transcodeClass),
        /*targetListener=*/ null,
        requestOptions,
        Executors.mainThreadExecutor());
  }
```
检查线程，然后对进行裁剪设置，最后调用同名的into()方法来进行后续操作：
``` java
  <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      Executor callbackExecutor) {
    return into(target, targetListener, /*options=*/ this, callbackExecutor);
  }
```
继续向下调用：
``` java
  private <Y extends Target<TranscodeType>> Y into(
      @NonNull Y target,
      @Nullable RequestListener<TranscodeType> targetListener,
      BaseRequestOptions<?> options,
      Executor callbackExecutor) {
    Preconditions.checkNotNull(target);
    if (!isModelSet) {
      throw new IllegalArgumentException("You must call #load() before calling #into()");
    }

    Request request = buildRequest(target, targetListener, options, callbackExecutor);

    Request previous = target.getRequest();
    if (request.isEquivalentTo(previous)
        && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
      if (!Preconditions.checkNotNull(previous).isRunning()) {
        previous.begin();
      }
      return target;
    }

    requestManager.clear(target);
    target.setRequest(request);
    requestManager.track(target, request);

    return target;
  }
```
这个方法有以下要点：
- 调用buildRequest方法创建Request
- 对比新创建出来的Request和target本来就有的Request是否一致并且isSkipMemoryCacheWithCompletePreviousRequest为假，
isSkipMemoryCacheWithCompletePreviousRequest的实现为：
``` java
    return !options.isMemoryCacheable() && previous.isComplete();
```
这种情况下直接使用previous这个Request
- 否则，将新创建出来的request设置到target的request上，并且requestManager开始track这个target和request

### 3.1 RequestManager.track(@NonNull Target\<?> target, @NonNull Request request)
``` java
  synchronized void track(@NonNull Target<?> target, @NonNull Request request) {
    targetTracker.track(target);
    requestTracker.runRequest(request);
  }
```
分成了两个tracker，主要关注下面的runRequest()函数的实现

``` java
  public void runRequest(@NonNull Request request) {
    requests.add(request);
    if (!isPaused) {
      request.begin();
    } else {
      request.clear();
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Paused, delaying request");
      }
      pendingRequests.add(request);
    }
  }
```
在这里会调用request的begin()方法真正开启一个请求，Request其实是一个接口，这里的实现类是ThumbnailRequestCoordinator

### 3.2 ThumbnailRequestCoordinator
首先看下这个类的结构
``` java
public class ThumbnailRequestCoordinator implements RequestCoordinator, Request {
  @Nullable private final RequestCoordinator parent;
  private final Object requestLock;

  private volatile Request full;
  private volatile Request thumb;
}
```
实际上包含了两个Request的类成员，full和thumb，从命名可以猜测分别是缩略图和大图对应的Request

接下来看下它的begin()方法的实现：
``` java
  @Override
  public void begin() {
    synchronized (requestLock) {
      isRunningDuringBegin = true;
      try {
        // If the request has completed previously, there's no need to restart both the full and the
        // thumb, we can just restart the full.
        if (fullState != RequestState.SUCCESS && thumbState != RequestState.RUNNING) {
          thumbState = RequestState.RUNNING;
          thumb.begin();
        }
        if (isRunningDuringBegin && fullState != RequestState.RUNNING) {
          fullState = RequestState.RUNNING;
          full.begin();
        }
      } finally {
        isRunningDuringBegin = false;
      }
    }
  }
```
会分别调用thumb和full的begin()方法；full的实现类是SingleRequest，接下来看下SingleRequest的begin()方法

### 3.3 SingleRequest.begin()
``` java
  public void begin() {
    synchronized (requestLock) {
      assertNotCallingCallbacks();
      stateVerifier.throwIfRecycled();
      startTime = LogTime.getLogTime();
      if (model == null) {
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
          width = overrideWidth;
          height = overrideHeight;
        }
        int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
        onLoadFailed(new GlideException("Received null model"), logLevel);
        return;
      }

      if (status == Status.RUNNING) {
        throw new IllegalArgumentException("Cannot restart a running request");
      }

      if (status == Status.COMPLETE) {
        onResourceReady(
            resource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
        return;
      }

      status = Status.WAITING_FOR_SIZE;
      if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
      } else {
        target.getSize(this);
      }

      if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
          && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
      }
      if (IS_VERBOSE_LOGGABLE) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
      }
    }
  }
```
在这个函数中会对当前状态进行判断，当处于COMPLETE时直接调用onResourceReady()方法并返回；正常来说会调用到onSizeReady()：
``` java
  @Override
  public void onSizeReady(int width, int height) {
    stateVerifier.throwIfRecycled();
    synchronized (requestLock) {
      // ……
      if (status != Status.WAITING_FOR_SIZE) {
        return;
      }
      status = Status.RUNNING;

      float sizeMultiplier = requestOptions.getSizeMultiplier();
      this.width = maybeApplySizeMultiplier(width, sizeMultiplier);
      this.height = maybeApplySizeMultiplier(height, sizeMultiplier);

      // ……
      loadStatus =
          engine.load(
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
              this,
              callbackExecutor);
      if (status != Status.RUNNING) {
        loadStatus = null;
      }
      // ……
      }
    }
  }
```
交给Engine的load()方法来进行
``` java
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
      ResourceCallback cb,
      Executor callbackExecutor) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;

    EngineKey key =
        keyFactory.buildKey(
            model,
            signature,
            width,
            height,
            transformations,
            resourceClass,
            transcodeClass,
            options);

    EngineResource<?> memoryResource;
    synchronized (this) {
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(
            glideContext,
            model,
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
            options,
            isMemoryCacheable,
            useUnlimitedSourceExecutorPool,
            useAnimationPool,
            onlyRetrieveFromCache,
            cb,
            callbackExecutor,
            key,
            startTime);
      }
    }

    cb.onResourceReady(
        memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;
  }
```
首先会尝试从内存缓存中取图片，如果取不到，将调用waitForExistingOrStartNewJob()方法来获取图片
``` java
  private <R> LoadStatus waitForExistingOrStartNewJob(
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
      ResourceCallback cb,
      Executor callbackExecutor,
      EngineKey key,
      long startTime) {

    EngineJob<?> current = jobs.get(key, onlyRetrieveFromCache);
    if (current != null) {
      current.addCallback(cb, callbackExecutor);
      if (VERBOSE_IS_LOGGABLE) {
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

    engineJob.addCallback(cb, callbackExecutor);
    engineJob.start(decodeJob);
    // ……
    return new LoadStatus(cb, engineJob);
  }
```
创建了两个job，一个engineJob一个decodeJob，engineJob的作用是通过start()方法来执行这个decodeJob

``` java
  public synchronized void start(DecodeJob<R> decodeJob) {
    this.decodeJob = decodeJob;
    GlideExecutor executor =
        decodeJob.willDecodeFromCache() ? diskCacheExecutor : getActiveSourceExecutor();
    executor.execute(decodeJob);
  }
```
这里会进行判断decodeJob是否是从磁盘缓存拿到的资源，从而使用不同的GlideExecutor；随后调用executor来execute这个decodeJob：
``` java
  @Override
  public void execute(@NonNull Runnable command) {
    delegate.execute(command);
  }
```
这里delegate的实现类是ThreadPoolExecutor；因此关注下传入的decodeJob的run()方法：

``` java
  public void run() {
    DataFetcher<?> localFetcher = currentFetcher;
    try {
      // ……
      runWrapped();
    } catch (CallbackException e) {
      throw e;
    } catch (Throwable t) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        // ……
      }
      if (stage != Stage.ENCODE) {
        throwables.add(t);
        notifyFailed();
      }
      if (!isCancelled) {
        throw t;
      }
      throw t;
    } finally {
      if (localFetcher != null) {
        localFetcher.cleanup();
      }
      GlideTrace.endSection();
    }
  }
```
真正的工作在runWrapped()函数中：
``` java
  private void runWrapped() {
    switch (runReason) {
      case INITIALIZE:
        stage = getNextStage(Stage.INITIALIZE);
        currentGenerator = getNextGenerator();
        runGenerators();
        break;
      case SWITCH_TO_SOURCE_SERVICE:
        runGenerators();
        break;
      case DECODE_DATA:
        decodeFromRetrievedData();
        break;
      default:
        throw new IllegalStateException("Unrecognized run reason: " + runReason);
    }
  }
```
主要分两步：
- getNextGenerator()
- runGenerators()

### 3.3.1 getNextGenerator()
``` java
  private DataFetcherGenerator getNextGenerator() {
    switch (stage) {
      case RESOURCE_CACHE:
        return new ResourceCacheGenerator(decodeHelper, this);
      case DATA_CACHE:
        return new DataCacheGenerator(decodeHelper, this);
      case SOURCE:
        return new SourceGenerator(decodeHelper, this);
      case FINISHED:
        return null;
      default:
        throw new IllegalStateException("Unrecognized stage: " + stage);
    }
  }
```
这个方法的作用是根据当前不同的阶段，获取不同的Generator()

### 3.3.2 runGenerators()
``` java
private void runGenerators() {
    currentThread = Thread.currentThread();
    startFetchTime = LogTime.getLogTime();
    boolean isStarted = false;
    while (!isCancelled
        && currentGenerator != null
        && !(isStarted = currentGenerator.startNext())) {
      stage = getNextStage(stage);
      currentGenerator = getNextGenerator();

      if (stage == Stage.SOURCE) {
        reschedule();
        return;
      }
    }
    if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      notifyFailed();
    }
  }
```
这里会触发到currentGenerator的startNext()函数，以上面出现的SourceGenerator为例，看下这个startNext()的内容：
``` java
 public boolean startNext() {
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      try {
        boolean isDataInCache = cacheData(data);
        if (!isDataInCache) {
          return true;
        }
      } catch (IOException e) {
        // ……
      }
    }

    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
              || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        startNextLoad(loadData);
      }
    }
    return started;
  }
```
首先会判断是否命中缓存，命中则直接返回，否则最后会调用到startNextLoad()函数：
``` java
  private void startNextLoad(final LoadData<?> toStart) {
    loadData.fetcher.loadData(
        helper.getPriority(),
        new DataCallback<Object>() {
          @Override
          public void onDataReady(@Nullable Object data) {
            if (isCurrentRequest(toStart)) {
              onDataReadyInternal(toStart, data);
            }
          }

          @Override
          public void onLoadFailed(@NonNull Exception e) {
            if (isCurrentRequest(toStart)) {
              onLoadFailedInternal(toStart, e);
            }
          }
        });
  }
```
在startNextLoad()函数中，使用loadData.fetcher.loadData()进行了图片加载操作；对于网络图片而言，使用的fetcher就是HttpUrlFetcher

### HttpUrlFetcher.loadData()
``` java
  public void loadData(
      @NonNull Priority priority, @NonNull DataCallback<? super InputStream> callback) {
    long startTime = LogTime.getLogTime();
    try {
      InputStream result = loadDataWithRedirects(glideUrl.toURL(), 0, null, glideUrl.getHeaders());
      callback.onDataReady(result);
    } catch (IOException e) {
      // ……
      callback.onLoadFailed(e);
    } finally {
      // ……
      }
    }
  }
```
使用loadDataWithRedirects()方法来进行图片请求，得到InputStream
``` java
  private InputStream loadDataWithRedirects(
      URL url, int redirects, URL lastUrl, Map<String, String> headers) throws HttpException {
    // ……

    urlConnection = buildAndConfigureConnection(url, headers);

    try {
      stream = urlConnection.getInputStream();
    } catch (IOException e) {
      // ……
    }

    if (isCancelled) {
      return null;
    }

    final int statusCode = getHttpStatusCodeOrInvalid(urlConnection);
    if (isHttpOk(statusCode)) {
      return getStreamForSuccessfulRequest(urlConnection);
    } else if (isHttpRedirect(statusCode)) {
      // ……
    } else if (statusCode == INVALID_STATUS_CODE) {
      throw new HttpException(statusCode);
    } else {
      // ……
    }
  }
```
从这个方法能看出，对于网络图片，Glide是通过HttpUrlConnection来进行请求的，最后会返回HttpUrlConnection得到的InputStream。

**通过HttpUrlConnection得到图片的InputStream知道，就是图片的解码操作，最终把图片加载到控件上**







