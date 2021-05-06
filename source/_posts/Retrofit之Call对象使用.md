---
title: Retrofit之Call对象
date: 2021-05-06 15:13:42
tags: 网络框架
---

### Overview
在上一篇《Retrofit源码学习》中已经了解了Retrofit是如何将一个http请求通过java接口的形式提供Call对象给用户代码，接下来看下如何使用这个Call对象。

### Call的使用
在retrofit的项目里，作者提供了一些samples，其中SimpleService的用法如下：
``` java
    Retrofit retrofit =
        new Retrofit.Builder()
            .baseUrl(API_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build();

    // Create an instance of our GitHub API interface.
    GitHub github = retrofit.create(GitHub.class);

    // Create a call instance for looking up Retrofit contributors.
    Call<List<Contributor>> call = github.contributors("square", "retrofit");

    // Fetch and print a list of the contributors to the library.
    List<Contributor> contributors = call.execute().body();
    for (Contributor contributor : contributors) {
      System.out.println(contributor.login + " (" + contributor.contributions + ")");
    }
```
从这里可以看到，Call\<T>对象转变到T对象，是通过execute().body()这两个方法的连续调用，因此沿着这个调用链进行分析。

### 1.Call.execute()
Call是一个接口，从上一篇的分析中可以得知，最终得到的实现类是OkHttpCall
``` java
  @Override
  public Response<T> execute() throws IOException {
    okhttp3.Call call;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = getRawCall();
    }

    if (canceled) {
      call.cancel();
    }

    return parseResponse(call.execute());
  }
```
这里实际上是对okhttp3里面的Call对象进行了一层封装，分成三步：
- 通过getRawCall()方法得到okhttp3的Call对象
- 对okhttp3的call对象进行execute()方法
- 对execute的结果进行parseResponse()操作

### 1.1 getRawCall()
``` java
  private okhttp3.Call getRawCall() throws IOException {
    okhttp3.Call call = rawCall;
    if (call != null) return call;

    // Re-throw previous failures if this isn't the first attempt.
    if (creationFailure != null) {
      if (creationFailure instanceof IOException) {
        throw (IOException) creationFailure;
      } else if (creationFailure instanceof RuntimeException) {
        throw (RuntimeException) creationFailure;
      } else {
        throw (Error) creationFailure;
      }
    }

    // Create and remember either the success or the failure.
    try {
      return rawCall = createRawCall();
    } catch (RuntimeException | Error | IOException e) {
      throwIfFatal(e); // Do not assign a fatal error to creationFailure.
      creationFailure = e;
      throw e;
    }
  }
```
调用到createRawCall()来进行
``` java
  private okhttp3.Call createRawCall() throws IOException {
    okhttp3.Call call = callFactory.newCall(requestFactory.create(args));
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
```
Call对象的创建分成两步：
- requestFactory.create()创建出okhttp3.Request对象
- callFactory.newCall()生产出Call对象

### 1.1.1 RequestFactory.create()
``` java
 okhttp3.Request create(Object[] args) throws IOException {
    @SuppressWarnings("unchecked") // It is an error to invoke a method with the wrong arg types.
    ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

    int argumentCount = args.length;
    if (argumentCount != handlers.length) {
        // ……
    }

    RequestBuilder requestBuilder =
        new RequestBuilder(
            httpMethod,
            baseUrl,
            relativeUrl,
            headers,
            contentType,
            hasBody,
            isFormEncoded,
            isMultipart);

    if (isKotlinSuspendFunction) {
      // The Continuation is the last parameter and the handlers array contains null at that index.
      argumentCount--;
    }

    List<Object> argumentList = new ArrayList<>(argumentCount);
    for (int p = 0; p < argumentCount; p++) {
      argumentList.add(args[p]);
      handlers[p].apply(requestBuilder, args[p]);
    }

    return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
  }
```
创建了RequestBuilder，然后出现了上篇暂时略过的ParameterHandler对象，将RequestBuilder对象和参数传入对应的ParameterHandler的apply方法中，因此看一下这个方法的作用：
ParameterHandler是一个抽象类，不同的参数类型使用的是不一样的实现类，我们仍以最常用的Query参数为例，使用的是Query这个实现类：
``` java
    void apply(RequestBuilder builder, @Nullable T value) throws IOException {
      if (value == null) return; // Skip null values.

      String queryValue = valueConverter.convert(value);
      if (queryValue == null) return; // Skip converted but null values

      builder.addQueryParam(name, queryValue, encoded);
    }
  }
```
query参数的处理就是将这个参数添加到RequestBuilder的queryParam上，由此可知，ParameterHandler的作用就是为不同类型的参数进行不同的操作，构建到RequestBuilder上。

最后通过RequestBuilder的build()方法就得到了一个okhttp使用的Request

### 1.1.2 CallFactory.newCall(Request request)
CallFactory是一个接口，在Retrofit中默认使用的是OkHttpClient
``` java
  @Override public Call newCall(Request request) {
    return RealCall.newRealCall(this, request, false /* for web socket */);
  }
```
OkHttpClient的newCall实现很简单。

**得到OkHttp的Call对象之后进行execute()操作**
### 1.2 okhttp3.Call.execute()
``` java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    transmitter.timeoutEnter();
    transmitter.callStart();
    try {
      client.dispatcher().executed(this);
      return getResponseWithInterceptorChain();
    } finally {
      client.dispatcher().finished(this);
    }
  }
```
这里就走到了okhttp的源码部分了，前面也写过文章学习了okhttp，简单来说就是在这里完成了网络请求拿到了Response

### 1.3 OkHttpCall.parseResponse
``` java
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse =
        rawResponse
            .newBuilder()
            .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
            .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingResponseBody catchingBody = new ExceptionCatchingResponseBody(rawBody);
    try {
      T body = responseConverter.convert(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }
```
将okhttp返回的Response进行处理，主要是调用responseConverter.convert()方法将Response类型转换成用户使用的数据类型，这里的responseConverter，是在创建OkHttpCall的时候传入的，一直往前追溯的话，responseConverter的创建代码在HttpServiceMethod的createResponseConverter()函数中：
``` java
  private static <ResponseT> Converter<ResponseBody, ResponseT> createResponseConverter(
      Retrofit retrofit, Method method, Type responseType) {
    Annotation[] annotations = method.getAnnotations();
    try {
      return retrofit.responseBodyConverter(responseType, annotations);
    } catch (RuntimeException e) { // Wide exception range because factories are user code.
      throw methodError(method, e, "Unable to create converter for %s", responseType);
    }
  }
```
继续追溯到retrofit的responseBodyConverter()：
``` java
  public <T> Converter<ResponseBody, T> responseBodyConverter(Type type, Annotation[] annotations) {
    return nextResponseBodyConverter(null, type, annotations);
  }
```
``` java
  public <T> Converter<ResponseBody, T> nextResponseBodyConverter(
      @Nullable Converter.Factory skipPast, Type type, Annotation[] annotations) {
    Objects.requireNonNull(type, "type == null");
    Objects.requireNonNull(annotations, "annotations == null");

    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
      Converter<ResponseBody, ?> converter =
          converterFactories.get(i).responseBodyConverter(type, annotations, this);
      if (converter != null) {
        //noinspection unchecked
        return (Converter<ResponseBody, T>) converter;
      }
    }

    StringBuilder builder =
        new StringBuilder("Could not locate ResponseBody converter for ")
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
这里是个一段关键代码，在创建converter的时候，是通过传入返回类型Type和Annotations，遍历convertFactorys中的converter，找到符合条件的converter来作为这个请求的converter；而在此前Retrofit的builid()方法中我们记得使用的默认的converterFactory是BuiltInConverters()；关于BuiltInConverters()的内容暂时不展开说

**总之，通过OkHttpCall.parseResponse()的方法，已经构建除了body为我们目标数据类型的Response**

### 2.Response.body()
``` java
  public @Nullable T body() {
    return body;
  }
```
这个方法其实很简单了，就是将保存在body字段的数据对象返回
**拿到数据对象之后我们就完成了从服务器获取所需数据的操作了，随后可以对对象进行本地的访问操作**

### 总结
- 通过触发Call对象的execute()方法，会通过okhttp进行网络请求，并且通过retrofit的converter将网络数据转换成用户需要的数据对象
- 通过Response的body()函数即可获取到前面说的数据对象