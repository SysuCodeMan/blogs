---
title: View触摸事件传递源码学习
date: 2021-04-13 23:35:57
tags: Android Framework
---
### 起点
事件传递，属于我们人与机器交互的范畴，因此选择交互的载体——Activity作为起点。
在Activity中与触摸事件相关的方法是dispatchTouchEvent(MotionEvent ev)与onTouchEvent(MotionEvent ev)
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
        if (ev.getAction() == MotionEvent.ACTION_DOWN) {
            onUserInteraction();
        }
        if (getWindow().superDispatchTouchEvent(ev)) {
            return true;
        }
        return onTouchEvent(ev);
    }
```
解析
- 判断事件类型，如果是*按下*事件，调用onUserInteraction()方法，这个方法在Activity中是一个空实现，表明我们可以根据需要重写这个方法的实现
- 调用该Activity对应的Window的superDispatchTouchEvent(MotionEvent ev)方法处理事件，如果该方法返回了true，证明事件被消费
- 如果上述Window的方法返回了false，证明事件没有被消费，此时由Activity的onTouchEvent(MotionEvent ev)方法来消费；

### 调用过程
调用Activity的dispatchTouchEvent(MotionEvent ev)方法之后，会调用到对应Window的superDispatchTouchEvent(MotionEvent ev)方法，根据Window类的注释说明，Window类唯一的实现类是PhoneWindow，因此直接到*PhoneWindow*中看其实现：
```java
    @Override
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return mDecor.superDispatchTouchEvent(event);
    }
```
调用的是其mDecor的同名方法，mDecor类型是DecorView，再到*DecorView*中看下它的实现
```java
    public boolean superDispatchTouchEvent(MotionEvent event) {
        return super.dispatchTouchEvent(event);
    }
```
调用的实际上是父类的dispatchTouchEvent(MotionEvent ev)实现，DecorView的父类是FrameLayout，但FrameLayout自身没有实现这个方法，因此调用的是FrameLayout的父类ViewGroup的方法，在ViewGroup中，dispatchTouchEvent(MotionEvent ev)方法是真正的处理事件的地方，在开始这部分源码阅读之前，先小结一下触摸事件是怎么传递到ViewGroup的：
**Activity --> PhoneWindow --> DecorView --> ViewGroup**

### ViewGroup的dispatchTouchEvent(MotionEvent ev)方法
这个方法的实现比较长，主要分成以下阶段来分析：
##### 是否拦截
```java
@Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        if (ev.isTargetAccessibilityFocus() && isAccessibilityFocusedViewOrHost()) {
            ev.setTargetAccessibilityFocus(false);
        }

        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }

            // 关键代码，判断是否拦截
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN
                    || mFirstTouchTarget != null) {
                // 判断子View是否不允许该父类拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }
            } else {
                intercepted = true;
            }
            if (intercepted || mFirstTouchTarget != null) {
                ev.setTargetAccessibilityFocus(false);
            }
            ……
}
```
在这段代码中，最关键的两点：
- 判断disallowIntercept，即子类是否允许父类拦截
- 调用onInterceptTouchEvent(MotionEvent ev)，确定拦截判定为true或者false，可想而知，如果为true的话，该事件将不再向子类分发，在ViewGroup中，onInterceptTouchEvent(MotionEvent ev)是有默认实现的：
```java
public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```
在特定的4种场景下，事件会被拦截

##### 向下分发
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
       ……
            if (!canceled && !intercepted) {
                View childWithAccessibilityFocus = ev.isTargetAccessibilityFocus()
                        ? findChildWithAccessibilityFocus() : null;

                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    final int actionIndex = ev.getActionIndex(); // always 0 for down
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

               
                    removePointersFromTouchTargets(idBitsToAssign);

                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                      // 关键代码1，构建子View遍历列表
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                        final boolean customOrder = preorderedList == null
                                && isChildrenDrawingOrderEnabled();
                        final View[] children = mChildren;
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = getAndVerifyPreorderedIndex(
                                    childrenCount, i, customOrder);
                            final View child = getAndVerifyPreorderedView(
                                    preorderedList, children, childIndex);
                      
                            if (childWithAccessibilityFocus != null) {
                                if (childWithAccessibilityFocus != child) {
                                    continue;
                                }
                                childWithAccessibilityFocus = null;
                                i = childrenCount - 1;
                            }

                            // 关键代码2，两个条件分别是是否可以接收触摸事件和触摸点坐标是否在View内
                            // 如果有其中1个条件不满足，就继续找下一个View；否则说明找到目标子View
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                ev.setTargetAccessibilityFocus(false);
                                continue;
                            }

                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            resetCancelNextUpFlag(child);
                            // 关键代码3，找到目标子View之后调用该方法进行处理
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                mLastTouchDownTime = ev.getDownTime();
                                if (preorderedList != null) {
                                    for (int j = 0; j < childrenCount; j++) {
                                        if (children[childIndex] == mChildren[j]) {
                                            mLastTouchDownIndex = j;
                                            break;
                                        }
                                    }
                                } else {
                                    mLastTouchDownIndex = childIndex;
                                }
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                 // 关键代码4，处理了Action.Down事件的View加入到链表中
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }

                        }
                        if (preorderedList != null) preorderedList.clear();
                    }

                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }
            // 
            if (mFirstTouchTarget == null) {
                // 关键代码5，这时证明还没有子View处理事件，调用该方法来自己处理
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                // 关键代码6，不断从链表中取出子View对应的节点，进行Action.Down以外的事件分发
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }
```
这一段代码比较长，但抓住注释中的6处关键代码即可把握整体流程：
- 构建子View的遍历列表
- 遍历子View，找到可以处理ACTION_DOWN的View
- 调用dispatchTransformedTouchEvent方法将事件分发到子View上进行处理
- 对所有处理了ACTION_DOWN方法的子View加入到一个链表中
- 如果没有子View处理过ACTION_DOWN方法，则调用dispatchTransformTouchEvent来分发给自身来处理
- 如果有子View处理过ACTION_DOWN方法，则不断从链表中取出子View，进行其余正常事件的分发

##### 处理
由上述代码可以得知，无论是找到了子View让其处理，还是没有子View处理而自身处理，都是通过调用dispatchTransformedTouchEvent方法进行的，来看下这个方法的实现：
```java
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;
       
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        if (newPointerIdBits == 0) {
            return false;
        }

       
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

       
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }


        transformedEvent.recycle();
        return handled;
    }
```
这个方法乍一看内容比较多，但其实返回的handled值，归根到底就是一个if-else：
```java
 if (child == null) {
     handled = super.dispatchTouchEvent(event);
 } else {
     handled = child.dispatchTouchEvent(event);
 }
```
因此这段函数要么就是调用子View的dispatchTouchEvent()，要么就是调用父类的dispatchTouchEvent()，而无论是调用子View还是父类，最终都会调用到View类的这个方法实现；因此接下来看下View类中dispatchTouchEvent()的源码：

##### View类的dispatchTouchEvent方法
```java
public boolean dispatchTouchEvent(MotionEvent event) {
      
        if (event.isTargetAccessibilityFocus()) {
            if (!isAccessibilityFocusedViewOrHost()) {
                return false;
            }        
            event.setTargetAccessibilityFocus(false);
        }

        boolean result = false;

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        final int actionMasked = event.getActionMasked();
        if (actionMasked == MotionEvent.ACTION_DOWN) {
           
            stopNestedScroll();
        }

        if (onFilterTouchEventForSecurity(event)) {
            if ((mViewFlags & ENABLED_MASK) == ENABLED && handleScrollBarDragging(event)) {
                result = true;
            }
           
            ListenerInfo li = mListenerInfo;
            // 关键代码1，是否设置了onTouchListener，并且消费
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }
            // 关键代码2，调用自身的onTouchEvent()决定是否消费
            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }

        if (!result && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }

      
        if (actionMasked == MotionEvent.ACTION_UP ||
                actionMasked == MotionEvent.ACTION_CANCEL ||
                (actionMasked == MotionEvent.ACTION_DOWN && !result)) {
            stopNestedScroll();
        }

        return result;
    
}
```
View类的dispatchTouchEvent方法并不长，而且代码核心在于注释的2处：
- 首先看该View是否设置了onTouchListener，调用其onTouch(MotionEvent ev)方法，如果onTouchListener的onTouch方法消费了该方法；则View自身的onTouchEvent(MotionEvent ev)方法将不被调用；
- 如果没有设置onTouchListener或者onTouchListener的onTouch(MotionEvent ev)方法没有消费事件，则调用View自身的onTouchEvent()方法

##### View的onTouchEvent方法
根据上述流程可以得到，如果触摸事件的整个传递过程中，没有被拦截也没有被onTouchListener消费的话，最终将走到View的onTouchEvent(MotionEvent ev)方法中执行，接下来看下View的onTouchEvent方法的实现：
```java
public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
                || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            return clickable;
        }
        // 关键代码1，是否设置了TouchDelegate并且onTouchEvent()方法消费了该事件
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        // 关键代码2，该View是否可以被点击
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    if ((viewFlags & TOOLTIP) == TOOLTIP) {
                        handleTooltipUp();
                    }
                    if (!clickable) {
                        removeTapCallback();
                        removeLongPressCallback();
                        mInContextButtonPress = false;
                        mHasPerformedLongPress = false;
                        mIgnoreNextUpEvent = false;
                        break;
                    }
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            setPressed(true, x, y);
                        }

                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            removeLongPressCallback();
                            if (!focusTaken) {
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    // 关键代码3，进行单击操作
                                    performClickInternal();
                                }
                            }
                        }

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            mUnsetPressedState.run();
                        }

                        removeTapCallback();
                    }
                    mIgnoreNextUpEvent = false;
                    break;

                case MotionEvent.ACTION_DOWN:
                    if (event.getSource() == InputDevice.SOURCE_TOUCHSCREEN) {
                        mPrivateFlags3 |= PFLAG3_FINGER_DOWN;
                    }
                    mHasPerformedLongPress = false;

                    if (!clickable) {
                        checkForLongClick(0, x, y);
                        break;
                    }

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }
                    boolean isInScrollingContainer = isInScrollingContainer();
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        mPendingCheckForTap.x = event.getX();
                        mPendingCheckForTap.y = event.getY();
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        setPressed(true, x, y);
                        // 关键代码4，进行长按检测
                        checkForLongClick(0, x, y);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    if (clickable) {
                        setPressed(false);
                    }
                    removeTapCallback();
                    removeLongPressCallback();
                    mInContextButtonPress = false;
                    mHasPerformedLongPress = false;
                    mIgnoreNextUpEvent = false;
                    mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    break;

                case MotionEvent.ACTION_MOVE:
                    if (clickable) {
                        drawableHotspotChanged(x, y);
                    }
                    if (!pointInView(x, y, mTouchSlop)) {
                        removeTapCallback();
                        removeLongPressCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            setPressed(false);
                        }
                        mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
                    }
                    break;
            }

            return true;
        }

        return false;
    }
```
View的onTouchEvent()方法也并不长，主要是分成DOWN/UP/MOVE/CANCEL这4种case进行处理，其中的几处关键代码分别是：
- 关键代码1会判断是否设置了onTouchDelegate，如果设置了，将优先被delegate处理
- 如果没有delegate或者delegate没有处理，将判断view是否可点击，可点击将走进这个if代码块，注意到这个if代码块最终是return true的
- 关键代码3，在ACTION_UP中进行单击操作performClickInternal()
- 关键代码4，在ACTION_DOWN中进行长按检测，如果检测到是长按，则进行长按回调

##### 单击和长按的处理函数
在View中，处理单击和长按的函数分别是perfromClick()和performLongClickInternal()，最后分别看下这两个函数的实现：
```java
public boolean performClick() {
        notifyAutofillManagerOnClick();

        final boolean result;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnClickListener != null) {
            playSoundEffect(SoundEffectConstants.CLICK);
            // 关键代码，调用监听器的onClick
            li.mOnClickListener.onClick(this);
            result = true;
        } else {
            result = false;
        }

        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_CLICKED);

        notifyEnterOrExitForAutoFillIfNeeded(true);

        return result;
    }
```
```java
private boolean performLongClickInternal(float x, float y) {
        sendAccessibilityEvent(AccessibilityEvent.TYPE_VIEW_LONG_CLICKED);

        boolean handled = false;
        final ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLongClickListener != null) {
            // 关键代码，调用监听器的onLongClick
            handled = li.mOnLongClickListener.onLongClick(View.this);
        }
        if (!handled) {
            final boolean isAnchored = !Float.isNaN(x) && !Float.isNaN(y);
            handled = isAnchored ? showContextMenu(x, y) : showContextMenu();
        }
        if ((mViewFlags & TOOLTIP) == TOOLTIP) {
            if (!handled) {
                handled = showLongClickTooltip((int) x, (int) y);
            }
        }
        if (handled) {
            performHapticFeedback(HapticFeedbackConstants.LONG_PRESS);
        }
        return handled;
    }
```
可以看到，对单击和长按的最终处理，是通过调用相应Listener的对应方法来进行的。
