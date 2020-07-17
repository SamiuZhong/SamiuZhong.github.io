# 注入框架Koin

__AppApplication__

```kotlin
class AppApplication : Application() {

    companion object {
        var context: Context by Delegates.notNull()
    }

    override fun onCreate() {
        super.onCreate()
        context = applicationContext
        LiveEventBus.config()

        startKoin {
            androidContext(this@AppApplication)
            modules(wanModule)
        }
    }
}
```

__CoroutineDispatcherProvider__

```kotlin
data class CoroutineDispatcherProvider(
    val main: CoroutineDispatcher = Main,
    val computation: CoroutineDispatcher = Default,
    val io: CoroutineDispatcher = IO
) {
    constructor() : this(Main, Default, IO)
}
```

__Module__

```kotlin
val wanViewModelModule = module {
    viewModel { HomeViewModel(get()) }
}

val wanRepositoryModule = module {
    single { WanClient.service }
    single { CoroutineDispatcherProvider() }
    single { HomeRepository() }
}

val wanModule = listOf(
    wanViewModelModule,
    wanRepositoryModule
)
```

# 使用

__具体Repository__

```kotlin
class HomeRepository : BaseWanRepository() {

    suspend fun getBanners(): WanResult<List<Banner>> {
        return safeApiCall(
            call = {
                executeResponse(WanClient.service.getBanner())
            }, errorMessage = ""
        )
    }

    suspend fun getArticlesList(page: Int): WanResult<ArticleList> {
        return safeApiCall(
            call = {
                executeResponse(WanClient.service.getHomeArticles(page))
            }, errorMessage = ""
        )
    }
}
```

__具体ViewModel__

```kotlin
class HomeViewModel(
    private val homeRepository: HomeRepository
) : BaseViewModel() {

    val mBanners: LiveData<List<Banner>> = liveData {
        kotlin.runCatching {
            val data = homeRepository.getBanners()
            if (data is WanResult.Success) emit(data.data)
        }
    }
}
```

__LiveData传递数据到页面__

```kotlin
class WanHomeFragment : BaseVMFragment<HomeViewModel>() {
    override fun getLayoutResId() = R.layout.fragment_wan_home
    override fun initView() = Unit
    override fun initData() = Unit

    private val mViewModel: HomeViewModel by viewModel()

    override fun startObserve() {
        mViewModel.run {
            mBanners.observe(this@WanHomeFragment, Observer { setBanner(it)})
        }
    }

    private fun setBanner(bannerList: List<Banner>) {
        banner.adapter = ImageBannerAdapter(bannerList)
        banner.start()
    }

    override fun onStart() {
        super.onStart()
        banner.start()
    }

    override fun onStop() {
        super.onStop()
        banner.stop()
    }
}
```

