## 前言
LayoutManager是RecyclerView中子Item的布局管理器，可控制Item的位置，回收，显示，大小，滚动等等，常用的有LinearLayoutManager、GridLayoutManager、StaggeredGridLayoutManager、FlexboxLayoutManager。
本文通过实现RecyclerView横向无限循环滑动来简单介绍下LayoutManager的自定义。

## 1. 创建自定义LayoutManager类
创建自定义类`HorizontalLayoutManager`，重新方法，并创建辅助类OrientationHelper。
```kotlin
class HorizontalLayoutManager : RecyclerView.LayoutManager() {
    private val orientationHelper = OrientationHelper.createHorizontalHelper(this)
    
    override fun generateDefaultLayoutParams(): RecyclerView.LayoutParams =
        RecyclerView.LayoutParams(RecyclerView.LayoutParams.WRAP_CONTENT, RecyclerView.LayoutParams.WRAP_CONTENT)

    override fun onLayoutChildren(recycler: RecyclerView.Recycler?, state: RecyclerView.State?) {
        super.onLayoutChildren(recycler, state)
    }
}
```
`generateDefaultLayoutParams` ：必须重写
`onLayoutChildren`：绘制子View
`OrientationHelper`：

此时效果：空白一片。

## 2. 绘制子View
横向滑动列表可以横向向前（手势往左）也可向后（手势往右）滑动，因此在绘制子View也应该考虑方向问题，自定义一个绘制View的方法`layoutChild`。
```kotlin
    /**
     * @param start View起点
     * @param forward 绘制方向是否向前（由左至右）
     */
    private fun layoutChild(view: View, start: Int, forward: Boolean) {
        //测量View 包含 decorations(装饰物/分割线) 和 margins
        measureChildWithMargins(view, 0, 0)
        val childWidth = orientationHelper.getDecoratedMeasurement(view)
        val childHeight = orientationHelper.getDecoratedMeasurementInOther(view)
        
        val left: Int
        val right: Int
        val top = paddingTop
        val bottom = top + childHeight
        if (forward) {
            addView(view)
            left = start
            right = start + childWidth
        } else {
            //由右向左时，添加View至第一个
            addView(view, 0)
            left = start - childWidth
            right = start
        }
        //绘制View
        layoutDecoratedWithMargins(view, left, top, right, bottom)
        orientationHelper.onLayoutComplete()
    }
```
在`onLayoutChildren`中循环绘制子View，直到超出RecyclerView坐标范围为止。
```kotlin
    override fun onLayoutChildren(recycler: RecyclerView.Recycler, state: RecyclerView.State) {
        super.onLayoutChildren(recycler, state)
        //分离并且回收当前附加的所有View
        detachAndScrapAttachedViews(recycler)

        if (itemCount == 0) return
        //除去Padding RecyclerView的最左侧坐标
        var start = orientationHelper.startAfterPadding
        for (i in 0 until itemCount) {
            val child = recycler.getViewForPosition(i)
            layoutChild(child, start, forward = true)
            //当前绘制View的最右侧坐标
            start = orientationHelper.getDecoratedEnd(child)
            //除去Padding RecyclerView的最右侧坐标
            //当前绘制View超出RecyclerView范围时则不再绘制子View
            if (start > orientationHelper.endAfterPadding) {
                break
            }
        }
    }
```
此时效果
- RecyclerView 有固定尺寸(width && height = match_parent) ：显示了子View但无法滑动。
- RecyclerView 无固定尺寸(width || height = wrap_content)：一片空白，还需要对子View进行测量，可查看步骤6 RecyclerView 测量。
## 3. 横向滑动
重写方法允许横向滑动
从此方法可以得到：左滑(向前 forward = true) ：dx > 0 ; 右滑（forward = false）: dx < 0
```kotlin
    override fun canScrollHorizontally(): Boolean = true

    override fun scrollHorizontallyBy(
        dx: Int, recycler: RecyclerView.Recycler, state: RecyclerView.State
    ): Int {
        //日志显示，左滑dx值为正数，右滑dx值为负数
        Log.i("TAG", "----------dx：$dx")
        if (childCount == 0 || dx == 0) return 0

        //原点(0，0)在屏幕左上角，左滑：已显示的View需左移动 -offset 距离，即 -1*dx，右滑相反。
        orientationHelper.offsetChildren(-1 * dx)
        return dx
    }
```
此时效果：可以左右滑动，但滑动后为空白，未绘制剩下的子View
## 4. 滑动时绘制子View
实现步骤
- 左滑：找到当前`显示`的最后一个View，根据它的位置找到下一个View并绘制。
- 右滑：找到当前`显示`的第一个View，根据它的位置找到排在它前面的View并绘制。

### 4.1 找到下一个需要绘制的View
实现无限循环：
- 左滑：当前为最后一个View时，下一个View就是第一个
- 右滑：当前为第一个View时，下一个View就是最后一个
```kotlin
    /**
     * @param currentView
     * @param forward 绘制方向是否向前（由左至右）
     */
    private fun nextView(
        currentView: View, forward: Boolean, recycler: RecyclerView.Recycler
    ): View? {
        val endPosition = itemCount - 1
        val currentPosition = getPosition(currentView)
        val nextViewPosition: Int = if (forward) {
            if (currentPosition == endPosition) 0 else currentPosition + 1
        } else {
            if (currentPosition == 0) endPosition else currentPosition - 1
        }
        return recycler.getViewForPosition(nextViewPosition)
    }
```
### 4.2 滑动时绘制
```kotlin
    private fun fill(dx: Int, recycler: RecyclerView.Recycler) {
        //循环绘制子View 直到没有下一个View为止
        while (true) {
            //下一个View绘制的起始点坐标
            var start: Int
            var currentView: View
            //绘制方向
            val forward = dx > 0

            if (forward) {
                val lastVisibleView = getChildAt(childCount - 1) ?: break
                start = orientationHelper.getDecoratedEnd(lastVisibleView)
                if (start - dx > orientationHelper.endAfterPadding) break

                currentView = lastVisibleView
            } else {
                val firstVisibleView = getChildAt(0) ?: break
                start = orientationHelper.getDecoratedStart(firstVisibleView)
                if (start - dx < orientationHelper.startAfterPadding) break

                currentView = firstVisibleView
            }

            val nextView = nextView(currentView, forward, recycler) ?: break
            //绘制View
            layoutChild(nextView, start, forward)
        }
    }

    override fun scrollHorizontallyBy(
        dx: Int, recycler: RecyclerView.Recycler, state: RecyclerView.State
    ): Int {
        ...
        fill(dx, recycler)
        ...
    }
```

此时状态：实现了横向无限循环滑动，但会发现滑动时在不停的调用 adapter的 onCreateViewHolder方法，即未实现RecyclerView的回收复用功能。
## 5. 回收子View
RecyclerView自己实现了回收复用功能，我们只需要在LayoutManager中调用相关方法即可。当子View超出屏幕范围便进行回收。
```kotlin
    private fun recycleViews(dx: Int, recycler: RecyclerView.Recycler) {
        for (i in 0 until itemCount) {
            val childView = getChildAt(i) ?: return
            //左滑
            if (dx > 0) {
                //移除并回收 原点 左侧的子View
                if (orientationHelper.getDecoratedEnd(childView) - dx <
                    orientationHelper.startAfterPadding
                ) {
                    removeAndRecycleViewAt(i, recycler)
                }
            } else { //右滑
                //移除并回收 右侧即RecyclerView宽度之以外的子View
                if (orientationHelper.getDecoratedStart(childView) - dx >
                    orientationHelper.endAfterPadding
                ) {
                    removeAndRecycleViewAt(i, recycler)
                }
            }
        }
    }
```
然后在 `scrollHorizontallyBy`中调用即可实现滑动时边绘制边回收。
## 6. RecyclerView 测量
在步骤2 中就发现，当RecyclerView未确定宽高即设置为 wrap_content 时，画面一片空白，此时需要对Recyclerview进行测量，通过重写LayoutManager的onMeasure方法可以实现。
方法调用流程：LayoutManager.onMeasure() -> uper.onMeasure() -> RecyclerView.defaultOnMeasure() -> setMeasuredDimension()

步骤：重写LayoutManager.onMeasure方法，当RecyclerView未确定宽高时，通过测量子View根据子View的宽高来确定RecyclerView宽高。
```kotlin
    override fun onMeasure(
        recycler: RecyclerView.Recycler,
        state: RecyclerView.State,
        widthSpec: Int,
        heightSpec: Int
    ) {
        var remeasureWidthSpec = widthSpec
        var remeasureHeightSpec = heightSpec

        val widthMode = MeasureSpec.getMode(widthSpec)
        val heightMode = MeasureSpec.getMode(heightSpec)

        if (widthMode == MeasureSpec.AT_MOST) {
            var measureWidth = MeasureSpec.getSize(widthSpec)
            //子View的宽之和
            val itemWidthSum = (0 until itemCount)
                .mapNotNull { recycler.getViewForPosition(it) }
                .map { measureChildView(it, widthSpec, heightSpec).first }
                .sum()

            if (itemWidthSum < measureWidth) {
                measureWidth = itemWidthSum
            }
            remeasureWidthSpec = MeasureSpec.makeMeasureSpec(measureWidth, MeasureSpec.EXACTLY)
        }

        if (heightMode == MeasureSpec.AT_MOST) {
            var measureHeight = MeasureSpec.getSize(heightSpec)
            //子View中最大的高度
            val itemMaxHeight = (0 until itemCount)
                .mapNotNull { recycler.getViewForPosition(it) }
                .map { measureChildView(it, widthSpec, heightSpec).second }
                .max() ?: 0

            if (itemMaxHeight < measureHeight) {
                measureHeight = itemMaxHeight
            }
            remeasureHeightSpec = MeasureSpec.makeMeasureSpec(measureHeight, MeasureSpec.EXACTLY)
        }
        super.onMeasure(recycler, state, remeasureWidthSpec, remeasureHeightSpec)
    }

    /**
     * 测量子View
     *
     * @return  Pair<Int, Int> : first 为宽，second 为高
     */
    private fun measureChildView(childView: View, widthSpec: Int, heightSpec: Int): Pair<Int, Int> {
        val layoutParams = childView.layoutParams as RecyclerView.LayoutParams
        val childHeightSpec = ViewGroup.getChildMeasureSpec(
            heightSpec, paddingTop + paddingBottom, layoutParams.height
        )
        childView.measure(widthSpec, childHeightSpec)

        val width = childView.measuredWidth + layoutParams.leftMargin + layoutParams.rightMargin
        val height = childView.measuredHeight + layoutParams.bottomMargin + layoutParams.topMargin

        return Pair(width, height)
    }
```
此时效果：已实现横向无限循环滑动，且具有回收复用功能。
