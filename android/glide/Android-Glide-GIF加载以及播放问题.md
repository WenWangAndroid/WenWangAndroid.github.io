本文使用的Glide版本 4.9.0

## 1. Gif加载
普通图片和Gif一起加载
```kotlin
 Glide.with(imageView).load(url).into(imageView)
```
只加载Gif图，如果不是Gif图会调用error()方法。
```kotlin
Glide.with(imageView).asGif().load(url).error(R.mipmap.ic_launcher).into(imageView)
```
## 2. 设置Gif播放次数
实现方式：添加RequestListener，在回调方法`onResourceReady`中得到`GifDrawable`，然后设置播放次数
```kotlin
Glide.with(imageView)
    .load(url)
    .listener(object : RequestListener<Drawable> {
        override fun onLoadFailed(
            e: GlideException?,
            model: Any?,
            target: Target<Drawable>?,
            isFirstResource: Boolean
        ): Boolean = false

        override fun onResourceReady(
            resource: Drawable?,
            model: Any?, target:
            Target<Drawable>?,
            dataSource: DataSource?,
            isFirstResource: Boolean
        ): Boolean {
            if (resource is GifDrawable) {
                resource.setLoopCount(GifDrawable.LOOP_INTRINSIC) //设置次数
            }
            return false
        }
    })
    .into(imageView)
```
### 2.1 关键方法
```kotlin
GifDrawable.setLoopCount(int loopCount)
//值
GifDrawable.LOOP_FOREVER    //设置GIF图循环播放
GifDrawable.LOOP_INTRINSIC  //设置GIF图的为默认播放次数
```
### 2.2 Gif默认播放次数
Gif资源本身是可以设置播放次数的，可参考： [用PS修改GIF动图循环播放次数](https://www.jianshu.com/p/7895975c33f4)
平时打开Gif图默认是循环播放的，大多是因为看图软件实现了循环播放的原因，可使用浏览器打开Gif图，基本是播放默认次数后就会停止播放。

## 3. 设置Gif重新播放
使用Glide加载Gif，在Gif未播放完的情况下，进行页面切换、跳转，返回后是继续上一状态进行播放的，而不是重新播放。

核心代码
```kotlin
if (drawable is GifDrawable) {
    drawable.setLoopCount(1)
    drawable.stop()
    drawable.startFromFirstFrame()
}
```
**注意：** 不要在Glide加载Gif的时候，调用startFromFirstFrame()，否则使用会出现后续描述中发生的崩溃。

## 4. startFromFirstFrame导致崩溃
错误日志：
```
java.lang.IllegalArgumentException: 
Pending target must be null when starting from the first frame
```
查看源码
```kotlin
  com.bumptech.glide.load.resource.gif.GifFrameLoader

  private DelayTarget pendingTarget;
  ...
  private void loadNextFrame() {
    ...
    if (startFromFirstFrame) {
      Preconditions.checkArgument(
          pendingTarget == null,
         "Pending target must be null when starting from the first frame"
      );
      gifDecoder.resetFrameIndex();
      startFromFirstFrame = false;
    }
    ...
  }
```
想要 startFromFirstFrame ，属性pendingTarget必须为null，当出现多个地方对pendingTarget属性同时赋值时，然后执行loadNextFrame() 就会抛出异常，导致崩溃。

### 4.1 多个相同Gif资源重新播放导致崩溃
在RecyclerView中，加载了两个相同url的gif图，设置startFromFirstFrame()后，滑动时崩溃。
解决方式：根据加载的View的不同，给Gif资源设置Tag以便区分Gif，GifDrawable。
```kotlin
Glide.with(imageView) //position 为itemView的位置
    .setDefaultRequestOptions(RequestOptions().signature(ObjectKey(position)))
    .load(url)
    .addListener(...)
    .into(imageView)
```
### 4.2 Glide加载Gif时在其它地方设置重新播放导致崩溃
在ImageView的onVisibilityChanged()中获取GifDrawable并调用了startFromFirstFrame()，然后在某些情况下（如RecyclerView滑动过程中）ImageView发生onVisibilityChanged()时，也会进行onBindView，Glide重新加载Gif，然后发生崩溃。
解决方式：避免此类情况发生。

### 4.3 需求的实现
代码片段一：
```kotlin
Glide.with(imageView)
    //position 为itemView的位置
    .setDefaultRequestOptions(RequestOptions().signature(ObjectKey(position)))
    .load(url)
    .addListener(object : RequestListener<Drawable> {
        override fun onLoadFailed(
            e: GlideException?,
            model: Any?,
            target: Target<Drawable>?,
            isFirstResource: Boolean
        ): Boolean = false

        override fun onResourceReady(
            resource: Drawable?,
            model: Any?,
            target: Target<Drawable>?,
            dataSource: DataSource?,
            isFirstResource: Boolean
        ): Boolean {
            if (resource is GifDrawable) {
                resource.setLoopCount(1) //设置次数
                resource.stop()
                resource.startFromFirstFrame()
            }
            return false
        }
    })
    .error(R.mipmap.ic_launcher)
    .into(imageView)
```

代码片段二：
```kotlin
class GifImageView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : ImageView(context, attrs, defStyleAttr) {
    override fun onVisibilityChanged(changedView: View, visibility: Int) {
        super.onVisibilityChanged(changedView, visibility)
        if (visibility == View.VISIBLE) {
            val imageDrawable = drawable
            if (imageDrawable is GifDrawable) {
                imageDrawable.stop()
                imageDrawable.startFromFirstFrame()
            }
        }
    }
}
```
根据场景的不同合理使用代码一与代码二中的代码。
如果Glide加载Gif的代码至始至终只会调用一次，那就通过代码二来实现需求，
如果Glide加载Gif的代码会多次调用，那通过再次调用代码一，例如通过adapter.notifyDataSetChanged()来实现。

