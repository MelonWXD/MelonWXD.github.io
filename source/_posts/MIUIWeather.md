---
title: 天气预报折线图
date: 2017-09-23 15:34:58
tags: [自定义View,Geather]
categories: Android
---
自定义View高仿小米天气24小时天气预报折线图
<!-- more -->
## 项目地址
[HourlyForecastView](https://github.com/MelonWXD/HourlyForecastView)


## 需求分析
- 圆点的宽高(宽为固定的，高与该时刻温度值线性相关）
- 虚线宽度的判定（根据数据源来）
- 动画效果（图片总是在 虚线或虚线与屏幕边缘中间）

## 数据源
[饥人谷24小时天气API](http://weixin.jirengu.com/weather/future24h?cityid=WX4FBXXFKE4F)

[饥人谷开放API](api.jirengu.com)

Json格式部分如下，完整见上述网址
```
{
    "status": "OK",
    "hourly": [
        {
            "text": "晴",
            "code": "1",
            "temperature": "17",
            "time": "2017-08-30T02:00:00+08:00"
        },
        {
            "text": "晴",
            "code": "1",
            "temperature": "17",
            "time": "2017-08-30T03:00:00+08:00"
        }
    ]
}
```
 
使用Android Studio的插件GsonFormat就能根据Json，自动生成bean类了。
> 代码块中 ... 为省略无关代码  下同  

```java
public class HourlyWeatherBean {

    /**
     * text : 多云
     * code : 4
     * temperature : 28
     * time : 2017-08-30T02:00:00+08:00
     */

    private String text;
    private String code;
    private String temperature;
    private String time;

    public HourlyWeatherBean(String text, String code, String temperature, String time) {
        this.text = text;
        this.code = code;
        this.temperature = temperature;
        this.time = time;
    }

    ...
    getter 
    setter
    ...
}
```

## 初始化与Utils

### 初始化与默认值  
将json数据转换为beanList传给View的initData方法
```java
private void initView() {

        ...
        List<HourlyWeatherBean> hourlyWeatherList = new ArrayList<>();
        Gson gson = new Gson();
        JsonObject jo = new JsonParser().parse(jsonData).getAsJsonObject();
        JsonArray ja = jo.getAsJsonArray("hourly");
        for (JsonElement element : ja) {
            HourlyWeatherBean bean = gson.fromJson(element, new TypeToken<HourlyWeatherBean>() {
            }.getType());
            hourlyWeatherList.add(bean);
        }

        //设置当天的最高最低温度
        hourlyForecastView.setHighestTemp(27);
        hourlyForecastView.setLowestTemp(16);
        hourlyForecastView.initData(hourlyWeatherList);
         ...
    }
```
根据传入的数据 在View内部确认画虚线的位置 同时初始化默认值和画笔
```java
    public void initData(List<HourlyWeatherBean> weatherData) {

        hourlyWeatherList = weatherData;
        dashLineWidth = new ArrayList<>();
        Iterator iterator = hourlyWeatherList.iterator();
        HourlyWeatherBean tmp;
        String lastText = "";
        int idx = 0;
        while (iterator.hasNext()) {
            tmp = (HourlyWeatherBean) iterator.next();
            if (!tmp.getText().equals(lastText)) {
                dashLineWidth.add(idx);//从0开始添加虚线位置的索引值idx
                lastText = tmp.getText();
            }
            idx++;
        }
        dashLineWidth.add(hourlyWeatherList.size() - 1);//添加最后一条虚线位置的索引值idx

        initDefValue();
        initPaint();

    }
```

### Utils工具类
包括了dp/sp转换 以及图片压缩方法 代码就不贴了  

## onMeasure  

```java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        //当设置的padding值小于默认值是设置为默认值
        paddingL = Math.max(paddingL, getPaddingLeft());
        paddingT = Math.max(paddingT, getPaddingTop());
        paddingR = Math.max(paddingR, getPaddingRight());
        paddingB = Math.max(paddingB, getPaddingBottom());

        //获取测量模式
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);

        //获取测量大小
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);


        if (widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY) {
            mWidth = widthSize + paddingL + paddingR;
            mHeight = heightSize;
        }

        //如果为wrap_content 那么View大小为默认值
        if (widthMode == MeasureSpec.UNSPECIFIED && heightMode == MeasureSpec.AT_MOST) {
            mWidth = defWidthPixel + paddingL + paddingR;
            mHeight = defHeightPixel + paddingT + paddingB;
        }
       
        //设置视图的大小
        setMeasuredDimension(mWidth, mHeight);
    }

```
当设置的padding值小于默认值时，将padding设为默认值，来保证左右两边都有足够空间来绘制

需要注意的是 HorizontalScrollView的子View 在没有明确指定dp值的情况下 widthMode总是MeasureSpec.UNSPECIFIED 同理 ScrollView的子View的heightMode也是同样的情况  


## onDraw  

```java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);
        if (hourlyWeatherList.size() != 0) {
            drawLines(canvas);
            drawBitmaps(canvas);
        }
    }    
 ```
 ### drawLines
 ```java
 private void drawLines(Canvas canvas) {
        //底部的线的高度 高度为控件高度减去text高度的1.5倍
        float baseLineHeight = mHeight - 1.5f * textSize;
        Path p = new Path();

        for (int i = 0; i < hourlyWeatherList.size(); i++) {
            float temp = Integer.parseInt(hourlyWeatherList.get(i).getTemperature());
            float w = itemWidth * i + paddingL;
            float h = tempHeightPixel(temp) + paddingT;
            if (i == 0) {
                p.moveTo(w, h);
            } else {
                p.lineTo(w, h);
            }

            //画虚线
            if (dashLineWidth.contains(i)) {
                canvas.drawLine(w, h, w, baseLineHeight, dashPaint);
            }

        }
        //画折线
        canvas.drawPath(p, foldLinePaint);
        //画底线
        canvas.drawLine(paddingL, baseLineHeight, mWidth - paddingR, baseLineHeight, baseLinePaint);
        ...
 	}
 ```
  取名不大会取，这个drawLines方法是绘制了所有非图片的东西  
 包括底部的线、温度折线以及虚线  
 通过遍历数据，使用Path类来确定折线的路径
```java
private void drawLines(Canvas canvas) {
 ...
 for (int i = 0; i < hourlyWeatherList.size(); i++) {
            float temp = Integer.parseInt(hourlyWeatherList.get(i).getTemperature());

            float w = itemWidth * i + paddingL;
            float h = tempHeightPixel(temp) + paddingT;

            //画大白圆
            circlePaint.setColor(getResources().getColor(R.color.white));
            circlePaint.setStyle(Paint.Style.FILL);
            canvas.drawCircle(w, h, defRadius + 6, circlePaint);
            //画小蓝圆
            circlePaint.setColor(getResources().getColor(R.color.dodgerblue));
            circlePaint.setStyle(Paint.Style.STROKE);
            canvas.drawCircle(w, h, defRadius, circlePaint);

            //画温度值  y轴是文本基线 故除2处理
            canvas.drawText(hourlyWeatherList.get(i).getTemperature(), w, h - textSize / 2f, textPaint);
            //画时间
            canvas.drawText(hourlyWeatherList.get(i).getTime(), w, baseLineHeight + textSize, textPaint);
        }
}
```
2个圆，在折线和虚线绘制完毕后，绘制一个白色的大圆，再绘制一个半径略小的蓝色小圆，这样就能达到圆点和线之间相隔一定距离的效果  
还有时间点和温度值，其中温度值的高度是由如下方法确定的，highestTemp和lowestTemp表示当天最高温和最低温的值，highestTempHeight表示长度(不是屏幕y轴的值，所以返回值需要 默认的View高度(y轴值)减去结果），根据该时间的温度值的比例，即可得到对应的高度
 ```java
     public float tempHeightPixel(float tmp) {
        float res = ((tmp - lowestTemp) / (highestTemp - lowestTemp)) * (highestTempHeight - lowestTempHeight) + lowestTempHeight;
        return defHeightPixel - res;
    } 
```
### 硬件加速的小坑
如果出现虚线绘制无效的情况，在Manifest中Activity标签下关闭硬件加速
```
android:hardwareAccelerated="false"
```
### drawBitmaps
绘制图片是这个自定义View最重要的地方，因为他需要满足如下规则:  
  
1. 左右虚线都显示在屏幕内，图片在两边虚线中间
2. 左虚线在屏幕外，图片在屏幕左边缘与右虚线中间
3. 右虚线在屏幕外，图片在左虚线与屏幕右边缘中间
4. 两个虚线都在屏幕外，图片在屏幕中央
5. 滑动过程图片不超过虚线
分析清楚了，那么代码也就好写了
```java
 private void drawBitmaps(Canvas canvas) {
		int scrollX = mScrollX;
        boolean leftHide;
        boolean rightHide;
        for (int i = 0; i < dashLineWidth.size() - 1; i++) {
            leftHide = true;
            rightHide = true;

            int left = itemWidth * dashLineWidth.get(i) + paddingL;
            int right = itemWidth * dashLineWidth.get(i + 1) + paddingL;
            float drawPoint = 0;//图的中间位置  drawBitmap是左边开始画
            if (left > scrollX && left < scrollX + screenWidth) {
                leftHide = false;//左边缘显示
            }
            if (right > scrollX && right < scrollX + screenWidth) {
                rightHide = false;
            }

            if (!leftHide && !rightHide) {//左右边缘都显示
                drawPoint = (left + right) / 2f;

            } else if (leftHide && !rightHide) {//右边缘与屏幕左边

                drawPoint = (scrollX + right) / 2f;
            } else if (!leftHide) {//左边缘与屏幕右边
                //rightHide is True when reach this statement
                drawPoint = (left + screenWidth + scrollX) / 2f;

            } else {//左右边缘都不显示
                if (right < scrollX + screenWidth) { //左右边缘都在屏幕左边
                    continue;
                } else if (left > scrollX + screenWidth) {//左右边缘都在屏幕右边
                    continue;
                } else {
                    drawPoint = (screenWidth) / 2f + scrollX;
                }
            }
}
```
注释已经写的很清楚了，最后得到drawPoint的位置，还得满足第五点，不超过虚线，如下
```java
//越界判断
            if (drawPoint >= right - bitmap.getWidth() / 2f) {
                drawPoint = right - bitmap.getWidth() / 2f;
            }
            if (drawPoint <= left + bitmap.getWidth() / 2f) {
                drawPoint = left + bitmap.getWidth() / 2f;
            }
```
拿到了正确的位置，调用canvas的drawBitmap就完成了图片的绘制

## Scroll滑动
由于继承的是View，所以没有ScrollListener，即上面drawBitmaps的Line1，判断位置最关键的scrollX的值，需要咱们自己想办法来计算  
有两种方案来实现，一个就是重写onTouchEvent方法，利用Scroller 和 VelocityTracker 来实现。或者你也可以像我一样偷懒，悄悄偷个鸡..  
### 通过回调实现scroll值的传入
因为外部嵌套的是HorizontalScrollView，而HorizontalScrollView已经实现了滑动的监听，那么只需要在HorizontalScrollView的onScrollChange方法里，拿到scrollX的值，并传给这个自定义View即可。  
通过设计模式的观察者模式来实现这一功能：
```java
public interface ScrollWatcher {
    void update(int scrollX);
}
public interface ScrollWatched {
    void addWatcher(ScrollWatcher watcher);
    void removeWatcher(ScrollWatcher watcher);
    void notifyWatcher(int x);
}
```
让HourlyForecastView实现ScrollWatcher接口，在update中将参数赋值给类变量mScrollX

```java
    private void initObserver() {
        watcherList = new ArrayList<>();
        watched = new ScrollWatched() {
            @Override
            public void addWatcher(ScrollWatcher watcher) {
                watcherList.add(watcher);
            }

            @Override
            public void removeWatcher(ScrollWatcher watcher) {
                watcherList.remove(watcher);
            }

            @Override
            public void notifyWatcher(int x) {
                for (ScrollWatcher watcher : watcherList) {
                    watcher.update(x);
                }
            }
        };
    }
```

```java
private void initView() {
		...
        watched.addWatcher(hourlyForecastView);
        horizontalScrollView.setOnScrollChangeListener(new View.OnScrollChangeListener() {
            @Override
            public void onScrollChange(View v, int scrollX, int scrollY, int oldScrollX, int oldScrollY) {
                watched.notifyWatcher(scrollX);
            }
        });
        ...
}
```
在MainActivity中实例化ScrollWatched类，并将的实例添加进去在HorizontalScrollView的onScrollChange方法里调用notifyWatcher，参数为scrollX

### 重写onTouchEvent
这里啥也莫得，有空再写吧..
 
## 写在最后
最新电脑配置升级了，代码写起来更方便了，不用build的时候先去打把游戏再回来看了，打算陆陆续续把之前的笔记整理并发出来，感觉写一篇耗时比我想的久，这篇从3点多一直到现在5点40分，居然花了整整2个小时，看来要把写博客的时间安排在写完代码之后，写的时候自言自语感觉蛮好玩的，思路也清晰了不少。
