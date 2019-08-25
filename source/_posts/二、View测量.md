---
title: 二、View测量
date: 2019-08-24 14:34:31
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

# View测量

## 前言

自定义`View`实际上是`Android`给我们定下了一些规则，我们需要遵循这些规则去定义一个`View`，符合这个规则的`View`才会更好的显示。实际上，它并没有如`Java`的强类型般的限制我们怎么做，我们在使用中可能时长在破坏这些规则。不了解规则就导致了我们定义的`View`却不是我们想要的。所以定义`View`之前，一定要清楚明白这些规则是什么。

## 基础知识

先来说明一下一个`View`显示在屏幕上需要哪些属性：

1. `View`的大小（尺寸）
2. `View`在屏幕上的位置（通常是相对其`parent`的位置）
3. `View`显示内容，当然也可以没有内容

`Android`对于一个`View`如何显示在屏幕上确定了一个流程：`measure -> layout -> draw`，即：测量 -> 布局 -> 绘制。

关于绘制的问题，我们后面再讨论，首先来看一下测量与布局。

## MeasureSpec（重点）

`MeasureSpec`是什么？这是一个非常重要，且难以理解的概念。

`MeasureSpec`是`View`的一个静态公开内部类，先看看的说明。

> A MeasureSpec encapsulates the layout requirements passed from parent to child. Each MeasureSpec represents a requirement for either the width or the height. A MeasureSpec is comprised of a size and a mode. There are three possible modes:
> UNSPECIFIED: The parent has not imposed any constraint on the child. It can be whatever size it wants.
> EXACTLY: The parent has determined an exact size for the child. The child is going to be given those bounds regardless of how big it wants to be.
> AT_MOST: The child can be as large as it wants up to the specified size.
> MeasureSpecs are implemented as ints to reduce object allocation. This class is provided to pack and unpack the &lt;size, mode&gt; tuple into the int.

翻译：`MeasureSpec`封装了`parent`对`child`的布局（实际上是尺寸）要求。每个`MeasureSpec`代表宽或高的要求。一个`MeasureSpec`是一个`mode`和一个`size`的组合。有如下三种`mode`：

1. `UNSPECIFED`：`parent`对`child`的尺寸不做限制。`child`想显示多大都可以
2. `EXACTLY`：`parent`确定了`child`的尺寸。不管`child`想要多大，必须按照`parent`给的尺寸
3. `AT_MOST`：`parent`对`child`限制了一个最大尺寸，`child`应该不大于这个尺寸

为了减少内存消耗，`MeasureSpec`使用的是`int`来实现。`MeasureSpec`类提供了封装和解封`<size, mode>`组合为`int`的方法。

---

从这几句话当中，我们可以提取出几点：

1. `MeasureSpec`是`parent`对`child`的布局（实际上是尺寸）要求
2. `MeasureSpec`是由两部分组成：`mode`、`size`
3. `MeasureSpec`实际上是`int`值

### MeasureSpec的组成

`MeasureSpec`是由一个32位的`int`值表示。其中，前2位表示`mode`，后30位表示`size`（屏幕尺寸远远小于30位组成的`int`值`1,073,741,823`，所以不用担心屏幕大小超出值的问题）。

我们知道，两位可以组成4个数，在这里`00`代表`UNSEPECIFIED`，`01`代表`EXACTLY`，`10`代表`AT_MOST`，`11`并没有用到。

而关于这些值的获取，`MeasureSpec`提供了很多方法获取，所以不用自己去计算。如下：

```java
public static int getMode(int measureSpec) {
    return (measureSpec & MODE_MASK);
}
public static int getSize(int measureSpec) {
    return (measureSpec & ~MODE_MASK);
}

public static int makeMeasureSpec(int size,@MeasureSpecMode int mode) {
    if (sUseBrokenMakeMeasureSpec) {
        return size + mode;
    } else {
        return (size & ~MODE_MASK) | (mode & MODE_MASK);
    }
}
public static int makeSafeMeasureSpec(int size, int mode) {
    if (sUseZeroUnspecifiedMeasureSpec && mode == UNSPECIFIED) {
        return 0;
    }
    return makeMeasureSpec(size, mode);
}
```

### MeasureSpec的产生

直接说结果：`child`的`MeasureSpec`是`parent`计算得到，传给`child`的。`child`的`MeasureSpec`是根据`parent`的`MeasureSpec`+`child`的`LayoutParams`得到的。如下图：

![MeasureSpec的产生](MeasureSpec的产生.jpg)

`ViewGroup`在`measureChild(View, int, int)`中调用了`getChildMeasureSpec(int, int, int)`。源码如下：

```java
/**
 * Ask one of the children of this view to measure itself, taking into
 * account both the MeasureSpec requirements for this view and its padding.
 * The heavy lifting is done in getChildMeasureSpec.
 *
 * 让其中一个child开始计算自己的尺寸，同时考虑到自身的padding属性和MeasureSpec要求。
 * 重要的工作在getChildMeasureSpec方法中。
 */
protected void measureChild(View child, int parentWidthMeasureSpec,
        int parentHeightMeasureSpec) {
    final LayoutParams lp = child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight, lp.width);
    final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
            mPaddingTop + mPaddingBottom, lp.height);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}

/**
 * Does the hard part of measureChildren: figuring out the MeasureSpec to
 * pass to a particular child. This method figures out the right MeasureSpec
 * for one dimension (height or width) of one child view.
 *
 * The goal is to combine information from our MeasureSpec with the
 * LayoutParams of the child to get the best possible results. For example,
 * if the this view knows its size (because its MeasureSpec has a mode of
 * EXACTLY), and the child has indicated in its LayoutParams that it wants
 * to be the same size as the parent, the parent should ask the child to
 * layout given an exact size.
 *
 * 处理measureChildren的难点部分：计算出传递个child的MeasureSpec。这个方法计算出正确child的MeasureSpec。
 * 这个方法的目标是结合ViewGroup的MeasureSpec信息和child的LayoutParams，获取有可能最好的结果。
 * 例如，如果我们知道View的尺寸（因为MeasureSpec属性是EXACTLY），
 * child并且在LayoutParams指定为MATCH_PARENT，和parent一样的尺寸，那么parent应该给child一个确定的大小。
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    int specMode = MeasureSpec.getMode(spec);
    int specSize = MeasureSpec.getSize(spec);

    int size = Math.max(0, specSize - padding);

    int resultSize = 0;
    int resultMode = 0;

    switch (specMode) {
    // Parent has imposed an exact size on us
    case MeasureSpec.EXACTLY:
        if (childDimension >= 0) {
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size. So be it.
            resultSize = size;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent has imposed a maximum size on us
    case MeasureSpec.AT_MOST:
        if (childDimension >= 0) {
            // Child wants a specific size... so be it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size, but our size is not fixed.
            // Constrain child to not be bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size. It can't be
            // bigger than us.
            resultSize = size;
            resultMode = MeasureSpec.AT_MOST;
        }
        break;

    // Parent asked to see how big we want to be
    case MeasureSpec.UNSPECIFIED:
        if (childDimension >= 0) {
            // Child wants a specific size... let him have it
            resultSize = childDimension;
            resultMode = MeasureSpec.EXACTLY;
        } else if (childDimension == LayoutParams.MATCH_PARENT) {
            // Child wants to be our size... find out how big it should
            // be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        } else if (childDimension == LayoutParams.WRAP_CONTENT) {
            // Child wants to determine its own size.... find out how
            // big it should be
            resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
            resultMode = MeasureSpec.UNSPECIFIED;
        }
        break;
    }
    //noinspection ResourceType
    return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```

代码非常的长，但是还是比较容易理解。起决定因素的是`ViewGroup`自己的`MeasureSpec`，和`View`的`LayoutParams`属性。简化之后如下图所示：

![MeasureSpec值的生成](MeasureSpec值的生成.webp)

### MeasureSpec如何影响尺寸

`MeasureSpec`决定了`View`的最终尺寸（宽高）。为什么呢，怎么做到的呢，对于这点，我们需要对`measure`的一些方法进行解释。

## View的measure相关的方法说明

这里着重`View`而不是`ViewGroup`。

### 1. `measure(int widthMeasureSpec, int heightMeasureSpec)`

> This is called to find out how big a view should be. The parent supplies constraint information in the width and height parameters.

翻译：这个方法被调用来确定一个`View`多大。`View`的`parent`提供在宽高上的约束信息。

### 谁使用

`View`的`parent`。`View`的`parent`通过自身的`MeasureSpec`和`View`的`LayoutParam`得到`View`的`MeasureSpec`.

#### 做什么的

使用`MeasureSpec`计算`View`的尺寸

### 2. `onMeasure(int widthMeasureSpec, int heightMeasureSpec)`

> Measure the view and its content to determine the measured width and the measured height. This method is invoked by {@link #measure(int, int)} and should be overridden by subclasses to provide accurate and efficient measurement of their contents.

翻译：测量`View`和它的内容，决定测量的宽高。这个方法被`measure(int, int)`调用，并且应该被子类重写，保证对`View`的内容有效、准确的测量。

### 谁使用

`View`自身的`measure(int, int)`调用的。重写其中的内容

#### 做什么的

`View`在这个方法里面根据自身的逻辑确定自身的尺寸（宽高）

### 3. `setMeasuredDimension(int measuredWidth, int measuredHeight)`

> This method must be called by {@link #onMeasure(int, int)} to store the measured width and measured height. Failing to do so will trigger an exception at measurement time.

翻译：这个方法一定要在`onMeasure(int, int)`调用，来存储测量好的宽高。如果没有调用，将在测量时间触发异常。

### 谁使用

`View`在`onMeasure(int, int)`方法中调用，设置宽高值。

#### 做什么

`View`设置确切的宽高值。

### 4. `getDefaultSize(int size, int measureSpec)`

> Utility to return a default size. Uses the supplied size if the MeasureSpec imposed no constraints. Will get larger if allowed by the MeasureSpec.

翻译：帮助类，用于返回一个默认的尺寸值。如果`MeasureSpec`是`UNSPECIFIED`类型（即父类不约束），那么尺寸就是默认的尺寸（第一个参数`size`）。如果`MeasureSpec`是`EXACTLY`和`AT_MOST`（父类限制了尺寸），那么就将是限制的尺寸。

### 谁使用

`View`自身，通常是在根据`parent`给的`MeasureSpec`计算实际宽高值时调用

#### 做什么

根据`parent`给的`MeasureSpec`，获得默认的一个尺寸，可以用来确认最终的尺寸值。

### 5. `getSuggestedMinimumWidth()`和`getSuggestedMinimumHeight()`

> Returns the suggested minimum height that the view should use. This returns the maximum of the view's minimum height and the background's minimum height

翻译：返回`View`最小的宽/高。这个方法返回`View`自身设置的`minHeight`和背景的`minHeight`的较大值。

### 总结

这些都是一些`View`在根据其`parent`传过来的`MeasureSpec`用于计算真正尺寸时会用的一些方法。不仅仅是`MeasureSpec`，多数情况下是根据我们自定义`View`的逻辑，来决定`View`应该多大的。

如我定义了一个时钟，如果时钟太小，那么它显示出来并没有任何意义，所以我希望它最小是`200dp*200dp`，所以这个时候，即是`parent`限制时钟的尺寸小于这个数，即是可能显示不全，那么我一定也要让它不小于我期望的尺寸。

可以这样理解，`MeasureSpec`是`parent`对`child`期望的尺寸要求，但是`View`也需要根据自身情况，决定如何去满足这个尺寸要求。

很多时候，使用者使用了我们不期望的属性，导致我们自定义的`View`显示不合理。这不是一个谁一定要遵守谁的规定，而是两者相互配合的结果。

## View的尺寸

前面，已经看到了真正设置`View`尺寸的是`setMeasuredDimension(int, int)`方法。而尺寸的来源，则是根据`parent`给的`MeasureSpec`加上一些限制条件的结果。这些限制条件有如背景的尺寸、设置的`minHeight`，业务逻辑等。

### 简单的自定义的View

我自定义了一个简单的时钟，在`onMeasure`中根据`parent`给的`MeasureSpec`和自身的逻辑确定了尺寸，而大部分工作是在`onDraw`。所以需要明白的一点是，`View`的工作实际上是显示内容的，很多时候决定尺寸的恰恰是内容，如`TextView`通常设置了`wrap_content`，尺寸也随着内容的变化而变化。

```kotlin
class Clock : View {

    constructor(context: Context?) : super(context)
    constructor(context: Context?, attrs: AttributeSet?) : super(context, attrs)
    constructor(context: Context?, attrs: AttributeSet?, defStyleAttr: Int) : super(
        context,
        attrs,
        defStyleAttr
    )

    constructor(
        context: Context?,
        attrs: AttributeSet?,
        defStyleAttr: Int,
        defStyleRes: Int
    ) : super(context, attrs, defStyleAttr, defStyleRes)

    companion object {
        // 最小边长
        private const val MIN_SLIDE = 200f
        // 时钟圆半径
        private const val RADIUS = 80f
        // 时钟圆线宽
        private const val CIRCLE_WIDTH = 1.5f
        // 时钟圆心圆圈半径
        private const val CENTER_RADIUS = 3f
        // 小时刻度长度
        private const val QUARTER_LENGTH = 10f
        // 分钟刻度长度
        private const val TIME_LENGTH = 5f
        // 小时刻度颜色
        private const val COLOR_HOUR = Color.RED
        // 分钟刻度颜色
        private const val COLOR_MINUTE = Color.BLACK
        // 小时文字大小
        private const val TEXT_SIZE_HOUR = 14f
        // 小时文字与小时刻度的间距
        private const val TEXT_DIVIDER = 5f
        // 时针宽度
        private const val WIDTH_HOUR = 6f
        // 分针宽度
        private const val WIDTH_MINUTE = 4f
        // 秒针宽度
        private const val WIDTH_SECOND = 2f
    }

    private val paint = Paint(Paint.ANTI_ALIAS_FLAG)

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec)

        val minSlide = dp2px(MIN_SLIDE)

        // 计算尺寸，保证边长不能小于MIN_SLIDE
        var width = getSize(suggestedMinimumWidth, widthMeasureSpec)
        var height = getSize(suggestedMinimumHeight, heightMeasureSpec)

        if (width < minSlide) {
            width = minSlide.toInt()
        }
        if (height < minSlide) {
            height = minSlide.toInt()
        }

        // 设置最终的尺寸
        setMeasuredDimension(width, height)
    }

    /**
     * 根据MeasureSpec得到理想的尺寸
     *
     * @param size 尺寸默认值
     * @param measureSpec parent给的MeasureSpec
     * @return
     */
    private fun getSize(size: Int, measureSpec: Int): Int {
        var result = size
        val specMode = MeasureSpec.getMode(measureSpec)
        val specSize = MeasureSpec.getSize(measureSpec)

        when (specMode) {
            MeasureSpec.AT_MOST, MeasureSpec.UNSPECIFIED -> result = size
            MeasureSpec.EXACTLY -> result = specSize
        }
        return result
    }

    /**
     * 存放文字的尺寸
     */
    private val textBounds = Rect()

    override fun onDraw(canvas: Canvas?) {
        super.onDraw(canvas)

        // 获取尺寸
        val width = measuredWidth
        val height = measuredHeight

        val radius = dp2px(RADIUS)

        val circleWidth = dp2px(CIRCLE_WIDTH)
        val centerRadius = dp2px(CENTER_RADIUS)

        val c = canvas!!
        c.save()

        // 将坐标轴移动到View中心
        c.translate(width.shr(1).toFloat(), height.shr(1).toFloat())

        // 画刻度
        drawScale(c, radius, circleWidth)
        // 画时钟外圆
        drawCircle(c, radius, circleWidth)

        // 画时针、分针、秒针
        drawTimeHand(c, radius)

        // 画中心圆
        drawCenterCircle(c, centerRadius)
        c.restore()

        // 保证实时更新
        invalidate()
    }

    /**
     * 画刻度
     *
     * @param c canvas
     * @param radius 半径
     * @param circleWidth 外圆的宽度
     */
    private fun drawScale(c: Canvas, radius: Float, circleWidth: Float) {

        val quarterLength = dp2px(QUARTER_LENGTH)
        val timeLength = dp2px(TIME_LENGTH)

        val hourTextSize = dp2px(TEXT_SIZE_HOUR)

        val textDivider = dp2px(TEXT_DIVIDER)

        for (i in 0..59) {
            if (i % 5 == 0) {
                paint.color = COLOR_HOUR
                c.drawLine(radius - quarterLength, 0f, radius - circleWidth, 0f, paint)

                val hour = ((i / 5 + 2) % 12 + 1).toString()

                paint.textSize = hourTextSize
                paint.getTextBounds(hour, 0, hour.length, textBounds)

                c.save()

                c.translate(radius - quarterLength - textBounds.width() / 2f - textDivider, 0f)

                c.rotate(-i * 6f)
                c.drawText(
                    hour,
                    0,
                    hour.length,
                    -textBounds.exactCenterX(),
                    -textBounds.exactCenterY(),
                    paint
                )

                c.restore()
            } else {
                paint.color = COLOR_MINUTE
                c.drawLine(radius - timeLength, 0f, radius - circleWidth, 0f, paint)
            }
            c.rotate(6f)
        }
    }

    /**
     * 画时钟外圆
     *
     * @param c canvas
     * @param radius 半径
     * @param circleWidth 外圆的宽度
     */
    private fun drawCircle(c: Canvas, radius: Float, circleWidth: Float) {

        c.save()
        paint.style = Paint.Style.STROKE
        paint.strokeWidth = circleWidth

        c.drawCircle(0f, 0f, radius - circleWidth, paint)
        c.restore()

    }

    /**
     * 画时针、分针、秒针
     *
     * @param c canvas
     * @param radius 外圆半径
     */
    private fun drawTimeHand(c: Canvas, radius: Float) {

        val now = Calendar.getInstance()
        now.time = Date()

        val hour = now.get(Calendar.HOUR)
        val minute = now.get(Calendar.MINUTE)
        val second = now.get(Calendar.SECOND)
        val millis = now.get(Calendar.MILLISECOND)

        val hourWidth = dp2px(WIDTH_HOUR)
        val minuteWidth = dp2px(WIDTH_MINUTE)
        val secondWidth = dp2px(WIDTH_SECOND)

        val hourLength = radius / 2

        paint.style = Paint.Style.FILL

        // draw hour hand
        c.save()
        c.rotate((hour - 3) * 30f)
        c.drawRoundRect(
            -hourLength / 4f,
            -hourWidth / 2f,
            hourLength,
            hourWidth / 2f,
            hourWidth / 2f,
            hourWidth / 2f,
            paint
        )

        c.restore()

        // draw minute hand
        val minuteLength = radius / 3 * 2
        c.save()
        c.rotate((minute - 15) * 6f)
        c.drawRoundRect(
            -minuteLength / 4f,
            -minuteWidth / 2f,
            minuteLength,
            minuteWidth / 2f,
            minuteWidth / 2,
            minuteWidth / 2f,
            paint
        )
        c.restore()

        // draw second hand
        val secondLength = radius / 3 * 2
        c.save()
        paint.color = COLOR_HOUR
        c.rotate((second - 15) * 6f + millis * 0.006f)
        c.drawRoundRect(
            -secondLength / 4f,
            -secondWidth / 2f,
            secondLength,
            secondWidth / 2f,
            secondWidth / 2,
            secondWidth / 2f,
            paint
        )
        c.restore()
    }

    /**
     * 画中心圆
     *
     * @param c canvas
     * @param centerRadius 中心圆半径
     */
    private fun drawCenterCircle(c: Canvas, centerRadius: Float) {

        c.save()

        paint.color = COLOR_HOUR
        paint.style = Paint.Style.FILL

        c.drawCircle(0f, 0f, centerRadius, paint)
        c.restore()
    }

    /**
     * dp to px
     *
     * @param dp dp
     * @return px
     */
    private fun dp2px(dp: Float): Float {
        return TypedValue.applyDimension(TypedValue.COMPLEX_UNIT_DIP, dp, resources.displayMetrics)
    }
}
```

在`onMeasure`中的逻辑：

1. 根据`parent`给的`MeasureSpec`，并结合`background`、`minWidth/minHeight`计算出理想的尺寸
2. 但是为了符合业务逻辑，如果得到的尺寸太小，则强制使用我期望的最小值。
3. 确定计算得到的宽高

## 总结

`View`的尺寸计算非常简单，因为它只是对它本身的计算，并没有`child`，大部分工作在`onDraw`里。所以，我们在定义`View`时，知道在`onMeasure`中如何计算得到想要的尺寸，再更多的学习`Canvas`相关的`API`就可以了。