---
title: Android ImageView源码浅析
date: 2021-04-16 17:50:43
tags: Android控件
---
### OverView
官方文档中关于ImageView的介绍是：
Displays image resources, for example Bitmap or Drawable resources. ImageView is also commonly used to apply tints to an image and handle image scaling.
即用来展示图像资源的控件。
其继承层次为
![image.png](https://upload-images.jianshu.io/upload_images/5866715-cc8fc944cabf6d5d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### ImageView.ScaleType
在ImageView中有一个内部枚举类ScaleType，用来控制图像和ImageView的尺寸不一致时的拉伸规则：
| 枚举值 | 含义
| ----    | ---- |
| CENTER| 不拉伸图像，居中置于ImageView中|
|CENTER_CROP|等比例拉伸图像，直至长和宽都等于或者大于ImageView的对应边，然后居中置于ImageView，（如果本来的长宽就都大于ImageView的对应边，则不拉伸）|
|CENTER_INSIDE|等比例拉伸图像，直至长和宽都等于或者小于ImageView的对应边，然后居中置于ImageView(如果本来的长宽就都小于ImageView的对应边，则不拉伸)|
|FILL_XY|图像的长和宽都拉伸到和ImageView一致，以充满ImageView，这种情况下会改变图像的比例|
|FILL_CENTER|等比例拉伸图片，直到某一边刚好和ImageView一致，然后居中置于ImageView|
|FILL_START|等比例拉伸图片，直到某一边刚好和ImageView一致，然后靠开始位置置于ImageView|
|FILL_END|等比例拉伸图片，直到某一边刚好和ImageView一致，然后靠结束位置置于ImageView|
|MATRIX|从左上角开始平铺图像|

### 构造函数
``` java
public ImageView(Context context) {
        super(context);
        initImageView();
    }

    public ImageView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public ImageView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public ImageView(Context context, @Nullable AttributeSet attrs, int defStyleAttr,
            int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        initImageView();

        if (getImportantForAutofill() == IMPORTANT_FOR_AUTOFILL_AUTO) {
            setImportantForAutofill(IMPORTANT_FOR_AUTOFILL_NO);
        }

        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.ImageView, defStyleAttr, defStyleRes);

        final Drawable d = a.getDrawable(R.styleable.ImageView_src);
        if (d != null) {
            setImageDrawable(d);
        }

        mBaselineAlignBottom = a.getBoolean(R.styleable.ImageView_baselineAlignBottom, false);
        mBaseline = a.getDimensionPixelSize(R.styleable.ImageView_baseline, -1);

        setAdjustViewBounds(a.getBoolean(R.styleable.ImageView_adjustViewBounds, false));
        setMaxWidth(a.getDimensionPixelSize(R.styleable.ImageView_maxWidth, Integer.MAX_VALUE));
        setMaxHeight(a.getDimensionPixelSize(R.styleable.ImageView_maxHeight, Integer.MAX_VALUE));

        final int index = a.getInt(R.styleable.ImageView_scaleType, -1);
        if (index >= 0) {
            setScaleType(sScaleTypeArray[index]);
        }

        if (a.hasValue(R.styleable.ImageView_tint)) {
            mDrawableTintList = a.getColorStateList(R.styleable.ImageView_tint);
            mHasDrawableTint = true;

            mDrawableTintMode = PorterDuff.Mode.SRC_ATOP;
            mHasDrawableTintMode = true;
        }

        if (a.hasValue(R.styleable.ImageView_tintMode)) {
            mDrawableTintMode = Drawable.parseTintMode(a.getInt(
                    R.styleable.ImageView_tintMode, -1), mDrawableTintMode);
            mHasDrawableTintMode = true;
        }

        applyImageTint();

        final int alpha = a.getInt(R.styleable.ImageView_drawableAlpha, 255);
        if (alpha != 255) {
            setImageAlpha(alpha);
        }

        mCropToPadding = a.getBoolean(
                R.styleable.ImageView_cropToPadding, false);

        a.recycle();
    }
```
View的构造函数的写法都比较模板化，4个构造函数，2参数和3参数最终调用到的是4参数的构造函数，4参数的构造函数从xml中解析出各个属性也和其他View的构造函数是相同套路
单参数和4参数的构造函数，主要的工作都是调用了initImageView()函数
看下initImageView()函数的实现：
``` java
   private void initImageView() {
        mMatrix = new Matrix();
        mScaleType = ScaleType.FIT_CENTER;

        if (!sCompatDone) {
            final int targetSdkVersion = mContext.getApplicationInfo().targetSdkVersion;
            sCompatAdjustViewBounds = targetSdkVersion <= Build.VERSION_CODES.JELLY_BEAN_MR1;
            sCompatUseCorrectStreamDensity = targetSdkVersion > Build.VERSION_CODES.M;
            sCompatDrawableVisibilityDispatch = targetSdkVersion < Build.VERSION_CODES.N;
            sCompatDone = true;
        }
    }
```
initImageView()函数也很简单，初始化了一个mMatrix变量，将mScaleType设置成默认的ScaleType.FIT_CENTER（后面展开说下ScaleType这个枚举类）
接下来看下4参数中调用的setImageDrawable(Drawable d)的调用，因为在实际使用中，ImageView的最关键就在于把图像展示出来

``` java
    public void setImageDrawable(@Nullable Drawable drawable) {
        if (mDrawable != drawable) {
            mResource = 0;
            mUri = null;

            final int oldWidth = mDrawableWidth;
            final int oldHeight = mDrawableHeight;

            updateDrawable(drawable);

            if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                requestLayout();
            }
            invalidate();
        }
    }
```
主要逻辑在updateDrawable(Drawable d)里面
``` java
    private void updateDrawable(Drawable d) {
        if (d != mRecycleableBitmapDrawable && mRecycleableBitmapDrawable != null) {
            mRecycleableBitmapDrawable.setBitmap(null);
        }

        boolean sameDrawable = false;

        if (mDrawable != null) {
            sameDrawable = mDrawable == d;
            mDrawable.setCallback(null);
            unscheduleDrawable(mDrawable);
            if (!sCompatDrawableVisibilityDispatch && !sameDrawable && isAttachedToWindow()) {
                mDrawable.setVisible(false, false);
            }
        }

        mDrawable = d;

        if (d != null) {
            d.setCallback(this);
            d.setLayoutDirection(getLayoutDirection());
            if (d.isStateful()) {
                d.setState(getDrawableState());
            }
            if (!sameDrawable || sCompatDrawableVisibilityDispatch) {
                final boolean visible = sCompatDrawableVisibilityDispatch
                        ? getVisibility() == VISIBLE
                        : isAttachedToWindow() && getWindowVisibility() == VISIBLE && isShown();
                d.setVisible(visible, true);
            }
            d.setLevel(mLevel);
            mDrawableWidth = d.getIntrinsicWidth();
            mDrawableHeight = d.getIntrinsicHeight();
            applyImageTint();
            applyColorMod();

            configureBounds();
        } else {
            mDrawableWidth = mDrawableHeight = -1;
        }
    }
```
其中的applyImageTint()和applyColorMod()函数的作用是设置着色、颜色相关的属性，比较关键的函数调用是configureBounds()
``` java
    private void configureBounds() {
        if (mDrawable == null || !mHaveFrame) {
            return;
        }

        final int dwidth = mDrawableWidth;
        final int dheight = mDrawableHeight;

        final int vwidth = getWidth() - mPaddingLeft - mPaddingRight;
        final int vheight = getHeight() - mPaddingTop - mPaddingBottom;

        final boolean fits = (dwidth < 0 || vwidth == dwidth)
                && (dheight < 0 || vheight == dheight);

        if (dwidth <= 0 || dheight <= 0 || ScaleType.FIT_XY == mScaleType) {
            /* If the drawable has no intrinsic size, or we're told to
                scaletofit, then we just fill our entire view.
            */
            mDrawable.setBounds(0, 0, vwidth, vheight);
            mDrawMatrix = null;
        } else {
            // We need to do the scaling ourself, so have the drawable
            // use its native size.
            mDrawable.setBounds(0, 0, dwidth, dheight);

            if (ScaleType.MATRIX == mScaleType) {
                // Use the specified matrix as-is.
                if (mMatrix.isIdentity()) {
                    mDrawMatrix = null;
                } else {
                    mDrawMatrix = mMatrix;
                }
            } else if (fits) {
                // The bitmap fits exactly, no transform needed.
                mDrawMatrix = null;
            } else if (ScaleType.CENTER == mScaleType) {
                // Center bitmap in view, no scaling.
                mDrawMatrix = mMatrix;
                mDrawMatrix.setTranslate(Math.round((vwidth - dwidth) * 0.5f),
                                         Math.round((vheight - dheight) * 0.5f));
            } else if (ScaleType.CENTER_CROP == mScaleType) {
                mDrawMatrix = mMatrix;

                float scale;
                float dx = 0, dy = 0;

                if (dwidth * vheight > vwidth * dheight) {
                    scale = (float) vheight / (float) dheight;
                    dx = (vwidth - dwidth * scale) * 0.5f;
                } else {
                    scale = (float) vwidth / (float) dwidth;
                    dy = (vheight - dheight * scale) * 0.5f;
                }

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(Math.round(dx), Math.round(dy));
            } else if (ScaleType.CENTER_INSIDE == mScaleType) {
                mDrawMatrix = mMatrix;
                float scale;
                float dx;
                float dy;

                if (dwidth <= vwidth && dheight <= vheight) {
                    scale = 1.0f;
                } else {
                    scale = Math.min((float) vwidth / (float) dwidth,
                            (float) vheight / (float) dheight);
                }

                dx = Math.round((vwidth - dwidth * scale) * 0.5f);
                dy = Math.round((vheight - dheight * scale) * 0.5f);

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(dx, dy);
            } else {
                // Generate the required transform.
                mTempSrc.set(0, 0, dwidth, dheight);
                mTempDst.set(0, 0, vwidth, vheight);

                mDrawMatrix = mMatrix;
                mDrawMatrix.setRectToRect(mTempSrc, mTempDst, scaleTypeToScaleToFit(mScaleType));
            }
        }
    }
```
函数的主要部分是一大堆的if条件对ScaleType的判断，对相应的ScaleType进行图像的矩阵Matrix处理

### onMeasure()
看完构造函数，接下来开始看绘制相关的内容，首先是第一阶段的onMeasure()
``` java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        resolveUri();
        int w;
        int h;

        // Desired aspect ratio of the view's contents (not including padding)
        float desiredAspect = 0.0f;

        // We are allowed to change the view's width
        boolean resizeWidth = false;

        // We are allowed to change the view's height
        boolean resizeHeight = false;

        final int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);

        if (mDrawable == null) {
            // If no drawable, its intrinsic size is 0.
            mDrawableWidth = -1;
            mDrawableHeight = -1;
            w = h = 0;
        } else {
            w = mDrawableWidth;
            h = mDrawableHeight;
            if (w <= 0) w = 1;
            if (h <= 0) h = 1;

            // We are supposed to adjust view bounds to match the aspect
            // ratio of our drawable. See if that is possible.
            if (mAdjustViewBounds) {
                resizeWidth = widthSpecMode != MeasureSpec.EXACTLY;
                resizeHeight = heightSpecMode != MeasureSpec.EXACTLY;

                desiredAspect = (float) w / (float) h;
            }
        }

        final int pleft = mPaddingLeft;
        final int pright = mPaddingRight;
        final int ptop = mPaddingTop;
        final int pbottom = mPaddingBottom;

        int widthSize;
        int heightSize;

        if (resizeWidth || resizeHeight) {
            /* If we get here, it means we want to resize to match the
                drawables aspect ratio, and we have the freedom to change at
                least one dimension.
            */

            // Get the max possible width given our constraints
            widthSize = resolveAdjustedSize(w + pleft + pright, mMaxWidth, widthMeasureSpec);

            // Get the max possible height given our constraints
            heightSize = resolveAdjustedSize(h + ptop + pbottom, mMaxHeight, heightMeasureSpec);

            if (desiredAspect != 0.0f) {
                // See what our actual aspect ratio is
                final float actualAspect = (float)(widthSize - pleft - pright) /
                                        (heightSize - ptop - pbottom);

                if (Math.abs(actualAspect - desiredAspect) > 0.0000001) {

                    boolean done = false;

                    // Try adjusting width to be proportional to height
                    if (resizeWidth) {
                        int newWidth = (int)(desiredAspect * (heightSize - ptop - pbottom)) +
                                pleft + pright;

                        // Allow the width to outgrow its original estimate if height is fixed.
                        if (!resizeHeight && !sCompatAdjustViewBounds) {
                            widthSize = resolveAdjustedSize(newWidth, mMaxWidth, widthMeasureSpec);
                        }

                        if (newWidth <= widthSize) {
                            widthSize = newWidth;
                            done = true;
                        }
                    }

                    // Try adjusting height to be proportional to width
                    if (!done && resizeHeight) {
                        int newHeight = (int)((widthSize - pleft - pright) / desiredAspect) +
                                ptop + pbottom;

                        // Allow the height to outgrow its original estimate if width is fixed.
                        if (!resizeWidth && !sCompatAdjustViewBounds) {
                            heightSize = resolveAdjustedSize(newHeight, mMaxHeight,
                                    heightMeasureSpec);
                        }

                        if (newHeight <= heightSize) {
                            heightSize = newHeight;
                        }
                    }
                }
            }
        } else {
            /* We are either don't want to preserve the drawables aspect ratio,
               or we are not allowed to change view dimensions. Just measure in
               the normal way.
            */
            w += pleft + pright;
            h += ptop + pbottom;

            w = Math.max(w, getSuggestedMinimumWidth());
            h = Math.max(h, getSuggestedMinimumHeight());

            widthSize = resolveSizeAndState(w, widthMeasureSpec, 0);
            heightSize = resolveSizeAndState(h, heightMeasureSpec, 0);
        }

        setMeasuredDimension(widthSize, heightSize);
    }
```
- 首先会调用一次 resolveUri()，这个函数的作用是，确保在进行onMeasure()时已经有mDrawble，resolveUri()会在未设置mDrawble的情况下通过Uri来获取Drawable
- 通过mDrawable的宽高来设置w和h
- 默认情况下resizeWidth/resizeHeight取值为false，这样只会走最下面的一段else分支，w/h加上对应的padding，然后通过resolveSizeAndState产生相应的
MeasureSpec
- 假如resizeWidth/resizeHeight为true，会走到中间的if代码块中，这个代码块的作用是，根据比例规则重算宽高

### onLayout()
因为ImageView只是普通View的子类，因此没有重写onLayout()

### onDraw()
``` java
    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mDrawable == null) {
            return; // couldn't resolve the URI
        }

        if (mDrawableWidth == 0 || mDrawableHeight == 0) {
            return;     // nothing to draw (empty bounds)
        }

        if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
            mDrawable.draw(canvas);
        } else {
            final int saveCount = canvas.getSaveCount();
            canvas.save();

            if (mCropToPadding) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;
                canvas.clipRect(scrollX + mPaddingLeft, scrollY + mPaddingTop,
                        scrollX + mRight - mLeft - mPaddingRight,
                        scrollY + mBottom - mTop - mPaddingBottom);
            }

            canvas.translate(mPaddingLeft, mPaddingTop);

            if (mDrawMatrix != null) {
                canvas.concat(mDrawMatrix);
            }
            mDrawable.draw(canvas);
            canvas.restoreToCount(saveCount);
        }
    }
```

在onDraw()的源码可以看出，ImageView的绘制，最终是通过Drawable的draw(Canvas canvas)方法完成的。
