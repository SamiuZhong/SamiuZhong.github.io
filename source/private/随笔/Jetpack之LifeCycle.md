出处：[硬核讲解 Jetpack 之 LifeCycle 使用篇](https://juejin.im/post/5df64c19518825121d6e2013)

比如我要在生命周期里面做一些事情，以前这样写

```kotlin
	override fun onPause() {
        super.onPause()
        player1?.onVideoPause()
        player2?.onVideoPause()
    }

    override fun onResume() {
        super.onResume()
        player1?.onVideoResume()
        player2?.onVideoResume()
    }

    override fun onDestroy() {
        super.onDestroy()
        GSYVideoManager.releaseAllVideos()
    }
```

现在用lifeCycle可以这样写

单独写一个lifeCycle的类

```kotlin
class PlayerLifecycle(private val player: StandardGSYVideoPlayer?):LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_PAUSE)
    fun pausePlayer(){
        player?.onVideoPause()
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_RESUME)
    fun resumePlayer(){
        player?.onVideoResume()
    }

    @OnLifecycleEvent(Lifecycle.Event.ON_DESTROY)
    fun destroyPlayer(){
        GSYVideoManager.releaseAllVideos()
    }
}
```

然后在activity里面添加订阅即可

```kotlin
	private var player1: StandardGSYVideoPlayer? = null
    private var player2: StandardGSYVideoPlayer? = null
    
    private fun initView(){
    	lifecycle.addObserver(PlayerLifecycle(player1))
        lifecycle.addObserver(PlayerLifecycle(player2))
    }
```

