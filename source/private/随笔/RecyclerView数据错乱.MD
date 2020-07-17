数据错乱是因为RecyclerView的刷新机制造成的

解决方案：

1. 暴力解决：禁止RecyclerView的复用

   ```kotlin
   override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int){
   	holder.setIsRecyclable(false)
   }
   ```

2. 重写getItemViewType

   ```kotlin
   override fun getItemViewType(position: Int): Int {
       return position
   }
   ```

   