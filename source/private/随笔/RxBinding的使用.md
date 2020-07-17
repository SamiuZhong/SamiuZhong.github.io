学了RxJava的响应式编程之后，感觉RxBinding也有必要记录一下使用，在项目中使用还是比较多的，只介绍一下使用。
 [RxBinding](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2FJakeWharton%2FRxBinding)

#### 1.依赖



```bash
       compile 'com.jakewharton.rxbinding2:rxbinding:2.0.0'
```

依赖RxBinding就不需要再依赖RxJava了，内部已经包含了

使用RxJava实例化的Disposable需在不用时及时销毁。



```java
public abstract class BaseActivity extends AppCompatActivity {
public CompositeDisposable mCompositeDisposable;
private Unbinder mUnbinder;

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(getLayoutId());
    mUnbinder = ButterKnife.bind(this);
    mCompositeDisposable = new CompositeDisposable();
    onViewCreated(savedInstanceState);
}

/**
 * 添加订阅
 */
public void addDisposable(Disposable mDisposable) {
    if (mCompositeDisposable == null) {
        mCompositeDisposable = new CompositeDisposable();
    }
    mCompositeDisposable.add(mDisposable);
}

/**
 * 取消所有订阅
 */
public void clearDisposable() {
    if (mCompositeDisposable != null) {
        mCompositeDisposable.clear();
    }
}

@Override
protected void onDestroy() {
    super.onDestroy();
    clearDisposable();
    mUnbinder.unbind();
}

protected abstract int getLayoutId();

protected abstract void onViewCreated(Bundle savedInstanceState);
 }
```

##### 2.RxView

使用RxBinding可做到点击防抖的效果，调用这个方法便可防止短时间内对View的重复点击，本例中设置的防抖时间为2s。



```dart
       RxView.clicks(btnLogin)
            .throttleFirst(2, TimeUnit.SECONDS)
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {
                    Toast.makeText(MainActivity.this, "11111111111", Toast.LENGTH_LONG).show();
                }
            });
```

###### throttleFirst(long windowDuration, TimeUnit unit)，设置一定时间内只响应首次(throttleFirst)或者末次(throttleLast)的点击事件。windowDuration为防抖时间，unit为时间单位。

长按监听



```dart
     RxView.longClicks(btnLogin)
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {
                    Toast.makeText(MainActivity.this, "2222222222", Toast.LENGTH_LONG).show();
                }
            });
```

绘制监听



```java
   //当btnLogin绘制时触发
  RxView.draws(btnLogin)
            .subscribe(new Consumer<Object>() {
                @Override
                public void accept(Object o) throws Exception {

                }
            });
```

拖拽监听



```java
    //拖拽监听
    RxView.drags(btnLogin)
            .subscribe(new Consumer<DragEvent>() {
                @Override
                public void accept(DragEvent dragEvent) throws Exception {
                    Toast.makeText(MainActivity.this, "被拖拽了", Toast.LENGTH_LONG).show();
                }
            });
```

滑动时触发



```java
    //滑动时触发
    RxView.scrollChangeEvents(btnLogin)
            .subscribe(new Consumer<ViewScrollChangeEvent>() {
                @Override
                public void accept(ViewScrollChangeEvent viewScrollChangeEvent) throws Exception {

                }
            });
```

EditText输入监听



```java
  // addTextChangedListener
    RxTextView.textChanges(userNameEt).subscribe(new Consumer<CharSequence>() {
        @Override
        public void accept(CharSequence text) throws Exception {
            Toast.makeText(MainActivity.this, "2222222222", Toast.LENGTH_LONG).show();
            clearContent.setVisibility(TextUtils.isEmpty(text) ? View.INVISIBLE : View.VISIBLE);
        }
    });
```

合并监听



```dart
   Observable<CharSequence> userNameObservable = RxTextView.textChanges(userNameEt);
    Observable<CharSequence> userPwdObservable = RxTextView.textChanges(userPasswordEt);
    // 相当于合并
    Observable.combineLatest(userNameObservable, userPwdObservable,
            new BiFunction<CharSequence, CharSequence, Object>() {
                @Override
                public Object apply(CharSequence userName, CharSequence userPwd) throws Exception {
                    Toast.makeText(MainActivity.this, "33333", Toast.LENGTH_LONG).show();
                    // 设置按钮是否可用(或者改变背景颜色)
                    clearContent.setEnabled(!TextUtils.isEmpty(userName) && !TextUtils.isEmpty(userPwd));
                    return null;
                }
            });
```

控件操作时间间隔



```java
   // 控件操作时间间隔
    RxTextView.textChanges(mUserNameEt).debounce(1200, TimeUnit.MILLISECONDS)
            .subscribe(new Consumer<CharSequence>() {
                @Override
                public void accept(CharSequence charSequence) throws Exception {
                    log("debounce");
                }
            });
```

接口轮询，轮询操作



```java
 // 接口轮询，轮询操作
    Observable.interval(2, 2, TimeUnit.SECONDS)
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe(new Consumer<Long>() {
        @Override
        public void accept(Long aLong) throws Exception {
            log("interval");
        }
    });
```

延时操作



```java
    // 延时操作
    Observable.timer(2, TimeUnit.SECONDS)
            .subscribe(new Consumer<Long>() {
                @Override
                public void accept(Long aLong) throws Exception {
                    log("timer");
                }
            });
```

其他



```css
        RxView.visibility(btnClick).accept(true);
        RxView.clickable(btnClick).accept(true);
        RxView.enabled(btnClick).accept(true);
```

#### RxTextView

textChanges

RxTextView.textChanges(TextView view)，内部封装了TextWatcher文本改变监听。



```java
       RxTextView.textChanges(etRxTextView)
              .subscribe(new Consumer<CharSequence>() {
                   @Override
                public void accept(CharSequence charSequence) throws Exception {
                  Log.e("rx_binding_test", "textChanges:文本改变了:" + charSequence.toString());
             }
           });
```

textChangeEvents

RxTextView.textChangeEvents(TextView view)，内部同样封装了TextWatcher文本改变监听。不同的是其返回数据的类型为TextViewTextChangeEvent，内部包含详细的文本改变数据。



```rust
    RxTextView.textChangeEvents(etRxTextView)
    .subscribe(textViewTextChangeEvent -> {
        Log.e("rx_binding_test", "textChanges:文本改变了:" + "before:" + textViewTextChangeEvent.before() + 
            ",start:" + textViewTextChangeEvent.start() + ",text:" + textViewTextChangeEvent.text() + 
            ",count:" + textViewTextChangeEvent.count());
    });
```

editorActions

RxTextView.editorActions(TextView view)，内部封装了OnEditorActionListener软键盘回车点击监听。



```bash
     RxTextView.editorActions(etRxTextView)
      .subscribe(integer -> {
        Log.e("rx_binding_test", "editorActions:输入完毕，点击回车:");
    });
```

editorActionEvents

RxTextView.editorActionEvents(TextView view)，内部同样封装了OnEditorActionListener软键盘回车点击监听。不同的是它的返回类型为TextViewEditorActionEvent，包含actionId，keyEvent等信息



```rust
     RxTextView.editorActionEvents(etRxTextView)
       .subscribe(textViewEditorActionEvent -> {
        KeyEvent keyEvent = textViewEditorActionEvent.keyEvent();
        //判断up状态
        if (keyEvent.getKeyCode() == KeyEvent.KEYCODE_ENTER && keyEvent.getAction() == KeyEvent.ACTION_UP) {
            Log.e("rx_binding_test", "editorActionEvents:输入完毕，点击回车:" + textViewEditorActionEvent.keyEvent());
        }
    });
```

RxTextView中还封装了一些常用的例如setText()、setHint()、setTextColor()等TextView的操作。使用起来也很简单，如下：



```bash
       RxTextView.editorActions(etRxTextView)
     .subscribe(integer -> {
        RxTextView.text(etRxTextView).accept("呵呵");
        RxTextView.hint(etRxTextView).accept("请输入姓名");
        RxTextView.color(etRxTextView).accept(Color.parseColor("#00ff00"));
     });
```

#### 实际场景

在实际项目开发中经常会遇到搜索文本框需根据用户实时输入的文本进行检索，或去数据库查询，或去服务器请求检索。如果文本每改变一次都去检索的话会导致重复大量查数据库或请求服务器，很影响内存。来看下使用RxBinding如何解决这个问题。



```rust
  addDisposable(RxTextView.textChanges(etRxTextView)
    //限流时间500ms
    .debounce(500, TimeUnit.MILLISECONDS)
    .observeOn(AndroidSchedulers.mainThread())
    //CharSequence转换为String
    .map(CharSequence::toString)
    .subscribe(s -> {
        //这里可以查询数据库或请求服务器查询
        Log.e("rx_binding_test", "textChanges:文本改变了:" + s);
    }));
```

正是由于RxBinding使用的是RxJava形式，所以RxJava中的操作符我们都可以使用。利用debounce操作符进行限流，就能避免因为用户输入速度过快导致多次请求服务器了。还可以利用map操作符将CharSequence转换为String再返回。

##### RxCompoundButton

checkedChanges选中状态改变事件

RxCompoundButton.checkedChanges(CompoundButton view)，内部封装了OnCheckedChangeListener选中状态改变监听。



```dart
    //默认注册按钮不可点击
   btnLogin.setEnabled(false);
   RxCompoundButton.checkedChanges(cbContract)
    .subscribe(aBoolean -> {
        RxView.enabled(btnLogin).accept(aBoolean);
        btnLogin.setBackgroundResource(aBoolean ? R.color.colorPrimary : R.color.colorGray);
        RxTextView.color(btnLogin).accept(aBoolean ? Color.parseColor("#ffffff") :
                Color.parseColor("#000000"));
    });
 RxView.clicks(btnLogin)
    //防抖2s
    .throttleFirst(2, TimeUnit.SECONDS)
    .subscribe(o -> Toast.makeText(RxCompoundButtonActivity.this, "注册成功",
            Toast.LENGTH_SHORT).show());
```

默认注册按钮不可点击，当CheckBox被选中后则可点击注册，并修改注册按钮的样式。

View操作

RxCompoundButton中也封装了CompoundButton中例如setchecked()、toggle()等常用的操作，使用方式如下：



```csharp
   RxView.clicks(btnLogin)
    .subscribe(o -> {
        RxCompoundButton.checked(cbContract).accept(true);
        RxCompoundButton.toggle(cbContract).accept(null);
    }));
```



作者：Ayres
链接：https://www.jianshu.com/p/a74843cf1b18
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。