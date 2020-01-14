```
mRunPredictiveAnimations
```

### RecyclerView源码解析

1、构造方法 

三个构造方法最终都是调用的三参的构造方法。

```java
public RecyclerView(@NonNull Context context) {
    this(context, null);
}

public RecyclerView(@NonNull Context context, @Nullable AttributeSet attrs) {
    this(context, attrs, 0);
}

public RecyclerView(@NonNull Context context, @Nullable AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        if (attrs != null) {
            TypedArray a = context.obtainStyledAttributes(attrs, CLIP_TO_PADDING_ATTR, defStyle, 0);//加载资源文件中的样式
            mClipToPadding = a.getBoolean(0, true);
            a.recycle();
        } else {
            mClipToPadding = true;
        }
        setScrollContainer(true);//设置可滚动
        setFocusableInTouchMode(true);//设置可以获取焦点

        final ViewConfiguration vc = ViewConfiguration.get(context);//获取vie的配置
        mTouchSlop = vc.getScaledTouchSlop();
        mScaledHorizontalScrollFactor =
                ViewConfigurationCompat.getScaledHorizontalScrollFactor(vc, context);
        mScaledVerticalScrollFactor =
                ViewConfigurationCompat.getScaledVerticalScrollFactor(vc, context);
        mMinFlingVelocity = vc.getScaledMinimumFlingVelocity();
        mMaxFlingVelocity = vc.getScaledMaximumFlingVelocity();
        setWillNotDraw(getOverScrollMode() == View.OVER_SCROLL_NEVER);

        mItemAnimator.setListener(mItemAnimatorListener);//设置动画效果
        initAdapterManager();//初始化adapterManager 调用01
        initChildrenHelper();//初始化子view的高  调用02
        initAutofill();//初始化自动填充 调用03
        // If not explicitly specified this view is important for accessibility.
        if (ViewCompat.getImportantForAccessibility(this)
                == ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_AUTO) {
            ViewCompat.setImportantForAccessibility(this,
                    ViewCompat.IMPORTANT_FOR_ACCESSIBILITY_YES);
        }
        mAccessibilityManager = (AccessibilityManager) getContext()
                .getSystemService(Context.ACCESSIBILITY_SERVICE);
        setAccessibilityDelegateCompat(new RecyclerViewAccessibilityDelegate(this));
        // Create the layoutManager if specified.

        boolean nestedScrollingEnabled = true;

        if (attrs != null) {
            int defStyleRes = 0;
            TypedArray a = context.obtainStyledAttributes(attrs, R.styleable.RecyclerView,
                    defStyle, defStyleRes);
            String layoutManagerName = a.getString(R.styleable.RecyclerView_layoutManager);
            int descendantFocusability = a.getInt(
                    R.styleable.RecyclerView_android_descendantFocusability, -1);
            if (descendantFocusability == -1) {
                setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
            }
            mEnableFastScroller = a.getBoolean(R.styleable.RecyclerView_fastScrollEnabled, false);
            if (mEnableFastScroller) {
                StateListDrawable verticalThumbDrawable = (StateListDrawable) a
                        .getDrawable(R.styleable.RecyclerView_fastScrollVerticalThumbDrawable);
                Drawable verticalTrackDrawable = a
                        .getDrawable(R.styleable.RecyclerView_fastScrollVerticalTrackDrawable);
                StateListDrawable horizontalThumbDrawable = (StateListDrawable) a
                        .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalThumbDrawable);
                Drawable horizontalTrackDrawable = a
                        .getDrawable(R.styleable.RecyclerView_fastScrollHorizontalTrackDrawable);
                initFastScroller(verticalThumbDrawable, verticalTrackDrawable,
                        horizontalThumbDrawable, horizontalTrackDrawable);
            }
            a.recycle();
            createLayoutManager(context, layoutManagerName, attrs, defStyle, defStyleRes);

            if (Build.VERSION.SDK_INT >= 21) {
                a = context.obtainStyledAttributes(attrs, NESTED_SCROLLING_ATTRS,
                        defStyle, defStyleRes);
                nestedScrollingEnabled = a.getBoolean(0, true);
                a.recycle();
            }
        } else {
            setDescendantFocusability(ViewGroup.FOCUS_AFTER_DESCENDANTS);
        }

        // Re-set whether nested scrolling is enabled so that it is set on all API levels
        setNestedScrollingEnabled(nestedScrollingEnabled);
    }



```




```java
//01 
void initAdapterManager() {
    //adapterHelper 维护adapter的状态等。后面再细讲
        mAdapterHelper = new AdapterHelper(new AdapterHelper.Callback() {
            @Override
            public ViewHolder findViewHolder(int position) {
                //根据position获取到viewHolder
                final ViewHolder vh = findViewHolderForPosition(position, true);
                if (vh == null) {
                    return null;
                }
                // ensure it is not hidden because for adapter helper, the only thing matter is that
                // LM thinks view is a child.
                if (mChildHelper.isHidden(vh.itemView)) {//判断该View是否显示在屏幕 调用 01-01
                    if (DEBUG) {
                        Log.d(TAG, "assuming view holder cannot be find because it is hidden");
                    }
                    return null;
                }
                return vh;
            }
@Override
        public void offsetPositionsForRemovingInvisible(int start, int count) {
            offsetPositionRecordsForRemove(start, count, true);
            mItemsAddedOrRemoved = true;
            mState.mDeletedInvisibleItemCountSincePreviousLayout += count;
        }

        @Override
        public void offsetPositionsForRemovingLaidOutOrNewView(
                int positionStart, int itemCount) {
            offsetPositionRecordsForRemove(positionStart, itemCount, false);
            mItemsAddedOrRemoved = true;
        }       
@Override
        public void markViewHoldersUpdated(int positionStart, int itemCount, Object payload) {
            viewRangeUpdate(positionStart, itemCount, payload);
            mItemsChanged = true;
        }

        @Override
        public void onDispatchFirstPass(AdapterHelper.UpdateOp op) {
            dispatchUpdate(op);
        }

        void dispatchUpdate(AdapterHelper.UpdateOp op) {
            switch (op.cmd) {
                case AdapterHelper.UpdateOp.ADD:
                    mLayout.onItemsAdded(RecyclerView.this, op.positionStart, op.itemCount);
                    break;
                case AdapterHelper.UpdateOp.REMOVE:
                    mLayout.onItemsRemoved(RecyclerView.this, op.positionStart, op.itemCount);
                    break;
                case AdapterHelper.UpdateOp.UPDATE:
                    mLayout.onItemsUpdated(RecyclerView.this, op.positionStart, op.itemCount,
                            op.payload);
                    break;
                case AdapterHelper.UpdateOp.MOVE:
                    mLayout.onItemsMoved(RecyclerView.this, op.positionStart, op.itemCount, 1);
                    break;
            }
        }

        @Override
        public void onDispatchSecondPass(AdapterHelper.UpdateOp op) {
            dispatchUpdate(op);
        }

        @Override
        public void offsetPositionsForAdd(int positionStart, int itemCount) {
            offsetPositionRecordsForInsert(positionStart, itemCount);
            mItemsAddedOrRemoved = true;
        }

        @Override
        public void offsetPositionsForMove(int from, int to) {
            offsetPositionRecordsForMove(from, to);
            // should we create mItemsMoved ?
            mItemsAddedOrRemoved = true;
        }
    });
}

class ChildHelper{
	final List<View> mHiddenViews;//缓存的屏幕之外的view视图
    
    //01-01 ChildHelper的方法
    boolean isHidden(View view) {
        return mHiddenViews.contains(view);
    }    
}
```




```java
//02
private void initChildrenHelper() {
    //子View帮助类
    mChildHelper = new ChildHelper(new ChildHelper.Callback() {
        @Override
        public int getChildCount() {
            return RecyclerView.this.getChildCount();//获取子View的数量
        }

        @Override
        public void addView(View child, int index) {
            if (VERBOSE_TRACING) {
                TraceCompat.beginSection("RV addView");
            }
            RecyclerView.this.addView(child, index);//添加子view
            if (VERBOSE_TRACING) {
                TraceCompat.endSection();
            }
            dispatchChildAttached(child);
        }

        @Override
        public int indexOfChild(View view) {
            return RecyclerView.this.indexOfChild(view);//得到子view的位置
        }

        @Override
        public void removeViewAt(int index) {//移除指定位置View
            final View child = RecyclerView.this.getChildAt(index);
            if (child != null) {
                dispatchChildDetached(child);

                // Clear any android.view.animation.Animation that may prevent the item from
                // detaching when being removed. If a child is re-added before the
                // lazy detach occurs, it will receive invalid attach/detach sequencing.
                child.clearAnimation();
            }
            if (VERBOSE_TRACING) {
                TraceCompat.beginSection("RV removeViewAt");
            }
            RecyclerView.this.removeViewAt(index);
            if (VERBOSE_TRACING) {
                TraceCompat.endSection();
            }
        }

        @Override
        public View getChildAt(int offset) {
            return RecyclerView.this.getChildAt(offset);//得到指定位置的子view
        }

        @Override
        public void removeAllViews() {//移除所有View
            final int count = getChildCount();
            for (int i = 0; i < count; i++) {
                View child = getChildAt(i);
                dispatchChildDetached(child);

                // Clear any android.view.animation.Animation that may prevent the item from
                // detaching when being removed. If a child is re-added before the
                // lazy detach occurs, it will receive invalid attach/detach sequencing.
                child.clearAnimation();
            }
            RecyclerView.this.removeAllViews();
        }

        @Override
        public ViewHolder getChildViewHolder(View view) {
            return getChildViewHolderInt(view);//根据View的到ViewHolder
        }

        @Override
        public void attachViewToParent(View child, int index,
                ViewGroup.LayoutParams layoutParams) {
            final ViewHolder vh = getChildViewHolderInt(child);
            if (vh != null) {
                if (!vh.isTmpDetached() && !vh.shouldIgnore()) {
                    throw new IllegalArgumentException("Called attach on a child which is not"
                            + " detached: " + vh + exceptionLabel());
                }
                if (DEBUG) {
                    Log.d(TAG, "reAttach " + vh);
                }
                vh.clearTmpDetachFlag();
            }
            RecyclerView.this.attachViewToParent(child, index, layoutParams);//添加到父View
        }

        @Override
        public void detachViewFromParent(int offset) {
            final View view = getChildAt(offset);
            if (view != null) {
                final ViewHolder vh = getChildViewHolderInt(view);
                if (vh != null) {
                    if (vh.isTmpDetached() && !vh.shouldIgnore()) {
                        throw new IllegalArgumentException("called detach on an already"
                                + " detached child " + vh + exceptionLabel());
                    }
                    if (DEBUG) {
                        Log.d(TAG, "tmpDetach " + vh);
                    }
                    vh.addFlags(ViewHolder.FLAG_TMP_DETACHED);
                }
            }
            RecyclerView.this.detachViewFromParent(offset);//分离从父View
        }

        @Override
        public void onEnteredHiddenState(View child) {
            final ViewHolder vh = getChildViewHolderInt(child);
            if (vh != null) {
                vh.onEnteredHiddenState(RecyclerView.this);//隐藏子View
            }
        }

        @Override
        public void onLeftHiddenState(View child) {
            final ViewHolder vh = getChildViewHolderInt(child);
            if (vh != null) {
                vh.onLeftHiddenState(RecyclerView.this);
            }
        }
    });
}
```

```java
//03
private void initAutofill() {
    if (ViewCompat.getImportantForAutofill(this) == View.IMPORTANT_FOR_AUTOFILL_AUTO) {
        ViewCompat.setImportantForAutofill(this,
                View.IMPORTANT_FOR_AUTOFILL_NO_EXCLUDE_DESCENDANTS);
    }
}
```

```java
//04
protected void onMeasure(int widthSpec, int heightSpec) {
    if (mLayout == null) {
        defaultOnMeasure(widthSpec, heightSpec);//测量宽高， 调用 04-01
        return;
    }
    if (mLayout.isAutoMeasureEnabled()) {
        final int widthMode = MeasureSpec.getMode(widthSpec);
        final int heightMode = MeasureSpec.getMode(heightSpec);

        /**
         * This specific call should be considered deprecated and replaced with
         * {@link #defaultOnMeasure(int, int)}. It can't actually be replaced as it could
         * break existing third party code but all documentation directs developers to not
         * override {@link LayoutManager#onMeasure(int, int)} when
         * {@link LayoutManager#isAutoMeasureEnabled()} returns true.
         */
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);

        final boolean measureSpecModeIsExactly =
                widthMode == MeasureSpec.EXACTLY && heightMode == MeasureSpec.EXACTLY;
        if (measureSpecModeIsExactly || mAdapter == null) {
            return;
        }

        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();
        }
        // set dimensions in 2nd step. Pre-layout should happen with old dimensions for
        // consistency
        mLayout.setMeasureSpecs(widthSpec, heightSpec);
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();

        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);

        // if RecyclerView has non-exact width and height and if there is at least one child
        // which also has non-exact width & height, we have to re-measure.
        if (mLayout.shouldMeasureTwice()) {
            mLayout.setMeasureSpecs(
                    MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                    MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
            mState.mIsMeasuring = true;
            dispatchLayoutStep2();
            // now we can get the width and height from the children.
            mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
        }
    } else {
        if (mHasFixedSize) {
            mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
            return;
        }
        // custom onMeasure
        if (mAdapterUpdateDuringMeasure) {
            startInterceptRequestLayout();
            onEnterLayoutOrScroll();
            processAdapterUpdatesAndSetAnimationFlags();
            onExitLayoutOrScroll();

            if (mState.mRunPredictiveAnimations) {
                mState.mInPreLayout = true;
            } else {
                // consume remaining updates to provide a consistent state with the layout pass.
                mAdapterHelper.consumeUpdatesInOnePass();
                mState.mInPreLayout = false;
            }
            mAdapterUpdateDuringMeasure = false;
            stopInterceptRequestLayout(false);
        } else if (mState.mRunPredictiveAnimations) {
            // If mAdapterUpdateDuringMeasure is false and mRunPredictiveAnimations is true:
            // this means there is already an onMeasure() call performed to handle the pending
            // adapter change, two onMeasure() calls can happen if RV is a child of LinearLayout
            // with layout_width=MATCH_PARENT. RV cannot call LM.onMeasure() second time
            // because getViewForPosition() will crash when LM uses a child to measure.
            setMeasuredDimension(getMeasuredWidth(), getMeasuredHeight());
            return;
        }

        if (mAdapter != null) {
            mState.mItemCount = mAdapter.getItemCount();//调用RecyclerView.Adapter的getItemCount
        } else {
            mState.mItemCount = 0;
        }
        startInterceptRequestLayout();
        mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
        stopInterceptRequestLayout(false);
        mState.mInPreLayout = false; // clear
    }
}

//04-01
void defaultOnMeasure(int widthSpec, int heightSpec) {
        // calling LayoutManager here is not pretty but that API is already public and it is better
        // than creating another method since this is internal.
        final int width = LayoutManager.chooseSize(widthSpec,
                getPaddingLeft() + getPaddingRight(),
                ViewCompat.getMinimumWidth(this));
        final int height = LayoutManager.chooseSize(heightSpec,
                getPaddingTop() + getPaddingBottom(),
                ViewCompat.getMinimumHeight(this));

        setMeasuredDimension(width, height);
    }
```



```java
//05
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();//分发布局， 调用 05-01
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}

void dispatchLayout() {
        if (mAdapter == null) {
            Log.e(TAG, "No adapter attached; skipping layout");
            // leave the state in START
            return;
        }
        if (mLayout == null) {
            Log.e(TAG, "No layout manager attached; skipping layout");
            // leave the state in START
            return;
        }
        mState.mIsMeasuring = false;
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();//内部通过调用LayoutManager然后又调用到 onCreateViewHolder与onBindViewHolder
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth()
                || mLayout.getHeight() != getHeight()) {
            // First 2 steps are done in onMeasure but looks like we have to run again due to
            // changed size.
            mLayout.setExactMeasureSpecsFrom(this);
            dispatchLayoutStep2();
        } else {
            // always make sure we sync them (to ensure mode is exact)
            mLayout.setExactMeasureSpecsFrom(this);
        }
        dispatchLayoutStep3();
    }

```

```java
//06
public void onDraw(Canvas c) {
    super.onDraw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDraw(c, this, mState);//绘制动画
    }
}
```



```java
//07
public void draw(Canvas c) {
    super.draw(c);

    final int count = mItemDecorations.size();
    for (int i = 0; i < count; i++) {
        mItemDecorations.get(i).onDrawOver(c, this, mState);//动画绘制结束。
    }
    // TODO If padding is not 0 and clipChildrenToPadding is false, to draw glows properly, we
    // need find children closest to edges. Not sure if it is worth the effort.
    boolean needsInvalidate = false;
    if (mLeftGlow != null && !mLeftGlow.isFinished()) {
        final int restore = c.save();
        final int padding = mClipToPadding ? getPaddingBottom() : 0;
        c.rotate(270);
        c.translate(-getHeight() + padding, 0);
        needsInvalidate = mLeftGlow != null && mLeftGlow.draw(c);
        c.restoreToCount(restore);
    }
    if (mTopGlow != null && !mTopGlow.isFinished()) {
        final int restore = c.save();
        if (mClipToPadding) {
            c.translate(getPaddingLeft(), getPaddingTop());
        }
        needsInvalidate |= mTopGlow != null && mTopGlow.draw(c);
        c.restoreToCount(restore);
    }
    if (mRightGlow != null && !mRightGlow.isFinished()) {
        final int restore = c.save();
        final int width = getWidth();
        final int padding = mClipToPadding ? getPaddingTop() : 0;
        c.rotate(90);
        c.translate(-padding, -width);
        needsInvalidate |= mRightGlow != null && mRightGlow.draw(c);
        c.restoreToCount(restore);
    }
    if (mBottomGlow != null && !mBottomGlow.isFinished()) {
        final int restore = c.save();
        c.rotate(180);
        if (mClipToPadding) {
            c.translate(-getWidth() + getPaddingRight(), -getHeight() + getPaddingBottom());
        } else {
            c.translate(-getWidth(), -getHeight());
        }
        needsInvalidate |= mBottomGlow != null && mBottomGlow.draw(c);
        c.restoreToCount(restore);
    }

    // If some views are animating, ItemDecorators are likely to move/change with them.
    // Invalidate RecyclerView to re-draw decorators. This is still efficient because children's
    // display lists are not invalidated.
    if (!needsInvalidate && mItemAnimator != null && mItemDecorations.size() > 0
            && mItemAnimator.isRunning()) {
        needsInvalidate = true;
    }

    if (needsInvalidate) {
        ViewCompat.postInvalidateOnAnimation(this);
    }
}
```

```java
public final class Recycler {
    //4级缓存
    final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();//存放全部的对象
    ArrayList<ViewHolder> mChangedScrap = null;//发生改变的对象
    final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();//存放remove调的视图的VIewHolder对象(保存的信息、绑定的数据未删除)，容量有限制默认是2。
    RecycledViewPool mRecyclerPool;//存放remove调的视图的VIewHolder对象（完全删除信息、数据等）。
    
    private final List<ViewHolder>
            mUnmodifiableAttachedScrap = Collections.unmodifiableList(mAttachedScrap);//当前在报废列表中的不可修改的列表

    
    private int mRequestedCacheMax = DEFAULT_CACHE_SIZE;//最大请求缓存个数，默认2.
    int mViewCacheMax = DEFAULT_CACHE_SIZE;//最大View缓存个数，默认2.


    private ViewCacheExtension mViewCacheExtension; //缓存扩展类，支持用户自定义

    static final int DEFAULT_CACHE_SIZE = 2;
}
```







