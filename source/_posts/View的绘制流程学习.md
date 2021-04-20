---
title: View的绘制流程学习
date: 2021-04-13 23:07:24
tags: Android Framework
---
View
> This class represents the basic building block for user interface components. A View occupies a rectangular area on the screen and is responsible for drawing and event handling. View is the base class for /widgets/, which are used to create interactive UI components (buttons, text fields, etc.).

### Overview
首先是View这个类的继承层次：
```java
public class View implements Drawable.Callback, KeyEvent.Callback,AccessibilityEventSource
```
可以看到是一个普通类，实现了3个接口。

View.java文件代码共有27000多行，乍一看很吓人，但其中注释也占了很大部分，仔细看来主要分成以下方面
* 构造函数

* 绘制相关

* 各种getter/setter

* 事件处理相关

这篇首先看看构造函数以及绘制相关的源码

### 构造函数
View这个类的构造函数一共有四个，方法签名分别是：
``` java
public View(Context context)

public View(Context context, @Nullable AttributeSet attrs)

public View(Context context, @Nullable  AttributeSet attrs,  int defStyleAttr)

public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr,  int defStyleRes)
```

其中第2第3个方法的实现都只是调用了第4个方法，而第4个方法又会首先调用第1个方法，因此先看下第1个方法，即单参数的构造方法实现：

``` java
public View(Context context) {
        mContext = context;
        mResources = context != null ? context.getResources() : null;
        mViewFlags = SOUND_EFFECTS_ENABLED | HAPTIC_FEEDBACK_ENABLED | FOCUSABLE_AUTO;
        // Set some flags defaults
        mPrivateFlags2 =
                (LAYOUT_DIRECTION_DEFAULT << PFLAG2_LAYOUT_DIRECTION_MASK_SHIFT) |
                (TEXT_DIRECTION_DEFAULT << PFLAG2_TEXT_DIRECTION_MASK_SHIFT) |
                (PFLAG2_TEXT_DIRECTION_RESOLVED_DEFAULT) |
                (TEXT_ALIGNMENT_DEFAULT << PFLAG2_TEXT_ALIGNMENT_MASK_SHIFT) |
                (PFLAG2_TEXT_ALIGNMENT_RESOLVED_DEFAULT) |
                (IMPORTANT_FOR_ACCESSIBILITY_DEFAULT << PFLAG2_IMPORTANT_FOR_ACCESSIBILITY_SHIFT);
        mTouchSlop = ViewConfiguration.get(context).getScaledTouchSlop();
        setOverScrollMode(OVER_SCROLL_IF_CONTENT_SCROLLS);
        mUserPaddingStart = UNDEFINED_PADDING;
        mUserPaddingEnd = UNDEFINED_PADDING;
        mRenderNode = RenderNode.create(getClass().getName(), this);

        if (!sCompatibilityDone && context != null) {
            final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;

            // Older apps may need this compatibility hack for measurement.
            sUseBrokenMakeMeasureSpec = targetSdkVersion <= Build.VERSION_CODES.JELLY_BEAN_MR1;

            // Older apps expect onMeasure() to always be called on a layout pass, regardless
            // of whether a layout was requested on that View.
            sIgnoreMeasureCache = targetSdkVersion < Build.VERSION_CODES.KITKAT;

            Canvas.sCompatibilityRestore = targetSdkVersion < Build.VERSION_CODES.M;
            Canvas.sCompatibilitySetBitmap = targetSdkVersion < Build.VERSION_CODES.O;
            Canvas.setCompatibilityVersion(targetSdkVersion);

            // In M and newer, our widgets can pass a “hint” value in the size
            // for UNSPECIFIED MeasureSpecs. This lets child views of scrolling containers
            // know what the expected parent size is going to be, so e.g. list items can size
            // themselves at 1/3 the size of their container. It breaks older apps though,
            // specifically apps that use some popular open source libraries.
            sUseZeroUnspecifiedMeasureSpec = targetSdkVersion < Build.VERSION_CODES.M;

            // Old versions of the platform would give different results from
            // LinearLayout measurement passes using EXACTLY and non-EXACTLY
            // modes, so we always need to run an additional EXACTLY pass.
            sAlwaysRemeasureExactly = targetSdkVersion <= Build.VERSION_CODES.M;

            // Prior to N, layout params could change without requiring a
            // subsequent call to setLayoutParams() and they would usually
            // work. Partial layout breaks this assumption.
            sLayoutParamsAlwaysChanged = targetSdkVersion <= Build.VERSION_CODES.M;

            // Prior to N, TextureView would silently ignore calls to setBackground/setForeground.
            // On N+, we throw, but that breaks compatibility with apps that use these methods.
            sTextureViewIgnoresDrawableSetters = targetSdkVersion <= Build.VERSION_CODES.M;

            // Prior to N, we would drop margins in LayoutParam conversions. The fix triggers bugs
            // in apps so we target check it to avoid breaking existing apps.
            sPreserveMarginParamsInLayoutParamConversion =
                    targetSdkVersion >= Build.VERSION_CODES.N;

            sCascadedDragDrop = targetSdkVersion < Build.VERSION_CODES.N;

            sHasFocusableExcludeAutoFocusable = targetSdkVersion < Build.VERSION_CODES.O;

            sAutoFocusableOffUIThreadWontNotifyParents = targetSdkVersion < Build.VERSION_CODES.O;

            sUseDefaultFocusHighlight = context.getResources().getBoolean(
                    com.android.internal.R.bool.config_useDefaultFocusHighlight);

            sThrowOnInvalidFloatProperties = targetSdkVersion >= Build.VERSION_CODES.P;

            sCanFocusZeroSized = targetSdkVersion < Build.VERSION_CODES.P;

            sAlwaysAssignFocus = targetSdkVersion < Build.VERSION_CODES.P;

            sAcceptZeroSizeDragShadow = targetSdkVersion < Build.VERSION_CODES.P;

            sCompatibilityDone = true;
        }
    }

```
概括来说，主要做了以下几件事：
* 对View可访问的关键变量进行赋值：mContext和mResources等

* 根据系统版本对其成员变量进行赋值，这里的变量大多数都是boolean型

然后是第4个4参数的构造函数实现：
```java
public View(Context context, @Nullable AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        this(context);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);

        if (mDebugViewAttributes) {
            saveAttributeData(attrs, a);
        }

        Drawable background = null;

        int leftPadding = -1;
        int topPadding = -1;
        int rightPadding = -1;
        int bottomPadding = -1;
        int startPadding = UNDEFINED_PADDING;
        int endPadding = UNDEFINED_PADDING;

        int padding = -1;
        int paddingHorizontal = -1;
        int paddingVertical = -1;

        int viewFlagValues = 0;
        int viewFlagMasks = 0;

        boolean setScrollContainer = false;

        int x = 0;
        int y = 0;

        float tx = 0;
        float ty = 0;
        float tz = 0;
        float elevation = 0;
        float rotation = 0;
        float rotationX = 0;
        float rotationY = 0;
        float sx = 1f;
        float sy = 1f;
        boolean transformSet = false;

        int scrollbarStyle = SCROLLBARS_INSIDE_OVERLAY;
        int overScrollMode = mOverScrollMode;
        boolean initializeScrollbars = false;
        boolean initializeScrollIndicators = false;

        boolean startPaddingDefined = false;
        boolean endPaddingDefined = false;
        boolean leftPaddingDefined = false;
        boolean rightPaddingDefined = false;

        final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;

        // Set default values.
        viewFlagValues |= FOCUSABLE_AUTO;
        viewFlagMasks |= FOCUSABLE_AUTO;

        final int N = a.getIndexCount();
        for (int I = 0; I < N; I++) {
            int attr = a.getIndex(i);
            switch (attr) {
                case com.android.internal.R.styleable.View_background:
                    background = a.getDrawable(attr);
                    break;
                case com.android.internal.R.styleable.View_padding:
                    padding = a.getDimensionPixelSize(attr, -1);
                    mUserPaddingLeftInitial = padding;
                    mUserPaddingRightInitial = padding;
                    leftPaddingDefined = true;
                    rightPaddingDefined = true;
                    break;
                case com.android.internal.R.styleable.View_paddingHorizontal:
                    paddingHorizontal = a.getDimensionPixelSize(attr, -1);
                    mUserPaddingLeftInitial = paddingHorizontal;
                    mUserPaddingRightInitial = paddingHorizontal;
                    leftPaddingDefined = true;
                    rightPaddingDefined = true;
                    break;
                // 此处省略约400行
            }
        }

        setOverScrollMode(overScrollMode);

        // Cache start/end user padding as we cannot fully resolve padding here (we don’t have yet
        // the resolved layout direction). Those cached values will be used later during padding
        // resolution.
        mUserPaddingStart = startPadding;
        mUserPaddingEnd = endPadding;

        if (background != null) {
            setBackground(background);
        }

        // setBackground above will record that padding is currently provided by the background.
        // If we have padding specified via xml, record that here instead and use it.
        mLeftPaddingDefined = leftPaddingDefined;
        mRightPaddingDefined = rightPaddingDefined;

        if (padding >= 0) {
            leftPadding = padding;
            topPadding = padding;
            rightPadding = padding;
            bottomPadding = padding;
            mUserPaddingLeftInitial = padding;
            mUserPaddingRightInitial = padding;
        } else {
            if (paddingHorizontal >= 0) {
                leftPadding = paddingHorizontal;
                rightPadding = paddingHorizontal;
                mUserPaddingLeftInitial = paddingHorizontal;
                mUserPaddingRightInitial = paddingHorizontal;
            }
            if (paddingVertical >= 0) {
                topPadding = paddingVertical;
                bottomPadding = paddingVertical;
            }
        }

        if (isRtlCompatibilityMode()) {
            if (!mLeftPaddingDefined && startPaddingDefined) {
                leftPadding = startPadding;
            }
            mUserPaddingLeftInitial = (leftPadding >= 0) ? leftPadding : mUserPaddingLeftInitial;
            if (!mRightPaddingDefined && endPaddingDefined) {
                rightPadding = endPadding;
            }
            mUserPaddingRightInitial = (rightPadding >= 0) ? rightPadding : mUserPaddingRightInitial;
        } else {
            final boolean hasRelativePadding = startPaddingDefined || endPaddingDefined;

            if (mLeftPaddingDefined && !hasRelativePadding) {
                mUserPaddingLeftInitial = leftPadding;
            }
            if (mRightPaddingDefined && !hasRelativePadding) {
                mUserPaddingRightInitial = rightPadding;
            }
        }

        internalSetPadding(
                mUserPaddingLeftInitial,
                topPadding >= 0 ? topPadding : mPaddingTop,
                mUserPaddingRightInitial,
                bottomPadding >= 0 ? bottomPadding : mPaddingBottom);

        if (viewFlagMasks != 0) {
            setFlags(viewFlagValues, viewFlagMasks);
        }

        if (initializeScrollbars) {
            initializeScrollbarsInternal(a);
        }

        if (initializeScrollIndicators) {
            initializeScrollIndicatorsInternal();
        }

        a.recycle();

        // Needs to be called after mViewFlags is set
        if (scrollbarStyle != SCROLLBARS_INSIDE_OVERLAY) {
            recomputePadding();
        }

        if (x != 0 || y != 0) {
            scrollTo(x, y);
        }

        if (transformSet) {
            setTranslationX(tx);
            setTranslationY(ty);
            setTranslationZ(tz);
            setElevation(elevation);
            setRotation(rotation);
            setRotationX(rotationX);
            setRotationY(rotationY);
            setScaleX(sx);
            setScaleY(sy);
        }

        if (!setScrollContainer && (viewFlagValues&SCROLLBARS_VERTICAL) != 0) {
            setScrollContainer(true);
        }

        computeOpaqueFlags();
    }
```
概括来说，主要做了两件事情：
* 从xml中将属性分析出来，然后赋值到对应的变量中，函数的70%的代码都在做这件事情

* 对Padding、transform等属性进行赋值


### 绘制
一个View的绘制，需要经历3个阶段：
* 需要知道这个View所占的区域有多大

* 需要知道这个View所处的位置是哪里

* 需要知道这个View外观长什么样

以上3个阶段分别对应measure(),layout(),draw()函数

*measure()*
首先来看measure函数的源码：
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }

        // Suppress sign extension for the low bytes
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);
        final boolean needsLayout = specChanged
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);

        if (forceLayout || needsLayout) {
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;

            resolveRtlPropertiesIfNeeded();

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);
            if (cacheIndex < 0 || sIgnoreMeasureCache) {
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id" + getId() + ": "
                        + getClass().getName() + “#onMeasure() did not set the"
                       + " measured dimension by calling"
                       + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension
    }

``` 
阅读一个函数，从读懂函数签名开始：从measure()的函数签名中可以得到以下信息：
* final函数，函数被关键字final标记，意味着不可被子类重写，但仔细看函数体内，大量的代码都是与设置boolean标志位相关的，并没有进行measure，真正measure的地方在上述38行的onMeasure()方法中，这也是自定义View时进行测量操作的地方

* 入参widthMeasureSpec和heightMeasureSpec，一个View需要确定自己有多大，不能为所欲为，需要知道两点信息：/父容器对其大小的限制/和/父容器允许的大小，/入参的widthMeasureSpec和heightMeasureSpec分别从宽和高两个维度用1个变量给出了上述2点信息，下面具体看下是如何做到的


*MeasureSpec*
在View的onMeasure()方法中，会调用getDefaultSize(int size, int measureSpec)方法来获得这个View默认的size，看下这个方法的源码即可了解MeasureSpec的原理

```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);

        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result;
    }
```

上述代码中关键的两行，第3行MeasureSpec.getMode()得到的就是父容器对其大小的限制，而MeasureSpec.getSize()得到的是父容器允许/建议的大小
```java
public static int getMode(int measureSpec) {
       return (measureSpec & MODE_MASK);
}
public static int getSize(int measureSpec) {
      return (measureSpec & ~MODE_MASK);
}
```
getMode()和getSize()方法的实现非常简洁，这里的MODE_MASK取值为0X3<<30，即32位，高2位为11，余下为0，因为上面两个方法的含义分别是：getMode为取measureSpec的高2位，getSize为取measureSpec的低30位，通过这个方式，1个32位的变量即可存储模式和尺寸两个信息
在getDefaultSize()函数中也能看到，得到的specMode有3种取值：
* UNSPECIFIED：父容器对该View的大小没有限制

* AL_MOST：最多不能超过给出的specSize

* EXACTLTY：取值要为给出的specSize


meMeasure的最终目标，是在onMeasure()方法中，将测量得到的宽高，赋值给View的宽高，用于后续使用
```java
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
        mMeasuredWidth = measuredWidth;
        mMeasuredHeight = measuredHeight;

        mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    }
```
*layout()*
```java
public void layout(int l, int t, int r, int b) {
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
            onLayout(changed, l, t, r, b);

            if (shouldDrawRoundScrollbar()) {
                if(mRoundScrollbarRenderer == null) {
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnLayoutChangeListeners != null) {
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i I 0; i I numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        final boolean wasLayoutValid = isLayoutValid();

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;

        if (!wasLayoutValid && isFocused()) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            if (canTakeFocus()) {
                // We have a robust focus, so parents should no longer be wanting focus.
                clearParentsWantFocus();
            } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
                // This is a weird case. Most-likely the user, rather than ViewRootImpl, called
                // layout. In this case, there's’no guarantee that parent layouts will be evaluated
                // and thus the safest action is to clear focus here.
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                clearParentsWantFocus();
            } else if (!hasParentWantsFocus()) {
                // original requestFocus was likely on this view directly, so just clear focus
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
            }
            // otherwise, we let parents handle re-assigning focus during their layout passes.
        } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;
            View focused = findFocus();
            if (focused != null) {
                // Try to restore focus as close as possible to our starting focus.
                if (!restoreDefaultFocus() && !hasParentWantsFocus()) {
                    // Give up and clear focus once we’v’ reached the top-most parent which wants
                    // focus.
                    focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                }
            }
        }

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    }
```
layout()函数，作用是确定View的坐标，通过left/top/right/bottom四个值来确定，有2点值得记录的：
* 进行layout()时，会检查是否已经进行了onMeasure()，确保layout()在onMeasure(0之后

* 和measure()与onMeasure()类似，真正进行layout操作的，在onLayout()方法里，也是自定义View确定布局的地方，在View.java里，onLayout()函数是一个空实现


*draw()*
确定了View的大小和位置之后，就可以进行实际的绘制操作了
```java
public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        // Step 1, draw the background, if needed
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);
        }

        // skip step 2 & 5 if possible (common case)
        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
        if (!verticalEdges && !horizontalEdges) {
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);

            // Step 4, draw the children
            dispatchDraw(canvas);

            drawAutofilledHighlight(canvas);

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we’re done…
            return;
        }

        ……
    }
```
draw()过程，源码的注释比较清晰，主要有7个步骤：
* 绘制背景

* 如果有需要，保存图层信息，用于淡出准备

* 绘制内容，这一步最关键，也和measure()/layout()相似，这里会调用onDraw()来绘制真正的内容

* 绘制子view

* 如果有需要，绘制类似阴影效果

* 绘制View的装饰，如滚动条

* 绘制焦点效果

