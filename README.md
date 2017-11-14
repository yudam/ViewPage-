# ViewPage-
ViewPage源码阅读：可视页面与预缓存页面  
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
    
可以看出首先会判断mAdapter是否为null，若为null，则会销毁所有的缓存View(除Adapter中View之外的View)，后重置所有ViewPage绘制条件。接下来会新建一个mObserver传递给mAdapter，用于观察数据的改变。若当前属于第一次layout，则调用populate来销毁与重构页面，mAdapterChangeListeners用于检测适配器的状况
    
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
    
循环遍历ViewPage的所有child ,清除非页卡的View，也就是Adapter中的View。接下来看一下populate方法，该方法比较长，看了半天一脸萌萌的，
1：ItemInfo保存页面的信息
2：mItems存储需要缓存的页面，即当前展示页和预缓存页面
    
    void populate() {

        populate(mCurItem);
    }

    void populate(int newCurrentItem) {

        //确保当前展示页一定是mCurItem
        ItemInfo oldCurInfo = null;
        if (mCurItem != newCurrentItem) {
            oldCurInfo = infoForPosition(mCurItem);
            mCurItem = newCurrentItem;
        }

        if (mAdapter == null) {
            //对ViewPage中页卡进行排序，有限绘制DecordView
            //然后按照按照Position大小进行绘制
            sortChildDrawingOrder();
            return;
        }

       /**
        * 等待populate，当用户手指滑动到新的position时推迟创建页卡
        * 直到滑动到最终位置，避免错误
        * 当手指滑动到边缘或者手指抬起，View仍在滑动mPopulatePending==true
        */
        if (mPopulatePending) {
            if (DEBUG) Log.i(TAG, "populate is pending, skipping for now...");
            sortChildDrawingOrder();
            return;
        }

       //没连接到Window之前不绘制
        if (getWindowToken() == null) {
            return;
        }
        //mAdapter开始更新页面
        mAdapter.startUpdate(this);
        //startPos<缓存页面数>endPos
        final int pageLimit = mOffscreenPageLimit;
        //确保起始位置>=0
        final int startPos = Math.max(0, mCurItem - pageLimit);
        final int N = mAdapter.getCount();
        //确保结束位置<=N - 1
        final int endPos = Math.min(N - 1, mCurItem + pageLimit);
        //判断用户是否增加了数据源的元素，若增加了没调用notify，报错
        if (N != mExpectedAdapterCount) {
            String resName;
            try {
                resName = getResources().getResourceName(getId());
            } catch (Resources.NotFoundException e) {
                resName = Integer.toHexString(getId());
            }
            throw new IllegalStateException("The application's MPageAdapter changed the adapter's"
                    + " contents without calling MPageAdapter#notifyDataSetChanged!"
                    + " Expected adapter item count: " + mExpectedAdapterCount + ", found: " + N
                    + " Pager id: " + resName
                    + " Pager class: " + getClass()
                    + " Problematic adapter: " + mAdapter.getClass());
        }

        /**
         * 找到当前的项目，或者在需要的时候创建它
         */
        int curIndex = -1;
        ItemInfo curItem = null;
        for (curIndex = 0; curIndex < mItems.size(); curIndex++) {
            final ItemInfo ii = mItems.get(curIndex);
            if (ii.position >= mCurItem) {
                if (ii.position == mCurItem) curItem = ii;
                break;
            }
        }
        //若不存在当前页卡，则说明列表中没有保存，添加该页面到mItems集合中
        if (curItem == null && N > 0) {
            //添加页面到列表尾部
            curItem = addNewItem(mCurItem, curIndex);
        }
        // 默认缓存当前页卡两边的View，若设定了缓存数量
        // 则当前页面两边都缓存指定数量
        // 若没有页面，则啥也不做
        if (curItem != null) {
            float extraWidthLeft = 0.f;
            //取出当前页面左边的页面下标
            int itemIndex = curIndex - 1;
            //存在则取出，不存在设为null
            ItemInfo ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
            final int clientWidth = getClientWidth();
            //算出左边页面需要的宽度，这里的宽度是实际宽度与可视区域宽度的比例
            //即实际宽度=leftWidthNeeded*clientWidth
            final float leftWidthNeeded = clientWidth <= 0 ? 0 :
                    2.f - curItem.widthFactor + (float) getPaddingLeft() / (float) clientWidth;

            //遍历当前页面左边每一个页面
            for (int pos = mCurItem - 1; pos >= 0; pos--) {
                //左边宽度超过了可需宽度，且当前页面比第一个缓存页面位置小，则需要销毁
                if (extraWidthLeft >= leftWidthNeeded && pos < startPos) {
                    //左边没有页面了，跳出循环
                    if (ii == null) {
                        break;
                    }
                    //将当前页面destroy
                    if (pos == ii.position && !ii.scrolling) {
                        mItems.remove(itemIndex);
                        mAdapter.destroyItem(this, pos, ii.object);
                        if (DEBUG) {
                            Log.i(TAG, "populate() - destroyItem() with pos: " + pos
                                    + " view: " + ((View) ii.object));
                        }
                        itemIndex--;
                        curIndex--;
                        ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;
                    }
                } else if (ii != null && pos == ii.position) {
                    //如果当前页面是需要缓存的页面，且页面已存在，则左边页面宽度+当前页面
                    extraWidthLeft += ii.widthFactor;
                    //往左遍历
                    itemIndex--;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;

                } else {
                    //若当前页面需要缓存，且不存在页面，则创建
                    ii = addNewItem(pos, itemIndex + 1);
                    extraWidthLeft += ii.widthFactor;
                    //左边新加一个View，当前可视页面索引号+1
                    curIndex++;
                    ii = itemIndex >= 0 ? mItems.get(itemIndex) : null;

                    Log.i("mao","111extraWidthLeft="+extraWidthLeft);
                }
            }

            //遍历当前页面右边View
            float extraWidthRight = curItem.widthFactor;
            //因为这里使用了curIndex，所以之前的可视化页面索引改变了则必须++
            itemIndex = curIndex + 1;
            if (extraWidthRight < 2.f) {
                ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                final float rightWidthNeeded = clientWidth <= 0 ? 0 :
                        (float) getPaddingRight() / (float) clientWidth + 2.f;
                for (int pos = mCurItem + 1; pos < N; pos++) {
                    //当前页面下标大于endPos且超出需要宽度
                    if (extraWidthRight >= rightWidthNeeded && pos > endPos) {
                        if (ii == null) {
                            break;
                        }
                        if (pos == ii.position && !ii.scrolling) {
                            mItems.remove(itemIndex);
                            mAdapter.destroyItem(this, pos, ii.object);
                            if (DEBUG) {
                                Log.i(TAG, "populate() - destroyItem() with pos: " + pos
                                        + " view: " + ((View) ii.object));
                            }
                            ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                        }
                    } else if (ii != null && pos == ii.position) {
                        extraWidthRight += ii.widthFactor;
                        itemIndex++;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    } else {
                        //因为实在集合右边添加item，之前已经调用itemIndex = curIndex + 1;，所以无需再次调用
                        ii = addNewItem(pos, itemIndex);
                        itemIndex++;
                        extraWidthRight += ii.widthFactor;
                        ii = itemIndex < mItems.size() ? mItems.get(itemIndex) : null;
                    }
                }
            }

            //新老页面之间的计算
            calculatePageOffsets(curItem, curIndex, oldCurInfo);
        }

        if (DEBUG) {
            Log.i(TAG, "Current page list:");
            for (int i = 0; i < mItems.size(); i++) {
                Log.i(TAG, "#" + i + ": page " + mItems.get(i).position);
            }
        }
        //显示当前页面
        mAdapter.setPrimaryItem(this, mCurItem, curItem != null ? curItem.object : null);
        //停止更新
        mAdapter.finishUpdate(this);

        // 检查页面宽度是否测量，没有则重新测量
        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            final View child = getChildAt(i);
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            lp.childIndex = i;
            if (!lp.isDecor && lp.widthFactor == 0.f) {
                // 0 means requery the adapter for this, it doesn't have a valid width.
                final ItemInfo ii = infoForChild(child);
                if (ii != null) {
                    lp.widthFactor = ii.widthFactor;
                    lp.position = ii.position;
                }
            }
        }
        sortChildDrawingOrder();

        //Viewpage被视为可获取焦点，则当前页面获取焦点
        if (hasFocus()) {
            View currentFocused = findFocus();
            ItemInfo ii = currentFocused != null ? infoForAnyChild(currentFocused) : null;
            if (ii == null || ii.position != mCurItem) {
                for (int i = 0; i < getChildCount(); i++) {
                    View child = getChildAt(i);
                    ii = infoForChild(child);
                    if (ii != null && ii.position == mCurItem) {
                        if (child.requestFocus(View.FOCUS_FORWARD)) {
                            break;
                        }
                    }
                }
            }
        }
    }
