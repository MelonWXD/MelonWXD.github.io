---
title: Http的缓存控制 
date: 2018-01-11 22:02:59
tags: [网络编程]  
categories: 网络基础
---
Http中关于缓存相关的知识与Android网络框架设置缓存的方法。

<!-- more -->  

# 常见缓存头

## Expires 过期时间

告知 客户端/缓存服务器 资源失效的日期，在Expires指定的日期之前，服务器响应的副本会一直被保留，作为响应发送给请求。如果在日期之后，缓存服务器就会向源服务器请求资源。

## Cache-Control 缓存控制

Cache-Control与Expires的作用一致，但Cache-Control的选择更多，设置更细致，如果同时设置的话，其优先级高于Expires。

指令的参数如下：

![](http://owu391pls.bkt.clouddn.com/http-cache1.png)

![](http://owu391pls.bkt.clouddn.com/httpcache2.png)

参数是可选的，多个指令之间通过逗号“,”分隔。首部字段 Cache-Control 的指令可用于请求和响应。

下面挑两个常见的指令来讲一下：

- no-cache:【请求】表示客户端强制缓存服务器向源服务器重新获取资源。【响应】表示源服务器禁止缓存服务器对资源进行缓存。
- max-age:【请求】在max-age指定时间内获取的是缓存服务器的缓存。当max-age为0的时候缓存服务器通常要将请求转发给源。【响应】表示缓存在max-age指定时间内不需要再像源服务器请求资源。（若于Expires同时存在，HTTP/1.1 中忽略Expires，HTTP/1.0中忽略max-age）

## Last-Modified/If-Modified-Since

Last-Modified:源服务器在响应请求时告诉客户端这个响应资源的最后修改时间

If-Modified-Since:客户端通过该字段指定一个时间T1，如果服务器上该资源最后更新日期在T1之后，那么就正常响应返回资源。如果在T1之前都没有更新过，则返回304(Not Modified)。

## ETag/If-None-Match

ETag:当资源被缓存的时候，会被分配唯一的标识id(类似于hash，生成规则不固定，由服务器来分配)，当数据没有改变的时候，服务器返回304(Not Modified)。当缓存数据改变了，ETag也随着改变，服务器就会重新返回资源给你。而客户端请求服务器时，是通过If-None-Match来携带已缓存资源的ETag值。



# OKHTTP中设置缓存

## 常规设置

```java
//缓存文件夹
File cacheFile = new File(getExternalCacheDir().toString(),"cache");
//缓存大小为10M
int cacheSize = 10 * 1024 * 1024;
//创建缓存对象
Cache cache = new Cache(cacheFile,cacheSize);

OkHttpClient client = new OkHttpClient.Builder()
        .cache(cache)
        .build();
```

这样的话当服务器返回的响应头中有Cache-Control时OkHttp自动就会帮我们缓存了。但是如果服务器不支持的话就需要我们手动添加了。

##拦截缓存 

跟拦截打印请求日志一样，我们可以手动在响应报文中添加Cache-Control

```java
class CacheInterceptor implements Interceptor{

        @Override
        public Response intercept(Chain chain) throws IOException {

            Response originResponse = chain.proceed(chain.request());

            //设置缓存时间为60秒，并移除了pragma消息头，移除它的原因是因为pragma也是控制缓存的一个消息头属性
            return originResponse.newBuilder().removeHeader("pragma")
                    .header("Cache-Control","max-age=60").build();
        }
    }
```





# 参考

[浏览器 HTTP 协议缓存机制详解](https://my.oschina.net/leejun2005/blog/369148)

图解HTTP-第六章