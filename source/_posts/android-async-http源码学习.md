---
title: android-async-http源码学习
date: 2021-04-26 16:27:35
tags: 网络框架
---
### Overview
android-async-http是loopj开源的一款网络请求库，可以说是最早出现的Android网络请求库，虽然目前作者已经不再维护，并且Android也放弃了其底层支持HttpClient，但作为Android平台网络请求库的鼻祖，其设计思想还是值得学习。

### 使用（来自官方文档）
``` java
import com.loopj.android.http.*;

public class TwitterRestClient {
  private static final String BASE_URL = "https://api.twitter.com/1/";

  private static AsyncHttpClient client = new AsyncHttpClient();

  public static void get(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {
      client.get(getAbsoluteUrl(url), params, responseHandler);
  }

  public static void post(String url, RequestParams params, AsyncHttpResponseHandler responseHandler) {
      client.post(getAbsoluteUrl(url), params, responseHandler);
  }

  private static String getAbsoluteUrl(String relativeUrl) {
      return BASE_URL + relativeUrl;
  }
}
```
从官方的使用介绍来看，只要调用AsyncHttpClient的get()方法传入url、参数和handler即可完成一次http的get请求，因此源码学习从AsyncHttpClient入手

### 1.构造函数
``` java
    public AsyncHttpClient() {
        this(false, 80, 443);
    }
    
    public AsyncHttpClient(int httpPort) {
        this(false, httpPort, 443);
    }
    
    public AsyncHttpClient(int httpPort, int httpsPort) {
        this(false, httpPort, httpsPort);
    }
    
    public AsyncHttpClient(boolean fixNoHttpResponseException, int httpPort, int httpsPort) {
        this(getDefaultSchemeRegistry(fixNoHttpResponseException, httpPort, httpsPort));
    }
```
0~3个参数的构造函数实际上就是对应不同的默认值，如果都没有设置，默认情况下http请求使用80端口，https请求使用443端口；最终都会调用到public AsyncHttpClient(SchemeRegistry schemeRegistry)这个构造函数，首先看下默认传入的schemeRegistry是什么东西：
``` java
    private static SchemeRegistry getDefaultSchemeRegistry(boolean fixNoHttpResponseException, int httpPort, int httpsPort) {
        // ……
        // Fix to SSL flaw in API < ICS
        // See https://code.google.com/p/android/issues/detail?id=13117
        SSLSocketFactory sslSocketFactory;
        if (fixNoHttpResponseException) {
            sslSocketFactory = MySSLSocketFactory.getFixedSocketFactory();
        } else {
            sslSocketFactory = SSLSocketFactory.getSocketFactory();
        }

        SchemeRegistry schemeRegistry = new SchemeRegistry();
        schemeRegistry.register(new Scheme("http", PlainSocketFactory.getSocketFactory(), httpPort));
        schemeRegistry.register(new Scheme("https", sslSocketFactory, httpsPort));

        return schemeRegistry;
    }
```
创建出SchemeRegistry对象，并且把http和https对应的SocketFactory注册给SchemeRegistry对象，而SchemeRegistry对象的构造函数也很简单：
``` java
    public SchemeRegistry() {
        super();
        registeredSchemes = new ConcurrentHashMap<String,Scheme>();
    }
```
言归正传，继续回到AsyncHttpClient真正的构造函数：
``` java
    public AsyncHttpClient(SchemeRegistry schemeRegistry) {

        BasicHttpParams httpParams = new BasicHttpParams();

        ConnManagerParams.setTimeout(httpParams, connectTimeout);
        ConnManagerParams.setMaxConnectionsPerRoute(httpParams, new ConnPerRouteBean(maxConnections));
        ConnManagerParams.setMaxTotalConnections(httpParams, DEFAULT_MAX_CONNECTIONS);

        HttpConnectionParams.setSoTimeout(httpParams, responseTimeout);
        HttpConnectionParams.setConnectionTimeout(httpParams, connectTimeout);
        HttpConnectionParams.setTcpNoDelay(httpParams, true);
        HttpConnectionParams.setSocketBufferSize(httpParams, DEFAULT_SOCKET_BUFFER_SIZE);

        HttpProtocolParams.setVersion(httpParams, HttpVersion.HTTP_1_1);

        ClientConnectionManager cm = createConnectionManager(schemeRegistry, httpParams);
        Utils.asserts(cm != null, "Custom implementation of #createConnectionManager(SchemeRegistry, BasicHttpParams) returned null");

        threadPool = getDefaultThreadPool();
        requestMap = Collections.synchronizedMap(new WeakHashMap<Context, List<RequestHandle>>());
        clientHeaderMap = new HashMap<String, String>();

        httpContext = new SyncBasicHttpContext(new BasicHttpContext());
        httpClient = new DefaultHttpClient(cm, httpParams);
        httpClient.addRequestInterceptor(new HttpRequestInterceptor() {
            @Override
            public void process(HttpRequest request, HttpContext context) {
                
                // ……
            }
        });

        httpClient.addResponseInterceptor(new HttpResponseInterceptor() {
            @Override
            public void process(HttpResponse response, HttpContext context) {
                
                // ……
            }
        });

        httpClient.addRequestInterceptor(new HttpRequestInterceptor() {
            @Override
            public void process(final HttpRequest request, final HttpContext context) throws HttpException, IOException {
                // ……
            }
        }, 0);

        httpClient.setHttpRequestRetryHandler(new RetryHandler(DEFAULT_MAX_RETRIES, DEFAULT_RETRY_SLEEP_TIME_MILLIS));
    }
```
主要进行了以下几个操作：
- 调用ConnManagerParams、HttpConnectionParams、HttpProtocolParams的静态方法进行参数设置
- 创建出了线程池
- 创建线程同步的Map记录request
- 创建出了HttpClient对象
- 为httpClient对象添加了多个requestInterceptor
- 为httpClient对象设置了请求重试的handler

### 2.get请求
``` java
    public RequestHandle get(String url, RequestParams params, ResponseHandlerInterface responseHandler) {
        return get(null, url, params, responseHandler);
    }
```
``` java
    public RequestHandle get(Context context, String url, RequestParams params, ResponseHandlerInterface responseHandler) {
        return sendRequest(httpClient, httpContext, new HttpGet(getUrlWithQueryString(isUrlEncodingEnabled, url, params)), null, responseHandler, context);
    }
```
``` java
    public RequestHandle get(Context context, String url, RequestParams params, ResponseHandlerInterface responseHandler) {
        return sendRequest(httpClient, httpContext, new HttpGet(getUrlWithQueryString(isUrlEncodingEnabled, url, params)), null, responseHandler, context);
    }
```
最后是转到sendRequest()方法进行请求发送

### 3.sendRequest()
``` java
    protected RequestHandle sendRequest(DefaultHttpClient client, HttpContext httpContext, HttpUriRequest uriRequest, String contentType, ResponseHandlerInterface responseHandler, Context context) {
        // ……
        if (contentType != null) {
            if (uriRequest instanceof HttpEntityEnclosingRequestBase && ((HttpEntityEnclosingRequestBase) uriRequest).getEntity() != null && uriRequest.containsHeader(HEADER_CONTENT_TYPE)) {
                log.w(LOG_TAG, "Passed contentType will be ignored because HttpEntity sets content type");
            } else {
                uriRequest.setHeader(HEADER_CONTENT_TYPE, contentType);
            }
        }

        responseHandler.setRequestHeaders(uriRequest.getAllHeaders());
        responseHandler.setRequestURI(uriRequest.getURI());

        AsyncHttpRequest request = newAsyncHttpRequest(client, httpContext, uriRequest, contentType, responseHandler, context);
        threadPool.submit(request);
        RequestHandle requestHandle = new RequestHandle(request);

        if (context != null) {
            List<RequestHandle> requestList;
            // Add request to request map
            synchronized (requestMap) {
                requestList = requestMap.get(context);
                if (requestList == null) {
                    requestList = Collections.synchronizedList(new LinkedList<RequestHandle>());
                    requestMap.put(context, requestList);
                }
            }

            requestList.add(requestHandle);

            Iterator<RequestHandle> iterator = requestList.iterator();
            while (iterator.hasNext()) {
                if (iterator.next().shouldBeGarbageCollected()) {
                    iterator.remove();
                }
            }
        }

        return requestHandle;
    }
```
在这段函数中，最主要的工作是创建出了一个AsyncHttpRequest，然后submit到threadPool线程池中，由此可以推断这个AsyncHttpRequest是一个Runnable对象，因此转到AsyncHttpRequest的run()函数中。

### 3.1 AsyncHttpRequest.run()
``` java
    public void run() {
        if (isCancelled()) {
            return;
        }

        // Carry out pre-processing for this request only once.
        if (!isRequestPreProcessed) {
            isRequestPreProcessed = true;
            onPreProcessRequest(this);
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendStartMessage();

        if (isCancelled()) {
            return;
        }

        try {
            makeRequestWithRetries();
        } catch (IOException e) {
            if (!isCancelled()) {
                responseHandler.sendFailureMessage(0, null, null, e);
            } else {
                AsyncHttpClient.log.e("AsyncHttpRequest", "makeRequestWithRetries returned error", e);
            }
        }

        if (isCancelled()) {
            return;
        }

        responseHandler.sendFinishMessage();

        if (isCancelled()) {
            return;
        }

        // Carry out post-processing for this request.
        onPostProcessRequest(this);

        isFinished = true;
    }
```
这段函数中多次调用isCancelled()来检查请求有没有被取消，除去这些检查代码，就是对onPreProcessRequest()、onPostProcessRequest()的调用，以及对responseHandler在请求发送前和发送后调用sendStartMessage()和sendFinishMessage()来发送通知
而真正进行请求发送的操作是调用makeRequestWithRetries()函数

### 3.1.1 makeRequestWithRetries()
``` java
    private void makeRequestWithRetries() throws IOException {
        boolean retry = true;
        IOException cause = null;
        HttpRequestRetryHandler retryHandler = client.getHttpRequestRetryHandler();
        try {
            while (retry) {
                try {
                    makeRequest();
                    return;
                } catch (UnknownHostException e) {
                    // switching between WI-FI and mobile data networks can cause a retry which then results in an UnknownHostException
                    // while the WI-FI is initialising. The retry logic will be invoked here, if this is NOT the first retry
                    // (to assist in genuine cases of unknown host) which seems better than outright failure
                    cause = new IOException("UnknownHostException exception: " + e.getMessage(), e);
                    retry = (executionCount > 0) && retryHandler.retryRequest(e, ++executionCount, context);
                } catch (NullPointerException e) {
                    // there's a bug in HttpClient 4.0.x that on some occasions causes
                    // DefaultRequestExecutor to throw an NPE, see
                    // https://code.google.com/p/android/issues/detail?id=5255
                    cause = new IOException("NPE in HttpClient: " + e.getMessage());
                    retry = retryHandler.retryRequest(cause, ++executionCount, context);
                } catch (IOException e) {
                    if (isCancelled()) {
                        // Eating exception, as the request was cancelled
                        return;
                    }
                    cause = e;
                    retry = retryHandler.retryRequest(cause, ++executionCount, context);
                }
                if (retry) {
                    responseHandler.sendRetryMessage(executionCount);
                }
            }
        } catch (Exception e) {
            // catch anything else to ensure failure message is propagated
            AsyncHttpClient.log.e("AsyncHttpRequest", "Unhandled exception origin cause", e);
            cause = new IOException("Unhandled exception: " + e.getMessage(), cause);
        }

        // cleaned up to throw IOException
        throw (cause);
    }
```
这个函数里面其实是重试逻辑的实现：通过一个while循环搭配上try-catch代码块，如果请求没有异常，则跳出这个循环；有异常的话则在catch中进行重试的准备和通知操作，然后通过while的下一次循环完成重试
而真正发送请求的是makeRequest()函数

### 3.1.1.1 makeRequest()
``` java
    private void makeRequest() throws IOException {
        if (isCancelled()) {
            return;
        }

        // Fixes #115
        if (request.getURI().getScheme() == null) {
            // subclass of IOException so processed in the caller
            throw new MalformedURLException("No valid URI scheme was provided");
        }

        if (responseHandler instanceof RangeFileAsyncHttpResponseHandler) {
            ((RangeFileAsyncHttpResponseHandler) responseHandler).updateRequestHeaders(request);
        }

        HttpResponse response = client.execute(request, context);

        if (isCancelled()) {
            return;
        }

        // Carry out pre-processing for this response.
        responseHandler.onPreProcessResponse(responseHandler, response);

        if (isCancelled()) {
            return;
        }

        // The response is ready, handle it.
        responseHandler.sendResponseMessage(response);

        if (isCancelled()) {
            return;
        }

        // Carry out post-processing for this response.
        responseHandler.onPostProcessResponse(responseHandler, response);
    }
```
和前面提到过的AsyncHttpRequest.run()方法是一样的，除去多次的isCancelled()检查以及请求前后的responseHandler通知方法回调以外，真正进行网络请求的代码是：
``` java
        HttpResponse response = client.execute(request, context);
```
这里的client是HttpClient的实例，因此到这里就基本完成android-async-http的工作了，往后的真正网络请求发出是通过HttpClient进行的。

### 总结
严格来说android-async-http是一个网络请求管理库，而不是网络请求库，因为最终网络请求的工作是交由HttpClient进行，android-async-http做的工作主要有以下这些：
- 通过线程池进行并发请求的管理
- 通过responseHandler在请求发送的对应时机进行和主线程的消息传递，更加方便用户代码操作
- 通过while循环+try-catch实现了重试机制


