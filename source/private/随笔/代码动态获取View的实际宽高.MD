## 1. onWindowFocusChanged方法

```java
	@Override
    public void onWindowFocusChanged(boolean hasWindowFocus) {
        super.onWindowFocusChanged(hasWindowFocus);
        if(hasWindowFocus){
            int width = view.getMeasuredWidth();
            int height = view.getMeasuredHeight();
        }
    }
```

## 2. View.post(new Runnable())方法

```java
mView.post(new Runnable() {
            @Override
            public void run() {
                int width = mView.getMeasuredWidth();
                int height = mView.getMeasuredHeight();
            }
        });
```

## 3. 通过ViewTreeObserver获取

```java
 ViewTreeObserver observer = mView.getViewTreeObserver();
        observer.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
            @Override
            public void onGlobalLayout() {
                mView.getViewTreeObserver().removeOnGlobalLayoutListener(this);
                int width = mView.getMeasuredWidth();
                int height = mView.getMeasuredHeight();
            }
        });
```

## 4. 手动调用View的measure方法后获取

手动调用measure后，View会调用onMeasure方法对View发起测量，测量完后，就可以获取测量后的宽度和高度了。

但是要对LayoutParams的参数分情况处理才能得到具体的参数值：

__View的LayoutParams参数为match_parent:__

这种情况下无法获取到具体宽高值，因为当View的测量模式为match_parent时，宽高值是取父容器的剩余空间大小作为它自己的宽高。而这时无法获取到父容器的尺寸大小，因此获取会失败。

__View的LayoutParams参数为具体值:__

```java
int width =View.MeasureSpec.makeMeasureSpec(0,View.MeasureSpec.EXACTLY);

int height =View.MeasureSpec.makeMeasureSpec(0,View.MeasureSpec.EXACTLY);

view.measure(width,height);

int height=view.getMeasuredHeight();

int width=view.getMeasuredWidth();
```

__View的LayoutParams参数为wrap_content:__

```java
int width =View.MeasureSpec.makeMeasureSpec((1<<30)-1,View.MeasureSpec.AT_MOST);

int height =View.MeasureSpec.makeMeasureSpec((1<<30)-1,View.MeasureSpec.AT_MOST);

view.measure(width,height);

int height=view.getMeasuredHeight();

int width=view.getMeasuredWidth(); 
```

