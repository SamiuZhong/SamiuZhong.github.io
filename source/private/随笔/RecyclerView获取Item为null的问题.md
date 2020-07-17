对于一个RecyclerView，当我们采用如下方式直接获取指定item的时候，会发现返回的是null

```kotlin
val layoutManager = binding.recycler.layoutManager as LinearLayoutManager
val view = layoutManager.findViewByPosition(0)
```

那是因为在获取的这个时候View还没有绘制出来，需要采用以下的方式来获取

```kotlin
var view: View? = null
binding.recycler.viewTreeObserver.addOnGlobalLayoutListener(object : ViewTreeObserver.OnGlobalLayoutListener {
    override fun onGlobalLayout() {
        val layoutManager = binding.recycler.layoutManager as LinearLayoutManager
        view = layoutManager.findViewByPosition(0)
        binding.recycler.viewTreeObserver.removeOnGlobalLayoutListener(this)
    }
})
```

