---
title: ViewGroup绘制源码学习
date: 2021-04-13 23:07:24
tags: Android Framework
---
ViewGroup 
> A ViewGroup is a special view that can contain other views. The view group is the base class for layouts and views containers. This class also defines the
  android.view.ViewGroup.LayoutParams class which serves as the base class for layouts parameters.

### Overview
首先看下ViewGroup的继承层次：
``` java
public abstract class ViewGroup extends View implements ViewParent, ViewManager 
```
可以看到ViewGroup与View第一个不同点在于，ViewGroup是一个抽象类，继承自View，实现了ViewParent和ViewManager两个接口

### ViewParent和ViewManager
首先是ViewParent这个接口，内含的部分方法如下：
![ViewParent部分方法](https://raw.githubusercontent.com/SysuCodeMan/PicBed/main/20210506172925.png)

其次是ViewManager的方法：
![ViewManager](https://raw.githubusercontent.com/SysuCodeMan/PicBed/main/20210506173006.png)
ViewManager的方法很好理解，对View进行的增删改3个操作

### 构造方法
和View类似，ViewGroup也有4个构造函数：
```java
public ViewGroup(Context context)
```
```java
public ViewGroup(Context context, AttributeSet attrs)
```
```java
public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr)
```
```java
public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes)
```
但是实际上，前3个构造函数最终也只是调用到第4个，而第4个构造函数的实现也非常简单明了：
```java
public ViewGroup(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        initViewGroup();
        initFromAttributes(context, attrs, defStyleAttr, defStyleRes);
    }
```
即两个init方法的调用：
```java
private void initViewGroup() {
        // ViewGroup doesn't draw by default
        if (!debugDraw()) {
            setFlags(WILL_NOT_DRAW, DRAW_MASK);
        }
        mGroupFlags |= FLAG_CLIP_CHILDREN;
        mGroupFlags |= FLAG_CLIP_TO_PADDING;
        mGroupFlags |= FLAG_ANIMATION_DONE;
        mGroupFlags |= FLAG_ANIMATION_CACHE;
        mGroupFlags |= FLAG_ALWAYS_DRAWN_WITH_CACHE;

        if (mContext.getApplicationInfo().targetSdkVersion >= Build.VERSION_CODES.HONEYCOMB) {
            mGroupFlags |= FLAG_SPLIT_MOTION_EVENTS;
        }

        setDescendantFocusability(FOCUS_BEFORE_DESCENDANTS);

        mChildren = new View[ARRAY_INITIAL_CAPACITY];
        mChildrenCount = 0;

        mPersistentDrawingCache = PERSISTENT_SCROLLING_CACHE;
    }
```
首先initViewGroup()方法对其自身的一些标志位等做了初始操作；然后调用initFromAttributes()，从函数名称也可以猜到，这个函数作用是从xml属性中进行初始化操作：
```java
private void initFromAttributes(
            Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        final TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.ViewGroup, defStyleAttr,
                defStyleRes);

        final int N = a.getIndexCount();
        for (int i = 0; i < N; i++) {
            int attr = a.getIndex(i);
            switch (attr) {
                case R.styleable.ViewGroup_clipChildren:
                    setClipChildren(a.getBoolean(attr, true));
                    break;
                case R.styleable.ViewGroup_clipToPadding:
                    setClipToPadding(a.getBoolean(attr, true));
                    break;
                case R.styleable.ViewGroup_animationCache:
                    setAnimationCacheEnabled(a.getBoolean(attr, true));
                    break;
                ……
                case R.styleable.ViewGroup_touchscreenBlocksFocus:
                    setTouchscreenBlocksFocus(a.getBoolean(attr, false));
                    break;
            }
        }

        a.recycle();
    }
```

### 绘制相关
从View的绘制源码学习中已经知道，一个View的绘制，包括measure/layout/draw三个阶段，那么作为View的子类，ViewGroup的绘制也无非这3个阶段：

##### measure
因为View的measure()方法是个final方法，因此在ViewGroup中不能重写改方法，而ViewGroup中也没有实现onMeasure()方法，但在measure阶段，ViewGroup提供了几个measure相关的方法：
```java
protected void measureChildren(int widthMeasureSpec, int heightMeasureSpec)
```
```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec)
```
```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) 
```
这几个方法在实现上，关键的点是一致的，因此选择相对简洁的measureChild()来看：
```java
protected void measureChild(View child, int parentWidthMeasureSpec,
            int parentHeightMeasureSpec) {
        final LayoutParams lp = child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
最关键的点在于最后一句，child.measure()，从View绘制过程已经知道，measure阶段的目的在于，让View得到自己所占据的大小，因此对ViewGroup来说，measure阶段，就需要让自己的所有子View知道自己的大小。

##### layout
在layout阶段，ViewGroup重写了View的layout()和onlayout()方法

实际上layout并没有做什么实质的工作：
```java
public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);
            }
            super.layout(l, t, r, b);
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
```

更加关键的是onLayout()方法，这里变成了abstract抽象方法，这也是为什么ViewGroup是一个抽象类的原因
``` java
    protected abstract void onLayout(b
            oolean changed,
            int l, int t, int r, int b);
```

##### draw
在draw阶段，ViewGroup里相关的函数是dispatchDraw()，drawChild()：在dispatchDraw()中遍历所有的子View，对所有子View执行drawChild()方法，而drawChild()方法的实现非常简单：
```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);
    }
```
ViewGroup的绘制，最终也就是各个子View自身的绘制。

### 总结
最后总结一下ViewGroup和View的差异：
- View是一个普通类，而ViewGroup是一个抽象类
- measure阶段，View通过measure()和onMeasure()来确定自身的占据范围，ViewGroup通过measureChild()等方法来确定各个子View的占据范围
- layout阶段，View通过layout()和onLayout()函数来确定位置，而ViewGroup中onLayout()变成抽象方法，需要子类实现；
- draw阶段，View通过draw()和onDraw()方法来实现绘制，ViewGroup通过dispatchDraw()和drawChild()方法来让各个子View绘制自身。

