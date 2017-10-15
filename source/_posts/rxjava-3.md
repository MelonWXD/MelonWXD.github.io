---
title: RxJava(三)  
date: 2017-10-15 14:35:43  
tags: [RxJava]
categories: Android
---
RxJava 操作符  
<!-- more -->
## 参考
一下根据操作符的类型进行说明，不会将每个类别下的都讲一遍，具体请查阅[官方文档](http://reactivex.io/documentation/operators.html)  
[中文文档](https://mcxiaoke.gitbooks.io/rxdocs/content/)

## 创建Observable的操作符
### create
最基本的创建被观察者的方式
```java
Observable<String> observable =Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                emitter.onNext("Event1");
                emitter.onNext("Event2");
                emitter.onNext("Event3");
                emitter.onComplete();
            }
        });
```  
### just
将对象或者对象集合转换成被观察者，并将值发送出去。可以看到下面代码中，Consumer的accept方法参数是List类型的。
```java
        List<Integer> integers = new ArrayList<>();
        integers.add(1);
        integers.add(2);
        integers.add(3);
        
        Observable.just(integers)
                .subscribeOn(AndroidSchedulers.mainThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<List<Integer>>() {
                    @Override
                    public void accept(List<Integer> integers) throws Exception {

                    }
                });
```
### from
作用跟just差不多，唯一区别就是他是将可迭代的数据结构中的对象一个个发送的，惨Consumer的accept的数据类型就知道跟just的区别了
```java
        Observable.fromIterable(integers)
                .subscribeOn(AndroidSchedulers.mainThread())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
				
                    }
                });
```


## 转化Observable类型的操作符
### FlatMap 
将Observable发送的数据经过转换再发送给观察者
![](http://reactivex.io/documentation/operators/images/flatMap.c.png)
与之差不多的是 ConcatMap，看上图可以知道FlatMap是无序的（绿色球和蓝色球在观察者中串在一起了），而ConcatMap是有序的，就保证了绿色四边形全都是在蓝色四边形之前收到的。
例子使用瓜乐天气里，场景如下，先get获取当前ip地址，再拿到ip地址去请求获取城市的编码id，然后对id进行处理。
```java
   public void getWeatherByIp() {
        mNetClient.getCurCityIP()
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.io())
                .concatMap(new Function<Response<ResponseBody>, ObservableSource<ResponseBody>>() {
                    @Override
                    public ObservableSource<ResponseBody> apply(Response<ResponseBody> response) throws Exception {

                        JsonObject jo = Utils.string2Json(response.body().string());
                        String ipAddr = jo.get("data").getAsString();
                        return mNetClient.getCityID(ipAddr);
                    }
                })
                .observeOn(Schedulers.io())
                .subscribe(new Consumer<ResponseBody>() {
                    @Override
                    public void accept(ResponseBody responseBody) throws Exception {
                        JsonObject jo = new JsonParser().parse(responseBody.string()).getAsJsonObject();
                        JsonArray ja = jo.getAsJsonArray("results");
                        String cityID = ja.get(0).getAsJsonObject().get("id").getAsString();
                        LogUtil.I("id=" + cityID);

                        getCityWeatherByID(cityID);
                        getHourlyByID(cityID);
                    }
                });
        
    }
```
在concatMap中通过new Function把getCurCityIP返回的结果，处理得到ipAddr，然后返回mNetClient.getCityID(ipAddr)，这也是个Observable对象（结合retrofit），然后在观察者根据getCityID的ResponseBody获得城市id，再进一步做其他请求。


### groupBy
使用groupBy来制定分组规则，对数据源进行分组，这样观察者就可以通过自动生成的GroupedObservable的getKey方法来获得分组key，从而实现对分组数据的获取。
上一个网上找的例子
```java

Observable.range(1, 8).groupBy(new Func1<Integer, String>() {  
            @Override  
            public String call(Integer integer) {  
                return (integer % 2 == 0) ? "偶数组" : "奇数组";  
            }  
        }).subscribe(new Action1<GroupedObservable<String, Integer>>() {  
            @Override  
            public void call(final GroupedObservable<String, Integer> stringIntegerGroupedObservable) {  
                System.out.println("group name:" + stringIntegerGroupedObservable.getKey());  
                if (stringIntegerGroupedObservable.getKey().equalsIgnoreCase("奇数组"))  
                    stringIntegerGroupedObservable.subscribe(new Action1<Integer>() {  
                        @Override  
                        public void call(Integer integer) {  
                            System.out.println(stringIntegerGroupedObservable.getKey() + "'member: " + integer);  
                        }  
                    });  
            }  
        }); 
```
#### 注意事项
gourpBy操作符将数据源进行分组，生成多个GroupedObservable，一旦有观察者开始订阅，每个GroupedObservable都会开始缓存，如果你只对其中一个分组的数据进行处理，就像例子中那样，对欧数组没有进行处理，那么可能造成内存泄漏。所以如果不需要该组数据，也要接收并处理掉。（官方文档上写的是take(0)，就是只接收前0个，那这样我只要奇数组不就不行了）

### buffer
定期的将Observable要发送的数据放在一起同时发送给观察者，而不是一个个发送。
![](http://reactivex.io/documentation/operators/images/Buffer.png)


## 不写了 健身去
写一半找例子的时候发现居然有人[翻译](https://mcxiaoke.gitbooks.io/rxdocs/content/)了  
这么火的技术，肯定早就有人翻译了..  
![](http://ww2.sinaimg.cn/large/9150e4e5ly1fjkl5d4d1tj204g04fglo.jpg)


