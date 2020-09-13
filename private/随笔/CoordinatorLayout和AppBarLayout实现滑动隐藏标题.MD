主要要注意的就是：

0. 注意：下面第4点提到的Layout必须是一个可滑动的布局才行，如RecyclerView,NestedScrollView等，放不可滑动的布局是不起作用的。
1. 最外层布局用CoordinatorLayout
2. 用AppBarLayout包裹标题栏，设置fitsSystemWindows=true属性
3. AppBarLayout里面包裹标题栏的实现，设置layout_scrollFlags属性，注意这里的flag必须要有一个scroll再加另外一个
4. 下面滑动的内容放在和AppBarLayout平级，必须要加个app:layout_behavior="@string/appbar_scrolling_view_behavior"这个属性才行

```xml
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:fitsSystemWindows="true"
        android:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar">

        <com.google.android.material.tabs.TabLayout
            android:id="@+id/tab"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:layout_scrollFlags="scroll|enterAlways"
            app:tabIndicatorColor="@color/color_29E7CD"
            app:tabSelectedTextColor="@color/color_29E7CD"
            app:tabTextColor="@color/white" />
    </com.google.android.material.appbar.AppBarLayout>

    <com.scwang.smartrefresh.layout.SmartRefreshLayout
        android:id="@+id/homeRefreshLayout"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_behavior="@string/appbar_scrolling_view_behavior">

        <com.scwang.smartrefresh.header.MaterialHeader
            android:layout_width="match_parent"
            android:layout_height="wrap_content" />

        <androidx.viewpager2.widget.ViewPager2
            android:id="@+id/pager"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:overScrollMode="never" />
    </com.scwang.smartrefresh.layout.SmartRefreshLayout>
</androidx.coordinatorlayout.widget.CoordinatorLayout>
```
