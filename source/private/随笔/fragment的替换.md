Activity的布局中用一个FragmentContainerView来盛装

```xml
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/fragment_container"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"/>
```

Activity中

```kotlin
private val fragmentList = ArrayList<Fragment>()
private val fragmentManager = supportFragmentManager
private val profileFragment by lazy { ScientistProfileFragment() }
private val worksFragment by lazy { ScientistWorksFragment() }
private val dynamicFragment by lazy { ScientistDynamicFragment() }

init {
    fragmentList.add(profileFragment)
    fragmentList.add(worksFragment)
    fragmentList.add(dynamicFragment)
}

private fun initFragment() {
    val transaction = fragmentManager.beginTransaction()
    transaction.add(R.id.fragment_container, fragmentList[0])
    transaction.addToBackStack(null)
    transaction.commit()
}

private fun switchFragment(index: Int) {
    val transaction = fragmentManager.beginTransaction()
    transaction.replace(R.id.fragment_container, fragmentList[index])
    transaction.addToBackStack(null)
    transaction.commit()
}
```

注意这里的transaction应该声明为局部变量