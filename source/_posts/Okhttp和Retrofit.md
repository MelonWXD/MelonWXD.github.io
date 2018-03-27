---
title: Retrofit与OkHttp学习
date: 2018-02-22 14:15:50
tags: [网络编程]  
categories: Android框架
---
Android网络请求框架okHttp源码分析
<!-- more -->  

# OkHttp

偷贴一张[Piasy](https://xiaozhuanlan.com/u/1338552356)的图

![](https://blog.piasy.com/img/201607/okhttp_full_process.png)



## 基本用法

```java
        OkHttpClient client=new OkHttpClient.Builder().build();
        Request request = new Request.Builder().url("xxx.com").build();

        //同步
        Response syncRep = client.newCall(request).execute();
        //异步
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

## 同步请求

先从同步请求入手，`newCall`返回的是RealCall实例，RealCall的execute

```java
   @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```

主要还是调用了`getResponseWithInterceptorChain`，最后在通知dispatcher结束。其他的都是取消、错误异常处理

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```

为请求添加拦截器（下面会讲），先添加自定义的拦截器，然后添加一堆默认的桥接、缓存等拦截器，最后添加的是**CallServerInterceptor**，划重点，要考的

最后调用RealInterceptorChain实例的proceed

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {

    calls++;

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    return response;
  }
```

第一次传进来的index为0，后面每次都是index+1，就是依次按上面interceptors的添加顺序，把拦截器取出来，调用intercept的方法。在拦截器内部又调用proceed，一直到最后来调用到上面说的最后添加的**CallServerInterceptor**这个拦截器来进行真正的网络操作。

```java
@Override 
public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();
    httpCodec.writeRequestHeaders(request);

    Response.Builder responseBuilder = null;
  //对Request的请求头"Expect: 100-continue"做个处理
  if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
      if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
        httpCodec.flushRequest();
        responseBuilder = httpCodec.readResponseHeaders(true);
      }

      if (responseBuilder == null) {
        // Write the request body if the "Expect: 100-continue" expectation was met.
        Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
        BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
        request.body().writeTo(bufferedRequestBody);
        bufferedRequestBody.close();
      } else if (!connection.isMultiplexed()) {
        // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection from
        // being reused. Otherwise we're still obligated to transmit the request body to leave the
        // connection in a consistent state.
        streamAllocation.noNewStreams();
      }
    }

    httpCodec.finishRequest();

    if (responseBuilder == null) {
      responseBuilder = httpCodec.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    int code = response.code();
    if (forWebSocket && code == 101) {
      // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }


    return response;
  }
```

实际上使用的是HttpCodec类来进行socket操作，基于Okio封装了BufferSink和BufferSource。

## 拦截器

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    @Nullable Connection connection();
  }
}
```

Intercept只有一个内部类Chain，一个`intercept`方法，参数为Chain的实例。

通过这个`chain.request`和`chain.proceed`来分别对Request和Response做额外的处理。

下面是一个自定义的拦截器，输出请求/响应的日志

```java
class LoggingInterceptor implements Interceptor {
  @Override public Response intercept(Interceptor.Chain chain) throws IOException {
    Request request = chain.request();
    
    long t1 = System.nanoTime();
    logger.info(String.format("Sending request %s on %s%n%s",
        request.url(), chain.connection(), request.headers()));

    Response response = chain.proceed(request);

    long t2 = System.nanoTime();
    logger.info(String.format("Received response for %s in %.1fms%n%s",
        response.request().url(), (t2 - t1) / 1e6d, response.headers()));

    return response;
  }
}
```

## 异步请求

下面再来看看异步请求，重新贴一下代码

```java
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });
```

RealCall的`enqueue`

```java
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }

```

主要是使用AsyncCall包装一下回调请求，然后通过client.dispatcher()的`enquue`来执行

```java
synchronized void enqueue(AsyncCall call) {
  if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    runningAsyncCalls.add(call);
    executorService().execute(call);
  } else {
    readyAsyncCalls.add(call);
  }
}

```

如果没有超过最大请求数，就使用executorService()返回的线程池实例执行call，否则添加到等待的readyAsyncCalls中。

再来介绍一下AsyncCall，是RealCall的一个内部类，继承自NamedRunnable，NamedRunnable实现Runnable。NamedRunnable在重写`run`方法的时执行了一个抽象方法`execute`，由AsyncCall来实现。所以当线程执行异步请求的时候实际上走的就是AsyncCall的`execute`方法。

```java
    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
			...
      } finally {
        client.dispatcher().finished(this);
      }
    }
```

跟RealCall的execute很像啊，`getResponseWithInterceptorChain`来执行。基本可以说明异步的请求跟同步的区别就是加了个线程池来实现网络请求，并通过callback回调。



# Retrofit

再来分析一下Retrofit是如何对OkHttp进行封装的

## 基本用法

**这块暂时不看，看下一个**

```java
    public static RetrofitClient getInstance() {
        if (mInstance == null) {
            synchronized (RetrofitClient.class) {
                if (mInstance == null) {
                    mInstance = new RetrofitClient();
                }
            }
        }
        return mInstance;
    }
    private RetrofitClient() {
        mHttpClient = new OkHttpClient.Builder()
                .addInterceptor(new LoggingInterceptor())
                .connectTimeout(10, TimeUnit.SECONDS)
                .build();
        mRetrofit = new Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(CustomConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .client(mHttpClient)
                .build();
    
        mApi = mRetrofit.create(ApiService.class);
    }
```


**↓↓看这个↓↓**

```java
public interface BlogService {
    @GET("blog/{id}")
    Call<ResponseBody> getBlog(@Path("id") int id);
}


Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("http://localhost:4567/")
    .build();
BlogService service = retrofit.create(BlogService.class);
Call<ResponseBody> call = service.getBlog(2);
call.enqueue(new Callback<ResponseBody>() {
  @Override
  public void onResponse(Call<ResponseBody> call, Response<ResponseBody> response) {

  }

  @Override
  public void onFailure(Call<ResponseBody> call, Throwable t) {

  }
});
```
## create

```java
  public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(
          service.getClassLoader()
          , new Class<?>[] { service }
          , new InvocationHandler() {
              @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
                  throws Throwable {
    				...
              }
        });
  }
```

通过动态代理技术直接返回代理类实例

**动态代理**

通过jdk在内存中动态生成一个代理对象，涉及Proxy类的静态方法`newProxyInstance`和`InvocationHandler`接口。

下面大体讲一下使用，直接手打可能有点错误，意思到了就行了。。。

```java
//接口
interface UserDao{
  String getName();
  int getAge();
}
//委托类
class UserDaoImp{
  String getName(){
    return "lewis";
  }
  int getAge(){
    return 22;
  }
}
//动态代理类
public class MyInvocationHandler implements InvocationHandler {  
    private Object target;  
    MyInvocationHandler(Object target) {  
        super();  
        this.target = target;  //在这里获取到委托类的实例
    }  
     
    @Override  
    public Object invoke(Object o, Method method, Object[] args) throws Throwable {  
            System.out.println("当前invoke的方法为：" + method.getName() );  
            return method.invoke(target, args); 
    }  
}
//使用
public static void main(String[] args) {  
 	//实例化委托类传入代理类
  	UserDao userImp = new UserDaoImp();
  	UserDao userProxy = (UserDao)Proxy.newProxyInstance(
      			userImp.getClass().getClassLoader()
      			,userImp.getClass().getInterfaces()
      			,new MyInvocationHandler(userImp)
      			);       
   System.out.println(userProxy.getName()); 
}  
//输出
当前invoke的方法为：getName
lewis
```

再回到create中InvocationHandler的invoke方法的代码

```java
 @Override 
 public Object invoke(Object proxy, Method method, @Nullable Object[] args) throws Throwable {
    ServiceMethod<Object, Object> serviceMethod = (ServiceMethod<Object, Object>) loadServiceMethod(method);
    OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
    return serviceMethod.callAdapter.adapt(okHttpCall);      
}
        
```

## loadServiceMethod

```java
  ServiceMethod<?, ?> loadServiceMethod(Method method) {
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }
```

先从缓存中查找，如果没有就通过`new ServiceMethod.Builder<>(this, method).build()`新建一个

```JAVA
public ServiceMethod build() {
      callAdapter = createCallAdapter();

      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      return new ServiceMethod<>(this);
    }
```

主要逻辑如上

- `createCallAdapter`是根据委托类方法的返回值，来确定要使用的CallAdapter，比如我们的返回值是`Observable`，那么就会使用`RxJava2CallAdapterFactory`，如果我们在创建Retrofit实例的时候没有通过`addCallAdapterFactory(RxJava2CallAdapterFactory.create())`来添加与返回值相应的处理函数，那么这里就会抛出异常。
- `createResponseConverter`也是同样的道理，比如我们使用`addConverterFactory`添加`GsonConverterFactory`
- `parseMethodAnnotation`解析每个方法的注解GET、POST、PUT等等

## new OkHttpCall()

```java
final class OkHttpCall<T> implements Call<T> {
  private final ServiceMethod<T, ?> serviceMethod;
  private final @Nullable Object[] args;
  @GuardedBy("this")
  private @Nullable okhttp3.Call rawCall;
  
  OkHttpCall(ServiceMethod<T, ?> serviceMethod, @Nullable Object[] args) {
    this.serviceMethod = serviceMethod;
    this.args = args;
  }
  
  
    private okhttp3.Call createRawCall() throws IOException {
    Request request = serviceMethod.toRequest(args);
    okhttp3.Call call = serviceMethod.callFactory.newCall(request);
    if (call == null) {
      throw new NullPointerException("Call.Factory returned null.");
    }
    return call;
  }
}
```

对内部okhttp3.Call类型的`rawCall`进行封装，默认使用的是OkHttpClient的实例。

```java
serviceMethod.callFactory = = builder.retrofit.callFactory(); //serviceMethod.java
 okhttp3.Call.Factory callFactory = this.callFactory; // Retrofit.Builder
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

```

### parseResponse

OkHttpCall内部对返回值做了处理，通过responseConverter

```java
T body = serviceMethod.toResponse(catchingBody);
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }
```

## callAdapter.adapt

再关注动态代理方法`invoke`的返回值

```java
return serviceMethod.callAdapter.adapt(okHttpCall);
```

调用serviceMethod的成员变量callAdapter.adapt来适配委托类的方法返回值。

在loadServiceMethod中创建ServiceMethod时，`callAdapter = createCallAdapter()`，跟进

```java
   private CallAdapter<T, R> createCallAdapter() {
      try {
        //noinspection unchecked
        return (CallAdapter<T, R>) retrofit.callAdapter(returnType, annotations);
      } catch (RuntimeException e) { // Wide exception range because factories are user code.
        throw methodError(e, "Unable to create call adapter for %s", returnType);
      }
    }
```
```java
public CallAdapter<?, ?> callAdapter(Type returnType, Annotation[] annotations) {
    return nextCallAdapter(null, returnType, annotations);
}

final List<CallAdapter.Factory> adapterFactories;

public CallAdapter<?, ?> nextCallAdapter(@Nullable CallAdapter.Factory skipPast, Type returnType,
      Annotation[] annotations) {
 
    int start = adapterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = adapterFactories.size(); i < count; i++) {
      CallAdapter<?, ?> adapter = adapterFactories.get(i).get(returnType, annotations, this);
      if (adapter != null) {
        return adapter;
      }
    }
  }

```

通过`adapterFactories.get(i).get(returnType, annotations, this);`可以知道是遍历retrofit的adapterFactories，并调用每个item的`get`方法来获取与返回值对应的Adapter

### adapterFactories

这个adapterFactories是在构造Retrofit时由Builder默认添加了一个

```java
   public Retrofit build() {
     ...
      // Make a defensive copy of the adapters and add the default Call adapter.
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));
	...
  }
```

以默认的`DefaultCallAdapterFactory`为例，看看他的get方法

```java
final class DefaultCallAdapterFactory extends CallAdapter.Factory {
  static final CallAdapter.Factory INSTANCE = new DefaultCallAdapterFactory();

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }

    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return call;
      }
    };
  }
}
```

返回的`new CallAdapter`对传入的OkHttpCall毫无包装，直接返回Call，毕竟是默认的。

我们再以常用的`RxJava2CallAdapterFactory`为例，看RxJava2CallAdapterFactory的get方法。

```java
 @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    Class<?> rawType = getRawType(returnType);

    if (rawType == Completable.class) {
      // Completable is not parameterized (which is what the rest of this method deals with) so it
      // can only be created with a single configuration.
      return new RxJava2CallAdapter(Void.class, scheduler, isAsync, false, true, false, false,
          false, true);
    }

    return new RxJava2CallAdapter(responseType, scheduler, isAsync, isResult, isBody, isFlowable,
        isSingle, isMaybe, false);
  }
```

返回的是RxJava2CallAdapter的实例，在看看RxJava2CallAdapter的adapt方法。

```java
 @Override public Object adapt(Call<R> call) {
    Observable<Response<R>> responseObservable = isAsync
        ? new CallEnqueueObservable<>(call)
        : new CallExecuteObservable<>(call);

    Observable<?> observable;
    if (isResult) {
      observable = new ResultObservable<>(responseObservable);
    } else if (isBody) {
      observable = new BodyObservable<>(responseObservable);
    } else {
      observable = responseObservable;
    }
	...
    return observable;
  }
```

根据各种条件做了一些判断，对Call进行封装，具体见源码。



再上一张图来帮助理解create的这个三部曲。

![](https://blog.piasy.com/img/201606/retrofit_stay.png)
