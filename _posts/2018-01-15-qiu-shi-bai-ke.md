---
layout: post
title: "2. QiuShiBaiKe 糗事百科"
author: iosdevlog
date: 2018-01-15 21:17:38 +0800
description: ""
category: 
tags: []
---

第2个应用是 *糗事百科*。

![002.QiuShiBaiKe](https://raw.githubusercontent.com/iOSDevLog/1Day1App/master/screenshot/002.QiuShiBaiKe.jpeg)


## 分析一下需求。

1. 从 <http://qiushibaike.com> 取 `json` 数据
1. `json`数据转换为 `kotlin` **data class**
1. 展示数据

## 开发

* 创建 *Android Studio* 项目

* *AndroidManifest.xml* 添加顶部窗口权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
```

* *build.gradle(Module:app)* 添加依赖库

```
implementation 'com.github.CymChad:BaseRecyclerViewAdapterHelper:2.9.34'
implementation 'io.reactivex.rxjava2:rxjava:2.1.0'
implementation 'io.reactivex.rxjava2:rxandroid:2.0.1'
implementation 'com.squareup.retrofit2:retrofit:2.3.0'
implementation 'com.jakewharton.retrofit:retrofit2-rxjava2-adapter:1.0.0'
implementation 'com.squareup.retrofit2:converter-gson:2.3.0'
implementation 'com.squareup.picasso:picasso:2.5.2'
implementation 'com.android.support:recyclerview-v7:27.0.2'
implementation 'com.android.support:cardview-v7:27.0.2'
```

* 创建 *QBEntity*

利用插件[JsonToKotlinClass](https://github.com/wuseal/JsonToKotlinClass)

* 创建 *IQBRequest*

```kotlin
interface IQBRequest {
    @GET("article/list/text")
    fun fetchQBEntity(@Query("page") page: Int): Observable<QBEntity>
}
```

* 测试 *IQBRequest*

```kotlin
class QBRequestUnitTest {

    private val BASE_URL = "http://m2.qiushibaike.com/"

    @Test
    fun request_isSuccess() {
        val retrofit = Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build()

        val qbRequest = retrofit.create(IQBRequest::class.java)

        val page = 1
        qbRequest.fetchQBEntity(page)
                .subscribe({ result ->
                    println(result)
                    assert(result.items.count() > 0)
                    assert(result.count > 0)
                }, { error ->
                    println("onError")
                    error.printStackTrace()
                    assert(false)
                }, {
                    println("onComplete")
                    assert(true)
                }, {
                    println("start")
                })
    }
}
```

* 创建显示 cell

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.v7.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/card_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    android:layout_marginLeft="5dp"
    android:layout_marginRight="5dp"
    android:foreground="?android:attr/selectableItemBackground">

    <LinearLayout
        android:id="@+id/constraintlayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">

        <ImageView
            android:id="@+id/head_imageview"
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:src="@drawable/iosdevlog" />

        <TextView
            android:id="@+id/content_textview"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />

    </LinearLayout>

</android.support.v7.widget.CardView>
```

* 创建*刷新 Adapter*

```kotlin
class PullToRefreshAdapter : BaseQuickAdapter<Item, BaseViewHolder>(R.layout.qb_cell, ArrayList<Item>()) {

    override fun convert(helper: BaseViewHolder, item: Item) {
        helper.setImageResource(R.id.head_imageview, R.drawable.iosdevlog)
        helper.setText(R.id.content_textview, item.content)
    }
}
```

* 在 *activity_main* 添加 `RecyclerView`

```xml
<?xml version="1.0" encoding="utf-8"?>
<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.iosdevlog.qiushibaike.MainActivity">

    <android.support.v4.widget.SwipeRefreshLayout
        android:id="@+id/swipeLayout"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        app:layout_constraintTop_toTopOf="parent">

        <android.support.v7.widget.RecyclerView
            android:id="@+id/recycleview"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
    </android.support.v4.widget.SwipeRefreshLayout>

</android.support.constraint.ConstraintLayout>
```

* *MainActivity* 初始化

```kotlin
    // LifeCycle
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Thread.setDefaultUncaughtExceptionHandler(this)

        mRecyclerView = findViewById(R.id.recycleview)
        mSwipeRefreshLayout = findViewById(R.id.swipeLayout)
        mSwipeRefreshLayout.setOnRefreshListener(this)
        mSwipeRefreshLayout.setColorSchemeColors(Color.YELLOW)
        mRecyclerView.layoutManager = LinearLayoutManager(this)

        initAdapter()
    }

    // Helper
    private fun initAdapter() {
        pullToRefreshAdapter = PullToRefreshAdapter()
        pullToRefreshAdapter.setOnLoadMoreListener(this, mRecyclerView)
        pullToRefreshAdapter.openLoadAnimation(BaseQuickAdapter.SLIDEIN_BOTTOM)
        mRecyclerView.adapter = pullToRefreshAdapter

        mRecyclerView.addOnItemTouchListener(object : OnItemClickListener() {
            override fun onSimpleItemClick(adapter: BaseQuickAdapter<*, *>, view: View, position: Int) {
                Toast.makeText(this@MainActivity, Integer.toString(position), Toast.LENGTH_LONG).show()
            }
        })
    }
```

* *MainActivity*实现`BaseQuickAdapter.RequestLoadMoreListener`, 1SwipeRefreshLayout.OnRefreshListener`

```xml
    // Refresh
    override fun onLoadMoreRequested() {
        mSwipeRefreshLayout.isEnabled = false

        fetch(currentPage).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe({ result ->
                    Log.d(TAG, result.toString())

                    pullToRefreshAdapter.addData(result.items)
                }, { error ->
                    error.printStackTrace()

                    mSwipeRefreshLayout.isEnabled = true
                    mSwipeRefreshLayout.isRefreshing = false
                    Toast.makeText(this@MainActivity, R.string.network_error, Toast.LENGTH_LONG).show()
                    pullToRefreshAdapter.setEnableLoadMore(true)

                    pullToRefreshAdapter.loadMoreFail()
                }, {
                    Log.d(TAG, "onComplete")

                    currentPage += 1
                    mSwipeRefreshLayout.isEnabled = true
                    mSwipeRefreshLayout.isRefreshing = false
                    pullToRefreshAdapter.setEnableLoadMore(true)

                    pullToRefreshAdapter.loadMoreComplete()
                }, {
                    Log.d(TAG, "onStart")
                })
    }

    override fun onRefresh() {
        pullToRefreshAdapter.setEnableLoadMore(false)

        currentPage = 1
        fetch(currentPage).subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe({ result ->
                    Log.d(TAG, result.toString())

                    pullToRefreshAdapter.setNewData(result.items)
                }, { error ->
                    Log.d(TAG, "onError")

                    error.printStackTrace()
                    mSwipeRefreshLayout.isRefreshing = false
                    pullToRefreshAdapter.setEnableLoadMore(true)
                }, {
                    Log.d(TAG, "onComplete")
                    mSwipeRefreshLayout.isRefreshing = false
                    pullToRefreshAdapter.setEnableLoadMore(true)
                    currentPage += 1
                }, {
                    Log.d(TAG, "onStart")
                })
    }

    private fun fetch(page: Int): Observable<QBEntity> {
        val retrofit = Retrofit.Builder()
                .baseUrl(BASE_URL)
                .addConverterFactory(GsonConverterFactory.create())
                .addCallAdapterFactory(RxJava2CallAdapterFactory.create())
                .build()

        val qbRequest = retrofit.create(IQBRequest::class.java)

        return qbRequest.fetchQBEntity(page)
    }
```

* （可选）*MainActivity*实现 `Thread.UncaughtExceptionHandler`

```kotlin
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Thread.setDefaultUncaughtExceptionHandler(this)
	...
    }

    // UncaughtExceptionHandler
    override fun uncaughtException(t: Thread?, e: Throwable?) = e!!.printStackTrace()
```

* 界面可显时显示最新数据

```kotlin
    override fun onResume() {
        super.onResume()

        Handler().postDelayed({
            mSwipeRefreshLayout.isRefreshing = true
            onRefresh()
        }, 10)
    }
```

* 加载用户信息

参考：<https://gist.github.com/julianshen/5829333>

```kotlin
    override fun convert(helper: BaseViewHolder, item: Item) {
        if (item.user != null) {
            val imageView = helper.getView<ImageView>(R.id.head_imageview)
            Picasso.with(mContext)
                    .load("http:" + item.user.medium)
                    .resize(40, 40)
                    .centerCrop()
                    .transform(CircleTransform())
                    .into(imageView)
            helper.setText(R.id.username_textview, item.user.login)
            helper.setText(R.id.username_detail, item.user.age.toString() + " " + item.user.astrology + " 赞: " + item.votes.up + " 踩: " + item.votes.down)

            val gender = when (item.user.gender) {
                "F" -> R.drawable.female
                "M" -> R.drawable.male
                else -> R.drawable.male
            }
            helper.setImageResource(R.id.gender_imageview, gender)
        }

        val published_at = java.lang.Long.valueOf(item.published_at)
        val simpleDateFormat = SimpleDateFormat("yyyy-MM-dd\nHH:mm:ss")
        val published_string = simpleDateFormat.format( Date(published_at * 1000L));
        helper.setText(R.id.datetime_textview, published_string)
        helper.setText(R.id.content_textview, item.content)
    }

    inner class CircleTransform : Transformation {
        override fun transform(source: Bitmap): Bitmap {
            val size = Math.min(source.width, source.height)

            val x = (source.width - size) / 2
            val y = (source.height - size) / 2

            val squaredBitmap = Bitmap.createBitmap(source, x, y, size, size)
            if (squaredBitmap != source) {
                source.recycle()
            }

            val config = if (source.config != null) source.config else Bitmap.Config.ARGB_8888
            val bitmap = Bitmap.createBitmap(size, size, config)

            val canvas = Canvas(bitmap)
            val paint = Paint()
            val shader = BitmapShader(squaredBitmap, Shader.TileMode.CLAMP, Shader.TileMode.CLAMP)
            paint.setShader(shader)
            paint.setAntiAlias(true)

            val r = size / 2f
            canvas.drawCircle(r, r, r, paint)

            squaredBitmap.recycle()
            return bitmap
        }

        override fun key(): String {
            return "circle"
        }
    }
```
