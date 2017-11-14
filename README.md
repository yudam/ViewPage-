# ViewPage-
ViewPage源码阅读  
ViewPage基本上每一个应用中都是必不可少的，主要适用于主页面的切换，轮播图，以及水平的滑动页面切换效果等。因为ViewPage内部帮我们实现了滑动冲突的处理，所以在开发时，我们可以忽略这个问题，但是本着知其所以然的态度，我还是花了三天的时间，好好梳理了一下ViewPage的大部分源码，因此写下笔记，记录一下体会：
首先看一下ViewPage的简单使用
    
    viewpage.setAdapter(new VpAdpter(list));
这里的VpAdpter是我创建的一个适配器继承自PageAdapter，可以来说仅仅一句话就可以实现页面的切换效果。接下来就根据这句代码来分析ViewPage的源码，首先从  
setAdapter方法开始：
    
    public void setAdapter(MPageAdapter adapter) {
        //若ViewPage之前存在adapter，清除之前的adapter
        if (mAdapter != null) {
            //清除观察者
            mAdapter.setViewPagerObserver(null);
            //通知适配器更新
            mAdapter.startUpdate(this);
            //销毁所有保存的View
            for (int i = 0; i < mItems.size(); i++) {
                final ItemInfo ii = mItems.get(i);
                mAdapter.destroyItem(this, ii.position, ii.object);
            }
            //停止更新
            mAdapter.finishUpdate(this);
            mItems.clear();
            //销毁非decordView
            removeNonDecorViews();
            //重置为0
            mCurItem = 0;
            //滑动到起始点
            scrollTo(0, 0);
        }

        final MPageAdapter oldAdapter = mAdapter;

        //使用当前adapter
        mAdapter = adapter;
        //初始化期待页卡数
        mExpectedAdapterCount = 0;

        if (mAdapter != null) {
            //创建一个观察者对象
            if (mObserver == null) {
                mObserver = new PagerObserver();
            }
            //mAdapter中添加观察者
            mAdapter.setViewPagerObserver(mObserver);
            mPopulatePending = false;
            //保存是否是第一次layout
            final boolean wasFirstLayout = mFirstLayout;
            mFirstLayout = true;
            mExpectedAdapterCount = mAdapter.getCount();

            Log.i("mao","mRestoredCurItem="+mRestoredCurItem);
            Log.i("mao","mFirstLayout="+mFirstLayout);
            //有数据需要回复
            if (mRestoredCurItem >= 0) {
                mAdapter.restoreState(mRestoredAdapterState, mRestoredClassLoader);
                setCurrentItemInternal(mRestoredCurItem, false, true);
                //标记回复结束，无需回复
                mRestoredCurItem = -1;
                mRestoredAdapterState = null;
                mRestoredClassLoader = null;
            } else if (!wasFirstLayout) {

                //Viewpage默认缓存指定页面，所以要进行页面的销毁与重建，调用populate方法
                populate();
            } else {
                //重新布局
                requestLayout();
            }
        }

        // Dispatch the change to any listeners
        if (mAdapterChangeListeners != null && !mAdapterChangeListeners.isEmpty()) {
            for (int i = 0, count = mAdapterChangeListeners.size(); i < count; i++) {
                mAdapterChangeListeners.get(i).onAdapterChanged(this, oldAdapter, adapter);
            }
        }
    }
可以看出首先会判断mAdapter是否为null，若为null，则会销毁所有的缓存View(除Adapter中View之外的View)，后重置所有ViewPage绘制条件。接下来会新建一个mObserver传递给mAdapter，用于观察数据的改变。若当前属于第一次layout，则调用populate来销毁与重构页面，mAdapterChangeListeners用于检测适配器的状况。  
    private void removeNonDecorViews() {
        for (int i = 0; i < getChildCount(); i++) {
            final View child = getChildAt(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            if (!lp.isDecor) {
                removeViewAt(i);
                i--;
            }
        }
    }
