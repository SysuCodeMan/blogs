---
title: Fresco图片加载库源码学习
date: 2021-05-07 16:40:48
tags: 图片加载框架
---
### Overview
> Fresco is a powerful system for displaying images in Android applications. ——官方说明

Fresco是Facebook公司开源的一款图片加载框架。Fresco一词的含义是”壁画“。

### 示例用法
进行全局初始化
``` java
public class MyApplication extends Application {
	@Override
	public void onCreate() {
		super.onCreate();
		Fresco.initialize(this);
	}
}
```
作为控件使用
``` xml
<com.facebook.drawee.view.SimpleDraweeView
    android:id="@+id/my_image_view"
    android:layout_width="130dp"
    android:layout_height="130dp"
    fresco:placeholderImage="@drawable/my_drawable"
  />
```
设置网络图片源：
``` java
Uri uri = Uri.parse("https://raw.githubusercontent.com/facebook/fresco/gh-pages/static/logo.png");
SimpleDraweeView draweeView = (SimpleDraweeView) findViewById(R.id.my_image_view);
draweeView.setImageURI(uri);
```

从初始化开始进行源码的学习
### 1.Fresco.initialize(Context context)
``` java
  public static void initialize(Context context) {
    initialize(context, null, null);
  }
```
``` java
  public static void initialize(
      Context context,
      @Nullable ImagePipelineConfig imagePipelineConfig,
      @Nullable DraweeConfig draweeConfig) {
    initialize(context, imagePipelineConfig, draweeConfig, true);
  }
```
``` java
  public static void initialize(
      Context context,
      @Nullable ImagePipelineConfig imagePipelineConfig,
      @Nullable DraweeConfig draweeConfig,
      boolean useNativeCode) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("Fresco#initialize");
    }
    if (sIsInitialized) {
      FLog.w(
          TAG,
          "Fresco has already been initialized! `Fresco.initialize(...)` should only be called "
              + "1 single time to avoid memory leaks!");
    } else {
      sIsInitialized = true;
    }

    NativeCodeSetup.setUseNativeCode(useNativeCode);

    if (!NativeLoader.isInitialized()) {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("Fresco.initialize->SoLoader.init");
      }
      try {
        Class<?> clazz =
            Class.forName("com.facebook.imagepipeline.nativecode.NativeCodeInitializer");
        clazz.getMethod("init", Context.class).invoke(null, context);
      } catch (ClassNotFoundException e) {
        // Failed to initialize SoLoader
        NativeLoader.initIfUninitialized(new SystemDelegate());
      } catch (IllegalAccessException e) {
        // Failed to initialize SoLoader
        NativeLoader.initIfUninitialized(new SystemDelegate());
      } catch (InvocationTargetException e) {
        // Failed to initialize SoLoader
        NativeLoader.initIfUninitialized(new SystemDelegate());
      } catch (NoSuchMethodException e) {
        // Failed to initialize SoLoader
        NativeLoader.initIfUninitialized(new SystemDelegate());
      } finally {
        if (FrescoSystrace.isTracing()) {
          FrescoSystrace.endSection();
        }
      }
    }
    // we should always use the application context to avoid memory leaks
    context = context.getApplicationContext();
    if (imagePipelineConfig == null) {
      ImagePipelineFactory.initialize(context);
    } else {
      ImagePipelineFactory.initialize(imagePipelineConfig);
    }
    initializeDrawee(context, draweeConfig);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
在Fresco的init()中，主要有以下部分：
- 通过Class.forName()将com.facebook.imagepipeline.nativecode.NativeCodeInitializer加载进来
- 调用ImagePipelineFactory.initialize()进行初始化
- 调用initializeDrawee()进行初始化

### 1.1 ImagePipelineFactory.initialize()
``` java
  public static synchronized void initialize(Context context) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ImagePipelineFactory#initialize");
    }
    initialize(ImagePipelineConfig.newBuilder(context).build());
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
调用的是同名函数:
``` java
  public static synchronized void initialize(ImagePipelineConfigInterface imagePipelineConfig) {
    if (sInstance != null) {
      FLog.w(
          TAG,
          "ImagePipelineFactory has already been initialized! `ImagePipelineFactory.initialize(...)` should only be called once to avoid unexpected behavior.");
    }

    sInstance = new ImagePipelineFactory(imagePipelineConfig);
  }
```
这里是个单例模式，创建出ImagePipelineFactory的实例

### 1.2 Fresco.initializeDrawee()
``` java
  private static void initializeDrawee(Context context, @Nullable DraweeConfig draweeConfig) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("Fresco.initializeDrawee");
    }
    sDraweeControllerBuilderSupplier =
        new PipelineDraweeControllerBuilderSupplier(context, draweeConfig);
    SimpleDraweeView.initialize(sDraweeControllerBuilderSupplier);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
创建出PipelineDraweeControllerBuilderSupplier实例，随后调用SimpleDraweeView的initialize()函数

``` java
  public static void initialize(
      Supplier<? extends AbstractDraweeControllerBuilder> draweeControllerBuilderSupplier) {
    sDraweecontrollerbuildersupplier = draweeControllerBuilderSupplier;
  }
```
将传入的draweeControllerBuilderSupplier赋值给成员变量

**上面就是Fresco的初始化过程，接下来看下使用时调用draweeView.setImageURI(uri)是如何完成网络图片的加载的**
### 2. SimpleDraweeView.setImageURI(uri)
``` java
  public void setImageURI(Uri uri) {
    setImageURI(uri, null);
  }
```
``` java
  public void setImageURI(Uri uri, @Nullable Object callerContext) {
    DraweeController controller =
        mControllerBuilder
            .setCallerContext(callerContext)
            .setUri(uri)
            .setOldController(getController())
            .build();
    setController(controller);
  }
```
首先是通过builder模式构造出一个DraweeController，然后调用setController()方法

``` java
  public void setController(@Nullable DraweeController draweeController) {
    mDraweeHolder.setController(draweeController);
    super.setImageDrawable(mDraweeHolder.getTopLevelDrawable());
  }
```
这里有两部分：
- DraweeHolder.setController()
- 这里的super就是原生的ImageView，因此关注的应该是后面mDraweeHolder.getTopLevelDrawable()的实现

### 2.1 DraweeHolder.setController()
``` java
  public void setController(@Nullable DraweeController draweeController) {
    boolean wasAttached = mIsControllerAttached;
    if (wasAttached) {
      detachController();
    }

    // Clear the old controller
    if (isControllerValid()) {
      mEventTracker.recordEvent(Event.ON_CLEAR_OLD_CONTROLLER);
      mController.setHierarchy(null);
    }
    mController = draweeController;
    if (mController != null) {
      mEventTracker.recordEvent(Event.ON_SET_CONTROLLER);
      mController.setHierarchy(mHierarchy);
    } else {
      mEventTracker.recordEvent(Event.ON_CLEAR_CONTROLLER);
    }

    if (wasAttached) {
      attachController();
    }
  }
```
这里主要的工作是调用attchController():

``` java
  private void attachController() {
    if (mIsControllerAttached) {
      return;
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    mIsControllerAttached = true;
    if (mController != null && mController.getHierarchy() != null) {
      mController.onAttach();
    }
  }
```
触发的是mController的onAttach()回调，这里的mController实现类是PipelineDraweeController，但PipelineDraweeController没有重写onAttach()，看下PipelineDraweeController父类AbstractDraweeController的onAttach()函数：
``` java
  public void onAttach() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractDraweeController#onAttach");
    }
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(
          TAG,
          "controller %x %s: onAttach: %s",
          System.identityHashCode(this),
          mId,
          mIsRequestSubmitted ? "request already submitted" : "request needs submit");
    }
    mEventTracker.recordEvent(Event.ON_ATTACH_CONTROLLER);
    Preconditions.checkNotNull(mSettableDraweeHierarchy);
    mDeferredReleaser.cancelDeferredRelease(this);
    mIsAttached = true;
    if (!mIsRequestSubmitted) {
      submitRequest();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
在onAttach()中会调用submitRequest()方法，从方法名中可以猜测这个方法的作用是用来发起图片请求的
``` java
  protected void submitRequest() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractDraweeController#submitRequest");
    }
    final T closeableImage = getCachedImage();
    if (closeableImage != null) {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.beginSection("AbstractDraweeController#submitRequest->cache");
      }
      mDataSource = null;
      mIsRequestSubmitted = true;
      mHasFetchFailed = false;
      mEventTracker.recordEvent(Event.ON_SUBMIT_CACHE_HIT);
      reportSubmit(mDataSource, getImageInfo(closeableImage));
      onImageLoadedFromCacheImmediately(mId, closeableImage);
      onNewResultInternal(mId, mDataSource, closeableImage, 1.0f, true, true, true);
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
      return;
    }
    mEventTracker.recordEvent(Event.ON_DATASOURCE_SUBMIT);
    mSettableDraweeHierarchy.setProgress(0, true);
    mIsRequestSubmitted = true;
    mHasFetchFailed = false;
    mDataSource = getDataSource();
    reportSubmit(mDataSource, null);
    // ……
    final String id = mId;
    final boolean wasImmediate = mDataSource.hasResult();
    final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
    mDataSource.subscribe(dataSubscriber, mUiThreadImmediateExecutor);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
函数略长，分成以下几个部分：
- 1.通过getCachedImage()方法尝试从内存缓存中读取图片，如果能拿到，则调用相关回调后直接返回
- 2.如果通过getCachedImage()拿不到缓存的图片，则调用getDataSource()方法获取数据源mDataSource
- 3.定义DataSubscriber，随后通过mDataSource.subscribe()传入DataSubscriber进行数据的订阅操作
下面具体看下这3部分的实现：

### 2.1.1 PipelineDraweeController.getCachedImage()
``` java
  protected @Nullable CloseableReference<CloseableImage> getCachedImage() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("PipelineDraweeController#getCachedImage");
    }
    try {
      if (mMemoryCache == null || mCacheKey == null) {
        return null;
      }
      // We get the CacheKey
      CloseableReference<CloseableImage> closeableImage = mMemoryCache.get(mCacheKey);
      if (closeableImage != null && !closeableImage.get().getQualityInfo().isOfFullQuality()) {
        closeableImage.close();
        return null;
      }
      return closeableImage;
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }
```
这里的CloseableImage是对Bitmap的一个封装，通过它可以获得我们需要的Bitmap；这里内存缓存实现的策略是常见的LRU策略，底层是LinkedHashMap实现

### 2.1.2 PipelineDraweeController.getDataSource()
``` java
  protected DataSource<CloseableReference<CloseableImage>> getDataSource() {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("PipelineDraweeController#getDataSource");
    }
    if (FLog.isLoggable(FLog.VERBOSE)) {
      FLog.v(TAG, "controller %x: getDataSource", System.identityHashCode(this));
    }
    DataSource<CloseableReference<CloseableImage>> result = mDataSourceSupplier.get();
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return result;
  }
```
这里的mDataSourceSupplier的实现类是一个匿名类，实现在AbstractDraweeControllerBuilder中：
``` java
  protected Supplier<DataSource<IMAGE>> getDataSourceSupplierForRequest(
      final DraweeController controller,
      final String controllerId,
      final REQUEST imageRequest,
      final CacheLevel cacheLevel) {
    final Object callerContext = getCallerContext();
    return new Supplier<DataSource<IMAGE>>() {
      @Override
      public DataSource<IMAGE> get() {
        return getDataSourceForRequest(
            controller, controllerId, imageRequest, callerContext, cacheLevel);
      }

      @Override
      public String toString() {
        return Objects.toStringHelper(this).add("request", imageRequest.toString()).toString();
      }
    };
  }
```
从这里可以看出，对mDataSourceSupplier调用get()方法，实际上是调用到getDataSourceForRequest()方法，而这个方法是一个抽象方法，在PipelineDraweeController使用的实现是：
``` java
  @Override
  protected DataSource<CloseableReference<CloseableImage>> getDataSourceForRequest(
      DraweeController controller,
      String controllerId,
      ImageRequest imageRequest,
      Object callerContext,
      AbstractDraweeControllerBuilder.CacheLevel cacheLevel) {
    return mImagePipeline.fetchDecodedImage(
        imageRequest,
        callerContext,
        convertCacheLevelToRequestLevel(cacheLevel),
        getRequestListener(controller),
        controllerId);
  }
```

### 2.1.2.1 ImagePipeline.fetchDecodedImage()
``` java
  public DataSource<CloseableReference<CloseableImage>> fetchDecodedImage(
      ImageRequest imageRequest,
      Object callerContext,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    try {
      Producer<CloseableReference<CloseableImage>> producerSequence =
          mProducerSequenceFactory.getDecodedImageProducerSequence(imageRequest);
      return submitFetchRequest(
          producerSequence,
          imageRequest,
          lowestPermittedRequestLevelOnSubmit,
          callerContext,
          requestListener,
          uiComponentId);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    }
  }
```
在这个方法中，创建出了一个Producer<CloseableReference<CloseableImage>>对象，调用submitFetchRequest方法
``` java
  private <T> DataSource<CloseableReference<T>> submitFetchRequest(
      Producer<CloseableReference<T>> producerSequence,
      ImageRequest imageRequest,
      ImageRequest.RequestLevel lowestPermittedRequestLevelOnSubmit,
      Object callerContext,
      @Nullable RequestListener requestListener,
      @Nullable String uiComponentId) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("ImagePipeline#submitFetchRequest");
    }
    final RequestListener2 requestListener2 =
        new InternalRequestListener(
            getRequestListenerForRequest(imageRequest, requestListener), mRequestListener2);

    if (mCallerContextVerifier != null) {
      mCallerContextVerifier.verifyCallerContext(callerContext, false);
    }

    try {
      ImageRequest.RequestLevel lowestPermittedRequestLevel =
          ImageRequest.RequestLevel.getMax(
              imageRequest.getLowestPermittedRequestLevel(), lowestPermittedRequestLevelOnSubmit);
      SettableProducerContext settableProducerContext =
          new SettableProducerContext(
              imageRequest,
              generateUniqueFutureId(),
              uiComponentId,
              requestListener2,
              callerContext,
              lowestPermittedRequestLevel,
              /* isPrefetch */ false,
              imageRequest.getProgressiveRenderingEnabled()
                  || !UriUtil.isNetworkUri(imageRequest.getSourceUri()),
              imageRequest.getPriority(),
              mConfig);
      return CloseableProducerToDataSourceAdapter.create(
          producerSequence, settableProducerContext, requestListener2);
    } catch (Exception exception) {
      return DataSources.immediateFailedDataSource(exception);
    } finally {
      if (FrescoSystrace.isTracing()) {
        FrescoSystrace.endSection();
      }
    }
  }
```
这个方法里实际上没有执行图片的请求操作，从方法名里的submit也可以看出只是"提交这"个请求，接下来会调用CloseableProducerToDataSourceAdapter.create()返回：
``` java
  public static <T> DataSource<CloseableReference<T>> create(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener2 listener) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("CloseableProducerToDataSourceAdapter#create");
    }
    CloseableProducerToDataSourceAdapter<T> result =
        new CloseableProducerToDataSourceAdapter<T>(producer, settableProducerContext, listener);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    return result;
  }
```
构造CloseableProducerToDataSourceAdapter实例作为结果返回，看下CloseableProducerToDataSourceAdapter的构造函数：
``` java
  private CloseableProducerToDataSourceAdapter(
      Producer<CloseableReference<T>> producer,
      SettableProducerContext settableProducerContext,
      RequestListener2 listener) {
    super(producer, settableProducerContext, listener);
  }
```
调用的是超类的构造函数，超类是AbstractProducerToDataSourceAdapter
``` java
  protected AbstractProducerToDataSourceAdapter(
      Producer<T> producer,
      SettableProducerContext settableProducerContext,
      RequestListener2 requestListener) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()");
    }
    mSettableProducerContext = settableProducerContext;
    mRequestListener = requestListener;
    setInitialExtras();
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()->onRequestStart");
    }
    mRequestListener.onRequestStart(mSettableProducerContext);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("AbstractProducerToDataSourceAdapter()->produceResult");
    }
    producer.produceResults(createConsumer(), settableProducerContext);
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
AbstractProducerToDataSourceAdapter的构造函数中多数是日志记录的代码，最关键的代码其实是这一行：
``` java
    producer.produceResults(createConsumer(), settableProducerContext);

```
下面有两个点分别说一下：
- producer.produceResults()
- createConsumer()

### 2.1.2.2 Producer.produceResults()
在Fresco中，Producer是一个接口，负责真正的获取图片的工作，对于网络图片而言，实现类是NetworkFetchProducer：
``` java
  public void produceResults(Consumer<EncodedImage> consumer, ProducerContext context) {
    context.getProducerListener().onProducerStart(context, PRODUCER_NAME);
    final FetchState fetchState = mNetworkFetcher.createFetchState(consumer, context);
    mNetworkFetcher.fetch(
        fetchState,
        new NetworkFetcher.Callback() {
          @Override
          public void onResponse(InputStream response, int responseLength) throws IOException {
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.beginSection("NetworkFetcher->onResponse");
            }
            NetworkFetchProducer.this.onResponse(fetchState, response, responseLength);
            if (FrescoSystrace.isTracing()) {
              FrescoSystrace.endSection();
            }
          }

          @Override
          public void onFailure(Throwable throwable) {
            NetworkFetchProducer.this.onFailure(fetchState, throwable);
          }

          @Override
          public void onCancellation() {
            NetworkFetchProducer.this.onCancellation(fetchState);
          }
        });
  }
```
在这里，是交给mNetworkFetcher.fetch()方法完成网络请求，mNetworkFetcher是NetworkFetcher类型，NetworkFetcher是一个接口，在工程中能够找到以下实现类：
- BaseNetworkFetcher
- HttpUrlConnectionNetworkFetcher
- OkHttpNetworkFetcher
- PriorityNetworkFetcher
- RecordingNetworkFetcher
- VolleyNetworkFetcher

从命名也可以看出，他们是通过OkHttp/Volley等不同的网络请求框架实现的网络请求；这里我们不再关注内部的网络请求是如何实现的，只需要知道最后能够得到网络图片对应的InputStream
拿到InputStream之后，会调用onResponse()方法：
``` java
  protected void onResponse(
      FetchState fetchState, InputStream responseData, int responseContentLength)
      throws IOException {
    final PooledByteBufferOutputStream pooledOutputStream;
    if (responseContentLength > 0) {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream(responseContentLength);
    } else {
      pooledOutputStream = mPooledByteBufferFactory.newOutputStream();
    }
    final byte[] ioArray = mByteArrayPool.get(READ_SIZE);
    try {
      int length;
      while ((length = responseData.read(ioArray)) >= 0) {
        if (length > 0) {
          pooledOutputStream.write(ioArray, 0, length);
          maybeHandleIntermediateResult(pooledOutputStream, fetchState);
          float progress = calculateProgress(pooledOutputStream.size(), responseContentLength);
          fetchState.getConsumer().onProgressUpdate(progress);
        }
      }
      mNetworkFetcher.onFetchCompletion(fetchState, pooledOutputStream.size());
      handleFinalResult(pooledOutputStream, fetchState);
    } finally {
      mByteArrayPool.release(ioArray);
      pooledOutputStream.close();
    }
  }
```
这个方法有以下几个要点：
- 通过calculateProgress()方法计算出进度并通知给Consumer
- 当流数据读取完毕，调用handleFinalResult()方法
下面看下handleFinalResult()的实现：
``` java
  protected void handleFinalResult(
      PooledByteBufferOutputStream pooledOutputStream, FetchState fetchState) {
    Map<String, String> extraMap = getExtraMap(fetchState, pooledOutputStream.size());
    ProducerListener2 listener = fetchState.getListener();
    listener.onProducerFinishWithSuccess(fetchState.getContext(), PRODUCER_NAME, extraMap);
    listener.onUltimateProducerReached(fetchState.getContext(), PRODUCER_NAME, true);
    fetchState.getContext().putOriginExtra("network");
    notifyConsumer(
        pooledOutputStream,
        Consumer.IS_LAST | fetchState.getOnNewResultStatusFlags(),
        fetchState.getResponseBytesRange(),
        fetchState.getConsumer(),
        fetchState.getContext());
  }
```
这里会完成最重要的工作：调用notifyConsumer()来对数据进行消费

**小结，至此，已经从远端网络请求得到图片的数据流，后面是Consumer对这个流的处理工作，这就是前面createConsumer()方法的内容**

### 2.1.2.3 AbstractProducerToDataSourceAdapter.createConsumer()
``` java
  private Consumer<T> createConsumer() {
    return new BaseConsumer<T>() {
      @Override
      protected void onNewResultImpl(@Nullable T newResult, @Status int status) {
        AbstractProducerToDataSourceAdapter.this.onNewResultImpl(
            newResult, status, mSettableProducerContext);
      }

      @Override
      protected void onFailureImpl(Throwable throwable) {
        AbstractProducerToDataSourceAdapter.this.onFailureImpl(throwable);
      }

      @Override
      protected void onCancellationImpl() {
        AbstractProducerToDataSourceAdapter.this.onCancellationImpl();
      }

      @Override
      protected void onProgressUpdateImpl(float progress) {
        AbstractProducerToDataSourceAdapter.this.setProgress(progress);
      }
    };
  }
```
在onNewResultImpl()中回调AbstractProducerToDataSourceAdapter的onNewResultImpl()
``` java
  protected void onNewResultImpl(@Nullable T result, int status, ProducerContext producerContext) {
    boolean isLast = BaseConsumer.isLast(status);
    if (super.setResult(result, isLast, getExtras(producerContext))) {
      if (isLast) {
        mRequestListener.onRequestSuccess(mSettableProducerContext);
      }
    }
  }
```
会调用父类的setResult()方法，如果是最终结果，还会触发mRequestListener的回调；看下父类的setResult()方法：
``` java
protected boolean setResult(
      @Nullable T value, boolean isLast, @Nullable Map<String, Object> extras) {
    setExtras(extras);
    boolean result = setResultInternal(value, isLast);
    if (result) {
      notifyDataSubscribers();
    }
    return result;
  }
```
**这里很关键的一点，会调用notifyDataSubscribers()来通知观察者**




**小结上面就是PipelineDraweeController.getDataSource()部分的内容最终会拿到CloseableProducerToDataSourceAdapter这个DataSource；并且在获取构造DataSource的过程中已经完成了图片请求，接下来关注的是对DataSource的subscribe()方法调用**

### 2.1.3   DataSource.subscribe(final DataSubscriber\<T> dataSubscriber, final Executor executor)
``` java
  public void subscribe(final DataSubscriber<T> dataSubscriber, final Executor executor) {
    Preconditions.checkNotNull(dataSubscriber);
    Preconditions.checkNotNull(executor);
    boolean shouldNotify;

    synchronized (this) {
      if (mIsClosed) {
        return;
      }

      if (mDataSourceStatus == DataSourceStatus.IN_PROGRESS) {
        mSubscribers.add(Pair.create(dataSubscriber, executor));
      }

      shouldNotify = hasResult() || isFinished() || wasCancelled();
    }

    if (shouldNotify) {
      notifyDataSubscriber(dataSubscriber, executor, hasFailed(), wasCancelled());
    }
  }
```
会调用到notifyDataSubscriber()方法：
``` java
  protected void notifyDataSubscriber(
      final DataSubscriber<T> dataSubscriber,
      final Executor executor,
      final boolean isFailure,
      final boolean isCancellation) {
    Runnable runnable =
        new Runnable() {
          @Override
          public void run() {
            if (isFailure) {
              dataSubscriber.onFailure(AbstractDataSource.this);
            } else if (isCancellation) {
              dataSubscriber.onCancellation(AbstractDataSource.this);
            } else {
              dataSubscriber.onNewResult(AbstractDataSource.this);
            }
          }
        };
    final DataSourceInstrumenter instrumenter = getDataSourceInstrumenter();
    if (instrumenter != null) {
      runnable = instrumenter.decorateRunnable(runnable, "AbstractDataSource_notifyDataSubscriber");
    }
    executor.execute(runnable);
  }
```
- 首先是调用getDataSourceInstrumenter()方法尝试获取实现类的DataSourceInstrumenter
- 如果获得的DataSourceInstrumenter不为空，则对runnable进行decorateRunnable操作
- 使用传入的Executor执行runnable

在源码中看了一下getDataSourceInstrumenter()如果用户代码没有介入，默认拿到的应该是空，在execute()执行的，其实就是传入的dataSubscriber的各个回调方法：
``` java
final DataSubscriber<T> dataSubscriber =
        new BaseDataSubscriber<T>() {
          @Override
          public void onNewResultImpl(DataSource<T> dataSource) {
            // isFinished must be obtained before image, otherwise we might set intermediate result
            // as final image.
            boolean isFinished = dataSource.isFinished();
            boolean hasMultipleResults = dataSource.hasMultipleResults();
            float progress = dataSource.getProgress();
            T image = dataSource.getResult();
            if (image != null) {
              onNewResultInternal(
                  id, dataSource, image, progress, isFinished, wasImmediate, hasMultipleResults);
            } else if (isFinished) {
              onFailureInternal(id, dataSource, new NullPointerException(), /* isFinished */ true);
            }
          }

          @Override
          public void onFailureImpl(DataSource<T> dataSource) {
            onFailureInternal(id, dataSource, dataSource.getFailureCause(), /* isFinished */ true);
          }

          @Override
          public void onProgressUpdate(DataSource<T> dataSource) {
            boolean isFinished = dataSource.isFinished();
            float progress = dataSource.getProgress();
            onProgressUpdateInternal(id, dataSource, progress, isFinished);
          }
        };
```
在onNewResultImpl()的回调中，通过dataSource.getResult()在主线程得到了图片，然后通过触发onNewResultInternal()回调：
``` java
  private void onNewResultInternal(
      String id,
      DataSource<T> dataSource,
      @Nullable T image,
      float progress,
      boolean isFinished,
      boolean wasImmediate,
      boolean deliverTempResult) {
    try {
      // ……
      mEventTracker.recordEvent(
          isFinished ? Event.ON_DATASOURCE_RESULT : Event.ON_DATASOURCE_RESULT_INT);
      // create drawable
      Drawable drawable;
      try {
        drawable = createDrawable(image);
      } catch (Exception exception) {
        // ……
        return;
      }
      T previousImage = mFetchedImage;
      Drawable previousDrawable = mDrawable;
      mFetchedImage = image;
      mDrawable = drawable;
      try {
        // set the new image
        if (isFinished) {
          logMessageAndImage("set_final_result @ onNewResult", image);
          mDataSource = null;
          mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
          reportSuccess(id, image, dataSource);
        } else if (deliverTempResult) {
          logMessageAndImage("set_temporary_result @ onNewResult", image);
          mSettableDraweeHierarchy.setImage(drawable, 1f, wasImmediate);
          reportSuccess(id, image, dataSource);
        } else {
          logMessageAndImage("set_intermediate_result @ onNewResult", image);
          mSettableDraweeHierarchy.setImage(drawable, progress, wasImmediate);
          reportIntermediateSet(id, image);
          // IMPORTANT: do not execute any instance-specific code after this point
        }
      } finally {
        if (previousDrawable != null && previousDrawable != drawable) {
          releaseDrawable(previousDrawable);
        }
        if (previousImage != null && previousImage != image) {
          logMessageAndImage("release_previous_result @ onNewResult", previousImage);
          releaseImage(previousImage);
        }
      }
    } finally {
        // ……
    }
  }
```
原函数较长，删掉注释和一些异常处理的代码之后，有如下要点：
- 将得到的image，转换成Drawable对象
- 调用mSettableDraweeHierarchy的setImage()方法把这个Drawable对象设置进去，这里的mSettableDraweeHierarchy是GenericDraweeHierarchy类型
``` java
  public void setImage(Drawable drawable, float progress, boolean immediate) {
    drawable = WrappingUtils.maybeApplyLeafRounding(drawable, mRoundingParams, mResources);
    drawable.mutate();
    mActualImageWrapper.setDrawable(drawable);
    mFadeDrawable.beginBatchMode();
    fadeOutBranches();
    fadeInLayer(ACTUAL_IMAGE_INDEX);
    setProgress(progress);
    if (immediate) {
      mFadeDrawable.finishTransitionImmediately();
    }
    mFadeDrawable.endBatchMode();
  }
```
这里关键的代码是调用mActualImageWrapper.setDrawable(drawable);把得到的图片对应的drawable设置到mActualImageWrapper这个成员变量上


**至此，我们完成了DraweeHolder.setController()的这一部分操作，接下来看下mDraweeHolder.getTopLevelDrawable()的实现：**

### 2.2 DraweeHolder.getTopLevelDrawable()
``` java
  public @Nullable Drawable getTopLevelDrawable() {
    return mHierarchy == null ? null : mHierarchy.getTopLevelDrawable();
  }
```
这里mHierarchy是SettableDraweeHierarchy接口类型，实现类是GenericDraweeHierarchy，看下它的getTopLevelDrawable()实现：
``` java
  public Drawable getTopLevelDrawable() {
    return mTopLevelDrawable;
  }
```
实际上就只是返回mTopLevelDrawable，而mTopLevelDrawable是在GenericDraweeHierarchy构造方法中创建的，看下构造方法的整个创建过程：
``` java
  GenericDraweeHierarchy(GenericDraweeHierarchyBuilder builder) {
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.beginSection("GenericDraweeHierarchy()");
    }
    mResources = builder.getResources();
    mRoundingParams = builder.getRoundingParams();

    mActualImageWrapper = new ForwardingDrawable(mEmptyActualImageDrawable);

    int numOverlays = (builder.getOverlays() != null) ? builder.getOverlays().size() : 1;

    // make sure there is at least one overlay to make setOverlayImage(Drawable)
    // method work.
    if (numOverlays == 0) {
      numOverlays = 1;
    }

    numOverlays += (builder.getPressedStateOverlay() != null) ? 1 : 0;

    // layer indices and count
    int numLayers = OVERLAY_IMAGES_INDEX + numOverlays;

    // array of layers
    Drawable[] layers = new Drawable[numLayers];
    layers[BACKGROUND_IMAGE_INDEX] = buildBranch(builder.getBackground(), null);
    layers[PLACEHOLDER_IMAGE_INDEX] =
        buildBranch(builder.getPlaceholderImage(), builder.getPlaceholderImageScaleType());
    layers[ACTUAL_IMAGE_INDEX] =
        buildActualImageBranch(
            mActualImageWrapper,
            builder.getActualImageScaleType(),
            builder.getActualImageFocusPoint(),
            builder.getActualImageColorFilter());
    layers[PROGRESS_BAR_IMAGE_INDEX] =
        buildBranch(builder.getProgressBarImage(), builder.getProgressBarImageScaleType());
    layers[RETRY_IMAGE_INDEX] =
        buildBranch(builder.getRetryImage(), builder.getRetryImageScaleType());
    layers[FAILURE_IMAGE_INDEX] =
        buildBranch(builder.getFailureImage(), builder.getFailureImageScaleType());
    if (numOverlays > 0) {
      int index = 0;
      if (builder.getOverlays() != null) {
        for (Drawable overlay : builder.getOverlays()) {
          layers[OVERLAY_IMAGES_INDEX + index++] = buildBranch(overlay, null);
        }
      } else {
        index = 1; // reserve space for one overlay
      }
      if (builder.getPressedStateOverlay() != null) {
        layers[OVERLAY_IMAGES_INDEX + index] = buildBranch(builder.getPressedStateOverlay(), null);
      }
    }

    // fade drawable composed of layers
    mFadeDrawable = new FadeDrawable(layers, false, ACTUAL_IMAGE_INDEX);
    mFadeDrawable.setTransitionDuration(builder.getFadeDuration());

    // rounded corners drawable (optional)
    Drawable maybeRoundedDrawable =
        WrappingUtils.maybeWrapWithRoundedOverlayColor(mFadeDrawable, mRoundingParams);

    // top-level drawable
    mTopLevelDrawable = new RootDrawable(maybeRoundedDrawable);
    mTopLevelDrawable.mutate();

    resetFade();
    if (FrescoSystrace.isTracing()) {
      FrescoSystrace.endSection();
    }
  }
```
这个构造方法看起来有点长，实际上分成以下几个部分比较好了解：
- 创建一个名为layers的Drawable数组，在对应下标构建对应图层的Drawable，其中ACTUAL_IMAGE_INDEX这个常量对应的位置，是上面看到过的mActualImageWrapper
- 其余图层分别是背景图、placeholder等等
- 通过layers数组，构建出FadeDrawable的对象mFadeDrawable
- 通过mFadeDrawable构建出maybeRoundedDrawable
- 通过maybeRoundedDrawable构建出了mTopLevelDrawable
这个mTopLevelDrawable就是我们最终设置到图片控件ImageView的Drawble，它包含了多个图层，当我们把网络图片的uri设置进去的时候，取到真正网络图片时，会更新真实图片的图层，来实现UI的更新

### 总结
- Fresco提供了直接可用的UI控件
- Fresco的UI控件展示的实际上是一个多图层的Drawable，通过更新Drawble的图层来实现图片加载









