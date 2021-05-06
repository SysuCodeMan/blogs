---
title: Android GridLayout源码学习
date: 2021-04-16 17:38:53
tags: Android布局
---
### OverView
GridLayout是Android4.0引入的网格控件，可以方便地实现网格式布局，减少嵌套层级，这周看下GridLayout具体的工作原理。

### onMeasure
照例从测量阶段开始，看下GridLayout在测量阶段进行了什么操作：
``` java
    @Override
    protected void onMeasure(int widthSpec, int heightSpec) {
        consistencyCheck();

        /** If we have been called by {@link View#measure(int, int)}, one of width or height
         *  is  likely to have changed. We must invalidate if so. */
        invalidateValues();

        int hPadding = getPaddingLeft() + getPaddingRight();
        int vPadding = getPaddingTop()  + getPaddingBottom();

        int widthSpecSansPadding =  adjust( widthSpec, -hPadding);
        int heightSpecSansPadding = adjust(heightSpec, -vPadding);

        measureChildrenWithMargins(widthSpecSansPadding, heightSpecSansPadding, true);

        int widthSansPadding;
        int heightSansPadding;

        // Use the orientation property to decide which axis should be laid out first.
        if (mOrientation == HORIZONTAL) {
            widthSansPadding = mHorizontalAxis.getMeasure(widthSpecSansPadding);
            measureChildrenWithMargins(widthSpecSansPadding, heightSpecSansPadding, false);
            heightSansPadding = mVerticalAxis.getMeasure(heightSpecSansPadding);
        } else {
            heightSansPadding = mVerticalAxis.getMeasure(heightSpecSansPadding);
            measureChildrenWithMargins(widthSpecSansPadding, heightSpecSansPadding, false);
            widthSansPadding = mHorizontalAxis.getMeasure(widthSpecSansPadding);
        }

        int measuredWidth  = Math.max(widthSansPadding  + hPadding, getSuggestedMinimumWidth());
        int measuredHeight = Math.max(heightSansPadding + vPadding, getSuggestedMinimumHeight());

        setMeasuredDimension(
                resolveSizeAndState(measuredWidth,   widthSpec, 0),
                resolveSizeAndState(measuredHeight, heightSpec, 0));
    }
```

GridLayout的onMeasure()方法并不长，可以分成以下3个部分：
- 兼容性检查、参数校验、padding的计算与调整
- 对子View进行第一次测量
- 对子View进行第二次测量

其中对子View的测量是通过调用measureChildrenWithMargins()实现的，看下这个函数的实现：
``` java
private void measureChildrenWithMargins(int widthSpec, int heightSpec, boolean firstPass) {
        for (int i = 0, N = getChildCount(); i < N; i++) {
            View c = getChildAt(i);
            if (c.getVisibility() == View.GONE) continue;
            LayoutParams lp = getLayoutParams(c);
            if (firstPass) {
                measureChildWithMargins2(c, widthSpec, heightSpec, lp.width, lp.height);
            } else {
                boolean horizontal = (mOrientation == HORIZONTAL);
                Spec spec = horizontal ? lp.columnSpec : lp.rowSpec;
                if (spec.getAbsoluteAlignment(horizontal) == FILL) {
                    Interval span = spec.span;
                    Axis axis = horizontal ? mHorizontalAxis : mVerticalAxis;
                    int[] locations = axis.getLocations();
                    int cellSize = locations[span.max] - locations[span.min];
                    int viewSize = cellSize - getTotalMargin(c, horizontal);
                    if (horizontal) {
                        measureChildWithMargins2(c, widthSpec, heightSpec, viewSize, lp.height);
                    } else {
                        measureChildWithMargins2(c, widthSpec, heightSpec, lp.width, viewSize);
                    }
                }
            }
        }
    }
```
在这个函数中，可以看到有1个for循环来遍历子View，然后通过firstPass参数判断是第一轮测量还是第二轮测量，如果是第一轮测量，传入的参数是lp.width/lp.height来对子View进行测量；
关键在于第二轮测量的处理，可以看到如果spec.getAbsoluteAlignment(horizontal) == FILL这个条件不成立的话，第二轮测量实际上是没有对子View进行测量操作的。

我们来看下spec.getAbsoluteAlignment(horizontal) 这个的实现：

``` java
private Alignment getAbsoluteAlignment(boolean horizontal) {
            if (alignment != UNDEFINED_ALIGNMENT) {
                return alignment;
            }
            if (weight == 0f) {
                return horizontal ? START : BASELINE;
            }
            return FILL;
        }
```
可以看到，这个函数的返回结果和alignment与weight的值有关，weight的值就是我们设置给某个单元格的行/列权重，alignment与设置的gravity有关：
``` java
    static Alignment getAlignment(int gravity, boolean horizontal) {
        int mask = horizontal ? HORIZONTAL_GRAVITY_MASK : VERTICAL_GRAVITY_MASK;
        int shift = horizontal ? AXIS_X_SHIFT : AXIS_Y_SHIFT;
        int flags = (gravity & mask) >> shift;
        switch (flags) {
            case (AXIS_SPECIFIED | AXIS_PULL_BEFORE):
                return horizontal ? LEFT : TOP;
            case (AXIS_SPECIFIED | AXIS_PULL_AFTER):
                return horizontal ? RIGHT : BOTTOM;
            case (AXIS_SPECIFIED | AXIS_PULL_BEFORE | AXIS_PULL_AFTER):
                return FILL;
            case AXIS_SPECIFIED:
                return CENTER;
            case (AXIS_SPECIFIED | AXIS_PULL_BEFORE | RELATIVE_LAYOUT_DIRECTION):
                return START;
            case (AXIS_SPECIFIED | AXIS_PULL_AFTER | RELATIVE_LAYOUT_DIRECTION):
                return END;
            default:
                return UNDEFINED_ALIGNMENT;
        }
    }
```
由此可以得出，**第二轮测量，实际上是根据gravity和weight值的设定，将多余的空间再次分配给子单元格**

### onLayout()
``` java
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        consistencyCheck();

        int targetWidth = right - left;
        int targetHeight = bottom - top;

        int paddingLeft = getPaddingLeft();
        int paddingTop = getPaddingTop();
        int paddingRight = getPaddingRight();
        int paddingBottom = getPaddingBottom();

        mHorizontalAxis.layout(targetWidth - paddingLeft - paddingRight);
        mVerticalAxis.layout(targetHeight - paddingTop - paddingBottom);

        int[] hLocations = mHorizontalAxis.getLocations();
        int[] vLocations = mVerticalAxis.getLocations();

        for (int i = 0, N = getChildCount(); i < N; i++) {
            View c = getChildAt(i);
            if (c.getVisibility() == View.GONE) continue;
            LayoutParams lp = getLayoutParams(c);
            Spec columnSpec = lp.columnSpec;
            Spec rowSpec = lp.rowSpec;

            Interval colSpan = columnSpec.span;
            Interval rowSpan = rowSpec.span;

            int x1 = hLocations[colSpan.min];
            int y1 = vLocations[rowSpan.min];

            int x2 = hLocations[colSpan.max];
            int y2 = vLocations[rowSpan.max];

            int cellWidth = x2 - x1;
            int cellHeight = y2 - y1;

            int pWidth = getMeasurement(c, true);
            int pHeight = getMeasurement(c, false);

            Alignment hAlign = columnSpec.getAbsoluteAlignment(true);
            Alignment vAlign = rowSpec.getAbsoluteAlignment(false);

            Bounds boundsX = mHorizontalAxis.getGroupBounds().getValue(i);
            Bounds boundsY = mVerticalAxis.getGroupBounds().getValue(i);

            // Gravity offsets: the location of the alignment group relative to its cell group.
            int gravityOffsetX = hAlign.getGravityOffset(c, cellWidth - boundsX.size(true));
            int gravityOffsetY = vAlign.getGravityOffset(c, cellHeight - boundsY.size(true));

            int leftMargin = getMargin(c, true, true);
            int topMargin = getMargin(c, false, true);
            int rightMargin = getMargin(c, true, false);
            int bottomMargin = getMargin(c, false, false);

            int sumMarginsX = leftMargin + rightMargin;
            int sumMarginsY = topMargin + bottomMargin;

            // Alignment offsets: the location of the view relative to its alignment group.
            int alignmentOffsetX = boundsX.getOffset(this, c, hAlign, pWidth + sumMarginsX, true);
            int alignmentOffsetY = boundsY.getOffset(this, c, vAlign, pHeight + sumMarginsY, false);

            int width = hAlign.getSizeInCell(c, pWidth, cellWidth - sumMarginsX);
            int height = vAlign.getSizeInCell(c, pHeight, cellHeight - sumMarginsY);

            int dx = x1 + gravityOffsetX + alignmentOffsetX;

            int cx = !isLayoutRtl() ? paddingLeft + leftMargin + dx :
                    targetWidth - width - paddingRight - rightMargin - dx;
            int cy = paddingTop + y1 + gravityOffsetY + alignmentOffsetY + topMargin;

            if (width != c.getMeasuredWidth() || height != c.getMeasuredHeight()) {
                c.measure(makeMeasureSpec(width, EXACTLY), makeMeasureSpec(height, EXACTLY));
            }
            c.layout(cx, cy, cx + width, cy + height);
        }
    }
```
GridLayout的onLayout()函数乍看起来有点长，但明确onLayout()阶段目的之后就可以很好地梳理该函数的实现，onLayout()阶段的目的是确定好各个子View的位置，而对于GridLayout来说，子View的位置是通过x行y列这样的方式来设置的，因此只要计算出x行y列相应的坐标即可。

在onLayout()函数中需要特别注意的一点是，如果某个子View算出来的width和测量得到的width不一致时，会将算出的width传入子View再调用1次measure

### onDraw()
GridLayout作为容器布局，也没有重写onDraw()函数。

### 总结
GridLayout的实现相对简单，主要注意几个点：
- 测量阶段会循环遍历2次子View，但第2次循环不一定会对子View进行测量
- 布局阶段有可能还会对子View进行measure()方法的调用。