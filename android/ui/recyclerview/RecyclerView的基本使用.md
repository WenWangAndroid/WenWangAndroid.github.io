# RecyclerView的简单用法
## 1. 基本使用
### 1.1 依赖
Recyclerview属于android.support.v7.widget包下的控件，需要进行远程依赖。
```
dependencies {
    implementation 'com.android.support:appcompat-v7:28.0.0'
    implementation 'com.android.support:recyclerview-v7:28.0.0'
}
```
### 1.2 使用
现在准备实现的是展示一个图片列表，需要进行如下步骤：

 - 编写布局文件
 - 编写RecyclerView.Adapter：将数据与RecyclerView中子View绑定显示。
 - 编写RecyclerView.ViewHolder：装载子View及其元素。
 - RecyclerViews属性设置：Adapter、LayoutManager(布局管理)、ItemDecoration（分割线）等等。

#### 1.2.1 子布局文件
父布局
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout
        xmlns:android="http://schemas.android.com/apk/res/android"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

    <android.support.v7.widget.RecyclerView
            android:id="@+id/recyclerView"
            android:layout_width="match_parent"
            android:layout_height="match_parent"/>
</android.support.constraint.ConstraintLayout>
```
子布局
图片横纵比 3:2
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <ImageView
        android:id="@+id/imageView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:scaleType="fitXY"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintDimensionRatio="3:2"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
</android.support.constraint.ConstraintLayout>
```
#### 1.2.2 编写Adapter
自定义Adapter继承RecyclerView.Adapter，且必须重写三个方法，另外还需把要展示的列表数据传入Adapter中
```
class ImageRecycleAdapter : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    //需要展示的图片列表数据
    var imageResources: List<Int> = ArrayList()
        set(value) {
            field = value
            //数据已改变，通知RecyclerView刷新
            notifyDataSetChanged()
        }
    // 创建一个承载子项视图的ViewHolder
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        TODO("not implemented")
    }
    //返回列表子项长度
    override fun getItemCount(): Int = imageResources.size
    //将指定位置的数据与视图绑定
    override fun onBindViewHolder(viewHolder: RecyclerView.ViewHolder, position: Int) {
        TODO("not implemented")
    }
}
```
1.2.3 编写ViewHolder

自定义ViewHolder，继承于RecyclerView.ViewHolder
```
class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
    //将数据与View绑定
    fun bindView(@DrawableRes imageResource : Int){
        /**
         * 为什么没使用FindViewById呢？
         * 因为 import kotlinx.android.synthetic.main.recycle_item_image.view.*
         * 创建Android Kotlin 工程会自动引入扩展插件 apply plugin: 'kotlin-android-extensions'
         * 详情可查看：https://www.kotlincn.net/docs/tutorials/android-plugin.html
         */
        itemView.imageView.setImageResource(imageResource)
    }
}
```
然后替换Adapter中的RecyclerView.ViewHolder为自定义的ViewHolder，完整的Adapter + ViewHolder 代码如下
```
class ImageRecycleAdapter : RecyclerView.Adapter<ImageRecycleAdapter.ViewHolder>() {
    //需要展示的图片列表数据
    var imageResources: List<Int> = ArrayList()
        set(value) {
            field = value
            //数据已改变，通知RecyclerView刷新
            notifyDataSetChanged()
        }

    // 创建一个承载子项视图的ViewHolder
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ImageRecycleAdapter.ViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.recycle_item_image, parent, false)
        return ViewHolder(view)
    }

    //返回列表子项长度
    override fun getItemCount(): Int = imageResources.size

    //将指定位置的数据与视图绑定
    override fun onBindViewHolder(viewHolder: ImageRecycleAdapter.ViewHolder, position: Int) {
        viewHolder.bindView(imageResources[position])
    }

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        //将数据与View绑定
        fun bindView(@DrawableRes imageResource: Int) {
            itemView.imageView.setImageResource(imageResource)
        }
    }
}
```
#### 1.2.4 RecyclerView设置
```
class RecyclerViewActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_recycle_view)

        //添加10张图片
        val imageList = ArrayList<Int>()
        for (i in 0 until 10) {
            imageList.add(R.mipmap.image)
        }

        val imageAdapter = ImageRecycleAdapter().apply {
            imageResources = imageList
        }
        val linearLayoutManager = LinearLayoutManager(this)
        //添加分割线
        val itemDecoration = DividerItemDecoration(this, DividerItemDecoration.VERTICAL)
        recyclerView.apply {
            layoutManager = linearLayoutManager
            adapter = imageAdapter
            addItemDecoration(itemDecoration)
        }
    }
}
```
#### 1.2.5 效果图

## 2. 多类型列表
很多时候我们需要在一个列表下展示不同式样的View，如下图所示


### 2.1 思路
多类型列表展示可采用RecyclerView 提供的两个方法来实现
```
override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
    TODO("not implemented") 
}

override fun getItemViewType(position: Int): Int {
    return super.getItemViewType(position)
}
```
通过 getItemViewType方法传入不同的 ViewType，然后 onCreateViewHolder 根据不同的ViewType 创建不同的ViewHolder，然后在根据不同的ViewHolder类型，进行 onBindViewHolder

2.2 编码
2.2.1 新增子项布局  
新建 recycle_item_text_image.xml ，为了方便，只是变更文本内容。
```
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <TextView
        android:id="@+id/titleTextView"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginTop="16dp"
        android:layout_marginBottom="16dp"
        android:textSize="15sp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@+id/imageView_normal"
        app:layout_constraintHorizontal_weight="5"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        tools:text="你的名字" />

    <ImageView
        android:id="@+id/imageView_normal"
        android:layout_width="0dp"
        android:layout_height="0dp"
        android:layout_marginStart="8dp"
        android:layout_marginTop="16dp"
        android:layout_marginBottom="16dp"
        android:scaleType="fitXY"
        android:src="@mipmap/image_positive"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintDimensionRatio="3:2"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintHorizontal_weight="3"
        app:layout_constraintStart_toEndOf="@+id/titleTextView"
        app:layout_constraintTop_toTopOf="parent" />
</android.support.constraint.ConstraintLayout>
```
2.2.2 新增ViewHolder
```
    class TextImageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bindView(titleStr: String) {
            itemView.titleTextView.text = titleStr
        }
    }
```
 2.2.3 重写getItemViewType
```
override fun getItemViewType(position: Int): Int {
    val data = items[position]
    //如果数据是 Int 则只展示图片，如果是String 则展示 文本+图片
    return when (data) {
        is Int -> VIEW_TYPE_IMAGE
        else -> VIEW_TYPE_TEXT_IMAGE
    }
}
```
 此处是根据数据类型的不同，展示不同的视图，也可以采用 根据位置的不同展示不同的视图，例如添加 Header 以及 Footer
```
override fun getItemViewType(position: Int): Int {
    return when (position) {
        0 -> 0//Header
        itemCount - 1 -> 1 //Footer
        else -> 2 //Normal
    }
}
```
2.2.4 修改后的Adapter

```

class ImageRecycleAdapter : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    //需要展示的图片列表数据
    var items: List<Any> = ArrayList()
        set(value) {
            field = value
            //数据已改变，通知RecyclerView刷新
            notifyDataSetChanged()
        }

    // 创建一个承载子项视图的ViewHolder
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        val inflater = LayoutInflater.from(parent.context)

        return when (viewType) {
            VIEW_TYPE_IMAGE ->
                ViewHolder(inflater.inflate(R.layout.recycle_item_image, parent, false))
            else ->
                TextImageViewHolder(inflater.inflate(R.layout.recycle_item_text_image, parent, false))
        }
    }

    //返回列表子项长度
    override fun getItemCount(): Int = items.size

    //将指定位置的数据与视图绑定
    override fun onBindViewHolder(viewHolder: RecyclerView.ViewHolder, position: Int) {
        val data = items[position]

        when (viewHolder) {
            is ViewHolder ->
                if (data is Int) viewHolder.bindView(data)

            is TextImageViewHolder ->
                if (data is String) viewHolder.bindView(data)
        }
    }

    override fun getItemViewType(position: Int): Int {
        val data = items[position]
        //如果数据是 Int 则只展示图片，如果是String 则展示 文本+图片
        return when (data) {
            is Int -> VIEW_TYPE_IMAGE
            else -> VIEW_TYPE_TEXT_IMAGE
        }
    }

    class ViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        //将数据与View绑定
        fun bindView(@DrawableRes imageResource: Int) {
            itemView.imageView.setImageResource(imageResource)
        }
    }

    class TextImageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bindView(titleStr: String) {
            itemView.titleTextView.text = titleStr
        }
    }

    companion object {
        const val VIEW_TYPE_IMAGE = 1
        const val VIEW_TYPE_TEXT_IMAGE = 2
    }
}
```
2.2.5 实现效果
在1.2.4 Activity的基础上，将模拟数据修改一下即可，实现效果。
```
class RecyclerViewActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        ......

        val dataList = ArrayList<Any>()
        for (i in 0 until 10) {
            if (i % 3 == 0)
                dataList.add(R.mipmap.image)
            else
                dataList.add(resources.getString(R.string.positive_content))
        }

        val imageAdapter = ImageRecycleAdapter().apply {
            items = dataList
        }
 
        ......
    }
}
```
