---
title: Android ListView源码学习
date: 2021-04-16 17:45:59
tags: Android源码学习
---
### OverView
Android ListView 是Android中常用的长列表组件，其继承层次如下：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-5599c6565bf045a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 用法
通常在业务代码中使用ListView的常用姿势是：
- 创建1个ListView
- 创建1个BaseAdapter的子类，实现getCount/getItem/getItemId/getView这4个方法，有时候还会实现getItemViewType/getViewTypeCount方法来满足有多种ItemView样式的需求
- 将BaseAdatper的子类实例通过ListView的setAdapter()方法，设置给ListView实例

### 常用优化
通常的ListView在View的复用上有2种优化：
- public View getView(int position, View convertView, ViewGroup parent)这个方法在实现是，首先判断一下传入的convertView是否为null，不为null即可复用，无需调用inflate或者new来新创建1个View
- 可以通过ViewHoloder的方法，将convertView的子View直接存一个引用在ViewHolder中，然后将ViewHolder通过convertView的setTag方法存储在convertView上；这种做法的好处在于，通过对子View的直接引用访问，避免了findViewById的耗时操作

### 源码浅析
ListView的源码比较长，暂时先把精力放在理解Adapter的6个方法(getViewTypeCount/getItemViewType/getCount/getItem/getItemId/getView)被调用的时机上，更详细的源码分析文章已经很多了，比较经典的有郭霖前辈的[https://blog.csdn.net/sinyu890807/article/details/44996879](https://blog.csdn.net/sinyu890807/article/details/44996879)

### int getViewTypeCount()
在源码中搜索getViewTypeCount()被引用的位置，得到的和ListView相关的结果是在setAdapter(ListAdapter adapter)方法中有1行：
``` java
mRecycler.setViewTypeCount(mAdapter.getViewTypeCount());
```
这个mRecycler成员是RecycleBin类型，RecycleBin的定义在AbsListView中，其作用顾名思义，就是起到1个回收的垃圾箱作用，其setViewTypeCount(int viewTypeCount)方法的实现为：
``` java
        public void setViewTypeCount(int viewTypeCount) {
            if (viewTypeCount < 1) {
                throw new IllegalArgumentException("Can't have a viewTypeCount < 1");
            }
            //noinspection unchecked
            ArrayList<View>[] scrapViews = new ArrayList[viewTypeCount];
            for (int i = 0; i < viewTypeCount; i++) {
                scrapViews[i] = new ArrayList<View>();
            }
            mViewTypeCount = viewTypeCount;
            mCurrentScrap = scrapViews[0];
            mScrapViews = scrapViews;
        }
```
RecycleBin回收废弃View的实现是通过其scrapViews数组实现的，而传入的viewTypeCount决定了这个数组的长度，注意scrapViews的每一个成员是一个ArrayList<View>；在这里我的理解是，viewTypeCount决定了有多少种View会被回收，而每1个被回收的View会根据viewType进入到对应的ArrayList<View>中去，方便在复用时从正确的类型中取出对应的View来进行复用。

### int getItemViewType(int position)
在源码中搜索该函数，有好几处调用的地方，但多数调用都是得到viewType之后设置到AbsListView.LayoutParams的viewType属性上使用，这种使用并不是非常重要，真正关键的调用在AbsListView的getScrapView(position)函数中：
``` java
        View getScrapView(int position) {
            final int whichScrap = mAdapter.getItemViewType(position);
            if (whichScrap < 0) {
                return null;
            }
            if (mViewTypeCount == 1) {
                return retrieveFromScrap(mCurrentScrap, position);
            } else if (whichScrap < mScrapViews.length) {
                return retrieveFromScrap(mScrapViews[whichScrap], position);
            }
            return null;
        }
```
getScrapView(int position)函数的作用在于，从废弃的View中获取一个View，准备复用，从实现上可以看出，getItemViewType的作用在于，得到正确的ViewType，从而从对应的mScrapViews数组中取出1个ScrapView

### int getCount()
getCount()的调用在源码中的搜索结果就实在是太多了，粗略浏览的一下，把觉得比较关键的点记录下来

- 首先是ListView的setAdatper()函数中有这么一句：
``` java
 mItemCount = mAdapter.getCount();
```
这个mItemCount是ListView的祖先类AdapterView的成员，设置到这上面之后就能更加方便地使用了

- 然后是ListView的layoutChildren()函数中有这么一段：
``` java
            if (mItemCount == 0) {
                resetList();
                invokeOnItemScrollListener();
                return;
            } else if (mItemCount != mAdapter.getCount()) {
                throw new IllegalStateException("The content of the adapter has changed but "
                        + "ListView did not receive a notification. Make sure the content of "
                        + "your adapter is not modified from a background thread, but only from "
                        + "the UI thread. Make sure your adapter calls notifyDataSetChanged() "
                        + "when its content changes. [in ListView(" + getId() + ", " + getClass()
                        + ") with Adapter(" + mAdapter.getClass() + ")]");
            }

```
在layoutChildren()过程中需要检查mItemCount和mAdapter.getCount()是否一致，如果不一致，证明数据源被改变了却没有调用notifyDataSetChanged()通知观察方

### Object getItem(int position)
getItem(int position)方法主要被调用的地方在AdapterView的getItemAtPosition(int position)函数中：
``` java
    public Object getItemAtPosition(int position) {
        T adapter = getAdapter();
        return (adapter == null || position < 0) ? null : adapter.getItem(position);
    }
```
除此之外，在源码中再没找到getItem(int position)的相关调用，这也可以理解，因为getItem(int position)更主要的使用场景是我们在业务代码中调用，通过该方法，能够从Adapter里拿出数据项，而不需要直接跟数据源接触。

### long getItemId(int position)
getItemId(int postion)函数在源码中搜索，ListView中的调用已经被标注为@Deprecate，其余主要的调用都在AbsListView中，选择其中一处来看下这个方法的作用
``` java
   public void setItemChecked(int position, boolean value) {
        if (mChoiceMode == CHOICE_MODE_NONE) {
            return;
        }

        // Start selection mode if needed. We don't need to if we're unchecking something.
        if (value && mChoiceMode == CHOICE_MODE_MULTIPLE_MODAL && mChoiceActionMode == null) {
            if (mMultiChoiceModeCallback == null ||
                    !mMultiChoiceModeCallback.hasWrappedCallback()) {
                throw new IllegalStateException("AbsListView: attempted to start selection mode " +
                        "for CHOICE_MODE_MULTIPLE_MODAL but no choice mode callback was " +
                        "supplied. Call setMultiChoiceModeListener to set a callback.");
            }
            mChoiceActionMode = startActionMode(mMultiChoiceModeCallback);
        }

        final boolean itemCheckChanged;
        if (mChoiceMode == CHOICE_MODE_MULTIPLE || mChoiceMode == CHOICE_MODE_MULTIPLE_MODAL) {
            boolean oldValue = mCheckStates.get(position);
            mCheckStates.put(position, value);
            if (mCheckedIdStates != null && mAdapter.hasStableIds()) {
                if (value) {
                    mCheckedIdStates.put(mAdapter.getItemId(position), position);
                } else {
                    mCheckedIdStates.delete(mAdapter.getItemId(position));
                }
            }
            itemCheckChanged = oldValue != value;
            if (itemCheckChanged) {
                if (value) {
                    mCheckedItemCount++;
                } else {
                    mCheckedItemCount--;
                }
            }
            if (mChoiceActionMode != null) {
                final long id = mAdapter.getItemId(position);
                mMultiChoiceModeCallback.onItemCheckedStateChanged(mChoiceActionMode,
                        position, id, value);
            }
        } else {
           ……
            }
            // this may end up selecting the value we just cleared but this way
            // we ensure length of mCheckStates is 1, a fact getCheckedItemPosition relies on
            if (value) {
                mCheckStates.put(position, true);
                if (updateIds) {
                    mCheckedIdStates.put(mAdapter.getItemId(position), position);
                }
                mCheckedItemCount = 1;
            } else if (mCheckStates.size() == 0 || !mCheckStates.valueAt(0)) {
                mCheckedItemCount = 0;
            }
        }

        // Do not generate a data change while we are in the layout phase or data has not changed
        if (!mInLayout && !mBlockLayoutRequests && itemCheckChanged) {
            mDataChanged = true;
            rememberSyncState();
            requestLayout();
        }
    }
```
可以看到，主要是通过该方法，获得对应位置的id后，能够作为一个索引，用于增删改查等快速操作。

### View getView(int position, View convertView, ViewGroup parent)
最后Adapter中最重要的方法，getView方法的作用是返回某一项对应的ItemView，在源码中关键的调用是AbsListView中的obtainView()方法中的调用：
```
        final View scrapView = mRecycler.getScrapView(position);
        final View child = mAdapter.getView(position, scrapView, this);
        if (scrapView != null) {
            if (child != scrapView) {
                // Failed to re-bind the data, return scrap to the heap.
                mRecycler.addScrapView(scrapView, position);
            } else if (child.isTemporarilyDetached()) {
                outMetadata[0] = true;

                // Finish the temporary detach started in addScrapView().
                child.dispatchFinishTemporaryDetach();
            }
        }
```
AbsListView的obtainView函数的作用就是构建出某一position对应的View，首先会从mRecycler中取出一个对应的废弃View，这个废弃View就是传入Adapter的getView()方法中的convertView，这里也就解释了为什么需要判空——在首次布局时，实际上是还没有废弃的View可用的，而后面布局时就有废弃的View可复用，无需重新构建了。
