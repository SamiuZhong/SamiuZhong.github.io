Github: [ExpandableRecyclerView](https://github.com/hgDendi/ExpandableRecyclerView)

Adapter实现：

```kotlin
class CourseListAdapter(
    private val mList: ArrayList<CourseGroupBean>
) : BaseExpandableRecyclerViewAdapter<CourseGroupBean, CourseChildBean, CourseGroupVH, CourseChildVH>() {

    override fun getGroupCount() = mList.size
    override fun getGroupItem(groupIndex: Int) = mList[groupIndex]

    override fun onCreateGroupViewHolder(parent: ViewGroup?, groupViewType: Int): CourseGroupVH {
        return CourseGroupVH(
            LayoutInflater.from(parent?.context)
                .inflate(R.layout.item_course_list, parent, false)
        )
    }

    override fun onCreateChildViewHolder(parent: ViewGroup?, childViewType: Int): CourseChildVH {
        return CourseChildVH(
            LayoutInflater.from(parent?.context)
                .inflate(R.layout.item_course_list_child, parent, false)
        )
    }

    override fun onBindGroupViewHolder(
        holder: CourseGroupVH,
        groupBean: CourseGroupBean,
        isExpand: Boolean
    ) = with(holder.itemView) {
        outline_text.text = groupBean.mName
    }


    override fun onBindChildViewHolder(
        holder: CourseChildVH,
        groupBean: CourseGroupBean,
        childBean: CourseChildBean
    ) = with(holder.itemView) {
        course_child_title.text = childBean.mName
        course_child_time.text="12:30"
    }
}

class  CourseGroupVH(itemView: View) :
    BaseExpandableRecyclerViewAdapter.BaseGroupViewHolder(itemView) {
    override fun onExpandStatusChanged(
        relatedAdapter: RecyclerView.Adapter<*>?,
        isExpanding: Boolean
    ) {

    }
}

class CourseChildVH(itemView: View) : RecyclerView.ViewHolder(itemView)
```

数据封装：

```kotlin
class CourseGroupBean(
    private val mList: ArrayList<CourseChildBean>,
    val mName: String
) : BaseExpandableRecyclerViewAdapter.BaseGroupBean<CourseChildBean> {

    override fun getChildCount() = mList.size
    override fun isExpandable() = childCount > 0

    override fun getChildAt(childIndex: Int): CourseChildBean? {
        return if (childCount <= 0) null
        else mList[childIndex]
    }
}
```

使用：

```kotlin
private var courseAdapter by Delegates.notNull<CourseListAdapter>()
private val courseList = arrayListOf<CourseGroupBean>()

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    courseAdapter = CourseListAdapter(courseList)
    binding.recycler.adapter = courseAdapter
}

private fun initList() {
    for (i in 0..10) {
        val childList = arrayListOf<CourseChildBean>()
        for (j in 0..i)
            childList.add(CourseChildBean("测试课程测试课程"))
        courseList.add(CourseGroupBean(childList, "测试课程测试课程"))
    }
}
```



