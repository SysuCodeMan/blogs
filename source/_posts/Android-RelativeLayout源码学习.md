---
title: Android RelativeLayout源码学习
date: 2021-04-13 23:46:10
tags: Android布局
---
### Overview
RelativeLayout是Android中一种常用的布局，通过View之间的相互关系能够方便地确定子View的位置，通过源码的学习来看下Android是如何实现这一功能的。Android中的一个组件，工作原理主要在onMeasure(),onLayout(),onDraw()这3大阶段中。

###onMeasure()
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        if (mDirtyHierarchy) {
            mDirtyHierarchy = false;
            sortChildren();
        }
        // ……
}
```
**测量过程的第一步是对子View进行排序**，在这一步骤会设置一个标志位mDirtyHierarchy进行优化，下面看下sortChildren的具体实现：
```java
private void sortChildren() {
        final int count = getChildCount();
        if (mSortedVerticalChildren == null || mSortedVerticalChildren.length != count) {
            mSortedVerticalChildren = new View[count];
        }

        if (mSortedHorizontalChildren == null || mSortedHorizontalChildren.length != count) {
            mSortedHorizontalChildren = new View[count];
        }

        final DependencyGraph graph = mGraph;
        graph.clear();

        for (int i = 0; i < count; i++) {
            graph.add(getChildAt(i));
        }

        graph.getSortedViews(mSortedVerticalChildren, RULES_VERTICAL);
        graph.getSortedViews(mSortedHorizontalChildren, RULES_HORIZONTAL);
    }
```
这个函数包含以下步骤：
- 初始化两个数组，分别用于记录水平方向和竖直方向的排序结果
- 初始化一个依赖图DependencyGraph，将子View都加入到依赖图中
- 调用DependencyGraph的getSortedViews方法，将两个方向的排序结果记录到对应数组中

下面首先看下DependencyGraph的结构：
```java
private static class DependencyGraph {
        /**
         * List of all views in the graph.
         */
        private ArrayList<Node> mNodes = new ArrayList<Node>();

        /**
         * List of nodes in the graph. Each node is identified by its
         * view id (see View#getId()).
         */
        private SparseArray<Node> mKeyNodes = new SparseArray<Node>();

        /**
         * Temporary data structure used to build the list of roots
         * for this graph.
         */
        private ArrayDeque<Node> mRoots = new ArrayDeque<Node>();
}
```
结果里每个成员代表的含义，其注释已经说清楚了；再具体看下Node的结构：
```java
static class Node {
            /**
             * The view representing this node in the layout.
             */
            View view;

            /**
             * The list of dependents for this node; a dependent is a node
             * that needs this node to be processed first.
             */
            final ArrayMap<Node, DependencyGraph> dependents =
                    new ArrayMap<Node, DependencyGraph>();

            /**
             * The list of dependencies for this node.
             */
            final SparseArray<Node> dependencies = new SparseArray<Node>();
}
```
根据注释也能够明白每个成员的含义

接下来看下DependencyGraph是如何实现getSortedViews方法的：
```java
void getSortedViews(View[] sorted, int... rules) {
            // 找出所有的不依赖其他节点的节点，即根节点
            final ArrayDeque<Node> roots = findRoots(rules);
            int index = 0;

            Node node;
            // 循环从根节点的队尾中取出元素
            while ((node = roots.pollLast()) != null) {
                final View view = node.view;
                final int key = view.getId();
                // 将该节点加入排序结果
                sorted[index++] = view;
                // 取出该节点的所有伴随节点
                final ArrayMap<Node, DependencyGraph> dependents = node.dependents;
                final int count = dependents.size();
                for (int i = 0; i < count; i++) {
                    // 遍历伴随节点
                    final Node dependent = dependents.keyAt(i);
                    // 取出伴随节点的所有依赖节点集合
                    final SparseArray<Node> dependencies = dependent.dependencies;
                    // 将当前节点从伴随节点的依赖节点集合中去掉
                    dependencies.remove(key);
                    // 如果该伴随节点去掉当前节点的依赖之后，没有再依赖其他节点，证明它是新的根节点，加入根节点集合
                    if (dependencies.size() == 0) {
                        roots.add(dependent);
                    }
                }
            }

            if (index < sorted.length) {
                throw new IllegalStateException("Circular dependencies cannot exist"
                        + " in RelativeLayout");
            }
        }
```
这个排序的过程已经在注释中说明，实现过程还是挺明确的。

再看完排序过程，我们回到onMeasure()方法中
```java
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // …… 
        int myWidth = -1;
        int myHeight = -1;

        int width = 0;
        int height = 0;

        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        final int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        final int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        // Record our dimensions if they are known;
        if (widthMode != MeasureSpec.UNSPECIFIED) {
            myWidth = widthSize;
        }

        if (heightMode != MeasureSpec.UNSPECIFIED) {
            myHeight = heightSize;
        }

        if (widthMode == MeasureSpec.EXACTLY) {
            width = myWidth;
        }

        if (heightMode == MeasureSpec.EXACTLY) {
            height = myHeight;
        }

        View ignore = null;
        int gravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
        final boolean horizontalGravity = gravity != Gravity.START && gravity != 0;
        gravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
        final boolean verticalGravity = gravity != Gravity.TOP && gravity != 0;

        int left = Integer.MAX_VALUE;
        int top = Integer.MAX_VALUE;
        int right = Integer.MIN_VALUE;
        int bottom = Integer.MIN_VALUE;

        boolean offsetHorizontalAxis = false;
        boolean offsetVerticalAxis = false;

        if ((horizontalGravity || verticalGravity) && mIgnoreGravity != View.NO_ID) {
            ignore = findViewById(mIgnoreGravity);
        }

        final boolean isWrapContentWidth = widthMode != MeasureSpec.EXACTLY;
        final boolean isWrapContentHeight = heightMode != MeasureSpec.EXACTLY;

        // We need to know our size for doing the correct computation of children positioning in RTL
        // mode but there is no practical way to get it instead of running the code below.
        // So, instead of running the code twice, we just set the width to a "default display width"
        // before the computation and then, as a last pass, we will update their real position with
        // an offset equals to "DEFAULT_WIDTH - width".
        final int layoutDirection = getLayoutDirection();
        if (isLayoutRtl() && myWidth == -1) {
            myWidth = DEFAULT_WIDTH;
        }
        // ……
}
```
中间这段是一些辅助变量的初始化，就不多说了，继续往下看
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        // …… 
        View[] views = mSortedHorizontalChildren;
        int count = views.length;

        for (int i = 0; i < count; i++) {
            View child = views[i];
            if (child.getVisibility() != GONE) {
                LayoutParams params = (LayoutParams) child.getLayoutParams();
                int[] rules = params.getRules(layoutDirection);

                applyHorizontalSizeRules(params, myWidth, rules);
                measureChildHorizontal(child, params, myWidth, myHeight);

                if (positionChildHorizontal(child, params, myWidth, isWrapContentWidth)) {
                    offsetHorizontalAxis = true;
                }
            }
        }

        // ……
}
```
**测量过程的第二步是遍历子View进行水平方向的依赖测量**，这一步通过3个函数进行，逐个看下函数的实现。

介绍函数实现之前先介绍一下rule的概念，在RelativeLayout.LayoutParams进行解析时，会将RelativeLayout中的子View的依赖关系通过rules数组存储起来，数组中的下标代表某条规则，例如LEFT_OF，对应的元素代表这条规则依赖的View id：
```java
switch (attr) {
     case com.android.internal.R.styleable.RelativeLayout_Layout_layout_toLeftOf:
           rules[LEFT_OF] = a.getResourceId(attr, 0);
     break;
                    
}
```
了解了rule的概念之后，看下遍历子View进行依赖测量时相关方法的实现：
```java
private void applyHorizontalSizeRules(LayoutParams childParams, int myWidth, int[] rules) {
        RelativeLayout.LayoutParams anchorParams;

        // VALUE_NOT_SET indicates a "soft requirement" in that direction. For example:
        // left=10, right=VALUE_NOT_SET means the view must start at 10, but can go as far as it
        // wants to the right
        // left=VALUE_NOT_SET, right=10 means the view must end at 10, but can go as far as it
        // wants to the left
        // left=10, right=20 means the left and right ends are both fixed
        childParams.mLeft = VALUE_NOT_SET;
        childParams.mRight = VALUE_NOT_SET;

        anchorParams = getRelatedViewParams(rules, LEFT_OF);
        if (anchorParams != null) {
            childParams.mRight = anchorParams.mLeft - (anchorParams.leftMargin +
                    childParams.rightMargin);
        } else if (childParams.alignWithParent && rules[LEFT_OF] != 0) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }

        anchorParams = getRelatedViewParams(rules, RIGHT_OF);
        if (anchorParams != null) {
            childParams.mLeft = anchorParams.mRight + (anchorParams.rightMargin +
                    childParams.leftMargin);
        } else if (childParams.alignWithParent && rules[RIGHT_OF] != 0) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        anchorParams = getRelatedViewParams(rules, ALIGN_LEFT);
        if (anchorParams != null) {
            childParams.mLeft = anchorParams.mLeft + childParams.leftMargin;
        } else if (childParams.alignWithParent && rules[ALIGN_LEFT] != 0) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        anchorParams = getRelatedViewParams(rules, ALIGN_RIGHT);
        if (anchorParams != null) {
            childParams.mRight = anchorParams.mRight - childParams.rightMargin;
        } else if (childParams.alignWithParent && rules[ALIGN_RIGHT] != 0) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }

        if (0 != rules[ALIGN_PARENT_LEFT]) {
            childParams.mLeft = mPaddingLeft + childParams.leftMargin;
        }

        if (0 != rules[ALIGN_PARENT_RIGHT]) {
            if (myWidth >= 0) {
                childParams.mRight = myWidth - mPaddingRight - childParams.rightMargin;
            }
        }
    }
```
纵观整个函数，其实就是对水平方向的几条规则逐个进行处理，这几条规则分别是：LEFT_OF/RIGHT_OF/ALIGN_LEFT/ALIGN_RIGHT/ALIGN_PARENT_LEFT/ALIGN_PARENT_RIGHT；
对于每个属性的处理方法都是相似的：通过相应的规则从getRelativeViewParams中取到对应依赖的View，然后通过依赖的View的对应属性和padding来调整该View的对应属性
看看下getRelativeViewParams的实现：
```java
private LayoutParams getRelatedViewParams(int[] rules, int relation) {
        View v = getRelatedView(rules, relation);
        if (v != null) {
            ViewGroup.LayoutParams params = v.getLayoutParams();
            if (params instanceof LayoutParams) {
                return (LayoutParams) v.getLayoutParams();
            }
        }
        return null;
    }
```
真正的实现在getRelatedView中
```java
private View getRelatedView(int[] rules, int relation) {
        int id = rules[relation];
        if (id != 0) {
            DependencyGraph.Node node = mGraph.mKeyNodes.get(id);
            if (node == null) return null;
            View v = node.view;

            // Find the first non-GONE view up the chain
            while (v.getVisibility() == View.GONE) {
                rules = ((LayoutParams) v.getLayoutParams()).getRules(v.getLayoutDirection());
                node = mGraph.mKeyNodes.get((rules[relation]));
                // ignore self dependency. for more info look in git commit: da3003
                if (node == null || v == node.view) return null;
                v = node.view;
            }

            return v;
        }

        return null;
    }
```
关键点：
- 通过rules数组取到对应依赖的View id
- 通过View id从依赖图中取出对应的View
- 如果取到的View可见性为GONE，则继续通过对应的依赖规则取依赖的View所依赖的View(有点拗口...)

经过applyHorizontalSizeRules方法的处理之后，子View的params中的left/right值都已经确定好，然后会进入measureChildHorizontal方法

```java
private void measureChildHorizontal(
            View child, LayoutParams params, int myWidth, int myHeight) {
        final int childWidthMeasureSpec = getChildMeasureSpec(params.mLeft, params.mRight,
                params.width, params.leftMargin, params.rightMargin, mPaddingLeft, mPaddingRight,
                myWidth);

        final int childHeightMeasureSpec;
        if (myHeight < 0 && !mAllowBrokenMeasureSpecs) {
            if (params.height >= 0) {
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        params.height, MeasureSpec.EXACTLY);
            } else {
                // Negative values in a mySize/myWidth/myWidth value in
                // RelativeLayout measurement is code for, "we got an
                // unspecified mode in the RelativeLayout's measure spec."
                // Carry it forward.
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(0, MeasureSpec.UNSPECIFIED);
            }
        } else {
            final int maxHeight;
            if (mMeasureVerticalWithPaddingMargin) {
                maxHeight = Math.max(0, myHeight - mPaddingTop - mPaddingBottom
                        - params.topMargin - params.bottomMargin);
            } else {
                maxHeight = Math.max(0, myHeight);
            }

            final int heightMode;
            if (params.height == LayoutParams.MATCH_PARENT) {
                heightMode = MeasureSpec.EXACTLY;
            } else {
                heightMode = MeasureSpec.AT_MOST;
            }
            childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(maxHeight, heightMode);
        }

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
在上一阶段我们确定了子View的left/right参数，在这一阶段，可以通过left/right得到子View的宽度，然后进行第一次子View的measure方法调用，进行第一次测量

接下来调用positionChildHorizontal方法：
```java
private boolean positionChildHorizontal(View child, LayoutParams params, int myWidth,
            boolean wrapContent) {

        final int layoutDirection = getLayoutDirection();
        int[] rules = params.getRules(layoutDirection);

        if (params.mLeft == VALUE_NOT_SET && params.mRight != VALUE_NOT_SET) {
            // Right is fixed, but left varies
            params.mLeft = params.mRight - child.getMeasuredWidth();
        } else if (params.mLeft != VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
            // Left is fixed, but right varies
            params.mRight = params.mLeft + child.getMeasuredWidth();
        } else if (params.mLeft == VALUE_NOT_SET && params.mRight == VALUE_NOT_SET) {
            // Both left and right vary
            if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                if (!wrapContent) {
                    centerHorizontal(child, params, myWidth);
                } else {
                    positionAtEdge(child, params, myWidth);
                }
                return true;
            } else {
                // This is the default case. For RTL we start from the right and for LTR we start
                // from the left. This will give LEFT/TOP for LTR and RIGHT/TOP for RTL.
                positionAtEdge(child, params, myWidth);
            }
        }
        return rules[ALIGN_PARENT_END] != 0;
    }
```
这个方法的代码不长，从实现中可以得知，该方法的作用在于，那些在前阶段没有设置的左右边界约束。

**接下来是第三步，遍历子View进行垂直方向的依赖测量**
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
// ……
    for (int i = 0; i < count; i++) {
            final View child = views[i];
            if (child.getVisibility() != GONE) {
                final LayoutParams params = (LayoutParams) child.getLayoutParams();

                applyVerticalSizeRules(params, myHeight, child.getBaseline());
                measureChild(child, params, myWidth, myHeight);
                if (positionChildVertical(child, params, myHeight, isWrapContentHeight)) {
                    offsetVerticalAxis = true;
                }

                if (isWrapContentWidth) {
                    if (isLayoutRtl()) {
                        if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                            width = Math.max(width, myWidth - params.mLeft);
                        } else {
                            width = Math.max(width, myWidth - params.mLeft + params.leftMargin);
                        }
                    } else {
                        if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                            width = Math.max(width, params.mRight);
                        } else {
                            width = Math.max(width, params.mRight + params.rightMargin);
                        }
                    }
                }

                if (isWrapContentHeight) {
                    if (targetSdkVersion < Build.VERSION_CODES.KITKAT) {
                        height = Math.max(height, params.mBottom);
                    } else {
                        height = Math.max(height, params.mBottom + params.bottomMargin);
                    }
                }

                if (child != ignore || verticalGravity) {
                    left = Math.min(left, params.mLeft - params.leftMargin);
                    top = Math.min(top, params.mTop - params.topMargin);
                }

                if (child != ignore || horizontalGravity) {
                    right = Math.max(right, params.mRight + params.rightMargin);
                    bottom = Math.max(bottom, params.mBottom + params.bottomMargin);
                }
            }
        }
// ……
}
```
垂直方向的约束测量过程和水平方向是一个套路：
- 遍历子View，根据垂直方向的规则进行top/bottom的约束
- 测量一次子View
- 补充边界约束
- 不同的是，在垂直方向的测量阶段，还会对RelativeLayout的最大宽度和最大高度进行计算

**经过水平和垂直方向两轮对子View的分发测量，已经完成了子View的测量过程，接下来是根据是否wrapContent调整RelativeLayout自身的高度**
```java
    if (isWrapContentWidth) {
            // Width already has left padding in it since it was calculated by looking at
            // the right of each child view
            width += mPaddingRight;

            if (mLayoutParams != null && mLayoutParams.width >= 0) {
                width = Math.max(width, mLayoutParams.width);
            }

            width = Math.max(width, getSuggestedMinimumWidth());
            width = resolveSize(width, widthMeasureSpec);

            if (offsetHorizontalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                            centerHorizontal(child, params, width);
                        } else if (rules[ALIGN_PARENT_RIGHT] != 0) {
                            final int childWidth = child.getMeasuredWidth();
                            params.mLeft = width - mPaddingRight - childWidth;
                            params.mRight = params.mLeft + childWidth;
                        }
                    }
                }
            }
        }

        if (isWrapContentHeight) {
            // Height already has top padding in it since it was calculated by looking at
            // the bottom of each child view
            height += mPaddingBottom;

            if (mLayoutParams != null && mLayoutParams.height >= 0) {
                height = Math.max(height, mLayoutParams.height);
            }

            height = Math.max(height, getSuggestedMinimumHeight());
            height = resolveSize(height, heightMeasureSpec);

            if (offsetVerticalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_VERTICAL] != 0) {
                            centerVertical(child, params, height);
                        } else if (rules[ALIGN_PARENT_BOTTOM] != 0) {
                            final int childHeight = child.getMeasuredHeight();
                            params.mTop = height - mPaddingBottom - childHeight;
                            params.mBottom = params.mTop + childHeight;
                        }
                    }
                }
            }
        }
```

**然后再根据Gravity属性调整位置**
```java
    if (isWrapContentWidth) {
            // Width already has left padding in it since it was calculated by looking at
            // the right of each child view
            width += mPaddingRight;

            if (mLayoutParams != null && mLayoutParams.width >= 0) {
                width = Math.max(width, mLayoutParams.width);
            }

            width = Math.max(width, getSuggestedMinimumWidth());
            width = resolveSize(width, widthMeasureSpec);

            if (offsetHorizontalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_HORIZONTAL] != 0) {
                            centerHorizontal(child, params, width);
                        } else if (rules[ALIGN_PARENT_RIGHT] != 0) {
                            final int childWidth = child.getMeasuredWidth();
                            params.mLeft = width - mPaddingRight - childWidth;
                            params.mRight = params.mLeft + childWidth;
                        }
                    }
                }
            }
        }

        if (isWrapContentHeight) {
            // Height already has top padding in it since it was calculated by looking at
            // the bottom of each child view
            height += mPaddingBottom;

            if (mLayoutParams != null && mLayoutParams.height >= 0) {
                height = Math.max(height, mLayoutParams.height);
            }

            height = Math.max(height, getSuggestedMinimumHeight());
            height = resolveSize(height, heightMeasureSpec);

            if (offsetVerticalAxis) {
                for (int i = 0; i < count; i++) {
                    final View child = views[i];
                    if (child.getVisibility() != GONE) {
                        final LayoutParams params = (LayoutParams) child.getLayoutParams();
                        final int[] rules = params.getRules(layoutDirection);
                        if (rules[CENTER_IN_PARENT] != 0 || rules[CENTER_VERTICAL] != 0) {
                            centerVertical(child, params, height);
                        } else if (rules[ALIGN_PARENT_BOTTOM] != 0) {
                            final int childHeight = child.getMeasuredHeight();
                            params.mTop = height - mPaddingBottom - childHeight;
                            params.mBottom = params.mTop + childHeight;
                        }
                    }
                }
            }
        }
```
**最后就是设置自身的宽高，完成测量过程**
```java
      if (isLayoutRtl()) {
            final int offsetWidth = myWidth - width;
            for (int i = 0; i < count; i++) {
                final View child = views[i];
                if (child.getVisibility() != GONE) {
                    final LayoutParams params = (LayoutParams) child.getLayoutParams();
                    params.mLeft -= offsetWidth;
                    params.mRight -= offsetWidth;
                }
            }
        }

        setMeasuredDimension(width, height);
```
上述就是onMeasure的过程。

### onLayout
```java
protected void onLayout(boolean changed, int l, int t, int r, int b) {
        //  The layout has actually already been performed and the positions
        //  cached.  Apply the cached values to the children.
        final int count = getChildCount();

        for (int i = 0; i < count; i++) {
            View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                RelativeLayout.LayoutParams st =
                        (RelativeLayout.LayoutParams) child.getLayoutParams();
                child.layout(st.mLeft, st.mTop, st.mRight, st.mBottom);
            }
        }
    }
```
onLayout的过程很简单，遍历子View，因为子View在分发测量过程中已经确定了上下左右的边界约束，所以直接调用对应的约束值进行layout。

###onDraw
在RelativeLayout中，没有对onDraw进行重写。

### 总结
和大多数容器布局类似，RelativeLayout的核心在于测量阶段，需要遍历两次子View，原因是水平和垂直方向各需一次来确定相应方向的边界约束。
