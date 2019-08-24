---
title: 三、View布局
date: 2019-08-24 21:51:24
categories:
- Android
    - View
tags:
- Android
- View
- measure
- layout
- draw
- touch event
- 面试
---

# View布局

## 前言

什么是`layout`布局？前面，我们通过`measure`测量得到了`View`的尺寸，那么`View`到底是放在哪个位置上的呢？这就是`layout`的功能，确定`View`在屏幕上的位置（通常是相对于其`parent`的位置）。

## 谁来布局

不同于`measure`，`View`的布局并不是在`View`内部设置，而是在其`parent`内确定。这也是合理的，因为`ViewGroup`的作用就是管理`View`在其内部的布局。

明白了这个概念，下面我们来看看`layout`的一些相关方法。

## layout相关的方法说明

### 1. `layout(int l, int t, int r, int b)`

> Assign a size and position to a view and all of its descendants.

翻译：为`View`及其`children`指定尺寸和位置。

> This is the second phase of the layout mechanism. (The first is measuring). In this phase, each parent calls layout on all of its children to position them. This is typically done using the child measurements that were stored in the measure pass().

翻译：这是布局机制的第二阶段（第一阶段是测量）。在这个阶段中，每个`parent`调用它所有`children`的`layout`方法，设置这些`children`的位置。

> Derived classes should not override this method. Derived classes with children should override onLayout. In that method, they should call layout on each of their children.

翻译：子类不应该重写`layout`方法，而是重写`onLayout`方法。在`onLayout`方法中，它应该调用它所有的`children`的`layout(int l, int t, int r, int b)`方法

#### 谁使用

`View`的`parent`用来设置这个`View`在`parent`中的位置。

#### 用来做什么

`parent`设置`View`在`parent`的位置。

### 2. `onLayout(boolean changed, int left, int top, int right, int bottom)`

> Called from layout when this view should assign a size and position to each of its children.

翻译：在`View`的`layout(int, int, int, int)`中被调用，用于指定每个`children`的尺寸和位置。

#### 谁使用

`View`自身的`layout`调用时，自动会被调用

#### 用来做什么

通常是`parent`用来确定它的`children`在它里面的位置（相对位置）。

#### 特别注意

第一个参数`changed`指的是：这个`View`的尺寸或（和）位置改变了，这通常表明可能进行了多次无意义的`layout`

其他参数都是`child`相对于`parent`的位置（相对位置）。

### 总结

1. `layout(int, int, int, int)`是`parent`调用，`child`在`parent`中的位置
2. `parent`则应该在`onLayout(boolean, int, int, int, int)`中调用其`child.layout(int, int, int, int)`

## FrameLayout源码分析

自定义一个`ViewGroup`我也暂时没有好的想法，这里来分析一下`Android`自带的布局中最简单的`FrameLayout`。

> FrameLayout is designed to block out an area on the screen to display a single item. Generally, FrameLayout should be used to hold a single child view, because it can be difficult to organize child views in a way that's scalable to different screen sizes without the children overlapping each other. You can, however, add multiple children to a FrameLayout and control their position within the FrameLayout by assigning gravity to each child, using the `android:layout_gravity` attribute.

翻译：`FrameLayout`被设计用来为单独的一个`View`划分出一个区域。通常`FrameLayout`应该用于包含单独的一个`View`，因为对于扩展到多个不同的屏幕尺寸来说，`FrameLayout`则很难组织它的`child views`以保证相互不重叠。当然一页可以添加多个`children`到`FrameLayout`中，并且为每个`child`指定`gravity`属性来控制他们的位置。

> Child views are drawn in a stack, with the most recently added child on top. The size of the FrameLayout is the size of its largest child (plus padding), visible or not (if the FrameLayout's parent permits). Views that are `android.view.View#GONE` are used for sizing only if `setMeasureAllChildren(boolean) setConsiderGoneChildrenWhenMeasuring()` is set to true.

翻译：`child views`以栈形式绘制，最后添加的在最上层。`FrameLayout`和它尺寸最大的`child`尺寸相同（加上`padding`属性）。只有当`setMeasureAllChildren(boolean) setConsiderGoneChildrenWhenMeasuring()`被设置为`true`时，设置为`GONE`的`child`才会进行测量。

```java
@RemoteView
public class FrameLayout extends ViewGroup {
    private static final int DEFAULT_CHILD_GRAVITY = Gravity.TOP | Gravity.START;

    @ViewDebug.ExportedProperty(category = "measurement")
    boolean mMeasureAllChildren = false;

    @ViewDebug.ExportedProperty(category = "padding")
    private int mForegroundPaddingLeft = 0;

    @ViewDebug.ExportedProperty(category = "padding")
    private int mForegroundPaddingTop = 0;

    @ViewDebug.ExportedProperty(category = "padding")
    private int mForegroundPaddingRight = 0;

    @ViewDebug.ExportedProperty(category = "padding")
    private int mForegroundPaddingBottom = 0;

    private final ArrayList<View> mMatchParentChildren = new ArrayList<>(1);

    public FrameLayout(@NonNull Context context) {
        super(context);
    }

    public FrameLayout(@NonNull Context context, @Nullable AttributeSet attrs) {
        this(context, attrs, 0);
    }

    public FrameLayout(@NonNull Context context, @Nullable AttributeSet attrs,
            @AttrRes int defStyleAttr) {
        this(context, attrs, defStyleAttr, 0);
    }

    public FrameLayout(@NonNull Context context, @Nullable AttributeSet attrs,
            @AttrRes int defStyleAttr, @StyleRes int defStyleRes) {
        super(context, attrs, defStyleAttr, defStyleRes);

        final TypedArray a = context.obtainStyledAttributes(
                attrs, R.styleable.FrameLayout, defStyleAttr, defStyleRes);

        if (a.getBoolean(R.styleable.FrameLayout_measureAllChildren, false)) {
            setMeasureAllChildren(true);
        }

        a.recycle();
    }

    /**
     * Describes how the foreground is positioned. Defaults to START and TOP.
     *
     * @param foregroundGravity See {@link android.view.Gravity}
     *
     * @see #getForegroundGravity()
     *
     * @attr ref android.R.styleable#View_foregroundGravity
     */
    @android.view.RemotableViewMethod
    public void setForegroundGravity(int foregroundGravity) {
        if (getForegroundGravity() != foregroundGravity) {
            super.setForegroundGravity(foregroundGravity);

            // calling get* again here because the set above may apply default constraints
            final Drawable foreground = getForeground();
            if (getForegroundGravity() == Gravity.FILL && foreground != null) {
                Rect padding = new Rect();
                if (foreground.getPadding(padding)) {
                    mForegroundPaddingLeft = padding.left;
                    mForegroundPaddingTop = padding.top;
                    mForegroundPaddingRight = padding.right;
                    mForegroundPaddingBottom = padding.bottom;
                }
            } else {
                mForegroundPaddingLeft = 0;
                mForegroundPaddingTop = 0;
                mForegroundPaddingRight = 0;
                mForegroundPaddingBottom = 0;
            }

            requestLayout();
        }
    }

    /**
     * Returns a set of layout parameters with a width of
     * {@link android.view.ViewGroup.LayoutParams#MATCH_PARENT},
     * and a height of {@link android.view.ViewGroup.LayoutParams#MATCH_PARENT}.
     */
    @Override
    protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    }

    int getPaddingLeftWithForeground() {
        return isForegroundInsidePadding() ? Math.max(mPaddingLeft, mForegroundPaddingLeft) :
            mPaddingLeft + mForegroundPaddingLeft;
    }

    int getPaddingRightWithForeground() {
        return isForegroundInsidePadding() ? Math.max(mPaddingRight, mForegroundPaddingRight) :
            mPaddingRight + mForegroundPaddingRight;
    }

    private int getPaddingTopWithForeground() {
        return isForegroundInsidePadding() ? Math.max(mPaddingTop, mForegroundPaddingTop) :
            mPaddingTop + mForegroundPaddingTop;
    }

    private int getPaddingBottomWithForeground() {
        return isForegroundInsidePadding() ? Math.max(mPaddingBottom, mForegroundPaddingBottom) :
            mPaddingBottom + mForegroundPaddingBottom;
    }

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        // Account for padding too
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // Check against our minimum height and width
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // Check against our foreground's minimum height and width
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        count = mMatchParentChildren.size();
        if (count > 1) {
            for (int i = 0; i < count; i++) {
                final View child = mMatchParentChildren.get(i);
                final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

                final int childWidthMeasureSpec;
                if (lp.width == LayoutParams.MATCH_PARENT) {
                    final int width = Math.max(0, getMeasuredWidth()
                            - getPaddingLeftWithForeground() - getPaddingRightWithForeground()
                            - lp.leftMargin - lp.rightMargin);
                    childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                            width, MeasureSpec.EXACTLY);
                } else {
                    childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                            getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                            lp.leftMargin + lp.rightMargin,
                            lp.width);
                }

                final int childHeightMeasureSpec;
                if (lp.height == LayoutParams.MATCH_PARENT) {
                    final int height = Math.max(0, getMeasuredHeight()
                            - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                            - lp.topMargin - lp.bottomMargin);
                    childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                            height, MeasureSpec.EXACTLY);
                } else {
                    childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                            getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                            lp.topMargin + lp.bottomMargin,
                            lp.height);
                }

                child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
            }
        }
    }

    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();

        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }

                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }

    /**
     * Sets whether to consider all children, or just those in
     * the VISIBLE or INVISIBLE state, when measuring. Defaults to false.
     *
     * @param measureAll true to consider children marked GONE, false otherwise.
     * Default value is false.
     *
     * @attr ref android.R.styleable#FrameLayout_measureAllChildren
     */
    @android.view.RemotableViewMethod
    public void setMeasureAllChildren(boolean measureAll) {
        mMeasureAllChildren = measureAll;
    }

    /**
     * Determines whether all children, or just those in the VISIBLE or
     * INVISIBLE state, are considered when measuring.
     *
     * @return Whether all children are considered when measuring.
     *
     * @deprecated This method is deprecated in favor of
     * {@link #getMeasureAllChildren() getMeasureAllChildren()}, which was
     * renamed for consistency with
     * {@link #setMeasureAllChildren(boolean) setMeasureAllChildren()}.
     */
    @Deprecated
    public boolean getConsiderGoneChildrenWhenMeasuring() {
        return getMeasureAllChildren();
    }

    /**
     * Determines whether all children, or just those in the VISIBLE or
     * INVISIBLE state, are considered when measuring.
     *
     * @return Whether all children are considered when measuring.
     */
    public boolean getMeasureAllChildren() {
        return mMeasureAllChildren;
    }

    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new FrameLayout.LayoutParams(getContext(), attrs);
    }

    @Override
    public boolean shouldDelayChildPressedState() {
        return false;
    }

    @Override
    protected boolean checkLayoutParams(ViewGroup.LayoutParams p) {
        return p instanceof LayoutParams;
    }

    @Override
    protected ViewGroup.LayoutParams generateLayoutParams(ViewGroup.LayoutParams lp) {
        if (sPreserveMarginParamsInLayoutParamConversion) {
            if (lp instanceof LayoutParams) {
                return new LayoutParams((LayoutParams) lp);
            } else if (lp instanceof MarginLayoutParams) {
                return new LayoutParams((MarginLayoutParams) lp);
            }
        }
        return new LayoutParams(lp);
    }

    @Override
    public CharSequence getAccessibilityClassName() {
        return FrameLayout.class.getName();
    }

    /** @hide */
    @Override
    protected void encodeProperties(@NonNull ViewHierarchyEncoder encoder) {
        super.encodeProperties(encoder);

        encoder.addProperty("measurement:measureAllChildren", mMeasureAllChildren);
        encoder.addProperty("padding:foregroundPaddingLeft", mForegroundPaddingLeft);
        encoder.addProperty("padding:foregroundPaddingTop", mForegroundPaddingTop);
        encoder.addProperty("padding:foregroundPaddingRight", mForegroundPaddingRight);
        encoder.addProperty("padding:foregroundPaddingBottom", mForegroundPaddingBottom);
    }

    /**
     * Per-child layout information for layouts that support margins.
     * See {@link android.R.styleable#FrameLayout_Layout FrameLayout Layout Attributes}
     * for a list of all child view attributes that this class supports.
     *
     * @attr ref android.R.styleable#FrameLayout_Layout_layout_gravity
     */
    public static class LayoutParams extends MarginLayoutParams {
        /**
         * Value for {@link #gravity} indicating that a gravity has not been
         * explicitly specified.
         */
        public static final int UNSPECIFIED_GRAVITY = -1;

        /**
         * The gravity to apply with the View to which these layout parameters
         * are associated.
         * <p>
         * The default value is {@link #UNSPECIFIED_GRAVITY}, which is treated
         * by FrameLayout as {@code Gravity.TOP | Gravity.START}.
         *
         * @see android.view.Gravity
         * @attr ref android.R.styleable#FrameLayout_Layout_layout_gravity
         */
        public int gravity = UNSPECIFIED_GRAVITY;

        public LayoutParams(@NonNull Context c, @Nullable AttributeSet attrs) {
            super(c, attrs);

            final TypedArray a = c.obtainStyledAttributes(attrs, R.styleable.FrameLayout_Layout);
            gravity = a.getInt(R.styleable.FrameLayout_Layout_layout_gravity, UNSPECIFIED_GRAVITY);
            a.recycle();
        }

        public LayoutParams(int width, int height) {
            super(width, height);
        }

        /**
         * Creates a new set of layout parameters with the specified width, height
         * and weight.
         *
         * @param width the width, either {@link #MATCH_PARENT},
         *              {@link #WRAP_CONTENT} or a fixed size in pixels
         * @param height the height, either {@link #MATCH_PARENT},
         *               {@link #WRAP_CONTENT} or a fixed size in pixels
         * @param gravity the gravity
         *
         * @see android.view.Gravity
         */
        public LayoutParams(int width, int height, int gravity) {
            super(width, height);
            this.gravity = gravity;
        }

        public LayoutParams(@NonNull ViewGroup.LayoutParams source) {
            super(source);
        }

        public LayoutParams(@NonNull ViewGroup.MarginLayoutParams source) {
            super(source);
        }

        /**
         * Copy constructor. Clones the width, height, margin values, and
         * gravity of the source.
         *
         * @param source The layout params to copy from.
         */
        public LayoutParams(@NonNull LayoutParams source) {
            super(source);

            this.gravity = source.gravity;
        }
    }
}
```