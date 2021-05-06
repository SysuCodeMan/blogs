---
title: Picasso图片加载库源码学习
date: 2021-05-06 20:09:27
tags: 图片加载框架
---
### Overview
>A powerful image downloading and caching library for Android. ——Picasso官方介绍

Picasso是Square公司出品的一款图片加载框架，通过Picasso可以很容易地进行网络图片的加载。

### 示例用法
``` java
Picasso.build().load("http://i.imgur.com/DvpvklR.png").into(imageView);
```
**从示例用法的调用链出发对Picasso的源码进行学习**

### 1.Picasso.build()
``` java
    public Picasso build() {
      Context context = this.context;

      okhttp3.Cache unsharedCache = null;
      if (callFactory == null) {
        File cacheDir = createDefaultCacheDir(context);
        long maxSize = calculateDiskCacheSize(cacheDir);
        unsharedCache = new okhttp3.Cache(cacheDir, maxSize);
        callFactory = new OkHttpClient.Builder()
            .cache(unsharedCache)
            .build();
      }
      if (cache == null) {
        cache = new PlatformLruCache(Utils.calculateMemoryCacheSize(context));
      }
      if (service == null) {
        service = new PicassoExecutorService(new PicassoThreadFactory());
      }

      Dispatcher dispatcher = new Dispatcher(context, service, HANDLER, cache);

      return new Picasso(context, dispatcher, callFactory, unsharedCache, cache, listener,
          requestTransformers, requestHandlers, eventListeners, defaultBitmapConfig,
          indicatorsEnabled, loggingEnabled);
    }
  }
```
这个方法有以下要点：
- 使用了okHttp3.Cache作为缓存unsharedCache
- 使用okHttp3作为callFactory
- 如果没有手动设置cache，会使用PlatformLruCache作为cache
- 如果没有手动设置service，使用PicassoExecutorService
- 创建Dispatcher对象
- 返回Picasso实例

### 2.Picasso.load(String path)
``` java
  @NonNull
  public RequestCreator load(@Nullable String path) {
    if (path == null) {
      return new RequestCreator(this, null, 0);
    }
    if (path.trim().length() == 0) {
      throw new IllegalArgumentException("Path must not be empty.");
    }
    return load(Uri.parse(path));
  }
```
调用的是同名函数load(Uri uri)
``` java
  public RequestCreator load(@Nullable Uri uri) {
    return new RequestCreator(this, uri, 0);
  }
```
返回的是创建出来的RequestCreator对象：
``` java
  RequestCreator(Picasso picasso, @Nullable Uri uri, int resourceId) {
    if (picasso.shutdown) {
      throw new IllegalStateException(
          "Picasso instance already shut down. Cannot submit new requests.");
    }
    this.picasso = picasso;
    this.data = new Request.Builder(uri, resourceId, picasso.defaultBitmapConfig);
  }
```

### 3.RequestCreator.into(ImageView target)
``` java
  public void into(@NonNull ImageView target) {
    into(target, null);
  }
```
``` java
public void into(@NonNull ImageView target, @Nullable Callback callback) {
    long started = System.nanoTime();
    checkMain();

    if (target == null) {
      throw new IllegalArgumentException("Target must not be null.");
    }

    if (!data.hasImage()) {
      picasso.cancelRequest(target);
      if (setPlaceholder) {
        setPlaceholder(target, getPlaceholderDrawable());
      }
      return;
    }

    if (deferred) {
      if (data.hasSize()) {
        throw new IllegalStateException("Fit cannot be used with resize.");
      }
      int width = target.getWidth();
      int height = target.getHeight();
      if (width == 0 || height == 0) {
        if (setPlaceholder) {
          setPlaceholder(target, getPlaceholderDrawable());
        }
        picasso.defer(target, new DeferredRequestCreator(this, target, callback));
        return;
      }
      data.resize(width, height);
    }

    Request request = createRequest(started);

    if (shouldReadFromMemoryCache(request.memoryPolicy)) {
      Bitmap bitmap = picasso.quickMemoryCacheCheck(request.key);
      if (bitmap != null) {
        picasso.cancelRequest(target);
        Result result = new Result.Bitmap(bitmap, MEMORY);
        setResult(target, picasso.context, result, noFade, picasso.indicatorsEnabled);
        if (picasso.loggingEnabled) {
          log(OWNER_MAIN, VERB_COMPLETED, request.plainId(), "from " + MEMORY);
        }
        if (callback != null) {
          callback.onSuccess();
        }
        return;
      }
    }

    if (setPlaceholder) {
      setPlaceholder(target, getPlaceholderDrawable());
    }

    Action action = new ImageViewAction(picasso, target, request, errorDrawable, errorResId, noFade,
        callback);
    picasso.enqueueAndSubmit(action);
  }
```
分成以下要点：
- 进行线程检查、空判断等异常情况的处理
- 判断是否需要延迟处理
- 构造Request
- 检查是否命中内存缓存，命中则取消请求直接返回
- 通过picasso/target/request创建出Action对象
- 将创建出的action通过picasso的enqueueAndSubmit()方法进行提交

### 3.1 Action的创建
``` kotlin
internal class ImageViewAction(
        picasso: Picasso,
        val target: ImageView,
        data: Request,
        val errorDrawable: Drawable?,
        @DrawableRes val errorResId: Int,
        val noFade: Boolean,
        var callback: Callback?
) : Action(picasso, data) {
    override fun complete(result: Result) {
        PicassoDrawable.setResult(target, picasso.context, result, noFade, picasso.indicatorsEnabled)
        callback?.onSuccess()
    }

    override fun error(e: Exception) {
        val placeholder = target.drawable
        if (placeholder is Animatable) {
            (placeholder as Animatable).stop()
        }
        if (errorResId != 0) {
            target.setImageResource(errorResId)
        } else if (errorDrawable != null) {
            target.setImageDrawable(errorDrawable)
        }
        callback?.onError(e)
    }

    override fun getTarget(): Any {
        return target
    }

    override fun cancel() {
        super.cancel()
        callback = null
    }
}
```
主要是有complete和error方法，两个方法的调用暂时略过。

### 3.2 Picasso.enqueueAndSubmit(Action action)
``` java
  void enqueueAndSubmit(Action action) {
    Object target = action.getTarget();
    if (targetToAction.get(target) != action) {
      // This will also check we are on the main thread.
      cancelExistingRequest(target);
      targetToAction.put(target, action);
    }
    submit(action);
  }
```
``` java
  void submit(Action action) {
    dispatcher.dispatchSubmit(action);
  }
```
实际上是分发到dispatcher.dispatchSubmit(action)方法来完成
``` java
  void dispatchSubmit(Action action) {
    handler.sendMessage(handler.obtainMessage(REQUEST_SUBMIT, action));
  }
```
通过handler切换到子线程处理，看下处理：
``` java
  private static class DispatcherHandler extends Handler {
    private final Dispatcher dispatcher;

    DispatcherHandler(Looper looper, Dispatcher dispatcher) {
      super(looper);
      this.dispatcher = dispatcher;
    }

    @Override public void handleMessage(final Message msg) {
      switch (msg.what) {
        case REQUEST_SUBMIT: {
          Action action = (Action) msg.obj;
          dispatcher.performSubmit(action);
          break;
        }
        // ……
      }
    }
  }
``` 
继续调用到dispatcher的performSubmit(Action action)方法
``` java
  void performSubmit(Action action) {
    performSubmit(action, true);
  }
```
``` java
  void performSubmit(Action action, boolean dismissFailed) {
    if (pausedTags.contains(action.getTag())) {
      pausedActions.put(action.getTarget(), action);
      // ……
      return;
    }

    BitmapHunter hunter = hunterMap.get(action.request.key);
    if (hunter != null) {
      hunter.attach(action);
      return;
    }

    // ……

    hunter = forRequest(action.picasso, this, cache, action);
    hunter.future = service.submit(hunter);
    hunterMap.put(action.request.key, hunter);
    if (dismissFailed) {
      failedActions.remove(action.getTarget());
    }
    // ……
  }
```
- 首先尝试从hunterMap中得到这个action.request.key对应的BitmapHunter，如果有则attach到这个action并返回
- 如果没有符合要求的BitmapHunter，则通过forRequest()方法创建出BitmapHunter
- 通过service.submit()来提交这个hunter，并把返回值设置到hunter的future字段上；这里的service是一个java的ExecutorService，由此可知BitmapHunter应该是一个Runnable

### 3.2.1 BitmapHunter.run()
``` kotlin
override fun run() {
        try {
            updateThreadName(data)

            if (picasso.loggingEnabled) {
                log(OWNER_HUNTER, VERB_EXECUTING, getLogIdsForHunter(this))
            }

            result = hunt()
            dispatcher.dispatchComplete(this)
        } catch (e: IOException) {
            exception = e
            if (retryCount > 0) {
                dispatcher.dispatchRetry(this)
            } else {
                dispatcher.dispatchFailed(this)
            }
        } catch (e: Exception) {
            exception = e
            dispatcher.dispatchFailed(this)
        } finally {
            Thread.currentThread().name = THREAD_IDLE_NAME
        }
    }
```
这个方法中实际上是通过hunt()方法来获取图片：
``` kotlin
fun hunt(): Bitmap? {
        // a.
        if (shouldReadFromMemoryCache(data.memoryPolicy)) {
            cache[key]?.let { bitmap ->
                picasso.cacheHit()
                if (picasso.loggingEnabled) {
                    log(OWNER_HUNTER, VERB_DECODED, data.logId(), "from cache")
                }

                return Bitmap(bitmap, LoadedFrom.MEMORY)
            }
        }
        // b.
        if (retryCount == 0) {
            data = data.newBuilder().networkPolicy(NetworkPolicy.OFFLINE).build()
        }
        // c.
        val resultReference = AtomicReference<RequestHandler.Result?>()
        val exceptionReference = AtomicReference<Throwable?>()

        val latch = CountDownLatch(1)
        try {
            requestHandler.load(picasso, data, object : RequestHandler.Callback {
                override fun onSuccess(result: RequestHandler.Result?) {
                    resultReference.set(result)
                    latch.countDown()
                }

                override fun onError(t: Throwable) {
                    exceptionReference.set(t)
                    latch.countDown()
                }
            })

            latch.await()
        } catch (ie: InterruptedException) {
            val interruptedIoException = InterruptedIOException()
            interruptedIoException.initCause(ie)
            throw interruptedIoException
        }
        // d.
        exceptionReference.get()?.let { throwable ->
            when (throwable) {
                is IOException, is Error, is RuntimeException -> throw throwable
                else -> throw RuntimeException(throwable)
            }
        }
        // e.
        val result = resultReference.get() as? Bitmap
                ?: throw AssertionError("Request handler neither returned a result nor an exception.")
        val bitmap = result.bitmap
        if (picasso.loggingEnabled) {
            log(OWNER_HUNTER, VERB_DECODED, data.logId())
        }
        picasso.bitmapDecoded(bitmap)

        val transformations = ArrayList<Transformation>(data.transformations.size + 1)
        if (data.needsMatrixTransform() || result.exifRotation != 0) {
            transformations += MatrixTransformation(data)
        }
        transformations += data.transformations

        val transformedResult =
                applyTransformations(picasso, data, transformations, result) ?: return null
        val transformedBitmap = transformedResult.bitmap
        picasso.bitmapTransformed(transformedBitmap)

        return transformedResult
    }
```
这个方法的几部分作用分别是：
- a.检查内存缓存有没有图片可以直接返回
- b.进行重试的计数操作
- c.通过CountDownLatch，实现当前线程的阻塞，因为实际上图片加载的工作交给了request.load()方法去做，在load()方法里通过latch.countDown()将CountDownLatch减到0，下面的latch.await()就是要等待前面的load()回调执行完成
- d.异常处理
- e.对前面load()方法得到的Bitmap进行decode和trandsformations操作并返回结果
从这个函数执行过程可知，真正加载图片的操作在load()函数中：

### 3.2.1.2 RequestHandler.load()
在BitmapHunter中使用的requestHandler，是在BitmapHunter构建时传入的，往前追溯的话，是在前面的forRequest()函数函数中：
``` java
fun forRequest(
                picasso: Picasso,
                dispatcher: Dispatcher,
                cache: PlatformLruCache,
                action: Action
        ): BitmapHunter {
            val request = action.request
            val requestHandlers = picasso.getRequestHandlers()

            // Index-based loop to avoid allocating an iterator.
            for (i in requestHandlers.indices) {
                val requestHandler = requestHandlers[i]
                if (requestHandler.canHandleRequest(request)) {
                    return BitmapHunter(picasso, dispatcher, cache, requestHandler, action)
                }
            }

            return BitmapHunter(picasso, dispatcher, cache, ERRORING_HANDLER, action)
        }
```
这里遍历picasso的所有RequestHandlers，如果有可以处理当前request的RequestHandler，则直接使用；在picasso中具有以下Handler：
``` java
    allRequestHandlers.add(ResourceDrawableRequestHandler.create(context));
    allRequestHandlers.add(new ResourceRequestHandler(context));
    allRequestHandlers.addAll(extraRequestHandlers);
    allRequestHandlers.add(new ContactsPhotoRequestHandler(context));
    allRequestHandlers.add(new MediaStoreRequestHandler(context));
    allRequestHandlers.add(new ContentStreamRequestHandler(context));
    allRequestHandlers.add(new AssetRequestHandler(context));
    allRequestHandlers.add(new FileRequestHandler(context));
    allRequestHandlers.add(new NetworkRequestHandler(callFactory));
    requestHandlers = Collections.unmodifiableList(allRequestHandlers);
```
通过名字可知，是用于处理不同类型的图片加载；对于我们比较常用的网络图片，自然使用的是NetworkRequestHandler，其load()函数如下：
``` java
@Override
    public void load(@NonNull final Picasso picasso, @NonNull final Request request,
            @NonNull final Callback callback) {
        okhttp3.Request callRequest = createRequest(request);
        callFactory.newCall(callRequest).enqueue(new okhttp3.Callback() {
            @Override
            public void onResponse(Call call, Response response) {
                if (!response.isSuccessful()) {
                    callback.onError(new ResponseException(response.code(), request.networkPolicy));
                    return;
                }

                ResponseBody body = response.body();
                if (loadedFrom == DISK && body.contentLength() == 0) {
                    body.close();
                    callback.onError(
                            new ContentLengthException(
                                    "Received response with 0 content-length header."));
                    return;
                }
                if (loadedFrom == NETWORK && body.contentLength() > 0) {
                    picasso.downloadFinished(body.contentLength());
                }
                try {
                    Bitmap bitmap = decodeStream(body.source(), request);
                    callback.onSuccess(new Result.Bitmap(bitmap, loadedFrom));
                } catch (IOException e) {
                    body.close();
                    callback.onError(e);
                }
            }

            @Override
            public void onFailure(Call call, IOException e) {
                callback.onError(e);
            }
        });
    }
```
有以下两步：
- 通过okhttp来触发网络请求
- 网络请求成功后通过decodeStream来将网络流decode成Bitmap

**至此，就看到Picasso是怎样完成网络图片的请求加载了，后面就是通过线程通信把得到的bitmap给到主线程，然后设置到imageView中**

### 总结
- Picasso内置了okhttp来完成网络图片的请求
- Picasso的缓存能力也是通过okhttp来完成的
- Picasso的功能是很好地封装了加载图片的代码方便用户代码使用


