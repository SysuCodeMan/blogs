---
title: Android TextView源码浅析之Layout
date: 2021-04-16 17:53:03
tags: Android源码
---
### OverView
在上一篇从顶层整体流程分析TextView时能看到Layout这个重要概念，无论是onMeasure()过程还是onDraw()过程，主要工作都是由Layout来完成。
Layout类负责的作用是，完成TextView的排版，包括折行、省略等等。

### TextView.makeNewLayout()
在上一篇的分析中，在onMeasure()过程，会调用到TextView的makeNewLayout()方法，现在来看下它的实现：
```java
    public void makeNewLayout(int wantWidth, int hintWidth,
                                 BoringLayout.Metrics boring,
                                 BoringLayout.Metrics hintBoring,
                                 int ellipsisWidth, boolean bringIntoView) {
        // 暂停走马灯动画
        stopMarquee();

        mOldMaximum = mMaximum;
        mOldMaxMode = mMaxMode;

        mHighlightPathBogus = true;

        if (wantWidth < 0) {
            wantWidth = 0;
        }
        if (hintWidth < 0) {
            hintWidth = 0;
        }

       // 1.获取对齐方式
        Layout.Alignment alignment = getLayoutAlignment();
        final boolean testDirChange = mSingleLine && mLayout != null
                && (alignment == Layout.Alignment.ALIGN_NORMAL
                        || alignment == Layout.Alignment.ALIGN_OPPOSITE);
        int oldDir = 0;
        if (testDirChange) oldDir = mLayout.getParagraphDirection(0);
        boolean shouldEllipsize = mEllipsize != null && getKeyListener() == null;
        // 2.处理省略方式
        final boolean switchEllipsize = mEllipsize == TruncateAt.MARQUEE
                && mMarqueeFadeMode != MARQUEE_FADE_NORMAL;
        TruncateAt effectiveEllipsize = mEllipsize;
        if (mEllipsize == TruncateAt.MARQUEE
                && mMarqueeFadeMode == MARQUEE_FADE_SWITCH_SHOW_ELLIPSIS) {
            effectiveEllipsize = TruncateAt.END_SMALL;
        }
        // 3.获取文字方向
        if (mTextDir == null) {
            mTextDir = getTextDirectionHeuristic();
        }
        // 4.通过上述信息，构建mLayout
        mLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment, shouldEllipsize,
                effectiveEllipsize, effectiveEllipsize == mEllipsize);
        if (switchEllipsize) {
            TruncateAt oppositeEllipsize = effectiveEllipsize == TruncateAt.MARQUEE
                    ? TruncateAt.END : TruncateAt.MARQUEE;
            mSavedMarqueeModeLayout = makeSingleLayout(wantWidth, boring, ellipsisWidth, alignment,
                    shouldEllipsize, oppositeEllipsize, effectiveEllipsize != mEllipsize);
        }

        shouldEllipsize = mEllipsize != null;
        mHintLayout = null;
        // 5.构建出用于显示hint的Layout
        if (mHint != null) {
            if (shouldEllipsize) hintWidth = wantWidth;

            if (hintBoring == UNKNOWN_BORING) {
                hintBoring = BoringLayout.isBoring(mHint, mTextPaint, mTextDir,
                                                   mHintBoring);
                if (hintBoring != null) {
                    mHintBoring = hintBoring;
                }
            }

            if (hintBoring != null) {
                if (hintBoring.width <= hintWidth
                        && (!shouldEllipsize || hintBoring.width <= ellipsisWidth)) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad);
                    }

                    mSavedHintLayout = (BoringLayout) mHintLayout;
                } else if (shouldEllipsize && hintBoring.width <= hintWidth) {
                    if (mSavedHintLayout != null) {
                        mHintLayout = mSavedHintLayout.replaceOrMake(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    } else {
                        mHintLayout = BoringLayout.make(mHint, mTextPaint,
                                hintWidth, alignment, mSpacingMult, mSpacingAdd,
                                hintBoring, mIncludePad, mEllipsize,
                                ellipsisWidth);
                    }
                }
            }
            if (mHintLayout == null) {
                StaticLayout.Builder builder = StaticLayout.Builder.obtain(mHint, 0,
                        mHint.length(), mTextPaint, hintWidth)
                        .setAlignment(alignment)
                        .setTextDirection(mTextDir)
                        .setLineSpacing(mSpacingAdd, mSpacingMult)
                        .setIncludePad(mIncludePad)
                        .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                        .setBreakStrategy(mBreakStrategy)
                        .setHyphenationFrequency(mHyphenationFrequency)
                        .setJustificationMode(mJustificationMode)
                        .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
                if (shouldEllipsize) {
                    builder.setEllipsize(mEllipsize)
                            .setEllipsizedWidth(ellipsisWidth);
                }
                mHintLayout = builder.build();
            }
        }

        if (bringIntoView || (testDirChange && oldDir != mLayout.getParagraphDirection(0))) {
            registerForPreDraw();
        }
        // 6.重启走马灯动画
        if (mEllipsize == TextUtils.TruncateAt.MARQUEE) {
            if (!compressText(ellipsisWidth)) {
                final int height = mLayoutParams.height;
                if (height != LayoutParams.WRAP_CONTENT && height != LayoutParams.MATCH_PARENT) {
                    startMarquee();
                } else {
                    mRestartMarquee = true;
                }
            }
        }

        // CursorControllers need a non-null mLayout
        if (mEditor != null) mEditor.prepareCursorControllers();
    }
```
在这个函数中，最重要的是第4点的通过调用makeSingleLayout()方法构建出主Layout，但第5点构建hint的Layout也很值得学习，因为涉及了BoringLayout和StaticLayout这两个重要的概念：
BoringLayout和StaticLayout都是Layout的子类，其中BoringLayout是指不包含Span而且文本都是左到右而且能够一行展示下的文本，这种情况下使用BoringLayout能够节省不必要的计算；当hint不满足BoringLayout的条件时，会使用StaticLayout来进行布局

### TextView.makeSingleLayout()方法
```java
    protected Layout makeSingleLayout(int wantWidth, BoringLayout.Metrics boring, int ellipsisWidth,
            Layout.Alignment alignment, boolean shouldEllipsize, TruncateAt effectiveEllipsize,
            boolean useSaved) {
        Layout result = null;
        // 这个方法的内部会判断文本是否是Spannable，是的话会使用DynamicLayout
        if (useDynamicLayout()) {
            final DynamicLayout.Builder builder = DynamicLayout.Builder.obtain(mText, mTextPaint,
                    wantWidth)
                    .setDisplayText(mTransformed)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency)
                    .setJustificationMode(mJustificationMode)
                    .setEllipsize(getKeyListener() == null ? effectiveEllipsize : null)
                    .setEllipsizedWidth(ellipsisWidth);
            result = builder.build();
        } else {
            // 判断是否确定了是不是Boring
            if (boring == UNKNOWN_BORING) {
                boring = BoringLayout.isBoring(mTransformed, mTextPaint, mTextDir, mBoring);
                if (boring != null) {
                    mBoring = boring;
                }
            }
            // 这个条件如果成立，则证明是Boring
            if (boring != null) {
                // 根据boring的不同属性，创建对应的Layout，if-else分支里都会尝试从mSavedLayout中创建，拿不到再调用BoringLayout.make()来创建新的
                if (boring.width <= wantWidth
                        && (effectiveEllipsize == null || boring.width <= ellipsisWidth)) {
                    if (useSaved && mSavedLayout != null) {
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    } else {
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad);
                    }

                    if (useSaved) {
                        mSavedLayout = (BoringLayout) result;
                    }
                } else if (shouldEllipsize && boring.width <= wantWidth) {
                    if (useSaved && mSavedLayout != null) {
                        result = mSavedLayout.replaceOrMake(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    } else {
                        result = BoringLayout.make(mTransformed, mTextPaint,
                                wantWidth, alignment, mSpacingMult, mSpacingAdd,
                                boring, mIncludePad, effectiveEllipsize,
                                ellipsisWidth);
                    }
                }
            }
        }
        // 在这里result依然为null，证明不是Boring，需要用StaticLayout
        if (result == null) {
            StaticLayout.Builder builder = StaticLayout.Builder.obtain(mTransformed,
                    0, mTransformed.length(), mTextPaint, wantWidth)
                    .setAlignment(alignment)
                    .setTextDirection(mTextDir)
                    .setLineSpacing(mSpacingAdd, mSpacingMult)
                    .setIncludePad(mIncludePad)
                    .setUseLineSpacingFromFallbacks(mUseFallbackLineSpacing)
                    .setBreakStrategy(mBreakStrategy)
                    .setHyphenationFrequency(mHyphenationFrequency)
                    .setJustificationMode(mJustificationMode)
                    .setMaxLines(mMaxMode == LINES ? mMaximum : Integer.MAX_VALUE);
            if (shouldEllipsize) {
                builder.setEllipsize(effectiveEllipsize)
                        .setEllipsizedWidth(ellipsisWidth);
            }
            result = builder.build();
        }
        return result;
    }
```

### StaticLayout.generate()
在调用StaticLayout.Builder.build()方法之后，最终会调用到StaticLayout的generate()方法来构建出真正对应的Layout，generate()方法很长，还是采取分部分的方式来看：
```java
    final CharSequence source = b.mText;
        final int bufStart = b.mStart;
        final int bufEnd = b.mEnd;
        TextPaint paint = b.mPaint;
        int outerWidth = b.mWidth;
        TextDirectionHeuristic textDir = b.mTextDir;
        final boolean fallbackLineSpacing = b.mFallbackLineSpacing;
        float spacingmult = b.mSpacingMult;
        float spacingadd = b.mSpacingAdd;
        float ellipsizedWidth = b.mEllipsizedWidth;
        TextUtils.TruncateAt ellipsize = b.mEllipsize;
        final boolean addLastLineSpacing = b.mAddLastLineLineSpacing;
        LineBreaks lineBreaks = new LineBreaks(); 
        FloatArray widths = new FloatArray();

        mLineCount = 0;
        mEllipsized = false;
        mMaxLineHeight = mMaximumVisibleLineCount < 1 ? 0 : DEFAULT_MAX_LINE_HEIGHT;

        int v = 0;
        boolean needMultiply = (spacingmult != 1 || spacingadd != 0);

        Paint.FontMetricsInt fm = b.mFontMetricsInt;
        int[] chooseHtv = null;
```
- 第1部分是变量的初始化
```java
    final int[] indents;
        if (mLeftIndents != null || mRightIndents != null) {
            final int leftLen = mLeftIndents == null ? 0 : mLeftIndents.length;
            final int rightLen = mRightIndents == null ? 0 : mRightIndents.length;
            final int indentsLen = Math.max(leftLen, rightLen);
            indents = new int[indentsLen];
            for (int i = 0; i < leftLen; i++) {
                indents[i] = mLeftIndents[i];
            }
            for (int i = 0; i < rightLen; i++) {
                indents[i] += mRightIndents[i];
            }
        } else {
            indents = null;
        }

        final long nativePtr = nInit(
                b.mBreakStrategy, b.mHyphenationFrequency,
                b.mJustificationMode != Layout.JUSTIFICATION_MODE_NONE,
                indents, mLeftPaddings, mRightPaddings);
```
- 第2部分是对文本缩进的处理，最终处理调用的是native函数实现，但实际这个if代码块，暂时还没有找到能够判断为true的地方
```java
        PrecomputedText.ParagraphInfo[] paragraphInfo = null;
        final Spanned spanned = (source instanceof Spanned) ? (Spanned) source : null;
        if (source instanceof PrecomputedText) {
            PrecomputedText precomputed = (PrecomputedText) source;
            if (precomputed.canUseMeasuredResult(bufStart, bufEnd, textDir, paint,
                      b.mBreakStrategy, b.mHyphenationFrequency)) {
                paragraphInfo = precomputed.getParagraphInfo();
            }
        }

        if (paragraphInfo == null) {
            final PrecomputedText.Params param = new PrecomputedText.Params(paint, textDir,
                    b.mBreakStrategy, b.mHyphenationFrequency);
            paragraphInfo = PrecomputedText.createMeasuredParagraphs(source, param, bufStart,
                    bufEnd, false);
        }
```
- 第3部分是分析文本的段落信息，这里涉及到的PrecomputedText是包含了文本测量信息的类，通过它来构建文本能够节省一些开销

接下来的部分，是根据段落信息，逐段地分析段落里的文本内容

```java
       for (int paraIndex = 0; paraIndex < paragraphInfo.length; paraIndex++) {
                final int paraStart = paraIndex == 0
                        ? bufStart : paragraphInfo[paraIndex - 1].paragraphEnd;
                final int paraEnd = paragraphInfo[paraIndex].paragraphEnd;
        // ……
        }
```
下面这几段的代码都发生在这个大的for循环中
```java
                int firstWidthLineCount = 1;
                int firstWidth = outerWidth;
                int restWidth = outerWidth;

                LineHeightSpan[] chooseHt = null;

                if (spanned != null) {
                    LeadingMarginSpan[] sp = getParagraphSpans(spanned, paraStart, paraEnd,
                            LeadingMarginSpan.class);
                    for (int i = 0; i < sp.length; i++) {
                        LeadingMarginSpan lms = sp[i];
                        firstWidth -= sp[i].getLeadingMargin(true);
                        restWidth -= sp[i].getLeadingMargin(false);

                        // LeadingMarginSpan2 is odd.  The count affects all
                        // leading margin spans, not just this particular one
                        if (lms instanceof LeadingMarginSpan2) {
                            LeadingMarginSpan2 lms2 = (LeadingMarginSpan2) lms;
                            firstWidthLineCount = Math.max(firstWidthLineCount,
                                    lms2.getLeadingMarginLineCount());
                        }
                    }

```
  - 首先是宽度方面的分析，解析LeadingMarginSpan和LeadingMargin2
```java
                    chooseHt = getParagraphSpans(spanned, paraStart, paraEnd, LineHeightSpan.class);

                    if (chooseHt.length == 0) {
                        chooseHt = null; // So that out() would not assume it has any contents
                    } else {
                        if (chooseHtv == null || chooseHtv.length < chooseHt.length) {
                            chooseHtv = ArrayUtils.newUnpaddedIntArray(chooseHt.length);
                        }

                        for (int i = 0; i < chooseHt.length; i++) {
                            int o = spanned.getSpanStart(chooseHt[i]);

                            if (o < paraStart) {
                                // starts in this layout, before the
                                // current paragraph

                                chooseHtv[i] = getLineTop(getLineForOffset(o));
                            } else {
                                // starts in this paragraph

                                chooseHtv[i] = v;
                            }
                        }
                    }
                }
```
- 然后是行高的分析，通过解析LineHeightSpan来实现
```
                int[] variableTabStops = null;
                if (spanned != null) {
                    TabStopSpan[] spans = getParagraphSpans(spanned, paraStart,
                            paraEnd, TabStopSpan.class);
                    if (spans.length > 0) {
                        int[] stops = new int[spans.length];
                        for (int i = 0; i < spans.length; i++) {
                            stops[i] = spans[i].getTabStop();
                        }
                        Arrays.sort(stops, 0, stops.length);
                        variableTabStops = stops;
                    }
                }
```
- 这一段通过解析TabStopSpan来获取tabStop，排序后存储在variableTabStop中
```
                final MeasuredParagraph measuredPara = paragraphInfo[paraIndex].measured;
                final char[] chs = measuredPara.getChars();
                final int[] spanEndCache = measuredPara.getSpanEndCache().getRawArray();
                final int[] fmCache = measuredPara.getFontMetrics().getRawArray();
                widths.resize(chs.length);
```
- 这部分作用是从段落信息paragraphInfo[paraIndex]中取出测量相关的信息，后面准备使用

```java
               int breakCount = nComputeLineBreaks(
                        nativePtr,

                        // Inputs
                        chs,
                        measuredPara.getNativePtr(),
                        paraEnd - paraStart,
                        firstWidth,
                        firstWidthLineCount,
                        restWidth,
                        variableTabStops,
                        TAB_INCREMENT,
                        mLineCount,

                        // Outputs
                        lineBreaks,
                        lineBreaks.breaks.length,
                        lineBreaks.breaks,
                        lineBreaks.widths,
                        lineBreaks.ascents,
                        lineBreaks.descents,
                        lineBreaks.flags,
                        widths.getRawArray());

                final int[] breaks = lineBreaks.breaks;
                final float[] lineWidths = lineBreaks.widths;
                final float[] ascents = lineBreaks.ascents;
                final float[] descents = lineBreaks.descents;
                final int[] flags = lineBreaks.flags;
```
- 这一段是根据前述步骤的宽度、行宽等信息，调用native的方法进行了折行处理，处理后的结果在lineBreaks中

```java
                final int remainingLineCount = mMaximumVisibleLineCount - mLineCount;
                final boolean ellipsisMayBeApplied = ellipsize != null
                        && (ellipsize == TextUtils.TruncateAt.END
                            || (mMaximumVisibleLineCount == 1
                                    && ellipsize != TextUtils.TruncateAt.MARQUEE));
                if (0 < remainingLineCount && remainingLineCount < breakCount
                        && ellipsisMayBeApplied) {
                    // Calculate width and flag.
                    float width = 0;
                    int flag = 0; // XXX May need to also have starting hyphen edit
                    for (int i = remainingLineCount - 1; i < breakCount; i++) {
                        if (i == breakCount - 1) {
                            width += lineWidths[i];
                        } else {
                            for (int j = (i == 0 ? 0 : breaks[i - 1]); j < breaks[i]; j++) {
                                width += widths.get(j);
                            }
                        }
                        flag |= flags[i] & TAB_MASK;
                    }
                    // Treat the last line and overflowed lines as a single line.
                    breaks[remainingLineCount - 1] = breaks[breakCount - 1];
                    lineWidths[remainingLineCount - 1] = width;
                    flags[remainingLineCount - 1] = flag;

                    breakCount = remainingLineCount;
                }
```
- 这里主要是对末行以及省略的相应处理

```java
                int here = paraStart;

                int fmTop = 0, fmBottom = 0, fmAscent = 0, fmDescent = 0;
                int fmCacheIndex = 0;
                int spanEndCacheIndex = 0;
                int breakIndex = 0;
                for (int spanStart = paraStart, spanEnd; spanStart < paraEnd; spanStart = spanEnd) {
                    // retrieve end of span
                    spanEnd = spanEndCache[spanEndCacheIndex++];

                    // retrieve cached metrics, order matches above
                    fm.top = fmCache[fmCacheIndex * 4 + 0];
                    fm.bottom = fmCache[fmCacheIndex * 4 + 1];
                    fm.ascent = fmCache[fmCacheIndex * 4 + 2];
                    fm.descent = fmCache[fmCacheIndex * 4 + 3];
                    fmCacheIndex++;

                    if (fm.top < fmTop) {
                        fmTop = fm.top;
                    }
                    if (fm.ascent < fmAscent) {
                        fmAscent = fm.ascent;
                    }
                    if (fm.descent > fmDescent) {
                        fmDescent = fm.descent;
                    }
                    if (fm.bottom > fmBottom) {
                        fmBottom = fm.bottom;
                    }

                    // skip breaks ending before current span range
                    while (breakIndex < breakCount && paraStart + breaks[breakIndex] < spanStart) {
                        breakIndex++;
                    }

                    while (breakIndex < breakCount && paraStart + breaks[breakIndex] <= spanEnd) {
                        int endPos = paraStart + breaks[breakIndex];

                        boolean moreChars = (endPos < bufEnd);

                        final int ascent = fallbackLineSpacing
                                ? Math.min(fmAscent, Math.round(ascents[breakIndex]))
                                : fmAscent;
                        final int descent = fallbackLineSpacing
                                ? Math.max(fmDescent, Math.round(descents[breakIndex]))
                                : fmDescent;
                        v = out(source, here, endPos,
                                ascent, descent, fmTop, fmBottom,
                                v, spacingmult, spacingadd, chooseHt, chooseHtv, fm,
                                flags[breakIndex], needMultiply, measuredPara, bufEnd,
                                includepad, trackpad, addLastLineSpacing, chs, widths.getRawArray(),
                                paraStart, ellipsize, ellipsizedWidth, lineWidths[breakIndex],
                                paint, moreChars);

                        if (endPos < spanEnd) {
                            // preserve metrics for current span
                            fmTop = fm.top;
                            fmBottom = fm.bottom;
                            fmAscent = fm.ascent;
                            fmDescent = fm.descent;
                        } else {
                            fmTop = fmBottom = fmAscent = fmDescent = 0;
                        }

                        here = endPos;
                        breakIndex++;

                        if (mLineCount >= mMaximumVisibleLineCount && mEllipsized) {
                            return;
                        }
                    }
                }

                if (paraEnd == bufEnd) {
                    break;
                }
            }
```
- 这一段是对段落中的Span和这行进行处理，其中fmCache是前面PrecomputeText在测量时已经测算好的各个Span在top/bottom/ascent/descent这几个维度上的值，并缓存在fmCache中，因此在这里需要计算某个段落的字体属性时，直接从fmCache中取出即可

```java
if ((bufEnd == bufStart || source.charAt(bufEnd - 1) == CHAR_NEW_LINE)
                    && mLineCount < mMaximumVisibleLineCount) {
                final MeasuredParagraph measuredPara =
                        MeasuredParagraph.buildForBidi(source, bufEnd, bufEnd, textDir, null);
                paint.getFontMetricsInt(fm);
                v = out(source,
                        bufEnd, bufEnd, fm.ascent, fm.descent,
                        fm.top, fm.bottom,
                        v,
                        spacingmult, spacingadd, null,
                        null, fm, 0,
                        needMultiply, measuredPara, bufEnd,
                        includepad, trackpad, addLastLineSpacing, null,
                        null, bufStart, ellipsize,
                        ellipsizedWidth, 0, paint, false);
            }
```
- 注意最后这一段的成立条件：结束为止和起始位置相等，并且前一个字符是换行符，那证明是一个新的空白段落，也需要作为一个单独的段落进行单独处理


### StaticLayout.out()
上面已经看完了整体的StaticLayout.generate()函数，其中看到有对out()函数的调用，out()函数完整的签名如下：
```java
private int out(final CharSequence text, final int start, final int end, int above, int below,
            int top, int bottom, int v, final float spacingmult, final float spacingadd,
            final LineHeightSpan[] chooseHt, final int[] chooseHtv, final Paint.FontMetricsInt fm,
            final int flags, final boolean needMultiply, @NonNull final MeasuredParagraph measured,
            final int bufEnd, final boolean includePad, final boolean trackPad,
            final boolean addLastLineLineSpacing, final char[] chs, final float[] widths,
            final int widthStart, final TextUtils.TruncateAt ellipsize, final float ellipsisWidth,
            final float textWidth, final TextPaint paint, final boolean moreChars)
```
参数非常多，但看参数名基本能看出和此前的generate()函数基本是对应的
接下来分段看下它做了什么

```java
        final int j = mLineCount;
        final int off = j * mColumns;
        final int want = off + mColumns + TOP;
        int[] lines = mLines;
        final int dir = measured.getParagraphDir();

        if (want >= lines.length) {
            final int[] grow = ArrayUtils.newUnpaddedIntArray(GrowingArrayUtils.growSize(want));
            System.arraycopy(lines, 0, grow, 0, lines.length);
            mLines = grow;
            lines = grow;
        }

        if (j >= mLineDirections.length) {
            final Directions[] grow = ArrayUtils.newUnpaddedArray(Directions.class,
                    GrowingArrayUtils.growSize(j));
            System.arraycopy(mLineDirections, 0, grow, 0, mLineDirections.length);
            mLineDirections = grow;
        }
```
- 第1部分是判断mLines/mLineDirections数组是否需要扩容以及在需要时进行扩容操作，mLines数组中存储的是每行的信息，包括每一行的信息包括START/TOP/DESCENT/HYPHEN/ELLIPSIS_START/ELLIPSIS_COUNT

```java
        if (chooseHt != null) {
            fm.ascent = above;
            fm.descent = below;
            fm.top = top;
            fm.bottom = bottom;

            for (int i = 0; i < chooseHt.length; i++) {
                if (chooseHt[i] instanceof LineHeightSpan.WithDensity) {
                    ((LineHeightSpan.WithDensity) chooseHt[i])
                            .chooseHeight(text, start, end, chooseHtv[i], v, fm, paint);
                } else {
                    chooseHt[i].chooseHeight(text, start, end, chooseHtv[i], v, fm);
                }
            }

            above = fm.ascent;
            below = fm.descent;
            top = fm.top;
            bottom = fm.bottom;
        }
```
第2部分是对高度进行处理，得到的结果存储在above/below/top/bottom变量中

```java
        if (ellipsize != null) {
            // If there is only one line, then do any type of ellipsis except when it is MARQUEE
            // if there are multiple lines, just allow END ellipsis on the last line
            boolean forceEllipsis = moreChars && (mLineCount + 1 == mMaximumVisibleLineCount);

            boolean doEllipsis =
                    (((mMaximumVisibleLineCount == 1 && moreChars) || (firstLine && !moreChars)) &&
                            ellipsize != TextUtils.TruncateAt.MARQUEE) ||
                    (!firstLine && (currentLineIsTheLastVisibleOne || !moreChars) &&
                            ellipsize == TextUtils.TruncateAt.END);
            if (doEllipsis) {
                calculateEllipsis(start, end, widths, widthStart,
                        ellipsisWidth, ellipsize, j,
                        textWidth, paint, forceEllipsis);
            }
        }
```
- 第3部分是对省略的计算和处理

```java
        final boolean lastLine;
        if (mEllipsized) {
            lastLine = true;
        } else {
            final boolean lastCharIsNewLine = widthStart != bufEnd && bufEnd > 0
                    && text.charAt(bufEnd - 1) == CHAR_NEW_LINE;
            if (end == bufEnd && !lastCharIsNewLine) {
                lastLine = true;
            } else if (start == bufEnd && lastCharIsNewLine) {
                lastLine = true;
            } else {
                lastLine = false;
            }
        }

        if (firstLine) {
            if (trackPad) {
                mTopPadding = top - above;
            }

            if (includePad) {
                above = top;
            }
        }

        int extra;

        if (lastLine) {
            if (trackPad) {
                mBottomPadding = bottom - below;
            }

            if (includePad) {
                below = bottom;
            }
        }

        if (needMultiply && (addLastLineLineSpacing || !lastLine)) {
            double ex = (below - above) * (spacingmult - 1) + spacingadd;
            if (ex >= 0) {
                extra = (int)(ex + EXTRA_ROUNDING);
            } else {
                extra = -(int)(-ex + EXTRA_ROUNDING);
            }
        } else {
            extra = 0;
        }
```
- 第4部分是对首行末行的特殊处理，因为要考虑上下留白；还有对行间距的特殊处理

```java
      lines[off + START] = start;
        lines[off + TOP] = v;
        lines[off + DESCENT] = below + extra;
        lines[off + EXTRA] = extra;

        // special case for non-ellipsized last visible line when maxLines is set
        // store the height as if it was ellipsized
        if (!mEllipsized && currentLineIsTheLastVisibleOne) {
            // below calculation as if it was the last line
            int maxLineBelow = includePad ? bottom : below;
            // similar to the calculation of v below, without the extra.
            mMaxLineHeight = v + (maxLineBelow - above);
        }

        v += (below - above) + extra;
        lines[off + mColumns + START] = end;
        lines[off + mColumns + TOP] = v;

        lines[off + TAB] |= flags & TAB_MASK;
        lines[off + HYPHEN] = flags;
        lines[off + DIR] |= dir << DIR_SHIFT;
        mLineDirections[j] = measured.getDirections(start - widthStart, end - widthStart);

        mLineCount++;
        return v;
```
- 第5部分就是将处理完的每一行的信息，都记录到lines数组中了

**从这里就能看出out函数的作用是对Layout中的每一行文本进行分析，最重要的产出就是得到mLines数组**

### Layout.draw()
在上一篇TextView的整体流程中我们看到，TextView的onDraw()执行过程中，最终实际上是通过Layout.draw()函数完成的，而在Layout前面的分析中我们已经得到了每一行的信息，所以接下来看下Layout.draw()函数是怎么执行的。
```java
    public void draw(Canvas canvas, Path highlight, Paint highlightPaint,
            int cursorOffsetVertical) {
        final long lineRange = getLineRangeForDraw(canvas);
        int firstLine = TextUtils.unpackRangeStartFromLong(lineRange);
        int lastLine = TextUtils.unpackRangeEndFromLong(lineRange);
        if (lastLine < 0) return;

        drawBackground(canvas, highlight, highlightPaint, cursorOffsetVertical,
                firstLine, lastLine);
        drawText(canvas, firstLine, lastLine);
    }
```
Layout.draw()实际上分成了两部分，drawBackground()和drawText()，下面看下它们的实现：

### Layout.drawBackground()
```
    public void drawBackground(Canvas canvas, Path highlight, Paint highlightPaint,
            int cursorOffsetVertical, int firstLine, int lastLine) {
        // First, draw LineBackgroundSpans.
        // LineBackgroundSpans know nothing about the alignment, margins, or
        // direction of the layout or line.  XXX: Should they?
        // They are evaluated at each line.
        if (mSpannedText) {
            if (mLineBackgroundSpans == null) {
                mLineBackgroundSpans = new SpanSet<LineBackgroundSpan>(LineBackgroundSpan.class);
            }

            Spanned buffer = (Spanned) mText;
            int textLength = buffer.length();
            mLineBackgroundSpans.init(buffer, 0, textLength);

            if (mLineBackgroundSpans.numberOfSpans > 0) {
                int previousLineBottom = getLineTop(firstLine);
                int previousLineEnd = getLineStart(firstLine);
                ParagraphStyle[] spans = NO_PARA_SPANS;
                int spansLength = 0;
                TextPaint paint = mPaint;
                int spanEnd = 0;
                final int width = mWidth;
                for (int i = firstLine; i <= lastLine; i++) {
                    int start = previousLineEnd;
                    int end = getLineStart(i + 1);
                    previousLineEnd = end;

                    int ltop = previousLineBottom;
                    int lbottom = getLineTop(i + 1);
                    previousLineBottom = lbottom;
                    int lbaseline = lbottom - getLineDescent(i);

                    if (start >= spanEnd) {
                        // These should be infrequent, so we'll use this so that
                        // we don't have to check as often.
                        spanEnd = mLineBackgroundSpans.getNextTransition(start, textLength);
                        // All LineBackgroundSpans on a line contribute to its background.
                        spansLength = 0;
                        // Duplication of the logic of getParagraphSpans
                        if (start != end || start == 0) {
                            // Equivalent to a getSpans(start, end), but filling the 'spans' local
                            // array instead to reduce memory allocation
                            for (int j = 0; j < mLineBackgroundSpans.numberOfSpans; j++) {
                                // equal test is valid since both intervals are not empty by
                                // construction
                                if (mLineBackgroundSpans.spanStarts[j] >= end ||
                                        mLineBackgroundSpans.spanEnds[j] <= start) continue;
                                spans = GrowingArrayUtils.append(
                                        spans, spansLength, mLineBackgroundSpans.spans[j]);
                                spansLength++;
                            }
                        }
                    }

                    for (int n = 0; n < spansLength; n++) {
                        LineBackgroundSpan lineBackgroundSpan = (LineBackgroundSpan) spans[n];
                        lineBackgroundSpan.drawBackground(canvas, paint, 0, width,
                                ltop, lbaseline, lbottom,
                                buffer, start, end, i);
                    }
                }
            }
            mLineBackgroundSpans.recycle();
        }
        // There can be a highlight even without spans if we are drawing
        // a non-spanned transformation of a spanned editing buffer.
        if (highlight != null) {
            if (cursorOffsetVertical != 0) canvas.translate(0, cursorOffsetVertical);
            canvas.drawPath(highlight, highlightPaint);
            if (cursorOffsetVertical != 0) canvas.translate(0, -cursorOffsetVertical);
        }
    }
```
- 代码不长，可以看到Layout的drawBackground()实际上是通过LineBackgroundSpan.drawBackground()来完成的，而LineBackgroundSpan只是一个接口，在Android源码中并没有实现类，因此drawBackground()的实际绘制效果是使用时自己定义的

### Layout.drawText()
![image.png](https://upload-images.jianshu.io/upload_images/5866715-104ca2a82c54aaac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
drawText()方法整体内容如图，主要的工作在for循环中，从注释中能看出，drawText()的过程是，逐行绘制，下面看下这个for循环内部的逻辑：

``` java
            int start = previousLineEnd;
            previousLineEnd = getLineStart(lineNum + 1);
            final boolean justify = isJustificationRequired(lineNum);
            int end = getLineVisibleEnd(lineNum, start, previousLineEnd);
            paint.setHyphenEdit(getHyphen(lineNum));

            int ltop = previousLineBottom;
            int lbottom = getLineTop(lineNum + 1);
            previousLineBottom = lbottom;
            int lbaseline = lbottom - getLineDescent(lineNum);

            int dir = getParagraphDirection(lineNum);
            int left = 0;
            int right = mWidth;

            if (mSpannedText) {
                Spanned sp = (Spanned) buf;
                int textLength = buf.length();
                boolean isFirstParaLine = (start == 0 || buf.charAt(start - 1) == '\n');

                // New batch of paragraph styles, collect into spans array.
                // Compute the alignment, last alignment style wins.
                // Reset tabStops, we'll rebuild if we encounter a line with
                // tabs.
                // We expect paragraph spans to be relatively infrequent, use
                // spanEnd so that we can check less frequently.  Since
                // paragraph styles ought to apply to entire paragraphs, we can
                // just collect the ones present at the start of the paragraph.
                // If spanEnd is before the end of the paragraph, that's not
                // our problem.
                if (start >= spanEnd && (lineNum == firstLine || isFirstParaLine)) {
                    spanEnd = sp.nextSpanTransition(start, textLength,
                                                    ParagraphStyle.class);
                    spans = getParagraphSpans(sp, start, spanEnd, ParagraphStyle.class);

                    paraAlign = mAlignment;
                    for (int n = spans.length - 1; n >= 0; n--) {
                        if (spans[n] instanceof AlignmentSpan) {
                            paraAlign = ((AlignmentSpan) spans[n]).getAlignment();
                            break;
                        }
                    }

                    tabStopsIsInitialized = false;
                }

                // Draw all leading margin spans.  Adjust left or right according
                // to the paragraph direction of the line.
                final int length = spans.length;
                boolean useFirstLineMargin = isFirstParaLine;
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan2) {
                        int count = ((LeadingMarginSpan2) spans[n]).getLeadingMarginLineCount();
                        int startLine = getLineForOffset(sp.getSpanStart(spans[n]));
                        // if there is more than one LeadingMarginSpan2, use
                        // the count that is greatest
                        if (lineNum < startLine + count) {
                            useFirstLineMargin = true;
                            break;
                        }
                    }
                }
                for (int n = 0; n < length; n++) {
                    if (spans[n] instanceof LeadingMarginSpan) {
                        LeadingMarginSpan margin = (LeadingMarginSpan) spans[n];
                        if (dir == DIR_RIGHT_TO_LEFT) {
                            margin.drawLeadingMargin(canvas, paint, right, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            right -= margin.getLeadingMargin(useFirstLineMargin);
                        } else {
                            margin.drawLeadingMargin(canvas, paint, left, dir, ltop,
                                                     lbaseline, lbottom, buf,
                                                     start, end, isFirstParaLine, this);
                            left += margin.getLeadingMargin(useFirstLineMargin);
                        }
                    }
                }
            }
```
- 第1部分，主要是判断是不是Spanned，是的话需要获取影响ParagraphStyle的Span，然后判断其中是否有LeadingMarginSpan，有的话会调用其drawLeadingMargin()方法来绘制其段首的缩进效果

```
            boolean hasTab = getLineContainsTab(lineNum);
            // Can't tell if we have tabs for sure, currently
            if (hasTab && !tabStopsIsInitialized) {
                if (tabStops == null) {
                    tabStops = new TabStops(TAB_INCREMENT, spans);
                } else {
                    tabStops.reset(TAB_INCREMENT, spans);
                }
                tabStopsIsInitialized = true;
            }

            // Determine whether the line aligns to normal, opposite, or center.
            Alignment align = paraAlign;
            if (align == Alignment.ALIGN_LEFT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_NORMAL : Alignment.ALIGN_OPPOSITE;
            } else if (align == Alignment.ALIGN_RIGHT) {
                align = (dir == DIR_LEFT_TO_RIGHT) ?
                    Alignment.ALIGN_OPPOSITE : Alignment.ALIGN_NORMAL;
            }

            int x;
            final int indentWidth;
            if (align == Alignment.ALIGN_NORMAL) {
                if (dir == DIR_LEFT_TO_RIGHT) {
                    indentWidth = getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                    x = left + indentWidth;
                } else {
                    indentWidth = -getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                    x = right - indentWidth;
                }
            } else {
                int max = (int)getLineExtent(lineNum, tabStops, false);
                if (align == Alignment.ALIGN_OPPOSITE) {
                    if (dir == DIR_LEFT_TO_RIGHT) {
                        indentWidth = -getIndentAdjust(lineNum, Alignment.ALIGN_RIGHT);
                        x = right - max - indentWidth;
                    } else {
                        indentWidth = getIndentAdjust(lineNum, Alignment.ALIGN_LEFT);
                        x = left - max + indentWidth;
                    }
                } else { // Alignment.ALIGN_CENTER
                    indentWidth = getIndentAdjust(lineNum, Alignment.ALIGN_CENTER);
                    max = max & ~1;
                    x = ((right + left - max) >> 1) + indentWidth;
                }
            }

            Directions directions = getLineDirections(lineNum);
```
- 第2部分，确定该行中是否有Tab，以及对其方式

``` java
           if (directions == DIRS_ALL_LEFT_TO_RIGHT && !mSpannedText && !hasTab && !justify) {
                // XXX: assumes there's nothing additional to be done
                canvas.drawText(buf, start, end, x, lbaseline, paint);
            } else {
                tl.set(paint, buf, start, end, dir, directions, hasTab, tabStops);
                if (justify) {
                    tl.justify(right - left - indentWidth);
                }
                tl.draw(canvas, x, ltop, lbaseline, lbottom);
            }
```
- 第3部分，根据前面的信息，如果是布局从左到右并且是普通文本而且没有tab而且不需要对齐，则调用canvas.drawText来绘制，否则调用的是TextLine的draw()方法来进行绘制