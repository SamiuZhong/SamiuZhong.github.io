## 1. 腾讯X5内核依赖

```xml
api "com.tencent.tbs.tbssdk:sdk:43697"
```

## 2. X5WebView

```java
public class X5WebView extends WebView {

    @SuppressLint("SetJavaScriptEnabled")
    public X5WebView(Context arg0, AttributeSet arg1) {
        super(arg0, arg1);
        /**
         * 防止加载网页时调起系统浏览器
         */
        WebViewClient client = new WebViewClient() {
            /**
             * 防止加载网页时调起系统浏览器
             */
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }
        };
        this.setWebViewClient(client);
        initWebViewSettings();
        this.getView().setClickable(true);
    }

    @SuppressLint("SetJavaScriptEnabled")
    private void initWebViewSettings() {
        WebSettings webSetting = this.getSettings();
        webSetting.setJavaScriptEnabled(true);
        webSetting.setJavaScriptCanOpenWindowsAutomatically(true);
        webSetting.setAllowFileAccess(true);
        webSetting.setLayoutAlgorithm(WebSettings.LayoutAlgorithm.NARROW_COLUMNS);
        webSetting.setSupportZoom(true);
        webSetting.setBuiltInZoomControls(true);
        webSetting.setUseWideViewPort(true);
        webSetting.setSupportMultipleWindows(true);
        webSetting.setLoadWithOverviewMode(true);
        webSetting.setAppCacheEnabled(true);
        webSetting.setDomStorageEnabled(true);
        webSetting.setGeolocationEnabled(true);
        webSetting.setAppCacheMaxSize(Long.MAX_VALUE);
        webSetting.setPluginState(WebSettings.PluginState.ON_DEMAND);
        webSetting.setCacheMode(WebSettings.LOAD_NO_CACHE);
    }

    public X5WebView(Context arg0) {
        super(arg0);
        setBackgroundColor(85621);
    }
}
```

## 3. 浏览器页面BrowserActivity

```kotlin
class BrowserActivity : BaseActivity() {

    companion object {
        const val URL = "url"
    }

    override fun getLayoutResId() = R.layout.activity_browser

    override fun initView() {
        toolBar.title = getString(R.string.is_loading)
        toolBar.navigationIcon = getDrawable(R.drawable.arrow_back)
        initWebView()
    }

    override fun initData() {
        toolBar.setNavigationOnClickListener { onBackPressed() }
        intent.getStringExtra(URL)?.let {
            webView.loadUrl(it)
        }
    }

    private fun initWebView() {
        progressBar.progressDrawable = getDrawable(R.drawable.color_progressbar)
        webView.run {
            webViewClient = object : WebViewClient() {

                override fun onPageStarted(p0: WebView?, p1: String?, p2: Bitmap?) {
                    super.onPageStarted(p0, p1, p2)
                    progressBar.visibility = View.VISIBLE
                }

                override fun onPageFinished(p0: WebView?, p1: String?) {
                    super.onPageFinished(p0, p1)
                    progressBar.visibility = View.GONE
                }
            }
            webChromeClient = object : WebChromeClient() {
                override fun onProgressChanged(p0: WebView?, p1: Int) {
                    super.onProgressChanged(p0, p1)
                    progressBar.progress = p1
                    Log.e("browser", p1.toString())
                }

                override fun onReceivedTitle(p0: WebView?, p1: String?) {
                    super.onReceivedTitle(p0, p1)
                    p1?.let { toolBar.title = p1 }
                }

            }
        }
    }

    override fun onBackPressed() {
        if (webView.canGoBack()) webView.goBack()
        super.onBackPressed()
    }
}
```

## 4. 布局activity_browser

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <androidx.appcompat.widget.Toolbar
        android:id="@+id/toolBar"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:background="@color/color_3066BE"
        android:theme="@style/ThemeOverlay.AppCompat.ActionBar"
        app:popupTheme="@style/ThemeOverlay.AppCompat.Light"
        app:titleTextColor="@color/white" />

    <ProgressBar
        android:id="@+id/progressBar"
        style="@style/Widget.AppCompat.ProgressBar.Horizontal"
        android:layout_width="match_parent"
        android:layout_height="3dp" />

    <com.samiu.base.view.X5WebView
        android:id="@+id/webView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />
</LinearLayout>
```

## 5. 进度条加载样式(drawable文件夹下)

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android" >

    <!-- 设置背景色（白色） -->
    <item android:id="@android:id/background">
    <shape>   
        <corners android:radius="3dip" />   
        <gradient android:startColor="#FFFFFF"   
                android:centerColor="#FFFFFF"   
                android:centerY="0.75"   
                android:endColor="#FFFFFF"   
                android:angle="270"   
        />   
    </shape> 
    </item>

    <!-- 设置进度条颜色（青色） -->
    <item android:id="@android:id/progress">
        <clip>
        <shape>   
            <corners android:radius="3dip" />   
            <gradient android:startColor="#80DED9"
                android:endColor="#80DED9"   />
        </shape> 
        </clip>
    </item>

</layer-list>
```

## 6. 使用

直接跳转activity就行了，传一个包含url地址的intent

```kotlin
fun toBrowser(url: String) {
        startActivity(Intent(this, BrowserActivity::class.java).apply {
            putExtra(URL, url)
        })
    }
```