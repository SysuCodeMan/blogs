---
title: Android AbsoluteLayout源码学习
date: 2021-04-16 17:43:07
tags: Android源码学习
---
### OverView
Android AbsoluteLayout是Android六大布局之一，但目前已经处于Deprecated状态，废弃的原因在于，绝对布局使用的是绝对坐标进行定位，而Android的屏幕大小各种各样，使用绝对坐标必然在兼容性上有极大问题。尽管如此，还是了解一下其实现。（整个类代码也不足300行）

### 继承层次及构造函数
``` java
public class AbsoluteLayout extends ViewGroup
```
AbsoluteLayout也是继承自ViewGroup，在六大布局中，只有TableLayout是继承自LinearLayout，其余均继承自ViewGroup

``` java
    public AbsoluteLayout(Context context) {
        this(context, null);
    }

    public AbsoluteLayout(Context context, AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public AbsoluteLayout(Context context, AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public AbsoluteLayout(Context context, AttributeSet attrs, int defStyleAttr, int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);
    }
```
AbsoluteLayout的4个构造函数，便是最简单的1个入参至4个入参的实现，最终调用到的是父类即ViewGroup的构造方法，没有进行额外的操作。

### AbsoluteLayout.LayoutParams
布局容器一般都有自己对应的LayoutParams，用于实现自己特定支持的布局方法，相比LinearLayout.LayoutParams和RelativeLayout.LayoutParams，AbsoluteLayout.LayoutParams的结构简单很多，因为只要支持x,y坐标即可：
``` java
public static class LayoutParams extends ViewGroup.LayoutParams {
        /**
         * The horizontal, or X, location of the child within the view group.
         */
        @InspectableProperty(name = "layout_x")
        public int x;
        /**
         * The vertical, or Y, location of the child within the view group.
         */
        @InspectableProperty(name = "layout_y")
        public int y;

        public LayoutParams(int width, int height, int x, int y) {
            super(width, height);
            this.x = x;
            this.y = y;
        }
        public LayoutParams(Context c, AttributeSet attrs) {
            super(c, attrs);
            TypedArray a = c.obtainStyledAttributes(attrs,
                    com.android.internal.R.styleable.AbsoluteLayout_Layout);
            x = a.getDimensionPixelOffset(
                    com.android.internal.R.styleable.AbsoluteLayout_Layout_layout_x, 0);
            y = a.getDimensionPixelOffset(
                    com.android.internal.R.styleable.AbsoluteLayout_Layout_layout_y, 0);
            a.recycle();
        }
        public LayoutParams(ViewGroup.LayoutParams source) {
            super(source);
        }

        @Override
        public String debug(String output) {
            return output + "Absolute.LayoutParams={width="
                    + sizeToString(width) + ", height=" + sizeToString(height)
                    + " x=" + x + " y=" + y + "}";
        }
    }
```
上面就是AbsoluteLayout.LayoutParams的全部代码，可以看到其构造函数也就是从xml中解析出x、y的值。

### onMeasure()
接下来看下AbsoluteLayout的onMeasure()函数实现：
``` java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        int maxHeight = 0;
        int maxWidth = 0;

        // Find out how big everyone wants to be
        measureChildren(widthMeasureSpec, heightMeasureSpec);

        // Find rightmost and bottom-most child
        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                int childRight;
                int childBottom;

                AbsoluteLayout.LayoutParams lp
                        = (AbsoluteLayout.LayoutParams) child.getLayoutParams();

                childRight = lp.x + child.getMeasuredWidth();
                childBottom = lp.y + child.getMeasuredHeight();

                maxWidth = Math.max(maxWidth, childRight);
                maxHeight = Math.max(maxHeight, childBottom);
            }
        }

        // Account for padding too
        maxWidth += mPaddingLeft + mPaddingRight;
        maxHeight += mPaddingTop + mPaddingBottom;

        // Check against minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());
        
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, 0),
                resolveSizeAndState(maxHeight, heightMeasureSpec, 0));
    }
```
其中主要的点有：
- 先调用一次measureChildren()，完成了对子View的测量
- 随后的for循环会遍历一次子View，其目的在于，通过子View的x坐标值加上子View的宽，能得到子View最右端的值，同理也可以通过y坐标和高得到最小的坐标值；遍历所有子View得到所有子View中最右和最下的的坐标，即AbsoluteLayout所需要的宽和高
- 最后通过setMeasureDimension把宽高设置进去

### onLayout()
``` java
    @Override
    protected void onLayout(boolean changed, int l, int t,
            int r, int b) {
        int count = getChildCount();

        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {

                AbsoluteLayout.LayoutParams lp =
                        (AbsoluteLayout.LayoutParams) child.getLayoutParams();

                int childLeft = mPaddingLeft + lp.x;
                int childTop = mPaddingTop + lp.y;
                child.layout(childLeft, childTop,
                        childLeft + child.getMeasuredWidth(),
                        childTop + child.getMeasuredHeight());

            }
        }
    }
```
AbsoluteLayout的onLayout()方法则更加直观了，因为本来onLayout()方法传入的left/top/right/bottom参数，就是一种绝对坐标的形式，因此只要拿到x/y坐标，加上相应的宽高，即可得到onLayout所需的参数。

### 总结
- AbsoluteLayout的测量阶段，确定自身宽高的方式是，先测量子View，然后遍历一次子View找到所有子View中最右和最下的坐标，即为父容器自身的宽高。