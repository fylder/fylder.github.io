---
published: true
layout: post
title: RecyclerView刷新与加载
category: android
tags: 
 - recyclerview
time: 2016.5.12 20:14:07
excerpt: 自定义RecyclerView支持刷新和加载更多，大鱼吃小鱼。

---

RecyclerView控件从前年开始发布，google发布此控件就是为了代替过去ListView的列表显示，从回收的效率到
复用到视图与数据容器的设计都比以往要好；
<br>
在使用的过程发现，官方有SwipeRefreshLayout支持RecyclerView的刷新，但在加载更多的情况没有接口调用，于是就想DIY一个自定义RecyclerView扩展支持上拉加载更多。

### DIY

<p>

#### FylderRecyclerView
继承于ReclcyerView，扩展添加划至最后一项触发回调onLoading()事件

```java
package fylder.recycler.demo.view;

import android.content.Context;
import android.support.annotation.Nullable;
import android.support.v7.widget.LinearLayoutManager;
import android.support.v7.widget.RecyclerView;
import android.util.AttributeSet;
import android.util.Log;
import android.view.GestureDetector;
import android.view.MotionEvent;
import android.view.View;

/**
 * 添加loading监听
 * <p/>
 * 添加手势滑动只能在向上滑触发
 * <p/>
 * Created by 剑指锁妖塔 on 15-11-24.
 */
public class FylderRecyclerView extends RecyclerView {

    boolean isDown = false; //手势是否下滑
    private LinearLayoutManager mLayoutManager;
    private GestureDetector mGestureDetector;

    boolean isLoading = false; //判断是否正在刷新或加载

    public FylderRecyclerView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public void setLayoutManager(LayoutManager layout) {
        super.setLayoutManager(layout);
        if (layout instanceof LinearLayoutManager)
            this.mLayoutManager = (LinearLayoutManager) layout;
    }

    public boolean isLoading() {
        return isLoading;
    }

    public void setLoading(boolean loading) {
        isLoading = loading;
    }

    /**
     * 设置监听事件
     */
    public void setLoadingListener(final LoadingListener listener) {

        this.addOnScrollListener(new OnScrollListener() {
            @Override
            public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                super.onScrollStateChanged(recyclerView, newState);
                int visibleItemCount = mLayoutManager.getChildCount();
                int lastVisibleItem = mLayoutManager.findLastVisibleItemPosition();
                int totalItemCount = mLayoutManager.getItemCount();
                if (visibleItemCount > 1 && newState == RecyclerView.SCROLL_STATE_IDLE && lastVisibleItem >= totalItemCount - 1) {
                    if (!isDown && !isLoading()) {
                        setLoading(true);
                        listener.onLoading();
                    } else {
                        Log.w("123", "onLoading");
                    }
                }
            }
        });

        mGestureDetector = new GestureDetector(new GestureDetector.OnGestureListener() {

            @Override
            public boolean onDown(MotionEvent e) {
                return false;
            }

            @Override
            public void onShowPress(MotionEvent e) {

            }

            @Override
            public boolean onSingleTapUp(MotionEvent e) {
                return false;
            }

            @Override
            public boolean onScroll(MotionEvent e1, MotionEvent e2, float distanceX, float distanceY) {
                //distanceY为负数则是向下
                if (distanceY < 0) {
                    isDown = true;
                } else {
                    isDown = false;
                }
                return false;
            }

            @Override
            public void onLongPress(MotionEvent e) {

            }

            @Override
            public boolean onFling(MotionEvent e1, MotionEvent e2, float velocityX, float velocityY) {

                return false;
            }
        });

        this.setOnTouchListener(new OnTouchListener() {
            @Override
            public boolean onTouch(View v, MotionEvent event) {
                return mGestureDetector.onTouchEvent(event);
            }
        });
    }

    public interface LoadingListener {

        /**
         * 触发加载更多
         */
        void onLoading();
    }
}

```

#### BaseRecyclerAdapter
基类数据适配器的封装用一个抽象类，在使用这个自定义控件的adapter必须继承于此，目前首部head-lay不需要考虑，主要用于封装尾部footer-lay，至于如何显示有四种不同的情况由STATS判定。

```java
package fylder.recycler.demo.adapter;

import android.content.Context;
import android.support.v7.widget.RecyclerView;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;
import android.widget.ProgressBar;
import android.widget.RelativeLayout;
import android.widget.TextView;

import butterknife.BindView;
import butterknife.ButterKnife;
import fylder.recycler.demo.R;
import fylder.recycler.demo.tools.AnimTools;


/**
 * Created by 剑指锁妖塔 on 16-4-29.
 */
public abstract class BaseRecyclerAdapter<T> extends RecyclerView.Adapter {

    private static final int TYPE_FOOTER_VIEW = 100;//尾部布局类型
    private int extraCount = 1;//额外多出来的

    protected final int STATS_EMPTY = 1;      //  空白
    protected final int STATS_LOADING = 2;    //  加载
    protected final int STATS_LOADED = 3;     //  加载完了
    protected final int STATS_END = 4;        //  到最后一条

    Context context;

    protected int STATS = STATS_EMPTY;

    public BaseRecyclerAdapter(Context context) {
        this.context = context;
    }

    public void refresh(T t) {
        STATS = STATS_EMPTY;
        notifyDataSetChanged();
    }

    public void addListData(T t) {
        STATS = STATS_LOADED;
        notifyDataSetChanged();
    }

    /**
     * 正在加载
     */
    public void loading() {
        STATS = STATS_LOADING;
        notifyDataSetChanged();
    }

    /**
     * 最后一条
     */
    public void ending() {
        STATS = STATS_END;
        notifyDataSetChanged();
    }

    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType) {

        switch (viewType) {
            case TYPE_FOOTER_VIEW:
                View footerView = LayoutInflater.from(context).inflate(R.layout.footer_lay, parent, false);
                return new FooterViewHolder(footerView);
            default:
                return createExcludeViewHolder(parent, viewType);
        }
    }

    @Override
    public void onBindViewHolder(RecyclerView.ViewHolder holder, int position) {

        if (getItemViewType(position) == TYPE_FOOTER_VIEW) {

            FooterViewHolder footerViewHolder = (FooterViewHolder) holder;

            if (STATS == STATS_LOADING) {
                //正在加载中
                footerViewHolder.lay.setVisibility(View.VISIBLE);
                footerViewHolder.loadingText.setVisibility(View.GONE);
                footerViewHolder.loadingPro.setVisibility(View.VISIBLE);
                footerViewHolder.loadingText.setText(R.string.recycler_loading);
                AnimTools.show(footerViewHolder.lay);
            } else if (STATS == STATS_LOADED) {
                footerViewHolder.lay.setVisibility(View.GONE);
            } else if (STATS == STATS_END) {
                //加载结束后
                footerViewHolder.lay.setVisibility(View.VISIBLE);
                footerViewHolder.loadingText.setVisibility(View.VISIBLE);
                footerViewHolder.loadingPro.setVisibility(View.GONE);
                footerViewHolder.loadingText.setText(R.string.recycler_load_end);
                AnimTools.show(footerViewHolder.lay);
            } else {
                //其余情况隐藏
                footerViewHolder.lay.setVisibility(View.GONE);
            }
        } else {
            onBindView(holder, position);
        }
    }

    /**
     * 获取该type的ViewHolder
     *
     * @param viewType
     * @return
     */
    public abstract RecyclerView.ViewHolder createExcludeViewHolder(ViewGroup parent, int viewType);

    @Override
    public int getItemCount() {
        return getExcludeItemCount() + extraCount;
    }

    @Override
    public int getItemViewType(int innerPosition) {
        if (getItemCount() - 1 == innerPosition) { // footer
            return TYPE_FOOTER_VIEW;
        } else {
            return getExcludeItemViewType(innerPosition);
        }
    }

    /**
     * 绑定数据
     *
     * @param holder
     */
    public abstract void onBindView(RecyclerView.ViewHolder holder, int position);

    /**
     * （不包括headerView和footerView）
     *
     * @return 获取item的数量
     */
    public abstract int getExcludeItemCount();

    /**
     * 通过realItemPosition得到该item的类型（不包括headerView和footerView）
     *
     * @param realItemPosition 位置
     * @return 得到该item的类型
     */
    public abstract int getExcludeItemViewType(int realItemPosition);

    class FooterViewHolder extends RecyclerView.ViewHolder {

        @BindView(R.id.footer_lay)
        RelativeLayout lay;
        @BindView(R.id.footer_loading)
        ProgressBar loadingPro;
        @BindView(R.id.footer_text)
        TextView loadingText;

        public FooterViewHolder(View itemView) {
            super(itemView);
            ButterKnife.bind(this, itemView);
        }
    }
}
```

### 如何使用 

<p>

#### DemoAdapter
只需要写显示的布局绑定，最后一项布局已在继承的BaseRecyclerAdapter，不需要考虑。

```java
package fylder.recycler.demo.adapter;

import android.content.Context;
import android.support.v7.widget.RecyclerView;
import android.view.LayoutInflater;
import android.view.View;
import android.view.ViewGroup;

import butterknife.ButterKnife;
import fylder.recycler.demo.R;

/**
 * Created by 剑指锁妖塔 on 2016/4/29.
 */
public class DemoAdapter extends BaseRecyclerAdapter<Integer> {

    LayoutInflater inflater;

    int c;

    public DemoAdapter(Context context) {
        super(context);
        inflater = LayoutInflater.from(context);
        c = 0;
    }

    @Override
    public void refresh(Integer integer) {
        this.c = integer;
        super.refresh(integer);
    }

    @Override
    public void addListData(Integer integer) {
        this.c += integer;
        super.addListData(integer);
    }


    @Override
    public RecyclerView.ViewHolder createExcludeViewHolder(ViewGroup parent, int viewType) {
        View view = inflater.inflate(R.layout.item_demo, parent, false);
        return new DemoViewHolder(view);
    }

    @Override
    public void onBindView(RecyclerView.ViewHolder holder, int position) {

    }

    @Override
    public int getExcludeItemCount() {
        return c;
    }

    @Override
    public int getExcludeItemViewType(int realItemPosition) {
        return 0;
    }

    class DemoViewHolder extends RecyclerView.ViewHolder {

        public DemoViewHolder(View itemView) {
            super(itemView);
            ButterKnife.bind(this, itemView);
        }
    }
}
```

#### DemoActivity

```java
package fylder.recycler.demo;

import android.content.Context;
import android.content.Intent;
import android.os.Handler;
import android.os.Looper;
import android.support.design.widget.Snackbar;
import android.support.v4.widget.SwipeRefreshLayout;
import android.support.v7.app.AppCompatActivity;
import android.os.Bundle;
import android.support.v7.widget.LinearLayoutManager;
import android.view.Menu;
import android.view.MenuItem;

import butterknife.BindView;
import butterknife.ButterKnife;
import fylder.recycler.demo.adapter.DemoAdapter;
import fylder.recycler.demo.adapter.FishAdapter;
import fylder.recycler.demo.view.FylderRecyclerView;

public class DemoActivity extends AppCompatActivity implements SwipeRefreshLayout.OnRefreshListener, FylderRecyclerView.LoadingListener {

    @BindView(R.id.demo_refresh)
    SwipeRefreshLayout refreshLayout;
    @BindView(R.id.demo_recycler)
    FylderRecyclerView recyclerView;

    LinearLayoutManager manager;
    DemoAdapter mAdapter;

    Context mContext;
    Handler mHandler = new Handler(Looper.getMainLooper());

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_demo);
        ButterKnife.bind(this);
        mContext = this;
        init();
    }


    void init() {
        manager = new LinearLayoutManager(mContext);
        recyclerView.setLayoutManager(manager);
        mAdapter = new DemoAdapter(mContext);
        recyclerView.setAdapter(mAdapter);
        refreshLayout.setOnRefreshListener(this);   //刷新监听
        recyclerView.setLoadingListener(this);      //加载监听

        refreshLayout.setColorSchemeResources(R.color.color1, R.color.color2,
                R.color.color3, R.color.color4);

        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                refreshLayout.setRefreshing(true);
                refresh();
            }
        }, 100);
    }

    /**
     * 刷新回调
     */
    @Override
    public void onRefresh() {
        refresh();
    }

    /**
     * 加载回调
     */
    @Override
    public void onLoading() {
        mAdapter.loading();
        mHandler.postDelayed(new Runnable() {
            @Override
            public void run() {
                refreshLayout.setRefreshing(false);
                recyclerView.setLoading(false);

                if (mAdapter.getItemCount() > 21) {
                    mAdapter.ending();
                } else {
                    mAdapter.addListData(7);
                }
            }
        }, 2000);
    }

    void refresh() {
        if (!recyclerView.isLoading()) {
            recyclerView.setLoading(true);
            mHandler.postDelayed(new Runnable() {
                @Override
                public void run() {
                    refreshLayout.setRefreshing(false);
                    recyclerView.setLoading(false);
                    mAdapter.refresh(7);
                }
            }, 1000);
        }
    }
}
```

<p>
为了支持网格列表GridLayoutManager，需要重写动态分配SpanSize的值，否则在加载的显示栅格占位不合理。


```java
/**
 * 动态分配span size
 * Created by 剑指锁妖塔 on 2016/4/29.
 */
public class MoreLayoutManager extends GridLayoutManager {

    public MoreLayoutManager(Context context, int spanCount) {
        super(context, spanCount);
        init(spanCount);
    }

    void init(final int spanCount) {

        //分配最后一个享有span size=2
        setSpanSizeLookup(new GridLayoutManager.SpanSizeLookup() {
            @Override
            public int getSpanSize(int position) {

                if (position == getItemCount() - 1)
                    return spanCount;
                else {
                    return 1;
                }
            }
        });
    }
}
```

### 大鱼吃小鱼
有次无意中装了个 **想去** app，发现里面有个加载动画的鲨鱼追小鱼挺有趣，就想做一个，然后花点时间弄了个自定义FishView，里面的动画轨迹用到赛贝尔曲线，我已将这个效果加入加载的RecyclerView的Demo里，详细源代码在Github [RecyclerRefresh](https://github.com/fylder/RecyclerRefresh){:target="_blank"}
