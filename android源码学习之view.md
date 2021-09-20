Title: Android源码学习之View
Date: 2014-12-20
Category: Android
Tags: Android源码学习 View

#Android源码学习之View
在Android中，View是所有视图控件的基类。Android的视图控件的设计采用了经典的设计模式--组合模式。View是基础控件，而ViewGroup是可以包含子View和管理子View的空间，ViewGroup同时也是一个View。这跟iOS的视图空间设计有很大不同。在iOS的视图框架中，所有视图都是继承UIView,而UIView本身是可以包含和管理子UIView的。我不知道这两种设计有什么优劣，感觉iOS的设计更简单直接。

View的源代码超过2万行，当然包括了很多注释。这里面定义了很多基本的方法，主要是视图渲染的相关方法和事件的相关方法，不过有一点我不是很明白的就是里面有不少是根scrollbar相关的方法。我们实际应用中貌似有scrollbar的控件貌似只有AdapterView的子控件，ScrollView和TextView，scrollbar的特性为什么不直接放到这些类中，或者专门为这些类专门建立一个父类来处理scrollbar。View中的scrollbar特性应该大部分视图控件都不需要吧。当然这些只是我的疑问罢了，Google这样设计也许有他的道理。


##重要方法
一个View要渲染到屏幕上，有几个重要的方法：

> * onMeasure系方法
> * onLayout系方法
> * onDraw系方法

###onMeasure系方法
每一个onXXX方法又有若干个相关XXX方法。例如onMeasure方法，相关的方法是measure(int widthMeasureSpec, int heightMeasureSpec)方法，setMeasuredDimension(int measuredWidth, int measuredHeight)方法，MeasureSpec内部类。其中measure(int widthMeasureSpec, int heightMeasureSpec)是final方法，由系统调用，这个方法会调用onMeasure方法，这个方法需要子类覆盖实现来计算本身的大小，而且覆盖这个方法的时候必须调用setMeasuredDimension方法。这里跟计算大小相关密切的是MeasureSpec内部类，这里要结合ViewGroup类讲解才比较清楚。View中这个相关方法的实现是比较简单的，更多定义一个框架，需要子类自己定制实现。我之前总是搞不太清这些方法的关系，看这篇 [ANDROID自定义视图——onMeasure，MeasureSpec源码 流程 思路详解](http://blog.csdn.net/a396901990/article/details/36475213)才比较清楚。这个我在ViewGroup学习中再讲。简单来讲View通过onMeasure来确定自己的大小。

###onLayout系方法
onLayout(boolean changed, int left, int top, int right, int bottom)方法在View中是一个空实现，子类需要自己覆盖。跟这个类密切相关的方法是layout(int l, int t, int r, int b)方法，这个给外部调用的，一般是父控件来给它设定位置。还有setFrame(int left, int top, int right, int bottom) 和 onSizeChanged(int w, int h, int oldw, int oldh)方法。onSizeChanged(int w, int h, int oldw, int oldh）方法也是一个空方法。onLayout方法我用的比较少，一般貌似在需要动态滚动视图时用这个方法比较多。以后加深学习这个方法。

###onDraw系方法
onDraw方法也是一个空实现。一般来说这个方法用得比较多，大家都知道要自定义View就需要重载这个方法。一开始我很奇怪，既然View的onDraw方法是空实现，而我经常用View来做一些线控件，即给View设置一个背景颜色，宽或者高设置为1像素。那它的背景是怎么画上去的？答案是在draw(Canvas canvas)方法中画的。onDraw系方法还有draw(Canvas canvas, ViewGroup parent, long drawingTime)方法，dispatchDraw(Canvas canvas)方法。dispatchDraw(Canvas canvas)方式是空实现，而draw(Canvas canvas, ViewGroup parent, long drawingTime)的实现相当复杂，这两个方法都是由ViewGroup来调用的。真正的重头戏在draw(Canvas canvas)方法中。


     /**
     * Manually render this view (and all of its children) to the   given Canvas.
     * The view must have already done a full layout before this function is
     * called.  When implementing a view, implement
     * {@link #onDraw(android.graphics.Canvas)} instead of overriding this method.
     * If you do need to override this method, call the superclass version.
     *
     * @param canvas The Canvas to which the View is rendered.
     */
     
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

        /*
         * Draw traversal performs several drawing steps which must be executed
         * in the appropriate order:
         *
         *      1. Draw the background
         *      2. If necessary, save the canvas' layers to prepare for fading
         *      3. Draw view's content
         *      4. Draw children
         *      5. If necessary, draw the fading edges and restore layers
         *      6. Draw decorations (scrollbars for instance)
         */

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

            // Step 6, draw decorations (scrollbars)
            onDrawScrollBars(canvas);

            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);
            }

            // we're done...
            return;
        }

        /*
         * Here we do the full fledged routine...
         * (this is an uncommon case where speed matters less,
         * this is why we repeat some of the tests that have been
         * done above)
         */

        boolean drawTop = false;
        boolean drawBottom = false;
        boolean drawLeft = false;
        boolean drawRight = false;

        float topFadeStrength = 0.0f;
        float bottomFadeStrength = 0.0f;
        float leftFadeStrength = 0.0f;
        float rightFadeStrength = 0.0f;

        // Step 2, save the canvas' layers
        int paddingLeft = mPaddingLeft;

        final boolean offsetRequired = isPaddingOffsetRequired();
        if (offsetRequired) {
            paddingLeft += getLeftPaddingOffset();
        }

        int left = mScrollX + paddingLeft;
        int right = left + mRight - mLeft - mPaddingRight - paddingLeft;
        int top = mScrollY + getFadeTop(offsetRequired);
        int bottom = top + getFadeHeight(offsetRequired);

        if (offsetRequired) {
            right += getRightPaddingOffset();
            bottom += getBottomPaddingOffset();
        }

        final ScrollabilityCache scrollabilityCache = mScrollCache;
        final float fadeHeight = scrollabilityCache.fadingEdgeLength;
        int length = (int) fadeHeight;

        // clip the fade length if top and bottom fades overlap
        // overlapping fades produce odd-looking artifacts
        if (verticalEdges && (top + length > bottom - length)) {
            length = (bottom - top) / 2;
        }

        // also clip horizontal fades if necessary
        if (horizontalEdges && (left + length > right - length)) {
            length = (right - left) / 2;
        }

        if (verticalEdges) {
            topFadeStrength = Math.max(0.0f, Math.min(1.0f, getTopFadingEdgeStrength()));
            drawTop = topFadeStrength * fadeHeight > 1.0f;
            bottomFadeStrength = Math.max(0.0f, Math.min(1.0f, getBottomFadingEdgeStrength()));
            drawBottom = bottomFadeStrength * fadeHeight > 1.0f;
        }

        if (horizontalEdges) {
            leftFadeStrength = Math.max(0.0f, Math.min(1.0f, getLeftFadingEdgeStrength()));
            drawLeft = leftFadeStrength * fadeHeight > 1.0f;
            rightFadeStrength = Math.max(0.0f, Math.min(1.0f, getRightFadingEdgeStrength()));
            drawRight = rightFadeStrength * fadeHeight > 1.0f;
        }

        saveCount = canvas.getSaveCount();

        int solidColor = getSolidColor();
        if (solidColor == 0) {
            final int flags = Canvas.HAS_ALPHA_LAYER_SAVE_FLAG;

            if (drawTop) {
                canvas.saveLayer(left, top, right, top + length, null, flags);
            }

            if (drawBottom) {
                canvas.saveLayer(left, bottom - length, right, bottom, null, flags);
            }

            if (drawLeft) {
                canvas.saveLayer(left, top, left + length, bottom, null, flags);
            }

            if (drawRight) {
                canvas.saveLayer(right - length, top, right, bottom, null, flags);
            }
        } else {
            scrollabilityCache.setFadeColor(solidColor);
        }

        // Step 3, draw the content
        if (!dirtyOpaque) onDraw(canvas);

        // Step 4, draw the children
        dispatchDraw(canvas);

        // Step 5, draw the fade effect and restore layers
        final Paint p = scrollabilityCache.paint;
        final Matrix matrix = scrollabilityCache.matrix;
        final Shader fade = scrollabilityCache.shader;

        if (drawTop) {
            matrix.setScale(1, fadeHeight * topFadeStrength);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, right, top + length, p);
        }

        if (drawBottom) {
            matrix.setScale(1, fadeHeight * bottomFadeStrength);
            matrix.postRotate(180);
            matrix.postTranslate(left, bottom);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, bottom - length, right, bottom, p);
        }

        if (drawLeft) {
            matrix.setScale(1, fadeHeight * leftFadeStrength);
            matrix.postRotate(-90);
            matrix.postTranslate(left, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(left, top, left + length, bottom, p);
        }

        if (drawRight) {
            matrix.setScale(1, fadeHeight * rightFadeStrength);
            matrix.postRotate(90);
            matrix.postTranslate(right, top);
            fade.setLocalMatrix(matrix);
            p.setShader(fade);
            canvas.drawRect(right - length, top, right, bottom, p);
        }

        canvas.restoreToCount(saveCount);

        // Step 6, draw decorations (scrollbars)
        onDrawScrollBars(canvas);

        if (mOverlay != null && !mOverlay.isEmpty()) {
            mOverlay.getOverlayView().dispatchDraw(canvas);
        }
    }
    
    
这个方法中说得很清楚，View的渲染分6步，也可以省略第2和第5步。

    第一步，画背景
    第二步，可能的话（一般是要做动画），把canvas保存起来
    第三步，画内容
    第四步，画子控件
    第五步，还原第二部保存的canvas
    第六步，画装饰（scrollbar）
    
 这样就把整个视图画出来了。
 
##Touch事件
在View中有很大部分是处理事件，包括按键事件，滚动球事件，触摸事件。这里主要讲触摸事件，因为这个用得最多。
这里主要有onTouch方法，dispatchTouch方法。dispatchTouch是个重要的方法，我们的触摸事件都是经过dispatchTouch分发下去的，但是View是子控件它的分发其实就是发送给自己。我看到源码里有这么一段：

      if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null
                    && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                result = true;
            }

            if (!result && onTouchEvent(event)) {
                result = true;
            }
        }
        
这里说明如果你重载了一个View的onTouch方法，并且设置了它的OnTouchListener，那么它将调用OnTouchListener的方法，而不会调用本身的onTouch方法。
而onTouch的实现，主要是焦点处理，点击和安装事件的处理，并没有太多特别的东西。我一般考面试者View的touch事件哪些，从屏幕中点击一个按钮，事件是怎么传递的，相关的方法有哪些。这里ViewGroup有一个拦截事件方法（忘了单词怎么打），是View里没有的，又是怎么工作。这里很考面试者对触摸事件的理解，后续我会说。

View还有一些重要的方法，这里就不一一列举，而且View的方法要结合ViewGroup的实现理解，下次写ViewGroup.