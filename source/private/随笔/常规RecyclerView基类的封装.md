## 单一布局的BaseAdapter

```java
public abstract class BaseSingleRecyclerAdapter<T> extends RecyclerView.Adapter {
    public Context context;
    public LayoutInflater layoutInflater;
    public List<T> mList = new ArrayList<>();

    public BaseSingleRecyclerAdapter(Context context) {
        this.context = context;
        this.layoutInflater = LayoutInflater.from(context);
    }

    /**
     * 获取Item的数量
     *
     * @return
     */
    @Override
    public int getItemCount() {
        return mList == null ? 0 : mList.size();
    }

    /**
     * 添加一个Item
     *
     * @param item
     */
    public void addItem(T item) {
        mList.add(item);
        notifyDataSetChanged();
    }

    /**
     * 添加一个Item列表
     *
     * @param list
     */
    public void addAll(List<T> list) {
        mList.addAll(list);
        notifyDataSetChanged();
    }

    /**
     * 替换Item列表
     *
     * @param list
     */
    public void replaceAll(List<T> list) {
        mList.clear();
        addAll(list);
    }

    /**
     * 清空Item列表
     */
    public void clearAll() {
        mList.clear();
        notifyDataSetChanged();
    }

    /**
     * 删除指定位置的Item
     *
     * @param index
     */
    public void remove(int index) {
        mList.remove(index);
        notifyDataSetChanged();
    }

    /**
     * 获取指定位置的Item
     *
     * @param index
     * @return
     */
    public T getItemByPosition(int index) {
        return mList.get(index);
    }
}
```

## 多布局的BaseAdapter

首先是BaseItem的封装

```java
public class BaseMultiRecyclerItem {

    private Object data;
    private int type;

    public BaseMultiRecyclerItem() {
    }

    public BaseMultiRecyclerItem(Object data, int type) {
        this.data = data;
        this.type = type;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }

    public int getType() {
        return type;
    }

    public void setType(int type) {
        this.type = type;
    }
}
```

然后是Adapter

```java
public abstract class BaseMultiRecyclerAdapter extends RecyclerView.Adapter<RecyclerView.ViewHolder> {

    public Context context;
    public LayoutInflater layoutInflater;
    public List<BaseMultiRecyclerItem> mList = new ArrayList<>();

    public BaseMultiRecyclerAdapter(Context context) {
        this.context = context;
        this.layoutInflater = LayoutInflater.from(context);
    }

    /**
     * 设定Item的Type
     *
     * @param position
     * @return
     */
    @Override
    public int getItemViewType(int position) {
        return mList.get(position).getType();
    }

    /**
     * 获取Item的数量
     *
     * @return
     */
    @Override
    public int getItemCount() {
        return mList.size();
    }

    /**
     * 添加一个Item
     *
     * @param item
     */
    public void addItem(BaseMultiRecyclerItem item) {
        mList.add(item);
        notifyDataSetChanged();
    }

    /**
     * 添加一个Item列表
     *
     * @param list
     */
    public void addAll(List<BaseMultiRecyclerItem> list) {
        mList.addAll(list);
        notifyDataSetChanged();
    }

    /**
     * 替换Item列表
     *
     * @param list
     */
    public void replaceAll(List<BaseMultiRecyclerItem> list) {
        mList.clear();
        addAll(list);
    }

    /**
     * 清空Item列表
     */
    public void clearAll() {
        mList.clear();
        notifyDataSetChanged();
    }

    /**
     * 删除指定位置的Item
     *
     * @param index
     */
    public void remove(int index) {
        mList.remove(index);
        notifyDataSetChanged();
    }

    /**
     * 删除指定Type的所有Item
     *
     * @param item Type就是指这个item的Type
     */
    public void removeType(BaseMultiRecyclerItem item) {
        for (int i = 0; i < mList.size(); i++) {
            if (item.getType() == mList.get(i).getType()) {
                mList.remove(i);
                i--;
            }
        }
        notifyDataSetChanged();
    }
}
```

## 使用

继承实现即可

```java
public class HealthClassifyAdapter extends BaseSingleRecyclerAdapter<String> {
    public HealthClassifyAdapter(Context context) {
        super(context);
    }

    @NonNull
    @Override
    public RecyclerView.ViewHolder onCreateViewHolder(@NonNull ViewGroup parent, int viewType) {
        return new HealthClassifyHolder(layoutInflater.inflate(R.layout.item_health_classify, parent, false));
    }

    @Override
    public void onBindViewHolder(@NonNull RecyclerView.ViewHolder holder, int position) {
        if (holder instanceof HealthClassifyHolder) {
            GlideUtils.loadRoundedImage(((HealthClassifyHolder) holder).img, Constants.TEST_IMAGE_URL);
            ((HealthClassifyHolder) holder).name.setText("周星星");
            ((HealthClassifyHolder) holder).title.setText(mList.get(position));
//            holder.itemView.setOnClickListener(v -> context.startActivity(new Intent(context, ArticleDetailActivity.class)));
            holder.itemView.setOnClickListener(v -> context.startActivity(new Intent(context, ArticleIntroduceActivity.class)));
        }
    }

    private static class HealthClassifyHolder extends RecyclerView.ViewHolder {
        private ImageView img;
        private TextView name;
        private TextView title;

        public HealthClassifyHolder(@NonNull View itemView) {
            super(itemView);
            img = itemView.findViewById(R.id.health_classify_img);
            name = itemView.findViewById(R.id.health_classify_name);
            title = itemView.findViewById(R.id.health_classify_title);
        }
    }
}
```



