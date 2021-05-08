---
title: Retrofit源码学习
date: 2021-04-30 15:05:28
tags: 网络框架
---

### Overview
Retrofit是Square公司开源的一款android网络请求库，官方介绍是Retrofit turns your HTTP API into a Java interface；说明主要作用是将HTTP API转换成Java接口，更方便使用。Retrofit这个词的中文意思是"改装"。

### 官方用法
通过注解定义请求路径和参数
``` java
public interface GitHubService {
  @GET("users/{user}/repos")
  Call<List<Repo>> listRepos(@Path("user") String user);
}
```

创建Retrofit实例并通过Retrofit创建service
``` java
Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("https://api.github.com/")
    .build();

GitHubService service = retrofit.create(GitHubService.class);
```

随后即可像调用本地接口一样来调用网络请求：
``` java
Call<List<Repo>> repos = service.listRepos("octocat");
```

从用法中可以看出，对Retrofit的源码学习应该从build()方法开始
### 1.Retrofit.Builder().build()
``` java
 public Retrofit build() {
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
      callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());

      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      converterFactories.add(new BuiltInConverters());
      converterFactories.addAll(this.converterFactories);
      converterFactories.addAll(platform.defaultConverterFactories());

      return new Retrofit(
          callFactory,
          baseUrl,
          unmodifiableList(converterFactories),
          unmodifiableList(callAdapterFactories),
          callbackExecutor,
          validateEagerly);
    }
  }
```
在调用Retrofit的构造函数之前，build()的方法做了一下准备：
- 如果用户代码没有传入callFactory，则使用OkHttpClient()作为callFactory，由此看出最终承担发送网络请求这个操作是通过okhttp来实现的
- 创建CallAdapter.Factory的List，并将用户代码传入的callAdapterFactories加到List中，并将默认使用的CallAdapterFactory都加入到List
- 创建Converter.Factory的List，并添加默认的ConverterFactory，同时也加入用户代码设置的ConverterFactory

然后调用Retrofit的构造函数：
``` java
  Retrofit(
      okhttp3.Call.Factory callFactory,
      HttpUrl baseUrl,
      List<Converter.Factory> converterFactories,
      List<CallAdapter.Factory> callAdapterFactories,
      @Nullable Executor callbackExecutor,
      boolean validateEagerly) {
    this.callFactory = callFactory;
    this.baseUrl = baseUrl;
    this.converterFactories = converterFactories; // Copy+unmodifiable at call site.
    this.callAdapterFactories = callAdapterFactories; // Copy+unmodifiable at call site.
    this.callbackExecutor = callbackExecutor;
    this.validateEagerly = validateEagerly;
  }
```
仅仅是对传入的字段进行设置

从使用文档上可以看到，创建了Retrofit实例之后就是调用其create()方法

### 2.Retrofit.create(final Class\<T> service)
``` java
  public <T> T create(final Class<T> service) {
    validateServiceInterface(service);
    return (T)
        Proxy.newProxyInstance(
            service.getClassLoader(),
            new Class<?>[] {service},
            new InvocationHandler() {
              private final Platform platform = Platform.get();
              private final Object[] emptyArgs = new Object[0];

              @Override
              public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
                // If the method is a method from Object then defer to normal invocation.
                if (method.getDeclaringClass() == Object.class) {
                  return method.invoke(this, args);
                }
                args = args != null ? args : emptyArgs;
                return platform.isDefaultMethod(method)
                    ? platform.invokeDefaultMethod(method, service, proxy, args)
                    : loadServiceMethod(method).invoke(args);
              }
            });
  }
```
这个函数是整个Retrofit的核心，将定义的网络请求接口转换可用的java接口，使用了Java的代理技术，关键在传入的InvocationHandler的invoke()方法中，完成了用户定义的接口的方法的实现，即调用service.listRepos("octocat")时，真正执行的方法就是这里。

最后的return语句中，会判断调用的方法是不是DefaultMethod，DefaultMethod是java8引入的概念，即接口中的非抽象方法；通常我们使用都不会在接口中定义非抽象方法， 因此直接看下loadServiceMethod(method)的实现

### 2.1 Retrofit.loadServiceMethod(Method method)
``` java
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
```
- 缓存中如果有，直接返回
- 通过synchronized关键字锁住cache，然后调用ServiceMethod.parseAnnotations()创建出目标方法
- 加入到缓存，然后返回

### 2.2 ServiceMethod.parseAnnotations()
``` java
  static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    // ……
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
```
主要有两步：
- 创建RequestFactory对象
- 调用HttpServiceMethod.parseAnnotations()创建ServiceMethod返回

下面看下这两步的实现

### 2.2.1 RequestFactory.parseAnnotations(retrofit, method)
``` java
  static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
    return new Builder(retrofit, method).build();
  }
```
其中Builder的构造方法如下：
``` java
    Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }
```
会拿到方法的注解和参数类型以及参数的注解，这里就是我们用户代码的注解传递到retrofit的过程
然后是build()方法：
``` java
    RequestFactory build() {
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      // ……

      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
        parameterHandlers[p] =
            parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
      }

      // ……

      return new RequestFactory(this);
    }
```
删减掉前后的校验代码之后，就剩下这些核心代码，主要是两部分：
- 通过parseMethodAnnotation()方法对每个此前得到的每个方法注解进行解析
- 通过parseParameter()方法对参数的注解进行解析
- 调用RequestFactory的构造函数
分别看下这三个部分的实现

### 2.2.1.1 parseMethodAnnotation()
``` java
    private void parseMethodAnnotation(Annotation annotation) {
      if (annotation instanceof DELETE) {
        parseHttpMethodAndPath("DELETE", ((DELETE) annotation).value(), false);
      } else if (annotation instanceof GET) {
        parseHttpMethodAndPath("GET", ((GET) annotation).value(), false);
      } else if (annotation instanceof HEAD) {
        parseHttpMethodAndPath("HEAD", ((HEAD) annotation).value(), false);
      } else if (annotation instanceof PATCH) {
        parseHttpMethodAndPath("PATCH", ((PATCH) annotation).value(), true);
      } else if (annotation instanceof POST) {
        parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
      } else if (annotation instanceof PUT) {
        parseHttpMethodAndPath("PUT", ((PUT) annotation).value(), true);
      } else if (annotation instanceof OPTIONS) {
        parseHttpMethodAndPath("OPTIONS", ((OPTIONS) annotation).value(), false);
      } else if (annotation instanceof HTTP) {
        HTTP http = (HTTP) annotation;
        parseHttpMethodAndPath(http.method(), http.path(), http.hasBody());
      } else if (annotation instanceof retrofit2.http.Headers) {
        String[] headersToParse = ((retrofit2.http.Headers) annotation).value();
        if (headersToParse.length == 0) {
          throw methodError(method, "@Headers annotation is empty.");
        }
        headers = parseHeaders(headersToParse);
      } else if (annotation instanceof Multipart) {
        if (isFormEncoded) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isMultipart = true;
      } else if (annotation instanceof FormUrlEncoded) {
        if (isMultipart) {
          throw methodError(method, "Only one encoding annotation is allowed.");
        }
        isFormEncoded = true;
      }
    }
```
这里可以看到会调用parseHttpMethodAndPath()对Http的具体方法进行处理:
``` java
 private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
      if (this.httpMethod != null) {
        // ……
      }
      this.httpMethod = httpMethod;
      this.hasBody = hasBody;

      if (value.isEmpty()) {
        return;
      }

      // Get the relative URL path and existing query string, if present.
      int question = value.indexOf('?');
      if (question != -1 && question < value.length() - 1) {
        // ……
      }

      this.relativeUrl = value;
      this.relativeUrlParamNames = parsePathParameters(value);
    }
```
除去一些异常处理，其实就是将httpMethod和relativeUrl等参数记录到RequestFactory中
上面这部分是对方法注解的处理

### 2.2.1.2 parseParameter(int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation)
``` java
 private @Nullable ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {
      ParameterHandler<?> result = null;
      if (annotations != null) {
        for (Annotation annotation : annotations) {
          ParameterHandler<?> annotationAction =
              parseParameterAnnotation(p, parameterType, annotations, annotation);
          // ……
          result = annotationAction;
        }
      }

      // ……
      return result;
    }
```
通过调用parseParameterAnnotation()方法来完成解析，这个方法的
``` java
 private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
      if (annotation instanceof Url) {
        // ……
      } else if (annotation instanceof Path) {
        // ……
      } else if (annotation instanceof Query) {
        validateResolvableType(p, type);
        Query query = (Query) annotation;
        String name = query.value();
        boolean encoded = query.encoded();

        Class<?> rawParameterType = Utils.getRawType(type);
        gotQuery = true;
        if (Iterable.class.isAssignableFrom(rawParameterType)) {
          if (!(type instanceof ParameterizedType)) {
            // ……
          }
          ParameterizedType parameterizedType = (ParameterizedType) type;
          Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
          Converter<?, String> converter = retrofit.stringConverter(iterableType, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded).iterable();
        } else if (rawParameterType.isArray()) {
          
        } else {
          Converter<?, String> converter = retrofit.stringConverter(type, annotations);
          return new ParameterHandler.Query<>(name, converter, encoded);
        }
        // ……
      } else if (annotation instanceof QueryName) {
        // ……

      } else if (annotation instanceof QueryMap) {
        // ……

      } else if (annotation instanceof Header) {
        // ……

      } else if (annotation instanceof HeaderMap) {
        // ……

      } else if (annotation instanceof FieldMap) {
        // ……

      } else if (annotation instanceof PartMap) {
        // ……
      } else if (annotation instanceof Tag) {
        // ……

        return new ParameterHandler.Tag<>(tagType);
      }

      return null; // Not a Retrofit annotation.
    }
```
原函数的实现非常长，因为要对参数的每一种可能的注解都进行解析，这里保留了比较常用的对@Query的解析代码，最终将会得到一个ParameterHandler.Query对象返回，这里的ParameterHandler.Query暂时略过，这里面记录了query的参数信息

对方法和参数的注解都解析完之后，会调用new RequestFactory(Builder builder)来进行RequestFactory的实际构建

### 2.2.1.3 RequestFactory(Builder builder)
``` java
  RequestFactory(Builder builder) {
    method = builder.method;
    baseUrl = builder.retrofit.baseUrl;
    httpMethod = builder.httpMethod;
    relativeUrl = builder.relativeUrl;
    headers = builder.headers;
    contentType = builder.contentType;
    hasBody = builder.hasBody;
    isFormEncoded = builder.isFormEncoded;
    isMultipart = builder.isMultipart;
    parameterHandlers = builder.parameterHandlers;
    isKotlinSuspendFunction = builder.isKotlinSuspendFunction;
  }
```
常见的builder设计模式，把参数都记录到RequestFactory

**前面走得有点深，收一下：上面介绍的是创建RequestFactory对象的创建过程，在这个创建过程了完成了对方法的注解解析和对参数的注解解析，随后将进行ServiceMethod的创建，将一个网络请求封装成java的方法**

### 2.2.2 HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(Retrofit retrofit, Method method, RequestFactory requestFactory)
``` java
  static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
      Retrofit retrofit, Method method, RequestFactory requestFactory) {
    boolean isKotlinSuspendFunction = requestFactory.isKotlinSuspendFunction;
    boolean continuationWantsResponse = false;
    boolean continuationBodyNullable = false;

    Annotation[] annotations = method.getAnnotations();
    Type adapterType;
    if (isKotlinSuspendFunction) {
      // ……
    } else {
      adapterType = method.getGenericReturnType();
    }

    CallAdapter<ResponseT, ReturnT> callAdapter =
        createCallAdapter(retrofit, method, adapterType, annotations);
    Type responseType = callAdapter.responseType();
    if (responseType == okhttp3.Response.class) {
      // ……
    }
    if (responseType == Response.class) {
      // ……
    }
    // TODO support Unit for Kotlin?
    if (requestFactory.httpMethod.equals("HEAD") && !Void.class.equals(responseType)) {
        // ……
    }

    Converter<ResponseBody, ResponseT> responseConverter =
        createResponseConverter(retrofit, method, responseType);

    okhttp3.Call.Factory callFactory = retrofit.callFactory;
    if (!isKotlinSuspendFunction) {
      return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    } else if (continuationWantsResponse) {
      // ……
    } else {
      // ……
    }
  }
```
略去一些异常处理的代码之后，可以看到最终返回的是一个CallAdapted，通过传入相应的requestFactory、callFactory、responseConverter、callAdapter进行创建：
``` java
    CallAdapted(
        RequestFactory requestFactory,
        okhttp3.Call.Factory callFactory,
        Converter<ResponseBody, ResponseT> responseConverter,
        CallAdapter<ResponseT, ReturnT> callAdapter) {
      super(requestFactory, callFactory, responseConverter);
      this.callAdapter = callAdapter;
    }
```
调用了超类的构造函数：
``` java
  HttpServiceMethod(
      RequestFactory requestFactory,
      okhttp3.Call.Factory callFactory,
      Converter<ResponseBody, ResponseT> responseConverter) {
    this.requestFactory = requestFactory;
    this.callFactory = callFactory;
    this.responseConverter = responseConverter;
  }
```
这里也就是把变量记录到HttpServiceMethod的类成员变量上

**到这里已经完成了Retrofit.loadServiceMethod(Method method)的操作，创建出了ServiceMethod，但从2的代码中可以看到，我们在调用java接口进行网络请求时，触发的是ServiceMethod的invoke(Object[] args)方法，因此接下来看下这个方法是如何实现的**

### 3.ServiceMethod.invoke(Object[] args)
从前述的分析已经可以看到，最终得到的实际上是ServiceMethod的派生类CallAdapted，但CallAdapted没有重写invoke()方法，调用的是其直接父类HttpServiceMethod的invoke()方法
``` java
  final @Nullable ReturnT invoke(Object[] args) {
    Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
    return adapt(call, args);
  }
```
创建出一个OkHttpCall，然后调用adapt(call, args)方法，在CallAdapted中重写了adapt()方法：
``` java
    @Override
    protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
      return callAdapter.adapt(call);
    }
```
而这个callAdapter，就是前面分析里提到过的，retrofit传入的callAdapter，实现在DefaultCallAdapterFactory.java这个类中：
``` java
new CallAdapter<Object, Call<?>>() {
      @Override
      public Type responseType() {
        return responseType;
      }

      @Override
      public Call<Object> adapt(Call<Object> call) {
        return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
      }
    };
```
在Android平台中得到的是ExecutorCallbackCall<>(executor, call)，继承自Call，这也就是业务代码最终能够得到的结果。

### 总结
- Retrofit的作用是将http请求封装成java接口，方便用户代码使用
- http请求的信息，包括方法和参数，是通过java注解的形式定义的，Retrofit通过动态代理机制，解析注解的内容，生成java接口的实现类供用户代码使用
- http请求最终通过Retrofit变成了一个Call\<T>对象，用户代码随后可以对这个Call\<T>进行操作
- 从源码解析上来看，得到Call对象时，并没有真正发出网络请求，网络请求的发出是对Call对象进行操作时才进行的

官方的使用文档里面示例仅仅给出了如何获得一个Call对象，但没有给出Call对象要怎么使用，从这个Call对象到用户所需的数据对象还需要进一步操作，这部分放到下篇再继续。



