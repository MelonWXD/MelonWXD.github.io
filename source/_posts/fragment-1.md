---
title: Fragment 杂记
date: 2017-11-13 21:10:52
tags: [Fragment]
categories: Android
---
Fragment使用静态单例构造器与setArguments来传递参数
<!-- more -->


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



## 参考
[如何向一个Fragment传递参数---setArguments方法的介绍](http://blog.csdn.net/small_lee/article/details/50553881)