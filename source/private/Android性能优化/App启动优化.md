## 冷的分类

- 冷启动

  耗时最多，是衡量标准

  click -> IPC -> Process.start -> ActivityThread -> bindApplication -> LifeCYcle -> ViewRootImpl

- 热启动

  最快

- 温启动

  重走生命周期

## 优化方向

- Application和Activity生命周期

## 启动时间的测量方式

### adb命令

adb shell am start -W packagename/首屏Activity（这里也需要包名）

- ThisTime：最后一个Activity启动耗时
- TotalTime：所有Activity启动耗时
- WaitTime：AMS启动Activity的总耗时

### 手动打点

启动时埋点，启动结束埋点，二者差值

## 总结

### 异步优化

在初始化init的时候使用线程池异步初始化一些SDK等等，拉满CPU的核心

