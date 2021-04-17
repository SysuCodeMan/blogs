---
title: Android TextView源码浅析之顶层流程
date: 2021-04-16 17:52:48
tags: Android源码
---

### OverView
TextView应该是Android中最基础也是使用最广的控件，其能力简单来说就是将一段文本显示出来，但TextView可能也是最复杂的控件之一，涉及到的类众多，我们来采取自顶向下的方式来学习一下TextView的源码，这一篇先从最顶层开始，整体熟悉TextView的工作过程。

### 继承层次
``` java
public class TextView extends View implements ViewTreeObserver.OnPreDrawListener
```
继承层次很简单，是View的直接子类，实现了ViewTreeObserver.OnPreDrawListener接口
而在官方的文档中，能看到其子类有：
![image.png](https://upload-images.jianshu.io/upload_images/5866715-b2cf6bc1d5ff0566.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
基础控件中常用的Button, EditText都是派生自TextView，这也说明了学习TextView的源码是大有裨益的。

### 构造函数
``` java
    public TextView(Context context) {
        this(context, null);
    }

    public TextView(Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, com.android.internal.R.attr.textViewStyle);
    }

    public TextView(Context context, @Nullable AttributeSet attrs, int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }
    public TextView(
        // ^
    }  
```
TextView的构造函数也是模板化的，有1~4个参数的构造函数，单参数的调用二参数然后以此类推，最终都会调用到4参数的构造函数
4参数的构造函数实现较长，分段来看下
``` java
        super(context, attrs, defStyleAttr, defStyleRes);

        // TextView is important by default, unless app developer overrode attribute.
        if (getImportantForAutofill() == IMPORTANT_FOR_AUTOFILL_AUTO) {
            setImportantForAutofill(IMPORTANT_FOR_AUTOFILL_YES);
        }

        setTextInternal("");

        final Resources res = getResources();
        final CompatibilityInfo compat = res.getCompatibilityInfo();

        mTextPaint = new TextPaint(Paint.ANTI_ALIAS_FLAG);
        mTextPaint.density = res.getDisplayMetrics().density;
        mTextPaint.setCompatibilityScaling(compat.applicationScale);

        mHighlightPaint = new Paint(Paint.ANTI_ALIAS_FLAG);
        mHighlightPaint.setCompatibilityScaling(compat.applicationScale);

        mMovement = getDefaultMovementMethod();

        mTransformation = null;

        final TextAppearanceAttributes attributes = new TextAppearanceAttributes();
        attributes.mTextColor = ColorStateList.valueOf(0xFF000000);
        attributes.mTextSize = 15;
        mBreakStrategy = Layout.BREAK_STRATEGY_SIMPLE;
        mHyphenationFrequency = Layout.HYPHENATION_FREQUENCY_NONE;
        mJustificationMode = Layout.JUSTIFICATION_MODE_NONE;

        final Resources.Theme theme = context.getTheme();
```
- 第1部分主要是相关类和变量的初始化
``` java
        TypedArray a = theme.obtainStyledAttributes(attrs,
                com.android.internal.R.styleable.TextViewAppearance, defStyleAttr, defStyleRes);
        TypedArray appearance = null;
        int ap = a.getResourceId(
                com.android.internal.R.styleable.TextViewAppearance_textAppearance, -1);
        a.recycle();
        if (ap != -1) {
            appearance = theme.obtainStyledAttributes(
                    ap, com.android.internal.R.styleable.TextAppearance);
        }
        if (appearance != null) {
            readTextAppearance(context, appearance, attributes, false /* styleArray */);
            attributes.mFontFamilyExplicit = false;
            appearance.recycle();
        }

        boolean editable = getDefaultEditable();
        CharSequence inputMethod = null;
        int numeric = 0;
        CharSequence digits = null;
        boolean phone = false;
        boolean autotext = false;
        int autocap = -1;
        int buffertype = 0;
        boolean selectallonfocus = false;
        Drawable drawableLeft = null, drawableTop = null, drawableRight = null,
                drawableBottom = null, drawableStart = null, drawableEnd = null;
        ColorStateList drawableTint = null;
        PorterDuff.Mode drawableTintMode = null;
        int drawablePadding = 0;
        int ellipsize = ELLIPSIZE_NOT_SET;
        boolean singleLine = false;
        int maxlength = -1;
        CharSequence text = "";
        CharSequence hint = null;
        boolean password = false;
        float autoSizeMinTextSizeInPx = UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE;
        float autoSizeMaxTextSizeInPx = UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE;
        float autoSizeStepGranularityInPx = UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE;
        int inputType = EditorInfo.TYPE_NULL;
        a = theme.obtainStyledAttributes(
                    attrs, com.android.internal.R.styleable.TextView, defStyleAttr, defStyleRes);
        int firstBaselineToTopHeight = -1;
        int lastBaselineToBottomHeight = -1;
        int lineHeight = -1;

        readTextAppearance(context, a, attributes, true /* styleArray */);
```
- 第2部分主要是获取应用的theme等全局属性，来初始化TextView的一些样式相关的变量
``` java
        int n = a.getIndexCount();

        // Must set id in a temporary variable because it will be reset by setText()
        boolean textIsSetFromXml = false;
        for (int i = 0; i < n; i++) {
            int attr = a.getIndex(i);

            switch (attr) {
                case com.android.internal.R.styleable.TextView_editable:
                    editable = a.getBoolean(attr, editable);
                    break;

                case com.android.internal.R.styleable.TextView_inputMethod:
                    inputMethod = a.getText(attr);
                    break;

                case com.android.internal.R.styleable.TextView_numeric:
                    numeric = a.getInt(attr, numeric);
                    break;
                // ……
                case com.android.internal.R.styleable.TextView_lineHeight:
                    lineHeight = a.getDimensionPixelSize(attr, -1);
                    break;
            }
        }

        a.recycle();
```
- 第3部分就是在XML中解析出对应的属性然后赋值到相关变量上
``` java
        final int variation =
                inputType & (EditorInfo.TYPE_MASK_CLASS | EditorInfo.TYPE_MASK_VARIATION);
        final boolean passwordInputType = variation
                == (EditorInfo.TYPE_CLASS_TEXT | EditorInfo.TYPE_TEXT_VARIATION_PASSWORD);
        final boolean webPasswordInputType = variation
                == (EditorInfo.TYPE_CLASS_TEXT | EditorInfo.TYPE_TEXT_VARIATION_WEB_PASSWORD);
        final boolean numberPasswordInputType = variation
                == (EditorInfo.TYPE_CLASS_NUMBER | EditorInfo.TYPE_NUMBER_VARIATION_PASSWORD);

        final int targetSdkVersion = context.getApplicationInfo().targetSdkVersion;
        mUseInternationalizedInput = targetSdkVersion >= VERSION_CODES.O;
        mUseFallbackLineSpacing = targetSdkVersion >= VERSION_CODES.P;

        if (inputMethod != null) {
            Class<?> c;

            try {
                c = Class.forName(inputMethod.toString());
            } catch (ClassNotFoundException ex) {
                throw new RuntimeException(ex);
            }

            try {
                createEditorIfNeeded();
                mEditor.mKeyListener = (KeyListener) c.newInstance();
            } catch (InstantiationException ex) {
                throw new RuntimeException(ex);
            } catch (IllegalAccessException ex) {
                throw new RuntimeException(ex);
            }
            try {
                mEditor.mInputType = inputType != EditorInfo.TYPE_NULL
                        ? inputType
                        : mEditor.mKeyListener.getInputType();
            } catch (IncompatibleClassChangeError e) {
                mEditor.mInputType = EditorInfo.TYPE_CLASS_TEXT;
            }
        } else if (digits != null) {
            createEditorIfNeeded();
            mEditor.mKeyListener = DigitsKeyListener.getInstance(digits.toString());
            // If no input type was specified, we will default to generic
            // text, since we can't tell the IME about the set of digits
            // that was selected.
            mEditor.mInputType = inputType != EditorInfo.TYPE_NULL
                    ? inputType : EditorInfo.TYPE_CLASS_TEXT;
        } else if (inputType != EditorInfo.TYPE_NULL) {
            setInputType(inputType, true);
            // If set, the input type overrides what was set using the deprecated singleLine flag.
            singleLine = !isMultilineInputType(inputType);
        } else if (phone) {
            createEditorIfNeeded();
            mEditor.mKeyListener = DialerKeyListener.getInstance();
            mEditor.mInputType = inputType = EditorInfo.TYPE_CLASS_PHONE;
        } else if (numeric != 0) {
            createEditorIfNeeded();
            mEditor.mKeyListener = DigitsKeyListener.getInstance(
                    null,  // locale
                    (numeric & SIGNED) != 0,
                    (numeric & DECIMAL) != 0);
            inputType = mEditor.mKeyListener.getInputType();
            mEditor.mInputType = inputType;
        } else if (autotext || autocap != -1) {
            TextKeyListener.Capitalize cap;

            inputType = EditorInfo.TYPE_CLASS_TEXT;

            switch (autocap) {
                case 1:
                    cap = TextKeyListener.Capitalize.SENTENCES;
                    inputType |= EditorInfo.TYPE_TEXT_FLAG_CAP_SENTENCES;
                    break;

                case 2:
                    cap = TextKeyListener.Capitalize.WORDS;
                    inputType |= EditorInfo.TYPE_TEXT_FLAG_CAP_WORDS;
                    break;

                case 3:
                    cap = TextKeyListener.Capitalize.CHARACTERS;
                    inputType |= EditorInfo.TYPE_TEXT_FLAG_CAP_CHARACTERS;
                    break;

                default:
                    cap = TextKeyListener.Capitalize.NONE;
                    break;
            }

            createEditorIfNeeded();
            mEditor.mKeyListener = TextKeyListener.getInstance(autotext, cap);
            mEditor.mInputType = inputType;
        } else if (editable) {
            createEditorIfNeeded();
            mEditor.mKeyListener = TextKeyListener.getInstance();
            mEditor.mInputType = EditorInfo.TYPE_CLASS_TEXT;
        } else if (isTextSelectable()) {
            // Prevent text changes from keyboard.
            if (mEditor != null) {
                mEditor.mKeyListener = null;
                mEditor.mInputType = EditorInfo.TYPE_NULL;
            }
            bufferType = BufferType.SPANNABLE;
            // So that selection can be changed using arrow keys and touch is handled.
            setMovementMethod(ArrowKeyMovementMethod.getInstance());
        } else {
            if (mEditor != null) mEditor.mKeyListener = null;

            switch (buffertype) {
                case 0:
                    bufferType = BufferType.NORMAL;
                    break;
                case 1:
                    bufferType = BufferType.SPANNABLE;
                    break;
                case 2:
                    bufferType = BufferType.EDITABLE;
                    break;
            }
        }

        if (mEditor != null) {
            mEditor.adjustInputType(password, passwordInputType, webPasswordInputType,
                    numberPasswordInputType);
        }

        if (selectallonfocus) {
            createEditorIfNeeded();
            mEditor.mSelectAllOnFocus = true;

            if (bufferType == BufferType.NORMAL) {
                bufferType = BufferType.SPANNABLE;
            }
        }
```
- 第4部分是和输入编辑相关的初始化，涉及到的Editor类，负责的功能就是处理文本的区域选择处理和判断、拼写检查、弹出文本菜单等
``` java
        // Set up the tint (if needed) before setting the drawables so that it
        // gets applied correctly.
        if (drawableTint != null || drawableTintMode != null) {
            if (mDrawables == null) {
                mDrawables = new Drawables(context);
            }
            if (drawableTint != null) {
                mDrawables.mTintList = drawableTint;
                mDrawables.mHasTint = true;
            }
            if (drawableTintMode != null) {
                mDrawables.mTintMode = drawableTintMode;
                mDrawables.mHasTintMode = true;
            }
        }

        // This call will save the initial left/right drawables
        setCompoundDrawablesWithIntrinsicBounds(
                drawableLeft, drawableTop, drawableRight, drawableBottom);
        setRelativeDrawablesIfNeeded(drawableStart, drawableEnd);
        setCompoundDrawablePadding(drawablePadding);
```
- 第5部分的工作是，设置对应位置的Drawable
``` java
       // Same as setSingleLine(), but make sure the transformation method and the maximum number
        // of lines of height are unchanged for multi-line TextViews.
        setInputTypeSingleLine(singleLine);
        applySingleLine(singleLine, singleLine, singleLine);

        if (singleLine && getKeyListener() == null && ellipsize == ELLIPSIZE_NOT_SET) {
            ellipsize = ELLIPSIZE_END;
        }

        switch (ellipsize) {
            case ELLIPSIZE_START:
                setEllipsize(TextUtils.TruncateAt.START);
                break;
            case ELLIPSIZE_MIDDLE:
                setEllipsize(TextUtils.TruncateAt.MIDDLE);
                break;
            case ELLIPSIZE_END:
                setEllipsize(TextUtils.TruncateAt.END);
                break;
            case ELLIPSIZE_MARQUEE:
                if (ViewConfiguration.get(context).isFadingMarqueeEnabled()) {
                    setHorizontalFadingEdgeEnabled(true);
                    mMarqueeFadeMode = MARQUEE_FADE_NORMAL;
                } else {
                    setHorizontalFadingEdgeEnabled(false);
                    mMarqueeFadeMode = MARQUEE_FADE_SWITCH_SHOW_ELLIPSIS;
                }
                setEllipsize(TextUtils.TruncateAt.MARQUEE);
                break;
        }

        final boolean isPassword = password || passwordInputType || webPasswordInputType
                || numberPasswordInputType;
        final boolean isMonospaceEnforced = isPassword || (mEditor != null
                && (mEditor.mInputType
                & (EditorInfo.TYPE_MASK_CLASS | EditorInfo.TYPE_MASK_VARIATION))
                == (EditorInfo.TYPE_CLASS_TEXT | EditorInfo.TYPE_TEXT_VARIATION_PASSWORD));
        if (isMonospaceEnforced) {
            attributes.mTypefaceIndex = MONOSPACE;
        }

        applyTextAppearance(attributes);

        if (isPassword) {
            setTransformationMethod(PasswordTransformationMethod.getInstance());
        }

        if (maxlength >= 0) {
            setFilters(new InputFilter[] { new InputFilter.LengthFilter(maxlength) });
        } else {
            setFilters(NO_FILTERS);
        }

        setText(text, bufferType);
        if (textIsSetFromXml) {
            mTextSetFromXmlOrResourceId = true;
        }

        if (hint != null) setHint(hint);
```
- 第6部分，是比较关键的设置文字的部分，先是确定好是否单行，然后设置省略方式，最后调用setText()，如果有必要还会设置hint
```
        /*
         * Views are not normally clickable unless specified to be.
         * However, TextViews that have input or movement methods *are*
         * clickable by default. By setting clickable here, we implicitly set focusable as well
         * if not overridden by the developer.
         */
        a = context.obtainStyledAttributes(
                attrs, com.android.internal.R.styleable.View, defStyleAttr, defStyleRes);
        boolean canInputOrMove = (mMovement != null || getKeyListener() != null);
        boolean clickable = canInputOrMove || isClickable();
        boolean longClickable = canInputOrMove || isLongClickable();
        int focusable = getFocusable();

        n = a.getIndexCount();
        for (int i = 0; i < n; i++) {
            int attr = a.getIndex(i);

            switch (attr) {
                case com.android.internal.R.styleable.View_focusable:
                    TypedValue val = new TypedValue();
                    if (a.getValue(attr, val)) {
                        focusable = (val.type == TypedValue.TYPE_INT_BOOLEAN)
                                ? (val.data == 0 ? NOT_FOCUSABLE : FOCUSABLE)
                                : val.data;
                    }
                    break;

                case com.android.internal.R.styleable.View_clickable:
                    clickable = a.getBoolean(attr, clickable);
                    break;

                case com.android.internal.R.styleable.View_longClickable:
                    longClickable = a.getBoolean(attr, longClickable);
                    break;
            }
        }
        a.recycle();

        // Some apps were relying on the undefined behavior of focusable winning over
        // focusableInTouchMode != focusable in TextViews if both were specified in XML (usually
        // when starting with EditText and setting only focusable=false). To keep those apps from
        // breaking, re-apply the focusable attribute here.
        if (focusable != getFocusable()) {
            setFocusable(focusable);
        }
        setClickable(clickable);
        setLongClickable(longClickable);
```
- 第7部分是设置能否点击的相关属性
``` java
        if (mEditor != null) mEditor.prepareCursorControllers();

        // If not explicitly specified this view is important for accessibility.
        if (getImportantForAccessibility() == IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            setImportantForAccessibility(IMPORTANT_FOR_ACCESSIBILITY_YES);
        }

        if (supportsAutoSizeText()) {
            if (mAutoSizeTextType == AUTO_SIZE_TEXT_TYPE_UNIFORM) {
                // If uniform auto-size has been specified but preset values have not been set then
                // replace the auto-size configuration values that have not been specified with the
                // defaults.
                if (!mHasPresetAutoSizeValues) {
                    final DisplayMetrics displayMetrics = getResources().getDisplayMetrics();

                    if (autoSizeMinTextSizeInPx == UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE) {
                        autoSizeMinTextSizeInPx = TypedValue.applyDimension(
                                TypedValue.COMPLEX_UNIT_SP,
                                DEFAULT_AUTO_SIZE_MIN_TEXT_SIZE_IN_SP,
                                displayMetrics);
                    }

                    if (autoSizeMaxTextSizeInPx == UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE) {
                        autoSizeMaxTextSizeInPx = TypedValue.applyDimension(
                                TypedValue.COMPLEX_UNIT_SP,
                                DEFAULT_AUTO_SIZE_MAX_TEXT_SIZE_IN_SP,
                                displayMetrics);
                    }

                    if (autoSizeStepGranularityInPx
                            == UNSET_AUTO_SIZE_UNIFORM_CONFIGURATION_VALUE) {
                        autoSizeStepGranularityInPx = DEFAULT_AUTO_SIZE_GRANULARITY_IN_PX;
                    }

                    validateAndSetAutoSizeTextTypeUniformConfiguration(autoSizeMinTextSizeInPx,
                            autoSizeMaxTextSizeInPx,
                            autoSizeStepGranularityInPx);
                }

                setupAutoSizeText();
            }
        } else {
            mAutoSizeTextType = AUTO_SIZE_TEXT_TYPE_NONE;
        }

        if (firstBaselineToTopHeight >= 0) {
            setFirstBaselineToTopHeight(firstBaselineToTopHeight);
        }
        if (lastBaselineToBottomHeight >= 0) {
            setLastBaselineToBottomHeight(lastBaselineToBottomHeight);
        }
        if (lineHeight >= 0) {
            setLineHeight(lineHeight);
        }
```
- 最后第8部分是设置AutoSize/BaseLine/LineHeight等外观相关的属性

### onMeasure()
对于onMeasure()源码的解析，同样还是分段来看：
``` java
        int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        int heightMode = MeasureSpec.getMode(heightMeasureSpec);
        int widthSize = MeasureSpec.getSize(widthMeasureSpec);
        int heightSize = MeasureSpec.getSize(heightMeasureSpec);

        int width;
        int height;

        BoringLayout.Metrics boring = UNKNOWN_BORING;
        BoringLayout.Metrics hintBoring = UNKNOWN_BORING;

        if (mTextDir == null) {
            mTextDir = getTextDirectionHeuristic();
        }
```
- 第1部分是相关变量的初始化，这里涉及到一个BoringLayout的概念，在这一层先不去管其实现，知道它的作用是对最简单的文本进行排版
``` java
        int des = -1;
        boolean fromexisting = false;
        final float widthLimit = (widthMode == MeasureSpec.AT_MOST)
                ?  (float) widthSize : Float.MAX_VALUE;

        if (widthMode == MeasureSpec.EXACTLY) {
            // Parent has told us how big to be. So be it.
            width = widthSize;
        } else {
            if (mLayout != null && mEllipsize == null) {
                des = desired(mLayout);
            }

            if (des < 0) {
                boring = BoringLayout.isBoring(mTransformed, mTextPaint, mTextDir, mBoring);
                if (boring != null) {
                    mBoring = boring;
                }
            } else {
                fromexisting = true;
            }

            if (boring == null || boring == UNKNOWN_BORING) {
                if (des < 0) {
                    des = (int) Math.ceil(Layout.getDesiredWidthWithLimit(mTransformed, 0,
                            mTransformed.length(), mTextPaint, mTextDir, widthLimit));
                }
                width = des;
            } else {
                width = boring.width;
            }

            final Drawables dr = mDrawables;
            if (dr != null) {
                width = Math.max(width, dr.mDrawableWidthTop);
                width = Math.max(width, dr.mDrawableWidthBottom);
            }

            if (mHint != null) {
                int hintDes = -1;
                int hintWidth;

                if (mHintLayout != null && mEllipsize == null) {
                    hintDes = desired(mHintLayout);
                }

                if (hintDes < 0) {
                    hintBoring = BoringLayout.isBoring(mHint, mTextPaint, mTextDir, mHintBoring);
                    if (hintBoring != null) {
                        mHintBoring = hintBoring;
                    }
                }

                if (hintBoring == null || hintBoring == UNKNOWN_BORING) {
                    if (hintDes < 0) {
                        hintDes = (int) Math.ceil(Layout.getDesiredWidthWithLimit(mHint, 0,
                                mHint.length(), mTextPaint, mTextDir, widthLimit));
                    }
                    hintWidth = hintDes;
                } else {
                    hintWidth = hintBoring.width;
                }

                if (hintWidth > width) {
                    width = hintWidth;
                }
            }

            width += getCompoundPaddingLeft() + getCompoundPaddingRight();

            if (mMaxWidthMode == EMS) {
                width = Math.min(width, mMaxWidth * getLineHeight());
            } else {
                width = Math.min(width, mMaxWidth);
            }

            if (mMinWidthMode == EMS) {
                width = Math.max(width, mMinWidth * getLineHeight());
            } else {
                width = Math.max(width, mMinWidth);
            }

            // Check against our minimum width
            width = Math.max(width, getSuggestedMinimumWidth());

            if (widthMode == MeasureSpec.AT_MOST) {
                width = Math.min(widthSize, width);
            }
        }

        int want = width - getCompoundPaddingLeft() - getCompoundPaddingRight();
        int unpaddedWidth = want;

        if (mHorizontallyScrolling) want = VERY_WIDE;

        int hintWant = want;
        int hintWidth = (mHintLayout == null) ? hintWant : mHintLayout.getWidth();
```
第2部分是对宽度的计算，如果MeasureMode是EXCACTLY，那直接用父View指定的宽度，否则需要通过Layout来计算，这里再重复一次Layout的概念——Layout负责TextView的排版，包括折行省略等；在计算宽度时，会在正文和hint都会产生一个对应的宽度，取两者最大值作为TextView的宽度
``` java
       if (mLayout == null) {
            makeNewLayout(want, hintWant, boring, hintBoring,
                          width - getCompoundPaddingLeft() - getCompoundPaddingRight(), false);
        } else {
            final boolean layoutChanged = (mLayout.getWidth() != want) || (hintWidth != hintWant)
                    || (mLayout.getEllipsizedWidth()
                            != width - getCompoundPaddingLeft() - getCompoundPaddingRight());

            final boolean widthChanged = (mHint == null) && (mEllipsize == null)
                    && (want > mLayout.getWidth())
                    && (mLayout instanceof BoringLayout
                            || (fromexisting && des >= 0 && des <= want));

            final boolean maximumChanged = (mMaxMode != mOldMaxMode) || (mMaximum != mOldMaximum);

            if (layoutChanged || maximumChanged) {
                if (!maximumChanged && widthChanged) {
                    mLayout.increaseWidthTo(want);
                } else {
                    makeNewLayout(want, hintWant, boring, hintBoring,
                            width - getCompoundPaddingLeft() - getCompoundPaddingRight(), false);
                }
            } else {
                // Nothing has changed
            }
        }
```
- 第3部分也仍然是宽度的相关计算，根据Layout的内容对宽度进行调整，这里涉及到比较关键的方法是makeNewLayout()，关于这个方法，在下一层讲Layout时再展开说。
``` java
        if (heightMode == MeasureSpec.EXACTLY) {
            // Parent has told us how big to be. So be it.
            height = heightSize;
            mDesiredHeightAtMeasure = -1;
        } else {
            int desired = getDesiredHeight();

            height = desired;
            mDesiredHeightAtMeasure = desired;

            if (heightMode == MeasureSpec.AT_MOST) {
                height = Math.min(desired, heightSize);
            }
        }

        int unpaddedHeight = height - getCompoundPaddingTop() - getCompoundPaddingBottom();
        if (mMaxMode == LINES && mLayout.getLineCount() > mMaximum) {
            unpaddedHeight = Math.min(unpaddedHeight, mLayout.getLineTop(mMaximum));
        }
```
- 第4部分是对高度的计算，内容比宽度的处理少很多，主要通过getDesiredHeight()获得高度；篇幅比width少的原因在于，在计算width的时候，实际上已经通过Layout处理完了排版，完成了height所需的部分工作，getDesiredHeight()的内部实现实际上也是通过mLayout来获得高度的
``` java
        /*
         * We didn't let makeNewLayout() register to bring the cursor into view,
         * so do it here if there is any possibility that it is needed.
         */
        if (mMovement != null
                || mLayout.getWidth() > unpaddedWidth
                || mLayout.getHeight() > unpaddedHeight) {
            registerForPreDraw();
        } else {
            scrollTo(0, 0);
        }

        setMeasuredDimension(width, height);
```
- 第5部分是最终width和height的结果处理

### onLayout()
``` java
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        super.onLayout(changed, left, top, right, bottom);
        if (mDeferScroll >= 0) {
            int curs = mDeferScroll;
            mDeferScroll = -1;
            bringPointIntoView(Math.min(curs, mText.length()));
        }
        // Call auto-size after the width and height have been calculated.
        autoSizeText();
    }
```
通常View的直接子类很少重写onLayout()函数，但TextView就重写了，主要工作就是在onLayout()中调用autoSizeText()

### onDraw()
onDraw()函数同样也分段来看
``` java
        restartMarqueeIfNeeded();

        // Draw the background for this view
        super.onDraw(canvas);

        final int compoundPaddingLeft = getCompoundPaddingLeft();
        final int compoundPaddingTop = getCompoundPaddingTop();
        final int compoundPaddingRight = getCompoundPaddingRight();
        final int compoundPaddingBottom = getCompoundPaddingBottom();
        final int scrollX = mScrollX;
        final int scrollY = mScrollY;
        final int right = mRight;
        final int left = mLeft;
        final int bottom = mBottom;
        final int top = mTop;
        final boolean isLayoutRtl = isLayoutRtl();
        final int offset = getHorizontalOffsetForDrawables();
        final int leftOffset = isLayoutRtl ? 0 : offset;
        final int rightOffset = isLayoutRtl ? offset : 0;
```
- 第1部分，是相关变量的初始化
``` java
        final Drawables dr = mDrawables;
        if (dr != null) {
            /*
             * Compound, not extended, because the icon is not clipped
             * if the text height is smaller.
             */

            int vspace = bottom - top - compoundPaddingBottom - compoundPaddingTop;
            int hspace = right - left - compoundPaddingRight - compoundPaddingLeft;

            // IMPORTANT: The coordinates computed are also used in invalidateDrawable()
            // Make sure to update invalidateDrawable() when changing this code.
            if (dr.mShowing[Drawables.LEFT] != null) {
                canvas.save();
                canvas.translate(scrollX + mPaddingLeft + leftOffset,
                        scrollY + compoundPaddingTop + (vspace - dr.mDrawableHeightLeft) / 2);
                dr.mShowing[Drawables.LEFT].draw(canvas);
                canvas.restore();
            }

            // IMPORTANT: The coordinates computed are also used in invalidateDrawable()
            // Make sure to update invalidateDrawable() when changing this code.
            if (dr.mShowing[Drawables.RIGHT] != null) {
                canvas.save();
                canvas.translate(scrollX + right - left - mPaddingRight
                        - dr.mDrawableSizeRight - rightOffset,
                         scrollY + compoundPaddingTop + (vspace - dr.mDrawableHeightRight) / 2);
                dr.mShowing[Drawables.RIGHT].draw(canvas);
                canvas.restore();
            }

            // IMPORTANT: The coordinates computed are also used in invalidateDrawable()
            // Make sure to update invalidateDrawable() when changing this code.
            if (dr.mShowing[Drawables.TOP] != null) {
                canvas.save();
                canvas.translate(scrollX + compoundPaddingLeft
                        + (hspace - dr.mDrawableWidthTop) / 2, scrollY + mPaddingTop);
                dr.mShowing[Drawables.TOP].draw(canvas);
                canvas.restore();
            }

            // IMPORTANT: The coordinates computed are also used in invalidateDrawable()
            // Make sure to update invalidateDrawable() when changing this code.
            if (dr.mShowing[Drawables.BOTTOM] != null) {
                canvas.save();
                canvas.translate(scrollX + compoundPaddingLeft
                        + (hspace - dr.mDrawableWidthBottom) / 2,
                         scrollY + bottom - top - mPaddingBottom - dr.mDrawableSizeBottom);
                dr.mShowing[Drawables.BOTTOM].draw(canvas);
                canvas.restore();
            }
        }
```
- 第2部分，对上下左右的Drawable进行绘制
``` java
        int color = mCurTextColor;

        if (mLayout == null) {
            assumeLayout();
        }

        Layout layout = mLayout;

        if (mHint != null && mText.length() == 0) {
            if (mHintTextColor != null) {
                color = mCurHintTextColor;
            }

            layout = mHintLayout;
        }

        mTextPaint.setColor(color);
        mTextPaint.drawableState = getDrawableState();

        canvas.save();
```
- 第3部分，确定此时绘制的是hint还是正文，设置对应的颜色和Layout
``` java
        int extendedPaddingTop = getExtendedPaddingTop();
        int extendedPaddingBottom = getExtendedPaddingBottom();

        final int vspace = mBottom - mTop - compoundPaddingBottom - compoundPaddingTop;
        final int maxScrollY = mLayout.getHeight() - vspace;

        float clipLeft = compoundPaddingLeft + scrollX;
        float clipTop = (scrollY == 0) ? 0 : extendedPaddingTop + scrollY;
        float clipRight = right - left - getCompoundPaddingRight() + scrollX;
        float clipBottom = bottom - top + scrollY
                - ((scrollY == maxScrollY) ? 0 : extendedPaddingBottom);

        if (mShadowRadius != 0) {
            clipLeft += Math.min(0, mShadowDx - mShadowRadius);
            clipRight += Math.max(0, mShadowDx + mShadowRadius);

            clipTop += Math.min(0, mShadowDy - mShadowRadius);
            clipBottom += Math.max(0, mShadowDy + mShadowRadius);
        }

        canvas.clipRect(clipLeft, clipTop, clipRight, clipBottom);

        int voffsetText = 0;
        int voffsetCursor = 0;

        // translate in by our padding
        /* shortcircuit calling getVerticaOffset() */
        if ((mGravity & Gravity.VERTICAL_GRAVITY_MASK) != Gravity.TOP) {
            voffsetText = getVerticalOffset(false);
            voffsetCursor = getVerticalOffset(true);
        }
        canvas.translate(compoundPaddingLeft, extendedPaddingTop + voffsetText);
```
- 第4部分，是对padding的相关处理，主要是和阴影效果相关
``` java
        final int layoutDirection = getLayoutDirection();
        final int absoluteGravity = Gravity.getAbsoluteGravity(mGravity, layoutDirection);
        if (isMarqueeFadeEnabled()) {
            if (!mSingleLine && getLineCount() == 1 && canMarquee()
                    && (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) != Gravity.LEFT) {
                final int width = mRight - mLeft;
                final int padding = getCompoundPaddingLeft() + getCompoundPaddingRight();
                final float dx = mLayout.getLineRight(0) - (width - padding);
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }

            if (mMarquee != null && mMarquee.isRunning()) {
                final float dx = -mMarquee.getScroll();
                canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            }
        }
```
- 第5部分，主要是走马灯效果的相关处理
``` java
      final int cursorOffsetVertical = voffsetCursor - voffsetText;

        Path highlight = getUpdatedHighlightPath();
        if (mEditor != null) {
            mEditor.onDraw(canvas, layout, highlight, mHighlightPaint, cursorOffsetVertical);
        } else {
            layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
        }

        if (mMarquee != null && mMarquee.shouldDrawGhost()) {
            final float dx = mMarquee.getGhostOffset();
            canvas.translate(layout.getParagraphDirection(0) * dx, 0.0f);
            layout.draw(canvas, highlight, mHighlightPaint, cursorOffsetVertical);
        }

        canvas.restore();
```
- 第6部分，真正绘制文字的地方，实际上是通过mEditor.draw()或者layout.draw()来进行绘制

### 小结
在TextView这一层，整体流程如上文所示，从源码学习的过程中能看到，实际上测量和绘制的逻辑都在Layout/Editor这些类里面，在下一篇我们具体看下它们的实现。
