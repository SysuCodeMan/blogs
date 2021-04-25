---
title: Volley源码学习
date: 2021-04-25 10:51:19
tags: 网络框架
---
### Overview
Volley是Google官方提供的网络请求库，支持Http请求

### 用法
官方demo
``` java
// Instantiate the RequestQueue.
    RequestQueue queue = Volley.newRequestQueue(this);
    String url ="http://www.google.com";

    // Request a string response from the provided URL.
    StringRequest stringRequest = new StringRequest(Request.Method.GET, url,
                new Response.Listener<String>() {
        @Override
        public void onResponse(String response) {
            // Display the first 500 characters of the response string.
            textView.setText("Response is: "+ response.substring(0,500));
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            textView.setText("That didn't work!");
        }
    });

    // Add the request to the RequestQueue.
    queue.add(stringRequest);
```

**从用法上看，最关键的类是RequestQueue，下面将从RequestQueue开始进行分析**

### 1.Volley.newRquestQueue()
``` java
    public static RequestQueue newRequestQueue(Context context) {
        return newRequestQueue(context, (BaseHttpStack) null);
    }
```
``` java
    public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
        BasicNetwork network;
        if (stack == null) {
            if (Build.VERSION.SDK_INT >= 9) {
                network = new BasicNetwork(new HurlStack());
            } else {
                // Prior to Gingerbread, HttpUrlConnection was unreliable.
                // See: http://android-developers.blogspot.com/2011/09/androids-http-clients.html
                // At some point in the future we'll move our minSdkVersion past Froyo and can
                // delete this fallback (along with all Apache HTTP code).
                String userAgent = "volley/0";
                try {
                    String packageName = context.getPackageName();
                    PackageInfo info =
                            context.getPackageManager().getPackageInfo(packageName, /* flags= */ 0);
                    userAgent = packageName + "/" + info.versionCode;
                } catch (NameNotFoundException e) {
                }

                network =
                        new BasicNetwork(
                                new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
            }
        } else {
            network = new BasicNetwork(stack);
        }

        return newRequestQueue(context, network);
    }
```
``` java
    private static RequestQueue newRequestQueue(Context context, Network network) {
        final Context appContext = context.getApplicationContext();
        // Use a lazy supplier for the cache directory so that newRequestQueue() can be called on
        // main thread without causing strict mode violation.
        DiskBasedCache.FileSupplier cacheSupplier =
                new DiskBasedCache.FileSupplier() {
                    private File cacheDir = null;

                    @Override
                    public File get() {
                        if (cacheDir == null) {
                            cacheDir = new File(appContext.getCacheDir(), DEFAULT_CACHE_DIR);
                        }
                        return cacheDir;
                    }
                };
        RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheSupplier), network);
        queue.start();
        return queue;
    }
```
过程中涉及到以下几个关键类的创建：
- BasicNetwork
- DiskBasedCache
下面了解下这几个类的作用和初始化过程

### 1.1 BasicNetwork
``` java
    public BasicNetwork(BaseHttpStack httpStack) {
        // If a pool isn't passed in, then build a small default pool that will give us a lot of
        // benefit and not use too much memory.
        this(httpStack, new ByteArrayPool(DEFAULT_POOL_SIZE));
    }
```
``` java
    public BasicNetwork(BaseHttpStack httpStack, ByteArrayPool pool) {
        mBaseHttpStack = httpStack;
        mHttpStack = httpStack;
        mPool = pool;
    }
```
BasicNetwork的构造过程仅仅是将传入的BaseHttpStack和ByteArrayPool进行设置，而前面在构建BasicNetwork时传入的BaseHttpStack的实现类是HurlStack，下面看下HurlStack的具体实现。

### 1.1.1 HurlStack
``` java
    public HurlStack() {
        this(/* urlRewriter = */ null);
    }
```
``` java
    public HurlStack(UrlRewriter urlRewriter) {
        this(urlRewriter, /* sslSocketFactory = */ null);
    }
```
``` java
    public HurlStack(UrlRewriter urlRewriter, SSLSocketFactory sslSocketFactory) {
        mUrlRewriter = urlRewriter;
        mSslSocketFactory = sslSocketFactory;
    }
```
Hurl的构造过程也仅仅是对成员变量进行设置，而且目前用户代码传入的都是null。

### 1.2 DiskBasedCache
``` java
    public DiskBasedCache(FileSupplier rootDirectorySupplier) {
        this(rootDirectorySupplier, DEFAULT_DISK_USAGE_BYTES);
    }
```
``` java
    public DiskBasedCache(FileSupplier rootDirectorySupplier, int maxCacheSizeInBytes) {
        mRootDirectorySupplier = rootDirectorySupplier;
        mMaxCacheSizeInBytes = maxCacheSizeInBytes;
    }
```
DiskBasedCache的构造函数也只是对成员变量进行设置，这里传入的FileSuppiler会提供一个File，看类名也可以猜测到这个File的作用是作为磁盘缓存的文件。

### 1.3 RequestQueue.start()
``` java
    public void start() {
        stop(); // Make sure any currently running dispatchers are stopped.
        // Create the cache dispatcher and start it.
        mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
        mCacheDispatcher.start();

        // Create network dispatchers (and corresponding threads) up to the pool size.
        for (int i = 0; i < mDispatchers.length; i++) {
            NetworkDispatcher networkDispatcher =
                    new NetworkDispatcher(mNetworkQueue, mNetwork, mCache, mDelivery);
            mDispatchers[i] = networkDispatcher;
            networkDispatcher.start();
        }
    }
```
涉及到CacheDispatcher和NetworkDispatcher的start()
``` java
public class CacheDispatcher extends Thread
```
``` java
public class NetworkDispatcher extends Thread
```
这两个Dispatcher都是Thread的子类，从这里可以看到Volley开启了多线程。


从官方用法中看出，构造RequestQueue之后，是通过add一个StringRequest进行网络请求，接下来看下StringRequest的相关代码。
### 2.StringRequest
``` java
public class StringRequest extends Request<String>
```
继承自Request类

``` java
public StringRequest(
            int method,
            String url,
            Listener<String> listener,
            @Nullable ErrorListener errorListener) {
        super(method, url, errorListener);
        mListener = listener;
    }
```
构造函数需要传入两个listener，分别是正常返回的listener和异常返回的listener

### 2.1 Listener
``` java
    public interface Listener<T> {
        void onResponse(T response);
    }
```

### 2.2 ErrorListener
``` java
    public interface ErrorListener {
        void onErrorResponse(VolleyError error);
    }
```

### 2.3 Request
``` java
    public Request(int method, String url, @Nullable Response.ErrorListener errorListener) {
        mMethod = method;
        mUrl = url;
        mErrorListener = errorListener;
        setRetryPolicy(new DefaultRetryPolicy());

        mDefaultTrafficStatsTag = findDefaultTrafficStatsTag(url);
    }
```
这里可以看到在StringRequest的父类Request的构造函数中，记录了这个请求对应的方法和url，并且进行了重试策略的设置

**在StringRequest对象构造出来后，通过RequestQueue的add(Request request)方法发出，看下这个方法的执行链路**

### 3.RequestQueue.add(Request request)
``` java
    public <T> Request<T> add(Request<T> request) {
        // Tag the request as belonging to this queue and add it to the set of current requests.
        request.setRequestQueue(this);
        synchronized (mCurrentRequests) {
            mCurrentRequests.add(request);
        }

        // Process requests in the order they are added.
        request.setSequence(getSequenceNumber());
        request.addMarker("add-to-queue");
        sendRequestEvent(request, RequestEvent.REQUEST_QUEUED);

        beginRequest(request);
        return request;
    }
```
- 标记这个request的queue为当前queue
- 将这个request加入到当前记录的mCurrentRequests对象中
- 调用sendRequestEvent()方法
- 调用beginRequest方法

### 3.1 RequestQueue.sendRequestEvent()
``` java
    void sendRequestEvent(Request<?> request, @RequestEvent int event) {
        synchronized (mEventListeners) {
            for (RequestEventListener listener : mEventListeners) {
                listener.onRequestEvent(request, event);
            }
        }
    }
```
这个方法作用是通知当前的RequestEventListener，有request的事件产生，前面用户代码传入的是RequestEvent.REQUEST_QUEUED这个事件，代表有request加入队列

### 3.2 RequestQueue.beginRequest(Request request)
``` java
<T> void beginRequest(Request<T> request) {
        // If the request is uncacheable, skip the cache queue and go straight to the network.
        if (!request.shouldCache()) {
            sendRequestOverNetwork(request);
        } else {
            mCacheQueue.add(request);
        }
    }
```
会调用到sendRequestOverNetwork(Request request)，看这个函数名，我们很接近真正发送请求的地方了！

### 3.2.1 sendRequestOverNetwork(Request request)
``` java
    <T> void sendRequestOverNetwork(Request<T> request) {
        mNetworkQueue.add(request);
    }
```
看到这里有点失望，原来只是加入了网络队列中，那继续看下这个队列被消费时的场景

### 3.3 NetworkDispatcher.processRequest()
找了一下，这个队列就是前面我们构造NetworkDispatcher时传入的队列，而这个队列被消费的场景就是在processRequest()方法中
``` java
    private void processRequest() throws InterruptedException {
        // Take a request from the queue.
        Request<?> request = mQueue.take();
        processRequest(request);
    }
```
从队列中取出一个request然后调用到processRequest(Request request)
``` java
    void processRequest(Request<?> request) {
            // ……

            // Perform the network request.
            NetworkResponse networkResponse = mNetwork.performRequest(request);
            request.addMarker("network-http-complete");

                       if (networkResponse.notModified && request.hasHadResponseDelivered()) {
                request.finish("not-modified");
                request.notifyListenerResponseNotUsable();
                return;
            }

            // Parse the response here on the worker thread.
            Response<?> response = request.parseNetworkResponse(networkResponse);
            request.addMarker("network-parse-complete");

            // Write to cache if applicable.
            // TODO: Only update cache metadata instead of entire record for 304s.
            if (request.shouldCache() && response.cacheEntry != null) {
                mCache.put(request.getCacheKey(), response.cacheEntry);
                request.addMarker("network-cache-written");
            }

            // Post the response back.
            request.markDelivered();
            mDelivery.postResponse(request, response);
            request.notifyListenerResponseReceived(response);
        } catch (VolleyError volleyError) {
            volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
            parseAndDeliverNetworkError(request, volleyError);
            request.notifyListenerResponseNotUsable();
        } catch (Exception e) {
            // ……
        } finally {
            // ……
    }
```
原代码有点长，删减后的关键代码如上，主要有以下几个操作
- 通过Network.performRequest()方法获取到NetworkResponse
- 对原始的NetworkResponse进行parse操作得到Response
- 通过ResponseDelivery将Response post到对应的地方

下面逐个看下这些操作

### 3.3.1 Network.performRequest()
``` java
   public NetworkResponse performRequest(Request<?> request) throws VolleyError {
        long requestStart = SystemClock.elapsedRealtime();
        while (true) {
            HttpResponse httpResponse = null;
            byte[] responseContents = null;
            List<Header> responseHeaders = Collections.emptyList();
            try {
                // Gather headers.
                Map<String, String> additionalRequestHeaders =
                        HttpHeaderParser.getCacheHeaders(request.getCacheEntry());
                httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders);
                int statusCode = httpResponse.getStatusCode();

                responseHeaders = httpResponse.getHeaders();
                // Handle cache validation.
                if (statusCode == HttpURLConnection.HTTP_NOT_MODIFIED) {
                    long requestDuration = SystemClock.elapsedRealtime() - requestStart;
                    return NetworkUtility.getNotModifiedNetworkResponse(
                            request, requestDuration, responseHeaders);
                }

                // Some responses such as 204s do not have content.  We must check.
                InputStream inputStream = httpResponse.getContent();
                if (inputStream != null) {
                    responseContents =
                            NetworkUtility.inputStreamToBytes(
                                    inputStream, httpResponse.getContentLength(), mPool);
                } else {
                    // Add 0 byte response as a way of honestly representing a
                    // no-content request.
                    responseContents = new byte[0];
                }

                // if the request is slow, log it.
                long requestLifetime = SystemClock.elapsedRealtime() - requestStart;
                NetworkUtility.logSlowRequests(
                        requestLifetime, request, responseContents, statusCode);

                if (statusCode < 200 || statusCode > 299) {
                    throw new IOException();
                }
                return new NetworkResponse(
                        statusCode,
                        responseContents,
                        /* notModified= */ false,
                        SystemClock.elapsedRealtime() - requestStart,
                        responseHeaders);
            } catch (IOException e) {
                // This will either throw an exception, breaking us from the loop, or will loop
                // again and retry the request.
                RetryInfo retryInfo =
                        NetworkUtility.shouldRetryException(
                                request, e, requestStart, httpResponse, responseContents);
                // We should already be on a background thread, so we can invoke the retry inline.
                NetworkUtility.attemptRetryOnException(request, retryInfo);
            }
        }
    }
```
函数约50行，但关键代码主要是第9行，后面都是对拿到的HttpResponse进行操作和检查，下面进到第9行的httpResponse = mBaseHttpStack.executeRequest(request, additionalRequestHeaders);


### 3.3.1.1 HurlStack.executeRequest()
``` java
    public HttpResponse executeRequest(Request<?> request, Map<String, String> additionalHeaders)
            throws IOException, AuthFailureError {
        String url = request.getUrl();
        HashMap<String, String> map = new HashMap<>();
        map.putAll(additionalHeaders);
        // Request.getHeaders() takes precedence over the given additional (cache) headers).
        map.putAll(request.getHeaders());
        if (mUrlRewriter != null) {
            String rewritten = mUrlRewriter.rewriteUrl(url);
            if (rewritten == null) {
                throw new IOException("URL blocked by rewriter: " + url);
            }
            url = rewritten;
        }
        URL parsedUrl = new URL(url);
        HttpURLConnection connection = openConnection(parsedUrl, request);
        boolean keepConnectionOpen = false;
        try {
            for (String headerName : map.keySet()) {
                connection.setRequestProperty(headerName, map.get(headerName));
            }
            setConnectionParametersForRequest(connection, request);
            // Initialize HttpResponse with data from the HttpURLConnection.
            int responseCode = connection.getResponseCode();
            if (responseCode == -1) {
                // -1 is returned by getResponseCode() if the response code could not be retrieved.
                // Signal to the caller that something was wrong with the connection.
                throw new IOException("Could not retrieve response code from HttpUrlConnection.");
            }

            if (!hasResponseBody(request.getMethod(), responseCode)) {
                return new HttpResponse(responseCode, convertHeaders(connection.getHeaderFields()));
            }

            // Need to keep the connection open until the stream is consumed by the caller. Wrap the
            // stream such that close() will disconnect the connection.
            keepConnectionOpen = true;
            return new HttpResponse(
                    responseCode,
                    convertHeaders(connection.getHeaderFields()),
                    connection.getContentLength(),
                    createInputStream(request, connection));
        } finally {
            if (!keepConnectionOpen) {
                connection.disconnect();
            }
        }
    }
```
又是大约50行的实现，这里已经看到了Volley是如何进行一个请求的，答案就是**通过Android库中自带的HttpURLConnection来进行实际的请求操作；最后也是通过HttpURLConnection的内容进行HttpResponse的构建**。

回到3.3中说的，拿到原始的HttpResponse之后要进行一次parse操作得到最终的Response
### 3.3.2 StringRequest.parseNetworkResponse(NetworkResponse response)
``` java
    protected Response<String> parseNetworkResponse(NetworkResponse response) {
        String parsed;
        try {
            parsed = new String(response.data, HttpHeaderParser.parseCharset(response.headers));
        } catch (UnsupportedEncodingException e) {
            // Since minSdkVersion = 8, we can't call
            // new String(response.data, Charset.defaultCharset())
            // So suppress the warning instead.
            parsed = new String(response.data);
        }
        return Response.success(parsed, HttpHeaderParser.parseCacheHeaders(response));
    }
```
通过原始的Response的data字段，和headers中指定的charset来构建出String对象，最后通过String对象构建出Response

parse出最终的Response之后，要将这个response post到需要的地方
### 3.3.3 ExecutorDelivery
``` java
    public void postResponse(Request<?> request, Response<?> response) {
        postResponse(request, response, null);
    }
```
``` java
    public void postResponse(Request<?> request, Response<?> response, Runnable runnable) {
        request.markDelivered();
        request.addMarker("post-response");
        mResponsePoster.execute(new ResponseDeliveryRunnable(request, response, runnable));
    }
```
这里的mResponsePoster只是一个Executor，因此excute方法执行的实际上是ResponseDeliveryRunnable的run()方法：
``` java
       public void run() {
            if (mRequest.isCanceled()) {
                mRequest.finish("canceled-at-delivery");
                return;
            }

            // Deliver a normal response or error, depending.
            if (mResponse.isSuccess()) {
                mRequest.deliverResponse(mResponse.result);
            } else {
                mRequest.deliverError(mResponse.error);
            }
            // ……
        }
    }
```
最终实际上是通过request自己的mRequest.deliverResponse()方法执行
``` java
    protected void deliverResponse(String response) {
        Response.Listener<String> listener;
        synchronized (mLock) {
            listener = mListener;
        }
        if (listener != null) {
            listener.onResponse(response);
        }
    }
```
在StringRequest中这个deliverResponse()方法就触发了最早用户代码传入的listner的onResponse方法啦！

### 总结
- Volley实际的网络请求是通过HttpURLConnection来进行的
- Volley最大的作用是帮助用户代码更好地调度网络请求













