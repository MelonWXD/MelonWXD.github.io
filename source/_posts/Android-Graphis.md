---
title: Android Graphis
date: 2017-09-22 23:00:07
tags: [Paint,Canvas]
categories: Android
---
# Paint

### Paint的style，共有3种

- Paint.Style.FILL：填充内部
- Paint.Style.FILL_AND_STROKE  ：填充内部和描边
- Paint.Style.STROKE  ：描边

### 常用属性

```
//重置Paint。
reset()
//设置一些标志，比如抗锯齿，下划线等等。
setFlags(int flags)
//设置抗锯齿，如果不设置，加载位图的时候可能会出现锯齿状的边界，如果设置，边界就会变的稍微有点模糊，锯齿就看不到了。
setAntiAlias(boolean aa)
//设置是否抖动，如果不设置感觉就会有一些僵硬的线条，如果设置图像就会看的更柔和一些，
setDither(boolean dither)
//这个是文本缓存，设置线性文本，如果设置为true就不需要缓存，
setLinearText(boolean linearText)
//设置亚像素，是对文本的一种优化设置，可以让文字看起来更加清晰明显，可以参考一下PC端的控制面板-外观和个性化-调整ClearType文本
setSubpixelText(boolean subpixelText)
//设置文本的下划线
setUnderlineText(boolean underlineText)
//设置文本的删除线
setStrikeThruText(boolean strikeThruText)
//设置文本粗体
setFakeBoldText(boolean fakeBoldText)
//对位图进行滤波处理，如果该项设置为true，则图像在动画进行中会滤掉对Bitmap图像的优化操作，加快显示 
setFilterBitmap(boolean filter)
//下面这几个就不用说了，上面已经演示过
setStyle(Style style)，setStrokeCap(Cap cap)，setStrokeJoin(Join join)，setTextAlign(Align align)，
//设置画笔颜色
setColor(int color)
//设置画笔的透明度[0-255]，0是完全透明，255是完全不透明
setAlpha(int a)
//设置画笔颜色，argb形式alpha，red，green，blue每个范围都是[0-255],
setARGB(int a, int r, int g, int b)
//画笔样式为空心时，设置空心画笔的宽度
setStrokeWidth(float width)
//当style为Stroke或StrokeAndFill时设置连接处的倾斜度，这个值必须大于0，看一下演示结果
setStrokeMiter(float miter)
左上角的没有设置setStrokeMiter，右上角setStrokeMiter(2.3f)，左下角setStrokeMiter(1.7f)，右下角setStrokeMiter(0f)
```
## 画虚线

DashPathEffect 构造方法的参数决定了绘制的路径效果：


```
public DashPathEffect(
    float[] intervals,
    float phase )
```

#### DashPathEffect 画虚线无效 
#### 在view层关闭硬件加速，直接在自定义View的构造方法中调用

```
 setLayerType(View.LAYER_TYPE_SOFTWARE, null);
```
#### 这句放在 < application />节点下表示关闭整个项目的硬件加速，放在 < activity />下表示关闭该组件硬件加速
```
android:hardwareAccelerated="false" 

```


## 获取字体宽度和高度（得到的值和Paint设置TextSize有关）

```
Paint paint = new Paint(); 
float strWidth = paint.measureText(String); 
paint.setTextSize(tv.getTextSize()); 

String test = "QinShiMingYue";
Rect rect = new Rect();
mPaint.getTextBounds(text, 0, test.length(), rect);
int width = rect.width();//文字宽
int height = rect.height();//文字高

```
intervals是一个float数组，且其长度必须是偶数且>=2，指定了多少长度的实线之后再画多少长度的空白。

# Path
### api

```
reset()                           
lineTo(float x, float y)
moveTo(float x, float y)
close()

//用于绘制圆弧，这个圆弧取自RectF矩形的内接椭圆上的一部分，圆弧长度由后两个参数决定
//startAngle：起始位置的角度值
//sweepAngle：旋转的角度值
addArc(oval, startAngle, sweepAngle)

//moveTo 后接圆弧,如果圆弧的起点和上次最后一个坐标点不相同，就连接两个点
arcTo(RectF oval, float startAngle, float sweepAngle)

//(x1,y1)为控制点，(x2,y2)为结束点画一条二次贝塞尔曲线
quadTo(float x1, float y1, float x2, float y2)

//绘制圆
addCircle(float x, float y, float radius, Direction dir)

//绘制椭圆
addOval(RectF oval, Path.Direction dir)
addPath(Path src, float dx, float dy)
```


# Canvas
### api

```
drawRect(RectF rect, Paint paint) //绘制区域，参数一为RectF一个区域 

drawPath(Path path, Paint paint) //绘制一个路径，参数一为Path路径对象

drawBitmap(Bitmap bitmap, Rect src, Rect dst, Paint paint)  //贴图，参数一就是我们常规的Bitmap对象，参数二是源区域(这里是bitmap)，参数三是目标区域(应该在canvas的位置和大小)，参数四是Paint画刷对象，因为用到了缩放和拉伸的可能，当原始Rect不等于目标Rect时性能将会有大幅损失。

drawLine(float startX, float startY, float stopX, float stopY, Paintpaint) //画线，参数一起始点的x轴位置，参数二起始点的y轴位置，参数三终点的x轴水平位置，参数四y轴垂直位置，最后一个参数为Paint 画刷对象。

drawPoint(float x, float y, Paint paint) //画点，参数一水平x轴，参数二垂直y轴，第三个参数为Paint对象。

drawText(String text, float x, floaty, Paint paint)  //渲染文本，Canvas类除了上面的还可以描绘文字，参数一是String类型的文本，参数二x轴，参数三y轴，参数四是Paint对象。

drawOval(RectF oval, Paint paint)//画椭圆，参数一是扫描区域，参数二为paint对象；

drawCircle(float cx, float cy, float radius,Paint paint)// 绘制圆，参数一是中心点的x轴，参数二是中心点的y轴，参数三是半径，参数四是paint对象；

drawArc(RectF oval, float startAngle, float sweepAngle, boolean useCenter, Paint paint)//画弧，
```

### drawText  y轴为文本基线
参数x：绘制文本起源的x坐标 
参数y：绘制文本基线的y坐标
## save 和 restore
restroe类似于将图层对齐后合并


```
        canvas.drawCircle(50, 50, 50, mPaint);
        canvas.translate(100, 100);
        canvas.drawCircle(50, 50, 50, mPaint);
```
有2个点

```

        canvas.save();  //保存当前canvas的状态
        canvas.drawCircle(50, 50, 50, mPaint);
        canvas.translate(100, 100);
        canvas.restore();  //恢复保存的Canvas的状态
        canvas.drawCircle(50, 50, 50, mPaint);
```
只有1个点

## 画布上的移动 实际上是我们通过井字形的窥孔在动   要区别好相对坐标与实际的画布坐标 
