---
title: Android OpenGL的简单使用
date: 2017-11-04 14:43:19
tags: [OpenGL]
categories: Android
---
Android OpenGL的Demo
<!-- more -->
# 写在前面的话
最近在弄framework层的注入，又是学习Android系统的启动流程，IPC，还有Android的整个GUI渲染机制，看源码看了一礼拜，眼睛疼，擦。。总感觉没有成功的打通整个环节，从Android系统的启动，到执行开机通话这个环节，又得研究游戏Apk是怎么调用OpenGL的渲染，为什么不走eglSwapBuffer这个方法，我hook了又不调，烦得很哎。。还是找个简单的来做一做，先研究在Android应用上面openGL的使用吧哎。。想到这一个礼拜来一直沉迷源码，赶紧停下来写写java，我就顺手把OpenGL在安卓的应用这一块理一理吧。。唉。。 

# SurfaceView
先简略的说一下SurfaceFlinger，这就是一个系统服务，负责系统底层GUI这一块的管理，后面我会详细的把这一块理一理。  
通常来说，我们一个Activity(或者说一个Window)都在SurfaceFlinger中，对应这一个Layer（见下图），像一般我们在Activity的布局中，需要显示Button啊、TextView这种的，都是往这个对应的Layer填充数据，然后由SurfaceFlinger来对各种各样的Layer进行合成渲染，最后控制底层显示在屏幕上。然后像Button这种的控件的绘制，都是在UI主线程中进行的，如果需要频繁的刷新界面，比如直播啊拍照的摄像头预览，那肯定就比较耗时嘛，耗时你主线程就没办法对用户的输入进行响应，容易ANR，所以就有了SurfaceView这个东西，他在SurfaceFlinger中单独对应一个Layer(LayerBuffer)，通过子线程来对它进行更新。
![](http://img.my.csdn.net/uploads/201303/11/1363016714_1787.jpg)  
SurfaceView继承View，有两个子类GLSurfaceView和VideoView 
## 使用
创建一个自定义的MySurfaceView，继承自SurfaceView，并实现两个接口SurfaceHolder.CallBack和Runnable(都是为了方便，如果代码复杂的应该要分别实现MySurfaceHolder已经DrawingThread来分别对应周期控制与子线程绘制)。  
SurfaceHolder.CallBack的三个方法分别对应着SurfaceView的三个生命周期
```
    public interface Callback {
        void surfaceCreated(SurfaceHolder var1);

        void surfaceChanged(SurfaceHolder var1, int var2, int var3, int var4);

        void surfaceDestroyed(SurfaceHolder var1);
    }
```
由此我们可以猜测这个Holder可能是Surface的主要控制类，看看这个类的部分方法，不难知道SurfaceHolder控制着SurfaceView的大小，格式，可以监控或者改变SurfaceView。
```
    void setFixedSize(int var1, int var2);

    void setSizeFromLayout();

    void setFormat(int var1);

    void setKeepScreenOn(boolean var1);

    Canvas lockCanvas();

    Canvas lockCanvas(Rect var1);

    default Canvas lockHardwareCanvas() {
        throw new RuntimeException("Stub!");
    }

    void unlockCanvasAndPost(Canvas var1);

    Rect getSurfaceFrame();

    Surface getSurface();
```
需要绘制的话，还需要一个画布类Canvas，在对应的生命周期中开始绘制并且提交，Holder也提供了相应的方法
```
mCanvas = mHolder.lockCanvas(); //lock
mHolder.unlockCanvasAndPost(mCanvas);//unLock Post
```
### 完整代码
```java
public class MySurfaceView extends SurfaceView implements SurfaceHolder.Callback,Runnable{

    SurfaceHolder mHolder ;
    // 画布
    private Canvas mCanvas;
    // 子线程标志位
    private boolean isDrawing;

    public MySurfaceView(Context context, AttributeSet attrs) {
    
        super(context, attrs);
        init();
    }

    private void init() {
        mHolder = getHolder();//得到SurfaceHolder对象
        mHolder.addCallback(this);//注册SurfaceHolder
        setFocusable(true);
        setFocusableInTouchMode(true);
        this.setKeepScreenOn(true);//保持屏幕长亮
    }



    @Override
    public void surfaceCreated(SurfaceHolder holder) {//创建
        isDrawing = true;
        new Thread(this).start();
    }

    @Override
    public void surfaceChanged(SurfaceHolder holder, int format, int width, int height) {//改变

    }

    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {//销毁
        isDrawing = false;
    }


    @Override
    public void run() {
        while (isDrawing){
            drawing();
        }
    }

    private void drawing() {
        try {
            mCanvas = mHolder.lockCanvas();
            //这里进行内容的绘制

        }finally {
            if (mCanvas != null){
                mHolder.unlockCanvasAndPost(mCanvas);
            }
        }
    }


}
```
    
 
# GLSurfaceView 和 Renderer
SurfaceView的子类，功能和SurfaceView相似。我们可以创建一个GLSurfaceView类的实例，实现GLSurfaceView.Renderer接口添加自定义渲染。GLSurfaceView没什么好说的，主要还是他的这个渲染器接口Renderer
```
    public interface Renderer {
        void onSurfaceCreated(GL10 var1, EGLConfig var2);

        void onSurfaceChanged(GL10 var1, int var2, int var3);

        void onDrawFrame(GL10 var1);
    }
```
这个接口有3个方法  
- onSurfaceCreated() surface被创建或者重新创建的时候调用。当线程开始渲染时或者EGL Context丢失时也会调用(EGL是Android系统对OpenGL的上层封装扩展，EGL Context通常在系统屏幕黑掉的时候丢失)。
- onSurfaceChanged()- 如果View的几和形状发生变化了就调用
- onDrawFrame()- 绘制当前的帧，每次View被重绘时被调用。

## 完整代码
根据以上3个方法的介绍，基本可以确定是onSurfaceCreated里面做一些配置任务，官网也推荐我们在这个方法里创建一些资源等。然后在onDrawFrame里面实现每一帧的绘制任务。
具体的位置就是要参考openGL的API了，先写上吧，以后有需要再进一步学习
```

public class SunnyGLRender  implements GLSurfaceView.Renderer {

    public float mAngle;
    float one = 0.5f;
    private FloatBuffer triggerBuffer2 = MainActivity.BufferUtil.floatToBuffer(new float []{
            0,one,0, //上顶点
            -one,-one,0, //左下点
            one,-one,0,}); //右下点
    private FloatBuffer triggerBuffer1 = MainActivity.BufferUtil.floatToBuffer(new float []{
            0,one,0, //上顶点
            -one,-one,0, //左下点
            one,-one,0,}); //右下点
    private float[] mTriangleArray = {
            // X, Y, Z 这是一个等边三角形
            -0.5f, -0.25f, 0,
            0.5f, -0.25f, 0,
            0.0f, 0.559016994f, 0 };

    @Override
    public void onSurfaceCreated(GL10 gl, EGLConfig config) {

        //GLES30:为OpenGL ES2.0版本,相应的
        //GLES30:OpenGL ES3.0
        Log.e("zcxgl","onSurfaceCreated");

        //黑色背景
        gl.glClearColor(0.6f, 0f, 0.5f, 0.5f);
        //gl.glClear(GL10.GL_COLOR_BUFFER_BIT);
        // 启用顶点数组（否则glDrawArrays不起作用）
        gl.glEnableClientState(GL10.GL_VERTEX_ARRAY);

    }

    @Override
    public void onSurfaceChanged(GL10 gl, int width, int height) {
        //mProgram = GLES30.glCreateProgram();
        //Log.e("zcxgl", "Could not link program: mProgram = "+mProgram);
//        Log.e("zcxgl","onSurfaceChanged");
        float ratio = (float) width / height;
        Log.e("zcxgl","onSurfaceChanged:"+ratio+","+width+","+height);

        gl.glMatrixMode(GL10.GL_PROJECTION); // 设置当前矩阵为投影矩阵
        gl.glLoadIdentity(); // 重置矩阵为初始值
        gl.glFrustumf(-ratio, ratio, -1, 1, 3, 7); // 根据长宽比设置投影矩阵
//        gl.glFrustumf(-ratio, ratio, -1, 1, 5, 6);

    }
    private FloatBuffer colorBuffer2 = MainActivity.BufferUtil.floatToBuffer(new float[]{
            one,0,0,one,
            0,one,0,one,
            0,0,one,one,
    });
    @Override
    public void onDrawFrame(GL10 gl) {
        // Redraw background color
        Log.d("zcxgl","onDrawFramew");
        gl.glClear(GL10.GL_COLOR_BUFFER_BIT);
        gl.glColor4f(1.0f, 0.0f, 0.0f, 1.0f);

        /************ 启用MODELVIEW模式，并使用GLU.gluLookAt()来设置视点 ***************/
        // 设置当前矩阵为模型视图模式
        gl.glMatrixMode(GL10.GL_MODELVIEW);
        gl.glLoadIdentity(); // reset the matrix to its default state
        // 设置视点
        GLU.gluLookAt(gl, 0, 0, -5, 0f, 0f, 0f, 0f, 1.0f, 0.0f);
        /*****************************************/

        long time = SystemClock.uptimeMillis() % 4000L;
        mAngle = 0.090f * ((int)time);
        // 重置当前的模型观察矩阵
        gl.glLoadIdentity();
        // 移动绘图原点的坐标与上面的语句连用就相当于设置新的绘图远点坐标，
        //gl.glFrustumf(-ratio, ratio, -1, 1, 1, 10);//后面的1-10是指图像的1-10层，
        // 图像所处层次越大，在屏幕上显示就越小。默认为（0，0，1),
        // 左移 1.5 单位，并移入屏幕 6.0。
        gl.glTranslatef(0f, 0.0f, -5.0f);

        gl.glRotatef(mAngle, 0.0f, 0.0f, 1.0f);
        //启用平滑着色
        gl.glEnableClientState(GL10.GL_COLOR_ARRAY);//
        //gl.glColor4f(1.0f, 0.0f, 0.0f, 1.0f);//可以直接设置绘图的单调颜色
        // 设置三角形点
        // gl.glVertexPointer(3, GL10.GL_FIXED, 0, triggerBuffer);
        gl.glVertexPointer(3,GL10.GL_FLOAT,0,triggerBuffer2);
        //设置平滑着色的颜色矩阵
        gl.glColorPointer(4,GL10.GL_FLOAT,0,colorBuffer2);//都是一维矩阵，因此第一个参数就是表示一个颜色的长度表示
        //绘制
        gl.glDrawArrays(GL10.GL_TRIANGLES, 0, 3);
        // 关闭颜色平滑着色设置
        gl.glDisableClientState(GL10.GL_COLOR_ARRAY);
        //gl.glFinish();

    }
}
```
在MainActivity中直接调用setRenderer即可
```java
    mGLSurfaceView = (GLSurfaceView) findViewById(R.id.glsv_main);
    mGLSurfaceView.setRenderer(new SunnyGLRender());
```



# 参考
[GLSurfaceView.Renderer](https://developer.android.com/reference/android/opengl/GLSurfaceView.Renderer.html)  
[SurfaceView入门学习](http://www.jianshu.com/p/15060fc9ef18)
