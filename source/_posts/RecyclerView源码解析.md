---
title: RecyclerView解析与Tangram框架
date: 2019-07-02 10:36:07
tags: [View]  
categories: Android
---
RecyclerView解析与Tangram框架
<!-- more -->  



平常对RecyclerView的使用，代码如下，以及sdk中对应方法的注释，(学习sdk看源码注释是最容易的方法了)

```java
 @Override
 protected void onCreate(Bundle savedInstanceState) {
 	recyclerView.setAdapter(myAdaoter);
 	recyclerView.setLayoutManager(new LinearLayoutManager(this));
 }


public class MyAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {
  
  	/**
   	当RV需要一个新的 表示给定vewiType的item的ViewHolder时  就调用这个方法
   	考虑到ViewHolder会被re-used用来显示不同的items 缓存sub views of View 可以避免过多的fbi调用
    */
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {
        return null;
    }
		/**
   	RV调用这个方法来展示给定postion上的数据  这个方法通常需要更新ViewHolder中的itemView
   	与ListView不同的是，如果当前item在数据集中的postion发生改变，RV是不会再调用这个方法的，除非item本身无  效了或者 new position cannot be determined (???)  
   	如果需要postion，通过ViewHolder#getAdapterPosition()来拿
    */
    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {

    }

    @Override
    public int getItemCount() {
        return 0;
    }
}

```

想要分析一个View，基本上按`measure` `layout` `draw`分析 基本上就没问题了

# Measure

RecyclerView 是 对逐个子view 依次 measure、layout   而不是全部measure完再layout

```java
	public LinearLayoutManager() {
        setOrientation(orientation);
        setReverseLayout(reverseLayout);
        setAutoMeasureEnabled(true); //mAutoMeasure 被设置为true
    }

@Override
protected void onMeasure(int widthSpec, int heightSpec) {
  // LayoutManager mLayout;  分析这里就用 LinearLayoutManager
    if (mLayout == null) {
				//省略
    }
    if (mLayout.mAutoMeasure) {//默认true
       			final int widthMode = MeasureSpec.getMode(widthSpec);
            final int heightMode = MeasureSpec.getMode(heightSpec);
      			// 如果 都是EXACTLY的话  就跳出measure 交给layout方法来做measure [注意看这里]
            final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
                    && heightMode == MeasureSpec.EXACTLY;
      			//委托出去  ① Measure
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            if (skipMeasure || mAdapter == null) {
                return;
            }
            if (mState.mLayoutStep == State.STEP_START) {
                 //② Layout
                dispatchLayoutStep1();
            }
            // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
            // consistency
            mLayout.setMeasureSpecs(widthSpec, heightSpec);
            mState.mIsMeasuring = true;
				      //② Layout
            dispatchLayoutStep2();

            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

            // if RecyclerView has non-exact width and height and if there is at least one child
            // which also has non-exact width & height, we have to re-measure.
            if (mLayout.shouldMeasureTwice()) {
                mLayout.setMeasureSpecs(
                        MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                        MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
                mState.mIsMeasuring = true;
                dispatchLayoutStep2();
                // now we can get the width and height from the children.
                mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
            }
    } else {
       //省略
    }
}
```



## mLayout.onMeasure

```java
public void onMeasure(Recycler recycler, State state, int widthSpec, int heightSpec) {
            mRecyclerView.defaultOnMeasure(widthSpec, heightSpec);
        }
void defaultOnMeasure(int widthSpec, int heightSpec) {
        // 平平无奇的 根据mode和size来返回 w h  然后setMeasuredDimension
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

        setMeasuredDimension(width, height);
    }


```

## dispatchLayoutStep1

```java
/**
 * The first step of a layout where we;
 * - process adapter updates
 * - decide which animation should run
 * - save information about current views
 * - If necessary, run predictive layout and save its information 如有必要 执行layout 并保存信息
 */
private void dispatchLayoutStep1() {
    mState.assertLayoutStep(State.STEP_START);
    mState.mIsMeasuring = false;
    //一堆跟动画有关的操作  记住改变状态
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

## dispatchLayoutStep2

```java
/**
 实际进行layout操作的步骤 可能会被调用多次
     */
private void dispatchLayoutStep2() {
  	//状态判断
    mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
    mState.mItemCount = mAdapter.getItemCount();
    mState.mDeletedInvisibleItemCountSincePreviousLayout = 0;

    // Step 2: Run layout
    mState.mInPreLayout = false;
    mLayout.onLayoutChildren(mRecycler, mState);

    mState.mStructureChanged = false;
    mPendingSavedState = null;

    // onLayoutChildren may have caused client code to disable item animations; re-check
    mState.mRunSimpleAnimations = mState.mRunSimpleAnimations && mItemAnimator != null;
    mState.mLayoutStep = State.STEP_ANIMATIONS;
    onExitLayoutOrScroll();
    resumeRequestLayout(false);
}
```

## mLayout.onLayoutChildren

```java
//LinearLayoutManager
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {
    // layout algorithm:
    // 1) by checking children and other variables, find an anchor coordinate and an anchor
    //  item position.
    // 2) fill towards start, stacking from bottom
    // 3) fill towards end, stacking from top
    // 4) scroll to fulfill requirements like stack from bottom.
    // 1 检查child 和 其他变量，确定 锚点坐标与锚点项的positon
    // 2 从下往上填充
    // 3 从上往下填充
    // 4 滚动已满足 步骤2、3 的要求
     
    if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION
                  || mPendingSavedState != null) {
              mAnchorInfo.reset();
              mAnchorInfo.mLayoutFromEnd = mShouldReverseLayout ^ mStackFromEnd;
              // calculate anchor position and coordinate
              //更新锚点信息
              updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
              mAnchorInfo.mValid = true;
    } else if () {
    }
    onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);
  	//先把所有的child 暂时detach
    detachAndScrapAttachedViews(recycler);
    mLayoutState.mInfinite = resolveIsInfinite();
    mLayoutState.mIsPreLayout = state.isPreLayout();
    if (mAnchorInfo.mLayoutFromEnd) {
        // fill towards start
        updateLayoutStateToFillStart(mAnchorInfo);
        mLayoutState.mExtra = extraForStart;
      	//实际操作还是在fill里
        fill(recycler, mLayoutState, state, false);
        startOffset = mLayoutState.mOffset;
        final int firstElement = mLayoutState.mCurrentPosition;
        if (mLayoutState.mAvailable > 0) {
            extraForEnd += mLayoutState.mAvailable;
        }
        // fill towards end
        updateLayoutStateToFillEnd(mAnchorInfo);
        mLayoutState.mExtra = extraForEnd;
        mLayoutState.mCurrentPosition += mLayoutState.mItemDirection;
        fill(recycler, mLayoutState, state, false);
        endOffset = mLayoutState.mOffset;

        if (mLayoutState.mAvailable > 0) {
            // end could not consume all. add more items towards start
            extraForStart = mLayoutState.mAvailable;
            updateLayoutStateToFillStart(firstElement, startOffset);
            mLayoutState.mExtra = extraForStart;
            fill(recycler, mLayoutState, state, false);
            startOffset = mLayoutState.mOffset;
        }
    } else {
      
    }

}
```

```java
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    // max offset we should set is mFastScroll + available
    final int start = layoutState.mAvailable;
   
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        //result
        layoutChunkResult.resetInternal();
        //layoutChunk
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        //计算已消耗的
        layoutState.mOffset += layoutChunkResult.mConsumed * layoutState.mLayoutDirection;

    }

    return start - layoutState.mAvailable;
}

void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
            LayoutState layoutState, LayoutChunkResult result) {
        View view = layoutState.next(recycler);
  
        LayoutParams params = (LayoutParams) view.getLayoutParams();
        if (layoutState.mScrapList == null) {
            if (mShouldReverseLayout == (layoutState.mLayoutDirection
                    == LayoutState.LAYOUT_START)) {
                addView(view);
            } else {
            		//① addview
                addView(view, 0);
            }
        } else {
           
        }
  			//② measure
        measureChildWithMargins(view, 0, 0);
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
       
  			//③ layout
        layoutDecoratedWithMargins(view, left, top, right, bottom);
 
        result.mFocusable = view.hasFocusable();
    }
```

# Layout

```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    dispatchLayout();
}
void dispatchLayout() {
        mState.mIsMeasuring = false;
  			//如果之前measure的时候 specMode都是EXCATLY的话  会在measure中完成layout 并改变Step值 所以这里会被跳过 直接到 dispatchLayoutStep3()
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // size changed 重新搞一下
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }

/**
     * The final step of the layout where we save the information about views for animations,
     * trigger animations and do any necessary cleanup.
     动画相关  忽略
     */
    private void dispatchLayoutStep3() {}
```





# Tangram

## HelloTangram

根据[官方给的接入指南](http://tangram.pingguohe.net/docs/android/develop-component)，写了个最简单的demo，封装了基础的Activity，建议先过一遍官方demo。

**自定义的view**

```java
public class TestView1 extends FrameLayout implements ITangramViewLifeCycle {
    private TextView textView;

    public TestView1(Context context) {
        super(context);
        init();
    }

    public TestView1(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public TestView1(Context context, AttributeSet attrs, int defStyleAttr) {
        super(context, attrs, defStyleAttr);
        init();
    }

    private void init() {
        inflate(getContext(), R.layout.item, this);
        textView = findViewById(R.id.title);
    }

    @Override
    public void cellInited(BaseCell cell) {

    }

    @Override
    public void postBindView(BaseCell cell) {
        //在绑定数据时，从json中拿具体数据
        textView.setText((String) cell.optParam("msg"));
    }

    @Override
    public void postUnBindView(BaseCell cell) {

    }
}

```

**Activity**

```java
public class L1Activity extends CommonActivity {
    public static final String TYPE_TEST_VIEW_1 = "TestView1";


    @Override
    public void doBuilderRegister(TangramBuilder.InnerBuilder builder) {
		//需要注册自定义的View 否则解析的时候找不到
        builder.registerCell(TYPE_TEST_VIEW_1, TestView1.class);

    }
    @Override
    public void doEngineRegister(TangramEngine engine) {

    }

    @Override
    public String getDataName() {
        return "data_1.json";
    }
}
```

```java
/**
 * @CreateDate: 2018/12/26 下午3:30
 * @Author: Lewis Weng
 * @Description: 封装的base类，让所有的使用Tangram的子类都继承这个，
 * 使用同样的layout，不同的行为定义为抽象方法交给子类。
 */
public abstract class CommonActivity extends AppCompatActivity {
    RecyclerView recyclerView;
    TangramBuilder.InnerBuilder builder;
    TangramEngine engine;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(getLayoutId());
        recyclerView = findViewById(R.id.rv_main);

        initView();
        //初始化 TangramBuilder
        builder = TangramBuilder.newInnerBuilder(this);
        doBuilderRegister(builder);

        //构造engine
        engine = builder.build();
        doEngineRegister(engine);

        //绑定
        engine.bindView(recyclerView);

        String json = new String(AssetUtil.getAssertsFile(this, getDataName()));
        JSONArray data = null;
        try {
            data = new JSONArray(json);
            engine.setData(data);
        } catch (JSONException e) {
            e.printStackTrace();
        }

    }

    public int getLayoutId() {
        return R.layout.activity_common;
    }

    //给子类提供一个hook点
    public void initView() {
    }

    public RecyclerView getRecyclerView(){
        return recyclerView;
    }

    public TangramEngine getEngine() {
        return engine;
    }

    //抽象出所有builder注册行为交给子类实现
    public abstract void doBuilderRegister(TangramBuilder.InnerBuilder builder);
    //抽象出所有engine注册行为交给子类实现
    public abstract void doEngineRegister(TangramEngine engine);
    //简单的返回每个子类需要解析的Json文件
    public abstract String getDataName();

    @Override
    protected void onDestroy() {
        super.onDestroy();
        if (engine != null) {
            engine.destroy();
        }
    }
}
```

**json文件:data_1.json**

```json
[
  {
    "type": "TestView1",
    "msg": "Hello Tangram"
  }
]
```

## Builder

逐行跟进，先看看`TangramBuilder.newInnerBuilder(context)`做了什么，以及这个builder是干嘛的。

```java
public static InnerBuilder newInnerBuilder(@NonNull final Context context) {
    if (!TangramBuilder.isInitialized()) {
        throw new IllegalStateException("Tangram must be init first");
    }

    DefaultResolverRegistry registry = new DefaultResolverRegistry();

    // install default cards & mCells
    installDefaultRegistry(registry);

    return new InnerBuilder(context, registry);
}
```

### DefaultResolverRegistry

看命名就知道是个负责分解注册的集合类。几个成员如下，先有个印象，后续用到再来分析

```java
public class DefaultResolverRegistry {
    final CardResolver mDefaultCardResolver = new CardResolver();
    final BaseCellBinderResolver mDefaultCellBinderResolver;
    final BaseCardBinderResolver mDefaultCardBinderResolver;
    ConcurrentHashMap<String, ViewHolderCreator> viewHolderMap;
    MVHelper mMVHelper;
    //...
}
```

### installDefaultRegistry

负责初始化一些built-in的card和cell类型

```java
public static void installDefaultRegistry(@NonNull final DefaultResolverRegistry registry) {
    MVHelper mvHelper = new MVHelper(new MVResolver());
    registry.setMVHelper(mvHelper);
    // built-in mCells
    registry.registerCell(TYPE_CONTAINER_SCROLL, LinearScrollView.class);
    // built-in cards
    registry.registerCard(TYPE_CAROUSEL_COMPACT, BannerCard.class);
    // 省略一堆 registerCell 和 registerCard
}
```

既然这里用到了，就跟进看看MVResolver和MVHelper是啥吧。

```java
public class MVResolver {
    private ConcurrentHashMap<String, Class<? extends View>> typeViewMap = new ConcurrentHashMap<>(64);

    private ConcurrentHashMap<String, Class<? extends BaseCell>> typeCellMap = new ConcurrentHashMap(64);

    private ConcurrentHashMap<String, Card> idCardMap = new ConcurrentHashMap<>();

    private ConcurrentHashMap<BaseCell, View> mvMap = new ConcurrentHashMap<>(128);

    private ConcurrentHashMap<View, BaseCell> vmMap = new ConcurrentHashMap<>(128);

    private ServiceManager mServiceManager;
}
```

根据成员和方法，不难看出是个维护 model-view 映射关系的类，内置了一些默认的json文件的key值，如type、style和id等等。

```java
public class MVHelper {
    private static final String TAG = "Tangram-MVHelper";

    private MVResolver mvResolver; 

    private VafContext mVafContext; //Tangram依赖的VirtualView框架的上下文 暂不管

    private ConcurrentHashMap<BaseCell, ConcurrentHashMap<Method, Object>> methodMap = new ConcurrentHashMap<>(128);
    private ConcurrentHashMap<Class, Method[]> methodCacheMap = new ConcurrentHashMap<>(128);
    private ConcurrentHashMap<BaseCell, Method> postBindMap = new ConcurrentHashMap<>(128);
    private ConcurrentHashMap<BaseCell, Method> postUnBindMap = new ConcurrentHashMap<>(128);
    private ConcurrentHashMap<BaseCell, Method> cellInitedMap = new ConcurrentHashMap<>(128);
}
```

MVHelper就是对MVResolver的封装，是个helper类嘛。让用户有了在MVResolver建立model-view映射关系时，hook的能力，比如通过`cell.serviceManager.getService`来通知用户自定义的Support类（见mountView方法）。

### DefaultResolverRegistry.registerCell



```java
public <V extends View> void registerCell(String type, final @NonNull Class<V> viewClz) {
    if (viewHolderMap.get(type) == null) {
        mDefaultCellBinderResolver.register(type, new BaseCellBinder<>(viewClz, mMVHelper));
    } else {
        mDefaultCellBinderResolver.register(type, new BaseCellBinder<ViewHolderCreator.ViewHolder, V>(viewHolderMap.get(type),
            mMVHelper));
    }
    mMVHelper.resolver().register(type, viewClz);
}
```

viewHolderMap存的是用户传入的自定义ViewHolder，一般没有，上述方法最后做的就是把type和viewClz存到对应的map里

```java
mMap.put(gen, type);//BaseReslover.class
mSparseArray.put(type, gen);//BaseReslover.class
typeViewMap.put(type, viewClazz); //MVReslover.class
```

registerCard同理。

### InnerBuilder

分析完上面几个类，再回归正题，InnerBuilder这个类。

```java
protected InnerBuilder(@NonNull final Context context, DefaultResolverRegistry registry) {
    this.mContext = context;
    this.mDefaultResolverRegistry = registry;
    mMVHelper = registry.getMVHelper();
    mMVResolver = mMVHelper.resolver();
    mPojoAdapterBuilder = new PojoAdapterBuilder();
    mDataParser = new PojoDataParser();
}
```

可以看做一个代理类了，封装了上面几个成员的各种方法。用到再说。

## Engine

进入到InnerBuilder的build方法看下是怎么构造TangramEngine的

```java
public TangramEngine build() {

    TangramEngine tangramEngine =
        new TangramEngine(mContext, mDataParser, mPojoAdapterBuilder);

    tangramEngine.setPerformanceMonitor(mPerformanceMonitor);
    // register service with default services
    tangramEngine.register(MVHelper.class, mMVHelper);
    tangramEngine
        .register(CardResolver.class, mDefaultResolverRegistry.mDefaultCardResolver);
    tangramEngine.register(BaseCellBinderResolver.class,
        mDefaultResolverRegistry.mDefaultCellBinderResolver);
    tangramEngine.register(BaseCardBinderResolver.class,
        mDefaultResolverRegistry.mDefaultCardBinderResolver);

    // add other features service
    tangramEngine.register(TimerSupport.class, new TimerSupport());
    tangramEngine.register(BusSupport.class, new BusSupport());

    // add virtual view context
    VafContext mVafContext = new VafContext(mContext.getApplicationContext());
    ViewManager mViewManager = mVafContext.getViewManager();
    mViewManager.init(mContext.getApplicationContext());
    tangramEngine.register(ViewManager.class, mViewManager);
    tangramEngine.register(VafContext.class, mVafContext);
    mMVHelper.setVafContext(mVafContext);

    mMVResolver.setServiceManager(tangramEngine);

    if(callback != null){
        callback.onBuild(tangramEngine);
    }
    return tangramEngine;
}
```

### TangramEngine

```java
public TangramEngine(/*...*/) {
    super(context, dataParser, adapterBuilder);
    this.register(DataParser.class, dataParser);
}
```

```java
public BaseTangramEngine(final Context context,
     final DataParser<O, T, C, L> dataParser,
     final IAdapterBuilder<C, L> adapterBuilder) {

    this.mContext = context;
    this.mLayoutManager = new VirtualLayoutManager(mContext);
    this.mLayoutManager.setLayoutViewFactory(new LayoutViewFactory() {
        @Override
        public View generateLayoutView(@NonNull Context context) {
            ImageView imageView = ImageUtils.createImageInstance(context);
            return imageView != null ? imageView : new View(context);
        }
    });
 
}
```

看看那Base类的成员

```java
//跟上面Registry一样的class与obj的键值对，保存注册的service
private ConcurrentHashMap<Class<?>, Object> mServices = new ConcurrentHashMap<>();

@NonNull
private final Context mContext;
//tangram核心 recyclerView
private RecyclerView mContentView;
//v-layout相关
private final VirtualLayoutManager mLayoutManager;

protected GroupBasicAdapter<C, L> mGroupBasicAdapter;

private final DataParser<O, T, C, L> mDataParser;

private final IAdapterBuilder<C, L> mAdapterBuilder;

private PerformanceMonitor mPerformanceMonitor;

```

register方法也就是将clz与obj保存起来，方便后续使用

```java
@Override
public <S> void register(@NonNull Class<S> type, @NonNull S service) {
    mServices.put(type, type.cast(service));
}
```

## setData

demo中，数据流的进入通过TangramEngine的setData方法，有了上面的基础，我们现在跟进去看看。

```java
@Override
public void setData(@Nullable JSONArray data) {
    super.setData(data);
    loadFirstPageCard();//这个不重要 没有自定义loadSupport的时候无视即可
}
```

```java
public void setData(@Nullable T data) {
    //第二个参数传入的serviceManager是this，即engine本身
    List<C> cards = mDataParser.parseGroup(data, this);
    this.setData(cards);
}
```

### parse

由之前的分析，Engine中的mDataParser是由Builder传入的，是在其构造函数中new的PojoDataParser实例。

```java
//PojoDataParser.java
public List<Card> parseGroup(@NonNull JSONArray data, @NonNull final ServiceManager serviceManager) {
    final CardResolver cardResolver = serviceManager.getService(CardResolver.class);
    final MVHelper cellResolver = serviceManager.getService(MVHelper.class);
    final int size = data.length();
    final List<Card> result = new ArrayList<>(size);//json转card的结果保存
    for (int i = 0; i < size; i++) {
        JSONObject cardData = data.optJSONObject(i); //取出每个card数据
        final Card card = parseSingleGroup(cardData, serviceManager);//解析card
        if (card != null) {
            if (card instanceof IDelegateCard) {
                List<Card> cards = ((IDelegateCard) card).getCards(new CardResolver() {
                    @Override
                    public Card create(String type) {
                        Card c = cardResolver.create(type);
                        c.serviceManager = serviceManager;
                        c.id = card.id;
                        c.setStringType(type);
                        c.rowId = card.rowId;
                        return c;
                    }
                });
                for (Card c : cards) {
                    if (c.isValid()) {
                        result.add(c);
                    }
                }
            } else {
                result.add(card);
            }
        }
    }
    cellResolver.resolver().setCards(result);
    return result;
}
```

```java
public Card parseSingleGroup(@Nullable JSONObject data, final ServiceManager serviceManager) {
    final CardResolver cardResolver = serviceManager.getService(CardResolver.class);
    final MVHelper cellResolver = serviceManager.getService(MVHelper.class);
    final String cardType = data.optString(Card.KEY_TYPE);
    if (!TextUtils.isEmpty(cardType)) {
        final Card card = cardResolver.create(cardType);//根据类型创建card
        if (card != null) {
           /*解析数据*/
        } else {
          /*解析数据*/
        }
    } else {
        LogUtils.w(TAG, "Invalid card type when parse JSON data");
    }
    return Card.NaN;
}
```

```java
//CardResolver 父类 ClassResolver
public T create(String type) {
    Class<? extends T> clz = mSparseArray.get(type);
    if (clz != null) {
		return clz.newInstance();
    }
    return null;
}
```

看到没，直接从mSparseArray中取。这个mSparseArray就是前面Builder.registerCell和Card的时候存到BaseResolver里的那个mSparseArray。

解析完所有card之后，调用engine自己的setData方法

```java
public void setData(@Nullable List<C> data) {
    MVHelper mvHelper = (MVHelper) mServices.get(MVHelper.class);
    if (mvHelper != null)
        mvHelper.reset();//reset
    this.mGroupBasicAdapter.setData(data);//设置List<Card>进去，下面load时会用到就是了 
}
```

### load

解析完成，就要加载了。回到上面的setData中的this.setData看看。

```java
public void setData(@Nullable List<C> data) {
        MVHelper mvHelper = (MVHelper) mServices.get(MVHelper.class);
        if (mvHelper != null)
            mvHelper.reset();
    //这里的adpater也是builder传入engine的PojoAdapterBuilder.newAdapter返回值
    //即 PojoGroupBasicAdapter
        this.mGroupBasicAdapter.setData(data);
    }
```

```java
//GroupBasicAdapter.java
public void setData(@Nullable List<L> cards, boolean silence) {
    createSnapshot();

    mCards.clear();
    mData.clear();


    if (cards != null && cards.size() != 0) {
        mCards.ensureCapacity(cards.size());
        setLayoutHelpers(transformCards(cards, mData, mCards)); //跟进,暂时忽略transformCards生成layoutHelper过程
    } else {
        setLayoutHelpers(Collections.<LayoutHelper>emptyList());
    }

    diffWithSnapshot();

    if (!silence)
        notifyDataSetChanged();
}
```



调用setLayoutHelpers 就正式进入Vlayout的世界了

```java
public abstract class VirtualLayoutAdapter<VH extends RecyclerView.ViewHolder> extends RecyclerView.Adapter<VH> {

    @NonNull
    protected VirtualLayoutManager mLayoutManager;

    public VirtualLayoutAdapter(@NonNull VirtualLayoutManager layoutManager) {
        this.mLayoutManager = layoutManager;
    }

    public void setLayoutHelpers(List<LayoutHelper> helpers) {
        this.mLayoutManager.setLayoutHelpers(helpers);
    }

    @NonNull
    public List<LayoutHelper> getLayoutHelpers() {
        return this.mLayoutManager.getLayoutHelpers();
    }

}

```

## V-layout的世界

官方使用

```java
public class MyAdapter extends VirtualLayoutAdapter {
   ......
}

MyAdapter myAdapter = new MyAdapter(layoutManager);

//create layoutHelper list
List<LayoutHelper> helpers = new LinkedList<>();
GridLayoutHelper gridLayoutHelper = new GridLayoutHelper(4);
gridLayoutHelper.setItemCount(25);
helpers.add(gridLayoutHelper);

GridLayoutHelper gridLayoutHelper2 = new GridLayoutHelper(2);
gridLayoutHelper2.setItemCount(25);
helpers.add(gridLayoutHelper2);

//set layoutHelper list to adapter
myAdapter.setLayoutHelpers(helpers);

//set adapter to recyclerView		
recycler.setAdapter(myAdapter);
```

### 从RecyclerView说起

LayoutManager在measure和layout中扮演的角色

Adapter又起到什么作用

Vlayout是如何介入的

### VirtualLayoutManager

VirtualLayoutManager继承自ExposeLinearLayoutManagerEx继承自LinearLayoutManager

```
ExposeLinearLayoutManagerEx is used to expose layoutChunk method
a valid class technically
```

结合RecyclerView和LayoutManager流程，就可以知道VirtualLayout是在layoutChunk中动了手脚，把layout行为分发出去。

```java
@Override
protected void layoutChunk(RecyclerView.Recycler recycler, 
                           RecyclerView.State state, 
                           LayoutState layoutState, 
                           LayoutChunkResult result) {
 
    LayoutHelper layoutHelper = mHelperFinder == null ? null : mHelperFinder.getLayoutHelper(position);
    
    if (layoutHelper == null)
        layoutHelper = mDefaultLayoutHelper;

    layoutHelper.doLayout(recycler, state, mTempLayoutStateWrapper, result, this);
}
```

由此可见，我们是可以自己写LayoutHelper来实现自己的view的layout过程



# 参考

<https://blog.csdn.net/qq_23012315/article/details/50807224>