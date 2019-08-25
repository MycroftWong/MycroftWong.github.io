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
    // 默认的gravity
    private static final int DEFAULT_CHILD_GRAVITY = Gravity.TOP | Gravity.START;

    // 是否强制测量所有的children，可以在xml中指定measureAllChildren
    boolean mMeasureAllChildren = false;

    // 下面四个属性，指的是foreground的padding，若foreground有padding，那么将影响最后的尺寸
    private int mForegroundPaddingLeft = 0;
    private int mForegroundPaddingTop = 0;
    private int mForegroundPaddingRight = 0;
    private int mForegroundPaddingBottom = 0;

    // 所有layout_width或layout_height指定了match_parent属性的children，用于2次计算尺寸
    private final ArrayList<View> mMatchParentChildren = new ArrayList<>(1);

    // 下面四个构造器
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
        // 获取并设置measureAllChildren属性
        if (a.getBoolean(R.styleable.FrameLayout_measureAllChildren, false)) {
            setMeasureAllChildren(true);
        }
        a.recycle();
    }

    /**
     * 描述前景色的gravity，默认是START|TOP，可以通过foregroundGravity属性设置
     */
    @android.view.RemotableViewMethod
    public void setForegroundGravity(int foregroundGravity) {
        if (getForegroundGravity() != foregroundGravity) {
            super.setForegroundGravity(foregroundGravity);

            // 获取foreground的padding属性，用于布局
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

            // 重新布局
            requestLayout();
        }
    }

    /**
     * 返回一个默认MATCH_PARENT的FrameLayout.LayoutParams
     * child没有LayoutParams时使用这个
     */
    @Override
    protected LayoutParams generateDefaultLayoutParams() {
        return new LayoutParams(LayoutParams.MATCH_PARENT, LayoutParams.MATCH_PARENT);
    }

    /**
     * 下面四个是获取FrameLayout的padding，将foreground计算在内
     * Android内部使用，只用于FrameLayout和内部的屏幕layout
     */
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

    /**
     * 真正的measure过程
     */
    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        int count = getChildCount();

        // 这句话的意思是：FrameLayout的尺寸需要根据children的尺寸确定（长或宽不是确定的）
        // 那么就需要测量两次：
        // 1. 找出最大的child，FrameLayout尺寸根据最大的child得出
        // 2. 重新测量所有match_parent的children
        final boolean measureMatchParentChildren =
                MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
                MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;
        // 用于存储需要重新测量的children(match_parent)
        mMatchParentChildren.clear();

        int maxHeight = 0;
        int maxWidth = 0;
        int childState = 0;

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            // View不是GONE，或者强制测量所有的children时才进行测量
            if (mMeasureAllChildren || child.getVisibility() != GONE) {
                // 测量一个child
                measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);
                // 获取LayoutParams
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                // 计算得到maxWidth
                maxWidth = Math.max(maxWidth,
                        child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);
                // 计算得到maxHeight
                maxHeight = Math.max(maxHeight,
                        child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
                // 计算childState
                childState = combineMeasuredStates(childState, child.getMeasuredState());
                // 找出需要测量两次的children
                if (measureMatchParentChildren) {
                    if (lp.width == LayoutParams.MATCH_PARENT ||
                            lp.height == LayoutParams.MATCH_PARENT) {
                        mMatchParentChildren.add(child);
                    }
                }
            }
        }

        // 在宽高上加上FrameLayout本身的padding
        maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();
        maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

        // 宽高不能小于最小值（background和minHeight检测）
        maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());
        maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

        // 宽高不能小于最小值（foreground检测）
        final Drawable drawable = getForeground();
        if (drawable != null) {
            maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());
            maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
        }

        // 确定FrameLayout的尺寸
        setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),
                resolveSizeAndState(maxHeight, heightMeasureSpec,
                        childState << MEASURED_HEIGHT_STATE_SHIFT));

        // 二次测量match_parent的children，下面的一些就不说了，说一下这个count
        count = mMatchParentChildren.size();
        // 这里判断count>1并不是count>0，是因为
        // 1.如果没有MATCH_PARENT，那么count==0
        // 2.如果有MATCH_PARENT，那么count=1的话，那么child就是尺寸最大的，不需要再次测量
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

    /**
     * 我们真正关心的地方
     */
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);
    }

    // 真正的逻辑执行地方
    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();

        // 获取padding用于layout
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();

        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();

        for (int i = 0; i < count; i++) {
            final View child = getChildAt(i);
            // 只layout非GONE的View
            if (child.getVisibility() != GONE) {
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();

                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();

                int childLeft;
                int childTop;

                int gravity = lp.gravity;
                // 若无gravity，则设置默认的gravity
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }

                // 布局方向，一般是从左至右
                final int layoutDirection = getLayoutDirection();
                // absoluteGravity，忽略布局方向，可以认为是horizontalGravity
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;

                // 计算horizontal方向，即childLeft
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    // centerHorizontal，计算出在中间位置是的childLeft
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    // right/end，注意，实际上一定会执行下面的if
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    // 默认是start|top
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }

                // 计算vertical方向，即childTop
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

                // 进行layout
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }

    /**
     * 设置是否measure所有的children，默认false，不measure设置GONE的View
     */
    @android.view.RemotableViewMethod
    public void setMeasureAllChildren(boolean measureAll) {
        mMeasureAllChildren = measureAll;
    }

    /**
     * 返回是否measure所有的children，deprecated
     */
    @Deprecated
    public boolean getConsiderGoneChildrenWhenMeasuring() {
        return getMeasureAllChildren();
    }

    /**
     * 返回是否measure所有的children
     */
    public boolean getMeasureAllChildren() {
        return mMeasureAllChildren;
    }

    /**
     * 根据xml配置，获取LayoutParams
     */
    @Override
    public LayoutParams generateLayoutParams(AttributeSet attrs) {
        return new FrameLayout.LayoutParams(getContext(), attrs);
    }

    /**
     * 不应该拦截press显示状态
     */
    @Override
    public boolean shouldDelayChildPressedState() {
        return false;
    }

    // 检查LayoutParams是否是FrameLayout.LayoutParams
    @Override
    protected boolean checkLayoutParams(ViewGroup.LayoutParams p) {
        return p instanceof LayoutParams;
    }

    // 转换LayoutParams
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

    // 用于accessibility
    @Override
    public CharSequence getAccessibilityClassName() {
        return FrameLayout.class.getName();
    }

    // 内部的方法
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
     * children的LayoutParams，包含了布局属性margin和layout_gravity
     */
    public static class LayoutParams extends MarginLayoutParams {
        /**
         * Value for {@link #gravity} indicating that a gravity has not been
         * explicitly specified.
         */
        public static final int UNSPECIFIED_GRAVITY = -1;

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

        public LayoutParams(@NonNull LayoutParams source) {
            super(source);

            this.gravity = source.gravity;
        }
    }
}
```

`FrameLayout`代码分析完了，其实非常简单，需要注意的是，`FrameLayout`可能会对`match_parent`的`children`进行两次测量。

我们这里着重需要关心的还是`layout`：

1. 考虑到了`padding`，所以我们在定义`View`时，注意`padding`是我们在定义时决定的，所以不要忘记
2. 考虑`margin`，`margin`是`child`相对于在`parent`的，需要在`ViewGroup`控制
3. 考虑`child`的`visibility`属性，如果是`GONE`，那么`ViewGroup`则不应该显示
4. 计算出`child`的左上角的位置，自然就可以得到右下角的位置，调用`child.layout(int, int, int, int)`，如此循环

## 自定义View需要注意的点

`FrameLayout`除了实现`onMeasure(int, int)`和`onLayout(boolean, int, int, int, int)`之外，还重写了一些方法。`FrameLayout`本身是一个非常简单的`ViewGroup`，所以它的实现可以作为定义初级`ViewGroup`的参考。

### LayoutParams generateDefaultLayoutParams()

> Returns a set of default layout parameters. These parameters are requested when the View passed to {@link #addView(View)} has no layout parameters already set. If null is returned, an exception is thrown from addView.

翻译：返回默认的`LayoutParams`。其中的属性要求在`addView(View)`时传递，因为这时没有指定`LayoutParams`。如果为`null`，那么在调用`addView(View)`时会抛出异常。

### LayoutParams generateLayoutParams(AttributeSet attrs)

> Returns a new set of layout parameters based on the supplied attributes set.

翻译：返回读取`AttributeSet`属性的`LayoutParams`。

这样可以在`xml`配置一些布局相关的属性，这些属性封装在自定义的`LayoutParams`中。

### boolean shouldDelayChildPressedState()

> Return true if the pressed state should be delayed for children or descendants of this ViewGroup. Generally, this should be done for containers that can scroll, such as a List. This prevents the pressed state from appearing when the user is actually trying to scroll the content.

翻译：如果`ViewGroup`的`children`或子代（可能`child`是一个`ViewGroup`，其内又有`View`），如果返回`true`，表示自身将优先处理`press`状态。通常，这应该是在可以滑动的容器中返回`true`，例如一个`ListView`。这避免了`press`状态在用户实际上想滑动内容时提前出现。

### boolean checkLayoutParams(ViewGroup.LayoutParams p)

检查`LayoutParams`是否是我们需要的类型。如果我们自定义了一个`LayoutParams`，那么我们应该应该实现这个方法，保证会配合`generateLayoutParams(LayoutParams)`生成我们自定义的`LayoutParams`。

### ViewGroup.LayoutParams generateLayoutParams(ViewGroup.LayoutParams lp)

> Returns a safe set of layout parameters based on the supplied layout params. When a ViewGroup is passed a View whose layout params do not pass the test of {@link #checkLayoutParams(android.view.ViewGroup.LayoutParams)}, this method is invoked. This method should return a new set of layout params suitable for this ViewGroup, possibly by copying the appropriate attributes from the specified set of layout params.

翻译：基于提供的`ViewGroup.LayoutParams`返回一个类型安全的`LayoutParams`。当一个`View`添加到`ViewGroup`中，但是`View`的`LayoutParams`并不能通过`checkLayoutParams(ViewGroup.LayoutParams)`检查，那么这个方法将会被调用。这个方法返回了一个新的`LayoutParams`（通常是自定义的），并且可能会从提供的`ViewGroup.LayoutParams`中复制一些合适的属性。

### CharSequence getAccessibilityClassName()

> Return the class name of this object to be used for accessibility purposes. Subclasses should only override this if they are implementing something that should be seen as a completely new class of view when used by accessibility, unrelated to the class it is deriving from.  This is used to fill in `AccessibilityNodeInfo#setClassName AccessibilityNodeInfo.setClassName`.

大致翻译：返回用于`accessibility`目的当前的类名。如果需要提供这样的功能，那么子类应该实现这个方法。

### LayoutParams

> LayoutParams are used by views to tell their parents how they want to be laid out.

翻译：`LayoutParams`被`view`用于告诉他们的`parent`他们想要的布局信息。

我们知道`LayoutParams`包含了很多布局信息，在`xml`中通常是以`android:layout_xxx`的形式存在，如`android:layout_gravity:start|top`。在`Android`读取`xml`生成对应的`View`对象时，将一些属性赋予`LayoutParams`，那么`View`的`parent`就可以根据`LayoutParams`来对它进行布局。

## 总结

其实`layout`是一个相对而言比较繁琐的工作。因为要考虑到各个方面，`padding`，`margin`，`gravity`等等。更重要的是，通常`Layout`如`LinearLayout`，`ConstraintLayout`会有自己的一套布局逻辑，这个逻辑可能非常的繁琐，如`ConstraintLayout`。逻辑相对简单的`LinearLayout`也有2000多行的代码。

`layout`的相关方法使用很简单，难的是需要实现自己的逻辑。这篇文章分析了简单的`FrameLayout`，我们在定义时，可以参考`FrameLayout`的实现，同时如果想要更好的定义`ViewGroup`，建议多阅读其他优秀实现的源码。

## 参考文章

[小Demo小知识-android:foreground与android:background](https://www.jianshu.com/p/b5ecd39ed494)