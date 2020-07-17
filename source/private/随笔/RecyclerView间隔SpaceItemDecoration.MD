## 1. 新建一个间隔的类

```kotlin
class SpaceItemDecoration(space: Int) : RecyclerView.ItemDecoration() {

    private val mSpace = space

    override fun getItemOffsets(outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State) {
        super.getItemOffsets(outRect, view, parent, state)
        outRect.top = 0
        outRect.bottom = mSpace
    }
}
```

2.调用

```kotlin
recyclerView.run {
            layoutManager = LinearLayoutManager(context)
            addItemDecoration(SpaceItemDecoration(10))
        }
```



## 2. Java写法

```Java
public class SpaceItemDecoration extends RecyclerView.ItemDecoration {

    private int mSpace;
    private int mTopSpace;
    private int mBottomSpace;
    private int mLeftSpace;
    private int mRightSpace;

    public SpaceItemDecoration(int mSpace) {
        this.mSpace = mSpace;
    }

    public SpaceItemDecoration(int mTopSpace, int mBottomSpace, int mLeftSpace, int mRightSpace) {
        this.mTopSpace = mTopSpace;
        this.mBottomSpace = mBottomSpace;
        this.mLeftSpace = mLeftSpace;
        this.mRightSpace = mRightSpace;
    }

    @Override
    public void getItemOffsets(@NonNull Rect outRect, @NonNull View view, @NonNull RecyclerView parent, @NonNull RecyclerView.State state) {
        super.getItemOffsets(outRect, view, parent, state);
        if (mSpace != 0) {
            outRect.bottom = mSpace;
            outRect.top = mSpace;
            outRect.left = mSpace;
            outRect.right = mSpace;
        } else {
            outRect.bottom = mBottomSpace;
            outRect.top = mTopSpace;
            outRect.left = mLeftSpace;
            outRect.right = mRightSpace;
        }
    }
}
```

调用

```java
        binding.recycler.addItemDecoration(new SpaceItemDecoration(0, 10, 0, 0));
```

