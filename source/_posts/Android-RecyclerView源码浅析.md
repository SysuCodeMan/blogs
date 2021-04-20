---
title: Android RecyclerView源码浅析
date: 2021-04-16 17:48:41
tags: Android控件
---

### OverView
RecyclerView是Android5.0推出的新组件，可以认为是更加灵活强大的ListView，在日常开发中基本上已经取代了ListView成为长列表控件的首选。
其继承结构如下：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-c382744bd06ca4a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
从继承结构可以看到，RecyclerView是ViewGroup的直接子类，这里就是RecyclerView与ListView第一点大不同，ListView和ViewGroup之间隔了两层，其实个人认为在继承结构上揭露了RecyclerView相比ListView的优点：由于RecyclerView这么简单的继承结构，说明了其主要功能都是通过**组合**来实现的，而ListView是通过**继承**实现的。

### 用法
通常来说，在业务代码中使用RecyclerView，主要有以下几个步骤：
- 创建RecyclerView实例
- 为RecyclerView设置LayoutManager，LayoutManager是一个抽象类，常用的实现类有LinearLayoutManager
- 如有必要，为RecyclerView设置ItemAnimator
- 如有必要，为RecyclerView设置ItemDecoration
- 继承RecyclerView.Adapter<VH extends ViewHolder>，实现getItemCount()/onCreateViewHolder(ViewGroup parent, int viewType)/onBindViewHolder(VH holder, int position)
- 在RecyclerView.Adapter中使用到的VH是ViewHolder的实现类，但ViewHolder虽然是一个抽象类，其中却没有需要我们实现的抽象方法，ViewHolder中关键的成员是itemView，即其承载的视图，可见性为public，可以直接获取

### 源码浅析
这部分跟随RecyclerView使用到的各个类的方法，看下这些关键方法是在何时被调用的

### Adapter
- public RecyclerView.ViewHolder onCreateViewHolder(ViewGroup parent, int viewType)
在源码中查找onCreateViewHolder(ViewGroup parent, int viewType)的调用，能得到下图
![image.png](https://upload-images.jianshu.io/upload_images/5866715-20937ebf735b9cca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
内一层是外一层的调用，最终可以看到主要的调用在LinearLayoutManager的onLayoutChildren()；直接的调用方法是在RecyclerView的tryGetViewHolderForPositionByDeadline(int position, boolean dryRun, long deadlineNs)中有这么1段：
``` java
                if (holder == null) {
                    long start = getNanoTime();
                    if (deadlineNs != FOREVER_NS
                            && !mRecyclerPool.willCreateInTime(type, start, deadlineNs)) {
                        // abort - we have a deadline we can't meet
                        return null;
                    }
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                    if (ALLOW_THREAD_GAP_WORK) {
                        // only bother finding nested RV if prefetching
                        RecyclerView innerView = findNestedRecyclerView(holder.itemView);
                        if (innerView != null) {
                            holder.mNestedRecyclerView = new WeakReference<>(innerView);
                        }
                    }

                    long end = getNanoTime();
                    mRecyclerPool.factorInCreateTime(type, end - start);
                    if (DEBUG) {
                        Log.d(TAG, "tryGetViewHolderForPositionByDeadline created new ViewHolder");
                    }
                }
```
简单来说，就是在layout()过程，如果holder为null，就调用这个方法来创建一个ViewHolder

- public void onBindViewHolder(RecyclerView.ViewHolder holder, int position)
首先在RecyclerView.Adapter的bindViewHolder(VH holder, int position)中有直接调用：
``` java
        public final void bindViewHolder(VH holder, int position) {
            holder.mPosition = position;
            if (hasStableIds()) {
                holder.mItemId = getItemId(position);
            }
            holder.setFlags(ViewHolder.FLAG_BOUND,
                    ViewHolder.FLAG_BOUND | ViewHolder.FLAG_UPDATE | ViewHolder.FLAG_INVALID
                            | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN);
            TraceCompat.beginSection(TRACE_BIND_VIEW_TAG);
            onBindViewHolder(holder, position, holder.getUnmodifiedPayloads());
            holder.clearPayload();
            final ViewGroup.LayoutParams layoutParams = holder.itemView.getLayoutParams();
            if (layoutParams instanceof RecyclerView.LayoutParams) {
                ((LayoutParams) layoutParams).mInsetsDirty = true;
            }
            TraceCompat.endSection();
        }
```
调用完之后从其itemView中取出LayoutParams来进行处理
然后再源码中查找最终的调用链，结果如下：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-af28fd79ed1d96b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
调用链中关键的一环是在RecyclerView的View getViewForPosition(int position, boolean dryRun)中
``` java
        View getViewForPosition(int position, boolean dryRun) {
            return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
        }
```
里面先调用了tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS)方法，在前面分析onCreateViewHolder()的时候已经看到了，这个方法里尝试获取holder，取不到就创建holder，创建后就进行onBindViewHolder()的操作，bind完之后在这里可以看到，返回的是holder.itemView，因此在布局过程中，就是从这里获得了正确的itemView

- int getItemCount()
这个方法的调用，在源码里看都是直接调用
![image.png](https://upload-images.jianshu.io/upload_images/5866715-0b8d7170c3e44049.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- int getItemViewType(int position)
这个方法在源码中的被调用位置主要是RecyclerView的tryGetViewHolderForPositionByDeadline()函数，这个函数在上面也已经出现过了
``` java
                final int type = mAdapter.getItemViewType(offsetPosition);
                // 2) Find from scrap/cache via stable ids, if exists
                if (mAdapter.hasStableIds()) {
                    holder = getScrapOrCachedViewForId(mAdapter.getItemId(offsetPosition),
                            type, dryRun);
                    if (holder != null) {
                        // update position
                        holder.mPosition = offsetPosition;
                        fromScrapOrHiddenOrCache = true;
                    }
                }
```
通过这个方法拿到的type，就是在onCreateViewHolder()/onBindViewHolder()函数中的入参type

### LayoutManager
- public abstract LayoutParams generateDefaultLayoutParams();
查找源码，这个方法被调用的位置为RecyclerView的同名函数中：
``` java
    @Override
    protected ViewGroup.LayoutParams generateDefaultLayoutParams() {
        if (mLayout == null) {
            throw new IllegalStateException("RecyclerView has no LayoutManager");
        }
        return mLayout.generateDefaultLayoutParams();
    }

```
- public void onLayoutChildren(Recycler recycler, State state) 
这个方法本身并不是abstract函数，但在LayoutManager中必须重写：
``` java
public void onLayoutChildren(Recycler recycler, State state) {
            Log.e(TAG, "You must override onLayoutChildren(Recycler recycler, State state) ");
}
```
在源码中查找其调用，可以看到被调用的地方主要有两处：
首先是RecyclerView的dispatchLayoutStep1()函数
``` java
    private void dispatchLayoutStep1() {
        mState.assertLayoutStep(State.STEP_START);
        mState.mIsMeasuring = false;
        eatRequestLayout();
        mViewInfoStore.clear();
        onEnterLayoutOrScroll();
        processAdapterUpdatesAndSetAnimationFlags();
        saveFocusInfo();
        mState.mTrackOldChangeHolders = mState.mRunSimpleAnimations && mItemsChanged;
        mItemsAddedOrRemoved = mItemsChanged = false;
        mState.mInPreLayout = mState.mRunPredictiveAnimations;
        mState.mItemCount = mAdapter.getItemCount();
        findMinMaxChildLayoutPositions(mMinMaxLayoutPositions);

        if (mState.mRunSimpleAnimations) {
            // Step 0: Find out where all non-removed items are, pre-layout
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                if (holder.shouldIgnore() || (holder.isInvalid() && !mAdapter.hasStableIds())) {
                    continue;
                }
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                if (mState.mTrackOldChangeHolders && holder.isUpdated() && !holder.isRemoved()
                        && !holder.shouldIgnore() && !holder.isInvalid()) {
                    long key = getChangedHolderKey(holder);
                    // This is NOT the only place where a ViewHolder is added to old change holders
                    // list. There is another case where:
                    //    * A VH is currently hidden but not deleted
                    //    * The hidden item is changed in the adapter
                    //    * Layout manager decides to layout the item in the pre-Layout pass (step1)
                    // When this case is detected, RV will un-hide that view and add to the old
                    // change holders list.
                    mViewInfoStore.addToOldChangeHolders(key, holder);
                }
            }
        }
        if (mState.mRunPredictiveAnimations) {
            // Step 1: run prelayout: This will use the old positions of items. The layout manager
            // is expected to layout everything, even removed items (though not to add removed
            // items back to the container). This gives the pre-layout position of APPEARING views
            // which come into existence as part of the real layout.

            // Save old positions so that LayoutManager can run its mapping logic.
            saveOldPositions();
            final boolean didStructureChange = mState.mStructureChanged;
            mState.mStructureChanged = false;
            // temporarily disable flag because we are asking for previous layout
            mLayout.onLayoutChildren(mRecycler, mState);
            mState.mStructureChanged = didStructureChange;

            for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
                final View child = mChildHelper.getChildAt(i);
                final ViewHolder viewHolder = getChildViewHolderInt(child);
                if (viewHolder.shouldIgnore()) {
                    continue;
                }
                if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                    int flags = ItemAnimator.buildAdapterChangeFlagsForAnimations(viewHolder);
                    boolean wasHidden = viewHolder
                            .hasAnyOfTheFlags(ViewHolder.FLAG_BOUNCED_FROM_HIDDEN_LIST);
                    if (!wasHidden) {
                        flags |= ItemAnimator.FLAG_APPEARED_IN_PRE_LAYOUT;
                    }
                    final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(
                            mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                    if (wasHidden) {
                        recordAnimationInfoIfBouncedHiddenView(viewHolder, animationInfo);
                    } else {
                        mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                    }
                }
            }
            // we don't process disappearing list because they may re-appear in post layout pass.
            clearOldPositions();
        } else {
            clearOldPositions();
        }
        onExitLayoutOrScroll();
        resumeRequestLayout(false);
        mState.mLayoutStep = State.STEP_LAYOUT;
    }
```
其次是dispatchLayoutStep2()中：
``` java
    private void dispatchLayoutStep2() {
        eatRequestLayout();
        onEnterLayoutOrScroll();
        mState.assertLayoutStep(State.STEP_LAYOUT | State.STEP_ANIMATIONS);
        mAdapterHelper.consumeUpdatesInOnePass();
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

### ItemDecoration
- onDraw(Canvas c, RecyclerView parent, RecyclerView.State state)
这个函数作用在于绘制分割线，在源码中被调用的地方在RecyclerView中：
``` java
    @Override
    public void onDraw(Canvas c) {
        super.onDraw(c);

        final int count = mItemDecorations.size();
        for (int i = 0; i < count; i++) {
            mItemDecorations.get(i).onDraw(c, this, mState);
        }
    }
```
其实就是RecyclerView的onDraw()函数中进行了分割线的绘制

- public void getItemOffsets(Rect outRect, View view, RecyclerView parent, State state)
这个函数是，在每一项itemView绘制的时候，通过控制outRect的left/top/right/bottom值，来确定itemView四周留出来的范围，效果类似于padding或margin
在源码中被调用的地方有：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-bf7a5a266bd563b9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到，主要是和measure有关的函数，很容易理解，因为就是在measure过程确定一个itemView占据多少空间

### ItemAnimator
ItemAnimator主要用于对item的插入、修改、删除动画处理主要有animateApprearance/animateDisappearance/animateChange/animatePersistence这几个在对应动作的函数
- public abstract boolean animateAppearance(@NonNull ViewHolder viewHolder, @Nullable ItemHolderInfo preLayoutInfo, @NonNull ItemHolderInfo postLayoutInfo);
在源码中查找animateAppearance的调用链为：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-f9faa327825620a5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
最终又是在熟悉的dispatchLayoutStep3()中被调用
``` java
            // Step 4: Process view info lists and trigger animations
            mViewInfoStore.process(mViewInfoProcessCallback);
```
但这一行并不是很能看出怎么就调用了animateAppearance，看下再上一步的调用：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-410a68952c1c2acf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到在调用process()函数的时候，是有根据record.flags来判断应该执行的是哪个处理

-  public abstract boolean animateDisappearance(@NonNull ViewHolder viewHolder, @NonNull ItemHolderInfo preLayoutInfo, @Nullable ItemHolderInfo postLayoutInfo)
在源码中查找其调用位置：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-79c3bdf424c8368c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
可以看到跟animateAppearance一样，最终都是被dispatchLayoutStep3()调用

- public abstract boolean animateChange(@NonNull ViewHolder oldHolder,
                @NonNull ViewHolder newHolder,
                @NonNull ItemHolderInfo preLayoutInfo, @NonNull ItemHolderInfo postLayoutInfo)
这个函数的调用链和上述一致，就不重复贴图了


- public abstract boolean animatePersistence(@NonNull ViewHolder viewHolder,
                @NonNull ItemHolderInfo preLayoutInfo, @NonNull ItemHolderInfo postLayoutInfo);
这个函数的调用链依然和上述一致

