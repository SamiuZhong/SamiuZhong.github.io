---
title: RecyclerView的ListAdapter喝DiffUtil
keywords: RecyclerView的ListAdapter喝DiffUtil
date: 2020-04-15 19:31:30
categories: Android
tags:
	- Android
---

## 这玩意儿是个啥？

所谓ListAdapter就是我们通常使用的RecyclerView，官方对它的Adapter又做了一次封装，同时配合DiffUtil这个类来使用。

那么为啥要做这个操作喃？是因为传统的Adapter写法，一个缺点就是代码冗余，不够优雅；另一个缺点就是我们刷新数据的时候必须要手动调用notifyDataSetChanged方法才行，这个方法不光是调用麻烦，而且它每次刷新都是全部item都刷一遍，那我只修改了一个item也要全部刷新，性能就很低。所以官方就又出了个DiffUtil来解决这个问题。

那下面我们就一一道来。

## DiffUtil

对于DiffUtil，我们其实只需要关注这个类

- DiffUtil.ItemCallback：具体用于限定数据集比对规则

### DiffUtil.ItemCallback

我们先来看看它长什么样子

```kotlin
public abstract static class ItemCallback<T> {

      public abstract boolean areItemsTheSame(@NonNull T oldItem, @NonNull T newItem);

      public abstract boolean areContentsTheSame(@NonNull T oldItem, @NonNull T newItem);

      @Nullable
      @SuppressWarnings({"WeakerAccess", "unused"})
      public Object getChangePayload(@NonNull T oldItem, @NonNull T newItem) {
          return null;
      }

}
```

我们就简单说说这两个抽象方法是拿来干嘛的

- areItemsTheSame(T oldItem, T newItem)这个方法用来比较传入的两个Item是否是同一个Item，是返回true，否则返回false
- areContentsTheSame(T oldItem, T newItem)如果上面返回true了，好然后我们调用这个方法来判断这相同的Item是不是里面的内容也相同，相同返回true，否则返回false

## ListAdapter

好，然后我们现在来说ListAdapter

首先看看这玩意儿长啥样子

```kotlin
public abstract class ListAdapter<T, VH extends RecyclerView.ViewHolder>
        extends RecyclerView.Adapter<VH> {

    final AsyncListDiffer<T> mDiffer;

    private final AsyncListDiffer.ListListener<T> mListener =
            new AsyncListDiffer.ListListener<T>() {
        @Override
        public void onCurrentListChanged(
                @NonNull List<T> previousList, @NonNull List<T> currentList) {
            ListAdapter.this.onCurrentListChanged(previousList, currentList);
        }
    };

    @SuppressWarnings("unused")
    protected ListAdapter(@NonNull DiffUtil.ItemCallback<T> diffCallback) {
        mDiffer = new AsyncListDiffer<>(new AdapterListUpdateCallback(this),
                new AsyncDifferConfig.Builder<>(diffCallback).build());
        mDiffer.addListListener(mListener);
    }

    @SuppressWarnings("unused")
    protected ListAdapter(@NonNull AsyncDifferConfig<T> config) {
        mDiffer = new AsyncListDiffer<>(new AdapterListUpdateCallback(this), config);
        mDiffer.addListListener(mListener);
    }

    public void submitList(@Nullable List<T> list) {
        mDiffer.submitList(list);
    }

    public void submitList(@Nullable List<T> list, @Nullable final Runnable commitCallback) {
        mDiffer.submitList(list, commitCallback);
    }

    protected T getItem(int position) {
        return mDiffer.getCurrentList().get(position);
    }

    @Override
    public int getItemCount() {
        return mDiffer.getCurrentList().size();
    }

    @NonNull
    public List<T> getCurrentList() {
        return mDiffer.getCurrentList();
    }

    public void onCurrentListChanged(@NonNull List<T> previousList, @NonNull List<T> currentList) {}
}
```

看起来有点复杂，不过没关系，我们只需要关注这几个点
1. 这个东西需要传入两个泛型，就是我们的Item和ViewHolder
2. 一般我们使用这一个构造方法ListAdapter(DiffUtil.ItemCallback<T> diffCallback)，传入我们自己重写的diffCallback
3. 最后我们调用submitList(List<T> list)这个方法来刷新数据，需要注意的是这里传过来的列表必须是一个新的才会刷新

好了，现在基本上没啥问题了，我们来写一个Adapter试试看

## For Example

那就先给封装一个Item的类,注意我们DiffUtil.ItemCallback基本上也写在这个类里面
```kotlin
sealed class NavigationModelItem {

    data class NavMenuItem(
        val id: Int,
        @DrawableRes val icon: Int,
        @StringRes val titleRes: Int,
        var checked: Boolean
    ) : NavigationModelItem()

    /**
     * item局部刷新
     */
    object NavModelItemDiff : DiffUtil.ItemCallback<NavigationModelItem>() {

        override fun areItemsTheSame(
            oldItem: NavigationModelItem,
            newItem: NavigationModelItem
        ): Boolean {
            return when {
                oldItem is NavMenuItem && newItem is NavMenuItem ->
                    oldItem.id == newItem.id
                else -> oldItem == newItem
            }
        }

        override fun areContentsTheSame(
            oldItem: NavigationModelItem,
            newItem: NavigationModelItem
        ): Boolean {
            return when {
                oldItem is NavMenuItem && newItem is NavMenuItem ->
                    oldItem.icon == newItem.icon &&
                            oldItem.titleRes == newItem.titleRes &&
                            oldItem.checked == newItem.checked
                else -> false
            }
        }
    }
}
```

然后封装一个Holder
```kotlin
sealed class NavigationViewHolder<T : NavigationModelItem>(
    view: View
) : RecyclerView.ViewHolder(view) {

    abstract fun bind(navItem:T)

    class NavMenuItemViewHolder(
        private val binding:NavMenuItemLayoutBinding,
    ):NavigationViewHolder<NavigationModelItem.NavMenuItem>(binding.root){

        override fun bind(navItem: NavigationModelItem.NavMenuItem) {
            binding.run {
                navMenuItem = navItem
                executePendingBindings()
            }
        }
    }
}
```

最后是Adapter
```kotlin
private const val VIEW_TYPE_NAV_MENU_ITEM = 4

@Suppress("UNCHECKED_CAST")
class NavigationAdapter() : ListAdapter<NavigationModelItem, NavigationViewHolder<NavigationModelItem>>(
    NavigationModelItem.NavModelItemDiff
) {

    interface NavigationAdapterListener {
        fun onNavMenuItemClicked(item: NavigationModelItem.NavMenuItem)
    }

    override fun getItemViewType(position: Int): Int {
        return when (getItem(position)) {
            is NavigationModelItem.NavMenuItem -> VIEW_TYPE_NAV_MENU_ITEM
            else -> throw RuntimeException("Unsupported ItemViewType for obj ${getItem(position)}")
        }
    }

    override fun onCreateViewHolder(
        parent: ViewGroup,
        viewType: Int
    ): NavigationViewHolder<NavigationModelItem> {
        return when (viewType) {
            VIEW_TYPE_NAV_MENU_ITEM -> NavigationViewHolder.NavMenuItemViewHolder(
                NavMenuItemLayoutBinding.inflate(
                    LayoutInflater.from(parent.context),
                    parent,
                    false
                ),
            )
            else -> throw RuntimeException("Unsupported view holder type")
        } as NavigationViewHolder<NavigationModelItem>
    }

    override fun onBindViewHolder(
        holder: NavigationViewHolder<NavigationModelItem>,
        position: Int
    ) {
        holder.bind(getItem(position))
    }
}
```
最后就是使用这个Adapter
```kotlin
    //给recyclerView设置adapter
    val adapter = NavigationAdapter()
    navRecyclerView.adapter = adapter
    //liveDate订阅数据的变化
    NavigationModel.navigationList.observe(viewLifecycleOwner) {
        adapter.submitList(it)
    }
```
