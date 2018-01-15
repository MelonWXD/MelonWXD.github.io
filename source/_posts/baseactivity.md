---
title: BaseActivity/BaseToolBarActivity
date: 2017-11-25 16:05:57  
tags: [Base]
categories: Android代码库
---
基类Activity的构建模板
<!-- more -->

## BaseActivity
```java


public abstract class BaseActivity extends AppCompatActivity {

    private Unbinder mUnbinder;

//    protected BasePresenter mPresenter;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutID());

        mUnbinder = ButterKnife.bind(this);

        setTranslucentStatus();

        initBundle(savedInstanceState);
//        initPresenter();
        initView();
        initData();
    }

    protected void initPresenter() {
//        mPresenter = getPresenter();
    }
//
//    protected BasePresenter getPresenter() {
//        return null;
//    }

    protected void initBundle(Bundle savedInstanceState) {

    }

    protected void saveBundle(Bundle outState) {

    }


    protected abstract int getLayoutID();

    @Override
    protected void onPostCreate(@Nullable Bundle savedInstanceState) {
        super.onPostCreate(savedInstanceState);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (mUnbinder != null) {
            mUnbinder.unbind();
        }
    }

    @Override
    protected void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);
        saveBundle(outState);
//        Logger.d("onSaveInstanceState");
    }

    protected void initView() {

    }


    protected void initData() {

    }
}

```

## BaseToolBarActivity
```java

public abstract class BaseToolBarActivity extends BaseActivity {



    private Toolbar mToolBar;
    //根布局
    private LinearLayout mRootView = null;
    //子Activity原布局
    private View mContentView = null;


    /**
     * 根据子类传入的布局ID来获取View，在该View外面增加一个LL和ToolBar再setContentView
     * @param layoutResID 子类重写BaseActivity的getLayoutID
     */
    @Override
    public void setContentView(int layoutResID) {
        mContentView = LayoutInflater.from(this).inflate(layoutResID,null);
        initDectorView();
        super.setContentView(mRootView);
    }


    /**
     * 初始化根布局
     */
    private void initDectorView() {
        mRootView = new LinearLayout(this);
        mRootView.setLayoutParams(new ViewGroup.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT,
                ViewGroup.LayoutParams.MATCH_PARENT));
        mRootView.setOrientation(LinearLayout.VERTICAL);
        initToolBar();
        mRootView.addView(mToolBar);
        mRootView.addView(mContentView);
    }

    /**
     * 初始化ToolBar
     */
    private void initToolBar() {
        mToolBar = (Toolbar) getLayoutInflater().inflate(R.layout.layout_toolbar,mRootView,false);
        setCustomToolbar(mToolBar);
    }

    /**
     * 子类自行扩展设置属性
     * @param toolbar
     */
    protected abstract void setCustomToolbar(Toolbar toolbar);

}

```


## BaseFragment
主要注意有个Bundle和setArguments的用法
```java
public abstract class BaseFragment extends Fragment {

    protected LayoutInflater mInflater;
    protected Context mContext;
    protected View mRoot;
    protected Bundle mBundle;//获取setArguments的值
    protected Unbinder mUnbinder;

    @Override
    public void onAttach(Context context) {
        mContext =context;
        super.onAttach(context);
    }



    @Override
    public void onDetach() {
        mContext = null;
        super.onDetach();

    }

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBundle = getArguments();
        initBundle(mBundle);


    }

    @Override
    public void onSaveInstanceState(Bundle outState) {
        super.onSaveInstanceState(outState);

    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        if(mUnbinder!=null){
            mUnbinder.unbind();
        }


    }

    @Nullable
    @Override
    public View onCreateView(LayoutInflater inflater, @Nullable ViewGroup container, @Nullable Bundle savedInstanceState) {
        mInflater = inflater;
        mRoot = inflater.inflate(getLayoutId(),container,false);
        mUnbinder = ButterKnife.bind(this,mRoot);

        initWidget(mRoot);
        initData();




        return mRoot;
    }




    protected abstract int getLayoutId();

    protected void initBundle(Bundle bundle) {

    }

    protected void initWidget(View root) {


    }

    protected void initData() {


    }
}
```
