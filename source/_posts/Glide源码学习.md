---
title: Glide源码学习
date: 2018-03-28 19:30:37
tags: [Glide]  
categories: Android
---

Glide源码学习

<!-- more -->  

# 基本使用

```java
        Glide.with(this).load("https://www.baidu.com/123.jpg").into(imageView);
```

## with

```java
    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }
    public static RequestManagerRetriever get() {
        return INSTANCE;
    }
    private static final RequestManagerRetriever INSTANCE = new RequestManagerRetriever();
```

```java
    public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
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
```

根据传入的context不同而返回不同的RequestManager，看似重载了很多get方法，但是实际上只区分为applicationCtx和非applicationCtx(Fragment)。下面基于applicationCtx来跟进`getApplicationManager(context)`。

```java
    private RequestManager getApplicationManager(Context context) {
  		applicationManager = new RequestManager(context.getApplicationContext(),
                                          new ApplicationLifecycle(), 
                                          new EmptyRequestManagerTreeNode());
        return applicationManager;
    }
```

即with返回了一个RequestManager对象，RequestManager具体是干什么的后面再说。

## load

同样的load也有很多重载方法，我们以String为参数的方法来跟进，懂了一个，其他也一通百通，源码阅读过程切忌沉迷细节。

```java
    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }
    public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }
    private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
      
        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
    public <A, X extends GenericRequestBuilder<A, ?, ?, ?>> X apply(X builder) {
            if (options != null) { //默认为null 
                options.apply(builder);
            }
            return builder;
    }
```

`fromString()`返回了DrawableTypeRequest的实例，DrawableTypeRequest继承了DrawableRequestBuilder，load方法也没有重写，直接调用的DrawableRequestBuilder的，而DrawableRequestBuilder又继承了GenericRequestBuilder。

```java
    @Override
    public DrawableRequestBuilder<ModelType> load(ModelType model) {
        super.load(model);
        return this;
    }
//GenericRequestBuilder.java
    public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> load(ModelType model) {
        this.model = model;
        isModelSet = true;
        return this;
    }
```

最后将返回值向下转型为子类 DrawableTypeRequest<String>。说白了这整个load流程就是根据传入参数的类型来设置加载数据的模式。

## into

还是走的父类方法

```java
//DrawableRequestBuilder.java    
	@Override
    public Target<GlideDrawable> into(ImageView view) {
        return super.into(view);
    }

//GenericRequestBuilder.java
    public Target<TranscodeType> into(ImageView view) {
        Util.assertMainThread();
      
        //略掉一些对imageView的ScaleType的处理
      
		//transcodeClass查看构造函数可以知道是 GlideDrawable.class
        return into(glide.buildImageViewTarget(view, transcodeClass));
    }
```
### GlideDrawableImageViewTarget

跟进看看这个buildImageViewTarget返回什么

```java

    <R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
        return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
    }

    public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (GlideDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        } 
    	...  	
    }
    public GlideDrawableImageViewTarget(ImageView view) {
        this(view, GlideDrawable.LOOP_FOREVER);
    }
    public GlideDrawableImageViewTarget(ImageView view, int maxLoopCount) {
        super(view);
        this.maxLoopCount = maxLoopCount;
    }
```

返回的是GlideDrawableImageViewTarget实例，再回到上面的into方法中跟进

```java
    public <Y extends Target<TranscodeType>> Y into(Y target) {
        Util.assertMainThread();

        Request previous = target.getRequest();

        if (previous != null) {
            previous.clear();
            requestTracker.removeRequest(previous);
            previous.recycle();
        }

        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);

        return target;
    }
```

getRequest在父类ViewTarget中实现

```java
    @Override
    public Request getRequest() {
        Object tag = getTag();
        Request request = null;
        if (tag != null) {
            if (tag instanceof Request) {
                request = (Request) tag;
            } else {
                throw new IllegalArgumentException("You must not call setTag() on a view Glide is targeting");
            }
        }
        return request;
    }
```

这里会对view的Tag做一个检查，如果不是由glide来setTag的话会抛出异常，这也是为什么有Glide时，处理recyclerView的错位问题，我们不能通过直接对ImageView来setTag判断是否错位。

回归正题，首次使用的时候返回的request应该是null，所以就走的下面这一部分核心代码

```java
        Request request = buildRequest(target);
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);
```

### buildRequest

```java
private Request buildRequest(Target<TranscodeType> target) {
        if (priority == null) {
            priority = Priority.NORMAL;
        }
        return buildRequestRecursive(target, null);
    }

    private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
        if (thumbnailRequestBuilder != null) {
            // Recursive case: contains a potentially recursive thumbnail request builder.
          	...
            return coordinator;
        } else if (thumbSizeMultiplier != null) {
            // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
            ...
            return coordinator;
        } else {
            // Base case: no thumbnail.
            return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
        }
    }
```

就看最基本情况，没有缩略图，返回一个`obtainRequest(target, sizeMultiplier, priority, parentCoordinator);`

```java
private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
            RequestCoordinator requestCoordinator) {
        return GenericRequest.obtain(
                loadProvider,
                model,
                signature,
                context,
                priority,
                target,
                sizeMultiplier,
                placeholderDrawable,
                placeholderId,
                errorPlaceholder,
                errorId,
                fallbackDrawable,
                fallbackResource,
                requestListener,
                requestCoordinator,
                glide.getEngine(),
                transformation,
                transcodeClass,
                isCacheable,
                animationFactory,
                overrideWidth,
                overrideHeight,
                diskCacheStrategy);
    }

private static final Queue<GenericRequest<?, ?, ?, ?>> REQUEST_POOL = Util.createQueue(0);
public static <A, T, Z, R> GenericRequest<A, T, Z, R> obtain(/*省略参数*/) {

        GenericRequest<A, T, Z, R> request = (GenericRequest<A, T, Z, R>) REQUEST_POOL.poll();
        if (request == null) {
            request = new GenericRequest<A, T, Z, R>();
        }
        request.init(/*省略参数*/);
        return request;
    }
```

在这根据一些传入的参数和默认参数构建一个GenericRequest实例。

后面setRequest就略过不提了，然后是`lifecycle.addListener(target);`，这个lifecycle结合代码是with中创建RequestManager的时候传入的`new ApplicationLifecycle()`，而target是GlideDrawableImageViewTarget

```java
public interface Target<R> extends LifecycleListener{
  ...
}

class ApplicationLifecycle implements Lifecycle {
    @Override
    public void addListener(LifecycleListener listener) {
        listener.onStart();
    }
}
//GlideDrawableImageViewTarget.java
    @Override
    public void onStart() {
        if (resource != null) {
            resource.start();
        }
    }
```

resource初始时为null 暂时略过

### runRequest

再来看看`requestTracker.runRequest(request);`

requestTrack也是构造RequestManager的时候传入一个`new RequestTracker();`

```java
    private final Set<Request> requests = Collections.newSetFromMap(new WeakHashMap<Request, Boolean>());    
	public void runRequest(Request request) {
        requests.add(request);
        if (!isPaused) {
            request.begin();
        } else {
            pendingRequests.add(request);
        }
    }
```

上面buildRequest返回的是GenericRequest的实例，跟进

```java
    @Override
    public void begin() {
      ...
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);//后面也会调用到onSizeReady
        }
      
        if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
            target.onLoadStarted(getPlaceholderDrawable());
        }
      ...
    }

```

跟进onSizeReady

```java
 	@Override
    public void onSizeReady(int width, int height) {
 		//修改状态
        status = Status.RUNNING;

        width = Math.round(sizeMultiplier * width);
        height = Math.round(sizeMultiplier * height);

        ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
      
        ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();

        loadedFromMemoryCache = true;
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,priority, isMemoryCacheable, diskCacheStrategy, this);
        loadedFromMemoryCache = resource != null;

    }
```

#### loaderprovider

其中的loadProvider来自上述load方法中的

```java
 //requestManage.java
	private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
      ...
		return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }
 //DrawableTypeRequest.java
    DrawableTypeRequest(/*省略参数*/) {
        super(context, modelClass,
                buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class,
                        GlideDrawable.class, null),//buildProvider参数就是创建loadProvider的方法
                glide, requestTracker, lifecycle);
      
 //DrawableTypeRequest.java     
	private static </*省略*/> FixedLoadProvider</*省略*/> buildProvider(Glide glide,
            ModelLoader<A, InputStream> streamModelLoader,
            ModelLoader<A, ParcelFileDescriptor> fileDescriptorModelLoader, Class<Z> resourceClass,
            Class<R> transcodedClass,
            ResourceTranscoder<Z, R> transcoder) {


        if (transcoder == null) {
            transcoder = glide.buildTranscoder(resourceClass, transcodedClass);
        }
        DataLoadProvider<ImageVideoWrapper, Z> dataLoadProvider = glide.buildDataProvider(ImageVideoWrapper.class,
                resourceClass);
      
        ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                fileDescriptorModelLoader);
      
        return new FixedLoadProvider</*省略*/>(modelLoader, transcoder, dataLoadProvider);
    }
```

可以看到主要在buildProvider中构造了transcoder、dataLoadProvider、modelLoader。

****

具体方法自己跟进源码，记住不要沉迷细节无法自拔：

- transcoder 是 ResourceTranscoder实例

- dataLoadProvider 是 DataLoadProvider实例

- modelLoader 是 ImageVideoModelLoader实例

  返回的是FixedLoadProvider实例

  ​

#### engin.load

理清了loadprovider再回到onSizeReady中

```java
 @Override
    public void onSizeReady(int width, int height) {
      
        ModelLoader<A, T> modelLoader = loadProvider.getModelLoader();
        final DataFetcher<T> dataFetcher = modelLoader.getResourceFetcher(model, width, height);
        ResourceTranscoder<Z, R> transcoder = loadProvider.getTranscoder();

        loadedFromMemoryCache = true;
      
        loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,priority, isMemoryCacheable, diskCacheStrategy, this);
      
        loadedFromMemoryCache = resource != null;

    }
```

在跟进到engine.load中之前，我们先看下这个核心方法的注释，一个有良心的框架，在重要的方法上都应该好好写一下注释。。

```java
     * <p>
     *     The flow for any request is as follows:
     *     <ul>
     *         <li>Check the memory cache and provide the cached resource if present</li>
     *         <li>Check the current set of actively used resources and return the active resource if present</li>
     *         <li>Check the current set of in progress loads and add the cb to the in progress load if present</li>
     *         <li>Start a new load</li>
     *     </ul>
     * </p>
     
       对于任何请求，都按照下面的逻辑步骤来执行：
       - 检测内存缓存，如果有就向View提供缓存的资源
       - 检查当前正在使用的资源，如果有就返回
       - 检查已经进行中的请求，如果已经包含了该资源，就添加一个回调到进行中的请求中
       - 如果上面都没有 就开启一个新的load请求
```

那么就可以把load源码中的部分进行删减，我们就从新开一个load请求这个情况分析

```java
public <T, Z, R> LoadStatus load(/*省略参数*/) {
        Util.assertMainThread();
        long startTime = LogTime.getLogTime();

        final String id = fetcher.getId();
        EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
                loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
                transcoder, loadProvider.getSourceEncoder());

        /*省略三个if检测条件*/

        EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
        DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
                transcoder, diskCacheProvider, diskCacheStrategy, priority);
        EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
        jobs.put(key, engineJob);
        engineJob.addCallback(cb);
        engineJob.start(runnable);

        return new LoadStatus(cb, engineJob);
    }
```

逐个分析吧  EngineJob

```java
/**
 * A class that manages a load by adding and removing callbacks for for the load and notifying callbacks when the load completes.
 一个通过添加和移除回调方法来管理load请求，并且当load完成之后通知回调的类。  有注释就是好哇
 */
class EngineJob implements EngineRunnable.EngineRunnableManager{...}       

public EngineJob build(Key key, boolean isMemoryCacheable) {
            return new EngineJob(key, diskCacheService, sourceService, isMemoryCacheable, listener);
        }
```

DecodeJob看名字就像负责解码的，暂时略过

EngineRunnable

```java
/**
 * A runnable class responsible for using an {@link com.bumptech.glide.load.engine.DecodeJob} to decode resources on a
 * background thread in two stages.
 * <p>
 *     In the first stage, this class attempts to decode a resource
 *     from cache, first using transformed data and then using source data. If no resource can be decoded from cache,
 *     this class then requests to be posted again. During the second stage this class then attempts to use the
 *     {@link com.bumptech.glide.load.engine.DecodeJob} to decode data directly from the original source.
 * </p>

	在第一阶段先尝试从cache中decode出资源，如果没有就直接从original中decode data
 */
class EngineRunnable implements Runnable, Prioritized {...}
```

然后把EngineRunnable放到EngineJob中执行

```java
	private final ExecutorService diskCacheService;    
	public void start(EngineRunnable engineRunnable) {
        this.engineRunnable = engineRunnable;
        future = diskCacheService.submit(engineRunnable);
    }
```

进入到EngineRunnable的run方法中：

```java
    @Override
    public void run() {
        Exception exception = null;
        Resource<?> resource = null;
        try {
            resource = decode();
        } catch (Exception e) {
            exception = e;
        }

        if (resource == null) {
            onLoadFailed(exception);
        } else {
            onLoadComplete(resource);
        }
    }
    private Resource<?> decode() throws Exception {
        if (isDecodingFromCache()) {
            return decodeFromCache();
        } else {
            return decodeFromSource();
        }
    }
    private Resource<?> decodeFromSource() throws Exception {
        return decodeJob.decodeFromSource();
    }
```

最后走到decodeJob来做网络请求

```java
    public Resource<Z> decodeFromSource() throws Exception {
        Resource<T> decoded = decodeSource();
        return transformEncodeAndTranscode(decoded);
    }

    private Resource<T> decodeSource() throws Exception {
        Resource<T> decoded = null;
        try {
            final A data = fetcher.loadData(priority);
            decoded = decodeFromSourceData(data);
        } finally {
            fetcher.cleanup();
        }
        return decoded;
    }
```

根据上面分析，可以知道这个fetcher是从ImageVideoModelLoader中获取的，返回的是ImageVideoFetcher实例。

跟进ImageVideoFetcher的`loadData`

```java
		@Override
        public ImageVideoWrapper loadData(Priority priority) throws Exception {
            InputStream is = null;
            if (streamFetcher != null) {
                try {
                    is = streamFetcher.loadData(priority);
                } catch (Exception e) {
                }
            }
            ParcelFileDescriptor fileDescriptor = null;
            if (fileDescriptorFetcher != null) {
                try {
                    fileDescriptor = fileDescriptorFetcher.loadData(priority);
                } catch (Exception e) {
                }
            }
            return new ImageVideoWrapper(is, fileDescriptor);
        }
```



```java
    @Override
    public DataFetcher<ImageVideoWrapper> getResourceFetcher(A model, int width, int height) {
        DataFetcher<InputStream> streamFetcher = null;
        if (streamLoader != null) {
            streamFetcher = streamLoader.getResourceFetcher(model, width, height);
        }
        DataFetcher<ParcelFileDescriptor> fileDescriptorFetcher = null;
        if (fileDescriptorLoader != null) {
            fileDescriptorFetcher = fileDescriptorLoader.getResourceFetcher(model, width, height);
        }

        if (streamFetcher != null || fileDescriptorFetcher != null) {
            return new ImageVideoFetcher(streamFetcher, fileDescriptorFetcher);
        } else {
            return null;
        }
    }
```

这个streamFetcher结合初始化`loadGeneric`的代码可以知道是HttpUrlFetcher的实例，跟进查看其`loadData`方法

```java
@Override
    public InputStream loadData(Priority priority) throws Exception {
        return loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/, glideUrl.getHeaders());
    }

    private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl, Map<String, String> headers)
            throws IOException {
        if (redirects >= MAXIMUM_REDIRECTS) {
            throw new IOException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
        } else {
            // Comparing the URLs using .equals performs additional network I/O and is generally broken.
            // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
            try {
                if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
                    throw new IOException("In re-direct loop");
                }
            } catch (URISyntaxException e) {
                // Do nothing, this is best effort.
            }
        }
        urlConnection = connectionFactory.build(url);
        for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
          urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
        }
        urlConnection.setConnectTimeout(2500);
        urlConnection.setReadTimeout(2500);
        urlConnection.setUseCaches(false);
        urlConnection.setDoInput(true);

        // Connect explicitly to avoid errors in decoders if connection fails.
        urlConnection.connect();
        if (isCancelled) {
            return null;
        }
        final int statusCode = urlConnection.getResponseCode();
        if (statusCode / 100 == 2) {
            return getStreamForSuccessfulRequest(urlConnection);
        } else if (statusCode / 100 == 3) {
            String redirectUrlString = urlConnection.getHeaderField("Location");
            if (TextUtils.isEmpty(redirectUrlString)) {
                throw new IOException("Received empty or null redirect url");
            }
            URL redirectUrl = new URL(url, redirectUrlString);
            return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
        } else {
            if (statusCode == -1) {
                throw new IOException("Unable to retrieve response code from HttpUrlConnection.");
            }
            throw new IOException("Request failed " + statusCode + ": " + urlConnection.getResponseMessage());
        }
    }
```





```java
//ImageViewTarget.java
    @Override
    public void onLoadStarted(Drawable placeholder) {
        view.setImageDrawable(placeholder);
    }
```

