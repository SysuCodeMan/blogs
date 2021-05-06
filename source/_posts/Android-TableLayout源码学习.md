---
title: Android TableLayout源码学习
date: 2021-04-16 17:40:50
tags: Android布局
---
### OverView
TableLayout是Android的六大布局容器之一，不过日常开发中使用得并不多，顾名思义，其主要特点是像表格一样，有行和列的概念。

### 重要属性
TableLayout有以下重要属性：
- android:shrinkColumns属性，当TableRow里边的空间布满布局的时候，指定列自动延伸以填充可用部分。当TableRow里边的控件还没有布满布局时，android:shrinkColumns不起作用。
- android:stretchColumns属性，用于指定列对空白部分进行填充
- android:collapseColumns属性，用于隐藏指定的列

### 继承层次及构造函数
``` java
public class TableLayout extends LinearLayout
```
从继承层次上能够看到，TableLayout实际上是继承自LinearLayout的，直观上可以理解成，TableLayout就是一行行的TableRow线性排列而成。

``` java
public TableLayout(Context context, AttributeSet attrs) {
        super(context, attrs);

        TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.TableLayout);

        String stretchedColumns = a.getString(R.styleable.TableLayout_stretchColumns);
        if (stretchedColumns != null) {
            if (stretchedColumns.charAt(0) == '*') {
                mStretchAllColumns = true;
            } else {
                mStretchableColumns = parseColumns(stretchedColumns);
            }
        }

        String shrinkedColumns = a.getString(R.styleable.TableLayout_shrinkColumns);
        if (shrinkedColumns != null) {
            if (shrinkedColumns.charAt(0) == '*') {
                mShrinkAllColumns = true;
            } else {
                mShrinkableColumns = parseColumns(shrinkedColumns);
            }
        }

        String collapsedColumns = a.getString(R.styleable.TableLayout_collapseColumns);
        if (collapsedColumns != null) {
            mCollapsedColumns = parseColumns(collapsedColumns);
        }

        a.recycle();
        initTableLayout();
    }
```
构造函数也不长，主要就是对前文提到过的几个重要属性的分析，看下里面的parseColumns的实现：

``` java
private static SparseBooleanArray parseColumns(String sequence) {
        SparseBooleanArray columns = new SparseBooleanArray();
        Pattern pattern = Pattern.compile("\\s*,\\s*");
        String[] columnDefs = pattern.split(sequence);

        for (String columnIdentifier : columnDefs) {
            try {
                int columnIndex = Integer.parseInt(columnIdentifier);
                // only valid, i.e. positive, columns indexes are handled
                if (columnIndex >= 0) {
                    // putting true in this sparse array indicates that the
                    // column index was defined in the XML file
                    columns.put(columnIndex, true);
                }
            } catch (NumberFormatException e) {
                // we just ignore columns that don't exist
            }
        }

        return columns;
    }
```
在设置shrinkColumns时都是可以设置多个值的，值之间通过逗号分隔，如"0,2"，在parseColumns中就将其中的值解析出来了。

### onMeasure
接下来看下TableLayout的测量过程做了什么：
``` java
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // enforce vertical layout
        measureVertical(widthMeasureSpec, heightMeasureSpec);
    }
```
``` java
@Override
    void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {
        findLargestCells(widthMeasureSpec, heightMeasureSpec);
        shrinkAndStretchColumns(widthMeasureSpec);

        super.measureVertical(widthMeasureSpec, heightMeasureSpec);
    }
```
TableLayout的onMeasure()过程很清晰，分别是自身两个方法findLargestCells()和shrinkAndStretchColumns()的调用，随后再调用父类LinearLayout的measureVertical()方法
先看下findLargestCells()的实现：
``` java
/**
     * <p>Finds the largest cell in each column. For each column, the width of
     * the largest cell is applied to all the other cells.</p>
     *
     * @param widthMeasureSpec the measure constraint imposed by our parent
     */
    private void findLargestCells(int widthMeasureSpec, int heightMeasureSpec) {
        boolean firstRow = true;

        // find the maximum width for each column
        // the total number of columns is dynamically changed if we find
        // wider rows as we go through the children
        // the array is reused for each layout operation; the array can grow
        // but never shrinks. Unused extra cells in the array are just ignored
        // this behavior avoids to unnecessary grow the array after the first
        // layout operation
        final int count = getChildCount();
        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() == GONE) {
                continue;
            }

            if (child instanceof TableRow) {
                final TableRow row = (TableRow) child;
                // forces the row's height
                final ViewGroup.LayoutParams layoutParams = row.getLayoutParams();
                layoutParams.height = LayoutParams.WRAP_CONTENT;

                final int[] widths = row.getColumnsWidths(widthMeasureSpec, heightMeasureSpec);
                final int newLength = widths.length;
                // this is the first row, we just need to copy the values
                if (firstRow) {
                    if (mMaxWidths == null || mMaxWidths.length != newLength) {
                        mMaxWidths = new int[newLength];
                    }
                    System.arraycopy(widths, 0, mMaxWidths, 0, newLength);
                    firstRow = false;
                } else {
                    int length = mMaxWidths.length;
                    final int difference = newLength - length;
                    // the current row is wider than the previous rows, so
                    // we just grow the array and copy the values
                    if (difference > 0) {
                        final int[] oldMaxWidths = mMaxWidths;
                        mMaxWidths = new int[newLength];
                        System.arraycopy(oldMaxWidths, 0, mMaxWidths, 0,
                                oldMaxWidths.length);
                        System.arraycopy(widths, oldMaxWidths.length,
                                mMaxWidths, oldMaxWidths.length, difference);
                    }

                    // the row is narrower or of the same width as the previous
                    // rows, so we find the maximum width for each column
                    // if the row is narrower than the previous ones,
                    // difference will be negative
                    final int[] maxWidths = mMaxWidths;
                    length = Math.min(length, newLength);
                    for (int j = 0; j < length; j++) {
                        maxWidths[j] = Math.max(maxWidths[j], widths[j]);
                    }
                }
            }
        }
    }
```
从方法前的注释可以看出，该方法的作用是，找出每一列最宽的单元格，从方法的具体实现中能看到，最终每一列的最宽宽度会存储到maxWidth这个数组中。

接下来看下为何要找到每一列的最大宽度，答案在shrinkAndStretchColumns()方法中：
``` java
    private void shrinkAndStretchColumns(int widthMeasureSpec) {
        // when we have no row, mMaxWidths is not initialized and the loop
        // below could cause a NPE
        if (mMaxWidths == null) {
            return;
        }

        // should we honor AT_MOST, EXACTLY and UNSPECIFIED?
        int totalWidth = 0;
        for (int width : mMaxWidths) {
            totalWidth += width;
        }

        int size = MeasureSpec.getSize(widthMeasureSpec) - mPaddingLeft - mPaddingRight;

        if ((totalWidth > size) && (mShrinkAllColumns || mShrinkableColumns.size() > 0)) {
            // oops, the largest columns are wider than the row itself
            // fairly redistribute the row's width among the columns
            mutateColumnsWidth(mShrinkableColumns, mShrinkAllColumns, size, totalWidth);
        } else if ((totalWidth < size) && (mStretchAllColumns || mStretchableColumns.size() > 0)) {
            // if we have some space left, we distribute it among the
            // expandable columns
            mutateColumnsWidth(mStretchableColumns, mStretchAllColumns, size, totalWidth);
        }
    }
```
从这个函数中可以看到上述maxWidth的作用：
- 将各列的maxWidth累加得到占据的总宽度
- 根据总宽度和容器宽度的大小关系，分别调用mutateColumnsWidth方法对此前设置的可伸缩列进行宽度调整，以达到设置前文提到的重要属性的设置效果

### onLayout
``` java
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        // enforce horizontal layout
        layoutHorizontal(l, t, r, b);
    }
```
在onLayout()阶段，调用的是LinearLayout中的layoutHorizontal()实现，再一次说明，TableLayout其实就是竖直方向上的LinearLayout。

### 总结
- TableLayout继承自LinearLayout，可以看做是竖直方向的LinearLayout
- TableLayout的测量阶段，主要的工作是，将各列的宽度之和和容器宽度相比，根据大小关系对列进行拉伸调整
