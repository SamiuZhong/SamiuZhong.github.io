## 布局文件

```
<com.google.android.material.tabs.TabLayout
    android:id="@+id/tab"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginTop="8dp"
    app:tabIndicatorColor="@color/color_D08B4E"
    app:tabIndicatorHeight="3dp"
    app:tabMode="scrollable"
    app:tabSelectedTextColor="@color/color_D08B4E"
    app:tabTextColor="@color/default_hint_text_color_C6C8D0" />

<androidx.viewpager2.widget.ViewPager2
    android:id="@+id/pager"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

## 代码使用
```
private val fragmentList = ArrayList<Fragment>()
private var titleList by Delegates.notNull<Array<String>>()

init{
    fragmentList.add(SearchVideoFragment())      //视频
    fragmentList.add(SearchLiveFragment())       //直播
    fragmentList.add(SearchCollectionFragment()) //合集
    fragmentList.add(SearchScientistFragment())  //科学家
}

override fun initData() {
    titleList = arrayOf(
            Constants.VIDEO, Constants.LIVE, Constants.COLLECTION, Constants.SCIENTIST
    )
    
    binding.searchPager.adapter = ScreenPagerAdapter(this)
    TabLayoutMediator(binding.searchTab, binding.searchPager) { tab, position ->
        tab.text = titleList[position]
    }.attach()
}

private inner class ScreenPagerAdapter(fragment: Fragment) : FragmentStateAdapter(fragment) {
    override fun getItemCount() = fragmentList.size
    override fun createFragment(position: Int) = fragmentList[position]
}
```
