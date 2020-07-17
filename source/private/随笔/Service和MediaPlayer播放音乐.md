先创建音乐的Service
```
class MusicService : Service() {

    private val mPlayer = MediaPlayer()
    private val mBinder = MusicBinder()
    private var prepared = false

    override fun onBind(intent: Intent?) = mBinder

    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }

    private fun playMusic(url: String) {
        mPlayer.run {
            reset()
            setDataSource(url)
            prepareAsync()
            setOnPreparedListener { mp -> mp?.start() }
            prepared = true
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        mPlayer.run {
            stop()
            release()
            prepared = false
        }
    }

    //开始播放
    fun start(url: String) = playMusic(url)

    //获取当前播放状态
    fun isPlaying(): Boolean = mPlayer.isPlaying

    //获取当前准备状态
    fun isPrepared(): Boolean = prepared

    //继续播放
    fun play(): Boolean {
        if (!isPlaying())
            mPlayer.start()
        return isPlaying()
    }

    //暂停播放
    fun pause(): Boolean {
        if (isPlaying())
            mPlayer.pause()
        return isPlaying()
    }

    //获取当前播放时长位置
    fun getCurrent() = mPlayer.currentPosition

    //获取总时长
    fun getDuration() = mPlayer.duration
}
```

Activity里面绑定Service

```
private lateinit var mMusicService: MusicService
private var mBound: Boolean = false

private val connection = object : ServiceConnection {
    override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
        val binder = service as MusicService.MusicBinder
        mMusicService = binder.getService()
        mBound = true
    }
    override fun onServiceDisconnected(name: ComponentName?) {
        mBound = false
    }
}

override fun onStart() {
    super.onStart()
    Intent(this, MusicService::class.java).also { intent ->
        bindService(intent, connection, Context.BIND_AUTO_CREATE)
    }
}

override fun onStop(){
    super.onStop()
    unbindService(connection)
    mBound = false
}
```
