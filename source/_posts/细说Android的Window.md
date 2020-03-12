---
title: 细说Android的Window
keywords: 细说Android的Window
date: 2020-03-12 19:31:30
categories: Android
tags:
	- Android
	- 源码
---

## Window是个啥？

Android中的Window是一个窗口的概念，并且它是一个抽象的东西，我们日常开发直接接触到的东西是View，包括Activity、Dialog、Toast等，他们的View都是附着在Window之上的。

所以我们可以说Window是View的直接管理者。

## Window一家人都有哪些成员？

- __Window__

Window是一个抽象类，具体实现类为PhoneWindow，它的职责是对View进行管理。

- __WindowManager__

WindowManager是一个接口，继承自ViewManager接口，它的实现类为WindowManagerImpl，而WindowManagerImpl又会具体的功能实现委托给WindowManagerGlobal。WindowManager的爸爸给它定义了三个方法

```java
public interface ViewManager{
	public void addView(View view, ViewGroup.LayoutParams params);
	public void updateViewLayout(View view, ViewGroup.LayoutParams params);
	public void removeView(View view);
}
```

它的主要职责是对Window进行管理，怎么管理？就是它爸爸给它的三个方法：添加、更新和删除。

- __WindowManagerService__

WindowManagerService是Android Framework层的一个重要服务，WindowManager是受它管理的，对WMS的探究会较为深入，本篇文章不会过多讲解。

一张图概括一下Window和WindowManager的关系：

![img1](https://upload-images.jianshu.io/upload_images/15143432-5e5d6b28a1a93324.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Window的属性

Window的属性有不少，但日常开发经常用到的大概有以下三种

- Type：Window的类型
- Flag：Window的标志
- SoftInputMode：软键盘相关模式

### Type

Window有很多种类型，比如应用程序窗口、输入法窗口、PopupWindow、Toast、Dialog等，眼睛都看花了。索性我们对这么多奇奇怪怪的Window统一分成三个大类：

- Application Window(应用程序窗口)
- Sub Window(子窗口)
- System Window(系统窗口)

举例说明一下，Activity就是一个典型的应用Window；PopupWindow就是一个子Window，子Window不能单独存在，必须附着在其他Window上才可以；系统Window包括Toast、输入法窗口、系统音量条等

这三种Window在显示上面有层级之分，具体表现在Type属性的值不同。应用Window的Type值范围为[1,99]，子Window的Type值范围[1000,1999]，系统Window的Type值范围[2000,2999]。Type值越大的Window会显示在越上层，这也就能解释为什么Dialog会显示在Activity上层，而Toast又会显示在Dialog上层了。

### Flag

Flag标志用于控制Window的显示，一共有数十种标志，这里列举几个常用的：

|         Flag          |             描述             |
| :-------------------: | :--------------------------: |
|  FLAG_NOT_FOCUSABLE   |   表示Window不需要获取焦点   |
|  FLAG_NOT_TOUCHABLE   | 表示Window不接收任何触摸事件 |
| FLAG_SHOW_WHEN_LOCKED |   Window可以显示在锁屏界面   |
|    FLAG_FULLSCREEN    |     允许窗口超过屏幕之外     |

设置Window的Flag有三种方式，第一种是Window的addFlags方法

```java
Window mWindow = getWindow();
mWindow.addFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

第二种是Window的setFlags方法，其实addFlags内部也会调用setFlags方法

```java
Window mWindow = getWindow();
mWindow.setFlags(WindowManager.LayoutParams.FLAG_FULLSCREEN,WindowManager.LayoutParams.FLAG_FULLSCREEN);
```

第三种方法是给LayoutParams设置Flag，并通过WindowManager的addView方法添加

```java
WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
mParams.flags = WindowManager.LayoutParams.FLAG_FULLSCREEN;
WindowManager mManager = (WindowManager)getSystemService(Context.WINDOW_SERVICE);
TextView mText = new TextView(this);
mManager.addView(mText,mParams);
```

### SoftInputMode

SoftInputMode和Window的Flags长得很像，它也有很多种，我们同样列举几个常用的

|              Flag              |            描述            |
| :----------------------------: | :------------------------: |
|     SOFT_INPUT_ADJUST_PAN      | 弹出软键盘时窗口不调整大小 |
|    SOFT_INPUT_ADJUST_RESIZE    | 弹出软键盘时窗口会调整大小 |
|    SOFT_INPUT_STATE_HIDDEN     |       软键盘默认隐藏       |
| SOFT_INPUT_STATE_ALWAYS_HIDDEN |       软键盘总是隐藏       |

当我们为Activity设置SoftInputMode时可以直接在Manifest文件中指定

```xml
android:windowSoftInputMode="adjustPan"
```

也可在代码中设定

```java
getWindow().setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_ADJUST_PAN);
```

## Window的操作

前面我们说过，Window的操作实质上就是对View的操作，而Window的操作是通过WindowManager来进行的，WindowManager可以进行的操作总共就添加、更新和删除这三个。

我们知道，WindowManager的这三大方法是从它老爹ViewManager继承过来的，而它本身又是一个接口，所以这三个方法会交由它的实现类WindowManagerImpl。而WindowManagerImpl也是一个不成器的儿子，你以为它自己就把这三个功能实现了？错了，它又把这三个功能委托给了WindowManagerGlobal来实现。

那我们现在就来看看这三个功能究竟是怎么整出来的。

### Window的添加过程

1. 我们调用WindowManager的addView方法，给它一个View喊它添加。

2. WindowManager拿到这个View丢给它儿子WindowManagerImpl。

3. WindowManagerImpl拿到老爹丢给它的这个View，反手丢给了小弟WindowManagerGlobal，喊它去整。

4. WindowManagerGlobal拿到这个View，发现没有小弟可用了，那咋办？只有自己整呗。先整几个列表出来存放东西

   ```JAVA
   private final ArrayList<View> mViews = new ArrayList<View>();
   private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();
   private final ArrayList<WindowManager.LayoutParams> mParams = 
           new ArrayList<WindowManager.LayoutParams>();
   ```

   这里mViews用来存放所有Window对应的View；mRoots用来存放所有Window对应的ViewRootImpl；mParams用来存放所有Window对应的布局参数。

   好，现在要添加一个View，那就也整一个addView方法吧

   ```java
   public void addView(View view, ViewGroup.LayoutParams params,
                       Display display, Window parentWindow) {
       ...
       final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
       if (parentWindow != null) {		//comment 1
           parentWindow.adjustLayoutParamsForSubWindow(wparams);
       } else {
           final Context context = view.getContext();
           if (context != null
                   && (context.getApplicationInfo().flags
                   & ApplicationInfo.FLAG_HARDWARE_ACCELERATED) != 0) {
               wparams.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
           }
       }
       ViewRootImpl root;
       View panelParentView = null;
       synchronized (mLock) {
           ...
           root = new ViewRootImpl(view.getContext(), display);	//comment 2
           view.setLayoutParams(wparams);
           mViews.add(view);	//comment 3
           mRoots.add(root);	//comment 4
           mParams.add(wparams);	//comment 5
           try {
               root.setView(view, wparams, panelParentView);	//comment 6
           } catch (RuntimeException e) {
               if (index >= 0) {
                   removeViewLocked(index, true);
               }
               throw e;
           }
       }
   }
   ```

   这里可以看出来，

   先在comment 1处检查当前Window是否为子Window，如果是的话就会把LayoutParams交给父Window来进行相应的布局调整。

   在comment 2处new一个ViewRootImpl对象出来。

   然后在comment 3到5分别把view、root和params放到上面对应的List中去。

   最后在comment 6处将Window和参数通过setView方法设置到ViewRootImpl中去。

5. 最后通过ViewRootImpl将任务提交给WMS来处理，这里面的具体实现是一个IPC的过程，我们这里不去深究。

### Window的更新过程

同样的，WindowManagerGlobal再整一个updateViewLayout方法

```java
public void updateViewLayout(View view, ViewGroup.LayoutParams params) {
    ...
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams)params;
    view.setLayoutParams(wparams);	//comment 1
    synchronized (mLock) {
        int index = findViewLocked(view, true);	//comment 2
        ViewRootImpl root = mRoots.get(index);	//comment 3
        mParams.remove(index);	//comment 4
        mParams.add(index, wparams);	//comment 5
        root.setLayoutParams(wparams, false);	//comment 6
    }
}
```

1. 在comment 1处将params更新到View中。
2. 在comment 2处拿到我们要更新的这个view在上面的mViews列表中的索引。
3. 我们在comment 3处根据这个索引从mRoots列表中找到这个view的对应的ViewRootImpl。
4. 在comment 4到5处将mParams列表中存放的对应的params进行替换
5. 最后在comment 6为对应的ViewRootImpl更新参数，ViewRootImpl在拿到更新的参数之后就开始重新对View进行绘制的流程了。

### View的删除过程

同理，WindowManagerGlobal整一个removeView方法

```java
public void removeView(View view, boolean immediate) {
	...
    synchronized (mLock) {
        int index = findViewLocked(view, true);
        View curView = mRoots.get(index).getView();
        removeViewLocked(index, immediate);
        if (curView == view) {
            return;
        }

        throw new IllegalStateException("Calling with view " + view
                + " but the ViewAncestor is attached to " + curView);
    }
}
```

删除的过程较为简单，通过索引在mViews列表中找到对应的View删除即可。

## Window的创建过程

上面我们分析了Window对View的几个操作，但是View本身不能单独存在，它必须依附于Window，也就是说有View的地方就必然有Window。那么，Window本身又是怎么创建的呢？我们用两个例子来进一步一探究竟。

### Activity的Window创建过程

首先，activity会在attach()方法里面创建一个phoneWindow

```java
final void attach(Context context, ActivityThread aThread,
                  Instrumentation instr, IBinder token, int ident,
                  Application application, Intent intent, ActivityInfo info,
                  CharSequence title, Activity parent, String id,
                  NonConfigurationInstances lastNonConfigurationInstances,
                  Configuration config, String referrer, IVoiceInteractor voiceInteractor,
                  Window window, ActivityConfigCallback activityConfigCallback) {
    ...
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    mWindow.setWindowControllerCallback(this);
    mWindow.setCallback(this);	//comment 1
    mWindow.setOnWindowDismissedCallback(this);
    mWindow.getLayoutInflater().setPrivateFactory(this);
    if (info.softInputMode != WindowManager.LayoutParams.SOFT_INPUT_STATE_UNSPECIFIED) {
        mWindow.setSoftInputMode(info.softInputMode);
    }
    if (info.uiOptions != 0) {
        mWindow.setUiOptions(info.uiOptions);
    }
    ...
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    mWindowManager = mWindow.getWindowManager();
    mCurrentConfig = config;
    mWindow.setColorMode(info.colorMode);
	...
}
```

这里的comment 1处我们可以看到activity是实现了前面phoneWindow里面的那个回调。

然后，如你所知，我们为activity设置视图是通过setContentView方法来进行的

```java
public void setContentView(@LayoutRes int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```

可以看到，这里直接把任务提交给了Window来处理。而我们又知道Window的实现类是phoneWindow，所以我们来看看phoneWindow里面的逻辑

```java
public void setContentView(int layoutResID) {
    if (mContentParent == null) {	//comment 1
        installDecor();
    } else if (!hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        mContentParent.removeAllViews();
    }

    if (hasFeature(FEATURE_CONTENT_TRANSITIONS)) {
        final Scene newScene = Scene.getSceneForLayout(mContentParent, layoutResID,
                getContext());
        transitionTo(newScene);
    } else {
        mLayoutInflater.inflate(layoutResID, mContentParent);	//comment 2
    }
    mContentParent.requestApplyInsets();
    final Callback cb = getCallback();	//comment 3
    if (cb != null && !isDestroyed()) {
        cb.onContentChanged();	//comment 4
    }
    mContentParentExplicitlySet = true;
}
```

这里需要补充说明一下，activity中最顶级的View是DecorView,内部包含一个标题栏和内容栏。

好，在上面的setContentView方法中：

1. 在comment 1处先检查DecorView是否存在，不存在的话就创建它。
2. 经过一系列的判断，在comment 2处将资源文件变成View添加到DecorView的ContentView里面去。
3. 在comment 3处拿到一个回调，这个回调是在回调个啥玩意儿？它主要是用来给activity实现的。（关于回调的知识可以看鄙人的另一篇文章[今天，我们细说回调](https://juejin.im/post/5e4cdaa3e51d4526e807e96e)）
4. 最后，我们通过刚才拿到的那个回调通知activity这个视图已经发生改变了。

经过一顿操作之后，我们的DecorView已经创建完毕，activity的布局文件也添加进去了，但是这个时候还没有将它添加到Window当中去。

最后我们需要把DecorView添加到activity的这个Window当中去，这个操作的实现是在makeVisible()方法

```java
void makeVisible() {
    if (!mWindowAdded) {
        ViewManager wm = getWindowManager();
        wm.addView(mDecor, getWindow().getAttributes());
        mWindowAdded = true;
    }
    mDecor.setVisibility(View.VISIBLE);
}
```

这样一个activity的Window就创建出来了。

### Dialog的Window创建过程

我们先来看Dialog的一个构造方法

```java
Dialog(@NonNull Context context, @StyleRes int themeResId, boolean createContextThemeWrapper) {
    ...
    mWindowManager = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);
    final Window w = new PhoneWindow(mContext);
    mWindow = w;
    w.setCallback(this);
    w.setOnWindowDismissedCallback(this);
    w.setOnWindowSwipeDismissedCallback(() -> {
        if (mCancelable) {
            cancel();
        }
    });
    w.setWindowManager(mWindowManager, null, null);
    w.setGravity(Gravity.CENTER);
    mListenersHandler = new ListenersHandler(this);
}
```

我们可以看到，Dialog是在这个构造方法里面new了一个phoneWindow出来。

与activity类似，我们需要把视图文件添加到DecorView中去，这一步也是在setContentView里面实现的，就不细说了。

然后就是显示和关闭了。

当我们调用show()方法的时候windowManager就会通过addView()方法将DecoreView添加到Window里面去；而当我们调用dismiss()方法的时候，它会调用dismissDialog()方法，在这个方法内部会通过windowManager将DecorView删除掉。

## 参考资料

__《Android开发艺术探究》__

__《Android进阶解密》__

