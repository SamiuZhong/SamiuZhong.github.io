## 播放Service和自定义控件的实现

主要就是三个单例

MusicPlayer

```java
public class MusicPlayer extends MediaPlayer {

    private volatile static MusicPlayer instance;

    private MusicPlayer() {
    }

    public static MusicPlayer getInstance() {
        if (instance == null)
            synchronized (MusicPlayer.class) {
                if (instance == null)
                    instance = new MusicPlayer();
            }
        return instance;
    }

    public static void releaseInstance() {
        instance = null;
    }
}
```

MusicControlBar

```java
public class MusicControlBar2 extends RelativeLayout {

    public static Context sContext;

    private static class ControlBarHolder {
        private static final MusicControlBar2 instance = new MusicControlBar2(sContext);
    }

    public static MusicControlBar2 getInstance() {
        return ControlBarHolder.instance;
    }

    public MusicControlBar2(Context context) {
        super(context);
        initView(context);
    }

    public MusicControlBar2(Context context, AttributeSet attrs) {
        super(context, attrs);
        initView(context);
    }

    private ImageView avatar;
    private FrameLayout playLayout;
    private TextView title;
    private FrameLayout close;
    private ImageView playIcon;
    private ImageView pauseIcon;

    @SuppressLint("CheckResult")
    private void initView(Context context) {
        LayoutInflater.from(context).inflate(R.layout.layout_music_control_bar, this, true);
        avatar = findViewById(R.id.play_avatar);
        playLayout = findViewById(R.id.play_icon_2);
        title = findViewById(R.id.play_title);
        close = findViewById(R.id.play_close);
        playIcon = findViewById(R.id.control_play_icon);
        pauseIcon = findViewById(R.id.control_pause_icon);

        pauseIcon.setSelected(false);
        playLayout.setOnClickListener(v -> {
            mCallback.onPlayMusic();
        });
        close.setOnClickListener(v -> {
            mCallback.onCloseMusic();
        });

        RxBus.getDefault().toObservable(Boolean.class)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(aBoolean -> {
                    if (aBoolean)
                        bindPlayIcon();
                });
    }

    public void bindPlayIcon() {
        if (playIcon.isSelected()) {
            playIcon.setSelected(false);
            playIcon.setVisibility(View.VISIBLE);
            pauseIcon.setVisibility(View.GONE);
        } else {
            playIcon.setSelected(true);
            playIcon.setVisibility(View.GONE);
            pauseIcon.setVisibility(View.VISIBLE);
        }
    }

    public void setTitle(String title) {
        this.title.setText(title);
    }

    public void setAvatar(String avatarRes) {
        GlideUtils.loadCircleImage(avatar, avatarRes);
    }

    private MusicPlayCallback mCallback;

    interface MusicPlayCallback {
        void onPlayMusic();

        void onCloseMusic();
    }

    public void setMusicCallback(MusicPlayCallback callback) {
        this.mCallback = callback;
    }
}
```

FloatMusicService

```java
public class FloatMusicService extends Service {

    private volatile static FloatMusicService instance;

    public static FloatMusicService getInstance() {
        return instance;
    }

    public static boolean isStarted = false;
    private Boolean prepared = false;
    private int addCount;

    private MusicControlBar2 mControlBar;

    private MusicPlayer mPlayer;
    private WindowManager windowManager;
    private WindowManager.LayoutParams layoutParams;

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        instance = this;
        isStarted = true;
        addCount = 0;
        windowManager = (WindowManager) getSystemService(WINDOW_SERVICE);
        layoutParams = new WindowManager.LayoutParams();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            layoutParams.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;
        } else {
            layoutParams.type = WindowManager.LayoutParams.TYPE_PHONE;
        }
        layoutParams.format = PixelFormat.RGBA_8888;
        layoutParams.gravity = Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL;
        layoutParams.flags = WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL | WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
        layoutParams.width = WindowManager.LayoutParams.MATCH_PARENT;
        layoutParams.height = WindowManager.LayoutParams.WRAP_CONTENT;

        MusicControlBar2.sContext = getApplicationContext();
    }

    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        showFloatingWindow();
        return super.onStartCommand(intent, flags, startId);
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        stopPlay();
    }

    private void playMusic(String url) {
        try {
            mPlayer.reset();
            mPlayer.setDataSource(url);
            mPlayer.prepareAsync();
            mPlayer.setOnPreparedListener(MediaPlayer::start);
            prepared = true;
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    @SuppressLint("ClickableViewAccessibility")
    public void showFloatingWindow() {
        if (Settings.canDrawOverlays(this)) {
            mControlBar = MusicControlBar2.getInstance();
            mControlBar.setAvatar(Constants.TEST_IMAGE_URL);
            mControlBar.setTitle("北京城的毛主席");
            bindPlayMusic();
            mControlBar.setMusicCallback(new MusicControlBar2.MusicPlayCallback() {
                @Override
                public void onPlayMusic() {
                    bindPlayMusic();
                }

                @Override
                public void onCloseMusic() {
                    bindCloseMusic();
                }
            });
            if (addCount == 0) {
                windowManager.addView(mControlBar, layoutParams);
                addCount++;
            }
        }
    }

    /**
     * 播放
     */
    public void bindPlayMusic() {
        RxBus.getDefault().post(true);
        if (!isPrepared())
            startPlay(Constants.TEST_MUSIC_URL);
        else if (!isPlaying())
            continuePlay();
        else
            pausePlay();
    }

    /**
     * 停止播放并移除悬浮窗
     */
    private void bindCloseMusic() {
        if (isStarted = true) {
            isStarted = false;
            if (isPlaying())
                RxBus.getDefault().post(true);
            stopPlay();
            windowManager.removeView(mControlBar);
        }
    }

    /**
     * 开始播放
     *
     * @param url
     * @throws IOException
     */
    public void startPlay(String url) {
        mPlayer = MusicPlayer.getInstance();
        playMusic(url);
    }

    /**
     * 获取当前播放状态
     *
     * @return
     */
    public Boolean isPlaying() {
        return mPlayer.isPlaying();
    }

    /**
     * 获取当前准备状态
     *
     * @return
     */
    public Boolean isPrepared() {
        return prepared;
    }

    /**
     * 继续播放
     *
     * @return
     */
    public Boolean continuePlay() {
        if (!isPlaying())
            mPlayer.start();
        return isPlaying();
    }

    /**
     * 暂停播放
     *
     * @return
     */
    public Boolean pausePlay() {
        if (isPlaying())
            mPlayer.pause();
        return isPlaying();
    }

    /**
     * 停止播放
     */
    public void stopPlay() {
        if (prepared) {
            mPlayer.stop();
            mPlayer.release();
            MusicPlayer.releaseInstance();
            prepared = false;
            addCount = 0;
        }
    }

    /**
     * 获取当前播放进度
     *
     * @return
     */
    public int getCurrentProgress() {
        return mPlayer.getCurrentPosition();
    }

    /**
     * 获取总时长
     *
     * @return
     */
    public int getTotalProgress() {
        return mPlayer.getDuration();
    }
}
```

然后是控制器的布局文件

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/play_layout"
    android:layout_width="match_parent"
    android:layout_height="60dp"
    android:layout_marginStart="@dimen/dp_10"
    android:layout_marginEnd="@dimen/dp_10"
    android:layout_marginBottom="@dimen/dp_20"
    android:elevation="2dp"
    android:orientation="horizontal">

    <RelativeLayout
        android:layout_width="0dp"
        android:layout_height="match_parent"
        android:layout_weight="1"
        android:background="@drawable/shape_10_fff_left">

        <ImageView
            android:id="@+id/play_avatar"
            android:layout_width="48dp"
            android:layout_height="48dp"
            android:layout_centerVertical="true"
            android:layout_marginStart="@dimen/dp_6" />

        <FrameLayout
            android:id="@+id/play_icon_2"
            android:layout_width="wrap_content"
            android:layout_height="match_parent"
            android:layout_alignParentEnd="true"
            android:layout_centerVertical="true">

            <ImageView
                android:id="@+id/control_play_icon"
                android:layout_width="@dimen/dp_12"
                android:layout_height="@dimen/dp_16"
                android:layout_gravity="center_vertical"
                android:layout_marginStart="@dimen/dp_14"
                android:layout_marginEnd="@dimen/dp_22"
                android:src="@drawable/jiantou_green"
                android:visibility="gone" />

            <ImageView
                android:id="@+id/control_pause_icon"
                android:layout_width="@dimen/dp_12"
                android:layout_height="@dimen/dp_16"
                android:layout_gravity="center_vertical"
                android:layout_marginStart="@dimen/dp_14"
                android:layout_marginEnd="@dimen/dp_22"
                android:src="@drawable/music_pause" />
        </FrameLayout>

        <TextView
            android:id="@+id/play_title"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginStart="@dimen/dp_14"
            android:layout_marginTop="@dimen/dp_7"
            android:layout_toStartOf="@id/play_icon_2"
            android:layout_toEndOf="@id/play_avatar"
            android:singleLine="true"
            android:textSize="@dimen/sp_16"
            tools:text="案例是绝对啦" />
    </RelativeLayout>

    <FrameLayout
        android:id="@+id/play_close"
        android:layout_width="@dimen/dp_44"
        android:layout_height="match_parent"
        android:background="@drawable/shape_10_eee_right">

        <ImageView
            android:layout_width="@dimen/dp_12"
            android:layout_height="@dimen/dp_12"
            android:layout_gravity="center"
            android:src="@drawable/chacha_black" />
    </FrameLayout>
</LinearLayout>
```



## 使用

```java
public void startFloatMusic() {
    if (FloatMusicService.isStarted) {
        FloatMusicService service = FloatMusicService.getInstance();
        service.bindPlayMusic();
    } else if (!Settings.canDrawOverlays(this)) {
        Toast.makeText(this, "当前无权限，请授权", Toast.LENGTH_SHORT).show();
        startActivityForResult(new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:" + getPackageName())), 0);
    } else {
        startService(new Intent(this, FloatMusicService.class));
    }
}
```

