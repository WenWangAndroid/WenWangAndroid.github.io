## 前言
在使用RecyclerView过程中，我们经常会使用到到DividerItemDecoration作为Item间的分割线，ItemDecoration除了可以被用作分割线，还可对其自定义实现很多其它功能，例如 时间轴、Item悬浮吸顶效果等。

![](https://lexiangla.com/assets/f1ed021c690711ea930c0a58ac13a36a)


## 1. 自定义流程
自定义ItemDecoration一般需要进行如下步骤

 - 新建类继承于 ItemDecoration
 -  重写 getItemOffsets 可设置Item的偏移值，例如分割线的间隔
 -  重写 onDraw 可在Item层绘制内容，例如分割线
 -  重写 onDrawOver 可在Item上层绘制内容，例如悬浮条目

## 2. 自定义时间轴

如下图所示，自定义步骤如下

 1. Item往右移动一段距离
 2. 确定圆心坐标
 3. 确定线条区域
 4. 绘制圆与线条

![](https://lexiangla.com/assets/11408b20690811ea8eba0a58ac13b80b)

### 完整代码
```kotlin
class TimeLineItemDecoration(val context: Context) : RecyclerView.ItemDecoration() {
    private val linePaint: TextPaint = TextPaint(Paint.ANTI_ALIAS_FLAG)
    private val circlePaint: TextPaint = TextPaint(Paint.ANTI_ALIAS_FLAG)

    @Px
    var timeLineWidth = 80
    @Px
    var lineRadius = 2
    @Px
    var circleRadius = 20
    @ColorRes
    var lineColor: Int = R.color.colorPrimary
        set(value) {
            field = value
            linePaint.color = ContextCompat.getColor(context, value)
        }
    @ColorRes
    var circleColor: Int = R.color.colorPrimary
        set(value) {
            field = value
            circlePaint.color = ContextCompat.getColor(context, value)
        }

    init {
        linePaint.color = ContextCompat.getColor(context, lineColor)
        circlePaint.color = ContextCompat.getColor(context, circleColor)
    }

    override fun onDraw(canvas: Canvas, recyclerView: RecyclerView, state: RecyclerView.State) {
        canvas.save()
        for (i in 0 until recyclerView.childCount) {
            val childView = recyclerView.getChildAt(i)
            val childRect = Rect().also {
                recyclerView.getDecoratedBoundsWithMargins(childView, it)
            }
            //圆心
            val centerX = timeLineWidth / 2F
            val centerY = childRect.top + circleRadius.toFloat()
            //线条坐标
            val lineTop = centerY
            val lineBottom = childRect.bottom.toFloat()
            val lineLeft = centerX - lineRadius
            val lineRight = centerX + lineRadius
            //绘制圆 与 线条
            canvas.drawRect(lineLeft, lineTop, lineRight, lineBottom, linePaint)
            canvas.drawCircle(centerX, centerY, circleRadius.toFloat(), circlePaint)
        }
        canvas.restore()
    }

    override fun getItemOffsets(
        outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State
    ) {
        // super.getItemOffsets(outRect, view, parent, state)
        // 查看源码可知： super.getItemOffsets 就是在调用 outRect.set(0, 0, 0, 0)
        // 设置Item的绘制区域
        outRect.set(timeLineWidth, 0, 0, 0)
    }
}
```
### 其它
```kotlin
recyclerView.childCount //recyclerView当前已绘制的Item数量
RecyclerView.State.itemCount //RecyclerView 应该展示的Item数量
recyclerView.layoutManager.findViewByPosition(childView) //当前Item在RecyclerView中的位置
```
通过以上3个方法，可根据需求在不同的位置绘制不同的 事件状态
若需要同时满足时间轴与分割线，写法与 Item悬浮吸顶效果 的实现类似。

## 3. Item悬浮吸顶
与自定义时间轴相比，要使用ItemDecoration实现Item悬浮吸顶效果则需要外部传入相关数据，并根据数据进行逻辑判断。
```kotlin
//联系人 包含拼音与名字
data class Contact(val initial: String, val name: String)
```
### 绘制逻辑
根据Items之间拼音名称的不同，判断是否绘制悬浮条目
```kotlin
//普通分割线
var dividerDrawable: Drawable    
//悬浮条目背景
var suspendedDrawable: Drawable
//上一条Item的拼音
val lastInitial: String
//当前Item的拼音
val currentInitial: String
//下一条Item的拼音
val nextInitial: String

//Drawable绘制判断
if (currentInitial == lastInitial) dividerDrawable else suspendedDrawable
//悬浮条 滑出判断
//顶部移出效果
if (currentInitial != nextInitial && childView.top + childView.height < drawable.intrinsicHeight) {
    //滑出
}
```

### 完整代码
```kotlin
class SuspendItemDecoration(val context: Context) : RecyclerView.ItemDecoration() {

    var dividerDrawable: Drawable? = null   //普通Item间的分割线
    var suspendedDrawable: Drawable? = null     //悬浮条目背景
    var contacts: List<Contact> = ArrayList()    //联系人数据

    private val textPaint: TextPaint = TextPaint(Paint.ANTI_ALIAS_FLAG)

    @ColorRes
    var textColor: Int = android.R.color.white
        set(value) {
            field = value
            textPaint.color = context.resources.getColor(value)
        }
    @Px
    var textSize: Float = 60F
        set(value) {
            field = value
            textPaint.textSize = context.sp2px(value)
        }

    var textTypeface = Typeface.DEFAULT_BOLD!!
        set(value) {
            field = value
            textPaint.typeface = value
        }

    init {
        //系统默认的分割线
        val tapeArray = context.obtainStyledAttributes(intArrayOf(android.R.attr.listDivider))
        dividerDrawable = tapeArray.getDrawable(0)
        tapeArray.recycle()

        textPaint.color = ContextCompat.getColor(context, textColor)
        textPaint.textSize = textSize
        textPaint.typeface = textTypeface
    }

    override fun onDraw(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        if (!dataCheck(state)) return
        val left = parent.paddingLeft
        val right = parent.width - parent.paddingRight
        c.save()

        for (i in 0 until parent.childCount) {
            val child = parent.getChildAt(i)
            val position = (child.layoutParams as RecyclerView.LayoutParams).viewLayoutPosition
            val rect = Rect().also { parent.getDecoratedBoundsWithMargins(child, it) }

            val currentInitial = contacts[position].initial
            val lastInitial = if (position >= 1) {
                contacts[position - 1].initial
            } else {
                null
            }

            val drawSuspension = currentInitial != lastInitial
            val drawable =
                (if (drawSuspension) suspendedDrawable else dividerDrawable) ?: return

            //绘制Drawable
            val top = rect.top
            val bottom = top + drawable.intrinsicHeight
            drawable.setBounds(left, top, right, bottom)
            drawable.draw(c)

            //绘制文本内容
            if (drawSuspension) {
                textPaint.getTextBounds(currentInitial, 0, currentInitial.length, rect)
                // 绘制文本原点 x坐标
                val textX = child.paddingLeft.toFloat()
                // 绘制文本基线的y坐标 (居中)
                val textY = (child.top - (drawable.intrinsicHeight - rect.height()) / 2).toFloat()
                c.drawText(currentInitial, textX, textY, textPaint)
            }
        }
        c.restore()
    }

    override fun onDrawOver(c: Canvas, parent: RecyclerView, state: RecyclerView.State) {
        if (!dataCheck(state)) return
        val drawable = suspendedDrawable ?: return
        //只判断layoutManager为LinearLayoutManager的情况，其他情况的不做处理
        val layoutManager = parent.layoutManager as? LinearLayoutManager ?: return
        val position = layoutManager.findFirstVisibleItemPosition()
        val child = parent.findViewHolderForLayoutPosition(position)?.itemView ?: return
        val rect = Rect().also { parent.getDecoratedBoundsWithMargins(child, it) }

        val currentInitial = contacts[position].initial
        val nextInitial = if (position + 1 < contacts.size) {
            contacts[position + 1].initial
        } else {
            null
        }

        c.save()
        //顶部移出效果
        if (currentInitial != nextInitial && child.top + child.height < drawable.intrinsicHeight) {
            c.translate(0f, (child.height + child.top - drawable.intrinsicHeight).toFloat())
        }
        //绘制Drawable
        val left = parent.paddingLeft
        val top = parent.paddingTop
        val right = parent.right - parent.paddingRight
        val bottom = parent.paddingTop + drawable.intrinsicHeight
        drawable.setBounds(left, top, right, bottom)
        drawable.draw(c)
        //绘制文本内容
        textPaint.getTextBounds(currentInitial, 0, currentInitial.length, rect)
        val textX = child.paddingLeft.toFloat()
        val textY =
            (parent.paddingTop + drawable.intrinsicHeight - (drawable.intrinsicHeight - rect.height()) / 2).toFloat()
        c.drawText(currentInitial, textX, textY, textPaint)
        c.restore()
    }

    override fun getItemOffsets(
        outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State
    ) {
        if (!dataCheck(state)) {
            outRect.set(0,  0, 0, 0)
            return
        }

        val position = (view.layoutParams as RecyclerView.LayoutParams).viewLayoutPosition
        val currentInitial = contacts[position].initial
        val lastInitial = if (position >= 1) {
            contacts[position - 1].initial
        } else {
            null
        }

        //当前首字母与上一个首字母相同 则属于同一组，使用 dividerDrawable
        val drawable =
            if (currentInitial == lastInitial) dividerDrawable else suspendedDrawable
        //设置Item的偏移区域
        outRect.set(0, drawable?.intrinsicHeight ?: 0, 0, 0)
    }

    private fun dataCheck(state: RecyclerView.State): Boolean {
        val dataCorrectly = contacts.size == state.itemCount
        if (!dataCorrectly) {
            Log.e(javaClass.name, "----The contact size does not match RecyclerView Item Count")
        }
        return dataCorrectly
    }
}
```
### 其他
大多数情况下RecyclerView会添加 header 与 footer，此时可以在 SuspendItemDecoration中添加条件判断，是否绘制第一条与最后一条。

## 4. ItemDecoration的使用
```
recyclerView.addItemDecoration(TimeLineItemDecoration(this))
```
