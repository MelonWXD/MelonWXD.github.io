---
title: Fragment 杂记
date: 2017-11-13 21:10:52
tags: [Fragment]
categories: Android
---
Fragment 使用过程中遇到得坑与使用心得
<!-- more -->

# 实例化
Fragment使用静态单例构造器与setArguments来传递初始化参数
```
public class FragmentOne extends Fragment{
    private TextView textView;
    public View onCreateView(LayoutInflater inflater,
            @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        View view = inflater.inflate(R.layout.fragment_one, null);
        textView = (TextView) view.findViewById(R.id.textview);
        if(getArguments()!=null){
            //取出保存的值
            textView.setText(getArguments().getString("name"));
        }
        return view;
    }
    public static  FragmentOne newInstance(String text){
        FragmentOne fragmentOne = new FragmentOne();
        Bundle bundle = new Bundle();
        bundle.putString("name", text);
        //fragment保存参数，传入一个Bundle对象
        fragmentOne.setArguments(bundle);
        return fragmentOne;
    }
}
```

# ButterKnife的使用
```java
    Unbinder unbinder;
    @Override
    public View onCreateView(LayoutInflater inflater, ViewGroup container,
                             Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        View view= inflater.inflate(R.layout.activity_weather, container, false);
        unbinder = ButterKnife.bind(this,view);
        }
    @Override
    public void onDestroyView() {
        super.onDestroyView();
        unbinder.unbind();
    }    
        
```


# 旋转导致重叠
## 保存数据：onSaveInstanceState
当手机屏幕旋转，或者Activity被回收的时候，系统会调用`onSaveInstanceState(Bundle outState)`来保存视图层（View Hierarchy）数据，参数Bundle是可以传递可序列化数据的类。观察Activity的onCreate的方法参数就明白了`onCreate(@Nullable Bundle savedInstanceState)`。

通过上面的介绍，大致也能理解Fragment的重叠原因也是因为数据的保存。当屏幕旋转的时候，Fragment执行onDetach，但是Activity中依然持有之前实例化的Fragment，在`onSaveInstanceState`中被保存，当Activity重新执行onCreate的时候，这个Fragment又被显示到屏幕上，就造成了屏幕重叠。

## 解决方法

按照上面的介绍，最简单的方法就是重写onSaveInstanceState，函数体置空，直接不保存状态，自然恢复的时候就不会有重叠的情况。
```
    @Override
    protected void onSaveInstanceState(Bundle outState) {
    }
```
但是如果这样的，又与恢复数据的需求背道而驰，假设用户在看视频，总不可能旋转一下，视频又得从头播放吧。  
对于Fragment的话可以在Activity的onAttachFragment的方法中对参数Fragment判断是否是某个具体Fragment的实例，然后直接设置该Fragment，而不用再一次去new一个。
## 参考
[如何向一个Fragment传递参数---setArguments方法的介绍](http://blog.csdn.net/small_lee/article/details/50553881)