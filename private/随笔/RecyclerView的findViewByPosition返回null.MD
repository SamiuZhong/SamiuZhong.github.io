[原文](https://blog.csdn.net/sinat_24946363/article/details/80642824?utm_source=blogxgwz4)

## 原因

因为在Activity的onCreate()方法中，View还没有开始绘制，只是做了初始化工作，所以获取不到childview，在Activity的onResume()方法中才真正开始了View的绘制

## 解决办法

重写addOnGlobalLayoutListener()接口，在View绘制完成之后再获取childview

```java
//View加载完成时回调
recyclerView.getViewTreeObserver().addOnGlobalLayoutListener(new ViewTreeObserver
        .OnGlobalLayoutListener() {
    @Override
    public void onGlobalLayout() {
        View view = layoutManager.findViewByPosition(3);
        Log.i(TAG, "onCreate: view = " + view.toString());

        //OnGlobalLayoutListener可能会被多次触发
        //所以完成了需求后需要移除OnGlobalLayoutListener
        recyclerView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
    }
});
```

## 补充

```java
// view重绘时回调  
view.getViewTreeObserver().addOnDrawListener(new OnDrawListener() {  
    @Override  
    public void onDraw() {  
        // TODO Auto-generated method stub  
    }  
});  
```

