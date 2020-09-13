# 一些第三方依赖

- 屏幕适配 [Autosize](https://github.com/JessYanCoding/AndroidAutoSize)
- 广告轮播 [Banner](https://github.com/youth5201314/banner)
- Kotlin注入框架 [Koin](https://github.com/InsertKoinIO/koin)
- 图片加载 [Glide](https://github.com/bumptech/glide)
- 网络请求 [OkHttp](https://github.com/square/okhttp)、[Retrofit](https://github.com/square/retrofit)
- 下拉框 [SmartRefreshLayout](https://github.com/scwang90/SmartRefreshLayout)
- 事件总线 [LiveEventBus](https://github.com/JeremyLiao/LiveEventBus)
- 流式布局  [FlowLayout](https://github.com/nex3z/FlowLayout)
- Dialog  [Material Dialog](https://github.com/afollestad/material-dialogs)、[LicensesDialog](https://github.com/PSDev/LicensesDialog)

# 基类

## 1.UI

**BaseActivity**

```kotlin
abstract class BaseActivity :AppCompatActivity(),CoroutineScope by MainScope(){

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(getLayoutResId())
        initView()
        initData()
    }

    abstract fun getLayoutResId(): Int
    abstract fun initView()
    abstract fun initData()

    override fun onDestroy() {
        super.onDestroy()
        cancel()
    }
}
```

__BaseFragment__

```kotlin
abstract class BaseFragment:Fragment(),CoroutineScope by MainScope() {

    override fun onCreateView(inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?): View? {
        return inflater.inflate(getLayoutResId(), container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        initView()
        initData()
        super.onViewCreated(view, savedInstanceState)
    }

    abstract fun getLayoutResId(): Int
    abstract fun initView()
    abstract fun initData()

    override fun onDestroy() {
        super.onDestroy()
        cancel()
    }
}
```

__BaseVMActivity__

```kotlin
abstract class BaseVMActivity<VM : BaseViewModel> : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        startObserve()
        setContentView(getLayoutResId())
        initData()
    }

    open fun getLayoutResId(): Int = 0
    abstract fun initView()
    abstract fun initData()
    abstract fun startObserve()
}
```

__BaseVMFragment__

```kotlin
abstract class BaseVMFragment<VM: BaseViewModel> :Fragment(){

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(getLayoutResId(),container,false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        initView()
        initData()
        startObserve()
        super.onViewCreated(view, savedInstanceState)
    }

    abstract fun getLayoutResId(): Int
    abstract fun initView()
    abstract fun initData()
    abstract fun startObserve()
}
```

__BaseViewModel__

```kotlin
open class BaseViewModel : ViewModel() {
    open class BaseUiModel<T>(
        var showLoading: Boolean = false,
        var showError: String? = null,
        var showSuccess: T? = null,
        var showEnd: Boolean = false, // 加载更多
        var isRefresh: Boolean = false // 刷新

    )

    private val mException: MutableLiveData<Throwable> = MutableLiveData()

    private fun launchOnUI(block: suspend CoroutineScope.() -> Unit) {
        viewModelScope.launch { block() }
    }

    suspend fun <T> launchOnIO(block: suspend CoroutineScope.() -> T) {
        withContext(Dispatchers.IO) {
            block
        }
    }

    fun launch(tryBlock: suspend CoroutineScope.() -> Unit) {
        launchOnUI {
            tryCatch(tryBlock, {}, {}, true)
        }
    }


    fun launchOnUITryCatch(
        tryBlock: suspend CoroutineScope.() -> Unit,
        catchBlock: suspend CoroutineScope.(Throwable) -> Unit,
        finallyBlock: suspend CoroutineScope.() -> Unit,
        handleCancellationExceptionManually: Boolean
    ) {
        launchOnUI {
            tryCatch(tryBlock, catchBlock, finallyBlock, handleCancellationExceptionManually)
        }
    }

    fun launchOnUITryCatch(
        tryBlock: suspend CoroutineScope.() -> Unit,
        handleCancellationExceptionManually: Boolean = false
    ) {
        launchOnUI {
            tryCatch(tryBlock, {}, {}, handleCancellationExceptionManually)
        }
    }


    private suspend fun tryCatch(
        tryBlock: suspend CoroutineScope.() -> Unit,
        catchBlock: suspend CoroutineScope.(Throwable) -> Unit,
        finallyBlock: suspend CoroutineScope.() -> Unit,
        handleCancellationExceptionManually: Boolean = false
    ) {
        coroutineScope {
            try {
                tryBlock()
            } catch (e: Throwable) {
                if (e !is CancellationException || handleCancellationExceptionManually) {
                    mException.value = e
                    catchBlock(e)
                } else {
                    throw e
                }
            } finally {
                finallyBlock()
            }
        }
    }
}
```

## 2. 网络

__NetworkUtil__

```kotlin
class NetWorkUtils {

    companion object {
        fun isNetworkAvailable(context: Context): Boolean {
            val manager = context.applicationContext.getSystemService(
                    Context.CONNECTIVITY_SERVICE) as ConnectivityManager
            val info = manager.activeNetworkInfo
            return !(null == info || !info.isAvailable)
        }
    }
}
```

__CacheInterceptor__

```kotlin
class CacheInterceptor(var context: Context) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        var request = chain.request()
        if (NetWorkUtils.isNetworkAvailable(context)) {
            val response = chain.proceed(request)
            val maxAge = 60
            return response.newBuilder()
                .removeHeader("Pragma")
                .removeHeader("Cache-Control")
                .header("Cache-Control", "public, max-age=" + maxAge)
                .build()
        } else {
            request = request.newBuilder()
                .cacheControl(CacheControl.FORCE_CACHE)
                .build()
            val response = chain.proceed(request)
            val maxStale = 60 * 60 * 24 * 3
            return response.newBuilder()
                .removeHeader("Pragma")
                .removeHeader("Cache-Control")
                .header("Cache-Control", "public, only-if-cached, max-stale=" + maxStale)
                .build();
        }
    }
}
```

__RetrofitClient__

```kotlin
abstract class BaseRetrofitClient(var context: Context, var baseUrl: String) {
    companion object {
        private const val DEFAULT_TIMEOUT = 20L
    }

    fun <S> getService(serviceClass: Class<S>): S {
        return Retrofit.Builder()
            .client(
                OkHttpClient.Builder()
                    .addInterceptor(CacheInterceptor(context))
                    .connectTimeout(DEFAULT_TIMEOUT, TimeUnit.SECONDS)
                    .writeTimeout(DEFAULT_TIMEOUT, TimeUnit.SECONDS)
                    .build()
            )
            .addConverterFactory(GsonConverterFactory.create())
            .baseUrl(baseUrl)
            .build().create(serviceClass)
    }
}
```

__具体业务Client__

```kotlin
const val WAN_BASE_URL = "https://www.wanandroid.com"

object WanClient : BaseRetrofitClient(AppApplication.context, WAN_BASE_URL) {
    val service by lazy { getService(WanService::class.java) }
}
```

__返回数据包装类__

```kotlin
sealed class WanResult<out  T:Any> {

    data class Success<out T : Any>(val data: T) : WanResult<T>()
    data class Error(val exception: Exception) : WanResult<Nothing>()

    override fun toString(): String {
        return when (this) {
            is Success<*> -> "Success[data=$data]"
            is Error -> "Error[exception=$exception]"
        }
    }
}
```

```kotlin
data class WanResponse<out T>(val errorCode: Int, val errorMsg: String, val data: T)
```

__API__

```kotlin
interface WanService {

    @GET("/article/list/{page}/json")
    suspend fun getHomeArticles(@Path("page") page: Int): WanResponse<ArticleList>

    @GET("/banner/json")
    suspend fun getBanner(): WanResponse<List<Banner>>

    @GET("/tree/json")
    suspend fun getSystemType(): WanResponse<List<SystemParent>>

    @GET("/article/list/{page}/json")
    suspend fun getSystemTypeDetail(@Path("page") page: Int, @Query("cid") cid: Int): WanResponse<ArticleList>

    @GET("/navi/json")
    suspend fun getNavigation(): WanResponse<List<Navigation>>

    @GET("/project/tree/json")
    suspend fun getProjectType(): WanResponse<List<SystemParent>>

    @GET("/wxarticle/chapters/json")
    suspend fun getBlogType(): WanResponse<List<SystemParent>>

    @GET("/wxarticle/list/{id}/{page}/json")
    fun getBlogArticle(@Path("id") id: Int, @Path("page") page: Int): WanResponse<ArticleList>

    @GET("/project/list/{page}/json")
    suspend fun getProjectTypeDetail(@Path("page") page: Int, @Query("cid") cid: Int): WanResponse<ArticleList>

    @GET("/article/listproject/{page}/json")
    suspend fun getLastedProject(@Path("page") page: Int): WanResponse<ArticleList>

    @GET("/friend/json")
    suspend fun getWebsites(): WanResponse<List<Hot>>

    @GET("/hotkey/json")
    suspend fun getHot(): WanResponse<List<Hot>>

    @FormUrlEncoded
    @POST("/article/query/{page}/json")
    suspend fun searchHot(@Path("page") page: Int, @Field("k") key: String): WanResponse<ArticleList>

    @GET("/lg/collect/list/{page}/json")
    suspend fun getCollectArticles(@Path("page") page: Int): WanResponse<ArticleList>

    @POST("/lg/collect/{id}/json")
    suspend fun collectArticle(@Path("id") id: Int): WanResponse<ArticleList>

    @POST("/lg/uncollect_originId/{id}/json")
    suspend fun cancelCollectArticle(@Path("id") id: Int): WanResponse<ArticleList>

    @GET("/user_article/list/{page}/json")
    suspend fun getSquareArticleList(@Path("page") page: Int): WanResponse<ArticleList>

    @FormUrlEncoded
    @POST("/lg/user_article/add/json")
    suspend fun shareArticle(@Field("title") title: String, @Field("link") url: String): WanResponse<String>
}
```

__BaseRepository__

```kotlin
open class BaseWanRepository {

    suspend fun <T : Any> safeApiCall(
        call: suspend () -> WanResult<T>,
        errorMessage: String
    ): WanResult<T> {
        return try {
            call()
        } catch (e: Exception) {
            WanResult.Error(IOException(errorMessage, e))
        }
    }

    suspend fun <T : Any> executeResponse(
        response: WanResponse<T>, successBlock: (suspend CoroutineScope.() -> Unit)? = null,
        errorBlock: (suspend CoroutineScope.() -> Unit)? = null
    ): WanResult<T> {
        return coroutineScope {
            if (response.errorCode == -1) {
                errorBlock?.let { it() }
                WanResult.Error(IOException(response.errorMsg))
            } else {
                successBlock?.let { it() }
                WanResult.Success(response.data)
            }
        }
    }
}
```